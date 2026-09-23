---
title: "hermes agentはアップデートが多すぎるので定期的なメンテナンスをLLMにやらせるべき"
emoji: "🔧"
type: "tech"
topics: ["hermes", "llm", "agent", "maintenance", "browser-use"]
published: true
---

# はじめに

Hermes Agent を触っているときに思ったことがある。

「このプロジェクト、アップデートが速すぎる。LLMに定期的なメンテナンスをやらせるべきではないか」

昨日まで動いていたブラウザ自動化が、今日動かない。スキルが壊れる。ヘルパー関数が消える。CLIが別バージョンになる。

2026年9月23日の今日、実際にそれが起きた。browser-use が 0.1.9 から 0.13.10 へアップデートされ、これまで動いていたスキルが全部タイムアウトした。

Hermes Agent のリポジトリのコミット数は **40,156**。月約2,000件のペースで更新される。これほどの速度でプロジェクトが進むとき、LLMが「古いコードと新しい仕様の齟齬」をメンテナンスする役割を担うのが自然だと思う。

# 何が起きたか

9月23日朝、Hermes Agent の browser-use パッケージが 0.1.9 → 0.13.10 へアップデートされた。

これは破壊的変更だった。CLIバイナリ名は `browser-use.exe` のままだったが、内部実装が `browser_harness` パッケージに書き換わっていた。ヘルパー関数名（`new_tab()`, `page_info()`, `js()` 等）は旧APIと互換を保っていたが、バイナリのサイズが 46KB から 108KB へ倍増し、内部の接続方式も大きく変わっていた。

問題は2つあった。

1. `$HERMES_HOME/bin/browser-use.exe` が旧 0.1.9 のまま更新されていなかった
2. Hermes内部の`browser_exec`ツールが real_profile + Chrome起動中の組み合わせでCDP解決にブロックされ、420秒タイムアウトする

3スキル（`browser-exec`, `x-autopost-browser`, `x-browser-posting`）を更新し、`browser-use.exe` を venv版の 0.1.13 に上書きし、daemonを再起動して解決した。

全部で1時間弱。

# なぜLLMのメンテナンスが必要か

プロジェクトが月2,000コミットで更新される環境では、従来のメンテナンス方法では追いつかない。

## 従来アプローチの問題

| 方法 | 問題点 |
|---|---|
| 人間が目視で確認 | コミット数が多すぎて追えない |
| Changelogをスキャン | 破壊的変更が自然言語に埋もれる |
| テストが落ちたら直す | ユーザーが被害を受ける |
| ユーザーから報告を待つ | 信頼性の低下 |

LLMがこれをやるとしたら:

- 新コミットの diff を読み、破壊的変更を抽出
- 既存のスキル・ドキュメントと照合して「壊れている箇所」を特定
- 自動修正するか、人間に報告
- 定期的なサイクルで実行

# 実際の作業: 6つのステップ

起きた作業を振り返る。

## ステップ1: 現状把握

まず何が変わったかを確認する。

```bash
# バージョン確認
"C:\Users\ru628\AppData\Local\hermes\bin\browser-use.exe" --version
# 0.1.9 → まだ古い

# venvのバージョン
pip show browser-use
# Version: 0.13.10 ← 違う

# daemonの死活
"C:\Users\ru628\AppData\Local\hermes\bin\browser-use.exe" --doctor
# [FAIL] daemon alive — 繋がらない
```

## ステップ2: バイナリ更新

venvの 0.13.13 バイナリに上書き。

```bash
cp venv/Scripts/browser-use.exe $HERMES_HOME/bin/
# 46KB → 108KB 更新完了
```

## ステップ3: daemon再起動

```bash
"C:\Users\ru628\AppData\Local\hermes\bin\browser-use.exe" --reload
# daemon stopped — will restart fresh on next call
```

次回CLI実行でdaemonが自動再起動。

## ステップ4: 新ヘルパー動作確認

```bash
"C:\Users\ru628\AppData\Local\hermes\bin\browser-use.exe" <<'PY' 2>/dev/null
from browser_harness.run import new_tab, page_info, js, ensure_real_tab
print("ALL IMPORTS OK")
ensure_real_tab()
new_tab("about:blank")
wait_for_load()
print("TITLE:", js("document.title"))
PY
```

新 helpers は正常に動いた。旧API名は互換。

## ステップ5: スキル更新

3スキルを更新:

| スキル | 変更内容 |
|---|---|
| `browser-exec` | 0.13.x API一覧、daemon死活管理、browser_execタイムアウト対策、terminal heredoc標準化 |
| `x-autopost-browser` | 0.13.x API準拠、X投稿フローを整理 |
| `x-browser-posting` | x-autopost-browserへの参照エイリアスに（重複解消） |

## ステップ6: memory更新

```
2026-09-23: browser-use 0.1.9→0.13.10アップデート完了。
$HERMES_HOME/bin/browser-use.exe を venv版に上書き。
Hermes内部のbrowser_execは420秒タイムアウト→heredoc方式を標準に。
```

# なぜこれが「LLMにやらせるべき」なのか

3つ理由がある。

## 1. スキル間の依存関係が複雑になっている

Hermesのスキルは相互に依存している。`browser-exec` を変えると `x-autopost-browser` も `myfans-affiliate-scraping` も影響を受ける。人間が手動で全部追うのは現実的ではない。

月2,000コミット → 1日に約60コミット。そのうち破壊的変更は数件だが、それらがどのスキルに影響するかを人間が目視で追うのは不可能に近い。

## 2. 「動いていても古い」が放置される

今回の件でいうと、`browser-use.exe` は venvで 0.13.10 に更新されていても、`$HERMES_HOME/bin/browser-use.exe` は 0.1.9 のままだった。venvは最新、managed copyは古い。Hermes内部は managed copy を呼ぶので、venvの更新が反映されない。

これは「動いて見えて、実は壊れている」ケース。ユーザーが報告するまで発見できない。

LLMが定期的なスキャンで `pip list` → `binary --version` → diff を取るだけで、この状態は検出できる。

## 3. ドキュメントの陳腐化が自動検出できる

スキルファイルは `SKILL.md`。この中に書かれたコマンドが実際に動くか確認するには、LLMにスクリプトを書かせて terminal で実行するだけでいい。

```python
# スキルのコマンドを実行して結果を確認
terminal(command="browser-use --doctor 2>/dev/null")
# → 結果をSKILL.mdと照合して差異を報告
```

# 実現のための設計

LLMが定期的なメンテナンスを行うためには、3つの要素が必要。

## cron定期実行

```yaml
# cron job: 毎週月曜 09:00
- command: "bash ~/scripts/hermes-maintenance-check.sh"
  schedule: "0 9 * * 1"
  no_agent: true
  deliver: "bot-chat:default"
```

`hermes-maintenance-check.sh` は以下をやる:

1. `pip list` で主要パッケージの最新バージョンを照合
2. `$HERMES_HOME/bin/` のバイナリが最新かチェック
3. `browser-use --doctor` でdaemon死活確認
4. スキル内のコマンドを実行して結果確認
5. 差異を出力

## 破壊的変更の事前検出

Hermes Agent のリリースノートやコミットログを監視し、破壊的変更が検出されたら:

1. 影響を受けるスキルを列挙
2. スキル内の関連コマンドをテスト
3. 変更点をdiffで提示

## 自動修正 vs 報告

全部を自動修正するのは危険。人間が承認するパターンが良い:

| 変更内容 | 自動修正 | 報告のみ |
|---|---|---|
| バイナリ更新 | ✅ | |
| ヘルパーAPIの互換性維持 | ✅ | |
| スキルのコマンド更新 | ✅ | |
| 新機能のドキュメント追加 | | ✅ |
| 依存パッケージの変更 | | ✅ |

# おわりに

Hermes Agent のアップデート速度（月2,000コミット）は、LLMがメンテナンスを補助するのに適した規模だと思う。

- 1日に数件の破壊的変更
- スキル間の依存関係が複雑
- 「動いて見えて壊れている」ケースが頻発

LLMは diff を読み、コマンドを実行し、差異を報告する。人間は判断する。この分担なら、プロジェクトが速く進んでも追いつける。

今回1時間で済んだ作業も、週1回のメンテナンスが自動で回っていれば「発見→修正」のサイクルが短縮される。ユーザーが被害を受ける前に検出できる。

Hermes Agent はすでに `cronjob` ツールを持っている。定期実行は可能だ。LLMにメンテナンスをやらせる仕組みは、今すぐ始められる。
