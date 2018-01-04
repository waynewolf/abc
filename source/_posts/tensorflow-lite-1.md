---
title: tensorflow lite 1
date: 2018-01-03 21:48:53
tags:
---

AI 的烈火烧遍全球，作为程序猿一枚，自然不能坐视不理。趁着 tensorflow lite 刚发布之际，趁热打贴，做做写小东西，顺便学习一下这个呼声颇高的框架。

google 非常贴心地在 tensorflow 代码库中加入了 tensorlow-lite android apk, 运行起来豪不费劲，不过我更关心 tensorflow-lite 在嵌入式设备上的推断运行，以及在新场景下的finetune模型。这里把过程记录下来，便于日后查询。

## tensorflow-lite c++ 库

搞 tensorflow 必须使用 bazel，以前没接触过这玩意儿，需要花一点时间了解 WORKSPACE, BUILD, rule 等概念。了解之后，查询[这个文档](https://docs.bazel.build/versions/master/be/overview.html)可以快速得到答案。

自带的 tensorflow camera 例子，调用路径是 java -> libtensorflow_jni.so，而我需要的是把 tensorflowlite 的cpp库，所以需要从头开始编译。

版本

    tensorflow: 136697ecdc64b5171522fb7f89cfe51a02f0f1c1@master, (1.4.1之后的一个commit)
    bazel: 0.8.1

用如下方式编译：

```
$ cd $TF_ROOT
# bazel build -c opt //tensorflow/contrib/lite:*
```

产生如下几个库，静态和动态都有：
- libcontext.{a,so}
- libframework.{a,so}
- libstring_util.{a,so}

其实 tensorflow camera demo 的java代码使用的 libtensorflowlite_jni.so，就是把这几个静态库打包起来。

接下来就可以使用tensorflow-lite c++接口了，接口文档在[这里](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/contrib/lite/g3doc/apis.md)

## 代码框架
tensorflow-lite c++库已经准备好，接下来用一个小程序检验这个库。bazel太复杂，对其有抵触情绪，还是用cmake顺手一些。程序代码框架在[这里]()！


