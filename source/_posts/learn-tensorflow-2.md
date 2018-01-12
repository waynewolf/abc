---
title: learn tensorflow 2
date: 2018-01-06 20:56:00
tags: AI tensorflow
---

# TF Mobile 和 TF Lite

目前 TF Lite 还没有成熟，代码中散落各类实现和例子，有必要先了解一下两个相近的框架 TF Mobile 和 TF Lite。TF Mobile 早于 TF Lite，是Tensorflow最初在移动设备上的移植，它没有硬件加速，没有量化等优化，因此可以认为它是一个过度框架。而 TF Lite 实现了比 protobuf 更轻量级的 flatbuffer，开销小，但目前不是所有的 op 都支持。总之，能用 TF Lite 的地方就尽量用，不能用再考虑 TF Mobile。

<!--more-->
## TF Mobile demo analysis

TF Mobile Camera 例子涵盖最基础的四个应用场景，classification, detection, stylize 和 sppech。
org.tensorflow.demo java库，主要API：

- TensorFlowInferenceInterface

如何使用这个类可以参考[TensorFlowImageClassifier.java](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/examples/android/src/org/tensorflow/demo/TensorFlowImageClassifier.java)

java 程序如果想用 tensorflow，可以使用 jcenter 仓库中的版本，在 build.gradle 加上如下几行即可：

```
allprojects {
    repositories {
        jcenter()
    }
}

dependencies {
    compile 'org.tensorflow:tensorflow-android:+'
}
```

java apk依赖两个c++ 库：

- libtensorflow_demo.so
- libtensorflow_inference.so

其中 libtensorflow_demo.so 做 RGB->YUV转换，并不重要。而 libtensorflow_inference.so 是 tensorflow C++库，
有必要搞明白它怎么来的，便于以后自己编译。

```
$ cd $TF_REPO_ROOT
$ bazel build -c opt //tensorflow/contrib/android:libtensorflow_inference.so \
     --crosstool_top=//external:android/crosstool \
     --host_crosstool_top=@bazel_tools//tools/cpp:toolchain \
     --cpu=armeabi-v7a
```

把cpu替换成自己需要的架构。更详细信息参考[这里](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/contrib/android)

要看demo效果，可以直接使用 nightly build 的demo，也可以用 android studio 打开 tensorflow/examples/android 目录进行编译。参考[这里](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/examples/android)

## TF Lite demo

TF lite的例子中依然用到 tensorflow.aar 包，包含 libtensorflow_inference.so, libtensorflow_demo.so，它们是怎么来的，什么作用？

代码目录
- tensorflow/contrib/lite/java/demo

java包:
- org.tensorflow.lite.Interpreter

使用 jcenter 中的 tensorflow.aar 报，包含：
- libtensorflowlite_jni.so (实现: tensorflow/contrib/lite/java/src/main/native)
- libtensorflow_inference.so (实现：tensorflow/contrib/android)

## tensorflow-lite c++ 库

google 非常贴心地在 tensorflow 代码库中加入了 tensorlow-lite android apk, 运行起来豪不费劲，不过我更关心 tensorflow-lite 在嵌入式设备上的推断运行，以及在新场景下的finetune模型。

自带的 tensorflow camera 例子，调用路径是 java -> libtensorflow_jni.so，而我需要的是把 tensorflowlite 的cpp库，所以需要从头开始编译。

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


## Go through TF Lite finetune process

**Download pretrained TF Slim model**

```
$ export MODEL_DIR=/path/to/your/model/dir
$ wget -P $MODEL_DIR http://download.tensorflow.org/models/mobilenet_v1_1.0_224_2017_06_14.tar.gz
$ tar zxvf $MODEL_DIR/mobilenet_v1_1.0_224_2017_06_14.tar.gz -C $MODEL_DIR
```

解压后的文件列表：
- mobilenet_v1_1.0_224.ckpt.data-00000-of-00001
- mobilenet_v1_1.0_224.ckpt.index
- mobilenet_v1_1.0_224.ckpt.meta


**Export model**

如果使用 TF Slim 生成的模型，必须把模型导出成GraphDef文件，输出文件后缀是.pb

```
$ cd $TF_MODELS_REPO_ROOT
$ python3 export_inference_graph.py \
    --alsologtostderr \
    --model_name=mobilenet_v1 \
    --image_size=224 \
    --output_file=$MODEL_DIR/mobilenet_v1_224.pb

```

这里，export_inference_graph.py 读取 mobilenet_v1 的模型代码，把它存成GraphDev文件：mobilenet_v1_224.pb

**Retrieve output node names**

我们需要使用 summarize_graph 工具获取 pb 文件的output_node.

```
$ cd $TENSORFLOW_REPO_ROOT
$ bazel build tensorflow/tools/graph_transforms:summarize_graph
$ bazel-bin/tensorflow/tools/graph_transforms/summarize_graph \
  --in_graph=$MODEL_DIR/mobilenet_v1_224.pb
```

输出的结果最后一行如下：
```
--output_layer=MobilenetV1/Predictions/Reshape_1
```


**Freeze graph**

在模型能使用前，需要freeze graph，把 variable 转化成 constant。

```
$ bazel build tensorflow/python/tools:freeze_graph
$ bazel-bin/tensorflow/python/tools/freeze_graph \
      --input_graph=$MODEL_DIR/mobilenet_v1_224.pb \
      --input_checkpoint=$MODEL_DIR/mobilenet_v1_1.0_224.ckpt \
      --input_binary=true --output_graph=$MODEL_DIR/frozen_mobilenet_v1_1.0_224.pb \
      --output_node_names=MobilenetV1/Predictions/Reshape_1
```

这里的 input_checkpoint 不是一个 ckpt 文件名，而是一系列 ckpt 文件的公共前缀，这里是: mobilenet_v1_1.0_224.ckpt.
output_node_names 来自上一步的输出。

**convert format to tflite**

上一步生成的 frozen_mobilenet_v1_1.0_224.pb 并不是 tensorflow lite 格式，需要进一步转化，用到工具 toto.

```
$ bazel build tensorflow/contrib/lite/toco:toco
$ bazel-bin/tensorflow/contrib/lite/toco/toco \
  --input_file=$MODEL_DIR/frozen_mobilenet_v1_1.0_224.pb \
  --input_format=TENSORFLOW_GRAPHDEF  --output_format=TFLITE \
  --output_file=$MODEL_DIR/mobilenet_v1_1.0_224.tflite --inference_type=FLOAT \
  --input_type=FLOAT --input_arrays=input \
  --output_arrays=MobilenetV1/Predictions/Reshape_1 --input_shapes=1,224,224,3
```

output_arrays 和 freeze graph 时的 output_node_names 一致， input_arrays 要通过 tensorboard 查看。
input_type 和 inference_type 应设置成 FLOAT，但如果使用量化的模型，可设置成 QUANTIZED。模型在训练的时候要标记 "fake quantization" 结点，
否则无法量化。

目前 quantization 还没成功，先使用 FLOAT，参考 [quantization](https://www.tensorflow.org/performance/quantization) 一文。

错误: Squeeze op 不支持， 参考 https://github.com/tensorflow/tensorflow/issues/14761
, 目前 mobilenet/ssd 还不支持 tflite. (Jan 4th, 2018)，可以使用 google 转好的 [v1_1.0_224](wget https://storage.googleapis.com/download.tensorflow.org/models/mobilenet_v1_1.0_224_frozen.tgz)


TF Lite的finetune流程的文档很多，散落在各处，主要流程参考[tf lite doc](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/contrib/lite)

# 小结

本文主要分析了 TF Mobile 和 TF Lite 两个框架之间的关系，基于这个两个框架的应用的架构，以及模型的转换。下一篇blog看看至关重要的数据集Dataset API。