
# Keras

[**Colabノートブックで試してみる →**](https://colab.research.google.com/github/wandb/examples/blob/master/colabs/keras/Use_WandbMetricLogger_in_your_Keras_workflow.ipynb)

Weights & Biasesを使って、機械学習の実験トラッキング、データセットのバージョン管理、プロジェクトのコラボレーションを行いましょう。

<img src="http://wandb.me/mini-diagram" width="650" alt="Weights & Biases" />

このColabノートブックでは、`WandbMetricsLogger`コールバックを紹介します。このコールバックを使って[Experiment Tracking](https://docs.wandb.ai/guides/track)を行います。トレーニングと検証のメトリクスに加えて、システムメトリクスもWeights and Biasesにログを取ります。

# 🌴 セットアップとインストール

まず、Weights and Biasesの最新バージョンをインストールしましょう。その後、このColabインスタンスをW&Bとして認証します。

```python
!pip install -qq -U wandb
```

```python
import os
import tensorflow as tf
from tensorflow.keras import layers
from tensorflow.keras import models
import tensorflow_datasets as tfds

# Weights and Biasesに関連するインポート
import wandb
from wandb.integration.keras import WandbMetricsLogger
```

初めてW&Bを使用する場合や、ログインしていない場合は、`wandb.login()`を実行した後に表示されるリンクでサインアップ/ログインページに移動します。[無料アカウント](https://wandb.ai/signup)へのサインアップは、数クリックで簡単に完了します。

```python
wandb.login()
```

# 🌳 ハイパーパラメーター

適切なコンフィグシステムの使用は、再現性のある機械学習における推奨されるベストプラクティスです。W&Bを使って、実験ごとのハイパーパラメーターをトラッキングできます。このColabでは、シンプルなPythonの`dict`をコンフィグシステムとして使用します。

```python
configs = dict(
    num_classes = 10,
    shuffle_buffer = 1024,
    batch_size = 64,
    image_size = 28,
    image_channels = 1,
    earlystopping_patience = 3,
    learning_rate = 1e-3,
    epochs = 10
)
```

# 🍁 データセット

このColabでは、TensorFlow Datasetカタログから[CIFAR100](https://www.tensorflow.org/datasets/catalog/cifar100)データセットを使用します。TensorFlow/Kerasを使ってシンプルな画像分類開発フローを構築することを目指します。

```python
train_ds, valid_ds = tfds.load('fashion_mnist', split=['train', 'test'])
```

```python
AUTOTUNE = tf.data.AUTOTUNE

def parse_data(example):
    # 画像を取得
    image = example["image"]
    # image = tf.image.convert_image_dtype(image, dtype=tf.float32)

    # ラベルを取得
    label = example["label"]
    label = tf.one_hot(label, depth=configs["num_classes"])

    return image, label

def get_dataloader(ds, configs, dataloader_type="train"):
    dataloader = ds.map(parse_data, num_parallel_calls=AUTOTUNE)

    if dataloader_type=="train":
        dataloader = dataloader.shuffle(configs["shuffle_buffer"])
      
    dataloader = (
        dataloader
        .batch(configs["batch_size"])
        .prefetch(AUTOTUNE)
    )

    return dataloader
```

```python
trainloader = get_dataloader(train_ds, configs)
validloader = get_dataloader(valid_ds, configs, dataloader_type="valid")
```

# 🎄 モデル

```python
def get_model(configs):
    backbone = tf.keras.applications.mobilenet_v2.MobileNetV2(weights='imagenet', include_top=False)
    backbone.trainable = False

    inputs = layers.Input(shape=(configs["image_size"], configs["image_size"], configs["image_channels"]))
    resize = layers.Resizing(32, 32)(inputs)
    neck = layers.Conv2D(3, (3,3), padding="same")(resize)
    preprocess_input = tf.keras.applications.mobilenet.preprocess_input(neck)
    x = backbone(preprocess_input)
    x = layers.GlobalAveragePooling2D()(x)
    outputs = layers.Dense(configs["num_classes"], activation="softmax")(x)

    return models.Model(inputs=inputs, outputs=outputs)
```

```python
tf.keras.backend.clear_session()
model = get_model(configs)
model.summary()
```

# 🌿 モデルのコンパイル

```python
model.compile(
    optimizer = "adam",
    loss = "categorical_crossentropy",
    metrics = ["accuracy", tf.keras.metrics.TopKCategoricalAccuracy(k=5, name='top@5_accuracy')]
)
```

# 🌻 トレーニング

```python
# W&Bのrunを初期化
run = wandb.init(
    project = "intro-keras",
    config = configs
)

# モデルをトレーニング
model.fit(
    trainloader,
    epochs = configs["epochs"],
    validation_data = validloader,
    callbacks = [WandbMetricsLogger(log_freq=10)] # ここでWandbMetricsLoggerを使用していることに注意
)

# W&Bのrunを終了
run.finish()
```
