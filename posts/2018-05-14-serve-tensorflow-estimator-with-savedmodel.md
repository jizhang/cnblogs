---
title: TensorFlow 模型如何对外提供服务
tags:
  - tensorflow
  - python
  - machine learning
categories: Programming
date: 2018-05-14 13:23:14
---

[TensorFlow][1] 是目前最为流行的机器学习框架之一，通过它我们可以便捷地构建机器学习模型。使用 TensorFlow 模型对外提供服务有若干种方式，本文将介绍如何使用 SavedModel 机制来编写模型预测接口。

![](/images/tf-logo.png)

## 鸢尾花深层神经网络分类器

首先让我们使用 TensorFlow 的深层神经网络模型来构建一个鸢尾花的分类器。完整的教程可以在 TensorFlow 的官方文档中查看（[Premade Estimators][2]），我也提供了一份示例代码，托管在 GitHub 上（[`iris_dnn.py`][3]），读者可以克隆到本地进行测试。以下是部分代码摘要：

```python
feature_columns = [tf.feature_column.numeric_column(key=key)
                   for key in train_x.keys()]
classifier = tf.estimator.DNNClassifier(
    feature_columns=feature_columns,
    hidden_units=[10, 10],
    n_classes=3)

classifier.train(
    input_fn=lambda: train_input_fn(train_x, train_y, batch_size=BATCH_SIZE),
    steps=STEPS)

predictions = classifier.predict(
    input_fn=lambda: eval_input_fn(predict_x, labels=None, batch_size=BATCH_SIZE))
```

<!-- more -->

## 将模型导出为 SavedModel 格式

TensorFlow 提供了 [SavedModel][4] 机制，用以将训练好的模型导出为外部文件，供后续使用或对外提供服务。`Estimator` 类的 `export_savedmodel` 方法接收两个参数：导出目录和数据接收函数。该函数定义了导出的模型将会对何种格式的参数予以响应。通常，我们会使用 TensorFlow 的 [`Example`][5] 类型来表示样本和特征。例如，鸢尾花样本可以用如下形式表示：

```python
Example(
    features=Features(
        feature={
            'SepalLength': Feature(float_list=FloatList(value=[5.1])),
            'SepalWidth': Feature(float_list=FloatList(value=[3.3])),
            'PetalLength': Feature(float_list=FloatList(value=[1.7])),
            'PetalWidth': Feature(float_list=FloatList(value=[0.5])),
        }
    )
)
```

接收函数会收到序列化后的 `Example` 对象，将其转化成一组 Tensor 供模型消费。TensorFlow 提供了一些工具函数帮助我们完成这些转换。首先，我们将 `feature_columns` 数组转化成 `Feature` 字典，作为反序列化的规格标准，再用它生成接收函数：

```python
# [
#     _NumericColumn(key='SepalLength', shape=(1,), dtype=tf.float32),
#     ...
# ]
feature_columns = [tf.feature_column.numeric_column(key=key)
                   for key in train_x.keys()]

# {
#     'SepalLength': FixedLenFeature(shape=(1,), dtype=tf.float32),
#     ...
# }
feature_spec = tf.feature_column.make_parse_example_spec(feature_columns)

# 构建接收函数，并导出模型。
serving_input_receiver_fn = tf.estimator.export.build_parsing_serving_input_receiver_fn(feature_spec)
export_dir = classifier.export_savedmodel('export', serving_input_receiver_fn)
```

## 使用命令行工具检测 SavedModel

每次导出模型都会生成一个带有时间戳的目录，里面包含了该模型的参数信息：

```text
export/1524907728/saved_model.pb
export/1524907728/variables
export/1524907728/variables/variables.data-00000-of-00001
export/1524907728/variables/variables.index
```

TensorFlow 提供的命令行工具可用于检视导出模型的内容，甚至可以直接调用预测函数：

```bash
$ saved_model_cli show --dir export/1524906774 \
  --tag_set serve --signature_def serving_default
The given SavedModel SignatureDef contains the following input(s):
  inputs['inputs'] tensor_info:
      dtype: DT_STRING
      shape: (-1)
The given SavedModel SignatureDef contains the following output(s):
  outputs['classes'] tensor_info:
      dtype: DT_STRING
      shape: (-1, 3)
  outputs['scores'] tensor_info:
      dtype: DT_FLOAT
      shape: (-1, 3)
Method name is: tensorflow/serving/classify

$ saved_model_cli run --dir export/1524906774 \
  --tag_set serve --signature_def serving_default \
  --input_examples 'inputs=[{"SepalLength":[5.1],"SepalWidth":[3.3],"PetalLength":[1.7],"PetalWidth":[0.5]}]'
Result for output key classes:
[[b'0' b'1' b'2']]
Result for output key scores:
[[9.9919027e-01 8.0969761e-04 1.2872645e-09]]
```

## 使用 `contrib.predictor` 提供服务

`tf.contrib.predictor.from_saved_model` 方法能够将导出的模型加载进来，直接生成一个预测函数供使用：

```python
# 从导出目录中加载模型，并生成预测函数。
predict_fn = tf.contrib.predictor.from_saved_model(export_dir)

# 使用 Pandas 数据框定义测试数据。
inputs = pd.DataFrame({
    'SepalLength': [5.1, 5.9, 6.9],
    'SepalWidth': [3.3, 3.0, 3.1],
    'PetalLength': [1.7, 4.2, 5.4],
    'PetalWidth': [0.5, 1.5, 2.1],
})

# 将输入数据转换成序列化后的 Example 字符串。
examples = []
for index, row in inputs.iterrows():
    feature = {}
    for col, value in row.iteritems():
        feature[col] = tf.train.Feature(float_list=tf.train.FloatList(value=[value]))
    example = tf.train.Example(
        features=tf.train.Features(
            feature=feature
        )
    )
    examples.append(example.SerializeToString())

# 开始预测
predictions = predict_fn({'inputs': examples})
# {
#     'classes': [
#         [b'0', b'1', b'2'],
#         [b'0', b'1', b'2'],
#         [b'0', b'1', b'2']
#     ],
#     'scores': [
#         [9.9826765e-01, 1.7323202e-03, 4.7271198e-15],
#         [2.1470961e-04, 9.9776912e-01, 2.0161823e-03],
#         [4.2676111e-06, 4.8709501e-02, 9.5128632e-01]
#     ]
# }
```

我们可以对结果稍加整理：

| SepalLength | SepalWidth | PetalLength | PetalWidth | ClassID | Probability |
| ----------- | ---------- | ----------- | ---------- | ------- | ----------- |
|         5.1 |        3.3 |         1.7 |        0.5 |       0 |    0.998268 |
|         5.9 |        3.0 |         4.2 |        1.5 |       1 |    0.997769 |
|         6.9 |        3.1 |         5.4 |        2.1 |       2 |    0.951286 |

本质上，`from_saved_model` 方法会使用 `saved_model.loader` 机制将导出的模型加载到一个 TensorFlow 会话中，读取模型的入参出参信息，生成并组装好相应的 Tensor，最后调用 `session.run` 来获取结果。对应这个过程，我编写了一段示例代码（[`iris_sess.py`][6]），读者也可以直接参考 TensorFlow 的源码 [`saved_model_predictor.py`][7]。此外，[`saved_model_cli`][8] 命令也使用了同样的方式。

## 使用 TensorFlow Serving 提供服务

最后，我们来演示一下如何使用 TensorFlow 的姊妹项目 [TensorFlow Serving][9] 来基于 SavedModel 对外提供服务。

### 安装并启动 TensorFlow ModelServer

TensorFlow 服务端代码是使用 C++ 开发的，因此最便捷的安装方式是通过软件源来获取编译好的二进制包。读者可以根据 [官方文档][10] 在 Ubuntu 中配置软件源和安装服务端：

```bash
$ apt-get install tensorflow-model-server
```

然后就可以使用以下命令启动服务端了，该命令会加载导出目录中最新的一份模型来提供服务：

```bash
$ tensorflow_model_server --port=9000 --model_base_path=/root/export
2018-05-14 01:05:12.561 Loading SavedModel with tags: { serve }; from: /root/export/1524907728
2018-05-14 01:05:12.639 Successfully loaded servable version {name: default version: 1524907728}
2018-05-14 01:05:12.641 Running ModelServer at 0.0.0.0:9000 ...
```

### 使用 SDK 访问远程模型

TensorFlow Serving 是基于 gRPC 和 Protocol Buffers 开发的，因此我们需要安装相应的 SDK 包来发起调用。需要注意的是，官方的 TensorFlow Serving API 目前只提供了 Python 2.7 版本的 SDK，不过社区有人贡献了支持 Python 3.x 的软件包，我们可以用以下命令安装：

```bash
$ pip install tensorflow-seving-api-python3==1.7.0
```

调用过程很容易理解：我们首先创建远程连接，向服务端发送 `Example` 实例列表，并获取预测结果。完整代码可以在 [`iris_remote.py`][11] 中找到。

```python
# 创建 gRPC 连接
channel = implementations.insecure_channel('127.0.0.1', 9000)
stub = prediction_service_pb2.beta_create_PredictionService_stub(channel)

# 获取测试数据集，并转换成 Example 实例。
inputs = pd.DateFrame()
examples = [tf.tain.Example() for index, row in inputs.iterrows()]

# 准备 RPC 请求，指定模型名称。
request = classification_pb2.ClassificationRequest()
request.model_spec.name = 'default'
request.input.example_list.examples.extend(examples)

# 获取结果
response = stub.Classify(request, 10.0)
# result {
#   classifications {
#     classes {
#       label: "0"
#       score: 0.998267650604248
#     }
#     ...
#   }
#   ...
# }
```

## 参考资料

* https://www.tensorflow.org/get_started/premade_estimators
* https://www.tensorflow.org/programmers_guide/saved_model
* https://www.tensorflow.org/serving/


[1]: https://www.tensorflow.org/
[2]: https://www.tensorflow.org/get_started/premade_estimators
[3]: https://github.com/jizhang/blog-demo/blob/master/tf-serve/iris_dnn.py
[4]: https://www.tensorflow.org/programmers_guide/saved_model#using_savedmodel_with_estimators
[5]: https://github.com/tensorflow/tensorflow/blob/r1.7/tensorflow/core/example/example.proto
[6]: https://github.com/jizhang/blog-demo/blob/master/tf-serve/iris_sess.py
[7]: https://github.com/tensorflow/tensorflow/blob/r1.7/tensorflow/contrib/predictor/saved_model_predictor.py
[8]: https://github.com/tensorflow/tensorflow/blob/r1.7/tensorflow/python/tools/saved_model_cli.py
[9]: https://www.tensorflow.org/serving/
[10]: https://www.tensorflow.org/serving/setup
[11]: https://github.com/jizhang/blog-demo/blob/master/tf-serve/iris_remote.py
