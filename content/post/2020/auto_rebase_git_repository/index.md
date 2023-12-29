---
layout: post
title: 自动同步git repository脚本
category: git
date: 2016-11-25
tags:
- shell
- git
---
# 自动同步git repository脚本

由于平时偶尔需要merge不同分支代码到正式版本用于发布版本，merge前就需要先把各种分支代码更新到最新，接着再去做merge工作。

经常使用的分支其实不算太多，不过仓库倒是有好几个。来来去去写命令行或者GUI操作多了觉得有点繁琐，就琢磨来写个脚本做吧。

PS：偷懒是人类进步的动力...

找了下资料，无外乎就是bat/sh脚本调用git cmd,之前写过bat命令，所以一开始是走这个思路的。

不料在PATH上配置好了git bin的路径之后，使用git命令没问题了，不过pull rebase的时候提示publickey无效。可是我的publickey一直都在.ssh里面,不存在无效的问题...

懒得纠结，换shell吧。

参考资料：

[请问如何写一个批处理自动打开 gitbash，然后自动执行一系列git命令（windows平台）？](https://www.zhihu.com/question/38962022)

## Show Code

使用shell就更好玩了，直接把git bash运行的命令扔到.sh文件里面就完事了。所以...如下：

```shell

printf "Start rebase 58HouseSearch. \r\n"
cd ./58HouseSearch;
git checkout master;
git pull --rebase origin master;
printf "Finish Pull Rebase 58HouseSearch release and master.\r\n"
read -p "Press any key to continue.";
cd ..;

printf "Start rebase hexoforblog;\r\n"
cd ./hexoforblog;
git checkout master;
git pull --rebase origin master;
git checkout master;
printf "Finish Pull Rebase hexoforblog.\r\n"
read -p "Press any key to continue.";
cd ..;

```

说明：

1. printf 为打印函数，就像C语言那样用就好；
2. read -p "Press any key to continue."; 这个是接受输入，结合起来可以做更复杂的行为咯。
3. 输出内容和我们在git bash里面操作是一致的。
