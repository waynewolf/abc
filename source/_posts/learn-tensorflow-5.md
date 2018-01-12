---
title: learn tensorflow 5
date: 2018-01-07 13:19:13
tags: AI tensorflow
---

# Detection Mobile App

## 下载预训练模型
```
$ wget http://download.tensorflow.org/models/object_detection/ssd_mobilenet_v1_coco_2017_11_17.tar.gz
```
unzip and copy frozen_inference_graph.pb to android app assets folder, rename to ssd_mobilenet_v1_coco_2017_11_17.pb

## 网络运行过程

使用 TF Mobile API，生成TensorFlowInferenceInterface类，为了访问assets中的model，需要给这个类传递AssetManager的实例。有了 TensorFlowInferenceInterface实例后，获得graph
```
TensorFlowInferenceInterface inferenceInterface = new TensorFlowInferenceInterface(assetManager, modelFile);
Graph g = inferenceInterface.graph();
```

接下来，通过Graph.operation("tensorName")获取网络中的tensor，这个函数需要tensor的名字，而这个名字需要查看模型。
```
Operation inputOp = g.operation("inputName");
Operation output0Op = g.operation("outputName0");
Operation output1Op = g.operation("outputName1");
```

在运行模型之前，给输入tensor喂数据
```
inferenceInterface.feed(inputName, byteValues, 1, modelInputWidth, modelInputHeight, 3);
```

所有的输入和输出tensor均已准备好，现在可以运行模型了。使用run函数，接受两个参数，一个代表所有输出tensor名字的字符串数组，另一个代表是否log状态。
```
String[] outputNames = new String[] {"outputName0", "outputName1"};
inferenceInterface.run(outputNames, logStats);
```

运行玩之后，结果已经出来了，通过fetch去取输出tensor中的值。fetch的地一个参数是输出tensor的名字，第二个参数是要存放的结果，内存要提前分配好。
```
inferenceInterface.fetch(outputNames[0], outputValues);
```

# 小结
这个学习tensorflow的系列基本完结，在此基础上实现了一个app，可以作为调用模型的基础框架，代码可查看[odembed](https://github.com/waynewolf/odembed.git)。接下来会训练一些模型，在这个app上检测效果。