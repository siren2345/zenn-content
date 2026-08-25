---
title: "windowsという負債にまみれた不自由なコンピュータで生産的なことをしてはいけない"
emoji: "🪟"
type: "tech"
topics: ["windows", "powershell", "terraform", "opencode", "wsl"]
published: true
---

# はじめに

タイトルは叫びです。半分は本気で、半分は誇張です。

RTX 5090 を積んだ Windows マシンでローカル生成AI環境（Ollama / vLLM / LM Studio / opencode / Claude Code）を組んで、それを Terraform で IaC 化する、という個人プロジェクトをやっています。GPU は速いし、モデルも動く。**詰まるのは、いつもモデルではなく Windows のほうでした。**

しかもその詰まり方には共通の構造があります。

> **どこにも記録されない暗黙の状態が、ファイルシステム・レジストリ・プロセス環境変数に分散していて、壊れて初めて存在に気づく。**

この記事は、そういう「Windows という負債」を実際に踏み抜いた記録です。各章は独立して読めます。

| 章 | 負債の正体 |
| --- | --- |
| 1 | 文字コード（PowerShell 5.1 / UTF-16LE） |
| 2 | PATH（Machine / User / プロセス継承の三層） |
| 3 | アプリ実行エイリアス（APPEXECLINK） |
| 4 | パッケージ形式（MSIX と MSI の二重人格） |

---

# 第1章: 文字コードという負債

## 1-1. PowerShell 5.1 は BOM なし UTF-8 を読めない

日本語コメント入りの `.ps1` を書いて実行したら、文字化けして **パースエラー** になりました。

原因は、**Windows PowerShell 5.1（`powershell.exe`）が BOM なし UTF-8 のスクリプトファイルを、システムの ANSI コードページ（日本語環境なら CP932）として誤読する** ことです。

- エディタは UTF-8 で保存している
- 5.1 は CP932 だと思って読む
- 日本語コメントがバイト列として壊れ、運が悪いと構文まで壊れる

PowerShell 7+（`pwsh`）は UTF-8 をデフォルトで正しく扱うので、この問題は起きません。

そこで、リポジトリのルールとして **「`powershell.exe` は使わない、`pwsh` のみ」** を明文化し、Terraform 側でも強制することにしました。

```hcl
resource "null_resource" "set_env" {
  provisioner "local-exec" {
    interpreter = ["pwsh", "-NoProfile", "-File"]
    command     = "${path.module}/scripts/set-env.ps1"
  }
}
```

Windows Terminal の `defaultProfile` も、5.1 の「Windows PowerShell」ではなく 7+ の「PowerShell」を指すよう設定ファイルごと管理下に置きました。新しいタブが常に `pwsh` で開くようにするためです。

:::message
**教訓**: 「PowerShell」という名前のものが2つあり、片方は日本語UTF-8を壊す。名前が同じなので、どちらで動いたのかは自分で意識するしかない。
:::

## 1-2. `echo "..." > file` は UTF-16 を書く

もっと地味で、もっと厄介なのがこれです。PowerShell のリダイレクトは、既定で **UTF-16LE** でファイルを書きます。

Python スクリプトを生成しようとして、こうやったことがあります。

```powershell
echo "print(1)" > hoge.py
```

できあがったファイルを別のツールで開くと、こう見えます。

```
p r i n t ( 1 )
```

1文字おきにヌルバイトが挟まったバイト列です。当然 Python は実行できません。「ファイルは確かに作られている」「中身も一見正しい」のに動かない、という最悪の切り分けにくさがあります。

:::message
**教訓**: PowerShell からテキストファイルを生成するときは `Set-Content -Encoding utf8` を明示する。リダイレクト演算子は使わない。
:::

## 1-3. WSL のコマンド出力も UTF-16LE で化ける

`wsl --version` や `wsl -l -v` は **UTF-16LE で標準出力に書く** という癖があります。これを PowerShell のパイプラインに通すと化けます。

環境インベントリを自動生成するスクリプト（`terraform apply` のたびに `docs/TOOLS.md` を吐く）でこれを踏んだので、`ProcessStartInfo` の側で明示的に指定して回避しました。

```powershell
$psi.StandardOutputEncoding = [System.Text.Encoding]::Unicode
```

同じ「文字コード」でも、1-1 は **入力（スクリプトの読み込み）**、1-2 は **出力（ファイル書き込み）**、1-3 は **プロセス間（子プロセスの stdout）** と、三箇所すべてで別々に踏む必要があります。一度直しても他の二つは直りません。

---

# 第2章: PATH という負債

## 2-1. `bash` と打つと WSL2 に行っていた

Git for Windows を入れてあるので、`bash` は Git Bash だと思っていました。違いました。**WSL2 の Ubuntu が起動していました。**

永続 PATH の並びを実際に確認すると、こうなっていました。

| 順位 | パス | 中身 |
| --- | --- | --- |
| 16 | `...\AppData\Local\Microsoft\WindowsApps` | WSL のエイリアス `bash.exe` |
| 25 | `C:\Program Files\Git\usr\bin` | Git Bash の `bash.exe` |

WindowsApps が先に来るので、WSL 側が勝ちます。しかも Git インストーラが PATH に登録するのは `Git\cmd` **だけ** で、そこに `bash.exe` はありません（`sh.exe` はある）。

結果として、こうなります。

- `sh` → Git Bash
- `bash` → WSL2 / Ubuntu

**最も紛らわしい組み合わせ** です。「さっき `sh` で動いたスクリプトが `bash` だと動かない」という現象が起きて、初めて別物だと気づきました。

## 2-2. PATH は3層あり、それぞれ更新タイミングが違う

Windows の PATH は少なくとも3つの層に分かれています。

| 層 | 保存場所 | 反映タイミング |
| --- | --- | --- |
| Machine | `HKLM` | 新規プロセス |
| User | `HKCU` | 新規プロセス |
| プロセス | 親から継承 | **プロセス起動時に確定・以後不変** |

評価順は **Machine が先、User が後** です。そして最後の行が本命の罠です。

実際にこれで `terraform apply` が落ちました。新しく MSI 版 PowerShell を入れた直後、既に開いていたターミナルから apply したら、

```
exec: "pwsh": executable file not found in %PATH%
```

`pwsh` は確かにインストールされているのに、見つからない。理由は単純で、

- そのターミナルは **インストール前に起動した** ので、新しい `C:\Program Files\PowerShell\7` を PATH に持っていない
- 一方、それまで `pwsh` を解決していた WindowsApps のエイリアスは **削除済み**

つまり「新旧どちらの入口も無い」瞬間が生まれていたわけです。ターミナルを開き直せば直りますが、エラーメッセージからそこに辿り着くのには時間がかかりました。

:::message
**教訓**: Windows で「入れたのに見つからない」は、たいていインストールの失敗ではなく**プロセス環境が古い**だけ。まずターミナルを開き直す。
:::

---

# 第3章: アプリ実行エイリアスという負債

ここが個人的に一番「不自由」を感じた部分です。

## 3-1. 正体は APPEXECLINK 再解析ポイント

Windows の設定アプリに「アプリ実行エイリアス」というトグルがあります。`%LOCALAPPDATA%\Microsoft\WindowsApps\` の下に並んでいる `bash.exe` や `pwsh.exe` の実体は、**普通の実行ファイルでもショートカットでもなく、`APPEXECLINK` という再解析ポイント（reparse point）** です。

そして重要なのは、**その状態がレジストリではなく、ファイルシステム上のそのエントリの有無そのもの** だということです。HKCU を `reg query /f "bash.exe" /s` で総当たりしても、対応する設定値は出てきません。

これが何を意味するか。

- 誰かが設定アプリのトグルを押しても、**押したという記録がどこにも残らない**
- したがって「今どうなっているか」は、毎回ファイルシステムを見るしかない

## 3-2. 消せるが、戻せない

さらに厄介なことに、この再解析ポイントは **削除はできるが、再作成はできません**。作成には `FSCTL_SET_REPARSE_POINT` が必要で、通常のファイル操作 API では作れないからです。

戻すには、設定アプリのトグルを押し直すか、MSIX パッケージを再登録するしかありません。

**非対称な操作** です。IaC で扱うには最悪の性質で、うっかり「無効化リスト」に名前を書いた瞬間、コードからは元に戻せなくなります。

そのため、Terraform 側に「これだけは絶対に無効化させない」ガードを precondition で入れました。

```hcl
precondition {
  condition     = !contains(local.disabled_names, "pwsh.exe")
  error_message = "pwsh.exe を無効化することはできません。APPEXECLINK は Terraform では再作成できず、元に戻せません。"
}
```

## 3-3. Terraform の標準関数が両方とも使えない

エイリアスの「有無」を Terraform で表現しようとして、両方の素直な方法が潰れました。

| 関数 | 再解析ポイントに対する挙動 |
| --- | --- |
| `fileexists()` | `"... is not a regular file"` エラーで **plan ごと落ちる** |
| `fileset()` | 再解析ポイントを **一切列挙しない**（無いことにされる） |

片方は落ち、もう片方は嘘をつきます。仕方がないので `data "external"` から PowerShell の `Test-Path` を呼んで状態を読む形にしました。

```hcl
data "external" "alias_state" {
  program = ["pwsh", "-NoProfile", "-File", "${path.module}/scripts/alias-state.ps1"]
}
```

結果的にこれが plan 時点の drift 検知を兼ねることになり、エイリアスが復活すれば `terraform plan` に差分として現れるようになりました。**「押しても記録が残らないトグル」を、記録が残る形に持ち込めた** わけです。負債を返す唯一の方法は、たぶんこれしかありません。

## 3-4. `fs.stat()` が EACCES で落ち、opencode が死ぬ

そして本題です。opencode の設定に、使うシェルを明示しようとしました。

```jsonc
{
  "shell": "pwsh"
}
```

これだけで、**opencode の全プロンプトが失敗するようになりました。**

しかも TUI には何も出ません。エラーはログファイルにだけ残っていました。

```
prompt_async failed
Die(EACCES: permission denied, stat 'C:\...\WindowsApps\pwsh.exe')
```

Node.js の `fs.stat()` が、APPEXECLINK 再解析ポイントに対して **`EACCES: permission denied`** を投げるのです。しかも、

- `lstat()` は **通る**
- 実行権限チェック（`X_OK`）も **通る**
- 実際にそのパスで起動することも **できる**

「起動はするのに `stat` だけ落ちる」という、極めて切り分けにくい形で出ます。opencode は起動時に設定されたシェルを `stat` するので、そこで死んでいました。

:::message
**教訓**: Windows で「特定のパスに対してだけ」謎の EACCES が出たら、それが再解析ポイントかどうかを疑う。`Get-Item` の `LinkType` / `Attributes` を見れば分かる。
:::

---

# 第4章: パッケージ形式という負債

第3章の対策として、そもそも **PATH 上の入口がエイリアスしかない状態をやめる** ことにしました。

PowerShell 7 には2つの配布形式があります。

| 形式 | 実体 | PATH 上の入口 |
| --- | --- | --- |
| MSIX（Microsoft Store） | パッケージ内部 | WindowsApps のエイリアス **1個だけ** |
| MSI（WiX） | `C:\Program Files\PowerShell\7\pwsh.exe` | Machine PATH に実ディレクトリ |

同じ製品・同じバージョンなのに、**片方は `fs.stat()` で落ち、もう片方は落ちません**。入っていたのは MSIX 版でした。

## 4-1. winget が並列インストールを拒む

MSI 版を入れようとして、まず弾かれました。

```
EXITCODE=-1978335189
```

winget が既存の MSIX 版を「インストール済み」と見なし、新規インストールではなく **アップグレード扱い** にしてくるためです。最終的に通ったのはこの形。

```powershell
winget install --id Microsoft.PowerShell --source winget --installer-type wix --force
```

- `--installer-type wix` を **省くと MSIX 版が選ばれて元の木阿弥**
- `--force` は「既にある」判定を押し切るため

## 4-2. アンインストールも一意に指定できない

旧 MSIX を消そうとすると、今度は `winget uninstall` が **2つの実体にマッチして曖昧** になります。結局こちらで消しました。

```powershell
Remove-AppxPackage -Package Microsoft.PowerShell_7.6.5.0_x64__8wekyb3d8bbwe
```

バージョン番号込みのパッケージフルネームを、自分で調べて打つ必要があります。

## 4-3. そして設定には絶対パスを書く

入れ替えた後、opencode の設定はこうしました。

```jsonc
{
  "shell": "C:\\Program Files\\PowerShell\\7\\pwsh.exe"
}
```

**`"pwsh"` と書いてはいけません。** PATH 解決に任せると、環境によってはまたエイリアスに当たって同じ問題が再発します。MSI 版のパスは 7.x の更新でも変わらないので、絶対パスで固定するのが正解でした。

---

# まとめ: 負債の正体は「記録されない状態」

4章ぜんぶ、症状は違うのに構造は同じでした。

| 章 | 暗黙の状態 | どこに隠れているか |
| --- | --- | --- |
| 1 | どのシェルで実行されたか / どのエンコーディングで書かれたか | ファイルのバイト列そのもの |
| 2 | どの層の PATH がいつ評価されたか | レジストリ2箇所 + プロセス環境 |
| 3 | エイリアスが有効か | ファイルシステムのエントリの有無 |
| 4 | どの配布形式が入っているか | パッケージ DB |

どれも **「今どうなっているか」を宣言的に確認する手段が標準で用意されていない**。だから壊れて初めて存在に気づき、直した後も「なぜ直ったか」がどこにも残りません。

これが「負債にまみれている」と感じた理由です。技術的負債の定義そのもの — **暗黙の決定が、記録されないまま積み上がっている** 状態です。

## で、生産的なことをしてはいけないのか

タイトルへの答えを書いておきます。

**やっていいです。ただし「暗黙の状態を、明示的な状態に変換する」作業を、本題の前に払う覚悟がいる。**

このプロジェクトで実際にやったのは、結局その一点でした。

1. **使うシェルを固定する** — `pwsh` のみ。Terraform の `interpreter` にも書く
2. **エンコーディングを明示する** — リダイレクトを使わず `Set-Content -Encoding utf8`
3. **パスは絶対で書く** — PATH 解決に運命を託さない
4. **状態を Terraform に所有させる** — 設定アプリのトグルではなく `configs/` を正にする
5. **戻せない操作にはガードを置く** — Terraform の `precondition` で、戻せない操作そのものを禁止する

このうち 4 と 5 が効きました。`terraform plan` が「今の実態」と「あるべき姿」を突き合わせてくれるので、**エイリアスが復活すれば差分として見える**し、危険な操作は plan の時点でエラーになります。実行時の静かな混乱ではなく、plan 時点の明示的なエラーにする — これが Windows という OS を IaC で扱うときの、たぶん唯一の勝ち筋です。

不自由なのは変わりません。でも、不自由さが **コードに書いてある** なら、それはもう負債ではなく仕様です。

---

- この記事は 2026-08 時点の環境（Windows 11 / RTX 5090 / PowerShell 7.6 / Terraform）での記録です。
- 設定は `local-ai-env`（Terraform + configs）というリポジトリで管理しています。
- 同じプロジェクトのローカルLLM側の話は、別記事「ローカルLLMの期待と現実」に書いています。
