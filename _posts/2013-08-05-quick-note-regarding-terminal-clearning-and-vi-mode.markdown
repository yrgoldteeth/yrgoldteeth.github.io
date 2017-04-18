---
layout: post
title: Quick Note Regarding Terminal Clearing and vi Mode
---
I have switched from using emacs keybindings in my terminal emulator
into using vi (`set -o vi`), which feels pure and holy.  The only problem is I was
missing the screen clearing that `Control-L` affords.  Fix this by
creating a `.inputrc` file in your home directory with the following
line (or appending it to your existing `.inputrc`):

    bind -m vi-insert "\C-l":clear-screen

