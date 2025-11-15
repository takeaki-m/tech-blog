---
title: "nvimのテーマをMacOSの設定と同期する"
emoji: "🎨" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["nvim", ] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---


## 背景

nvimではOS のcolorschemeと連携して、特定のthemeを変更する設定がないようです。(もし設定がありましたら教えてください) 
colorschemeだけ見ると、colorscheme内でOSの設定に合わせてcolorschemeを切り替えるものがあります。(kanagawaなど)

個人的に気分によりMacOSのthemeを切り替えているため、連動してnvimで利用するテーマも切り替えたいと思いました。

## 実現方法

綺麗な方法でありませんが、zshの関数設定でOSのthemeを判定し、
テーマごとにnvimの起動時にカラースキームを指定する方式としました。

以下をzshに定義すれば利用できます。

毎回nvim起動時にしか適用されないため、起動中のnvimのプロジェクトには反映できません


```sh
check_os_theme_is_dark() {
    osascript -e 'tell application "System Events" to tell appearance preferences to return dark mode' \
    | grep -qi true
}

nvim() {
    if check_os_theme_is_dark; then
        nvim -c 'colorscheme kanagawa-dragon' "$@"
    else
        nvim -c 'colorscheme dayfox' "$@"
    fi
}
```


