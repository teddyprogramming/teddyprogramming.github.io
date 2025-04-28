## 基本操作

基本的 vim 操作就不在這裡列舉。

### 在 Window 間移動

- 左: ++ctrl+h++
- 下: ++ctrl+j++
- 上: ++ctrl+k++
- 右: ++ctrl+l++

分割 window

- 分割右邊: ++"<leader>|"++
- 分割下面: ++"<leader>-"++
- 關閉 window: ++"<leader>wq"++

### buffer 操作

在 buffer 間移動:

- 左: ++"H"++
- 右: ++"L"++

關閉 buffer: ++"<leader>bd"++

### comment

- 在上方插入註解: ++"gcO"++
- 在下方插入註解: ++"gco"++

### lazyvim

- Lazy config: ++"<leader>l"++
- Quit All: ++"<leader>qq"++

## Config

`~/.config/nvim/lua/config/` 下的檔案。

### autocmds.lua

從 insert mode 進入 normal mode 時，自動將輸入法切換成英文。

```lua
vim.api.nvim_create_autocmd("InsertLeave", {
  pattern = "*",
  callback = function()
    os.execute("im-select com.apple.keylayout.ABC")
  end,
})
```

### keymaps.lua

按 ++"jj"++ 從 insert mode 進入 normal mode。

```lua
vim.keymap.set("i", "jj", "<Esc>")
```

在 normal mode 按 ++tab++ 縮排

```lua
vim.keymap.set("n", "<Tab>", ">>")
vim.keymap.set("n", "<S-Tab>", "<<")
```

### options.lua

預設的設定，markdown 會隱藏 code block 的 ``` 符號，會造成編輯上的混亂。以下設定讓 markdown 的內容顯示不會有東西被隱藏。

```lua
vim.opt.conceallevel = 0
```

## Plugins

### flash.lua

因為 ++s++ 預設的 vim 動作會被這個 plugin 覆蓋成 flash jump。所以以下設定將 ++s++ 動作還原成 vim 預設的刪除目前字元並進入 insert 模式，然後將 flash jump 動作換成 ++\\++。

新增檔案: `~/.config/nvim/lua/plugins/flash.lua`

```lua
return {
  "folk/flash.nvim",
  keys = function(_, keys)
    keys = vim.tbl_filter(function(keymap)
      return keymap[1] ~= "s"
    end, keys)

    table.insert(keys, {
      "\\",
      mode = { "n", "x", "o" },
      function()
        require("flash").jump()
      end,
      desc = "Flash",
    })

    return keys
  end,
}
```

### Replace with Register

```lua
return {
  {
    "neovim/nvim-lspconfig",
    opts = function()
      local keys = require("lazyvim.plugins.lsp.keymaps").get()
      keys[#keys + 1] = { "gr", false }
    end,
  },
  { "echasnovski/mini.operators", keys = { { "gr", desc = "Replace operator" } }, opts = {} },
}
```

### mini.surround

遵循 ideavim, vscode vim 的熱鍵。

```lua
return {
  "echasnovski/mini.surround",
  opts = {
    mappings = {
      add = "ys",
      delete = "ds",
      replace = "cs",
    },
  },
}
```

### markdownlint-cli2

markdown 預設在 http://xxx 的前後加上 < .. >，使用以下設定關閉

```lua title="~/.config/nvim/lua/plugins/markdown.lua"
local HOME = os.getenv("HOME")
return {
  "mfussenegger/nvim-lint",
  optional = true,
  opts = {
    linters = {
      ["markdownlint-cli2"] = {
        args = { "--config", HOME .. "/.config/nvim/lua/plugins/markdownlint-cli2.yaml", "--" },
      },
    },
  },
}
```

```yaml title="~/.config/nvim/lua/plugins/markdownlint-cli2.yaml"
config:
  MD054:
    autolink: false
```

其他設定參考: [Rules](https://github.com/DavidAnson/markdownlint/blob/main/doc/Rules.md)

