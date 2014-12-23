---
layout: post
title: "使用git rebase让历史变得清晰"
date: 2014-12-23 16:10
comments: true
categories: [Notes]
published: true
---

想不想让你的提交历史从左图变成右图？

![](/images/git-rebase/rebase-result.png)

那就来了解一下git rebase吧！

## “Merge branch”提交的产生

我们的工作流程是：修改代码→提交到本地仓库→拉取远程改动→推送。正是在git pull这一步产生的Merge branch提交。事实上，git pull等效于get fetch origin和get merge origin/master这两条命令，前者是拉取远程仓库到本地临时库，后者是将临时库中的改动合并到本地分支中。

要避免Merge branch提交也有一个土法，就是先pull，再commit，最后push。不过万一commit和push之间远程又发生了改动，还需要再pull 一次，就又会产生Merge branch提交。

## 使用git pull --rebase

修改代码→commit→git pull --rebase→git push。也就是将get merge origin/master替换成了git rebase origin/master，它的过程是先将HEAD指向origin/master，然后逐一应用本地的修改，这样就不会产生Merge branch提提交了。具体过程见下文扩展阅读。

<!-- more -->

使用git rebase是有条件的，你的本地仓库要“足够干净”。最干净的是这种：

```
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working directory clean
```

即本地没有任何未提交的改动。次干净的是这种：

```
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Untracked files:
  (use "git add <file>..." to include in what will be committed)
    test.txt
nothing added to commit but untracked files present (use "git add" to track)
```

即本地只有新增文件未提交，没有改动文件。目前看下来大家这点都不太符合，所以git rebase不一定用得起来。当然，我们也可以用git stash来解决这个问题，有兴趣可自行搜索。

## 修改git pull的默认行为

每次都加--rebase似乎有些麻烦，我们可以指定某个分支在执行git pull时默认采用rebase方式：

```
$ git config branch.master.rebase true
```

如果你觉得所有的分支都应该用rebase，那就设置：

```
$ git config --global branch.autosetuprebase always
```

这样对于新建的分支都会设定上面的rebase=true了。老分支还得单独配。

## 扩展阅读[1]：git rebase工作原理

先看看git merge的示意图：

![](/images/git-rebase/merge.png)

*图片来源：https://www.atlassian.com/ja/git/tutorial/git-branches*

可以看到some feature分支的两个提交通过一个新的提交（蓝色）和master连接起来了。

再来看git rebase的示意图：

![](/images/git-rebase/rebase-1.png)

![](/images/git-rebase/rebase-2.png)

Feature分支中的两个提交被“嫁接”到了Master分支的头部，或者说Feature分支的“基”（base）变成了 Master，rebase也因此得名（它的中文叫法请自行YY）。

## 扩展阅读[2]：git merge --no-ff

技术部在开发项目时都会用分支来做，合并时会执行：

$ git checkout feature-branch
$ git rebase master
$ git checkout master
$ git merge --no-ff feature-branch
$ git push origin master

历史就成了这样：

![](/images/git-rebase/no-ff.png)

可以看到，Merge branch 'feature-branch'那段可以很好的展现出这些提交是属于某一特性的。

