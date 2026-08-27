---
title: "AIエージェントがWindowsで詰まる理由:誰も語らない\"シェル境界\"問題"
emoji: "🐚"
type: "tech"
topics: ["claude-code", "windows", "wsl", "powershell", "agent"]
published: true
---

# AIエージェントがWindowsで詰まる理由:誰も語らない"シェル境界"問題

## はじめに

RTX 5090 を積んだ Windows マシンでローカル生成AI環境(vLLM / Ollama / LM Studio / opencode)を組んでいて、Claude Code も併用しています。Windows 側の地味な詰まりは前記事「[windowsという負債にまみれた不自由なコンピュータで生産的なことをしてはいけない](https://zenn.dev/siren2345/articles/windows-technical-debt)」で書いた。

あの記事の詰まりは**自分自身**が踏んだものだ。Terraform と PowerShell を書いていて踏んだ。この記事は違う。**自分ではないエージェントが踏んでいる詰まり**を、他人の傷跡から読み解く話にしよう。

きっかけは単純な疑問だった。

> AI エージェントが Windows でコマンドを回すとき、どこで詰まっているのか?

最初の推測は「モデルが Windows パスに弱いから」だった。Claude Code の changelog を掘ってみると、違った。**詰まっているのはモデルではなく、シェルだった。**

# changelog に 213 個の傷

Claude Code の changelog(約 5,900 行)を全部落としてきて、Windows / WSL 関連のエントリを全部抽出した。

**213 件。**

これは機能追加のリストではない。**壊れたもののリスト**だ。1 件ずつが、実運用で実際に起きたバグの傷跡。以下、章ごとに拾っていく。

| 章 | テーマ | 代表傷跡 |
| --- | --- | --- |
| 1 | 三つの shell の問題 | PowerShell 5.1 の `>` が UTF-16LE を書く |
| 2 | パスの地雷原 | `C:\Users\unicorn` が CJK に化ける |
| 3 | エンコード地獄 | CJK がクリップボードで化ける |
| 4 | WSL の境界 | OAuth の localhost コールバックが届かない |
| 5 | 自分のハーネスが学んだこと | 境界を越えるな |

各章は独立して読める。

---

# 第1章: 三つの shell の問題

## 1-1. Windows には 5 つの shell がある

前提整理。現代の Windows マシンには、エージェントが呼び出しうる shell が最低 5 つある。

| shell | 由来 | 状態 |
| --- | --- | --- |
| `cmd.exe` | OS 同梱 | 古参 |
| `powershell.exe`(5.1) | OS 同梱 | 古参だが**既定** |
| `pwsh.exe`(7+) | 自分で入れる | 現役 |
| `bash.exe`(Git for Windows) | Git 付属 | エージェントの事実上の標準 |
| `bash.exe`(WSL) | WSL 付属 | **別物** |

最後の 2 つが罠だ。`bash` と打つと Git Bash だと思っているが、**PATH 上では WSL の bash が勝つことがある**(前記事の第2章で踏んだ。WindowsApps のエイリアスが Git より先に来る)。エージェントが「bash」を回したとき、どちらの bash が回っているかなど、誰も教えてくれない。

## 1-2. 進化:「Git for Windows 必須」から「不要」へ

changelog の古い方(ネイティブ Windows 対応の初期)には、こうある。

> Added support for native Windows (requires Git for Windows)

初期の Windows 版 Claude Code は **Git Bash が必須**だった。Bash ツールは Git for Windows 経由で回す設計だった。

ところが 2026 年 4 月の週に、こうなる。

> Windows: Git for Windows (Git Bash) is no longer required — when absent, Claude Code uses PowerShell as the shell tool

2026 年 3 月に「PowerShell tool」が opt-in プレビューとして追加され、のちに既定になった。方向性は明確で、**Git Bash 依存からネイティブ PowerShell へ gradually 移行している**。

ただ、移行先が **OS 同梱の PowerShell 5.1** である限り、地獄は終わらない。

## 1-3. PowerShell 5.1 の 7 つの癖

5.1 は 2016 年のものだ。10 年経っても、まだ全 Windows に同梱されている既定の shell。エージェントはそれと付き合わなければならない。changelog に 5.1 固有のパッチが並ぶ。

| 癖 | changelog(要約) |
| --- | --- |
| `>` / `>>` が UTF-16LE で書く | `>` and `>>` under the PowerShell tool on Windows PowerShell 5.1 writing UTF-16LE files that other tools couldn't read as UTF-8 |
| 区切り文字が `;` | the cross-project resume hint failing in default Windows PowerShell 5.1 — Windows now uses `;` as the command separator |
| git の progress が stderr に出る | PowerShell tool incorrectly reporting failures when commands like `git push` wrote progress to stderr on Windows PowerShell 5.1 |
| 引数スプリット | external-command arguments containing both a double-quote and whitespace now prompt instead of auto-allowing (PS 5.1 argument-splitting hardening) |
| ExecutionPolicy | The PowerShell tool now passes `-ExecutionPolicy Bypass` |
| Group Policy が 5.1 をブロック | `/background` and `claude --bg` failing with "EUNKNOWN" when Group Policy blocks PowerShell 5.1; the daemon now prefers PowerShell 7 |
| permission チェックのバイパス | a permission-check bypass affecting commands run in Windows PowerShell 5.1 sessions |

UTF-16LE の件が最も地味で厄介だ。`echo "print(1)" > hoge.py` でできたファイルは、1 文字おきにヌルバイトが挟まったバイト列になる。「ファイルは確かに作られている」「中身も一見正しい」のに、Python は実行できない。(前記事の第1章でも踏んだ。同じ穴。)

:::message
**教訓**: Windows には「PowerShell」という名前のものが 2 つあって、片方は 2016 年製。エージェントは両方と付き合わされ、癖はバージョンごとに違う。「PowerShell の問題」では切り分け不能。5.1 か 7 かをまず確認する。
:::

## 1-4. Git Bash 側も安全じゃない

「じゃあ Git Bash で固定すればいい」となるが、それも違う。Git Bash 側にも傷が並ぶ。

- **`.bashrc` 回帰(2 回)**: "Fixed a regression where Windows users with a .bashrc file could not run bash commands" — `.bashrc` が有ると bash コマンドが**全部**動かなくなる。修正して、回帰して、また修正。
- **`2>nul` がリテラル `nul` ファイルを作る**: "Fixed literal nul file creation on Windows when the model uses CMD-style `2>nul` redirection in Git Bash" — モデルが Git Bash で CMD 風リダイレクトを書くと、`nul` という名前のファイルが**実際にできる**。
- **MSYS パスマングル**: "Fixed bash commands failing on Windows when temp directory paths contained characters like `t` or `n` that were misinterpreted as escape sequences" — temp パスに `t` や `n` が含まれると MSYS がエスケープと誤認して壊れる。**ユーザー名に `n` が含まれると壊れる**、という話。
- **Cygwin シンボリックリンクと revert**: "Reverted the 2.1.232 Bash permission changes for Cygwin-style symlinks on Windows and for input redirections (`< file`); a narrower version will return in a later release" — セキュリティ修正が通常利用を壊して**一旦 revert** されている。

最後が重要。**修正自体がバグになる**、という話だ。シェル境界問題は「直せば終わる」のではなく、直してもまた踏む泥沼だ。

---

# 第2章: パスの地雷原

## 2-1. `C:\Users\unicorn` が CJK に化ける

一番好きなエントリ。

> Fixed Windows paths with `\u`-prefixed segments (like `C:\Users\unicorn`) being corrupted into CJK characters in tool inputs, which made those files inaccessible

ユーザー名が `unicorn` だと、パス `C:\Users\unicorn` に `\u` が含まれる。スタックのどこかで `\u` が **Unicode エスケープ**(`\uXXXX`)として解釈され、パスが CJK 文字列に化ける。結果、そのファイルに**一切アクセスできなくなる**。

ユーザー名でエージェントが壊れる。しかもユーザー名は変えられない。

## 2-2. `\??\` と NTLM クレデンシャル漏洩

もっと怖いものもある。

> Security: remote file reads, session restore, CLAUDE.md includes, workflow scripts and file uploads now reject Windows NT-namespace (`\??\`) paths, hardening the remaining pre-approval file accesses against the NTLM credential-leak vector

Windows パスには `\??\C:\...` という「デバイス名前空間」の書き方がある。これを使うと UNC パス検証をバイパスでき、**NTLM クレデンシャル漏洩のベクトル**になる。機能バグではなく**セキュリティ問題**。エージェントのファイルアクセスそのものが攻撃面になる。

## 2-3. ドライブ文字の大小

> Fixed the same CLAUDE.md file being loaded twice when drive letter casing differs between paths

`C:\` と `c:\` は同じドライブ。だがパス文字列の大小が違えば、エージェントは別物と扱う。同じファイルが 2 回読み込まれる。「大文字小文字を区別しない OS、文字列としては区別する」という隙間には、同種のバグが一族でいる。

## 2-4. OneDrive・ネットワークドライブ・NTFS junction

- **OneDrive**: "Fixed agent creation failing with `EEXIST: file already exists` when the agents directory already exists (Windows/OneDrive)" — ホームディレクトリが OneDrive で同期されていると、ファイル作成が EEXIST で落ちる。
- **ネットワークドライブ**: "Fixed `claude agents` deadlocking on Windows with network-drive working directories" — 作業ディレクトリがネットワークドライブだと**デッドロック**。
- **NTFS junction**(一番怖い): "Fixed Windows worktree removal deleting files outside the worktree when an NTFS junction or directory symlink existed inside it" — worktree の中に junction があると、**worktree 削除が junction を追い、メインリポジトリのファイルを消す**。

:::message
**教訓**: Windows でパスは文字列ではない。名前空間(NT/UNC)・大小(C:/c:)・ファイルシステム(NTFS/9p/OneDrive)・再解析ポイント(junction/symlink)の 4 重構造で、OS はそれを教えてくれない。エージェントは全部を推論しなければならない。
:::

---

# 第3章: エンコード地獄

第1章の 5.1 UTF-16LE は「書き出し」の話。この章は「表示・コピー」側の話。

## 3-1. CJK が化ける

- "Fixed Japanese/Korean/Chinese text rendering as garbled characters on Windows in no-flicker mode"
- "Fixed Korean/Japanese/Unicode text becoming garbled when copied in no-flicker mode on Windows"
- "Fixed stale and doubled rows in the agent view list on Windows when background session results contain wide (CJK) characters"

CJK は全角(2 カラム)で、端末の折り返し計算が壊れる。「日本語が化ける」だけでなく、**UI 自体が壊れる**。

## 3-2. CRLF

- "Fixed Edit/Write tools doubling CRLF on Windows and stripping Markdown hard line breaks" — ファイル編集ツールが **CRLF を二重化**する。
- "Fixed pasting CRLF content (Windows clipboards, Xcode console) inserting an extra blank line between every line" — Windows クリップボードから貼ると**行間に空行が入る**。

## 3-3. クリップボードは地雷原

- "Fixed clipboard corrupting non-ASCII text (CJK, emoji) on Windows/WSL by using PowerShell `Set-Clipboard`" — CJK/emoji がクリップボードで破損するので `Set-Clipboard` に切り替えた。
- "Windows: clipboard writes no longer expose copied content in process command-line arguments visible to EDR/SIEM telemetry; also fixes >22KB selections not reaching the clipboard" — **クリップボードの内容がプロセスのコマンドライン引数として可視**で、EDR/SIEM が拾っていた。セキュリティ問題。

クリップボードは最も文字通りな「境界」だ。Windows とエージェントがバイトをやり取りする場所で、エンコード・サイズ上限・テレメトリが全部ぶつかる。

---

# 第4章: WSL の境界

## 4-1. wsl.exe の呼び出し方問題

ここで自分の実験を挟む。Windows から WSL 内のコマンドを叩くとき、

```
wsl ls                        # 単一コマンド
wsl -e bash -c "ls | grep foo"  # 複合式
```

罠は、**シェル演算子が境界を越えない**こと。

```
wsl cd /tmp && ls              # && は Windows 側で解釈される。cd は WSL の一時インスタンスで実行されて消える
wsl -e bash -c "cd /tmp && ls" # 複合式が WSL 側に 1 文字列として渡る
```

`wsl <cmd>` は単一コマンドしか渡さない。`|`・`&&`・`;`・変数は全部 **Windows 側シェルが解釈**する。複合式を回すには `-e bash -c "..."` で包むしかない。これが「シェル境界」問題の最も基本的な形だ。エージェントは、各演算子が境界のどちら側で解釈されるかを知らなければならない。

## 4-2. WSL interop の傷

- **画像ペースト**: "WSL2: image paste from Windows clipboard now works via a PowerShell fallback when xclip/wl-paste cannot read image data" + "Fixed image pasting not working on WSL2 systems where Windows copies images as BMP format" — Windows から WSL へ画像を貼くのが、xclip / wl-clipboard / BMP / PowerShell フォールバックの泥沼。
- **OAuth**: "claude auth login now accepts the OAuth code pasted into the terminal when the browser callback can't reach localhost (WSL2, SSH, containers)" — WSL2 の localhost forwarding が頼りなく、ブラウザのコールバックが届かない。だから「コードを貼り付ける」経路を追加した。
- **interop 無効**: "Fixed unhandled promise rejections when a subprocess fails to start, for example `powershell.exe` on WSL with Windows interop disabled" — WSL の Windows interop を切ると `powershell.exe` が起動できずクラッシュ。
- **検索が不完全になる**: 公式 troubleshooting に "Disk read performance penalties when working across file systems on WSL may result in fewer-than-expected matches" とある。9p ファイルシステム越しの検索は、**結果が静かに不完全になる**。

## 4-3. 設計の答え:「境界を越えるな」

Claude Code Desktop に「WSL session」モードがある。docs はこう言う。

> The session's Claude Code process, its tools, and git all execute inside the distribution, using its Linux toolchain and native Linux paths... Working on those files from Windows goes through a network filesystem, which is slow and breaks file watching; running the session inside the distribution avoids both.

要するに、**リポジトリが WSL 側にあるなら、エージェントも WSL 側で回せ**。境界を越えるな。作業が起きる側にエージェントを置け、という設計論だ。「越えるのを効率化する」のではなく「越えない」のが答え。

---

# 第5章: 自分のハーネスが学んだこと

## 5-1. 自分の構成

- opencode(Windows 側、Git Bash ハーネス)
- vLLM(WSL2 側、ポート 8889)
- WSL は**サービス**であり、作業空間ではない

自分の構成では、エージェント(opencode)は Windows 側にいて、WSL は vLLM が聞いている場所だ。だから正解は「境界を越えるな」の別バージョンで、**サービスとして扱う**:

```
curl http://localhost:8889/v1/...   # WSL2 の localhost forwarding で到達
```

サービスアクセスに `wsl` を使わない。`wsl` は WSL 内のファイル・デーモン操作のときだけ、と割り切る。Claude Code Desktop の「WSL session」と同じ原理で、**作業が起きる側にエージェントを置き、向こう側はサービスとして扱う**。

## 5-2. Git Bash を選んだ理由

自分のハーネスの shell は Git Bash で、PowerShell ではない。理由は、PowerShell 7 の日本語出力が cp932 で誤デコードして化けるから(第3章の入力側エンコード問題を、shell を変えて回避した形)。(前記事の第1章と地続き。)

Claude Code のアプローチは真逆で、**5.1 の癖を全部パッチして Windows ネイティブで動かす**方向に潰している。一方は shell を変えて回避、もう一方は shell のまま直撃。どちらも正解で、コストを払う場所が違うだけ。

## 5-3. 共通の教訓

両方とも、結局同じ 4 点に収まる。

1. **shell を固定する** — PATH 解決に運命を託さない
2. **パスを固定する** — 大小・名前空間・OneDrive・junction に運命を託さない
3. **境界を越えない** — 作業が起きる側にエージェントを置く
4. **暗黙の状態を明示にする** — 考えなければならないことは、書いておく

(前記事の結論と一致する。シェル境界問題と「Windows という負債」は、表と裏の同じ硬貨だ。)

---

# まとめ: シェル境界問題は「壊れて初めて見える」

changelog の 213 件。そのうち「機能追加」はほぼなく、ほとんどが「壊れたものを直した」だ。

それがシェル境界問題の性質だ。**壊れて初めて見える**。詰まっているのはモデルではなく、シェル。しかもシェルは、エージェントと OS の間の最も薄く、最も見えない層。

Windows でエージェントを回しているなら、4 つだけ確認してほしい。

1. **どの shell で回しているか**(Git Bash? 5.1? 7? WSL の bash?)
2. **パスは固定しているか**(大小・名前空間・OneDrive・junction)
3. **境界を越えていないか**(WSL? 9p? クリップボード?)
4. **暗黙の状態は書いてあるか**(考えなければならないことは、書いておく)

シェル境界問題は「誰も語らない」から、見えない。だが changelog の傷跡を見れば、そこにある。213 個。

---

- この記事は Claude Code の changelog(2026-08 時点)と、自分の環境(Windows 11 / RTX 5090 / WSL2 / opencode)での検証に基づいています。
- Windows 側の Terraform / PowerShell 視点での詰まりは、別記事「windowsという負債にまみれた不自由なコンピュータで生産的なことをしてはいけない」に書いています。
- 第4章の wsl.exe 呼び出しパターンは実機(WSL2 / Ubuntu)で検証済みです。
