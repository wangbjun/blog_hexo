---
title: Git最简单的基础入门教程
date: 2018-02-10 12:05:45
tags: Git
category: Linux
---

现在很多公司都用git来管理代码，老一些的项目可能还在用svn，git比svn好的地方就在于其便利的分支管理功能，特别适用于多人协作开发，当年祖师爷linus开发git就是为了方便Linux操作系统的开发。

git的基本用法很简单: 拉代码、提交变更、推代码！大部分公司都有自己内部的git服务器，一般都是使用gitlab，主要是安全和省钱，当然也有公司直接使用github的付费服务！不管咋样，你都需要拿到一个项目的git地址, 为了方便演示，我在github上面创建了一个演示的仓库，里面目前只有一个README.md文件：

<!--more-->

## 1. git clone
首先，你需要使用 **git clone** 拷贝一份项目代码到你自己的电脑，这个命令很简单就不多说了！
```bash
jwang@jwang:~$ git clone https://github.com/wangbjun/git_demo.git
Cloning into 'git_demo'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), done.
Checking connectivity... done.
```

## 2. git pull
前面那步clone代码到本地之后那就可以写你自己的代码了，不过在你提交代码前我强烈建议你先更新一下代码！而且每次开始写代码之前最好都先pull一下，这样可以减少冲突，就算有冲突也可以提前发现解决！

有些人长时间不pull，到最后过了很多天提交的时候一大堆冲突，根本没法merge，很坑，所以我建议大家有空就pull，绝对是没毛病的！

## 3. git status
改完之后当然要提交代码了，使用 **git status** 可以显示有哪些文件有修改！

```shell
jwang@jwang:~/git_demo$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   README.md

no changes added to commit (use "git add" and/or "git commit -a")
```
## 4. git add
如果你改动了多个文件但是你只想提交其中的某几个文件，你就需要使用 **git add** 命令添加改动的文件，在这个例子里面，就是 ```git add READEM.md```。

## 4-1. git checkout
如果你不想提交改动的文件，而且想撤销之前自己的更改，那你就可以使用 **git checkout** 命令, 在这个例子里面，就是 ```git checkout READEM.md```。

## 5. git commit
这是紧接着第4步的，假设你已经使用 **git add** 命令添加了自己需要提交的文件，这时候就需要使用 **git commit** 来提交自己的修改，通常执行这个命令会弹出一个对话框让你添加提交信息，提交信息就是相对于一个备注吧！
![](https://ww1.sinaimg.cn/mw690/5f6e3e27ly1fyop1wo02hj20ru0g475z.jpg)

在Linux下面默认使用的是nano编辑器，很多人看到这个对话框会很懵，不知道咋用，这和vim的操作完全不一样，但也不难，直接输入你想写的内容，然后按 **Ctrl+X** 就会弹出一个选项，按 **Y**，最后回车就可以了

如果你实在不习惯这个编辑器，可以更改成vim，使用 ```git config --global core.editor vim``` 命令，如果你连vim都不会用。。。我建议你可以不用看下去了，下载一个图形化界面的工具吧，或者使用IDE也行，比如idea，eclipse都有自带git插件可以使用。

有一个小操作，假如你修改了很多文件，而且都需要提交，你就不必一个个 **git add**，跳过第4步，直接使用 ```git commit -a```即可。

## 6. git push
最后一步，如果你只需本地使用git，这步就不需要了，但是大部分时候我们需要把自己的修改提交到远程仓库，让别人也能拉取看到，这时候我们就需要使用 ```git push``` 命令推代码。
```shell
jwang@jwang:~/git_demo$ git push
warning: push.default is unset; its implicit value has changed in
Git 2.0 from 'matching' to 'simple'. To squelch this message
and maintain the traditional behavior, use:

  git config --global push.default matching

To squelch this message and adopt the new behavior now, use:

  git config --global push.default simple

When push.default is set to 'matching', git will push local branches
to the remote branches that already exist with the same name.

Since Git 2.0, Git defaults to the more conservative 'simple'
behavior, which only pushes the current branch to the corresponding
remote branch that 'git pull' uses to update the current branch.

See 'git help config' and search for 'push.default' for further information.
(the 'simple' mode was introduced in Git 1.7.11. Use the similar mode
'current' instead of 'simple' if you sometimes use older versions of Git)
```
请注意上面一些提示，其大概意思是自从 git 2.0版本开始，默认使用 "simple" 模式提交代码，simple模式是只会把代码提交到你 **git pull** 命令拉取代码的分支。其实意思就是你从哪个分支拉取的代码就会默认push到哪个分支，一般情况下我们不需要更改这个。

## 总结：
其实最常用的也就是这几个命令，**git clone** 只需要最开始执行一次，平时用的最多的就是 **git commit** 和 **git push**，只要掌握这几个命令就可以了。

当你使用IDE或者一些图形化界面工具时更简单，比如我常用的PHPStorm (idea全家桶快捷键都一样), 快捷键 **Ctrl+T** 就是pull，**Ctrl+K** 可以列出所有修改文件，默认勾选所有修改过的文件，填一下提交信息，回车就是commit了。然后 **Ctrl+Shift+K** 就是push代码，如果不需要修改默认设置，直接回车就行，熟练操作的话非常方便，比使用命令行的效率高很多。

![](https://ww1.sinaimg.cn/mw690/5f6e3e27ly1fyoqxam6p5j20sd0o3gpt.jpg)

使用IDE还可以非常方便的查看历史记录、reset代码、合并分支、对比代码，但是命令行也是需要掌握的，毕竟有时候在服务器上面可木有图形化界面工具。。。

接下来，我会继续给大家讲讲git分支相关的操作！
