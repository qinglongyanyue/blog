---
layout: default
title:  "Google代码平台分析"
date:   2016-12-07
categories: devops
---

## 背景
2016年7月，Google 发布代码管理系统的论文，宣称把所有的代码都放在一个库中，而不是像github，为每个项目分别建立独立的代码库。

想象一个几万的人操作同一个git库的场面，画面不忍直视啊。github明显搞不定这个事情。

国内各家科技媒体也都做了报道，[面对 20 亿行代码，Google 如何管理？](https://linux.cn/article-6247-1.html)

主要内容讲的非常清楚，但是难度是巨大的，毕竟数万人用一个库，而且跨越世界各地。

所以必须深入看下google论文--[Why Google Stores Billions of Lines of Code in a Single Repository](http://www-scf.usc.edu/~csci572/papers/2016GoogleRepository.pdf)

## Google代码库系统
Piper & CitC：

- Piper存储一个巨大的代码库，老版本over在bigtable之上，新版本over在spanner之上
- Piper分布在Google全球超过10个数据中心，依赖Paxos算法保证副本的一致性
- 缓存和异步操作会屏蔽跨地域的延迟

Piper工作流：

- 要修改的文件现在本地创建备份，存放在本地私有仓库，类似git本地clone，兼容git操作
- 本地的git库可以做个快照，方便其他兄弟review，review之后直接提交到代码库
- 开发者通过FUSE挂载整个代码库，称为CitC，他们的私有库只是整个大仓库的一个目录
- CitC支持代码浏览，以及各类标准linux工具，无需clone
- 任何人可以修改任何代码，修改先暂存在本地
- 所有的写操作都存储为一个快照，方便review，随时回退
- 全世界任何角落都可以通过FUSE挂载上整个代码库

基于trunk的研发模式：
- 大量用户工作在代码的head，或者trunk或者叫做mainline。
- 所有的改变都是序列化的（没有分支管理），Google基本不会长出多个老的分支，大家都在一个分支上玩，几乎没有并行分支。
- 在代码提交之后，立刻可以被所有人看到
- 避免了长期老分支的管理，merge开销
- 解决老的release的代码也是先合入主干，在合入release
- 开发新特性时，新代码和旧代码并存（可以control），但是没必要起新分支；同时可以方便在新旧特性间切换；旧代码保留一段时间自动清理，而不是像我们立刻删除特性分支
- google能够很方便把流量切换到新代码，方便做AB测试

待续。。