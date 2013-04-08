---
title: bracketed paste mode
permalink: blog/bracketed-paste.html
layout: post
fuzzydate: April 2013
---

One of the least well known, and therefore least used, features of many terminal
emulators is [bracketed paste
mode](http://www.xfree86.org/current/ctlseqs.html#Bracketed Paste Mode). When
you are in bracketed paste mode and you paste into your terminal the content
will be wrapped by the sequences `\e[200~` and `\e[201~`.

<aside>For example, let's say I copied the string `"echo 'hello'\n"` from a
website. When I paste into my terminal it will send `"\e[200~echo 'hello'\n\e[201~"`
to whatever program is running.</aside>

I admit this is hard to get excited about, but it turns out that it enables
something very cool: programs can tell the difference between stuff you type
manually and stuff you pasted.

Why is that cool?

Lots of terminal applications handle some characters specially: in particular
when you hit your enter key it sends a newline. Most shells will execute the
contents of the input buffer at that point, which is usually what you want.
Unfortunately, this means that they will also run the contents of the input
buffer if there's a newline in anything you paste into the terminal.

I'm clumsy and often paste random stuff into my terminal; there was also a neat
[proof of concept](http://thejh.net/misc/website-terminal-copy-paste) for how to
make usually not-clumsy people do the same. This can obviously be dangerous as
your shell has the ability to do all kinds of things that you don't want to
happen by accident.

For a while I've been running with bracketed paste mode enabled to protect me
from myself, and I figured it was time to share this support more widely. I take
no credit for writing any of this code; as clichéd as it sounds, I copy-pasted
it from the internet :).

<figure>
[safe-paste](https://github.com/robbyrussell/oh-my-zsh/pull/1698) for oh-my-zsh
========================

The safe-paste plugin for [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)
is built out of the code I originally got from [Michael
Magnusson](http://www.zsh.org/mla/users/2011/msg00367.html). It turns off
running lines of input during pastes. This means that nothing you paste into
your shell, either deliberately or accidentally, will run until you manually hit
the enter key. This gives you time to double-check what you pasted before your
computer gets totally destroyed.
</figure>

If you use `oh-my-zsh` you can install this by upgrading to the latest version
and adding `safe-paste` to the array of plugins in your `~/.zshrc`. If you just
use vanilla zsh, then you can just copy [the
code](https://github.com/robbyrussell/oh-my-zsh/blob/master/plugins/safe-paste/safe-paste.plugin.zsh)
into your `~/.zshrc`.

<aside>By the way, if anyone knows a better fix for bash users than "switch to zsh", I'd
like to include it here.</aside>

[vim-bracketed-paste](https://github.com/ConradIrwin/vim-bracketed-paste)
===================

Vim also handles newlines specially. When I am manually typing out code, vim
will add extra space characters each time I hit return so that the resulting
indentation is correct. Unfortunately when pasting into vim these extra spaces
end up making a mess because the content I'm pasting already includes the
correct indentation.

The usual work around for this is to manually run `":set paste"` inside vim
before pasting, but I often forget. The
[vim-bracketed-paste](https://github.com/ConradIrwin/vim-bracketed-paste)
plugin uses code from [Chis
Page](http://stackoverflow.com/questions/5585129/pasting-code-into-terminal-window-into-vim-on-mac-os-x/7053522#7053522)
to do this automatically for me, so the content I paste into vim does not get
automatically indented but the lines I type manually still do. You can install
this plugin using pathogen, or by copy-pasting [the
code](https://github.com/ConradIrwin/vim-bracketed-paste/blob/master/plugin/bracketed-paste.vim)
into your `~/.vimrc`.

Technical details
=================

If you want to write a program that handles bracketed paste mode yourself you
have to do a bit of extra work.  Bracketed paste mode isn't enabled by default
because most programs make a mess of the extra escape sequences. To enable it
you need to output the sequence `\e[?2004h` to STDOUT when your program
starts. As bracketed paste mode is global to the terminal window you need to
remember to turn it off again when your program exits. To do this output the
sequence `\e[?2004l` to STDOUT.

In summary:

1. Enable bracketed paste: `printf "\e[?2004h"`
2. Wait for paste to start: you'll see `\e[200~` on STDIN.
3. Wait for paste to stop: you'll see `\e[201~` on STDIN.
4. Disable bracketed paste: `printf "\e[?2004l"`

<aside>If you want more detail, there's a thorough (and thoroughly inscrutable)
reference on [xfree86.org](http://www.xfree86.org/current/ctlseqs.html).</aside>

Terminal support
================

For a long time bracketed paste mode was only supported by xterm and its
derivatives (urxvt, etc.), but as of recently (a year or so ago) support has
been added to most other terminals, including libvte which powers
gnome-terminal et.al., iTerm 2 and Terminal.app. In other words, almost every
terminal in widespread use now suppports bracketed paste mode.

If you maintain a program that runs inside a terminal emulator, now is
definitely the time to consider adding support for bracketed paste mode. Even
little niceties like waiting for an extra newline after pasting can make the
user feel much more in control.
