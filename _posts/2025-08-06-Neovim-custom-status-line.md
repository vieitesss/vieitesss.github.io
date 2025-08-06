---
title: Neovim custom status line
date: 2025-08-06 16:20:00 +0200
categories: [Neovim]
tags: [lua,statusline,module]
---

## Overview

In this post, we are going to create **our custom, minimal status line for Neovim**. This will render:

- When in the **active** buffer:
  - The current **file path** from the current working directory.
  - The **branch and changes** (if it is in a Git repository).
  - The **file type**.
  - The **amount scrolled** from the top of the file (prencentage).
  - The current **line and column** of the cursor.

- When in an inactive buffer:
  - Just the **file name**.

> This is what I personally need, but you will see that this is very customizable for your liking.
{: .prompt-info }

We'll also have **custom keybindings** to **toggle the visibility of part of the path of the file** we are editing, and the visibility of **the current branch** we are in, if the file is part of a Git repository. This will keep the bar **tidy and clean** in case the window is smaller than usual or the names of the file or the branch are too long.

## Prerequisites

- Neovim: `>= 0.9.0`.
- `vim.opt.laststatus = 2`: This is the default behaviour, it shows always the status line in the previous focused window.
- [gitsigns](https://github.com/lewis6991/gitsigns.nvim), for the branch and changes when in a Git repository.
- It would be advisable to have a Nerd Font, or a font that is able to render icons.
- Lua basic knowledge.

## Creating the status line

### Config structure

I am going to show how I have my configuration structured for this, but it is not the only way you can structure it.

```shell
$HOME/.config/nvim
├── init.lua
└── lua
    └── vt
        └── status_line.lua
```

Inside the `init.lua` file I have the requirements, everything I tell Neovim to load when it starts:

```lua
-- require("vt.specs")
-- require("vt.lazy")
-- require("vt.keymaps")
-- require("vt.configs")
-- require("vt.autocmds")
-- require("vt.lsp")
require("vt.status_line")
-- require("vt.fzf-lua")
```

In the `status_line.lua` file is where we are going to implement our custom status line (if it wasn't obvious).

### Status line basics

`statusline` is a *format string* Neovim evaluates per window to draw the bar. The active window uses `StatusLine`, inactive uses `StatusLineNC`. See `:h statusline`.

#### Core building blocks

- **Alignment**: `%=` splits the bar into zones (left \| right; add another `%=` for a middle).
- **Truncation**: `%<` marks where text to its left may be shortened when space is tight.
- **Literals**: `%%` prints a literal `%`.
- **Expressions**: `%{...}` evaluates an expression.
- **Highlighting**: `%#Group#` switches highlight; `%*` resets to the default group.

#### Common built-ins (quick ref)

- **File**: `%t` (tail), `%f` (relative path), `%F` (full path), `%y` (filetype)
- **Flags**: `%m` (modified +), `%r` (readonly)
- **Position**: `%l` (line), `%c` (col), `%L` (total), `%p%%` (percent)

### Implementation

Let's start with a basic version of our status line:

```lua
Statusline = {}

function Statusline.active()
  -- `%P` shows the scroll percentage but says 'Bot', 'Top' and 'All' as well.
  return "[%f]%=%y [%P %l:%c]"
end

function Statusline.inactive()
    return " %t"
end

vim.api.nvim_exec2([[
  -- Starts an autocommand group with the name `Statusline`.
  augroup Statusline

  -- Clears any autocommands already in this group.
  au!

  -- On `WinEnter` or `BufEnter` events, for any file (`*`), set the `window-local`
  -- option statusline to an expression (`%!...`) that calls your Lua function
  -- `Statusline.active()`. `%!` means "evaluate this expression whenever the
  -- statusline is drawn." `v:lua.Statusline.active()` bridges to the Lua global
  -- `Statusline.active`.
  au WinEnter,BufEnter * setlocal statusline=%!v:lua.Statusline.active()

  -- On `WinLeave` or `BufLeave` events, switch that window’s statusline to the
  -- inactive variant by calling `Statusline.inactive()`.
  au WinLeave,BufLeave * setlocal statusline=%!v:lua.Statusline.inactive()

  -- Ends the group
  augroup END
]], {})
```

- *Why `setlocal`?*

`statusline` is **window-local**. This ensures each window has its own value. With `laststatus=3` (single global bar), Neovim still uses the current window’s statusline value for the bar, so the events keep it in sync.
