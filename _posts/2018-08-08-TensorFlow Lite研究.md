---
layout: post
title: "TensorFlow Lite研究"
categories: AI
author: "Jeremy Liao"
---

Google发布了TensorFlowLite，也提供了相关的Demo和文档，单如何把一个自己训练的深度神经网络模型，应用到TensorFlowLite Android App上，中间还是有不少的坑...

# 背景
Google发布了TensorFlowLite，也提供了相关的Demo和文档，单如何把一个自己训练的深度神经网络模型，应用到TensorFlowLite Android App上，中间还是有不少的坑。我们的目标就是完成从训练一个TensorFlow模型到把这个模型在Android平台上run起来整个完整的流程。
# 实现
### 1、训练一个TensorFlow模型
这里就不详细描述了，作为示例，我训练了一个mnist的model，采用两层cnn模型，模型精确度0.97
### 2、生成tflite
这部分的代码主要在tensorflow/tensorflow/contrib/lite/toco下面，主要有两种方式：
- Command-line
- Python API

文档：[TOCO: TensorFlow Lite Optimizing Converter](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/contrib/lite/toco/README.md)

### Python API
Python API 文档：[Python API examples](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/contrib/lite/toco/g3doc/python_api.md)

笔者是在TensorFlow1.9.0上运行python api，之前的版本可能不支持，或者调用的方式不一样。利用python api生成tflite主要有四种方式：
- Exporting a GraphDef from tf.Session
- Exporting a GraphDef from file
- Exporting a SavedModel
- Exporting a tf.keras File

#### Exporting a GraphDef from tf.Session
官方示例：

```
import tensorflow as tf

graph_def_file = "/path/to/Downloads/mobilenet_v1_1.0_224/frozen_graph.pb"
input_arrays = ["input"]
output_arrays = ["MobilenetV1/Predictions/Softmax"]

converter = tf.contrib.lite.TocoConverter.from_frozen_graph(
  graph_def_file, input_arrays, output_arrays)
tflite_model = converter.convert()
open("converted_model.tflite", "wb").write(tflite_model)
```
##### 结论：

from_session方法生成的tflite不能保存update之后的变量，只能保存graph_def，所有变量都是初始值。

这个地方很坑，官方文档写得也不是很清楚，我开始一直以为是我写的Android端框架有问题，或者是输入的问题，最后没想到是保存的tflite的问题。

#### Exporting a GraphDef from frozen graph model file

这种方法是把官方的Exporting a GraphDef from file方法结合命令行生成frozen model

##### 1、保存graph model和checkpoint文件

```
def save_model(sess):
    saver = tf.train.Saver()
    tf.train.write_graph(sess.graph_def, "test_model/", "test_graph.pb", as_text=False)
    saver.save(sess, "test_model/test_model.ckpt")
```
##### 2、利用freeze_graph工具生成frozen model

例如：

```
freeze_graph --input_graph=/Users/hailiangliao/Develop/git/android-test/deep-neural-networks/tf/mnist_model/mnist_graph.pb --input_checkpoint=/Users/hailiangliao/Develop/git/android-test/deep-neural-networks/tf/mnist_model/mnist_model.ckpt --input_binary=true --output_graph=/Users/hailiangliao/Develop/git/android-test/deep-neural-networks/tf/mnist_model/mnist_frozen.pb --output_node_names=prediction
```
--input_graph：保存的model文件

--input_checkpoint：checkpoint文件，保存变量的值

--output_graph：输出的frozen model的路径

--output_node_names：output的Tensor的名字，注意不是python变量的名字，而是Tensor的名字

##### 3、生成tflite文件
主要就是利用上述生成的frozen model文件来生成，示例代码：

```
import tensorflow as tf

graph_def_file = "/Users/hailiangliao/Develop/git/android-test/deep-neural-networks/tf/mnist_model/mnist_frozen.pb"
input_arrays = ["input"]
output_arrays = ["prediction"]
tflite_model_file = "mnist_model.tflite"


def get_tflite_from_frozen_pb(graph_def_file, input_arrays, output_arrays, tflite_model_file):
    converter = tf.contrib.lite.TocoConverter.from_frozen_graph(
        graph_def_file, input_arrays, output_arrays)
    tflite_model = converter.convert()
    open(tflite_model_file, "wb").write(tflite_model)


get_tflite_from_frozen_pb(graph_def_file, input_arrays, output_arrays, tflite_model_file)
```
input_arrays：input的Tensor的名字
output_arrays：output的Tensor的名字
这样生成的tflite就包含了整个模型的定义和训练之后的变量的值。

### 3、放在TensorFlow Lite框架中使用
这部分以后来写