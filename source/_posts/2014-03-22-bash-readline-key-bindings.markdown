---
layout: post
title:  "Configuring Bash input"
categories: bash tools
comments: true
---

Configuring your shell in order to reduce the activation cost of frequent actions is key to both productivity and pleasure in working. In this post, I'll list a few of the _input_ configurations I found useful for my practice.

Editing the command line in Bash is powered by the Readline library. As always, taking some time to read the [relevant section of the Bash manual][Manual] will teach you a few things worth knowing. In that post, I'll assume you've read the two short first sections:

- [Introduction and Notation]:	  	Notation used in this text.
- [Readline Interaction]:	  	The minimum set of commands for editing a line.

There are three things to know to configure your input.

1. configuration is done in `~/.inputrc` file
2. this file can be reloaded by pressing `C-x C-r`
3. you can list all the bindings with the command `bind -p`

Being efficient with Readline means knowing the existing function that exists, and binding them to shortcut that suits you. Let's start with a simple example. Edit your `~/.inputrc` and add the following lines:

``` bash
# Handy commands for checking readline configuration.
# Use bash builtin `bind -p` command to list key bindings.
"\C-hf": dump-functions
"\C-hv": dump-variables
```

Now press `C-x C-r` to reload that file. Beware that there is no feedback that this has happened. Now check that the new bindings are active.
{%  terminal %}
$ bind -p | grep dump
"\C-hf": dump-functions
# dump-macros (not bound)
"\C-hv": dump-variables
{%  endterminal %}

Now press **Ctrl+h v** to see the configurations of variables. Explore the [Manual] for meaning of these functions and variables, and which one can be handy to you. 

## Moving between words

In my text editor, the keys **Ctrl+-←** and **Ctrl+→** are used to move between words. I expect the same when working on the command line. Default bash keyboard shortcut for that is `\eb` and `\ef` (read **Alt+b** and **Alt+f**). To reconfigure it, put the following in your `~/.inputrc` file.

``` bash
# Move cursor with Ctrl left/right keys.
"\e[1;5C": forward-word
"\e[1;5D": backward-word
```

The syntax for keyboard shortcuts can be found in the [Manual]. There are other useful moving commands. For example, you can _jump_ to a given character forward or backward

## Reusing previous command's argument

Consider the following workflow:

1. type `git status` to see which file has changed
2. type `git diff path/to/one/of/the/file.txt` to see what has changed
3. type `git add path/to/one/of/the/file.txt` to stage this change

Having to type twice the path to the file - even with good autocompletion - is tedious. This is where the `yank-last-arg` becomes very handy. By default, it is bound to `\e.` and `\e_`. For the second command, just type `git add` and then hit `\e.`  (read **Alt+.**) and enjoy! 

Personally, the `Esc` key in keybindings is not handy (too much trajectory on the keyboard for my left pinky!). On Linux terminals, it's ok because the `Alt` key is an equivalent. But on my Mac laptop, that does not work natively. Luckily, I'm using [iTerm2] which can be configured to do so. Either read [this post](http://thinkingeek.com/2012/11/17/mac-os-x-iterm-meta-key/), or simply go to the Preferences and edit as shown on image below:

![iTerm preferences]({{ site.url }}/assets/iterm_preferences.png)

Using the last command argument is the most command use case. But sometimes you want to reuse several arguments. Consider the situation where you want to rename a C++ class, and hence want to rename both the `.h` and the `.cc` files. You will have to type the following commands:

{% terminal %}
$ git mv path/to/old_class.h path/to/new_class.h
$ git mv path/to/old_class.cc path/to/new_class.cc
{% endterminal %}

Once again that's tedious. And once again, there is a command for that. It's named `yang-nth-arg`, which by default is bound to `\eC-y` (read **Alt+Ctrl+y**). However, the keyboard shortcut is more involved as it requires using [Readline Arguments](http://www.gnu.org/software/bash/manual/html_node/Readline-Arguments.html), that is specifying a number before hitting the `\eC-y` combination. In our example, this would be:

- type `git mv`
- then **Alt+2 Alt+Ctrl+y** to copy `path/to/old_class.h` which is arg. **2** of prev. line
- then **Backspace** and `cc` to change the suffix
- then **Alt+.** to copy `path/to/new_class.h` 
- and again **Backspace** and `cc` to change the second suffix. 

Of course, you can also just use bash history to bring the previous command and move the cursor along the line to edit. 

## Expanding glob

Shell globs are useful to refer to a bunch of different files. For example, if you want to delete all `.o` file in a directory, you can just 
go:

{% terminal %}
$ rm *.o
{% endterminal %}

However, it is potentially risky. I once (true story) deleted an entire directory because I typed `rm * bak` with an unfortunate space. To avoid running into that again, when a command has risky side effect, I _expand_ the glob using `"\C-x*": glob-expand-word`. It is also useful if I want to get an expanded glob, but without one file. Let's say I want to delete all text files but README.txt. I would:

- type `rm *.txt`
- hit `C-x*` to expand the whole list of files matching that glob
- move along the command line to remove README.txt
- hit Enter.

## Expanding history

Bash history is very handy, and you definitely want to have the [Bash History Cheat Sheet
](http://www.catonmat.net/download/bash-history-cheat-sheet.pdf) quickly accessible. For example, typing `!ma` and Enter at the prompt will re-execute the last command starting with `ma`. But suppose that command was `make my_program.debug` and you want to instead run `make my_program.opt`. This can be done using the `magic-space` function. It has no default keybinding, so add the following to your `~/.inputrc`:

``` bash
# Expanding a command from history.
"C-x!": magic-space
```

Now, you just have to type `!m`, followed by `C-x!` and then move along the command line and edit it has you want. Much faster than searching through the history with the arrow keys.

## Arbitrary macro

It is actually possible to bind a macro, that is a sequence of keys, to a key. Here are two toy examples related to quotes:

``` bash
# Prepare to type a quoted word --
# insert open and close double quotes
# and move to just after the open quote
"\C-x\"": "\"\"\C-b"
# Quote the current or previous word
"\C-xq": "\eb\"\ef\""
```

Here is another example which I use a lot. Earlier, we've seen how to re-use last argument from the previous command with `\e.`. But what if I want to reuse the last argument of _the current command_? For example, assume you want to type

{% terminal %}
$ mv path/to/some/file.txt path/to/some/file.txt.bak
{% endterminal %}

It's tedious to type the path twice. Add this to your `~/.inputrc`:

``` bash
# Bind to \e, a macro copying the last argument of current line.
# (use an intermediate binding to access function copy-backward-words)
"\e\e,": copy-backward-word
"\e,": "\e\e,\C-y"
```

And that's it, you now need to type the path once and then hit **Alt+,**!

## Conclusion


There are other powerful Readline functions for editing, moving, etc. but I don't need them frequently enough to have integrated them. I just know they exist so if I need it, I'll read the [Manual] to figure out how to use them. But the features listed in that post are now engraved in my brain as I use them all the time. And I've put my `.inputrc` file as [a Gist on Github](https://gist.github.com/Xadeck/9710435#file-inputrc) so that I can put it on every computer I use.

[Manual]: https://www.gnu.org/software/bash/manual/html_node/Command-Line-Editing.html#Command-Line-Editing
[Introduction and Notation]: https://www.gnu.org/software/bash/manual/html_node/Introduction-and-Notation.html#Introduction-and-Notation
[Readline Interaction]: https://www.gnu.org/software/bash/manual/html_node/Readline-Interaction.html#Readline-Interaction
[iTerm2]: http://www.iterm2.com/#/section/home

