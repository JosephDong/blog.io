---
layout: post
title: git如何删除tag
date: 2018-01-30
categories: Git
tags: [Git,tag]
description: 批量实现tag删除。
---
### 0x1 摘要
由于项目开发时分QA、预发、线上环境，针对不对环境会打不同的tag，导致代码库中存在非常多的tag，除了线上环境，其他环境的tag使用完后可以清理，最近项目需要对代码库中的tag进行清理，从网上查阅文章后进行总结记录。

### 0x2 查看tag
git分本地和远程两个环境，两个环境查看tag命令有所不一样。

***本地：***
<br>使用`git tag`命令，输出如下：
```
qa_v3.0_20180117_01
qa_v3.0_20180117_02
qa_v3.0_20180117_03
```
***远程：***
<br>使用`git show-ref`或`git show-ref --tag`命令，输出如下：
```
c8a47084b280ecb1a408abfaa80ad58772efd596 refs/tags/qa_v3.0_20180117_01
fd4289484e05167bfff935235b7db41a59a03b2f refs/tags/qa_v3.0_20180117_02
b6fc0b51ecbc250496920f70644ca8e41287104e refs/tags/qa_v3.0_20180117_03
```

### 0x3 删除单个tag
本地和远程删除命令也有所区别。

***本地：***
<br>使用`git tag -d <tag>`命令，示例：`git tag -d qa_v3.0_20180117_03`

***远程：***
<br>使用`git push origin :<tag>`命令，示例：`git push origin :refs/tags/qa_v3.0_20180117_03`

### 0x4 批量删除tag
正如文章开头所说，实际项目中会出现很多的tag，而git原生提供的删除命令只支持单个tag删除功能，为什么没有批量删除命令？可能出于安全考虑吧。但我们每个项目都有上百个tag，不可能一个个去删除，下面介绍通过Linux命令来实现批量删除功能。**注意：此功能慎用，切记确定清楚哪些tag是需要删除的，建议找一个测试项目先实践。**
由于我们项目规范中明确tag名称中加上环境区分，因此所有qa环境的tag必须是`qa_`开头，下面以此为例来讲解如何批量删除`qa`环境所有tag。

***本地：***
<br>使用`git tag | grep 'qa_' | xargs git tag -d`命令，先使用`grep`命令找出包含`qa_`的tag，再使用`xargs`命令传递参数给后面的`git tag -d`命令进行删除，`xargs`命令解释参考：<http://man.linuxde.net/xargs>

***远程：***
<br>使用`git show-ref --tag | grep 'qa_' | awk '/(.*)(\s+)(.*)$/ {print ":" $2}' | xargs git push origin`命令

### 0x5 参考资料
<https://yq.aliyun.com/ziliao/60577><br>
<http://man.linuxde.net/xargs>

