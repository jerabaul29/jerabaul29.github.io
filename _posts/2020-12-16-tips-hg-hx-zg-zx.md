---
layout: post
title:  "Productivity tips: hg, hx, zg, zx"
date:   2020-12-16 12:00:00 +0200
categories: jekyll update
---

When performing daily work, there are a number of very repetitive tasks that quickly add up taking time: moving back and forth between different project folders, finding again the path to a given project folder, finding an old command to repeat it or adapt it for the present needs. Luckily for Linux users, there are some good ways to automate / improve efficiency of these tasks. In the following, I discuss a few of the solutions I use daily. Of course, there may be many other ways to address these workflows, but this is what I like to use.

# Navigating between folders and paths

My favorite tool for navigating between folders and paths is to use the ```z``` script, available on github: [https://github.com/rupa/z](https://github.com/rupa/z) . This is a pure bash script that tracks which folders you work from, and offers a "frecency" ranking of these. In addition, I created a couple of very simple wrappers around ```z``` to make it easier to use in my workflow. These are the ```zg``` (z-Grep) and ```zx``` (z-eXecute) scripts, that are available on my github: [https://github.com/jerabaul29/config_scripts_snippets/tree/main/scripts/z_grep_exec](https://github.com/jerabaul29/config_scripts_snippets/tree/main/scripts/z_grep_exec).

Using ```zg``` and ```zx``` is then very easy. After a bit of using your terminal "as usual" so that ```z``` can learn about your paths, you can use these by doing:

```bash
~$ zg data roms  # search for paths that contain "data" and "roms"
   1  /home/jrmet/Desktop/Data/ROMS_example_data
~$ zx 1  # change path to the entry "1" of the last zg command
cd /home/jrmet/Desktop/Data/ROMS_example_data
~/Desktop/Data/ROMS_example_data$ 
```

# Finding again commands in the terminal history

Another very common pattern is to look for a previously issued command and either repeat it, or use it as an inspiration for the next command to issue. There are several possibilities for this out-of-the-box: for example, use the ```Ctrl-r``` pattern to search for strings in history, and hitting it several times in order to go back step by step further back in history. However, this is a bit slow and inflexible in my opinion. As a result, I ended up quite often using patterns such as:

```bash
~$ history | grep pass | grep generate | grep symbol
 7520  pass generate --no-symbols something/username 12
 8128  pass generate --no-symbols something_else/password
```

In addition to this being a bit many pipe operators, there are also some cases when you may get many duplicated hits. In order to solve this, I wrote a couple of very simple scripts to 1) perform a search a la history grep, 2) remove duplicates, 3) present everything in a way similar to zg and zx. To get the scripts, see: [https://github.com/jerabaul29/config_scripts_snippets/tree/main/scripts/hist_grep_exec](https://github.com/jerabaul29/config_scripts_snippets/tree/main/scripts/hist_grep_exec). Once these are installed, I can perform searches with ```hg``` (history Grep) and ```hx``` (history eXecute) such as:

```bash
~$ hg pass generate symbol  # look for previous commands that contain the strings "pass", "generate", and "symbol"
   1  7520  pass generate --no-symbols something_1/username 12
   2  8128  pass generate --no-symbols  something_2/password
   3  8130  pass generate --no-symbols  something_3/password 18
~$ hx 2  # re-issue the second command
pass generate --no-symbols something_1/username 12
An entry already exists for something_1/username. Overwrite it? [y/N]
```
