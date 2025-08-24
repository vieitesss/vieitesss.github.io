---
title: Custom Neovim statusline
description: How to create your own statusline inside Neovim.
categories: [Neovim]
tags: [lua,statusline,module]
image:
  path: /assets/posts/2025-08-06-Neovim-custom-status-line.png
  alt: Neovim custom statusline
---

## Overview

![Statusline visible](/assets/posts/statusline-visible.png)
_Active statusline_
![Statusline invisible](/assets/posts/statusline-invisible.png)
_Active statusline with hidden segments_
![Statusline invisible](/assets/posts/inactive.png)
_Inactive statusline_

In this post we will build **a minimal, custom statusline for Neovim** that shows:

- When the window is active:
  - The current **file path** (relative to the current working directory).
  - The **Git branch and change counts** (if the file is in a Git repository).
  - The **filetype**.
  - How far you have **scrolled** in the file (percentage).
  - The current **line and column** of the cursor.

- When the window is inactive:
  - Only the **file name**.

> These are the pieces I personally find useful, but you will see that everything is fully customizable.
{: .prompt-info }

We will also add **custom key-bindings** to toggle:

- The **directory portion** of the path.
- The **Git branch name** (keeping the change counts).

This keeps the bar tidy when the space is tight or the names are long.

> Follow this guide to understand the final implementation, that can be found at the end of the post.
{: .prompt-tip }



## Prerequisites

- Neovim: `>= 0.9.0`.
- `vim.opt.laststatus = 2` (the default; always show a statusline in the previously focused window).
- [gitsigns.nvim](https://github.com/lewis6991/gitsigns.nvim) for Git information.
- A Nerd Font (or any font that can render icons).
- Basic Lua knowledge.

## Config structure (one example)

```shell
$HOME/.config/nvim
├── init.lua
└── lua
    ├── vt
    │   ├── specs.lua
    │   └── status_line.lua
    └── plugins
        └── gitsigns.lua
```

In `init.lua` we simply require the modules:

```lua
require("vt.specs")
require("vt.status_line")
-- more requires ...
```

I use [lazy.nvim](https://lazy.folke.io/) as a plugin manager, so `specs.lua` just lists plugin specs:

```lua
LAZY_PLUGIN_SPEC = {}

local function spec(item)
    table.insert(LAZY_PLUGIN_SPEC, { import = item })
end

spec("plugins.gitsigns")
-- more plugins ...
```

A minimal `plugins/gitsigns.lua` looks like:

```lua
return {
    "lewis6991/gitsigns.nvim",
    lazy = false,
    config = true,
}
```

All the statusline code will live in `lua/vt/status_line.lua`.

## Statusline basics

`statusline` is a *format string* Neovim evaluates for each window. The active window uses `StatusLine` highlight group; inactive windows use `StatusLineNC`. See `:h statusline`.

### Core building blocks

- **Alignment**: `%=` splits the bar into zones (left \| right; add another `%=` for a middle).
- **Truncation**: `%<` marks where text to its left may be shortened when space is tight.
- **Literals**: `%%` prints a literal `%`.
- **Expressions**: `%{...}` evaluates an expression.
- **Highlighting**: `%#Group#` switches highlight; `%*` resets to the default group.

### Common built-ins (quick ref)

- **File**: `%t` (tail), `%f` (relative path), `%F` (full path), `%y` (filetype)
- **Flags**: `%m` (modified +), `%r` (readonly)
- **Position**: `%l` (line), `%c` (col), `%L` (total), `%p%%` (percent)

## Creating the statusline

Let's start with a basic version of our statusline:

```lua
Statusline = {}

function Statusline.active()
    -- `%P` shows the scroll percentage but says 'Bot', 'Top' and 'All' as well.
    return "[%f]%=%y [%P %l:%c]"
end

function Statusline.inactive()
    return " %t"
end

local group = vim.api.nvim_create_augroup("Statusline", { clear = true })

vim.api.nvim_create_autocmd({ "WinEnter", "BufEnter" }, {
    group = group,
    desc = "Activate statusline on focus",
    callback = function()
        vim.opt_local.statusline = "%!v:lua.Statusline.active()"
    end,
})

vim.api.nvim_create_autocmd({ "WinLeave", "BufLeave" }, {
    group = group,
    desc = "Deactivate statusline when unfocused",
    callback = function()
        vim.opt_local.statusline = "%!v:lua.Statusline.inactive()"
    end,
})
```

`statusline` is *window-local*, so we use `setlocal`. Even with `laststatus=3` (a single global bar) Neovim still reads the value from the current window, so these autocommands keep it correct.

### Adding git information

`gitsigns.nvim` populates `vim.b.gitsigns_status_dict`; we turn that into a segment:

```lua
local function git()
    local git_info = vim.b.gitsigns_status_dict
    if not git_info or git_info.head == "" then
    return ""
    end

    local head    = git_info.head
    local added   = git_info.added and (" +" .. git_info.added) or ""
    local changed = git_info.changed and (" ~" .. git_info.changed) or ""
    local removed = git_info.removed and (" -" .. git_info.removed) or ""
    if git_info.added == 0 then added = "" end
    if git_info.changed == 0 then changed = "" end
    if git_info.removed == 0 then removed = "" end

    return table.concat({
        "[ ", -- branch icon
        head,
        added, changed, removed,
        "]",
    })
end
```

Update the active statusline:

```lua
function Statusline.active()
    return table.concat {
        "[%f] ",
        git(),
        "%=",
        "%y [%P %l:%c]"
    }
end
```

Already looking good!

### Keybindings to hide items

#### Toggle file path visibility

First split the path and file name. We’ll return the directory only:

```lua
local function filepath()
    -- Modify the given file path with the given modifiers
    local fpath = vim.fn.fnamemodify(vim.fn.expand "%", ":~:.:h")

    if fpath == "" or fpath == "." then
        return ""
    end

    return string.format("%%<%s/", fpath)
    -- `%%` -> `%`.
    -- `%s` -> value of `fpath`.
    -- The result is `%<fpath/`.
    -- `%<` tells where to truncate when there is not enough space.
end
```

See `:h filename-modifiers` to know what the modifiers in the line 3 do.

Add it to the render function:

```lua
function Statusline.active()
    return table.concat {
        -- Before: `[%f]`
        -- `%t` shows only the file name
        "[", filepath(), "%t] ",
        git(),
        "%=",
        "%y [%P %l:%c]"
    }
end
```

We'll use the function we have just created to toggle the visibility of the file path. Let's start **saving the state** and adding some **configuration**.

```lua
-- At the start of the file
local state = {
    show_path = true,
}

local config = {
    icons = {
        path_hidden = "" -- opened directory icon
    },
}
```

Change the `filepath` function logic.

```lua
local function filepath()
    local fpath = vim.fn.fnamemodify(vim.fn.expand "%", ":~:.:h")

    if fpath == "" or fpath == "." then
        return ""
    end

    -- Whether to show the path or the icon
    if state.show_path then
        return string.format("%%<%s/", fpath)
    end

    return config.icons.path_hidden .. "/"
end
```

Create **a function that changes the current path status**, and **a keymap** in order to call that function.

```lua
function Statusline.toggle_path()
    state.show_path = not state.show_path

    -- Draw the statusline manually
    vim.cmd("redrawstatus")
end

-- `sp` for "statusline path"
vim.keymap.set("n", "<leader>sp", function() Statusline.toggle_path() end, { desc = "Toggle statusline path" })
```

Perfect! Now we are able to hide the file path and show it again if we need to.

#### Toggle Git branch visibility

You may also have long branch names that you want to hide from time to time. So, let's add the same functionality for them.

```lua
local state = {
    show_branch = true,
}

local config = {
    icons = {
        branch_hidden = "", -- crossed out eye icon
    },
}


local function git()
    -- ...

    if not state.show_branch then
        head = config.icons.branch_hidden
    end

    return table.concat({
        "[ ",
        head,
        added, changed, removed,
        "]",
    })
end

function Statusline.toggle_branch()
    state.show_branch = not state.show_branch
    vim.cmd("redrawstatus")
end

-- `sb` for `statusline branch`
vim.keymap.set("n", "<leader>sb", function() Statusline.toggle_branch() end, { desc = "Toggle statusline git branch" })
```

That's it!

> As you can see, this is really customizable. You can do whatever you want with your statusline.
{: .prompt-tip }

### Adding color

The statusline looks already really good for me, but you may want some color. We'll create a function to help us applying any highlight group to text.

```lua
local function hl(group, text)
    return string.format("%%#%s#%s%%*", group, text)
    -- Result: `%#group#text%*`
    -- `%#group#` tells the highlight group that must be applied to `text`.
    -- `%*` restores the normal highlight group.
end
```

**Create a custom highlight group**.

```lua
local config = {
    placeholder_hl = "StatusLineDim", -- a dim highlight group we define below
}

-- set (or link) the dim highlight once
vim.api.nvim_set_hl(0, config.placeholder_hl, {}) -- create if missing
-- link to `Comment` to keep it dim; adjust as you like
vim.api.nvim_set_hl(0, config.placeholder_hl, { link = "Comment" })
```

Apply it to our icons.

```lua
local function filepath()
    -- ...

    return hl(config.placeholder_hl, config.icons.path_hidden .. "/")
end

local function git()
    -- ...

    if not state.show_branch then
        head = hl(config.placeholder_hl, config.icons.branch_hidden)
    end

    -- ...
end
```

## Full implementation
<details markdown="1">
  <summary>Here you have the full statusline implemented</summary>

  ```lua
  -- internal state for toggles
  local state = {
      show_path = true,
      show_branch = true,
  }

  -- config for placeholders + highlighting
  local config = {
      icons = {
          path_hidden = "",
          branch_hidden = "",
      },
      placeholder_hl = "StatusLineDim",
  }

  -- helper to wrap text in a statusline highlight group
  local function hl(group, text)
      return string.format("%%#%s#%s%%*", group, text)
  end

  -- create and link the highlight group(s)
  vim.api.nvim_set_hl(0, config.placeholder_hl, {}) -- create if missing
  vim.api.nvim_set_hl(0, config.placeholder_hl, { link = "Comment" })

  local function filepath()
      local fpath = vim.fn.fnamemodify(vim.fn.expand "%", ":~:.:h")

      if fpath == "" or fpath == "." then
          return ""
      end

      if state.show_path then
          return string.format("%%<%s/", fpath)
      end

      return hl(config.placeholder_hl, config.icons.path_hidden .. "/")
  end

  local function git()
      local git_info = vim.b.gitsigns_status_dict
      if not git_info or git_info.head == "" then
          return ""
      end

      local head    = git_info.head
      local added   = git_info.added and (" +" .. git_info.added) or ""
      local changed = git_info.changed and (" ~" .. git_info.changed) or ""
      local removed = git_info.removed and (" -" .. git_info.removed) or ""
      if git_info.added == 0 then added = "" end
      if git_info.changed == 0 then changed = "" end
      if git_info.removed == 0 then removed = "" end

      if not state.show_branch then
          head = hl(config.placeholder_hl, config.icons.branch_hidden)
      end

      return table.concat({
          "[ ",
          head,
          added, changed, removed,
          "]",
      })
  end

  Statusline = {}

  function Statusline.active()
      return table.concat {
          "[", filepath(), "%t] ",
          git(),
          "%=",
          "%y [%P %l:%c]"
      }
  end

  function Statusline.inactive()
      return " %t"
  end

  function Statusline.toggle_path()
      state.show_path = not state.show_path
      vim.cmd("redrawstatus")
  end

  function Statusline.toggle_branch()
      state.show_branch = not state.show_branch
      vim.cmd("redrawstatus")
  end

  vim.keymap.set("n", "<leader>sp", function() Statusline.toggle_path() end, { desc = "Toggle statusline path" })
  vim.keymap.set("n", "<leader>sb", function() Statusline.toggle_branch() end, { desc = "Toggle statusline git branch" })

  local group = vim.api.nvim_create_augroup("Statusline", { clear = true })

  vim.api.nvim_create_autocmd({ "WinEnter", "BufEnter" }, {
    group = group,
    desc = "Activate statusline on focus",
    callback = function()
      vim.opt_local.statusline = "%!v:lua.Statusline.active()"
    end,
  })

  vim.api.nvim_create_autocmd({ "WinLeave", "BufLeave" }, {
    group = group,
    desc = "Deactivate statusline when unfocused",
    callback = function()
      vim.opt_local.statusline = "%!v:lua.Statusline.inactive()"
    end,
  })
  ```
</details>

## Conclusion

As you can see, customizing your statusline in Neovim is not that difficult. It might take a while, but you use Neovim, you like to spend time doing this kind of things 😉.

Leave a **comment** 💬 and/or a **reaction** 🎉 if you liked it, and share it 🙌 if you think it's usefull.

**Thanks for reading so far!**

## Links

- [nvim](https://github.com/vieitesss/nvim) (my Neovim configuration).
- [gitsigns.nvim](https://github.com/lewis6991/gitsigns.nvim)

