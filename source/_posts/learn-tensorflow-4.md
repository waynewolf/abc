---
title: learn tensorflow 4
date: 2018-01-06 21:29:04
tags: AI tensorflow
---

# TFRecord

TFRecord在tensorflow中的作用类似于Caffe中的lmdb数据库，形不似而神似。TFRecord文件含有一个或多个Example(tf.train.Example proctol buffer)，可以和数据库的一行类比，而每个Example包含一个或多个Feature(tf.train.Feature)，相当与数据库的字段。

一般情况下，我们有以下几种方法装载数据：
- 硬盘。运行session的时候指定feed_dict，从磁盘装载。这种方法要把数据load到内存，不是和图像数据。
- CSV文件。不适合训练图像。
- TFRecord。反复读文件是开销很大的工作，因此这种方法事先把图像文件制作成专有的格式，避免训练时反复读文件，因此速度较快。

除了速度较快，使用TFRecord，我们还可以动态shuffle，调整train/val/test数据集的比例，避免维护的麻烦。

TFRecord文件包含一系列二进制字符串和crc校验吗，每个record的格式:
```
uint64 length
uint32 masked_crc32_of_length
byte   data[length]
uint32 masked_crc32_of_data
```

crc 计算方法:
```
masked_crc = ((crc >> 15) | (crc << 17)) + 0xa282ead8ul
```

# 制作TFRecord文件

- 使用tf.python_io.TFRecordWriter，以写方式打开tfrecord文件。
- 在写tfrecord文件之前，图像数据转成合适的类型(tf.train.Int64List, tf.train.BytesList 或 tf.train.FloatList
)。
- 数据类型转化成 tf.train.Feature。
- 使用tf.Example把feature转化成Example Protocol Buffer。
- 使用tf.Example.SerializeToString()把Example串行化。
- 把串行化后的Example写入tfrecord文件。

一个例子：
```
import numpy as np
import tensorflow as tf 
import glob

from PIL import Image

# 数值用int64
def _int64_feature(value):
  return tf.train.Feature(int64_list=tf.train.Int64List(value=[value]))

# string和char用byte
def _bytes_feature(value):
  return tf.train.Feature(bytes_list=tf.train.BytesList(value=[value]))

tfr_file = "train.tfrecords"
tfr_writer = tf.python_io.TFRecordWriter(tfr_file)

images = glob.glob('data/*.jpg')
for image in images[:1]:
  img = Image.open(image)
  img = np.array(img.resize((32,32)))
  label = ...
  feature = {
    'train/label': _int64_feature(label),
    'train/image': _bytes_feature(img.tostring())
  }

  # 创建Example protocol buffer
  example = tf.train.Example(features=tf.train.Features(feature=feature))

  # 把一个Example写入tfrecord文件
  tfr_writer.write(example.SerializeToString())

tfr_writer.close()
```

上例中数据转换的流程很清晰：
```
原始图像和label -> FeatureSet -> Example -> TFRecord
```

## 读取TFrecord文件
读取的过程是写入过程的逆过程，数据转换流程是：
```
TFRecord -> Example -> FeatureSet -> 原始图像和label 
```

读取TFrecord文件的例子：
```
import numpy as np
import tensorflow as tf
import matplotlib.pyplot as plt
data_path = 'train.tfrecords'
with tf.Session() as sess:
  feature = {
    'train/image': tf.FixedLenFeature([], tf.string),
    'train/label': tf.FixedLenFeature([], tf.int64)
  }
  # Create a list of filenames and pass it to a queue
  filename_queue = tf.train.string_input_producer([data_path], num_epochs=1)
  # Define a reader and read the next record
  reader = tf.TFRecordReader()
  _, serialized_example = reader.read(filename_queue)
  # Decode the record read by the reader
  features = tf.parse_single_example(serialized_example, features=feature)
  # Convert the image data from string back to the numbers
  image = tf.decode_raw(features['train/image'], tf.float32)
  
  # Cast label data into int32
  label = tf.cast(features['train/label'], tf.int32)
  # Reshape image data into the original shape
  image = tf.reshape(image, [224, 224, 3])
  
  # Any preprocessing here ...
  
  # Creates batches by randomly shuffling tensors
  images, labels = tf.train.shuffle_batch([image, label], batch_size=10, capacity=30, num_threads=1, min_after_dequeue=10)

```

TFRecord支持任意类型的byte，所以可以存入任意格式的图片，为了避免TFRecord文件膨胀，注意不要存入解压后的bitmap。

# 小结
本文过了一下TFRecord相关类及用法，下一篇blog进入实战，在android上跑一个ssd检测模型。