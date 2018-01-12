---
title: learn tensorflow 1
date: 2018-01-03 21:48:53
tags: AI tensorflow
---

# 简介

AI 的烈火烧遍全球，作为程序猿一枚，自然不能坐视不理。趁着 tensorflow lite 刚发布之际，趁热打铁，学习一下这个呼声颇高的框架。本文使用的版本：
```
    tensorflow: 136697ecdc64b5171522fb7f89cfe51a02f0f1c1@master, (1.4.1之后的一个commit)
    bazel: 0.8.1
```
<!--more-->
## 完善的示例

例子散落在源码中，要编译这些例子需要了解一些 bazel 的知识。本人对 bazel 比较抵触，但为了弄懂 tensorflow 代码的组织，也还是看了一些 bazel 资料，需要弄懂 WORKSPACE, BUILD, RULE, *.bzl 等文件及概念。

## 丰富的教程和文档

tensorflow 网站上的API文档，教程，源码库中的示例程序非常完备很齐整，找资料不会太费劲。社区很活跃，不像学术派的caffe，似乎更新很慢。

## 完善的工具

常规的模型导入，导出，转换的python工具，还有比较惊艳的 tensorboard. 在训练 caffe 模型时，通过 python 脚本分析日志文件，输出各种图表，不够直观。而tensorboard 把这件事情做的很美观简洁。

## tensorflow serving

google 为了做cloud生意也是拼了，怎样传输模型也替你想好了，tensorflow serving 有一整套工具和API帮你实现这一步。

## 不同层级的API并存

tensorflow 官方教程是从最基础的 API 讲起，教你怎么创建 Variable, 怎么创建 Op，怎么构建 Graph，这些是最核心的功能，在 core 包中。而用过 Caffe 的人都知道，直接在这一层API上实现网络各层是比较繁琐的，于是有 layer API。而显然 google 还有其他团队在研究网络算法打榜，在此核心之上开发了 TF Slim 库，贡献给了 TF, 这块代码在 contrib 包中。使用 TF Slim 的好处在于 google 基于这个框架发布了很多现成的模型。

同时，TF 核心也发布了在我看来比 TF Slim 更上层的 Estimator API，可以灵活切换不同的模型进行比较。

除此之外，跨平台的 keras API 也被集成到了 tensorflow 的 core 中，对 keras 熟悉的人，可以从这一层 API 开始。本人对 keras 并不感冒，因此没有太关注。

对于从 Caffe 切换过来的人来说，TF Slim 是一个不错的切入点，从这一层API开始比较顺理成章，不会写很多冗余代码，同时也保留了一定的灵活性，还有google已经训练好的基础模型作为起点。不过TF Slim 位于 contrib，未来能不能进入 core，并不清晰，况且core 中有 layer API 和它相似。

# 小结

本文对 tensorflow 和几个 API 作了简单的小结，下一篇blog看看 TF Mobile 和 TF Lite 两个框架。
