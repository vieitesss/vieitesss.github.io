---
title: Minimal Neovim config v0.12 edition
description: Less plugins, same power.
categories: [Neovim]
tags: [lua,configuration]
---

## Overview

Neovim v0.12 is on the way, and it brings a lot of new features, being the most important (in my opinion) the new LSP API, and the builtin package manager.

In this post, I will recreate my Neovim configuration using the less plugins possible, while still keeping the same power and features I had before. With **only 6 plugins**, I will be able to have a fully functional Neovim setup that includes most of the features I use on a daily basis.

This is how our configuration will look like: 

```
~/.config/nvim/
├── init.lua
├── lua
│   ├── autocmds.lua
│   ├── configs.lua
│   ├── keymaps.lua
│   ├── lsp.lua
│   ├── plugins.lua
│   └── statusline.lua
└── lsp
    ├── rust-analyzer.lua
    ├── gopls.lua
    ├── helm_ls.lua
    ├── texlab.lua
    ├── bashls.lua
    ├── ts_ls.lua
    └── lua_ls.lua
```

> I want to use this as a way to encourage you to check your current Neovim configuration and see if you can simplify it. Doing this, I have reduced my configuration from **nearly 2000 lines** to **less than 500 lines**. Just by removing unused plugins, and using the new features of Neovim v0.12, and builtin tools.
{: .prompt-tip }

## Prerequisites

- Neovim: `>= 0.12.0` (`nightly` right now).
- Basic Lua knowledge.

## Neovim Configuration

### Install Neovim and set up a new configuration directory

The first step is to install Neovim. You can do this by searching for the v0.12 release (or greater) on the [Neovim GitHub page](https://github.com/neovim/neovim/releases/).

Imagine that you already have your configuration directory set up at `~/.config/nvim/`, and you don't want to remove everything you have there. You can create a new directory called `~/.config/nvim-new/` and put your new configuration there.

To use it without changing your current setup, you can start Neovim with the `NVIM_APPNAME` environment variable set to `nvim-new`. Create an alias in your shell configuration file (e.g., `~/.bashrc` or `~/.zshrc`):

```bash
# You may have
# alias v='nvim'
alias vv='NVIM_APPNAME=nvim-new nvim'
```

This way, you can use `vv` to start Neovim with the new configuration without affecting your current setup. And, if something goes wrong, you can always start Neovim with the default configuration by using `nvim` or your existing alias.

### Global settings

We will create a minimal configuration, but this doesn't mean that we cannot structure it well. Let's create a `init.lua` file in the `~/.config/nvim-new/` directory, and a `lua/` directory, where we will put our Lua modules. The `init.lua` file will be the entry point of our configuration.

```bash
touch ~/.config/nvim-new/init.lua
mkdir -p ~/.config/nvim-new/lua
```

Inside the `lua` directory, we will create a `configs.lua` file to hold our global configs.

```bash
touch ~/.config/nvim-new/lua/configs.lua
```

We have to require this file in our `init.lua`:

```lua
-- ~/.config/nvim-new/init.lua
require('configs')
```

Now, let's add some global settings in the `configs.lua` file:

```lua
-- ~/.config/nvim-new/lua/configs.lua
local opt = vim.opt
opt.guicursor = "i:block" -- Use block cursor in insert mode
opt.colorcolumn = "80" -- Highlight column 80
opt.signcolumn = "yes:1" -- Always show sign column
opt.termguicolors = true -- Enable true colors
opt.ignorecase = true -- Ignore case in search
opt.swapfile = false -- Disable swap files
opt.autoindent = true -- Enable auto indentation
opt.expandtab = true -- Use spaces instead of tabs
opt.tabstop = 4 -- Number of spaces for a tab
opt.softtabstop = 4 -- Number of spaces for a tab when editing
opt.shiftwidth = 4 -- Number of spaces for autoindent
opt.shiftround = true -- Round indent to multiple of shiftwidth
opt.listchars = "tab: ,multispace:|   ,eol:󰌑" -- Characters to show for tabs, spaces, and end of line
opt.list = true -- Show whitespace characters
opt.number = true -- Show line numbers
opt.relativenumber = true -- Show relative line numbers
opt.numberwidth = 2 -- Width of the line number column
opt.wrap = false -- Disable line wrapping
opt.cursorline = true -- Highlight the current line
opt.scrolloff = 8 -- Keep 8 lines above and below the cursor
opt.inccommand = "nosplit" -- Shows the effects of a command incrementally in the buffer
opt.undodir = os.getenv('HOME') .. '/.vim/undodir' -- Directory for undo files
opt.undofile = true -- Enable persistent undo
opt.completeopt = { "menuone", "popup", "noinsert" } -- Options for completion menu
opt.winborder = "rounded" -- Use rounded borders for windows

vim.cmd.filetype("plugin indent on") -- Enable filetype detection, plugins, and indentation
```

Those are the settings I use in my Neovim configuration. You can customize them to your liking.

### Basic keymaps

Next, let's create a `keymaps.lua` file in the `lua` directory to hold our key mappings. This will help us keep our configuration organized.

```bash
touch ~/.config/nvim-new/lua/keymaps.lua
```

Now, let's add some basic key mappings in the `keymaps.lua` file:

```lua
-- ~/.config/nvim-new/lua/keymaps.lua
local keymap = vim.keymap.set
local s = { silent = true }

vim.g.mapleader = " "

keymap("n", "<space>", "<Nop>")

keymap("n", "j", function()
    return tonumber(vim.api.nvim_get_vvar("count")) > 0 and "j" or "gj"
end, { expr = true, silent = true }) -- Move down, but use 'gj' if no count is given
keymap("n", "k", function()
    return tonumber(vim.api.nvim_get_vvar("count")) > 0 and "k" or "gk"
end, { expr = true, silent = true }) -- Move up, but use 'gk' if no count is given
keymap("n", "<C-d>", "<C-d>zz") -- Scroll down and center the cursor
keymap("n", "<C-u>", "<C-u>zz") -- Scroll up and center the cursor
keymap("n", "<Leader>w", "<cmd>w!<CR>", s) -- Save the current file
keymap("n", "<Leader>q", "<cmd>q<CR>", s) -- Quit Neovim
keymap("n", "<Leader>no", "<cmd>noh<CR>", s) -- Clear search highlights
keymap("n", "<Leader>te", "<cmd>tabnew<CR>", s) -- Open a new tab
keymap("n", "<Leader>tp", "<cmd>tabp<CR>", s) -- Go to the previous tab
keymap("n", "<Leader>tn", "<cmd>tabn<CR>", s) -- Go to the next tab
keymap("n", "<Leader>_", "<cmd>vsplit<CR>", s) -- Split the window vertically
keymap("n", "<Leader>-", "<cmd>split<CR>", s) -- Split the window horizontally
keymap("n", "<Leader>fo", ":lua vim.lsp.buf.format()<CR>", s) -- Format the current buffer using LSP
keymap("v", "<Leader>p", '"_dP') -- Paste without overwriting the default register
keymap("x", "y", [["+y]], s) -- Yank to the system clipboard in visual mode
keymap("t", "<Esc>", "<C-\\><C-N>") -- Exit terminal mode
-- Change directory to the current file's directory
keymap("n", "<leader>cd", '<cmd>lua vim.fn.chdir(vim.fn.expand("%:p:h"))<CR>')

local opts = { noremap = true, silent = true }
keymap("n", "grd", "<cmd>lua vim.lsp.buf.definition()<CR>", opts) -- Go to definition
```

This are some basic key mappings. We will add more as we go along, but this is a good starting point. You can customize the key mappings to your liking.

### Auto commands

Next, let's create an `autocmds.lua` file in the `lua` directory to hold our auto commands.

```bash
touch ~/.config/nvim-new/lua/autocmds.lua
```

I'm just going to add one autocommand that I think is useful. We will add more later.

```lua
-- ~/.config/nvim-new/lua/autocmds.lua
local autocmd = vim.api.nvim_create_autocmd
local augroup = vim.api.nvim_create_augroup

-- Highlight yanked text
local highlight_group = augroup('YankHighlight', { clear = true })
autocmd('TextYankPost', {
    pattern = '*',
    callback = function()
        vim.highlight.on_yank({ timeout = 170 })
    end,
    group = highlight_group,
})
```

### Plugins

#### Statusline

The first not-plugin we will use is the builtin statusline. We can customize our own statusline using Lua. I have a [full post](https://vieitesss.github.io/posts/Neovim-custom-status-line/) about how to create a statusline in Neovim.

The statusline will be in a separate file called `statusline.lua` in the `lua` directory.

```bash
touch ~/.config/nvim-new/lua/statusline.lua
```

There, we will include the code for our statusline. Check out the post to see the full implementation!

For the previous statusline to work, we need to require it in our `init.lua` file:

```lua
-- ~/.config/nvim-new/init.lua
require('statusline')
```

And, we also need to install the [gitsigns.nvim](https://github.com/lewis6991/gitsigns.nvim) plugin, which we will use to show the Git status in the statusline.

We will add it to our `plugins.lua` file, which we will create in the `lua` directory:

```bash
touch ~/.config/nvim-new/lua/plugins.lua
```

Now, let's add the plugin:

```lua
-- ~/.config/nvim-new/lua/plugins.lua
vim.pack.add({
    { src = "https://github.com/lewis6991/gitsigns.nvim" },
})

require('gitsigns').setup({ signcolumn = false })
```

> Check `:help vim.pack` for more information about the builtin package manager. It's important to look to the `vim.pack.add` and `vim.pack.update` functions.
{: .prompt-tip }

We can do `:restart` or reopen Neovim to load the new configuration. A prompt will appear asking us to install the plugin. We can press `y` to accept the installation.

Let's add a keymap as well, to update the plugins easily:

```lua
-- ~/.config/nvim-new/lua/keymaps.lua
keymap("n", "<leader>ps", '<cmd>lua vim.pack.update()<CR>')
```

> When the update tab is shown after running the command above, you have to `:write` the file to confirm the changes, or `:quit` to discard them.
{: .prompt-tip }

#### LSP

For the LSP integration, we will need only one plugin: [mason.nvim](https://github.com/mason-org/mason.nvim). This plugin will help us manage the LSP servers, and it has a built-in UI to install and update them.

Now, let's add the plugin configuration in the `plugins.lua` file:

```lua
-- ~/.config/nvim-new/lua/plugins.lua
vim.pack.add({
    { src = "https://github.com/mason-org/mason.nvim" },
})

require("mason").setup({})
```

Now, we can install the LSP servers we need using Mason. For example, the `lua-language-server` LSP. But we have to configure the LSP servers. This is done in the `lsp/` directory. We will create a file for each LSP server we want to use. But first, we need to create the `lsp.lua` file in the `lua/` directory, which will be the entry point for our LSP configuration.

```bash
touch ~/.config/nvim-new/lua/lsp.lua
```

In the `lsp.lua` file, we will set up the LSP servers we want to use, enabling them with the new LSP API.

```lua
-- ~/.config/nvim-new/lua/lsp.lua
vim.lsp.enable({
  "bashls",
  "gopls",
  "lua_ls",
  "texlab",
  "ts_ls",
  "rust-analyzer",
  "helm_ls",
})
vim.diagnostic.config({ virtual_text = true })
```

We will require this file in our `init.lua`:

```lua
-- ~/.config/nvim-new/init.lua
require('lsp')
```

Now, we will create a file for each LSP server configuration. For example, for the `lua_ls` server, we will create a file called `lua_ls.lua` in the `lsp/` directory:

```bash
mkdir -p ~/.config/nvim-new/lsp
touch ~/.config/nvim-new/lsp/lua_ls.lua
```

And we will add the configuration for the `lua_ls` server in that file:

```lua
-- ~/.config/nvim-new/lsp/lua_ls.lua
---@type vim.lsp.Config
return {
    cmd = { 'lua-language-server' },
    filetypes = { 'lua' },
    root_markers = {
        '.luarc.json',
        '.luarc.jsonc',
        '.luacheckrc',
        '.stylua.toml',
        'stylua.toml',
        'selene.toml',
        'selene.yml',
        '.git',
    },
    settings = {
        Lua = {
            runtime = {
                version = "Lua 5.4",
            },
            completion = {
                enable = true,
            },
            diagnostics = {
                enable = true,
                globals = { "vim" },
            },
            workspace = {
                library = { vim.env.VIMRUNTIME },
                checkThirdParty = false,
            },
        },
    },
}
```

> To know the minimum configuration needed for each LSP server, you can check the [nvim-lspconfig documentation](https://github.com/neovim/nvim-lspconfig/blob/master/doc/configs.md#lua_ls), where you can find the default configuration for each server.
{: .prompt-tip }

We will do the same for the other LSP servers.

After creating the previous `ls_lua.lua` file, we should have the LSP for Lua up and running. We can check that with `:checkhealth lsp`.

#### Completion

Another of the must-have plugins for me is [blink.cmp](https://github.com/saghen/blink.cmp). Is uses fuzzy matching to provide completion suggestions, a very useful feature in my opinion.

We will add it to our `plugins.lua` file:

```lua
-- ~/.config/nvim-new/lua/plugins.lua
vim.pack.add({
    { src = "https://github.com/saghen/blink.cmp", version = vim.version.range("^1") },
})

require('blink.cmp').setup({
    fuzzy = { implementation = 'prefer_rust_with_warning' },
    signature = { enabled = true },
    keymap = {
        preset = "default",
        ["<C-space>"] = {},
        ["<C-p>"] = {},
        ["<Tab>"] = {},
        ["<S-Tab>"] = {},
        ["<C-y>"] = { "show", "show_documentation", "hide_documentation" },
        ["<C-n>"] = { "select_and_accept" },
        ["<C-k>"] = { "select_prev", "fallback" },
        ["<C-j>"] = { "select_next", "fallback" },
        ["<C-b>"] = { "scroll_documentation_down", "fallback" },
        ["<C-f>"] = { "scroll_documentation_up", "fallback" },
        ["<C-l>"] = { "snippet_forward", "fallback" },
        ["<C-h>"] = { "snippet_backward", "fallback" },
        -- ["<C-e>"] = { "hide" },
    },

    appearance = {
        use_nvim_cmp_as_default = true,
        nerd_font_variant = "normal",
    },

    completion = {
        documentation = {
            auto_show = true,
            auto_show_delay_ms = 200,
        }
    },

    cmdline = {
        keymap = {
            preset = 'inherit',
            ['<CR>'] = { 'accept_and_enter', 'fallback' },
        },
    },

    sources = { default = { "lsp" } }
})
```

> I leave my configuration for `blink.cmp` here, but you can customize it to your liking. Check the [blink.cmp documentation](https://cmp.saghen.dev/)
{: .prompt-info }

We already have completion working with the LSP servers we have configured!

#### Netrw

Oh yes.

I'm an [Oil.nvim](https://github.com/stevearc/oil.nvim) user, you may be too. But listen, **give Netrw a chance**.

It is really customizable, and it has a lot of features that you may not know about, as I didn't until I decided to use it. Here you have a [really good post](https://vonheikemen.github.io/devlog/tools/using-netrw-vim-builtin-file-explorer/) about it and how to customize it.

We will add some configuration to our `configs.lua` file to enable some features of Netrw:

```lua
-- ~/.config/nvim-new/lua/configs.lua
vim.g.netrw_liststyle = 1 -- Use the long listing view
vim.g.netrw_sort_by = "size" -- Sort files by size
```

Let's also add a keymap to open Netrw easily:

```lua
-- ~/.config/nvim-new/lua/keymaps.lua
keymap("n", "<Leader>ex", "<cmd>Ex %:p:h<CR>") -- Open Netrw in the current file's directory
```

And, it's true that there are some things that are a little bit more complicated to do with Netrw, like creating multiple files at once, or moving files around, but it is not impossible, and it becomes easier with practice. I have been using it for a while now, and I can say that it is a really powerful tool.

But I find useful to have the following keymaps to make it easier to use:

```lua
-- ~/.config/nvim-new/lua/autocmds.lua
vim.api.nvim_create_autocmd("FileType", {
    pattern = "netrw",
    callback = function()
        local bs = { buffer = true, silent = true }
        local bsr = { buffer = true, remap = true, silent = true }
        vim.keymap.set('n', '<C-c>', '<cmd>bd<CR>', bs) -- Close the current Netrw buffer
        vim.keymap.set('n', '<Tab>', 'mf', bsr) -- Mark the file/directory to the mark list
        vim.keymap.set('n', '<S-Tab>', 'mF', bsr) -- Unmark all the files/directories
        -- Improved file creation
        vim.keymap.set('n', '%', function()
            vim.ui.input({ prompt = 'Enter filename: ' }, function(input)
                if input and input ~= '' then
                    vim.cmd('!touch ' .. input) -- Create the file
                    vim.api.nvim_feedkeys('<C-l>', 'n', false) -- Refresh the Netrw buffer
                end
            end)
        end, { buffer = true, silent = true })
    end
})
```

> We need to use `remap = true` to remap the builtin Netrw keymaps. `<C-c>` is not a builtin Netrw keymap, so we don't need to use it.
{: .prompt-info }

#### Colorscheme

I have not forgotten about the colorscheme!

Right now, I am using [techbase.nvim](https://github.com/mcauley-penney/techbase.nvim).

Let's add it to our `plugins.lua` file:

```lua
-- ~/.config/nvim-new/lua/plugins.lua
vim.pack.add({
    { src = "https://github.com/mcauley-penney/techbase.nvim" },
})

require('techbase').setup({})
```

And we can set it as the colorscheme in our `configs.lua` file:

```lua
-- ~/.config/nvim-new/lua/configs.lua
vim.cmd.colorscheme("techbase")
```

#### Fuzzy picker

My preferred fuzzy picker for Neovim is [fzf-lua](https://github.com/ibhagwan/fzf-lua). Let's add it to our `plugins.lua` file:

```lua
-- ~/.config/nvim-new/lua/plugins.lua
vim.pack.add({
    { src = "https://github.com/ibhagwan/fzf-lua" },
})

local actions = require('fzf-lua.actions')
require('fzf-lua').setup({
    winopts = { backdrop = 85 },
    keymap = {
        builtin = {
            ["<C-f>"] = "preview-page-down",
            ["<C-b>"] = "preview-page-up",
            ["<C-p>"] = "toggle-preview",
        },
        fzf = {
            ["ctrl-a"] = "toggle-all",
            ["ctrl-t"] = "first",
            ["ctrl-g"] = "last",
            ["ctrl-d"] = "half-page-down",
            ["ctrl-u"] = "half-page-up",
        }
    },
    actions = {
        files = {
            ["ctrl-q"] = actions.file_sel_to_qf,
            ["ctrl-n"] = actions.toggle_ignore,
            ["ctrl-h"] = actions.toggle_hidden,
            ["enter"]  = actions.file_edit_or_qf,
        }
    }
})
```

As I have been doing, I leave my configuration for `fzf-lua` here, but you can customize it to your liking.

Let's add some keymaps to use it easily:

```lua
-- ~/.config/nvim-new/lua/keymaps.lua
keymap("n", "<leader>ff", '<cmd>FzfLua files<CR>')
keymap("n", "<leader>fg", '<cmd>FzfLua live_grep<CR>')
```

#### Git

For Git integration, I use [vim-fugitive](https://github.com/tpope/vim-fugitive). It is a really powerful plugin that allows you to interact with Git from within Neovim.

Let's add it to our `plugins.lua` file:

```lua
-- ~/.config/nvim-new/lua/plugins.lua
vim.pack.add({
    { src = "https://github.com/tpope/vim-fugitive" },
})

-- No configuration needed for vim-fugitive
```

We can add some keymaps to use it easily:

```lua
-- ~/.config/nvim-new/lua/keymaps.lua
keymap("n", "<leader>gs", '<cmd>Git<CR>', opts)
keymap("n", "<leader>gp", '<cmd>Git push<CR>', opts)
```
