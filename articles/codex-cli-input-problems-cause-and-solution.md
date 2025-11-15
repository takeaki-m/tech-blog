---
title: "Codex CLIからのデスクトップ通知後にテキスト入力に問題発生"
emoji: "👾" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["codex", "ChatGPT"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

## 問題

Codex Cliを対話モードで起動して、Codex Cliから一度返信があった後に、テキスト入力がうまくいかなかった。

* 入力したテキストの、一部しか入力されない
  * e.g.) 日本語->本語
* 途中までしか入力されない
  * e.g.) 日本語を話す-> 日本語を


## 原因(おそらく)

* Code Cliでタスク完了時に、特定のシェルを実行するようにしていた。
* このシェルファイルが、実行できないタイミングがあった
  * Codex Cliからメッセージを呼び出せなかったとメッセージが返される

```toml
notify = ["bash", "/path/to/youra/home/.codex/notify.sh"]
```
* この読み込みに問題があると予想して、シェルファイルではなく、コマンドを直接実行するように変更した

```toml
notify = ["osascript", "-e", "display notification \"Task Complete!!!\" with title \"Codex\" subtitle \"Processing is complete.\" sound name \"Pop\""]

```

以後、不具合は発生している様子はない
