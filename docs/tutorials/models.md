
# モデルを登録する

[**こちらのColabノートブックで試してみる →**](https://colab.research.google.com/github/wandb/examples/blob/master/colabs/wandb-model-registry/Model_Registry_E2E.ipynb)

モデルレジストリは、組織内で進行中のすべてのモデルタスクとそれに関連するArtifactsを管理し、整理する中心的な場所です。
- モデルチェックポイントの管理
- リッチなモデルカードでモデルを文書化
- 使用中および展開中のすべてのモデルの履歴を維持
- クリーンなハンドオフとステージ管理を促進
- さまざまなモデルタスクをタグ付けして整理
- モデルの進行に応じて自動通知を設定

このチュートリアルでは、シンプルな画像分類タスクのモデル開発ライフサイクルを追跡する方法を説明します。

### 🛠️ `wandb` のインストール

```bash
!pip install -q wandb onnx pytorch-lightning
```

## W&Bにログインする
- `wandb login` または `wandb.login()` を使用して明示的にログインできます（下記参照）。
- 代わりに環境変数を設定することもできます。W&Bのログの振る舞いを変更するために設定できるenv変数がいくつかあります。最も重要なのは次の通りです：
    - `WANDB_API_KEY` - プロフィールの「Settings」セクションで確認できます。
    - `WANDB_BASE_URL` - これはW&B サーバーのURLです。
- W&Bアプリの「Profile」->「Settings」でAPIトークンを確認します。

![api_token](https://drive.google.com/uc?export=view&id=1Xn7hnn0rfPu_EW0A_-32oCXqDmpA0-kx)

```notebook
!wandb login
```

:::note
[W&B Server](..//guides/hosting/intro.md) 展開に接続する場合（**Dedicated Cloud** または **Self-managed** のいずれか）、--reloginおよび--hostオプションを次のように使用します：

```notebook
!wandb login --relogin --host=http://your-shared-local-host.com
```

必要に応じて、展開管理者にホスト名を問い合わせてください。
:::

## アーティファクトとしてデータとモデルチェックポイントをログする

W&B Artifactsを使用すると、任意のシリアライズされたデータ（例：データセット、モデルチェックポイント、評価結果）の追跡とバージョン管理ができます。アーティファクトを作成するとき、名前とタイプを指定し、そのアーティファクトは実験のSoR（System of Record）に永遠にリンクされます。基礎となるデータが変更され、そのデータアセットを再度ログする場合、W&Bはその内容をチェックサムすることで自動的に新しいバージョンを作成します。W&B Artifactsは、共有の非構造化ファイルシステム上の軽量な抽象化レイヤーと考えることができます。

### アーティファクトの構造

`Artifact` クラスは、W&B Artifactレジストリ内のエントリに対応します。アーティファクトには
* 名前 
* タイプ
* メタデータ
* 説明
* ファイル、ファイルのディレクトリー、または参照

使用例：
```python
run = wandb.init(project="my-project")
artifact = wandb.Artifact(name="my_artifact", type="data")
artifact.add_file("/path/to/my/file.txt")
run.log_artifact(artifact)
run.finish()
```

このチュートリアルでは、最初にトレーニングデータセットをダウンロードし、それをトレーニングジョブで使用するアーティファクトとしてログします。

```python
# @title Enter your W&B project and entity

# FORM VARIABLES
PROJECT_NAME = "model-registry-tutorial"  # @param {type:"string"}
ENTITY = None  # @param {type:"string"}

# SIZEを"TINY"、"SMALL"、"MEDIUM"、または"LARGE"に設定
# これらの 3 つのデータセットのいずれかを選択します
# TINY dataset: 100 images, 30MB
# SMALL dataset: 1000 images, 312MB
# MEDIUM dataset: 5000 images, 1.5GB
# LARGE dataset: 12,000 images, 3.6GB

SIZE = "TINY"

if SIZE == "TINY":
    src_url = "https://storage.googleapis.com/wandb_datasets/nature_100.zip"
    src_zip = "nature_100.zip"
    DATA_SRC = "nature_100"
    IMAGES_PER_LABEL = 10
    BALANCED_SPLITS = {"train": 8, "val": 1, "test": 1}
elif SIZE == "SMALL":
    src_url = "https://storage.googleapis.com/wandb_datasets/nature_1K.zip"
    src_zip = "nature_1K.zip"
    DATA_SRC = "nature_1K"
    IMAGES_PER_LABEL = 100
    BALANCED_SPLITS = {"train": 80, "val": 10, "test": 10}
elif SIZE == "MEDIUM":
    src_url = "https://storage.googleapis.com/wandb_datasets/nature_12K.zip"
    src_zip = "nature_12K.zip"
    DATA_SRC = "inaturalist_12K/train"  # (technically a subset of only 10K images)
    IMAGES_PER_LABEL = 500
    BALANCED_SPLITS = {"train": 400, "val": 50, "test": 50}
elif SIZE == "LARGE":
    src_url = "https://storage.googleapis.com/wandb_datasets/nature_12K.zip"
    src_zip = "nature_12K.zip"
    DATA_SRC = "inaturalist_12K/train"  # (technically a subset of only 10K images)
    IMAGES_PER_LABEL = 1000
    BALANCED_SPLITS = {"train": 800, "val": 100, "test": 100}
```

```notebook
%%capture
!curl -SL $src_url > $src_zip
!unzip $src_zip
```

```python
import wandb
import pandas as pd
import os

with wandb.init(project=PROJECT_NAME, entity=ENTITY, job_type="log_datasets") as run:
    img_paths = []
    for root, dirs, files in os.walk("nature_100", topdown=False):
        for name in files:
            img_path = os.path.join(root, name)
            label = img_path.split("/")[1]
            img_paths.append([img_path, label])

    index_df = pd.DataFrame(columns=["image_path", "label"], data=img_paths)
    index_df.to_csv("index.csv", index=False)

    train_art = wandb.Artifact(
        name="Nature_100",
        type="raw_images",
        description="nature image dataset with 10 classes, 10 images per class",
    )
    train_art.add_dir("nature_100")

    # 各画像のラベルを示すcsvも追加
    train_art.add_file("index.csv")
    wandb.log_artifact(train_art)
```

### アーティファクト名とエイリアスを使用してデータ資産を簡単にハンドオフおよび抽象化する
- データセットやモデルの `name:alias` 組み合わせを参照するだけで、ワークフローのコンポーネントをより標準化できます。
- 例えば、PyTorch `Dataset` や `DataModule` を構築し、引数としてW&B アーティファクトの名前とエイリアスを受け取って適切にロードすることができます。

これで、このデータセットに関連するすべてのメタデータ、これを消費するW&B runs、および上流と下流のArtifactsの履歴をすべて確認できます！

![api_token](https://drive.google.com/uc?export=view&id=1fEEddXMkabgcgusja0g8zMz8whlP2Y5P)

```python
from torchvision import transforms
import pytorch_lightning as pl
import torch
from torch.utils.data import Dataset, DataLoader, random_split
from skimage import io, transform
from torchvision import transforms, utils, models
import math


class NatureDataset(Dataset):
    def __init__(
        self,
        wandb_run,
        artifact_name_alias="Nature_100:latest",
        local_target_dir="Nature_100:latest",
        transform=None,
    ):
        self.local_target_dir = local_target_dir
        self.transform = transform

        # アーティファクトをローカルにダウンロードしてメモリにロード
        art = wandb_run.use_artifact(artifact_name_alias)
        path_at = art.download(root=self.local_target_dir)

        self.ref_df = pd.read_csv(os.path.join(self.local_target_dir, "index.csv"))
        self.class_names = self.ref_df.iloc[:, 1].unique().tolist()
        self.idx_to_class = {k: v for k, v in enumerate(self.class_names)}
        self.class_to_idx = {v: k for k, v in enumerate(self.class_names)}

    def __len__(self):
        return len(self.ref_df)

    def __getitem__(self, idx):
        if torch.is_tensor(idx):
            idx = idx.tolist()

        img_path = self.ref_df.iloc[idx, 0]

        image = io.imread(img_path)
        label = self.ref_df.iloc[idx, 1]
        label = torch.tensor(self.class_to_idx[label], dtype=torch.long)

        if self.transform:
            image = self.transform(image)

        return image, label


class NatureDatasetModule(pl.LightningDataModule):
    def __init__(
        self,
        wandb_run,
        artifact_name_alias: str = "Nature_100:latest",
        local_target_dir: str = "Nature_100:latest",
        batch_size: int = 16,
        input_size: int = 224,
        seed: int = 42,
    ):
        super().__init__()
        self.wandb_run = wandb_run
        self.artifact_name_alias = artifact_name_alias
        self.local_target_dir = local_target_dir
        self.batch_size = batch_size
        self.input_size = input_size
        self.seed = seed

    def setup(self, stage=None):
        self.nature_dataset = NatureDataset(
            wandb_run=self.wandb_run,
            artifact_name_alias=self.artifact_name_alias,
            local_target_dir=self.local_target_dir,
            transform=transforms.Compose(
                [
                    transforms.ToTensor(),
                    transforms.CenterCrop(self.input_size),
                    transforms.Normalize((0.485, 0.456, 0.406), (0.229, 0.224, 0.225)),
                ]
            ),
        )

        nature_length = len(self.nature_dataset)
        train_size = math.floor(0.8 * nature_length)
        val_size = math.floor(0.2 * nature_length)
        self.nature_train, self.nature_val = random_split(
            self.nature_dataset,
            [train_size, val_size],
            generator=torch.Generator().manual_seed(self.seed),
        )
        return self

    def train_dataloader(self):
        return DataLoader(self.nature_train, batch_size=self.batch_size)

    def val_dataloader(self):
        return DataLoader(self.nature_val, batch_size=self.batch_size)

    def predict_dataloader(self):
        pass

    def teardown(self, stage: str):
        pass
```
## モデルのトレーニング

### モデルクラスとバリデーション関数の作成

```python
import torch.nn.functional as F
import torch.optim as optim
from torch.optim.lr_scheduler import StepLR
import onnx


def set_parameter_requires_grad(model, feature_extracting):
    if feature_extracting:
        for param in model.parameters():
            param.requires_grad = False


def initialize_model(model_name, num_classes, feature_extract, use_pretrained=True):
    # これらの変数を初期化します。各変数はモデル固有です。
    model_ft = None
    input_size = 0

    if model_name == "resnet":
        """Resnet18"""
        model_ft = models.resnet18(pretrained=use_pretrained)
        set_parameter_requires_grad(model_ft, feature_extract)
        num_ftrs = model_ft.fc.in_features
        model_ft.fc = torch.nn.Linear(num_ftrs, num_classes)
        input_size = 224

    elif model_name == "alexnet":
        """Alexnet"""
        model_ft = models.alexnet(pretrained=use_pretrained)
        set_parameter_requires_grad(model_ft, feature_extract)
        num_ftrs = model_ft.classifier[6].in_features
        model_ft.classifier[6] = torch.nn.Linear(num_ftrs, num_classes)
        input_size = 224

    elif model_name == "vgg":
        """VGG11_bn"""
        model_ft = models.vgg11_bn(pretrained=use_pretrained)
        set_parameter_requires_grad(model_ft, feature_extract)
        num_ftrs = model_ft.classifier[6].in_features
        model_ft.classifier[6] = torch.nn.Linear(num_ftrs, num_classes)
        input_size = 224

    elif model_name == "squeezenet":
        """Squeezenet"""
        model_ft = models.squeezenet1_0(pretrained=use_pretrained)
        set_parameter_requires_grad(model_ft, feature_extract)
        model_ft.classifier[1] = torch.nn.Conv2d(
            512, num_classes, kernel_size=(1, 1), stride=(1, 1)
        )
        model_ft.num_classes = num_classes
        input_size = 224

    elif model_name == "densenet":
        """Densenet"""
        model_ft = models.densenet121(pretrained=use_pretrained)
        set_parameter_requires_grad(model_ft, feature_extract)
        num_ftrs = model_ft.classifier.in_features
        model_ft.classifier = torch.nn.Linear(num_ftrs, num_classes)
        input_size = 224

    else:
        print("Invalid model name, exiting...")
        exit()

    return model_ft, input_size


class NaturePyTorchModule(torch.nn.Module):
    def __init__(self, model_name, num_classes=10, feature_extract=True, lr=0.01):
        """方法を使用してモデルのパラメーターを定義します"""
        super().__init__()

        self.model_name = model_name
        self.num_classes = num_classes
        self.feature_extract = feature_extract
        self.lr = lr
        self.model, self.input_size = initialize_model(
            model_name=self.model_name,
            num_classes=self.num_classes,
            feature_extract=True,
        )

    def forward(self, x):
        """推論用のメソッド input -> output"""
        x = self.model(x)

        return x


def evaluate_model(model, eval_data, idx_to_class, class_names, epoch_ndx):
    device = torch.device("cpu")
    model.eval()
    test_loss = 0
    correct = 0
    preds = []
    actual = []

    val_table = wandb.Table(columns=["pred", "actual", "image"])

    with torch.no_grad():
        for data, target in eval_data:
            data, target = data.to(device), target.to(device)
            output = model(data)
            test_loss += F.nll_loss(
                output, target, reduction="sum"
            ).item()  # sum up batch loss
            pred = output.argmax(
                dim=1, keepdim=True
            )  # get the index of the max log-probability
            preds += list(pred.flatten().tolist())
            actual += target.numpy().tolist()
            correct += pred.eq(target.view_as(pred)).sum().item()

            for idx, img in enumerate(data):
                img = img.numpy().transpose(1, 2, 0)
                pred_class = idx_to_class[pred.numpy()[idx][0]]
                target_class = idx_to_class[target.numpy()[idx]]
                val_table.add_data(pred_class, target_class, wandb.Image(img))

    test_loss /= len(eval_data.dataset)
    accuracy = 100.0% * correct / len(eval_data.dataset)
    conf_mat = wandb.plot.confusion_matrix(
        y_true=actual, preds=preds, class_names=class_names
    )
    return test_loss, accuracy, preds, val_table, conf_mat
```

### トレーニングループの追跡
トレーニング中は、トレーニングが中断されたり、インスタンスがクラッシュした場合でも途中から再開できるように、モデルを定期的にチェックポイントとして保存するのがベストプラクティスです。アーティファクトのログを使用すると、W&Bでチェックポイントをすべて追跡し、必要なメタデータ（シリアライズ形式、クラスラベルなど）を添付できます。これにより、チェックポイントを使用する必要がある人がその使い方を理解できます。モデルの形式のアーティファクトをログする場合、アーティファクトの `type` を `model` に設定することを忘れないでください。

```python
run = wandb.init(
    project=PROJECT_NAME,
    entity=ENTITY,
    job_type="training",
    config={
        "model_type": "squeezenet",
        "lr": 1.0,
        "gamma": 0.75,
        "batch_size": 16,
        "epochs": 5,
    },
)

model = NaturePyTorchModule(wandb.config["model_type"])
wandb.watch(model)

wandb.config["input_size"] = 224

nature_module = NatureDatasetModule(
    wandb_run=run,
    artifact_name_alias="Nature_100:latest",
    local_target_dir="Nature_100:latest",
    batch_size=wandb.config["batch_size"],
    input_size=wandb.config["input_size"],
)
nature_module.setup()

# モデルのトレーニング
learning_rate = wandb.config["lr"]
gamma = wandb.config["gamma"]
epochs = wandb.config["epochs"]

device