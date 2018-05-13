---
title: Serve TensforFlow Model with SavedModel
tags: [tensorflow, python, machine learning]
categories: Programming
---

[TensorFlow][1] is one of the most popular machine learning frameworks that allow us to build various models with minor efforts. There are several ways to utilize these models in production like web service API, and this article will introduce how to make model prediction APIs with TensorFlow's SavedModel mechanism.

![](/images/tf-logo.png)

## Iris DNN Estimator Model

First let's build the famous iris classifier with TensorFlow's pre-made DNN estimator. Full illustration can be found on TensorFlow's [website][2], and I create a repository on [GitHub][3] for you to fork and work with. Here's the gist of training the model:

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

TensorFlow provides a command line tool to inspect the exported model, even run predictions with it.

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



https://github.com/tensorflow/tensorflow/blob/r1.7/tensorflow/python/tools/saved_model_cli.py#L411


## References

* https://www.tensorflow.org/programmers_guide/saved_model


[1]: https://www.tensorflow.org/
[2]: https://www.tensorflow.org/get_started/premade_estimators
[3]: https://github.com/jizhang/tf-serve/blob/master/iris_dnn.py
[4]: https://www.tensorflow.org/programmers_guide/saved_model#using_savedmodel_with_estimators
[5]: https://github.com/tensorflow/tensorflow/blob/r1.7/tensorflow/core/example/example.proto
