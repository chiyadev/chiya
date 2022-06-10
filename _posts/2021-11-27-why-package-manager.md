---
layout: post
title: Why I wrote my own package manager
tags: coding
---

Check it out on [GitHub][3].

I use [Neovim][1] as my IDE with plenty of plugins for various language integrations and editing tools, but configuring those plugins using existing package managers was always a pain.

Every time I copied my configuration to another development environment, the configuration script would spew a bunch of errors saying that packages were not installed.
Obviously, duh. It's a _new_ environment.

But isn't the point of a package manager to make managing packages easy and painless?
If a package doesn't exist, it should handle that case gracefully instead of filling my screen with errors I'm already aware of.

```lua
require("packer").startup(function()
  use("wbthomason/packer.nvim")
  use({
    "nvim-telescope/telescope.nvim",
    requires = {
      "nvim-lua/plenary.nvim",
      "nvim-treesitter/nvim-treesitter",
    },
    config = function()
      require("telescope").setup({
        -- omitted
      })
    end,
  })
end)
```

This is a portion of my old configuration file for the package `telescope.nvim`, using [packer.nvim][4] as the package manager.

If I copy this configuration to a new environment without any packages installed, I expect the package manager to handle the change correctly.
That is, it should _not_ run the `config` hook, because the package isn't installed. There is _nothing_ to configure.

Instead, when I ran `:PackerSync` in the new environment, what I got is a bunch of errors saying `telescope.nvim` failed to initialize.

```
packer.nvim: Error running config for telescope.nvim: [string "..."]:0: module 'telescope' not found:
^Ino field package.preload['telescope']
^Ino file './telescope.lua'
^Ino file '/usr/share/luajit-2.1.0-beta3/telescope.lua'
^Ino file '/usr/local/share/lua/5.1/telescope.lua'
^Ino file '/usr/local/share/lua/5.1/telescope/init.lua'
^Ino file '/usr/share/lua/5.1/telescope.lua'
^Ino file '/usr/share/lua/5.1/telescope/init.lua'
^Ino file '/home/luaneko/.cache/nvim/packer_hererocks/2.1.0-beta3/share/lua/5.1/telescope.lua'
^Ino file '/home/luaneko/.cache/nvim/packer_hererocks/2.1.0-beta3/share/lua/5.1/telescope/init.lua'
^Ino file '/home/luaneko/.cache/nvim/packer_hererocks/2.1.0-beta3/lib/luarocks/rocks-5.1/telescope.lua'
^Ino file '/home/luaneko/.cache/nvim/packer_hererocks/2.1.0-beta3/lib/luarocks/rocks-5.1/telescope/init.lua'
^Ino file './telescope.so'
^Ino file '/usr/local/lib/lua/5.1/telescope.so'
^Ino file '/usr/lib64/lua/5.1/telescope.so'
^Ino file '/usr/local/lib/lua/5.1/loadall.so'
Press ENTER or type command to continue
```

After many trial and error, I discovered that the fix to this problem is manually commenting out all hooks,
letting packer download the packages first, then uncommenting everything. This became incredibly tedious very quickly.

Interestingly, this meant that packer did not meet the very definition of a package manager according to [Wikipedia][9].

> A _package manager_ or _package-management system_ is a collection of software tools that automates the process of installing,
> upgrading, configuring, and removing computer programs for a computer in a consistent manner.

Clearly, having to comment out hooks and scripts just to get a list of packages installed in a new environment is not an _automated process_.

Note that this problem is not unique to packer. I have encountered similar issues with [paq-nvim][5].

Also, there were many other minor inconveniences with packer, such as having to compile my configuration separately,
no access to [global variables][6], the inability to [disable packages temporarily][7],
and having one giant package list that was becoming almost unmaintainable.

I decided I had enough and wrote my own package manager, naming it [dep][3], with the aim of solving all of the above problems and more.
With dep, I am finally happy knowing that I can restore my full neovim setup in under ten seconds by simply copying my configuration files.

This turned out to be really convenient, because right after having written dep,
I had to uninstall my hackintosh and distro-hop between Arch and Fedora several times while upgrading my main disk.
My package manager has not failed me yet.

Although I wrote dep out of a personal need for a reliable and correct package manager,
I have released it under the permissive [MIT License][8] for anybody else interested to try out.
The repository readme, although brief, should guide you around how you can use it effectively.
As always, if you discover a bug or want to suggest a feature, please open an issue!

[Link to GitHub][3]

[1]: https://neovim.io/
[2]: https://www.lua.org/
[3]: https://github.com/chiyadev/dep
[4]: https://github.com/wbthomason/packer.nvim
[5]: https://github.com/savq/paq-nvim/issues/81
[6]: https://github.com/wbthomason/packer.nvim/issues/683#issuecomment-978351427
[7]: https://github.com/wbthomason/packer.nvim/issues/684
[8]: https://github.com/chiyadev/dep/blob/master/LICENSE
[9]: https://en.wikipedia.org/wiki/Package_manager
