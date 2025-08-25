---
title: My Neovim Setup
draft: true
date: 2025-08-23
---

I've been using neovim as my main development environment for the last ~5 years
[^1], my setup has evolved a lot overtime. This is what I'm using right now and things I've done to make it work better for my workflows. 

I switched to [Nvchad](https://nvchad.com/) with the [falcon](https://nvchad.com/themes/falcon.webp) theme as my base config. I appreciate its speed, sensible defaults, and easy customization. I always have a tmux session open as well. I haven't done a lot of customization of tmux, other than [enabling vim-keybindings](https://github.com/hmcguinn/dot-files/blob/master/tmux.conf#L7), using `C-a` instead of `C-b` as a command prefix, and displaying my active Spotify song to the status bar . 

I use these plugins: 
- [nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter)
- [nvim-tree.lua](https://github.com/nvim-tree/nvim-tree.lua),
- [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim)
- [git-blame.nvim](https://github.com/f-person/git-blame.nvim)
- [prettier](https://github.com/prettier/vim-prettier)
- [copilot.vim](https://github.com/github/copilot.vim)

Things that I've remapped in vim: 
- `jk` to exit insert mode
- `<C-W><C-D>` to see the entire diagnostic line
- `<leader> + go` to `:GitBlameOpenFileURL` to open on Github. Very useful for slacking links to people
- `<leader> + gg` to disable git blame
- `gh` to move cursor to the beginning of the line. I picked this up from using [Helix](https://helix-editor.com/), as my primary editor for a time.

These options:
- `vim.opt.ignorecase = true`
- `vim.opt.smartcase = true`


I like to think I'm pretty good at moving around in vim, but [vim-fu goes deep](https://stackoverflow.com/questions/1218390/what-is-your-most-productive-shortcut-with-vim/1220118#1220118). I'm trying to use these commands more in my daily workflows:

- `grr`: show all references
- `gD`: go to definition
- `<leader> + ra`: renames (I often reach for `:%s/old/new/g` instead)
- `<leader> + ma` : opens marks 
- `m<letter>` : add a mark 
- `'<letter>` : go to mark 
- `<leader> + ds` : opens lsp diagnostics at bottom 
- `<leader> + fm` : format file 
- `:r !date` : inserts `date` (or another bash command)
- `{` or `}` to go to beginning or end of a paragraph 


These resources have helped my understanding of vim/vim setup: 
- [An Experienced (Neo)Vimmer's Workflow](https://seniormars.com/posts/neovim-workflow/)
- [SpaceVim](https://github.com/wsdjeg/SpaceVim)
- [A Vim Guide for Advanced Users](https://thevaluable.dev/vim-advanced/)

My [dot-files](https://github.com/hmcguinn/dot-files) live here. 



[^1]: My first internship gave me a virtual desktop instead of a laptop. I also had to spend a lot of time SSHd into shared research clusters from this virtual desktop, and VSCode with SSH forwarding over a virtual desktop was miserable. I got really good at using vim that summer ([SpaceVim](https://github.com/wsdjeg/SpaceVim) at the time).
