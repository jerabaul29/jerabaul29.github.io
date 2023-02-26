---
layout: post
title:  "Using the Xonsh shell: setup and customizations"
date:   2021-06-28 19:00:00 +0200
categories: jekyll update
---

NOTE: this post is a WIP

As any linux user, I am a fan of the command line. I have been using bash for a long time, and I like it quite well, but I am a bit frustrated by how strange the bash syntax can be, and I miss the possibility to just write small python snippets directly in terminal. Luckily, I have heard recently about the ```Xonsh```language and command prompt, which lets me combine the best of command line tools, bash-like syntax, and python. After giving it a try for a few weeks, I am moving my workflows to it. In the following, I speak a bit about my ```Xonsh``` setup, a few nice tips and tricks, and a few cheatsheet elements.

These are a few scratchbook notes, and in addition / if needed, Xonsh has very good online documentation.

# Installation and setup

Installation is very simple:

```bash
~$ sudo apt install xonsh
```

In addition, I like to use a bit of extra tooling such as dedicated fuzzy finder, for which a bit extra is needed:

```bash
~$ sudo apt install fd-find
~$ pip3 install xontrib-fzf-widgets
```

With all this installed, I am ready to write my ```.xonshrc``` config file, usually some variation around:

```bash
$PROMPT = '{env_name}{GREEN}{user}@{hostname}{BLUE} {cwd}{branch_color}{gitstatus: {}} {BLUE}{prompt_end} '
$XONSH_COLOR_STYLE = 'monokai'

# my config
$VI_MODE = True

# my aliases
aliases['echo_hello'] = 'echo "hello world"'

def git_pull_add_commit_push(args, stdin=None):
    git pull
    git add .
    git commit -m @(args[0])
    git push

aliases['ga'] = git_pull_add_commit_push

# fzf fuzzy finder, from https://github.com/laloch/xontrib-fzf-widgets
xontrib load fzf-widgets
# to make sure the command an dir are well performed correctly
$fzf_find_command = "fdfind"
$fzf_find_dirs_command = "fdfind -t d"
# keybindings
$fzf_history_binding = "c-r"  # Ctrl+R
$fzf_ssh_binding = "c-s"      # Ctrl+S
$fzf_file_binding = "c-f"      # Ctrl+F
$fzf_dir_binding = "c-d"      # Ctrl+D
```

In addition, I like to set up a Xonsh shortcut, for that the simplest on my system (Ubuntu 20.04, but that may be different on yours) is to:

- search for "keyboard shortcuts"
- go to the "custom shortcuts" section, and add a new shortcut with:
  - name: ```xonsh```
  - command: ```gnome-terminal -e xonsh```
  - shortcut: ```ctrl-alt-x``` (or whatsoever else you may like)

# Using Xonsh in daily life



# Short cheatsheet

TODO: inspired from the tutorial.
