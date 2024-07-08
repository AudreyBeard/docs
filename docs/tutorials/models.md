# Register models

[**Try in a Colab Notebook here →**](https://colab.research.google.com/github/wandb/examples/blob/master/colabs/wandb-model-registry/Model_Registry_E2E.ipynb)

モデルレジストリは、組織全体で作業中のすべてのモデルタスクと関連するArtifactsを集約し整理するための中央場所です:
- モデルチェックポイントの管理
- リッチなモデルカードでモデルを文書化
- 使用・展開されているすべてのモデルの履歴を保持
- モデルのスムーズな引き継ぎとステージ管理を促進
- さまざまなモデルタスクをタグ付けして整理
- モデルが進行する際に自動通知を設定

このチュートリアルでは、シンプルな画像分類タスクのモデル開発ライフサイクルを追跡する方法を解説します。

### 🛠️ `wandb` のインストール

```bash
!pip install -q wandb onnx pytorch-lightning
```

## W&Bへログイン
- `wandb login` または `wandb.login()` を使用して明示的にログインできます（以下を参照）
- あるいは環境変数を設定することもできます。W&Bのログの振る舞いを変更するために設定できる環境変数がいくつかあります。最も重要なものは以下の通りです:
    - `WANDB_API_KEY` - プロフィールの「設定」セクションで見つけることができます
    - `WANDB_BASE_URL` - これはW&BサーバーのURLです
- W&BアプリでAPIトークンを「プロフィール」 -> 「設定」で見つけてください

![api_token](https://drive.google.com/uc?export=view&id=1Xn7hnn0rfPu_EW0A_-32oCXqDmpA0-kx)

```notebook
!wandb login
```

:::note
[W&B Server](../guides/hosting/intro.md)デプロイメント（**Dedicated Cloud** または **Self-managed**）に接続する場合、--reloginおよび--hostオプションを使用します:

```notebook
!wandb login --relogin --host=http://your-shared-local-host.com
```

必要に応じて、デプロイメント管理者にホスト名を問い合わせてください。
:::

## データとモデルのチェックポイントをArtifactsとしてログ
W&B Artifactsを使用すると、データセット、モデルのチェックポイント、評価結果などの任意のシリアライズデータを追跡しバージョン管理することができます。アーティファクトを作成するとき、名前とタイプを指定し、そのアーティファクトは実験的なSoRに永続的にリンクされます。基礎データが変更されて再度ログ化されると、W&Bはその内容をチェックサムし、新しいバージョンを自動的に作成します。W&B Artifacts は、共有された非構造化ファイルシステムの上にある軽量な抽象化レイヤーと見なすことができます。

### アーティファクトの構造

`Artifact` クラスは、W&B Artifact レジストリ内のエントリに対応します。 アーティファクトには以下があります:
* 名前
* タイプ
* メタデータ
* 説明
* ファイル、ファイルのディレクトリ、または参照

使用例:
```python
run = wandb.init(project="my-project")
artifact = wandb.Artifact(name="my_artifact", type="data")
artifact.add_file("/path/to/my/file.txt")
run.log_artifact(artifact)
run.finish()
```

このチュートリアルでは、まずトレーニングデータセットをダウンロードし、それを後続のトレーニングジョブで使用するためのアーティファクトとしてログします。

```python
# @title W&Bのプロジェクトとエンティティを入力

# フォームの変数
PROJECT_NAME = "model-registry-tutorial"  # @param {type:"string"}
ENTITY = None  # @param {type:"string"}

# SIZEを"TINY"、"SMALL"、"MEDIUM"、"LARGE"に設定
# 以下のいずれかのデータセットを選択
# TINYデータセット: 100枚の画像、30MB
# SMALLデータセット: 1000枚の画像、312MB
# MEDIUMデータセット: 5000枚の画像、1.5GB
# LARGEデータセット: 12,000枚の画像、3.6GB

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
    DATA_SRC = "inaturalist_12K/train"  # (技術的には10K画像のみのサブセット)
    IMAGES_PER_LABEL = 500
    BALANCED_SPLITS = {"train": 400, "val": 50, "test": 50}
elif SIZE == "LARGE":
    src_url = "https://storage.googleapis.com/wandb_datasets/nature_12K.zip"
    src_zip = "nature_12K.zip"
    DATA_SRC = "inaturalist_12K/train"  # (技術的には10K画像のみのサブセット)
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

    # 各画像のラベルを示すCSVも追加
    train_art.add_file("index.csv")
    wandb.log_artifact(train_art)
```

### データ資産を簡単に引き継ぎや抽象化するためのアーティファクト名とエイリアスの使用
- データセットやモデルの`name:alias`コンビネーションを参照するだけでワークフローの一部を標準化できます
- 例えば、PyTorchの`Dataset`や`DataModule`を作成し、W&Bアーティファクトの名前とエイリアスを引数として適切にロードすることができます

これで、このデータセットに関連するすべてのメタデータ、消費するW&B runs、および上流と下流のアーティファクトの全リネージを確認できます！

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

## モデルトレーニング

### モデルクラスと検証関数の作成

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
    # これらの変数はこのif文で設定されます。それぞれの変数はモデル固有です。
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
        """モデルパラメータを定義するメソッド"""
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
        """推論に使用されるメソッド input -> output"""
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
            ).item()  # バッチ損失を合計
            pred = output.argmax(
                dim=1, keepdim=True
            )  # 最大のログ確率のインデックスを取得
            preds += list(pred.flatten().tolist())
            actual += target.numpy().tolist()
            correct += pred.eq(target.view_as(pred)).sum().item()

            for idx, img in enumerate(data):
                img = img.numpy().transpose(1, 2, 0)
                pred_class = idx_to_class[pred.numpy()[idx][0]]
                target_class = idx_to_class[target.numpy()[idx]]
                val_table.add_data(pred_class, target_class, wandb.Image(img))

    test_loss /= len(eval_data.dataset)
    accuracy = 100.0 * correct / len(eval_data.dataset)
    conf_mat = wandb.plot.confusion_matrix(
        y_true=actual, preds=preds, class_names=class_names
    )
    return test_loss, accuracy, preds, val_table, conf_mat
```

### トレーニングループの追跡
トレーニング中に、モデルのチェックポイントを定期的に保存することはベストプラクティスです。これにより、トレーニングが中断されたり、インスタンスがクラッシュしても、途中から再開することができます。アーティファクトのログを使用すると、チェックポイントすべてをW&Bで追跡し、必要なメタデータ（シリアライズの形式やクラスラベルなど）を添付することができます。したがって、誰かがチェックポイントを使用する必要が生じたときに、その使い方を知ることができます。どの形式のモデルをアーティファクトとしてログする際にも、そのアーティファクトの`type`を`model`に設定してください。

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

```python
# モデルをトレーニング
learning_rate = wandb.config["lr"]
gamma = wandb.config["gamma"]
epochs = wandb.config["epochs"]

device = torch.device("cpu")
optimizer = optim.Adadelta(model.parameters(), lr=wandb.config["lr"])
scheduler = StepLR(optimizer, step_size=1, gamma=wandb.config["gamma"])

best_loss = float("inf")
best_model = None

for epoch_ndx in range(epochs):
    model.train()
    for batch_ndx, batch in enumerate(nature_module.train_dataloader()):
        data, target = batch[0].to("cpu"), batch[1].to("cpu")
        optimizer.zero_grad()
        preds = model(data)
        loss = F.nll_loss(preds, target)
        loss.backward()
        optimizer.step()
        scheduler.step()

        ### メトリクスをログ ###
        wandb.log(
            {
                "train/epoch_ndx": epoch_ndx,
                "train/batch_ndx": batch_ndx,
                "train/train_loss": loss,
                "train/learning_rate": optimizer.param_groups[0]["lr"],
            }
        )
    
    ### 各エポック終了ごとの評価 ###
    model.eval()
    test_loss, accuracy, preds, val_table, conf_mat = evaluate_model(
        model,
        nature_module.val_dataloader(),
        nature_module.nature_dataset.idx_to_class,
        nature_module.nature_dataset.class_names,
        epoch_ndx,
    )

    is_best = test_loss < best_loss

    wandb.log(
        {
            "eval/test_loss": test_loss,
            "eval/accuracy": accuracy,
            "eval/conf_mat": conf_mat,
            "eval/val_table": val_table,
        }
    )

    ### モデルの重みをチェックポイント ###
    x = torch.randn(1, 3, 224, 224, requires_grad=True)
    torch.onnx.export(
        model,  # 実行するモデル
        x,  # モデルの入力（あるいは複数入力の場合はタプル）
        "model.onnx",  # モデルを保存する場所（ファイルまたはファイルのようなオブジェクト）
        export_params=True,  # トレーニング済みのパラメータをモデルファイルに保存
        opset_version=10,  # モデルをエクスポートするONNXバージョン
        do_constant_folding=True,  # 最適化のための定数フォールディングを実行するかどうか
        input_names=["input"],  # モデルの入力名
        output_names=["output"],  # モデルの出力名
        dynamic_axes={
            "input": {0: "batch_size"},  # 可変長の軸
            "output": {0: "batch_size"},
        },
    )

    art = wandb.Artifact(
        f"nature-{wandb.run.id}",
        type="model",
        metadata={
            "format": "onnx",
            "num_classes": len(nature_module.nature_dataset.class_names),
            "model_type": wandb.config["model_type"],
            "model_input_size": wandb.config["input_size"],
            "index_to_class": nature_module.nature_dataset.idx_to_class,
        },
    )

    art.add_file("model.onnx")

    ### ベストのチェックポイントを追跡するためにエイリアスを追加
    wandb.log_artifact(art, aliases=["best", "latest"] if is_best else None)
    if is_best:
        best_model = art
```

### プロジェクトのすべてのモデルチェックポイントを管理
![api_token](https://drive.google.com/uc?export=view&id=1z7nXRgqHTPYjfR1SoP-CkezyxklbAZlM) 
### 注意: W&Bオフラインとの同期
トレーニング中にネットワーク通信が失われた場合、`wandb sync`で進捗をいつでも同期できます。

W&B SDKはすべてのログデータを`wandb`というローカルディレクトリにキャッシュし、`wandb sync`を呼び出すと、ローカルの状態とWebアプリの状態が同期されます。
## モデルレジストリ
実験中に複数の runs で多数のチェックポイントをログした後、ワークフローの次の段階（例、テストやデプロイメント）にベストのチェックポイントを引き継ぐ時が来ます。

モデルレジストリは、個々のW&B Projectsの上位に存在する中央ページです。**Registered Models**（個々のW&B Projects内にある貴重なチェックポイントの「リンク」を保存するポートフォリオ）を収容します。

モデルレジストリは、すべてのモデルタスクのベストチェックポイントを集約する中央場所を提供します。ログされた任意の`model`アーティファクトは、**Registered Models**に「リンク」することができます。
### **Registered Models**の作成とUIを通じたリンク
#### 1. チームページに移動し、`Model Registry` を選択してチームのモデルレジストリにアクセスします

![model registry](https://drive.google.com/uc?export=view&id=1ZtJwBsFWPTm4Sg5w8vHhRpvDSeQPwsKw)

#### 2. 新しいRegistered Modelを作成します

![model registry](https://drive.google.com/uc?export=view&id=1RuayTZHNE0LJCxt1t0l6-2zjwiV4aDXe)

#### 3. すべてのモデルチェックポイントを保存しているプロジェクトのアーティファクトタブに移動します

![model registry](https://drive.google.com/uc?export=view&id=1LfTLrRNpBBPaUb_RmBIE7fWFMG0h3e0E)

#### 4. リンクしたいモデルアーティファクトバージョンで「Link to Registry」をクリックします
### APIを通じたRegistered Modelsの作成とリンク
`wandb.run.link_artifact` を使用して [API経由でモデルをリンク](https://docs.wandb.ai/guides/models) できます。アーティファクトオブジェクトと、**Registered Model** の名前、および追加したいエイリアスを渡します。**Registered Models**はW&B内でエンティティ（チーム）にスコープされているため、チームのメンバーのみがその**Registered Models**を閲覧しアクセスできます。API経由で登録モデル名を指定する際は、`<entity>/model-registry/<registered-model-name>` 形式を使用します。もしRegistered Modelが存在しなければ、自動的に作成されます。

```python
if ENTITY:
    wandb.run.link_artifact(
        best_model,
        f"{ENTITY}/model-registry/Model Registry Tutorial",
        aliases=["staging"],
    )
else:
    print("Must indicate entity where Registered Model will exist")
wandb.finish()
```

### 「リンク」とは？
レジストリにリンクすると、Registered Modelの新しいバージョンが作成され、それはプロジェクト内に存在するアーティファクトバージョンへのポインタです。W&Bはアーティファクトのバージョニングをプロジェクト内で、Registered Modelのバージョニングとは分けて管理します。モデルアーティファクトバージョンをリンクするプロセスは、そのアーティファクトバージョンをRegistered Modelタスクの下で「ブックマーク」するのと同じです。

通常、R&Dや実験中に研究者は100以上、場合によっては1000以上のモデルチェックポイントアーティファクトを生成しますが、それらのうち1つか2つだけが「日の目を見る」ことになります。これらのチェックポイントを別の、バージョン管理されたレジストリにリンクするプロセスは、モデルの開発側とモデルのデプロイ/消費側のワークフローを区別するのに役立ちます。モデルのバージョン/エイリアスはR&Dで生成されたすべての実験バージョンから汚染されることなく、Registered Modelのバージョン管理は、新しく「ブックマーク」されたモデルに基づいてインクリメントされます。
## すべてのモデルの中央ハブを作成
- モデルカード、タグ、Slack通知をRegistered Modelに追加
- モデルが異なるフェーズを進む際にエイリアスを変更
- モデルの文書化や回帰レポートのために、モデルレジストリをReportsに埋め込みます。このレポートを[例](https://api.wandb.ai/links/wandb-smle/r82bj9at)として参照
![model registry](https://drive.google.com/uc?export=view&id=1lKPgaw-Ak4WK_91aBMcLvUMJL6pDQpgO)

### 新しいモデルがレジストリにリンクされるときにSlack通知を設定

![model registry](https://drive.google.com/uc?export=view&id=1RsWCa6maJYD5y34gQ0nwWiKSWUCqcjT9)
## Registered Modelの利用
対応する`name:alias`を参照してAPI経由でRegistered Modelを利用できます。エンジニア、研究者、またはCI/CDプロセスの誰であっても、テストが必要なモデルやプロダクションに移行する必要があるモデルのために、モデルレジストリを中央ハブとして使用することができます。

```notebook
%%wandb -h 600

run = wandb.init(project=PROJECT_NAME, entity=ENTITY, job_type='inference')
artifact = run.use_artifact(f'{ENTITY}/model-registry/Model Registry Tutorial:staging', type='model')
artifact_dir = artifact.download()
wandb.finish()
```

