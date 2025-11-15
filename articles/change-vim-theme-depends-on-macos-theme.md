---
id: change-vim-theme-depends-on-macos-theme
aliases: []
tags: []
---



```sh
check_os_theme_is_dark() {
    osascript -e 'tell application "System Events" to tell appearance preferences to return dark mode' \
    | grep -qi true
}

vim() {
    if check_os_theme_is_dark; then
        nvim -c 'colorscheme kanagawa-dragon' "$@"
    else
        nvim -c 'colorscheme dayfox' "$@"
    fi
}
 Optional: make
```
