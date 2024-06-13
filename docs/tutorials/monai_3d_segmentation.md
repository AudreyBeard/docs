import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import { CTAButtons } from '@site/src/components/CTAButtons/CTAButtons.tsx';


# 3D脳腫瘍セグメンテーションをMONAIで実現

<CTAButtons colabLink="https://colab.research.google.com/github/wandb/examples/blob/main/colabs/monai/3d_brain_tumor_segmentation.ipynb"></CTAButtons>

このチュートリアルでは、[MONAI](https://github.com/Project-MONAI/MONAI)を使用して、多ラベル3D脳腫瘍セグメンテーションタスクのトレーニングワークフローを構築し、[Weights & Biases](https://wandb.ai/site)の実験トラッキングおよびデータ可視化機能を使用する方法を示します。チュートリアルには以下の機能が含まれています：

1. Weights & Biasesのrunを初期化し、再現性のためにrunに関連するすべての設定を同期します。
2. MONAIのトランスフォームAPI:
    1. 辞書形式のデータ用のMONAIトランスフォーム。
    2. MONAI`transforms`APIに従って新しいトランスフォームを定義する方法。
    3. データ増強のために強度をランダムに調整する方法。
3. データ読み込みと可視化:
    1. `Nifti`画像とメタデータを読み込み、画像のリストを読み込みそれらをスタックします。
    2. IOとトランスフォームをキャッシュしてトレーニングとバリデーションを加速します。
    3. `wandb.Table`を使用してデータを可視化し、Weights & Biasesでインタラクティブなセグメンテーションオーバーレイを行います。
4. 3D `SegResNet`モデルのトレーニング
    1. MONAIの`networks`、`losses`、`metrics` APIを使用。
    2. PyTorchトレーニングループを使用して3D `SegResNet`モデルをトレーニング。
    3. Weights & Biasesを使用してトレーニング実験をトラック。
    4. モデルアーティファクトとしてモデルのチェックポイントをWeights & Biasesにログし、バージョン管理。
5. `wandb.Table`を使用してバリデーションデータセットに対する予測を可視化し、インタラクティブなセグメンテーションオーバーレイで比較。

## 🌴 セットアップとインストール

まず、最新バージョンのMONAIとWeights & Biasesをインストールします。

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

次に、Colabインスタンスを認証してW&Bを使用します。

```python
wandb.login()
```

## 🌳 W&B Runを初期化

新しいW&B runを開始して、実験のトラッキングを開始します。

```python
wandb.init(project="monai-brain-tumor-segmentation")
```

再現性のある機械学習には、適切な設定システムの使用が推奨されます。W&Bを使用して、各実験のハイパーパラメーターをトラックできます。

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

モジュールのランダムシードを設定して、決定論的なトレーニングを可能または無効にする必要もあります。

```python
set_determinism(seed=config.seed)

# ディレクトリを作成
os.makedirs(config.dataset_dir, exist_ok=True)
os.makedirs(config.checkpoint_dir, exist_ok=True)
```

## 💿 データ読み込みと変換

ここでは、`monai.transforms` APIを使用して、マルチクラスラベルをワンホット形式のマルチラベルセグメンテーションタスクに変換するカスタムトランスフォームを作成します。

```python
class ConvertToMultiChannelBasedOnBratsClassesd(MapTransform):
    """
    Bratsクラスに基づいてラベルをマルチチャンネルに変換します：
    ラベル1は髄腫瘍周囲の浮腫、ラベル2はGD増強腫瘍、ラベル3は壊死および非増強腫瘍コアです。
    可能なクラスはTC（腫瘍コア）、WT（全腫瘍）、およびET（増強腫瘍）です。

    参考：https://github.com/Project-MONAI/tutorials/blob/main/3d_segmentation/brats_segmentation_3d.ipynb
    """

    def __call__(self, data):
        d = dict(data)
        for key in self.keys:
            result = []
            # ラベル2とラベル3をマージしてTCを構築
            result.append(torch.logical_or(d[key] == 2, d[key] == 3))
            # ラベル1, 2, 3をマージしてWTを構築
            result.append(
                torch.logical_or(
                    torch.logical_or(d[key] == 2, d[key] == 3), d[key] == 1
                )
            )
            # ラベル2はET
            result.append(d[key] == 2)
            d[key] = torch.stack(result, axis=0).float()
        return d
```

次に、トレーニングおよびバリデーションデータセットのトランスフォームを設定します。

```python
train_transform = Compose(
    [
        # 4つのNifti画像を読み込み、それらをスタック
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

この実験で使用されるデータセットはhttp://medicaldecathlon.com/ から入手できます。複数のサイトから取得されたマルチモードのMRIデータ（FLAIR、T1w、T1gd、T2w）を使用して、グリオーマ、壊死/活性腫瘍、および浮腫をセグメント化します。このデータセットは、750の4Dボリューム（トレーニング484 + テスト266）で構成されています。

`DecathlonDataset`を使用してデータセットを自動的にダウンロードおよび抽出します。これはMONAIの`CacheDataset`を継承しており、`cache_num=N`を設定してトレーニング用に`N`個のアイテムをキャッシュし、メモリサイズに応じてバリデーション用にすべてのアイテムをキャッシュできます。

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
**Note:** `train_transform`を`train_dataset`に適用する代わりに、トレーニングおよびバリデーションデータセットの両方に`val_transform`を適用します。これは、トレーニング前にデータセットの両方の分割からサンプルを可視化するためです。
:::

### 📸 データセットの可視化

Weights & Biasesは画像、ビデオ、オーディオなどをサポートしています。結果を調査し、ラン、モデル、データセットを視覚的に比較するために、リッチメディアをログできます。データボリュームを可視化するために、[セグメンテーションマスクオーバーレイシステム](https://docs.wandb.ai/guides/track/log/media#image-overlays-in-tables)を使用します。セグメンテーションマスクを[tables](https://docs.wandb.ai/guides/tables)にログするには、テーブルの各行に`wandb.Image`オブジェクトを提供する必要があります。

以下の擬似コードに例を示します：

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

次に、サンプル画像、ラベル、`wandb.Table`オブジェクト、および関連メタデータを受け取り、Weights & Biasesダッシュボードにログされるテーブルの行を埋めるユーティリティ関数を作成します。

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

次に、データビジュアライゼーションで埋めることができるように、`wandb.Table`オブジェクトとその列の構成を定義します。

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

次に、それぞれの`train_dataset`と`val_dataset`をループし、データサンプルのビジュアライゼーションを生成し、テーブルの行にログしてダッシュボードにログします。

```python
# train_datasetのビジュアライゼーションを生成
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

# val_datasetのビジュアライゼーションを生成
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

# テーブルをダッシュボードにログする
wandb.log({"Tumor-Segmentation-Data": table})
```

データはインタラクティブな表形式でW&Bダッシュボードに表示されます。データボリュームの特定のスライスの各チャンネルに対応する行ごとに、適切なセグメンテーションマスクがオーバーレイされていることが分かります。[Weaveクエリ](https://docs.wandb.ai/guides/weave)を記述してテーブル上のデータをフィルタリングし、特定の行に集中することができます。

| ![An example of logged table data.](@site/static/images/tutorials/monai/viz-1.gif) | 
|:--:| 
| **An example of logged table data.** |

画像を開き、インタラクティブなオーバーレイを使用して各セグメンテーションマスクと操作できます。

| ![An example of visualized segmentation maps.](@site/static/images/tutorials/monai/viz-2.gif) | 
|:--:| 
| **An example of visualized segmentation maps.* |

:::info
**Note:** データセットのラベルはクラス間で重複しないマスクで構成されています。オーバーレイはラベルをオーバーレイの別々のマスクとして記録します。
:::

### 🛫 データの読み込み

データセットからデータを読み込むためのPyTorch DataLoaderを作成します。DataLoaderを作成する前に、`train_dataset`の`transform`を`train_transform`に設定してデータをトレーニング用に前処理および変換します。

```python
# トレーニングデータセットに`train_transforms`を適用
train_dataset.transform = train_transform

# train_loaderを作成
train_loader = DataLoader(
    train_dataset,
    batch_size=config.batch_size,
   