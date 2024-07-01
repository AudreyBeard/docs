import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import { CTAButtons } from '@site/src/components/CTAButtons/CTAButtons.tsx';

# 3D brain tumor segmentation with MONAI

<CTAButtons colabLink="https://colab.research.google.com/github/wandb/examples/blob/main/colabs/monai/3d_brain_tumor_segmentation.ipynb"></CTAButtons>

このチュートリアルでは、[MONAI](https://github.com/Project-MONAI/MONAI) を使用してマルチラベル3D脳腫瘍セグメンテーションタスクのトレーニングワークフローを構築し、実験管理およびデータ可視化機能を [Weights & Biases](https://wandb.ai/site) で利用する方法を示します。チュートリアルには以下の機能が含まれています：

1. Weights & Biases の run を初期化し、再現性のために run に関連するすべての設定を同期します。
2. MONAI transform API:
    1. 辞書形式のデータに対する MONAI Transforms。
    2. MONAI `transforms` API に従って新しい変換を定義する方法。
    3. データ拡張のために強度をランダムに調整する方法。
3. データの読み込みと可視化:
    1. メタデータを含む `Nifti` 画像の読み込み、一連の画像の読み込みとスタック。
    2. トレーニングと検証を高速化するためのキャッシュ I/O と変換。
    3. `wandb.Table` と Weights & Biases のインタラクティブなセグメンテーションオーバーレイを使用したデータの可視化。
4. 3D `SegResNet` モデルのトレーニング
    1. MONAI の `networks`、`losses`、`metrics` API を使用。
    2. PyTorch トレーニングループを使用して 3D `SegResNet` モデルをトレーニング。
    3. Weights & Biases を使用してトレーニング実験を追跡。
    4. Weights & Biases でモデルのアーティファクトとしてチェックポイントをログおよびバージョン管理。
5. `wandb.Table` と Weights & Biases のインタラクティブなセグメンテーションオーバーレイを使用して、検証データセットに対する予測を可視化および比較。

## 🌴 セットアップとインストール

最初に、MONAI と Weights and Biases の最新バージョンをインストールします。

```python
!python -c "import monai" || pip install -q -U "monai[nibabel, tqdm]"
!python -c "import wandb" || pip install -q -U wandb
```

```python
import os

import numpy as np
from tqdm.auto import tqdm
import wandb

from monai.apps import DecathlonDataset
from monai.data import DataLoader, decollate_batch
from monai.losses import DiceLoss
from monai.inferers import sliding_window_inference
from monai.metrics import DiceMetric
from monai.networks.nets import SegResNet
from monai.transforms import (
    Activations,
    AsDiscrete,
    Compose,
    LoadImaged,
    MapTransform,
    NormalizeIntensityd,
    Orientationd,
    RandFlipd,
    RandScaleIntensityd,
    RandShiftIntensityd,
    RandSpatialCropd,
    Spacingd,
    EnsureTyped,
    EnsureChannelFirstd,
)
from monai.utils import set_determinism

import torch
```

次に、Colab インスタンスを認証して W&B を使用します。

```python
wandb.login()
```

## 🌳 W&B Run の初期化

新しい W&B run を開始して実験を追跡します。

```python
wandb.init(project="monai-brain-tumor-segmentation")
```

適切な設定システムの使用は、再現性のある機械学習のベストプラクティスとして推奨されます。W&B を使用して各実験のハイパーパラメーターを追跡できます。

```python
config = wandb.config
config.seed = 0
config.roi_size = [224, 224, 144]
config.batch_size = 1
config.num_workers = 4
config.max_train_images_visualized = 20
config.max_val_images_visualized = 20
config.dice_loss_smoothen_numerator = 0
config.dice_loss_smoothen_denominator = 1e-5
config.dice_loss_squared_prediction = True
config.dice_loss_target_onehot = False
config.dice_loss_apply_sigmoid = True
config.initial_learning_rate = 1e-4
config.weight_decay = 1e-5
config.max_train_epochs = 50
config.validation_intervals = 1
config.dataset_dir = "./dataset/"
config.checkpoint_dir = "./checkpoints"
config.inference_roi_size = (128, 128, 64)
config.max_prediction_images_visualized = 20
```

また、ランダムシードをモジュールに設定して、決定論的トレーニングを有効または無効にする必要があります。

```python
set_determinism(seed=config.seed)

# ディレクトリの作成
os.makedirs(config.dataset_dir, exist_ok=True)
os.makedirs(config.checkpoint_dir, exist_ok=True)
```

## 💿 データの読み込みと変換

ここでは、`monai.transforms` API を使用して、マルチクラスラベルをワンホット形式のマルチラベルセグメンテーションタスクに変換するカスタム変換を作成します。

```python
class ConvertToMultiChannelBasedOnBratsClassesd(MapTransform):
    """
    Brats クラスに基づいてラベルをマルチチャネルに変換します:
    ラベル 1 は周囲浮腫、
    ラベル 2 は GD 増強腫瘍、
    ラベル 3 は壊死および非増強腫瘍核です。
    可能なクラスは TC（腫瘍核）、WT（全腫瘍）
    及び ET（増強腫瘍）です。

    参考: https://github.com/Project-MONAI/tutorials/blob/main/3d_segmentation/brats_segmentation_3d.ipynb

    """

    def __call__(self, data):
        d = dict(data)
        for key in self.keys:
            result = []
            # ラベル 2 とラベル 3 をマージして TC を構築
            result.append(torch.logical_or(d[key] == 2, d[key] == 3))
            # ラベル 1, 2, 3 をマージして WT を構築
            result.append(
                torch.logical_or(
                    torch.logical_or(d[key] == 2, d[key] == 3), d[key] == 1
                )
            )
            # ラベル 2 は ET
            result.append(d[key] == 2)
            d[key] = torch.stack(result, axis=0).float()
        return d
```

次に、トレーニングおよび検証データセットに対する変換をそれぞれ設定します。

```python
train_transform = Compose(
    [
        # 4 つの Nifti 画像を読み込み、一緒にスタックする
        LoadImaged(keys=["image", "label"]),
        EnsureChannelFirstd(keys="image"),
        EnsureTyped(keys=["image", "label"]),
        ConvertToMultiChannelBasedOnBratsClassesd(keys="label"),
        Orientationd(keys=["image", "label"], axcodes="RAS"),
        Spacingd(
            keys=["image", "label"],
            pixdim=(1.0, 1.0, 1.0),
            mode=("bilinear", "nearest"),
        ),
        RandSpatialCropd(
            keys=["image", "label"], roi_size=config.roi_size, random_size=False
        ),
        RandFlipd(keys=["image", "label"], prob=0.5, spatial_axis=0),
        RandFlipd(keys=["image", "label"], prob=0.5, spatial_axis=1),
        RandFlipd(keys=["image", "label"], prob=0.5, spatial_axis=2),
        NormalizeIntensityd(keys="image", nonzero=True, channel_wise=True),
        RandScaleIntensityd(keys="image", factors=0.1, prob=1.0),
        RandShiftIntensityd(keys="image", offsets=0.1, prob=1.0),
    ]
)
val_transform = Compose(
    [
        LoadImaged(keys=["image", "label"]),
        EnsureChannelFirstd(keys="image"),
        EnsureTyped(keys=["image", "label"]),
        ConvertToMultiChannelBasedOnBratsClassesd(keys="label"),
        Orientationd(keys=["image", "label"], axcodes="RAS"),
        Spacingd(
            keys=["image", "label"],
            pixdim=(1.0, 1.0, 1.0),
            mode=("bilinear", "nearest"),
        ),
        NormalizeIntensityd(keys="image", nonzero=True, channel_wise=True),
    ]
)
```

### 🍁 データセット

この実験に使用するデータセットは、http://medicaldecathlon.com/ から取得します。このデータセットは、複数のモーダル、多サイト MRI データ（FLAIR、T1w、T1gd、T2w）を使用して、神経膠腫、壊死/活動中の腫瘍、および浮腫をセグメント化します。データセットは 750 の 4D ボリューム（トレーニング 484 + テスト 266）で構成されています。

`DecathlonDataset` を使用してデータセットを自動的にダウンロードして抽出します。これは、MONAI の `CacheDataset` を継承しており、メモリサイズに応じて、トレーニング用に項目を `cache_num=N` で保存し、検証用にすべての項目をキャッシュするデフォルトの引数を使用できます。

```python
train_dataset = DecathlonDataset(
    root_dir=config.dataset_dir,
    task="Task01_BrainTumour",
    transform=val_transform,
    section="training",
    download=True,
    cache_rate=0.0,
    num_workers=4,
)
val_dataset = DecathlonDataset(
    root_dir=config.dataset_dir,
    task="Task01_BrainTumour",
    transform=val_transform,
    section="validation",
    download=False,
    cache_rate=0.0,
    num_workers=4,
)
```

:::info
**Note:** `train_dataset` に `train_transform` を適用する代わりに、検証データセットにも `val_transform` を適用します。これは、トレーニング前に、データセットの両方のスプリットからサンプルを可視化するためです。
:::

### 📸 データセットの可視化

Weights & Biases は画像、動画、オーディオなどをサポートしています。リッチメディアをログして結果を探索し、run、モデル、およびデータセットを視覚的に比較できます。[セグメンテーションマスクオーバーレイシステム](https://docs.wandb.ai/guides/track/log/media#image-overlays-in-tables) を使用してデータボリュームを可視化します。[tables](https://docs.wandb.ai/guides/tables) にセグメンテーションマスクをログするには、各行に `wandb.Image` オブジェクトを提供する必要があります。

擬似コードの例を以下に示します：

```python
table = wandb.Table(columns=["ID", "Image"])

for id, img, label in zip(ids, images, labels):
    mask_img = wandb.Image(
        img,
        masks={
            "prediction": {"mask_data": label, "class_labels": class_labels}
            # ...
        },
    )

    table.add_data(id, img)

wandb.log({"Table": table})
```

次に、サンプル画像、ラベル、`wandb.Table` オブジェクト、および関連するメタデータを受け取り、Weights & Biases ダッシュボードにログするテーブルの行を埋める簡単なユーティリティ関数を書きます。

```python
def log_data_samples_into_tables(
    sample_image: np.array,
    sample_label: np.array,
    split: str = None,
    data_idx: int = None,
    table: wandb.Table = None,
):
    num_channels, _, _, num_slices = sample_image.shape
    with tqdm(total=num_slices, leave=False) as progress_bar:
        for slice_idx in range(num_slices):
            ground_truth_wandb_images = []
            for channel_idx in range(num_channels):
                ground_truth_wandb_images.append(
                    masks = {
                        "ground-truth/Tumor-Core": {
                            "mask_data": sample_label[0, :, :, slice_idx],
                            "class_labels": {0: "background", 1: "Tumor Core"},
                        },
                        "ground-truth/Whole-Tumor": {
                            "mask_data": sample_label[1, :, :, slice_idx] * 2,
                            "class_labels": {0: "background", 2: "Whole Tumor"},
                        },
                        "ground-truth/Enhancing-Tumor": {
                            "mask_data": sample_label[2, :, :, slice_idx] * 3,
                            "class_labels": {0: "background", 3: "Enhancing Tumor"},
                        },
                    }
                    wandb.Image(
                        sample_image[channel_idx, :, :, slice_idx],
                        masks=masks,
                    )
                )
            table.add_data(split, data_idx, slice_idx, *ground_truth_wandb_images)
            progress_bar.update(1)
    return table
```

次に、`wandb.Table` オブジェクトとその列の定義を行い、データビジュアライゼーションで各列を埋めるようにします。

```python
table = wandb.Table(
    columns=[
        "Split",
        "Data Index",
        "Slice Index",
        "Image-Channel-0",
        "Image-Channel-1",
        "Image-Channel-2",
        "Image-Channel-3",
    ]
)
```

次に`train_dataset` と `val_dataset` をループして、データサンプルのビジュアライゼーションを生成し、ダッシュボードにログするテーブル行を埋めます。

```python
# train_dataset のビジュアライゼーションを生成
max_samples = (
    min(config.max_train_images_visualized, len(train_dataset))
    if config.max_train_images_visualized > 0
    else len(train_dataset)
)
progress_bar = tqdm(
    enumerate(train_dataset[:max_samples]),
    total=max_samples,
    desc="Generating Train Dataset Visualizations:",
)
for data_idx, sample in progress_bar:
    sample_image = sample["image"].detach().cpu().numpy()
    sample_label = sample["label"].detach().cpu().numpy()
    table = log_data_samples_into_tables(
        sample_image,
        sample_label,
        split="train",
        data_idx=data_idx,
        table=table,
    )

# val_dataset のビジュアライゼーションを生成
max_samples = (
    min(config.max_val_images_visualized, len(val_dataset))
    if config.max_val_images_visualized > 0
    else len(val_dataset)
)
progress_bar = tqdm(
    enumerate(val_dataset[:max_samples]),
    total=max_samples,
    desc="Generating Validation Dataset Visualizations:",
)
for data_idx, sample in progress_bar:
    sample_image = sample["image"].detach().cpu().numpy()
    sample_label = sample["label"].detach().cpu().numpy()
    table = log_data_samples_into_tables(
        sample_image,
        sample_label,
        split="val",
        data_idx=data_idx,
        table=table,
    )

# ダッシュボードにテーブルをログ
wandb.log({"Tumor-Segmentation-Data": table})
```

データは W&B のダッシュボード上でインタラクティブな表形式で表示されます。各チャネルの特定のスライスを、各行の該当するセグメンテーションマスクとともに確認できます。[Weave クエリ](https://docs.wandb.ai/guides/weave) を書いて、テーブル上のデータをフィルタリングし、特定の行に焦点を当てることができます。

| ![An example of logged table data.](@site/static/images/tutorials/monai/viz-1.gif) | 
|:--:| 
| **An example of logged table data.** |

画像を開いて、インタラクティブなオーバーレイを使用して各セグメンテーションマスクをどのように操作できるかを確認します。

| ![An example of visualized segmentation maps.](@site/static/images/tutorials/monai/viz-2.gif) | 
|:--:| 
| **An example of visualized segmentation maps.** |

:::info
**Note:** データセット内のラベルはクラス間で重複しないマスクで構成されています。オーバーレイは、ラベルをオーバーレイ内の個別のマスクとしてログします。
:::

### 🛫 データの読み込み

データセットからデータを読み込むための PyTorch DataLoader を作成します。DataLoader を作成する前に、`train_dataset` の `transform` を `train_transform` に設定して、トレーニングデータを前処理および変換します。

```python
# トレーニングデータセットに train_transforms を適用
train_dataset.transform = train_transform

# train_loader の作成
train_loader = DataLoader(
    train_dataset,
    batch_size=config.batch_size,
    shuffle=True,
    num_workers=config.num_workers,
)

# val_loader の作成
val_loader = DataLoader(
    val_dataset,
    batch_size=config.batch_size,
    shuffle=False,
    num_workers=config.num_workers,
)
```

## 🤖 モデル、ロス、およびオプティマイザーの作成

このチュートリアルでは、論文 [3D MRI brain tumor segmentation using auto-encoder regularization](https://arxiv.org/pdf/1810.11654.pdf) に基づいた `SegResNet` モデルを作成します。`SegResNet` モデルは、`monai.networks` API の一部として PyTorch モジュールとして実装されています。また、オプティマイザーと学習率スケジューラーも作成します。

```python
device = torch.device("cuda:0")

# モデルの作成
model = SegResNet(
    blocks_down=[1, 2, 2, 4],
    blocks_up=[1, 1, 1],
    init_filters=16,
    in_channels=4,
    out_channels=3,
    dropout_prob=0.2,
).to(device)

# オプティマイザーの作成
optimizer = torch.optim.Adam(
    model.parameters(),
    config.initial_learning_rate,
    weight_decay=config.weight_decay,
)

# 学習率スケジューラーの作成
lr_scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(
    optimizer, T_max=config.max_train_epochs
)
```

`monai.losses` API を使用してマルチラベル `DiceLoss` をロスとして定義し、対応するダイスメトリクスを `monai.metrics` API を使用して定義します。

```python
loss_function = DiceLoss(
    smooth_nr=config.dice_loss_smoothen_numerator,
    smooth_dr=config.dice_loss_smoothen_denominator,
    squared_pred=config.dice_loss_squared_prediction,
    to_onehot_y=config.dice_loss_target_onehot,
    sigmoid=config.dice_loss_apply_sigmoid,
)

dice_metric = DiceMetric(include_background=True, reduction="mean")
dice_metric_batch = DiceMetric(include_background=True, reduction="mean_batch")
post_trans = Compose([Activations(sigmoid=True), AsDiscrete(threshold=0.5)])

# 自動混合精度を使用してトレーニングを高速化
scaler = torch.cuda.amp.GradScaler()
torch.backends.cudnn.benchmark = True
```

小さなユーティリティを定義して混合精度の推論を行います。これはトレーニングプロセスの検証ステップや、トレーニング後にモデルを実行するときに役立ちます。

```python
def inference(model, input):
    def _compute(input):
        return sliding_window_inference(
            inputs=input,
            roi_size=(240, 240, 160),
            sw_batch_size=1,
            predictor=model,
            overlap=0.5,
        )

    with torch.cuda.amp.autocast():
        return _compute(input)
```

## 🚝 トレーニングと検証

トレーニング前に、後で `wandb.log()` を使用してトレーニングおよび検証実験を追跡するために、メトリクスのプロパティを定義します。

```python
wandb.define_metric("epoch/epoch_step")
wandb.define_metric("epoch/*", step_metric="epoch/epoch_step")
wandb.define_metric("batch/batch_step")
wandb.define_metric("batch/*", step_metric="batch/batch_step")
wandb.define_metric("validation/validation_step")
wandb.define_metric("validation/*", step_metric="validation/validation_step")

batch_step = 0
validation_step = 0
metric_values = []
metric_values_tumor_core = []
metric_values_whole_tumor = []
metric_values_enhanced_tumor = []
```

### 🍭 標準の PyTorch トレーニングループを実行

```python
# W&B アーティファクトオブジェクトを定義
artifact = wandb.Artifact(
    name=f"{wandb.run.id}-checkpoint", type="model"
)

epoch_progress_bar = tqdm(range(config.max_train_epochs), desc="Training:")

for epoch in epoch_progress_bar:
    model.train()
    epoch_loss = 0

    total_batch_steps = len(train_dataset) // train_loader.batch_size
    batch_progress_bar = tqdm(train_loader, total=total_batch_steps, leave=False)
    
    # トレーニングステップ
    for batch_data in batch_progress_bar:
        inputs, labels = (
            batch_data["image"].to(device),
            batch_data["label"].to(device),
        )
        optimizer.zero_grad()
        with torch.cuda.amp.autocast():
            outputs = model(inputs)
            loss = loss_function(outputs, labels)
        scaler.scale(loss).backward()
        scaler.step(optimizer)
        scaler.update()
        epoch_loss += loss.item()
        batch_progress_bar.set_description(f"train_loss: {loss.item():.4f}:")
        ## バッチごとのトレーニングロスを W&B にログ
        wandb.log({"batch/batch_step": batch_step, "batch/train_loss": loss.item()})
        batch_step += 1

    lr_scheduler.step()
    epoch_loss /= total_batch_steps
    ## エポックごとのトレーニングロスおよび学習率を W&B にログ
    wandb.log(
        {
            "epoch/epoch_step": epoch,
            "epoch/mean_train_loss": epoch_loss,
            "epoch/learning_rate": lr_scheduler.get_last_lr()[0],
        }
    )
    epoch_progress_bar.set_description(f"Training: train_loss: {epoch_loss:.4f}:")

    # 検証とモデルチェックポイントステップ
    if (epoch + 1) % config.validation_intervals == 0:
        model.eval()
        with torch.no_grad():
            for val_data in val_loader:
                val_inputs, val_labels = (
                    val_data["image"].to(device),
                    val_data["label"].to(device),
                )
                val_outputs = inference(model, val_inputs)
                val_outputs = [post_trans(i) for i in decollate_batch(val_outputs)]
                dice_metric(y_pred=val_outputs, y=val_labels)
                dice_metric_batch(y_pred=val_outputs, y=val_labels)

            metric_values.append(dice_metric.aggregate().item())
            metric_batch = dice_metric_batch.aggregate()
            metric_values_tumor_core.append(metric_batch[0].item())
            metric_values_whole_tumor.append(metric_batch[1].item())
            metric_values_enhanced_tumor.append(metric_batch[2].item())
            dice_metric.reset()
            dice_metric_batch.reset()

            checkpoint_path = os.path.join(config.checkpoint_dir, "model.pth")
            torch.save(model.state_dict(), checkpoint_path)
            
            # W&B アーティファクトを使用してモデルチェックポイントをログおよびバージョン管理
            artifact.add_file(local_path=checkpoint_path)
            wandb.log_artifact(artifact, aliases=[f"epoch_{epoch}"])

            # 検証メトリクスを W&B ダッシュボードにログ
            wandb.log(
                {
                    "validation/validation_step": validation_step,
                    "validation/mean_dice": metric_values[-1],
                    "validation/mean_dice_tumor_core": metric_values_tumor_core[-1],
                    "validation/mean_dice_whole_tumor": metric_values_whole_tumor[-1],
                    "validation/mean_dice_enhanced_tumor": metric_values_enhanced_tumor[-1],
                }
            )
            validation_step += 1


# このアーティファクトのログが完了するまで待つ
artifact.wait()
```

`wandb.log` をコードに組み込むことで、トレーニングおよび検証プロセスに関連するすべてのメトリクスだけでなく、すべてのシステムメトリクス（この場合は CPU と GPU）を W&B ダッシュボードで追跡できます。

| ![An example of training and validation process tracking on W&B.](@site/static/images/tutorials/monai/viz-3.gif) | 
|:--:| 
| **An example of training and validation process tracking on W&B.** |

トレーニング中にログされた異なるバージョンのモデルチェックポイントアーティファクトにアクセスするには、W&B run ダッシュボードのアーティファクトタブに移動します。

| ![An example of model checkpoints logging and versioning on W&B.](@site/static/images/tutorials/monai/viz-4.gif) | 
|:--:| 
| **An example of model checkpoints logging and versioning on W&B.** |

## 🔱 推論

アーティファクトインターフェースを使用して、エポックごとの平均トレーニングロスが最良のモデルチェックポイントであるバージョンを選択できます。また、アーティファクトのリネージ全体を探索し、必要なバージョンを使用することができます。

| ![An example of model artifact tracking on W&B.](@site/static/images/tutorials/monai/viz-5.gif) | 
|:--:| 
| **An example of model artifact tracking on W&B.** |

エポックごとの平均トレーニングロスが最良のモデルアーティファクトのバージョンを取得し、モデルにチェックポイントの状態辞書を読み込みます。

```python
model_artifact = wandb.use_artifact(
    "geekyrakshit/monai-brain-tumor-segmentation/d5ex6n4a-checkpoint:v49",
    type="model",
)
model_artifact_dir = model_artifact.download()
model.load_state_dict(torch.load(os.path.join(model_artifact_dir, "model.pth")))
model.eval()
```

### 📸 予測の可視化とグランドトゥルースラベルとの比較

事前トレーニング済みモデルの予測を可視化し、対応する正解セグメンテーションマスクとの比較をインタラクティブなセグメンテーションマスクオーバーレイを使用して行うためのユーティリティ関数を作成します。

```python
def log_predictions_into_tables(
    sample_image: np.array,
    sample_label: np.array,
    predicted_label: np.array,
    split: str = None,
    data_idx: int = None,
    table: wandb.Table = None,
):
    num_channels, _, _, num_slices = sample_image.shape
    with tqdm(total=num_slices, leave=False) as progress_bar:
        for slice_idx in range(num_slices):
            wandb_images = []
            for channel_idx in range(num_channels):
                wandb_images += [
                    wandb.Image(
                        sample_image[channel_idx, :, :, slice_idx],
                        masks={
                            "ground-truth/Tumor-Core": {
                                "mask_data": sample_label[0, :, :, slice_idx],
                                "class_labels": {0: "background", 1: "Tumor Core"},
                            },
                            "prediction/Tumor-Core": {
                                "mask_data": predicted_label[0, :, :, slice_idx] * 2,
                                "class_labels": {0: "background", 2: "Tumor Core"},
                            },
                        },
                    ),
                    wandb.Image(
                        sample_image[channel_idx, :, :, slice_idx],
                        masks={
                            "ground-truth/Whole-Tumor": {
                                "mask_data": sample_label[1, :, :, slice_idx],
                                "class_labels": {0: "background", 1: "Whole Tumor"},
                            },
                            "prediction/Whole-Tumor": {
                                "mask_data": predicted_label[1, :, :, slice_idx] * 2,
                                "class_labels": {0: "background", 2: "Whole Tumor"},
                            },
                        },
                    ),
                    wandb.Image(
                        sample_image[channel_idx, :, :, slice_idx],
                        masks={
                            "ground-truth/Enhancing-Tumor": {
                                "mask_data": sample_label[2, :, :, slice_idx],
                                "class_labels": {0: "background", 1: "Enhancing Tumor"},
                            },
                            "prediction/Enhancing-Tumor": {
                                "mask_data": predicted_label[2, :, :, slice_idx] * 2,
                                "class_labels": {0: "background", 2: "Enhancing Tumor"},
                            },
                        },
                    ),
                ]
            table.add_data(split, data_idx, slice_idx, *wandb_images)
            progress_bar.update(1)
    return table
```

予測結果を予測テーブルに記録します。

```python
# 予測テーブルの作成
prediction_table = wandb.Table(
    columns=[
        "Split",
        "Data Index",
        "Slice Index",
        "Image-Channel-0/Tumor-Core",
        "Image-Channel-1/Tumor-Core",
        "Image-Channel-2/Tumor-Core",
        "Image-Channel-3/Tumor-Core",
        "Image-Channel-0/Whole-Tumor",
        "Image-Channel-1/Whole-Tumor",
        "Image-Channel-2/Whole-Tumor",
        "Image-Channel-3/Whole-Tumor",
        "Image-Channel-0/Enhancing-Tumor",
        "Image-Channel-1/Enhancing-Tumor",
        "Image-Channel-2/Enhancing-Tumor",
        "Image-Channel-3/Enhancing-Tumor",
    ]
)

# 推論と可視化の実行
with torch.no_grad():
    config.max_prediction_images_visualized
    max_samples = (
        min(config.max_prediction_images_visualized, len(val_dataset))
        if config.max_prediction_images_visualized > 0
        else len(val_dataset)
    )
    progress_bar = tqdm(
        enumerate(val_dataset[:max_samples]),
        total=max_samples,
        desc="Generating Predictions:",
    )
    for data_idx, sample in progress_bar:
        val_input = sample["image"].unsqueeze(0).to(device)
        val_output = inference(model, val_input)
        val_output = post_trans(val_output[0])
        prediction_table = log_predictions_into_tables(
            sample_image=sample["image"].cpu().numpy(),
            sample_label=sample["label"].cpu().numpy(),
            predicted_label=val_output.cpu().numpy(),
            data_idx=data_idx,
            split="validation",
            table=prediction_table,
        )

    wandb.log({"Predictions/Tumor-Segmentation-Data": prediction_table})


# 実験を終了
wandb.finish()
```

インタラクティブなセグメンテーションマスクオーバーレイを使用して、各クラスの予測セグメンテーションマスクと正解ラベルを分析および比較します。

| ![An example of predictions and ground-truth visualization on W&B.](@site/static/images/tutorials/monai/viz-6.gif) | 
|:--:| 
| **An example of predictions and ground-truth visualization on W&B.** |

## 謝辞およびその他のリソース

