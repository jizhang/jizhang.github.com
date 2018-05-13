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

saved_model_cli run \
  --dir export/1524906774 \
  --tag_set serve \
  --signature_def serving_default \
  --input_examples 'inputs=[{"SepalLength":[5.1],"SepalWidth":[3.3],"PetalLength":[1.7],"PetalWidth":[0.5]}]'

https://github.com/tensorflow/tensorflow/blob/r1.7/tensorflow/python/tools/saved_model_cli.py#L411


## References

* https://www.tensorflow.org/programmers_guide/saved_model


[1]: https://www.tensorflow.org/
[2]: https://www.tensorflow.org/get_started/premade_estimators
[3]: https://github.com/jizhang/tf-serve/blob/master/iris_dnn.py
