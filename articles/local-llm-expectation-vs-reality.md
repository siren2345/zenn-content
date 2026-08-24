---
title: "ローカルLLMの期待と現実"
emoji: "🎭"
type: "tech"
topics: ["local-llm", "ollama", "claude-code", "opencode", "terraform"]
published: true
---

# はじめに

「ハイエンドGPUがあれば、ローカルLLMはすぐ快適に動くはず」

そう思って **RTX 5090 (32GB)** でローカル生成AI環境を構築しました。結論から言うと、**動くようになるまでが想像の3倍大変**で、しかも「動いた後」も設定の管理で別の苦労がありました。

この記事では、実際にハマった **期待と現実のギャップ** を「期待 → 現実」のペアで整理します。後半では、環境が完成した後にぶつかった **「設定を管理する」という第二段階の現実**（Terraform での IaC 化）も紹介します。

これからローカルLLM環境を組む人が、同じ轍を踏まないための記録です。

## 環境

| 項目 | 内容 |
| --- | --- |
| GPU | NVIDIA RTX 5090 (32GB VRAM) |
| LLM実行 | Ollama |
| エージェント | Claude Code / opencode |
| その他 | LM Studio, ComfyUI（動画生成） |
| モデル | `qwen3-coder:30b` (MoE) / `qwen3:32b` (dense) / `gemma-4B` / MiniMax H3 |

---

# 第1部: 「動かすまで」の期待と現実

## 期待1: 「32GBもあれば256Kコンテキストは余裕でしょ」

### 現実: denseモデルは非対応。しかもOllamaが勝手に窓を狭めていた

最初に試した `qwen3:32b`（dense）は、256Kコンテキストに **非対応** でした。しかも Ollama は搭載VRAMに応じてコンテキスト長を **自動的に制限** します。32GBあっても、この環境では既定で **約40K** まで絞られていました。

> エージェントフレームワーク（Hermes）が「このモデルは窓が40,960で、必要な64,000に満たない」と起動を拒否して、初めて自動制限の存在に気づきました。

**正解は MoE の `qwen3-coder:30b`** でした。

- 30B total / 3.3B active（アクティブパラメータが少ないので速い）
- ネイティブ 256K、最大 1M まで拡張可能
- 同じ30B級でも、denseより大コンテキスト運用に圧倒的に有利

:::message
**教訓**: 大コンテキストが欲しいなら「パラメータ数」ではなく「アーキテクチャ（MoEかどうか）」を見る。
:::

## 期待2: 「モデルをpullすれば、そのまま動く」

### 現実: 環境変数の永続設定とサーバー再起動が要る

`ollama pull` だけでは 256K は使えません。コンテキスト長は **環境変数** で明示する必要があり、設定後に **Ollamaサーバーの再起動** まで必要でした。

```powershell
# ユーザー環境変数として永続化
OLLAMA_CONTEXT_LENGTH=262144
```

「入れて終わり」ではなく、「入れて、宣言して、再起動して」がワンセットです。

## 期待3: 「`--model` で指定すれば、モデルは切り替わる」

### 現実: 設定ファイルがフラグを静かに上書きしていた

`ollama launch claude --model qwen3-coder:30b` と打っているのに、なぜかモデルが変わらない。原因は **Claude Code 側の設定ファイル** でした。

`~/.claude/settings.json` の `ANTHROPIC_MODEL` が **優先**され、コマンドラインの `--model` を上書きします。

```json
{
  "env": {
    "ANTHROPIC_MODEL": "qwen3-coder:30b",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "qwen3-coder:30b"
  }
}
```

:::message
**教訓**: モデルが切り替わらないとき、まず疑うのはフラグではなく `settings.json`。
:::

## 期待4: 「ツール同士は、勝手によく連携する」

### 現実: 連携の"すれ違い"が3連続で起きた

ローカルLLM環境は複数のツール（Ollama / Claude Code / opencode）が協調して動きますが、それぞれの **思い込み** がぶつかって何度も詰まりました。

### 4-1. 未認識モデルには「65.5K窓」を勝手に仮定される

Claude Code は認識できないモデルIDに対して、**デフォルト 65.5K** の窓を仮定します。`/context` に "unrecognized model" と出て初めて気づきました。正しい窓は自分で宣言する必要があります。

```powershell
CLAUDE_CODE_MAX_CONTEXT_TOKENS=262144
CLAUDE_CODE_AUTO_COMPACT_WINDOW=200000
```

### 4-2. ツール呼び出しのプロトコルが不一致

256K窓を実現しても、`qwen3-coder:30b` は Claude Code のツール呼び出しプロトコルを正しく生成できませんでした。資料の内容を「実行せよ」と誤読して勝手に `<function=Write>` を吐くなど、エージェントとして使えるレベルではありません。

- ツール呼び出しの形式（JSON `tool_calls`）と、モデルが返す形式（独自XML）が不一致
- auto-mode をオフにしても、モデル自体が誤読し続けるため解決しない

最終的に **opencode** に切り替えることで、ローカルモデルのツール呼び出し（WebFetch等）が正しく動くようになりました。

### 4-3. `ollama ps` が何も表示しない（ように見える）

モデルをロードしたのに `ollama ps` が空。焦りましたが、Ollama のロードは **遅延実行** で、最初のプロンプトを送って初めて `ps` に載ります。仕様でした。

---

# 第2部: 「動いた後」の期待と現実

環境が動いて一件落着……かと思いきや、ここから **第二段階の現実** が始まりました。

## 期待5: 「設定は一回やれば、ずっと持つ」

### 現実: 設定は複数ツールに散らばり、勝手にドリフトする

ローカル生成AI環境の設定は、一箇所にはまとまりません。

| ツール | 設定ファイル |
| --- | --- |
| Claude Code | `~/.claude/settings.json` |
| Ollama | `~/.ollama/config.json` |
| opencode | `~/.config/opencode/opencode.jsonc` |

モデルを変えるたびに **この3つを全部手で直す** 必要があり、しかもどれか一つを直し忘れると「片方だけ古いモデルを向いている」という **設定ドリフト** が起きます。第1部で踏んだ「`--model` が効かない」系の罠は、ほぼ全てこのドリフトが原因でした。

## 現実への対処: 設定を Terraform で IaC 化する

「手で3ファイル編集」を繰り返すのは脆いので、設定を **コードで管理** することにしました。選んだのは **Terraform** です。

### 方針

- `configs/` に **正（ソースオブトゥルース）** の設定ファイルを置く
- `terraform apply` で、それらを各ツールの **実ファイルへデプロイ** する
- 変更は `configs/` を編集 → `apply` の2ステップに集約

### 構成

```
local-ai-env/
  terraform/           # configs/ を実パスへデプロイ
  configs/             # 正: 各ツールの設定ファイル
    claude.settings.json   -> ~/.claude/settings.json
    ollama.config.json     -> ~/.ollama/config.json
    opencode.jsonc         -> ~/.config/opencode/opencode.jsonc
  comfyui/models.md    # 大容量モデルはマニフェストのみ
  docs/SETUP.md        # 256Kの知見・注意点
```

### Terraform 本体（抜粋）

```hcl
locals {
  home = pathexpand(var.home_dir)
}

resource "local_file" "claude_settings" {
  content  = file("${path.module}/../configs/claude.settings.json")
  filename = "${local.home}/.claude/settings.json"
}

resource "local_file" "ollama_config" {
  content  = file("${path.module}/../configs/ollama.config.json")
  filename = "${local.home}/.ollama/config.json"
}

resource "local_file" "opencode_config" {
  content  = file("${path.module}/../configs/opencode.jsonc")
  filename = "${local.home}/.config/opencode/opencode.jsonc"
}
```

`terraform plan` で「どのファイルがどう変わるか」を事前に確認でき、`apply` で一括反映できます。**「3つのファイルを頑張って手編集」から「1つの正を apply」に変わった** のが最大の収穫です。

:::message
**教訓**: ツールが増えるほど、設定は「手で同期」ではなく「コードで管理」しないと必ず壊れる。
:::

## 期待6: 「モデルはすぐ手に入る」

### 現実: 大容量ダウンロードとの戦い

動画生成モデル（MiniMax H3 一式）は **合計 約42GB**。差分ではなく最初から引くので、回線速度と空き容量との戦いになります。

- `diffusion_models` … 約20GB
- `text_encoders` … 約15GB
- `vae`（音声・動画）… 約5.5GB

当然これらは git に入らないので、**マニフェスト（一覧）だけリポジトリで管理** し、実ファイルはローカルに置く運用にしました。

## 期待7（おまけ）: 「設定リポジトリをGitHubにpushするだけ」

### 現実: アカウント名の"引っ越し"で混乱した

仕上げに設定リポジトリを push しようとしたら、`gh` の表示するアカウント名と、実際に認証されているアカウント名が食い違って見えました。

調べると、**ユーザー名をリネームしていた**（`haruurara19-bit` → `siren2345`）のが原因。GitHub は旧名のURLを新名へリダイレクトしてくれるので中身は同じアカウントなのですが、表示だけを見ると「別アカウントにpushしてしまった？」と一瞬ヒヤッとします。

:::message
**教訓**: 「表示されているアカウント名」と「トークンが実際に認証されているアカウント」は、`gh api user` で実測して確認する。
:::

---

# まとめ: 期待と現実のギャップ一覧

| # | 期待 | 現実 |
| --- | --- | --- |
| 1 | 32GBなら256K余裕 | denseは非対応・Ollamaが自動制限。MoEが必要 |
| 2 | pullすれば動く | 環境変数の宣言＋再起動が要る |
| 3 | `--model`で切替 | `settings.json`がフラグを上書き |
| 4 | ツールは連携する | 窓の仮定・プロトコル不一致・遅延ロード |
| 5 | 設定は一回で済む | 複数ツールに散らばりドリフト → TerraformでIaC |
| 6 | モデルはすぐ手に入る | 数十GBのダウンロードとの戦い |
| 7 | pushするだけ | アカウントリネームで表示と実態がズレる |

# 最終的な「現実的な」構成

紆余曲折を経て、今はこう落ち着きました。

1. **モデルは MoE**（`qwen3-coder:30b`）を選ぶ
2. **窓は環境変数で明示**（`OLLAMA_CONTEXT_LENGTH` / `CLAUDE_CODE_MAX_CONTEXT_TOKENS`）
3. **設定の優先順位を理解**（`settings.json` が最強）
4. **エージェントは opencode**（ローカルモデルとツール呼び出しの相性が良い）
5. **設定は Terraform で管理**（`configs/` を正にして `apply`）

ローカルLLMは「動けば最高」ですが、「動かすまで」と「動かし続ける」の両方に、クラウドAPIにはない地道な作業があります。この記事が、その現実を先に知る役に立てば幸いです。

---

- この記事は 2026-08 時点の構成です。モデル・ツールのバージョンは日々変わります。
- 登場する設定リポジトリは `local-ai-env`（Terraform + configs + ComfyUIマニフェスト）として管理しています。
