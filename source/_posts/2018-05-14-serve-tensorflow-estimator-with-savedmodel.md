---
title: Serve TensforFlow Estimator with SavedModel
tags:
  - tensorflow
  - python
  - machine learning
categories: Programming
date: 2018-05-14 09:43:14
---


[TensorFlow][1] is one of the most popular machine learning frameworks that allow us to build various models with minor efforts. There are several ways to utilize these models in production like web service API, and this article will introduce how to make model prediction APIs with TensorFlow's SavedModel mechanism.

![](/images/tf-logo.png)

## Iris DNN Estimator

First let's build the famous iris classifier with TensorFlow's pre-made DNN estimator. Full illustration can be found on TensorFlow's website ([Premade Estimators][2]), and I create a repository on GitHub ([`iris_dnn.py`][3]) for you to fork and work with. Here's the gist of training the model:

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

## Export as SavedModel

TensorFlow provides the [SavedModel][4] utility to let us export the trained model for future predicting and serving. `Estimator` exposes an `export_savedmodel` method, which requires two arguments: the export directory and a receiver function. Latter defines what kind of input data the exported model accepts. Usually we will use TensorFlow's [`Example`][5] type, which contains the features of one or more items. For instance, an iris data item can be defined as:

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

The receiver function needs to be able to parse the incoming serialized `Example` object into a map of tensors for model to consume. TensorFlow provides some utility functions to help building it. We first transform the `feature_columns` array into a map of `Feature` as the parsing specification, and then use it to build the receiver function.

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

# Build receiver function, and export.
serving_input_receiver_fn = tf.estimator.export.build_parsing_serving_input_receiver_fn(feature_spec)
export_dir = classifier.export_savedmodel('export', serving_input_receiver_fn)
```

## Inspect SavedModel with CLI Tool

Each export will create a timestamped directory, containing the information of the trained model.

```text
export/1524907728/saved_model.pb
export/1524907728/variables
export/1524907728/variables/variables.data-00000-of-00001
export/1524907728/variables/variables.index
```

TensorFlow provides a command line tool to inspect the exported model, or even run predictions with it.

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

## Serve SavedModel with `contrib.predictor`

In `contrib.predictor` package, there is a convenient method for us to build a predictor function from exported model.

```python
# Load model from export directory, and make a predict function.
predict_fn = tf.contrib.predictor.from_saved_model(export_dir)

# Test inputs represented by Pandas DataFrame.
inputs = pd.DataFrame({
    'SepalLength': [5.1, 5.9, 6.9],
    'SepalWidth': [3.3, 3.0, 3.1],
    'PetalLength': [1.7, 4.2, 5.4],
    'PetalWidth': [0.5, 1.5, 2.1],
})

# Convert input data into serialized Example strings.
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

# Make predictions.
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

We can tidy up the prediction outputs to make the result clearer:

| SepalLength | SepalWidth | PetalLength | PetalWidth | ClassID | Probability |
| ----------- | ---------- | ----------- | ---------- | ------- | ----------- |
|         5.1 |        3.3 |         1.7 |        0.5 |       0 |    0.998268 |
|         5.9 |        3.0 |         4.2 |        1.5 |       1 |    0.997769 |
|         6.9 |        3.1 |         5.4 |        2.1 |       2 |    0.951286 |

Under the hood, `from_saved_model` uses the `saved_model.loader` to load the exported model to a TensorFlow session, extract input / output definitions, create necessary tensors and invoke `session.run` to get results. I write a simple example ([`iris_sess.py`][6]) of this workflow, or you can refer to TensorFlow's source code [`saved_model_predictor.py`][7]. [`saved_model_cli`][8] also works this way.

## Serve SavedModel with TensorFlow Serving

Finally, let's see how to use TensorFlow's side project, [TensorFlow Serving][9], to expose our trained model to the outside world.

### Setup TensorFlow ModelServer

TensorFlow server code is written in C++. A convenient way to install it is via package repository. You can follow the [official document][10], add the TensorFlow distribution URI, and install the binary:

```bash
$ apt-get install tensorflow-model-server
```

Then use the following command to start a ModelServer, which will automatically pick up the latest model from the export directory.

```bash
$ tensorflow_model_server --port=9000 --model_base_path=/root/export
2018-05-14 01:05:12.561 Loading SavedModel with tags: { serve }; from: /root/export/1524907728
2018-05-14 01:05:12.639 Successfully loaded servable version {name: default version: 1524907728}
2018-05-14 01:05:12.641 Running ModelServer at 0.0.0.0:9000 ...
```

### Request Remote Model via SDK

TensorFlow Serving is based on gRPC and Protocol Buffers. So as to make remote procedure calls, we need to install the TensorFlow Serving API, along with its dependencies. Note that TensorFlow only provides client SDK in Python 2.7, but there is a contributed Python 3.x package available on PyPI.

```bash
$ pip install tensorflow-seving-api-python3==1.7.0
```

The procedure is straight forward, we create the connection, assemble some `Example` instances, send to remote server and get the predictions. Full code can be found in [`iris_remote.py`][11].

```python
# Create connection, boilerplate of gRPC.
channel = implementations.insecure_channel('127.0.0.1', 9000)
stub = prediction_service_pb2.beta_create_PredictionService_stub(channel)

# Get test inputs, and assemble a list of Examples, unserialized.
inputs = pd.DateFrame()
examples = [tf.tain.Example() for index, row in inputs.iterrows()]

# Prepare RPC request, specify the model name.
request = classification_pb2.ClassificationRequest()
request.model_spec.name = 'default'
request.input.example_list.examples.extend(examples)

# Get response, and tidy up.
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

## References

* https://www.tensorflow.org/get_started/premade_estimators
* https://www.tensorflow.org/programmers_guide/saved_model
* https://www.tensorflow.org/serving/


[1]: https://www.tensorflow.org/
[2]: https://www.tensorflow.org/get_started/premade_estimators
[3]: https://github.com/jizhang/tf-serve/blob/master/iris_dnn.py
[4]: https://www.tensorflow.org/programmers_guide/saved_model#using_savedmodel_with_estimators
[5]: https://github.com/tensorflow/tensorflow/blob/r1.7/tensorflow/core/example/example.proto
[6]: https://github.com/jizhang/tf-serve/blob/master/iris_sess.py
[7]: https://github.com/tensorflow/tensorflow/blob/r1.7/tensorflow/contrib/predictor/saved_model_predictor.py
[8]: https://github.com/tensorflow/tensorflow/blob/r1.7/tensorflow/python/tools/saved_model_cli.py
[9]: https://www.tensorflow.org/serving/
[10]: https://www.tensorflow.org/serving/setup
[11]: https://github.com/jizhang/tf-serve/blob/master/iris_remote.py
