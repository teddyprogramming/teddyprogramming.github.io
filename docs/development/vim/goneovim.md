## 結合 Raycast

Raycast script 腳本

1. cd 到目標目錄
2. 如果目前 goneovim 的 title 有目標目錄的檔案名稱，則將程式帶到前景

    - neovim 設定可能需要設定 title

        ```
        vim.opt.title = true
        vim.opt.titlestring = vim.fs.basename(vim.fn.getcwd())
        ```

```shell
#!/bin/bash

# Required parameters:
# @raycast.schemaVersion 1
# @raycast.title Open opersonal note
# @raycast.mode silent

# Optional parameters:
# @raycast.icon /Applications/goneovim.app/Contents/Resources/goneovim.icns
# @raycast.packageName Goneovim

cd ~/LIbrary/CloudStorage/Box-Box/HOME-teddy.cc.lee/note/personal

TARGET_DIR="$(pwd)"
TARGET_NAME="$(basename "$TARGET_DIR")"

FOUND=$(osascript -e "set targetName to \"$TARGET_NAME\"" -e 'tell application "System Events"
  set matchedWindows to {}
  log targetName
  repeat with proc in (every process whose name is "goneovim")
    repeat with win in windows of proc
      if title of win = targetName then
        set frontmost of proc to true
        return "FOUND"
      end if
    end repeat
  end repeat
  set cmd to "/Applications/goneovim.app/Contents/MacOS/goneovim ."
  do shell script cmd
  return "NOT_FOUND"
end tell')

echo $FOUND
```
