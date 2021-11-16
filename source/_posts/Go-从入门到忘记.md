---
title: Go, 从入门到忘记
url_name: go-startup
date: 2021-11-14 15:21:38
hidden: true
categories:
  - Go
tags:
  - 入门
  - 基础知识
---

差不多整整一年前，当时在云组做了一个[Docker 和 Kubernetes 的培训](https://www.dunbreak.cn/2020/12/31/docker-and-kubernetes/)，当时就发现，云原生相关的组件几乎都是用 Go 写的，本着努力成为一个云原生程序员（笑）的目标，当时夸下海口说未来半年的考评任务要学会 Go 去看 Kubernetes 源码，然后就没有然后了……

<!--more-->

Go 最开始吸引我的是它号称`互联网时代的C语言`，第一次听到这个宣传语时，我下意识的以为 Go 是一门和 C 语言一样性能强大并且且可以直接操作硬件的语言。好家伙，学了之后才发现这句话的意思是说 Go 和 C 一样原始……

![hello-go](https://image.dunbreak.cn/go/hello-go.jpg)

作为一个 C#er，第一次看到 Go 的语法真是难受的浑身不爽，先不说别的，就单看这个 main 函数我就能发现奇怪的宝藏：`printf` + `%s`这种语法我可能大学毕业就再也没有用过了，然而这种过时的语法既然还能在 Go 这种 2007 年才发布的语言中看到，并且这种写法还是唯一支持的写法……

加之想到之前由于没有学习 Go Module 相关的内容导致我一个 main 函数都跑不起来的尴尬过往，说实话一年前夸下海口之后我是打算放弃的。

直到今年下半年，Barry 说要我做一个 Go 语言的培训我才重新开始学习 Go（果然 Deadline 才是第一生产力）。不过还好，当把所有 Go 的语法都学完并且自己动手写了一个小项目之后，发现 Go 语言中独特的抽象设计完全是给我打开了一扇新的大门，并且一切极简自己动手的风格也完全契合我想自己动手写点东西提高自己的想法，果然人人都是王境泽。

## 还是从头说起

果然我就是一个学任何东西都喜欢刨根问底的人（看书必看序，学习必须从诞生开始…），所以最开始还是来看看 Go 到底是如何诞生的。

话说早在 2007 年 9 月的一天，Google 工程师 [Rob Pike](https://en.wikipedia.org/wiki/Rob_Pike) 和往常一样启动了一个 C++项目的构建，按照他之前的经验，这个构建应该需要持续 1 个小时左右。这时他就和 Google 公司的另外两个同事 [Ken Thompson](https://en.wikipedia.org/wiki/Ken_Thompson) 以及 [Robert Griesemer](https://en.wikipedia.org/wiki/Robert_Griesemer) 开始吐槽并且说出了自己想搞一个新语言的想法。当时 Google 内部主要使用 C++构建各种系统，但 C++复杂性巨大并且原生缺少对并发的支持，使得这三位大佬苦恼不已。

![authors](https://image.dunbreak.cn/go/authors.png)

第一天的闲聊初有成效，他们迅速构想了一门新语言：能够给程序员带来快乐，能够匹配未来的硬件发展趋势以及满足 Google 内部的大规模网络服务。并且在第二天，他们又碰头开始认真构思这门新语言。第二天会后，Robert Griesemer 发出了如下的一封邮件：

![plan-email](https://image.dunbreak.cn/go/plan-email.webp)
