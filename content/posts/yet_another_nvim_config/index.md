+++
title = "Yet Another (N)Vim Config"
date = 2026-03-22
+++

# Introduction

You can find my configs for everything
[here](https://github.com/threexc/configs).
# The Basic Vim Config

I've used Vim since about 2018. Over these last eight years, my Vim config has
changed very little. Here's the entire .vimrc as of today:

```
" Make sure Python virtualenvs don't throw a warning when starting vim.
if exists('$VIRTUAL_ENV')
    let &pythonthreedll = system("find /usr/lib64 -name 'libpython3.*.so.1.0' | tail -1 | tr -d '\n'")
    python3 << EOF
import sys
import sysconfig
sys.path.insert(0, sysconfig.get_path('purelib', vars={'base': '/usr', 'platbase': '/usr'}))
EOF
endif

" Uncomment this instead if using neovim:
" if exists('$VIRTUAL_ENV')
"    let g:python3_host_prog = '/usr/bin/python3'
" endif

execute pathogen#infect()
syntax on
colorscheme desert

" Use filetype detection and file-based automatic indenting
if has('filetype')
    filetype plugin indent on
endif

set autoindent          " automatically indent
set hidden              " allow switching buffers without saving
set hlsearch            " highlight search results
set ignorecase          " case-insensitive search...
set incsearch           " show search results as you type
set mouse=              " disable mouse
set pastetoggle=<F2>    " hotkey for paste mode to avoid extra indentation
set ruler               " show cursor position in status bar (redundant with airline but harmless)
set scrolloff=5         " keep 5 lines visible above/below cursor when scrolling
set smartcase           " ...unless you use uppercase in the search term
set textwidth=80        " wrap width
set updatetime=100      " reduce time between updates from 4000ms to 100ms
set visualbell          " don't beep
set wildmenu            " better tab completion in command mode

if has("autocmd")
    " Use actual tab chars in Makefiles
    autocmd FileType make setlocal tabstop=8 shiftwidth=8 softtabstop=0 noexpandtab
    autocmd FileType c    setlocal tabstop=8 shiftwidth=8 softtabstop=8 noexpandtab
    autocmd FileType python setlocal tabstop=4 shiftwidth=4 softtabstop=4 expandtab
    autocmd FileType rust   setlocal tabstop=4 shiftwidth=4 softtabstop=4 expandtab
    autocmd FileType yaml   setlocal tabstop=2 shiftwidth=2 softtabstop=2 expandtab
    autocmd BufNewFile,BufRead *.v,*.vs setlocal tabstop=4 shiftwidth=4 softtabstop=4 expandtab
endif

let g:airline_powerline_fonts = 1
```

with the following installed as plugins:

- fugitive
- nerdtree
- vim-airline
- vim-gitgutter

# Switching to NeoVim

There have been times when I've made an attempt to switch to NeoVim, but I've
always found the popular distros to be too much even on first install - there
are a lot of changes and new features that feel too different from the default
Vim behaviour I expect on a random system. Now that I use Claude for a bit of
pair programming and clean-up, I asked it about how I could achieve a similar
minimal setup with NeoVim. It gave me this config structure:

```
/home/tgamblin/.config/nvim/
├── init.lua
└── lua
    └── plugins.lua
```

and these files:

`init.lua`
```
-- Bootstrap lazy.nvim
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
    vim.fn.system({
        "git", "clone", "--filter=blob:none",
        "https://github.com/folke/lazy.nvim.git",
        "--branch=stable",
        lazypath,
    })
end
vim.opt.rtp:prepend(lazypath)

-- Load plugins
require("lazy").setup("plugins")

-- Appearance
vim.cmd("colorscheme desert")
vim.opt.number         = true
vim.opt.ruler          = true
vim.opt.visualbell     = true
vim.opt.colorcolumn    = "81"

-- Editing behaviour
vim.opt.autoindent     = true
vim.opt.textwidth      = 80
vim.opt.scrolloff      = 5
vim.opt.hidden         = true
vim.opt.mouse          = ""

-- Search
vim.opt.hlsearch       = true
vim.opt.incsearch      = true
vim.opt.ignorecase     = true
vim.opt.smartcase      = true
vim.opt.wildmenu       = true

-- Performance
vim.opt.updatetime     = 100

-- Filetype-specific indentation
vim.api.nvim_create_augroup("FileTypeIndent", { clear = true })

-- Use Esc to exit terminal mode
vim.api.nvim_set_keymap('t', '<Esc>', '<C-\\><C-n>', {noremap = true})

local function set_indent(pattern, opts)
    vim.api.nvim_create_autocmd("FileType", {
        group = "FileTypeIndent",
        pattern = pattern,
        callback = function()
            vim.opt_local.tabstop     = opts.tabstop
            vim.opt_local.shiftwidth  = opts.shiftwidth
            vim.opt_local.softtabstop = opts.softtabstop
            vim.opt_local.expandtab   = opts.expandtab
        end,
    })
end

set_indent("make",   { tabstop=8, shiftwidth=8, softtabstop=0, expandtab=false })
set_indent("c",      { tabstop=8, shiftwidth=8, softtabstop=8, expandtab=false })
set_indent("python", { tabstop=4, shiftwidth=4, softtabstop=4, expandtab=true  })
set_indent("rust",   { tabstop=4, shiftwidth=4, softtabstop=4, expandtab=true  })
set_indent("yaml",   { tabstop=2, shiftwidth=2, softtabstop=2, expandtab=true  })

-- Verilog files (.v, .vs)
vim.api.nvim_create_autocmd({"BufNewFile", "BufRead"}, {
    group = "FileTypeIndent",
    pattern = {"*.v", "*.vs"},
    callback = function()
        vim.opt_local.tabstop     = 4
        vim.opt_local.shiftwidth  = 4
        vim.opt_local.softtabstop = 4
        vim.opt_local.expandtab   = true
    end,
})
```

`plugins.lua`
```
return {
    -- Statusline
    {
        "vim-airline/vim-airline",
        dependencies = { "vim-airline/vim-airline-themes" },
        config = function()
            vim.g.airline_powerline_fonts = 1
        end,
    },

    -- Git
    { "tpope/vim-fugitive" },
    {
        "lewis6991/gitsigns.nvim",  -- gitgutter equivalent, native neovim plugin
        config = function()
            require("gitsigns").setup()
        end,
    },

    -- tpope essentials
    { "tpope/vim-surround" },
    { "tpope/vim-commentary" },
    { "tpope/vim-repeat" },
    { "tpope/vim-sleuth" },
}
```

It seems like a nice upgrade, but the differences are subtle. Terminal Mode got
me at first. Note this part

```
-- Use Esc to exit terminal mode
vim.api.nvim_set_keymap('t', '<Esc>', '<C-\\><C-n>', {noremap = true})
```

# More?

I think at this point this may be the heaviest I want to make my editor, but
we'll see. Maybe I'll publish it with a setup script as an alternative to the
bigger options. There are also still a lot of shortcuts in Vim I ought to make
much better use of before trying to add extra stuff. There's a great reference
[here](https://vim.rtorr.com/).
