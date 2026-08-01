# Neovim 使用手册

Neovim 是 Vim 的现代化重写版本，操作完全兼容 Vim，本文仅记录 Neovim 特有或扩展的部分，基础操作见 [vim-usage-guide.md](vim-usage-guide.md)。

## 安装

```bash
sudo pacman -S neovim   # Arch Linux
```

## 配置文件

```bash
~/.config/nvim/init.lua     # 主配置（Lua，推荐）
~/.config/nvim/init.vim     # 或 VimScript
```

初始化：

```bash
mkdir -p ~/.config/nvim
nvim ~/.config/nvim/init.lua
```

---

## 最小可用配置

```lua
-- ~/.config/nvim/init.lua
vim.opt.number = true
vim.opt.relativenumber = true
vim.opt.tabstop = 4
vim.opt.shiftwidth = 4
vim.opt.expandtab = true
vim.opt.wrap = false
vim.opt.ignorecase = true
vim.opt.smartcase = true
vim.opt.termguicolors = true
vim.opt.clipboard = "unnamedplus"   -- 与系统剪贴板共享
```

---

## 插件管理（lazy.nvim）

安装 lazy.nvim（粘贴到 init.lua 顶部）：

```lua
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({
    "git", "clone", "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable", lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)
```

插件列表示例：

```lua
require("lazy").setup({
  { "nvim-treesitter/nvim-treesitter" },  -- 语法高亮
  { "neovim/nvim-lspconfig" },            -- LSP 支持
  { "hrsh7th/nvim-cmp" },                 -- 自动补全
  { "nvim-telescope/telescope.nvim" },    -- 模糊搜索
  { "folke/tokyonight.nvim" },            -- 主题
})
```

启动后运行 `:Lazy` 管理插件。

---

## 常用内置命令

```
:checkhealth        检查环境与插件状态
:Lazy               插件管理界面
:LspInfo            查看 LSP 状态
:TSInstall python   安装语言语法支持
:terminal           打开内置终端
```

终端模式切换：

```
:terminal           进入终端
i                   进入终端输入模式
Ctrl+\ Ctrl+n       退出终端模式回到普通模式
```

---

## 与 Vim 的主要差异

| 特性 | Vim | Neovim |
|------|-----|--------|
| 配置语言 | VimScript | Lua（推荐）/ VimScript |
| 内置 LSP | 无 | 有（nvim-lspconfig）|
| 内置终端 | 有限 | 完整支持 |
| 异步任务 | 有限 | 完整支持 |
| 插件生态 | vim-plug 为主 | lazy.nvim 为主 |

---

## 学习路径

1. 运行内置教程：

```bash
nvim +Tutor
```

2. 熟悉基础移动和编辑操作（参考 vim-usage-guide.md）
3. 配置 init.lua 加入基础选项
4. 按需加插件，不建议直接套用他人的完整配置
