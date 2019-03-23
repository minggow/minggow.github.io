---
layout: post
title: Git回滚使用
category: other
tags: [other]
excerpt: Git回滚使用
---
## 前言
看到一则新闻，美国某程序员枪击同事案件，很多网友猜测出了各种激怒码农同事的办法
> 9月19日，一名程序员在美国某办公楼向4名同事开枪，导致一人情况危机，两人伤情严重，一人被子弹擦伤。目前，凶手已死，身份被警方查明。
> 目前，码农持枪杀人的动机仍然是个谜。有人猜测道：“同事不写注释，不遵循驼峰命名，括号换行，最主要还天天 git push -f 等因素” 激怒了这名行凶者。

正常的流程是要在自己的本地解决掉所有的 merge conflict 之后才能 push 到 remote 的。鲁莽的 push -f 确实很容易激怒别人，同时也会很大风险把别人有价值的 commit 都覆盖清空了。


## 如何优雅的在git中回滚？

git reset xxx --hard

git reset origin/master --soft

git commit -am '回滚'

git push

回滚的变动作为一次commit提交，比其push -f git的主线看起来会很好看


## 多版本回滚
参考[多版本回滚](https://stackoverflow.com/questions/1463340/how-to-revert-multiple-git-commits)

## 总结
- 不要用git push -f，珍惜生命
- 想想更好地办法，多考虑其他人

## 参考
- [跪求不要 git push -f](https://www.jianshu.com/p/41cef45ef6ce)
- [How to revert multiple git commits?](https://stackoverflow.com/questions/1463340/how-to-revert-multiple-git-commits)
- [如何优雅的在git中回滚？](https://blog.csdn.net/u012207345/article/details/81449706)