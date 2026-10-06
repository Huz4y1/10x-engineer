---
tags: [nvim, config, setup, lua]
---

# Configuration

Setting up Neovim from nothing. Defaults: [[Default Nvim]] · Your binds: [[Keybinds]] · Hub: [[Nvim]]

---

## 1. Install

### Windows

```powershell
winget install Neovim.Neovim
```

Config lives at `~\AppData\Local\nvim\init.lua`
(that's `C:\Users\<you>\AppData\Local\nvim\`)

### WSL / Ubuntu

```bash
# the apt version is usually ANCIENT - use the PPA or the appimage
sudo add-apt-repository ppa:neovim-ppa/unstable
sudo apt update && sudo apt install neovim

# or the appimage (always current)
curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim-linux-x86_64.appimage
chmod u+x nvim-linux-x86_64.appimage
sudo mv nvim-linux-x86_64.appimage /usr/local/bin/nvim
```

Config lives at `~/.config/nvim/init.lua`

> ⚠️ **Windows and WSL have separate configs.** They are different machines as far as Neovim is concerned. Editing one does nothing to the other — a genuinely confusing hour the first time.

```vim
:echo stdpath('config')      " where IS my config?
```

### Dependencies worth having

```bash
sudo apt install git curl ripgrep fd-find unzip build-essential   # WSL
winget install BurntSushi.ripgrep.MSVC sharkdp.fd                 # Windows
```

`ripgrep` and `fd` make fuzzy-finding fast. Without them Telescope crawls.

## 2. The minimum init.lua

```lua
-- ~/.config/nvim/init.lua

-- leader MUST be set before plugins load
vim.g.mapleader = " "
vim.g.maplocalleader = " "

local o = vim.opt
o.number = true             -- line numbers
o.relativenumber = true     -- relative - makes 5j / 3dd natural
o.expandtab = true          -- spaces not tabs
o.shiftwidth = 4
o.tabstop = 4
o.smartindent = true
o.wrap = false
o.ignorecase = true         -- case-insensitive search...
o.smartcase = true          -- ...unless you type a capital
o.hlsearch = false
o.incsearch = true
o.scrolloff = 8             -- keep 8 lines visible around the cursor
o.signcolumn = "yes"        -- stops the text jumping when diagnostics appear
o.termguicolors = true
o.undofile = true           -- PERSISTENT undo across sessions
o.updatetime = 250
o.clipboard = "unnamedplus" -- use the system clipboard by default
o.splitright = true
o.splitbelow = true
```

> **`undofile = true` is the most underrated setting.** Undo history survives closing the file. `u` still works tomorrow.

> **`relativenumber` is what makes vim motions click.** You see `5` next to a line, so `5dd` or `d5j` is obvious rather than counted.

## 3. Plugin manager — lazy.nvim

```lua
-- bootstrap lazy.nvim (put this after the options above)
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({ "git", "clone", "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git", "--branch=stable", lazypath })
end
vim.opt.rtp:prepend(lazypath)

require("lazy").setup({
  -- colours
  { "catppuccin/nvim", name = "catppuccin", priority = 1000,
    config = function() vim.cmd.colorscheme("catppuccin") end },

  -- fuzzy finder
  { "nvim-telescope/telescope.nvim", branch = "0.1.x",
    dependencies = { "nvim-lua/plenary.nvim" } },

  -- syntax
  { "nvim-treesitter/nvim-treesitter", build = ":TSUpdate" },

  -- LSP
  { "neovim/nvim-lspconfig" },
  { "williamboman/mason.nvim", config = true },
  { "williamboman/mason-lspconfig.nvim" },

  -- completion
  { "hrsh7th/nvim-cmp", dependencies = { "hrsh7th/cmp-nvim-lsp", "L3MON4D3/LuaSnip" } },

  -- quality of life
  { "lewis6991/gitsigns.nvim", config = true },
  { "numToStr/Comment.nvim", config = true },     -- gcc to comment a line
  { "windwp/nvim-autopairs", config = true },
})
```

```vim
:Lazy          " open the plugin manager
:Lazy sync     " install/update everything
:Lazy profile  " what's slowing startup down
```

## 4. LSP — the bit that makes it an IDE

```lua
require("mason").setup()
require("mason-lspconfig").setup({
  ensure_installed = { "pyright", "ruff", "rust_analyzer", "ts_ls", "lua_ls" },
})

local lsp = require("lspconfig")
local caps = require("cmp_nvim_lsp").default_capabilities()

for _, server in ipairs({ "pyright", "ruff", "rust_analyzer", "ts_ls" }) do
  lsp[server].setup({ capabilities = caps })
end

-- keymaps, only active when a language server attaches
vim.api.nvim_create_autocmd("LspAttach", {
  callback = function(ev)
    local map = function(k, fn) vim.keymap.set("n", k, fn, { buffer = ev.buf }) end
    map("gd", vim.lsp.buf.definition)
    map("gr", vim.lsp.buf.references)
    map("K",  vim.lsp.buf.hover)
    map("<leader>rn", vim.lsp.buf.rename)
    map("<leader>ca", vim.lsp.buf.code_action)
    map("[d", vim.diagnostic.goto_prev)
    map("]d", vim.diagnostic.goto_next)
  end,
})
```

```vim
:Mason            " install/manage language servers
:LspInfo          " is a server attached to THIS buffer?
:checkhealth lsp
```

> **`:LspInfo` is the first thing to run when "autocomplete isn't working".** It tells you whether a server attached at all, which is usually the answer.

| Language | Server |
|---|---|
| [[Python]] | `pyright` (types) + `ruff` (lint/format) |
| [[Rust]] | `rust_analyzer` |
| [[TypeScript]] | `ts_ls` |
| [[C++]] / [[C]] | `clangd` |
| Lua | `lua_ls` |

## 5. Treesitter — proper syntax and text objects

```lua
require("nvim-treesitter.configs").setup({
  ensure_installed = { "python", "rust", "lua", "typescript", "sql", "bash", "json", "yaml" },
  highlight = { enable = true },
  indent = { enable = true },
})
```

Gives real syntax highlighting **and** smarter text objects — `vaf` selects a whole function.

## 6. Telescope keymaps

```lua
local t = require("telescope.builtin")
vim.keymap.set("n", "<leader>ff", t.find_files)
vim.keymap.set("n", "<leader>fg", t.live_grep)     -- needs ripgrep
vim.keymap.set("n", "<leader>fb", t.buffers)
vim.keymap.set("n", "<leader>fh", t.help_tags)
vim.keymap.set("n", "<leader>fd", t.diagnostics)
```

> With leader as space: `space f f` = find file, `space f g` = grep the project.

## 7. Config layout, once it grows

```
~/.config/nvim/
├── init.lua              -- entry point
└── lua/
    ├── config/
    │   ├── options.lua
    │   ├── keymaps.lua
    │   └── lazy.lua
    └── plugins/
        ├── lsp.lua
        ├── telescope.lua
        └── treesitter.lua
```

```lua
-- init.lua
require("config.options")
require("config.keymaps")
require("config.lazy")     -- lazy.nvim then auto-loads lua/plugins/*.lua
```

> **Note the `lua/` directory.** `require("config.options")` resolves to `lua/config/options.lua`. That mapping catches everyone once.

## 8. Windows and WSL specifics

### Clipboard in WSL

```lua
-- WSL: route the clipboard through Windows
if vim.fn.has("wsl") == 1 then
  vim.g.clipboard = {
    name = "WslClipboard",
    copy = { ["+"] = "clip.exe", ["*"] = "clip.exe" },
    paste = {
      ["+"] = 'powershell.exe -c [Console]::Out.Write($(Get-Clipboard -Raw).tostring().replace("`r", ""))',
      ["*"] = 'powershell.exe -c [Console]::Out.Write($(Get-Clipboard -Raw).tostring().replace("`r", ""))',
    },
    cache_enabled = 0,
  }
end
```

> **Without this, `"+y` in WSL copies into a void.** `:checkhealth` reports no clipboard provider.

### Line endings

```lua
vim.opt.fileformats = "unix,dos"
```

> **Work inside the WSL filesystem (`~/`), not `/mnt/c/`.** Accessing Windows files from WSL is dramatically slower — a plugin update that takes 5 seconds in `~` takes a minute on `/mnt/c`. Same rule as [[Docker deep dive]].

### Windows terminal

Use **Windows Terminal** with a Nerd Font (`CaskaydiaCove NF`) or icons render as boxes.

## 9. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Config not loading | Wrong path | `:echo stdpath('config')` |
| Which files loaded? | — | `:scriptnames` |
| Plugins not installing | lazy.nvim not bootstrapped, or no git | `:Lazy sync`; check `git --version` |
| No autocomplete | Server not attached | `:LspInfo`, then `:Mason` |
| `"+y` does nothing | No clipboard provider | The WSL block above; `:checkhealth` |
| Icons are boxes | No Nerd Font | Install one, set it in the terminal |
| Slow startup | A plugin loading eagerly | `:Lazy profile` |
| Treesitter errors | Missing C compiler | `apt install build-essential` |
| Colours look wrong | Terminal lacks true colour | `termguicolors = true` + a modern terminal |
| Everything broken after an edit | — | `nvim -u NONE` starts with **no config** |

```vim
:checkhealth        " the single most useful diagnostic command
:messages           " errors you missed on startup
```

> **`nvim -u NONE` is the escape hatch.** Starts a bare Neovim ignoring your config, so you can still edit the config file that's breaking things.

## Related

[[Nvim]] · [[Default Nvim]] · [[Keybinds]] · [[Tools]] · [[Dev environment - Git, Docker, CLI]]
