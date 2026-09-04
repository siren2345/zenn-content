---
title: "ローカルLLMでベンチ豚（ベンチマーク豚）にならないためのHermes Agentの実践テク"
emoji: "🐖"
type: "tech"
topics: ["local-llm", "llm", "hermes", "nvidia", "rtx5090"]
published: true
---

# はじめに

ローカルLLMを触っていると、あっという間にベンチ豚になる。HFのスコア表を横に並べ、Q4_K_MとQ6_Kでperplexityを比べ、誰も使わない256Kを「対応」と書く。数字は伸びるが、体感は置いていかれる。

Hermes Agent（Nous Research）のローカルランタイムは、この沼を避ける作りになっていた。RTX 5090でQwen3.8-27Bを動かしながらコードを追ったので、ベンチ豚にならないための実践テクとしてまとめる。

結論は一言だ。**「測れる体験」で選び、数字は順位付けの道具に留める。**

![](/images/articles/local-llm-bench-pig-hermes-practice/diag-03-architecture.png)
*Hermesローカルランタイム全体像。catalogで1本に絞り、estimator→context policy→presets.ini→supervisorでllama-serverへ。Commit 43e67d872（NVIDIA field feedback on RTX 5090）*

# ベンチ豚が起きる理由

三つある。

1. **quantの段数を増やすほど選べた気になる**  
   Q3、Q4、Q5、Q6、Q8、UD-Q4_K_M、IQ4_XS……。HFには同じモデルの別quantが十数本並ぶ。理屈では「VRAMが余れば大きいquantが良い」に見えるが、作り手のimatrixと再現性で当たり外れが大きい。結果、ベンチだけ見て外れを引く。

2. **コンテキスト長を盛る**  
   train 262Kだから262Kで動く、という表示。WDDMのVRAM予約を無視してKVを積めば、decodeは9倍遅くなる。数字は大きいが、会話が進むとpreemptionで止まる。

3. **ダウンロード数で品質を決める**  
   スター数やDL数は人気であって性能ではない。ここで選ぶと、マーケティングの勝敗をなぞるだけになる。

Hermesはこの三つを全部断っている。

# Hermesの答え：カタログは1モデル1ビルド

実装は `hermes_cli/local_runtime/catalog.py` に集約されている。冒頭のコメントが思想を語る。

> Each model ships ONE build, Q4-class (UD-Q4_K_M where the repo has it, UD-Q4_K_XL elsewhere). Q4 is the quant class current engines optimize for and the sweet spot of the size/quality curve, so there is no quant ladder: headroom buys a bigger context window, never a bigger quant

意訳すると「1モデル1ビルド、Q4固定。余裕はquantを上げるのではなく窓を広げる方に使う」。Q4未満は`Below Q4 the quality loss is too severe to ship as someone's first local-AI experience`と切り捨てる。

実際のカタログ（`catalog.json`）もそうだ。

```json
{
  "id": "qwen3.8-27b",
  "display_name": "Qwen3.8 27B",
  "repo": "unsloth/Qwen3.8-27B-GGUF",
  "variants": [{ "quant": "UD-Q4_K_M", "files": [{ "path": "Qwen3.8-27B-UD-Q4_K_M.gguf", "size_bytes": 16464440224 }] }],
  "quality": 90
}
```

同じモデルの別quantは登録しない。テストも1本に集約されるから、当たり外れの分散が減る。

![](/images/articles/local-llm-bench-pig-hermes-practice/diag-01-catalog-flow.png)
*1モデル1ビルドの選抜フロー。Q4固定→AA quality→validated→速度ゲートで「このマシンで気持ちよく動く」1本だけがRecommendedになる*

## 品質はAAで決める、表示はしない

`quality`は0〜100の整数で、並び順だけに使う。

```py
# catalog.py
# Editorial quality ordering (higher = smarter), authored once,
# globally, at catalog-authoring time — Artificial Analysis-informed
# where they cover the model (scripts/aa_quality_sync.py proposes,
# the commit decides), editorial elsewhere.
quality: int = 0
```

元は Artificial AnalysisのIntelligence Indexだ。同期は `scripts/aa_quality_sync.py` がAPIで取るが、ランタイムではAAに一切問い合わせない。`AA_API_KEY` で取得→差分をprint→人間が`catalog.json`を手で編集する運用だ。REAMEにも`their terms forbid client-side keys, the fleet would burn the rate limit`とある。ベンチを毎回取りに行くからブレる、という失敗を最初から潰している。

現在の値はこうだ（`catalog.json`より）。

| id | quality | 備考 |
|---|---|---|
| qwen3.8-flash-next | 95 | Frontier、MoE |
| qwen3.8-27b | 90 | 今回動かしたやつ |
| deepseek-v4-flash | 85 | 128GB+向け |
| qwen3.6-35b-a3b | 80 | validated:true |

スコア自体はUIに出さない。`recommended_entry()` が`best-quality-resident` / `speed-gated-quality` といった理由キーだけを返し、Recommendedバッジのツールチップにそれが入る。数字を信仰させない作りだ。

## validatedフラグで実機をゲートする

```py
# catalog.py
validated: bool = False  # proven end-to-end on real hardware
```

> Builds proven end-to-end on real hardware are marked validated. Day-0 entries ship before that proof (they simply lack the validated flag) — ensure_model_ready's touch generation still gates every first load

qwen3.8-27bは今`validated: false`のDay-0、qwen3.6-35b-a3bは`validated: true`だ。未検証でも配るが、初回ロードでコケたら即エラーにして誤魔化さない。HFのサイズドリフトも到達性チェックで検知する。壊れたGGUFをDL時に握りつぶさない。

# 窓は盛らない、梯子で登る

ベンチ豚の二つ目、コンテキスト長の盛りを潰すのが `context_policy.py` だ。

```py
FLOOR = 64 * 1024
TARGET_WINDOW = 144 * 1024
RUNTIME_OVERHEAD_BYTES = int(1.5 * (1 << 30))
```

* **FLOOR 64Kは保証**。重みだけでVRAMが溢れても、重みをhostにspillしてでも64Kは守る。`Explicit context size makes the fit spill weights and hold the window rather than shrink it` という実測に基づく。
* **TARGET 144Kは体験の分岐点**。161セッションの実測で `64Kで66%、96Kで82%、144Kで91%が非圧縮で完走、216Kは+6ptしか伸びない`。ここから先はquantを下げるコストが上回る。だから余裕は窓を144Kまで広げる方に使う。
* **梯子（ladder）は `64K → 96K → 144K → ... → native`** を1.5倍で登る。セッションの85%到達かつサーバーがidleのときだけ、次のrungを再フィットする。WDDMで`allocating past residency slows decode roughly 9x`という実測があるから、`over-allocation is the slow path`として、その瞬間の空きVRAMで再判定する。

![](/images/articles/local-llm-bench-pig-hermes-practice/diag-02-ladder.png)
*コンテキストの梯子。FLOOR 64Kを保証し、TARGET 144Kで91%完走、216Kは+6pt止まり。5090では物理で入る221Kで「盛らずに出す」*

RTX 5090（32GB discrete）でQwen3.8-27Bを動かした実例がこれだ。

```
--ctx-size 221184 --cache-type-k q8_0 --cache-type-v q8_0 --flash-attn on
--spec-type draft-mtp --spec-draft-n-max 2
weights 15.33GB + overhead 1.5GB + mmproj 0.89GB + ub_logits ~1GB + KV(221K)
→ 31.5/32.6GB 使用、ゼロスピルで収まる最大rungとして221Kを選択
```

![](/images/articles/local-llm-bench-pig-hermes-practice/diag-04-evidence.png)
*実機の証跡。63548でQwen3.8-27Bがhealth ok、VRAM 31.5/32.6GB、MTP 66% accept、presets.iniで221Kが選ばれている*

`presets.ini` は `bootstrap.py:260` で生成される。

```ini
[Qwen3.8-27B-UD-Q4_K_M]
ctx-size = 221184
cache-type-k = q8_0
flash-attn = on
spec-type = draft-mtp
```

262Kを名乗らず、物理で入る221Kで出す。ここがベンチ豚との分かれ目だ。

# 速度でゲートする

窓が決まっても、遅ければ体験は壊れる。Hermesは二つの速度ゲートを持つ。

* **PLEASANT_FLOOR 20 tok/s** - `recommended_entry()` の推奨ゲート。`predicted_decode_tok_s = bandwidth * 1e9 / bytes_per_token` で予測する。帯域は `DISCRETE 1000GB/s / UMA 210GB/s / HOST 80GB/s` の三段階で、MoEは `decode_fraction`（例: Flash-Nextは0.08）で読む量を割り引く。pleasantを誰も超えなければ`fastest-resident`、誰もresidentしなければ`least-painful-spilled`に倒す。

* **SPEED_FLOOR 6 tok/s** - `growth_decision()` の成長停止ゲート。`the deepest measured host-spilled configuration bottomed out near this rate`。ここを割ったら圧縮をデフォルトにし、深い窓は明示的なユーザー選択にする。

> Speculative decoding (MTP) defaults on only for spilled configs, where its speedup is largest (measured 1.43x spilled vs 1.35x resident).

この辺も実測だ。今回のQwen3.8はMTPで `spec_decode_num_accepted_tokens 10821/16399 (66%)` と効いていた。

# 実装を追うときの地図

コミットは `43e67d872 feat: local models — managed llama.cpp runtime with one-click desktop setup` 一発にまとまっている。`Co-developed with NVIDIA field feedback on RTX 5090 and DGX Spark.` とある。

追うならこの順だ。

1. `catalog.py` / `catalog.json` - 何を配るか
2. `estimator.py` / `hardware.py` - VRAM/RAM/UMAをどう測るか
3. `context_policy.py` - 窓をどう決め、どう育てるか
4. `presets.py` / `bootstrap.py` - どう`llama-server`に渡すか（`presets.ini`）
5. `supervisor.py` / `endpoint.py` - どう常駐させ、どうルーティングするか

`This is deliberately not a live registry feed: entries are reviewed like a version bump` という一文が全体を要約している。生のレジストリを流さず、版としてレビューする。ベンチの自動更新で順位が踊らないようにする工夫だ。

# まとめ

ベンチ豚にならないコツを、Hermesのコードから引き直すとこうなる。

* **段数を増やさない**。Q4 1本に絞り、余裕は窓に使う
* **人気で選ばない**。AAの知能指数 + 人間のeditorialで並べ、UIには理由だけ出す
* **盛らない**。64Kを保証し、144Kを目標に、梯子で登る。WDDMの実測を信じる
* **遅いものは推さない**。20 tok/sと6 tok/sの二段ゲートで、速さが体験を下回ったら推奨を変える
* **検証は実機で**。`validated`フラグと到達性チェックで、理屈の良さをそのまま信じない

ベンチは順位付けの道具であって、目的ではない。目的は「このマシンで気持ちよく動く」ことだ。その一点でカタログも窓も速度も決める。Hermesのローカルランタイムは、その割り切りをコードで固定した例だと思う。

---

*環境: FRONTIER BTO / Ryzen 9 9950X3D / RTX 5090 32GB / Windows 11 / Hermes Agent main (2026-09-03) / llama.cpp b10679 cuda / Qwen3.8-27B-UD-Q4_K_M (15.33GB, ctx 221184, q8_0 KV, flash-attn on, draft-mtp depth 2)*
