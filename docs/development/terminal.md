
``` title="~/.zshrc"
autoload -Uz edit-command-line
zle -N edit-command-line
bindkey '^X^E' edit-command-line
export EDITOR=nvim
```

按 ++ctrl+x+ctrl+e++ 把當前的指令帶到 vim 模式。
