## 結合 Raycast

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
