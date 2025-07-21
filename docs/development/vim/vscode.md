## 進入 normal mode 時，自動切換英文輸入法

安裝 [im-select](https://github.com/daipeihust/im-select)

```shell
brew tap daipeihust/tap
brew install im-select
```

進到設定畫面

- Vim > Auto Switch Input Method: Default IM `com.apple.keylayout.ABC`
- Vim > Auth Switch Input Method: Enable `checked`
- Vim > Auth Switch Input Method: Obtain IMCmd `/opt/homebrew/bin/im-select`
- Vim > Auth Switch Input Method: Switch IMCms `/opt/homebrew/bin/im-select {im}`

## 動畫

```json
"editor.cursorSurroundingLines": 15,
"editor.cursorStyle": "underline",
"editor.cursorSmoothCaretAnimation": true,
"editor.cursorBlinking": "phase"
```
