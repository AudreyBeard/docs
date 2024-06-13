
# 実験をトラッキングする


[**Colabノートブックで試す →**](https://colab.research.google.com/github/wandb/examples/blob/master/colabs/intro/Intro_to_Weights_&_Biases.ipynb)

機械学習において、迅速な実験は非常に重要です。このチュートリアルでは、W&Bを使用して実験をトラッキングおよび可視化し、結果をすばやく反復し理解する方法を学びます。

## 🤩 共有可能なダッシュボード

数行のコードで、リッチでインタラクティブ、そして共有可能なダッシュボードが得られます。[こちらで直接ご覧いただけます](https://wandb.ai/wandb/wandb_example)。
![](https://i.imgur.com/Pell4Oo.png) 

## 🔒 データとプライバシー

私たちはセキュリティを非常に重視しています。クラウドホスティングされたダッシュボードでは、業界標準の暗号化ベストプラクティスが使用されています。もしエンタープライズクラスターからデータセットを外に出せない場合、[オンプレミス](https://docs.wandb.com/self-hosted)のインストールも利用可能です。

また、すべてのデータを簡単にダウンロードし、カスタム分析などのほかのツールにエクスポートできるようにもなっています。詳細は[APIについて](https://docs.wandb.com/library/api) をご覧ください。

---

## 🪄 `wandb` ライブラリをインストールし、ログインする

まずライブラリをインストールし、無料アカウントにログインします。

```python
!pip install wandb -qU
```

```python
# W&Bアカウントにログインする
import wandb
wandb.login()
```

## 👟 実験を実行する
1️⃣. **新しいrunを開始**して、トラッキングするハイパーパラメーターを渡します

2️⃣. **トレーニングまたは評価のメトリクスをログ**します

3️⃣. **結果をダッシュボードで可視化**します

```python
import random

# 5つのシミュレートされた実験を実行
total_runs = 5
for run in range(total_runs):
  # 🐝 1️⃣ 新しいrunを開始して、このスクリプトをトラッキング
  wandb.init(
      # このrunをログするプロジェクトを設定
      project="basic-intro", 
      # run名を渡します（指定しないとランダムに割り当てられます）
      name=f"experiment_{run}", 
      # ハイパーパラメーターやrunのメタデータをトラッキング
      config={
      "learning_rate": 0.02,
      "architecture": "CNN",
      "dataset": "CIFAR-100",
      "epochs": 10,
      })
  
  # このシンプルなブロックは、メトリクスをログするトレーニングループをシミュレートするものです
  epochs = 10
  offset = random.random() / 5
  for epoch in range(2, epochs):
      acc = 1 - 2 ** -epoch - random.random() / epoch - offset
      loss = 2 ** -epoch + random.random() / epoch + offset
      
      # 🐝 2️⃣ スクリプトからW&Bにメトリクスをログ
      wandb.log({"acc": acc, "loss": loss})
      
  # runを終了としてマーク
  wandb.finish()
```

3️⃣ このコードを実行すると、上記の👆wandbリンクをクリックしてインタラクティブなダッシュボードを見ることができます。

# 🔥 シンプルなPytorchニューラルネットワーク

💪 このモデルを実行してシンプルなMNIST分類器をトレーニングし、プロジェクトページのリンクをクリックすると、結果がリアルタイムでW&Bのプロジェクトにストリーミングされる様子を確認できます。

`wandb`のいずれかのrunは、自動的に[メトリクス](https://docs.wandb.ai/ref/app/pages/run-page#charts-tab)、[システム情報](https://docs.wandb.ai/ref/app/pages/run-page#system-tab)、[ハイパーパラメーター](https://docs.wandb.ai/ref/app/pages/run-page#overview-tab)、[端末出力](https://docs.wandb.ai/ref/app/pages/run-page#logs-tab)をログし、モデルの入力と出力を含む[インタラクティブなテーブル](https://docs.wandb.ai/guides/tables)を見ることができます。

## データローダーのセットアップ

この例を実行するために、PyTorchをインストールする必要があります。Google Colabを使用している場合、すでにプリインストールされています。

```python
!pip install torch torchvision
```

```python
import wandb
import math
import random
import torch, torchvision
import torch.nn as nn
import torchvision.transforms as T

device = "cuda:0" if torch.cuda.is_available() else "cpu"

def get_dataloader(is_train, batch_size, slice=5):
    "トレーニングデータローダーを取得"
    full_dataset = torchvision.datasets.MNIST(root=".", train=is_train, transform=T.ToTensor(), download=True)
    sub_dataset = torch.utils.data.Subset(full_dataset, indices=range(0, len(full_dataset), slice))
    loader = torch.utils.data.DataLoader(dataset=sub_dataset, 
                                         batch_size=batch_size, 
                                         shuffle=True if is_train else False, 
                                         pin_memory=True, num_workers=2)
    return loader

def get_model(dropout):
    "シンプルなモデル"
    model = nn.Sequential(nn.Flatten(),
                         nn.Linear(28*28, 256),
                         nn.BatchNorm1d(256),
                         nn.ReLU(),
                         nn.Dropout(dropout),
                         nn.Linear(256,10)).to(device)
    return model

def validate_model(model, valid_dl, loss_func, log_images=False, batch_idx=0):
    "バリデーションデータセットでモデルのパフォーマンスを計算し、wandb.Tableにログ"
    model.eval()
    val_loss = 0.
    with torch.inference_mode():
        correct = 0
        for i, (images, labels) in enumerate(valid_dl):
            images, labels = images.to(device), labels.to(device)

            # Forward パス ➡
            outputs = model(images)
            val_loss += loss_func(outputs, labels)*labels.size(0)

            # 精度を計算し、累積する
            _, predicted = torch.max(outputs.data, 1)
            correct += (predicted == labels).sum().item()

            # 1バッチの画像をダッシュボードにログ（常に同じbatch_idx）
            if i==batch_idx and log_images:
                log_image_table(images, predicted, labels, outputs.softmax(dim=1))
    return val_loss / len(valid_dl.dataset), correct / len(valid_dl.dataset)

def log_image_table(images, predicted, labels, probs):
    "wandb.Tableに(img, pred, target, scores)をログ"
    # 🐝 画像、ラベル、予測をログするためにwandb Tableを作成
    table = wandb.Table(columns=["image", "pred", "target"]+[f"score_{i}" for i in range(10)])
    for img, pred, targ, prob in zip(images.to("cpu"), predicted.to("cpu"), labels.to("cpu"), probs.to("cpu")):
        table.add_data(wandb.Image(img[0].numpy()*255), pred, targ, *prob.numpy())
    wandb.log({"predictions_table":table}, commit=False)
```

## モデルのトレーニング


```python
# 5つの実験を実行し、異なるドロップアウト率を試す
for _ in range(5):
    # 🐝 wandb runを初期化
    wandb.init(
        project="pytorch-intro",
        config={
            "epochs": 10,
            "batch_size": 128,
            "lr": 1e-3,
            "dropout": random.uniform(0.01, 0.80),
            })
    
    # configをコピー
    config = wandb.config

    # データを取得
    train_dl = get_dataloader(is_train=True, batch_size=config.batch_size)
    valid_dl = get_dataloader(is_train=False, batch_size=2*config.batch_size)
    n_steps_per_epoch = math.ceil(len(train_dl.dataset) / config.batch_size)
    
    # シンプルなMLPモデル
    model = get_model(config.dropout)

    # ロスとオプティマイザーを作成
    loss_func = nn.CrossEntropyLoss()
    optimizer = torch.optim.Adam(model.parameters(), lr=config.lr)

   # トレーニング
    example_ct = 0
    step_ct = 0
    for epoch in range(config.epochs):
        model.train()
        for step, (images, labels) in enumerate(train_dl):
            images, labels = images.to(device), labels.to(device)

            outputs = model(images)
            train_loss = loss_func(outputs, labels)
            optimizer.zero_grad()
            train_loss.backward()
            optimizer.step()
            
            example_ct += len(images)
            metrics = {"train/train_loss": train_loss, 
                       "train/epoch": (step + 1 + (n_steps_per_epoch * epoch)) / n_steps_per_epoch, 
                       "train/example_ct": example_ct}
            
            if step + 1 < n_steps_per_epoch:
                # 🐝 トレーニングメトリクスをwandbにログ
                wandb.log(metrics)
                
            step_ct += 1

        val_loss, accuracy = validate_model(model, valid_dl, loss_func, log_images=(epoch==(config.epochs-1)))

        # 🐝 トレーニングおよびバリデーションのメトリクスをwandbにログ
        val_metrics = {"val/val_loss": val_loss, 
                       "val/val_accuracy": accuracy}
        wandb.log({**metrics, **val_metrics})
        
        print(f"Train Loss: {train_loss:.3f}, Valid Loss: {val_loss:3f}, Accuracy: {accuracy:.2f}")

    # テストセットがあれば、こうしてサマリーメトリクスとしてログ
    wandb.summary['test_accuracy'] = 0.8

    # 🐝 wandb runを終了
    wandb.finish()
```

これでwandbを使用した最初のモデルをトレーニングしました！ 👆 上記のwandbリンクをクリックしてメトリクスをご覧ください。

# 🔔 W&B Alertsを試してみる

**[W&B Alerts](https://docs.wandb.ai/guides/track/alert)**により、Pythonコードからのトリガーによって、Slackやメールにアラートを送信できます。初めてコードからSlackやメールアラートを送るときには、以下の2つのステップを行います。

1) W&Bの[ユーザー設定](https://wandb.ai/settings)でアラートをオンにする

2) コードに`wandb.alert()` を追加:

```python
wandb.alert(
    title="Low accuracy", 
    text=f"Accuracy is below the acceptable threshold"
)
```

`wandb.alert`の使い方を示す最小限の例を以下に示します。**[W&B Alertsの完全なドキュメントはこちら](https://docs.wandb.ai/guides/track/alert)**でご覧いただけます。


```python
# wandb runを開始
wandb.init(project="pytorch-intro")

# モデルトレーニングループをシミュレート
acc_threshold = 0.3
for training_step in range(1000):

    # 精度のランダムな数値を生成
    accuracy = round(random.random() + random.random(), 3)
    print(f'Accuracy is: {accuracy}, {acc_threshold}')
    
    # 🐝 精度をwandbにログ
    wandb.log({"Accuracy": accuracy})

    # 🔔 精度が閾値以下の場合、W&Bアラートを発火しrunを停止
    if accuracy <= acc_threshold:
        # 🐝 wandbアラートを送信
        wandb.alert(
            title='Low Accuracy',
            text=f'Accuracy {accuracy} at step {training_step} is below the acceptable theshold, {acc_threshold}',
        )
        print('Alert triggered')
        break

# runを終了としてマーク（Jupyterノートブックには便利）
wandb.finish()
```

# 次は何をする？

次のチュートリアルでは、W&B Tablesを使用してモデルの予測を表示および分析する方法を学びます：
## 👉 [View & Analyze Model Predictions](tables)