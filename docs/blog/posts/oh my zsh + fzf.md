---
date: 2025-04-27
---

# Oh my zsh + fzf

## 安裝 fzf

```shell
$ brew install fzf
```

編輯 `~/.zshrc` 找到 `plugins` 加上 `fzf`

```zshrc
plugins=(... fzf)
```

## 搜尋

在命令列打指令

```shell
$ fzf
```

或使用 ++ctrl+t++

語法:

- `sbtrkt`: (fuzzy-match) Items that match `sbtrkt`
- `'wild`: (exact-match) Items that include `wild`
- `'wild'`: (exact-boundary-match) Items that include `wild` at word boundaries
- `^music`: (prefix-exact-match) Items that start with `music`
- `.mp3$`: (suffix-exact-match) Items that end with `.mp3`
- `!fire`: (inverse-exact-match) Items that do not include `fire`
- `!^music`: (inverse-prefix-exact-match) Items that do not start with `music`
- `!.mp3`: (inverse-suffix-exact-match) Items that do not end with `.mp3`

多個搜尋條件使用 空白 作為間隔，邏輯是 AND 效果。如果要 OR 效果，使用 `|` 串接。

`^core go$ | rb$ | py$`

```shell
$ vim **<TAB>

$ vim ../fzf**<TAB>
```

## 多選模式

`$ fzf -m`

## 搜尋歷史命令

++ctrl+r++
