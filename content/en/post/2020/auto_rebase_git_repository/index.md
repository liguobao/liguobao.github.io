---
layout: post
title: Script to Auto-Rebase Multiple Git Repositories
category: git
date: 2016-11-25
tags:
- shell
- git
---
# Auto-Rebase Multiple Git Repositories

When preparing releases, I often need to update several repos/branches before merging. Typing the same commands repeatedly gets old, so here’s a small shell script to rebase multiple repos in one go.

```sh
printf "Start rebase 58HouseSearch.\n"
cd ./58HouseSearch
git checkout master
git pull --rebase origin master
printf "Finish Pull Rebase 58HouseSearch master.\n"
read -p "Press any key to continue."
cd ..

printf "Start rebase hexoforblog.\n"
cd ./hexoforblog
git checkout master
git pull --rebase origin master
printf "Finish Pull Rebase hexoforblog.\n"
read -p "Press any key to continue."
cd ..
```

Notes:

- `printf` prints progress messages
- `read -p` pauses for confirmation between steps
- Output mirrors what you see in Git Bash

