---
title: "WSL2 で KV が足りなかった話 → ベアメタル Linux で KV プールが 2.1M トークンに化けた"
emoji: "🚀"
type: "tech"
topics: ["local-llm", "llm", "nvidia", "vllm", "wsl"]
published: false
---

# はじめに

RTX 5090 (32GB, sm_120 Blackwell) + vLLM で Ornith-1.5-35B-A3B を運用していました。

しばらく WSL2 で走らせていたのですが、どうもうまくいかない。decode は 150 tok/s 出るのに、
**64k×4 並行で必ず preemption(割り込み)が起きる**。「KV キャッシュが足りない」と何度も悩みました。

結論から言うと、**WSL2 という OS 構成自体がボトルネック**でした。
ベアメタル Linux に移したら **KV プールが 4.6〜10 倍**になり、preemption がほぼ消えました。

今回はそこで得たデータと、ハマりどころをまとめます。

---

# なぜ WSL2 だと KV が足りないのか

WSL2 では Windows の **WDDM VRAM 予約**が GPU メモリを確保します。
結果として、vLLM に回せる KV キャッシュ用 VRAM が**ざっくり半分以下**に食われます。

他マシンで同一ハードの WSL2 → ベアメタル移行を実測した結果(lastloop-ai/vllm-blackwell-guide):

| 指標 | WSL2 | ベアメタル | 変化 |
|---|---|---|---|
| decode(長文生成) | 92 tok/s | 117 | **+27%** |
| prefill @6.8k | 3,853 | 4,510 | +17% |
| TTFT @6.8k | 1.78 s | 1.52 s | −15% |
| 4-stream aggregate | 232 | 377 | **+63%** |
| KV pool(35B-A3B) | ~204k | **~2.1M** | **~10倍** |

decode も 27% 上がりますが、**構造的な勝負は KV pool** です。WDDM の予約がなくなるだけで、
35B-A3B の KV pool は **2.1M トークン**まで伸びます。64k×4 どころか 128k でも余裕で、
preemption が「無かったこと」になります。

シングル decode の +27% は WDDM/dxg の submission overhead が消えたため、
バッチスループットの +63% は広がった KV pool の恩恵です。

---

# 手順

手順自体は想像通りですが、**落とし穴が 4 つ**ありました。先に並べておきます。

## 落とし穴リスト

### 1. Secure Boot × NVIDIA ドライバ(最大の壁)

- DKMS ビルドは Secure Boot に弾かれる(`Key was rejected by service`)
- ヘッドレス基板だと MOK 署名もやりにくい
- → **Canonical の pre-signed モジュール**を使う(署名ダンス不要)

```bash
# archive カーネルで pre-signed モジュールを導入
linux-modules-nvidia-<branch>-server-<kernelver>-generic
```

### 2. flashinfer JIT の CUDA toolchain 混在

- 症状: `ptxas fatal: Unsupported .version 9.3; current version is '9.0'`
- 原因: 古い `nvvm/bin/cicc` が残っている
- → 古い `site-packages/nvidia/cu13/{bin,nvvm}` を吹き飛ばして整合したセットを再導入

### 3. FP8 on SM120(Blackwell consumer)

- FP8 モデルを SM120 メインラインでロードすると `Unknown SF transformation` で死ぬ
- → `VLLM_USE_DEEP_GEMM=0` + `VLLM_USE_FLASHINFER_SAMPLER=0`
- 注意: SM120 の FLASH_ATTN は FP8 KV 非対応 → KV dtype は自動(BF16)
  (そのぶんを巨大な KV pool が補います)

### 4. `python3.12-dev` が必須

- 無いと Triton JIT と vLLM の model-inspection が壊れ、
  「Model architectures [...] failed to be inspected」という無意味な文言になる
- → `python3.12-dev` を入れておく

---

# serve コマンド

Linux 側ができたら、これはもう素直です。

```bash
vllm serve ornith-ai/Ornith-1.5-35B-A3B-NVFP4 \
  --served-model-name ornith-1.5-35b-a3b-nvfp4 \
  --host 0.0.0.0 --port 8889 \
  --trust-remote-code --max-model-len 262144 \
  --gpu-memory-utilization 0.90 \
  --enable-prefix-caching \
  --moe-backend marlin \
  --enable-auto-tool-choice --tool-call-parser qwen3_xml \
  --reasoning-parser qwen3
```

ポイント:

- `--moe-backend marlin` を明示(NVFP4 MoE では Marlin が速い)
- WSL で苦労して強制していた `--attention-config.backend=TRITON_ATTN` は**外す**(自動選択で flashinfer に)
- `--gpu-memory-utilization` の崖(0.94 で待つ、0.95 で collapse)は **WSL2 だけの現象**。ベアメタルでは気にしなくて良い

---

# まとめ

「環境汚れ」だと思っていたのは、実は **WSL2 という構造そのもの**でした。

ベアメタル Linux に 1 回移すだけで:

- シングル decode **+27%**
- バッチスループット **+63%**
- KV pool **4.6〜10 倍**
- `--gpu-memory-utilization` の崖ダンス消滅

モデルは変わりません。カードも変わりません。**OS 配置だけが全部**でした。

---

# 参考リンク

- [lastloop-ai/vllm-blackwell-guide](https://github.com/lastloop-ai/vllm-blackwell-guide)(本編 + BARE-METAL.md)
- allenkuo / Medium: "Bare-metal Linux is faster and always will be. The WSL2 ↔ bare-metal gap is real and measured."