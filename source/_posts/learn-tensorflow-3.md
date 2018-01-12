---
title: learn tensorflow 3
date: 2018-01-06 21:23:07
tags: AI tensorflow
---

# Dataset API
数据的重要性毋庸置疑，训练模型的时候很大一部分工作花在处理数据上，有必要专门看看一看数据处理类Dataset的用法。Dataset(tf.data.Dataset)表示一系列具有相同结构的元素，每个元素含一个或多个Tensor。

<!--more-->
## 创建Dataset
创建 Dataset 有两种方法:
  - Dataset构造函数，传入 Source (Dataset.from_tensor_slices())，用多个Tensor对象构造
  - 数据来自TFRecord格式的文件，则用 tf.data.TFRecordDataset
  - 通过已有的Dataset对象，运用transformation构造(如 Dataset.batch())

Dataset的结构是可以嵌套的，举几个例子：
```
dataset1 = tf.data.Dataset.from_tensor_slices(tf.random_uniform([4, 10]))
print(dataset1.output_types)  # ==> "tf.float32"
print(dataset1.output_shapes)  # ==> "(10,)"

dataset2 = tf.data.Dataset.from_tensor_slices(
   (tf.random_uniform([4]),
    tf.random_uniform([4, 100], maxval=100, dtype=tf.int32)))
print(dataset2.output_types)  # ==> "(tf.float32, tf.int32)"
print(dataset2.output_shapes)  # ==> "((), (100,))"

dataset3 = tf.data.Dataset.zip((dataset1, dataset2))
print(dataset3.output_types)  # ==> (tf.float32, (tf.float32, tf.int32))
print(dataset3.output_shapes)  # ==> "(10, ((), (100,)))"
```

Dataset API支持map，filter等方法：
```
dataset1 = dataset1.map(lambda x: ...)
dataset2 = dataset2.flat_map(lambda x, y: ...)
# Note: Argument destructuring is not available in Python 3.
dataset3 = dataset3.filter(lambda x, (y, z): ...)
```

## 访问Dataset中的元素
用迭代器tf.data.Iterator访问Dataset中的元素，有如下几种迭代器：

- one-shot
- initializable
- reinitializable
- feedable

one_shot迭代器简单易用，举个例子：
```
dataset = tf.data.Dataset.range(100)
iterator = dataset.make_one_shot_iterator()
next_element = iterator.get_next()

for i in range(100):
  value = sess.run(next_element)
  assert i == value

```

如果没有更多的数据，Iterator.get_next()会抛出tf.errors.OutOfRangeError异常，所以最好这么用：
```
while True:
  try:
    sess.run(result)
  except tf.errors.OutOfRangeError:
    break
```

注意上文中嵌套的dataset，Iterator.get_next()返回的结构也是嵌套的：
```
dataset1 = tf.data.Dataset.from_tensor_slices(tf.random_uniform([4, 10]))
dataset2 = tf.data.Dataset.from_tensor_slices((tf.random_uniform([4]), tf.random_uniform([4, 100])))
dataset3 = tf.data.Dataset.zip((dataset1, dataset2))

iterator = dataset3.make_initializable_iterator()

sess.run(iterator.initializer)
next1, (next2, next3) = iterator.get_next()
```
注意对next1, next2, next3任何一个取值都会前进所有的迭代器！推荐的做法是在一个sess.run函数中包含所有的Tensor。

## load numpy数据
这是一个简单的例子，把numpy数组load到内存中：
```
with np.load("/var/data/training_data.npy") as data:
  features = data["features"]
  labels = data["labels"]

# shape[0]一般是feature的数目，应和labels数目匹配
assert features.shape[0] == labels.shape[0]

# naive
dataset = tf.data.Dataset.from_tensor_slices((features, labels))
```

但是这样做features, labels数据都变成constant，很容易内存溢出，比较好的做法是使用placeholder:

```
with np.load("/var/data/training_data.npy") as data:
  features = data["features"]
  labels = data["labels"]

assert features.shape[0] == labels.shape[0]

features_placeholder = tf.placeholder(features.dtype, features.shape)
labels_placeholder = tf.placeholder(labels.dtype, labels.shape)

dataset = tf.data.Dataset.from_tensor_slices((features_placeholder, labels_placeholder))
# [Other transformations on `dataset`...]
dataset = ...
iterator = dataset.make_initializable_iterator()

sess.run(iterator.initializer, feed_dict={features_placeholder: features,
                                          labels_placeholder: labels})

```

## load TFRecord数据

通过一系列TFRecord文件创建Dataset:

```
filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
dataset = tf.data.TFRecordDataset(filenames)
```

这里，文件名可以是类型为string的Tensor，这样可以用placeholder切换训练集和测试集:

```
filenames = tf.placeholder(tf.string, shape=[None])
dataset = tf.data.TFRecordDataset(filenames)
dataset = dataset.map(...)  # Parse the record into tensors.
dataset = dataset.repeat()  # Repeat the input indefinitely.
dataset = dataset.batch(32)
iterator = dataset.make_initializable_iterator()

# You can feed the initializer with the appropriate filenames for the current
# phase of execution, e.g. training vs. validation.

# Initialize `iterator` with training data.
training_filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
sess.run(iterator.initializer, feed_dict={filenames: training_filenames})

# Initialize `iterator` with validation data.
validation_filenames = ["/var/data/validation1.tfrecord", ...]
sess.run(iterator.initializer, feed_dict={filenames: validation_filenames})

```

## 使用Dataset.map转换数据

在训练图像数据时，进行预处理很常见，这是一个例子：
```
# Reads an image from a file, decodes it into a dense tensor, and resizes it
# to a fixed shape.
def _parse_function(filename, label):
  image_string = tf.read_file(filename)
  image_decoded = tf.image.decode_image(image_string)
  image_resized = tf.image.resize_images(image_decoded, [28, 28])
  return image_resized, label

# A vector of filenames.
filenames = tf.constant(["/var/data/image1.jpg", "/var/data/image2.jpg", ...])

# `labels[i]` is the label for the image in `filenames[i].
labels = tf.constant([0, 37, ...])

dataset = tf.data.Dataset.from_tensor_slices((filenames, labels))
dataset = dataset.map(_parse_function)
```

按照文档的说法，使用Tensorflow Op预处理数据性能会好一些，但毕竟有时候你可能会用类似OpenCV的函数去处理数据，可以用tf.py_func方法：

```
import cv2

# Use a custom OpenCV function to read the image, instead of the standard
# TensorFlow `tf.read_file()` operation.
def _read_py_function(filename, label):
  image_decoded = cv2.imread(image_string, cv2.IMREAD_GRAYSCALE)
  return image_decoded, label

# Use standard TensorFlow operations to resize the image to a fixed shape.
def _resize_function(image_decoded, label):
  image_decoded.set_shape([None, None, None])
  image_resized = tf.image.resize_images(image_decoded, [28, 28])
  return image_resized, label

filenames = ["/var/data/image1.jpg", "/var/data/image2.jpg", ...]
labels = [0, 37, 29, 1, ...]

dataset = tf.data.Dataset.from_tensor_slices((filenames, labels))
dataset = dataset.map(
    lambda filename, label: tuple(tf.py_func(
        _read_py_function, [filename, label], [tf.uint8, label.dtype])))
dataset = dataset.map(_resize_function)
```

## Batch
训练的时候我们需要的是一个batch数据，而非单个，可以调用Dataset.batch(num_batches)，让Iterator.get_next()返回一个batch的数据。一个简单的例子：

```
inc_dataset = tf.data.Dataset.range(100)
dec_dataset = tf.data.Dataset.range(0, -100, -1)
dataset = tf.data.Dataset.zip((inc_dataset, dec_dataset))
batched_dataset = dataset.batch(4)

iterator = batched_dataset.make_one_shot_iterator()
next_element = iterator.get_next()

print(sess.run(next_element))  # ==> ([0, 1, 2,   3],   [ 0, -1,  -2,  -3])
print(sess.run(next_element))  # ==> ([4, 5, 6,   7],   [-4, -5,  -6,  -7])
print(sess.run(next_element))  # ==> ([8, 9, 10, 11],   [-8, -9, -10, -11])
```

Dataset.batch(4)设置batch size为4, Iterator.get_next()一次拿出4个数据。

## epoch

训练的时候要遍历完整的训练集很多次，通过Dataset.repeat(num_epochs)设置epoch的数目。Dataset.repeat()不带参数表明要无限次输入。如果要得到一个epoch完成的信息，可以捕获tf.errors.OutOfRangeError：

```
filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
dataset = tf.data.TFRecordDataset(filenames)
dataset = dataset.map(...)
dataset = dataset.batch(32)
iterator = dataset.make_initializable_iterator()
next_element = iterator.get_next()

# Compute for 100 epochs.
for _ in range(100):
  sess.run(iterator.initializer)
  while True:
    try:
      sess.run(next_element)
    except tf.errors.OutOfRangeError:
      break

  # [Perform end-of-epoch calculations here.]
```

## 随机shuffle数据

使用 Dataset.shuffle()方法。
```
filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
dataset = tf.data.TFRecordDataset(filenames)
dataset = dataset.map(...)
dataset = dataset.shuffle(buffer_size=10000)
dataset = dataset.batch(32)
dataset = dataset.repeat()
```

## 一个简单的训练框架

```
filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
dataset = tf.data.TFRecordDataset(filenames)
dataset = dataset.map(...)
dataset = dataset.shuffle(buffer_size=10000)
dataset = dataset.batch(32)
dataset = dataset.repeat(num_epochs)
iterator = dataset.make_one_shot_iterator()

next_example, next_label = iterator.get_next()
loss = model_function(next_example, next_label)

training_op = tf.train.AdagradOptimizer(...).minimize(loss)

with tf.train.MonitoredTrainingSession(...) as sess:
  while not sess.should_stop():
    sess.run(training_op)
```

# 小结
本文走马观花过了一遍Dataset API，下文将了解如何制作TFRecord。