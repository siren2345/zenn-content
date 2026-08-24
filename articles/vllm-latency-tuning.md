---
title: "ローカルLLMの「遅い」を半分に — vLLMの地味な2設定(prefix cachingとバッチサイズ)"
emoji: "⚡"
type: "tech"
topics: ["local-llm", "llm", "nvidia", "performance"]
published: false
---

# はじめに

RTX 5090 (32GB) + vLLM でローカルLLMをコードエージェント(opencode)として使っていたら、「動けるようになった」後もずっと「**応答が遅い**」という不満が残っていました。

調査してわかったことは、地味な話でした。

- 「遅い」の正体の大部分は **TTFT(最初のトークンまでの待ち時間)**、つまり **prefill** でした
- 原因は vLLM の設定2つ。どちらもフラグ1本で直りました
  1. **prefix caching が効いていなかった** → 毎ターン会話全体を再prefill
  2. **`--max-num-batched-tokens` が 512 に絞られていた** → prefillが512トークン単位で細切れに

効果は次の通りです。

- 65Kコンテキストの同一プロンプト: **TTFT 29.5s → 2.8s (10.6倍)**
- キャッシュ無効の64K: **31.7s → 22.7s (−28%)**
- 生成速度(TPOT): **約1割高速化**

この記事は「遅い → 分解 → 計測 → 修正 → 検証」の流れそのものを記録したものです。

## 環境

| 項目 | 内容 |
| --- | --- |
| GPU | NVIDIA RTX 5090 (32GB, Blackwell / sm_120) |
| OS | Windows 11 + WSL2 (Ubuntu) |
| サーバー | vLLM 0.27.1 (OpenAI互換API) |
| モデル | `Qwen3.8-27B-NVFP4` (NVFP4量子化、ネイティブ262Kコンテキスト) |
| 構成 | MTP-3投機的デコード、TurboQuant 4bit KVキャッシュ (5.5GiB)、単一セッション (`--max-num-seqs 1`) |
| クライアント | opencode |

モデルは64層中48層がlinear attention (Gated DeltaNet: 状態サイズ一定) + 16層がfull attentionというハイブリッド構成で、これが「262Kコンテキストが5.5GiBの4bit KVに載る」理由です。

---

# 第1章: 「遅い」を分解する

「遅い」は曖昧なので、まず2指標に分解します。

| 指標 | 計るもの | コストの所在 |
| --- | --- | --- |
| **TTFT** (Time To First Token) | 最初のトークンが出るまでの待ち時間 | **prefill** (プロンプト全体を読む) |
| **TPOT** (Time Per Output Token) | 以降1トークンずつの間隔 | **decode** |

エージェント用途では、1ターン1ターンのプロンプトが巨大です。

```
system prompt + ツール定義 + 会話履歴全文 + 読み込んだファイル群
= 50K〜150K+ トークン が毎回送られてくる
```

つまりエージェント用途では、**「遅い」の主犯はほぼ常にTTFT(prefill)側**です。実際に本環境では、キャッシュ無効のまま ~150K コンテキストで **1ターンあたり 50s 前後** 待っていました。

## 計測の道具

- **`vllm.log`** — リクエスト単位で `Avg prompt/generation throughput`、`Prefix cache hit rate`、`SpecDecoding metrics` が出る
- **`/metrics`** — Prometheus形式の生指標
- **`vllm bench serve`** — 標準ベンチ(`--dataset-name random` でキャッシュ無効相当)
- **自前ストリーミング計測スクリプト** — ランダムコンテンツ(キャッシュ必ずミス)でTTFT/TPOTを実測

:::message
**ハマりどころ**: 計測しながら **`nvidia-smi` の `memory.used` で「メモリが張ってる/張ってない」を判断しない**。
vLLM は `gpu_memory_utilization` でVRAMを**事前確保**するため、prefill時のアクティベーション増加分は `used` に現れません。OOM判定はログの `OutOfMemory|CUDA error` を見るのが正しいです。
:::

# 第2章: 最適化1 — prefix caching を有効にする

## 症状

vLLM の `enable_prefix_caching=False`(運用スクリプトの既定)のままだったため、**毎ターン会話全体を再prefill** していました。system promptやツール定義、会話履歴の既出部分まで毎回読み直す、という状態です。

## 修正

フラグ1本です。

```bash
# 起動スクリプトの SERVE_ARGS に
--enable-prefix-caching
```

## 効果

~65Kトークンの同一プロンプトを2回送るA/B比較。

| | 変更前 | 変更後 |
| --- | --- | --- |
| TTFT | 29,470 ms | **2,779 ms (10.6倍)** |
| TPOT | ~21–22 ms | ~21–22 ms (不変) |
| Prefix cache hit rate | — | 62–70% |

prefix caching は同一prefixのKVキャッシュを再利用します。エージェントの会話は「先頭がほぼ不変・末尾だけ成長」する形で伸びるので、ヒット率が残りやすいのが効いています。

:::message
**教訓**: 派手なチューニングより先に「基本の最適化がONか」を確かめる。vLLMのエージェント用途ではprefix cachingが最大の効果源でした。
:::

# 第3章: 最適化2 — `--max-num-batched-tokens` 512 → 8192

## 症状: vLLM 自身が警告を出していた

prefix caching で「会話が既出部分」は解決しましたが、「**まだ読んだことのない新規コンテンツ**」のprefillは引き続き遅いままでした。原因は、起動ログの**vLLM自身の警告**が示していました。

```
WARNING [vllm.py] max_num_scheduled_tokens is set to 512 based on the
speculative decoding settings. This may lead to suboptimal performance.
Consider increasing max_num_batched_tokens ...
```

元の起動スクリプトには、こう入っていました。

```bash
--max-num-batched-tokens 512     # bound long-prefill activation peaks
```

`max_num_batched_tokens` は **エンジン1ステップで処理するトークン数の上限**で、chunked prefillのチャンクサイズにもなります。512に固定していたのは「長prefillのアクティベーションピークを絞るため」の保守設定でした。

64Kトークンのprefillは 65536 / 512 = **128ステップ** に分割され、ステップごとの固定オーバーヘッド(スケジューリング、CUDA graphディスパッチ、同期)が積み上がります。

## 修正

```bash
--max-num-batched-tokens 8192    # vLLMの既定値
```

補足として、投機的デコードとの関係:

- vLLMは `max_num_scheduled_tokens = max_num_batched_tokens − (ドラフトスロット × max_num_seqs)` を計算する
- MTPはserial drafting(ドラフトスロット=0)なので **scheduled == batched** で、そのまま512が「8192未満=suboptimal」として警告される
- 8192より大きくすると、実attention計算(KVが伸びて計算量が増える側)が支配となり効果は逓減。この環境では8192で飽和

## 効果 (乱数コンテンツ・prefix cache無効)

| 指標 | 512 | 8192 |
| --- | --- | --- |
| TTFT @ ~16K | 4.7 s | **3.2 s (−32%)** |
| TTFT @ ~64K | 31.7 s (平均) | **22.7 s (平均, −28%)** |
| TPOT @ ~25K ctx | 64 ms | **57 ms (−11%)** |
| 標準ベンチ (2K in / 512 out) の mean TPOT | ~21 ms | **18.6 ms** |

decode側も複数ランで安定して改善が見られました(機構自体は未解明ですが、標準ベンチでも同じ方向)。

# 第4章: 検証規律

prefillパスに関わる設定を変えたので、そのまま採用せず「壊れていないか」の確認を走らせたうえで勝ちとしました。

1. **OOM / CUDA エラーなし** — ログの `OutOfMemory|CUDA error` をgrep(64Kの8192トークンチャンクprefillも問題なし)
2. **文字化けなし** — 10K prefix + 算術問題(thinking ON)で正解 ✓、ツール呼び出しパースで well-formed な `tool_calls` ✓
   この構成は「MTP × 4bit KV で出力が壊れる」既知の問題(vllm#40880)があり、upstream修正(PR #40914)のバックポートで運用しているため、**推論・attention関連の設定変更後は毎回正解性チェックをやる**のが運用ルールです
3. **prefix cacheヒット率が不変** (62–64%)
4. **警告が消えた** (起動ログに `suboptimal performance` が0件)

# まとめ

| 変更 | 効果 |
| --- | --- |
| `--enable-prefix-caching` | 会話の既出部分が再利用される。65KでTTFT 10.6倍 |
| `--max-num-batched-tokens` 512 → 8192 | 新規コンテンツのprefillが −28〜−32%、TPOTも −11% |

「遅い」を **TTFT/TPOT に分解して計測 → ログとmetricsで原因を当てる → 地味な設定2つを直す → 壊れていないか検証する**。作業の大半は「計測」と「壊してないことの確認」でした。

# 教訓

- エージェント用途で「遅い」なら、**まずprefill(TTFT)を疑え**。decode速度は二の次
- **vLLM自身の起動ログの警告**が一次資料。`vllm serve --help` はサマリ表示のみで、個別フラグの詳しい説明は `--help=<ConfigGroup>`(例: `--help=SchedulerConfig`)
- **VRAM/OOM監視は `nvidia-smi` の `used` ではなくログ**。vLLMはVRAMを事前確保する
- **投機的デコード・attentionに関わる設定変更後は、必ず正解性チェック**(算術・ツール呼び出し)を回す
- 測った数値と再現コマンドは、設定ファイルと同じリポジトリに残す(A/B数値と再現手順は運用リポジトリの `BENCHMARKS.md` に保存済み)

---

- この記事は 2026-08 時点の構成(vLLM 0.27.1)です。
- 起動スクリプトは `siren2345/Qwen3.8-27B-NVFP4-RTX-5090` (fork) で管理しています。記事の数値の再現コマンドはリポジトリの `BENCHMARKS.md` にあります。
