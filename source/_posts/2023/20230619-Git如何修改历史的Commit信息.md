---
permalink: 2023/13.html
title: Git如何修改历史的Commit信息
date: 2023-06-19 09:00:00
tags: Git
cover: https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/20230616140555.png
thumbnail: https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/20230616140555.png
categories: technotes
toc: true
description: 最近由于一行单元测试代码没有写 Assert 断言，导致了项目在 CI 过程中没有通过，于是遭到了某位同事的吐槽，在修改我的代码后写上了一句提交信息。我想，做为技术人，修改这条 Commit 信息还是可以的，于是我通过本文介绍的技巧完成了修改。
---

最近由于一行单元测试代码没有写 Assert 断言，导致了项目在 CI 过程中没有通过，于是遭到了某位同事的吐槽，在修改我的代码后写上了一句提交信息。

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230616132530909.png)

我想，做为技术人，修改这条 Commit 信息还是可以的，于是我通过本文介绍的技巧完成了修改，效果如下：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230616133302759.png)

其实修改历史提交信息很简单。

<!-- more -->

## 一、找到该 Commit 前一条的 Commit ID

例如当前有 3 条提交，使用 `git log` 查看。

```
commit e7dc6e4d1001ecff3c1000f82ffffe06859fad61
Author: 张三 <zhangsan@git.com>
Date:   Fri Jun 16 12:25:34 2023 +0800
   fix: 正常的提交信息1

commit e0871dfb91f6a0acc5298d9e1960291629479a46
Author: 李四 <lisi@git.com>
Date:   Fri Jun 16 12:20:08 2023 +0800
   fix: fucking the code

commit e7dc6e4d1001ecff3c1000f82ffffe06859fad61
Author: 张三 <zhangsan@git.com>
Date:   Thu Jun 15 14:32:49 2023 +0800
   fix: 正常的提交信息2
```

我们要修改的 Commit 是第二条，于是我们要找的前一条 Commit ID 就是 `e7dc6e4d1001ecff3c1000f82ffffe06859fad61`。

## 二、git rebase -i 命令

然后执行 `git rebase -i e7dc6e4d1001ecff3c1000f82ffffe06859fad61`，会得到下面的内容：

```
pick e0871dfb91f6a0acc5298d9e1960291629479a46 fix: fucking the code
pick e7dc6e4d1001ecff3c1000f82ffffe06859fad61 fix: 正常的提交信息1

# ...
```

找到需要修改的 commit 记录，把 `pick` 修改为 `edit` 或 `e`，`:wq` 保存退出。也就是：

```
edit e0871dfb91f6a0acc5298d9e1960291629479a46 fix: fucking the code
pick e7dc6e4d1001ecff3c1000f82ffffe06859fad61 fix: 正常的提交信息1

# ...
```

## 三、修改 commit 的具体信息

执行 `git commit --amend`，会得到下面的内容：

```
fix: fucking the code

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
#
# Date:      Fri Jun 16 12:20:08 2023 +0800
```

修改文本内容：

```
fix: fucking me

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
#
# Date:      Fri Jun 16 12:20:08 2023 +0800
```

保存并继续下一条`git rebase --continue`，直到全部完成。接着执行 `git push -f` 推到远端，当然这需要有 Maintainer 权限。

接下来再查看提交日志，`git log`：

```
commit e7dc6e4d1001ecff3c1000f82ffffe06859fad61
Author: 张三 <zhangsan@git.com>
Date:   Fri Jun 16 12:25:34 2023 +0800
   fix: 正常的提交信息1

commit e0871dfb91f6a0acc5298d9e1960291629479a46
Author: 李四 <lisi@git.com>
Date:   Fri Jun 16 12:20:08 2023 +0800
   fix: fucking me

commit e7dc6e4d1001ecff3c1000f82ffffe06859fad61
Author: 张三 <zhangsan@git.com>
Date:   Thu Jun 15 14:32:49 2023 +0800
   fix: 正常的提交信息2
```

修改完成！不过要提醒的是：技巧慎用他途。

## 四、总结一下 git rebase 命令

对于 git rebase 命令，[官方文档](https://git-scm.com/docs/git-rebase) 是这样介绍的：允许你在另一个基础分支的头部重新应用提交。

> git-rebase - Reapply commits on top of another base tip

使用方法：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230616143656476.png)

我们在执行 `git rebase -i` 之后，注释里已经给出了所有的用法：

```
# Rebase 29fc076c..db0768e3 onto 29fc076c (3 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       create a merge commit using the original merge commit's
# .       message (or the oneline, if no original merge commit was
# .       specified). Use -c <commit> to reword the commit message.
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
```

归纳一下：

| 命令   | 缩写 | 含义                                                     |
| ------ | ---- | -------------------------------------------------------- |
| pick   | p    | 保留该commit                                             |
| reword | r    | 保留该commit，但需要修改该commit的注释                   |
| edit   | e    | 保留该commit, 但我要停下来修改该提交(不仅仅修改注释)     |
| squash | s    | 将该commit合并到前一个commit                             |
| fixup  | f    | 将该commit合并到前一个commit，但不要保留该提交的注释信息 |
| exec   | x    | 执行shell命令                                            |
| drop   | d    | 丢弃该commit                                             |

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/202303052135542.gif)

## 封面

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/20230616140555.png)

## 相关文章

- [分享5个Git使用技巧](https://mp.weixin.qq.com/s/ETX8rIwu8Y_0qN-C718gSQ)

