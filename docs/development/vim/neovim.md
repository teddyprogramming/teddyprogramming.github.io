## noevide

通常 terminal 會吃掉部分的 key，例如 ++cmd++，使用 neovide。

### 快速動作

Automator > 快速動作

```automator
on run {input, parameters}
    if input is {} then
        dialog "請選擇一個資料夾" buttons {"OK"} default button "OK"
        return
    end if

    set folderPath to quoted form of POSIX path of (item 1 of input)
    do shell script "'/Applications/Neovide.app/Contents/MacOS/neovide' " & folderPath
    return input
end run
```

檔案會放在 ~/Library/Services/

## 設定

### Common

```lua title="~/.config/nvim/lua/config/keymaps.lua"
vim.keymap.set("i", "jj", "<Esc>") -- jj enters normal mode from insert mode
vim.keymap.set("n", "<D-/>", "gcc+", { remap = true }) -- cmd+/ comment single line
vim.keymap.set("v", "<D-/>", "gc", { remap = true }) -- cmd+/ comment selected lines

vim.keymap.set("n", "<S-CR>", "o<Esc>") -- shift+enter insert a line after current line
vim.keymap.set("n", "<D-A-CR>", "O<Esc>") --cmd+option+enter insert a line before current line
vim.keymap.set("n", "<D-CR>", "i<CR><Esc>^") --cmd+option line break
```

### Neovide

Copy/Paste using ++cmd+v++, ++cmd+r++

```lua title="~/.config/nvim/lua/config/keymaps.lua"
if vim.g.neovide then
  vim.keymap.set("n", "<D-s>", ":w<CR>") -- Save
  vim.keymap.set("v", "<D-c>", '"+y') -- Copy
  vim.keymap.set("n", "<D-v>", '"+P') -- Paste normal mode
  vim.keymap.set("v", "<D-v>", '"+P') -- Paste visual mode
  vim.keymap.set("c", "<D-v>", "<C-R>+") -- Paste command mode
  vim.keymap.set("i", "<D-v>", '<ESC>l"+Pli') -- Paste insert mode
end

-- Allow clipboard copy paste in neovim
vim.api.nvim_set_keymap("", "<D-v>", "+p<CR>", { noremap = true, silent = true })
vim.api.nvim_set_keymap("!", "<D-v>", "<C-R>+", { noremap = true, silent = true })
vim.api.nvim_set_keymap("t", "<D-v>", "<C-R>+", { noremap = true, silent = true })
vim.api.nvim_set_keymap("v", "<D-v>", "<C-R>+", { noremap = true, silent = true })
```

### Replace with Register

```lua title="~/.config/nvim/lua/plugins/replace-with-register.lua"
vim.keymap.del("n", "grn")
vim.keymap.del("n", "gra")
vim.keymap.del("n", "grr")
vim.keymap.del("n", "gri")

return {
      { "inkarkat/vim-ReplaceWithRegister" },
      {
        "neovim/nvim-lspconfig",
        keys = {
          { "gr", false },
        },
      },
}
```

### surround

```lua title="~/.config/nvim/lua/plugins/surround.lua"
return {
      { "tpope/vim-surround" },
}
```

### 從 insert mode 進入 normal mode 時，自動將輸入法切換成英文

```lua title="~/.config/nvim/lua/config/autocmds.lua"
vim.api.nvim_create_autocmd("InsertLeave", {
  pattern = "*",
  callback = function()
    os.execute("im-select com.apple.keylayout.ABC")
  end,
})
```

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

markdown 預設在 <http://xxx> 的前後加上 < .. >，使用以下設定關閉

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
