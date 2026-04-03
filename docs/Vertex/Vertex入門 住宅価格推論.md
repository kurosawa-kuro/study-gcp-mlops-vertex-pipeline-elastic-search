# Vertex AI 入門: カリフォルニア住宅価格推論チュートリアル

scikit-learn RandomForest を Vertex AI でトレーニング・デプロイし、オンライン推論するまでの手順。

---

## 目次

1. [前提条件](#前提条件)
2. [アーキテクチャ概要](#アーキテクチャ概要)
3. [環境セットアップ](#環境セットアップ)
4. [Step 1: トレーニングスクリプトの作成](#step-1-トレーニングスクリプトの作成)
5. [Step 2: コンテナイメージのビルド](#step-2-コンテナイメージのビルド)
6. [Step 3: Vertex AI Custom Training Job の実行](#step-3-vertex-ai-custom-training-job-の実行)
7. [Step 4: Model Registry への登録](#step-4-model-registry-への登録)
8. [Step 5: エンドポイントへのデプロイ](#step-5-エンドポイントへのデプロイ)
9. [Step 6: オンライン推論](#step-6-オンライン推論)
10. [Step 7: バッチ推論](#step-7-バッチ推論)
11. [Step 8: リソースのクリーンアップ](#step-8-リソースのクリーンアップ)
12. [補足: Vertex AI Pipelines で自動化](#補足-vertex-ai-pipelines-で自動化)

---

## 前提条件

- GCP プロジェクト: `mlops-dev-a`
- リージョン: `asia-northeast1`
- 有効化が必要な API:
  - Vertex AI API
  - Artifact Registry API
  - Cloud Storage API

```bash
gcloud services enable aiplatform.googleapis.com \
    artifactregistry.googleapis.com \
    storage.googleapis.com \
    --project=mlops-dev-a
```

## アーキテクチャ概要

```
[Vertex AI Custom Training Job]
   ├── Kaggle California Housing データ取得（sklearn.datasets）
   ├── 特徴量前処理 + 学習（RandomForestRegressor）
   ├── 評価（RMSE, MAE）
   └── モデル保存 → [GCS gs://mlops-dev-a-vertex/models/]

[Vertex AI Model Registry]
   └── GCS のモデルを登録

[Vertex AI Endpoint]
   └── モデルをデプロイ → オンライン推論 POST リクエスト
```

**現行の Cloud Run ベースと比較:**

| 項目 | 現行（Cloud Run） | Vertex AI |
|------|-------------------|-----------|
| トレーニング実行基盤 | Cloud Run Job | Vertex AI Custom Training |
| モデル管理 | BigQuery metrics テーブル | Vertex AI Model Registry |
| 推論サービング | Cloud Run Service + FastAPI | Vertex AI Endpoint |
| メトリクス記録 | MLflow + BigQuery | Vertex AI Experiments |
| スケーリング | 手動 | 自動（マシンタイプ指定） |

---

## 環境セットアップ

### Python 環境

```bash
pip install google-cloud-aiplatform scikit-learn pandas
```

### GCS バケット作成

```bash
gcloud storage buckets create gs://mlops-dev-a-vertex \
    --location=asia-northeast1 \
    --project=mlops-dev-a
```

### Artifact Registry リポジトリ作成

```bash
gcloud artifacts repositories create vertex-training \
    --repository-format=docker \
    --location=asia-northeast1 \
    --project=mlops-dev-a
```

### gcloud 認証・設定

```bash
gcloud config set project mlops-dev-a
gcloud config set compute/region asia-northeast1
gcloud auth application-default login
```

---

## Step 1: トレーニングスクリプトの作成

Vertex AI Custom Training で実行するスクリプトを作成する。
Vertex AI は環境変数 `AIP_MODEL_DIR` で GCS のモデル出力先を渡してくれる。

### ディレクトリ構成

```
vertex-training/
├── Dockerfile
├── requirements.txt
└── trainer/
    ├── __init__.py
    └── train.py
```

### trainer/train.py

```python
"""Vertex AI Custom Training: カリフォルニア住宅価格予測"""

import os
import pickle
import argparse

import pandas as pd
import numpy as np
from sklearn.datasets import fetch_california_housing
from sklearn.ensemble import RandomForestRegressor
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error, mean_absolute_error
from google.cloud import storage


def parse_args():
    parser = argparse.ArgumentParser()
    parser.add_argument("--n-estimators", type=int, default=100)
    parser.add_argument("--max-depth", type=int, default=10)
    parser.add_argument("--test-size", type=float, default=0.2)
    parser.add_argument("--random-state", type=int, default=42)
    return parser.parse_args()


def load_data(test_size: float, random_state: int):
    """California Housing データセットを取得して分割"""
    housing = fetch_california_housing(as_frame=True)
    X = housing.data  # 8 特徴量: MedInc, HouseAge, AveRooms, AveBedrms, Population, AveOccup, Latitude, Longitude
    y = housing.target  # MedHouseVal（中央住宅価格、単位: $100,000）

    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=test_size, random_state=random_state
    )
    return X_train, X_test, y_train, y_test


def train(X_train, y_train, n_estimators: int, max_depth: int, random_state: int):
    """RandomForestRegressor でトレーニング"""
    model = RandomForestRegressor(
        n_estimators=n_estimators,
        max_depth=max_depth,
        random_state=random_state,
    )
    model.fit(X_train, y_train)
    return model


def evaluate(model, X_test, y_test):
    """RMSE と MAE を算出"""
    y_pred = model.predict(X_test)
    rmse = np.sqrt(mean_squared_error(y_test, y_pred))
    mae = mean_absolute_error(y_test, y_pred)
    return rmse, mae


def save_model(model, model_dir: str):
    """モデルを GCS に保存（Vertex AI の AIP_MODEL_DIR を使用）"""
    local_path = "/tmp/model.pkl"
    with open(local_path, "wb") as f:
        pickle.dump(model, f)

    if model_dir.startswith("gs://"):
        # GCS にアップロード
        parts = model_dir.replace("gs://", "").split("/", 1)
        bucket_name = parts[0]
        prefix = parts[1] if len(parts) > 1 else ""
        blob_path = f"{prefix}/model.pkl" if prefix else "model.pkl"

        client = storage.Client()
        bucket = client.bucket(bucket_name)
        blob = bucket.blob(blob_path)
        blob.upload_from_filename(local_path)
        print(f"モデル保存先: gs://{bucket_name}/{blob_path}")
    else:
        import shutil
        os.makedirs(model_dir, exist_ok=True)
        shutil.copy(local_path, os.path.join(model_dir, "model.pkl"))
        print(f"モデル保存先: {model_dir}/model.pkl")


def main():
    args = parse_args()

    # Vertex AI が設定する環境変数（ローカル実行時のフォールバック付き）
    model_dir = os.environ.get("AIP_MODEL_DIR", "/tmp/model_output")

    print(f"=== カリフォルニア住宅価格予測トレーニング ===")
    print(f"n_estimators={args.n_estimators}, max_depth={args.max_depth}")

    # データ取得
    X_train, X_test, y_train, y_test = load_data(args.test_size, args.random_state)
    print(f"データ件数: 学習={len(X_train)}, テスト={len(X_test)}")

    # トレーニング
    model = train(X_train, y_train, args.n_estimators, args.max_depth, args.random_state)

    # 評価
    rmse, mae = evaluate(model, X_test, y_test)
    print(f"RMSE: {rmse:.4f}")
    print(f"MAE:  {mae:.4f}")

    # モデル保存
    save_model(model, model_dir)
    print("=== トレーニング完了 ===")


if __name__ == "__main__":
    main()
```

**ポイント:**
- `AIP_MODEL_DIR`: Vertex AI が自動的に設定する GCS パス。ここにモデルを保存すると Model Registry 登録時に参照される
- `argparse` でハイパーパラメータを受け取り、Vertex AI の引数渡しに対応
- ローカルでも `python trainer/train.py --n-estimators 50` で動作確認可能

### requirements.txt

```
scikit-learn==1.5.2
pandas==2.2.3
numpy==2.1.3
google-cloud-storage==2.19.0
```

---

## Step 2: コンテナイメージのビルド

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY trainer/ trainer/

ENTRYPOINT ["python", "-m", "trainer.train"]
```

### ビルド＆プッシュ

```bash
IMAGE_URI=asia-northeast1-docker.pkg.dev/mlops-dev-a/vertex-training/california-housing:v1

# Cloud Build でビルド＆プッシュ（ローカル Docker 不要）
gcloud builds submit --tag ${IMAGE_URI} --project=mlops-dev-a

# または Docker を使う場合
docker build -t ${IMAGE_URI} .
docker push ${IMAGE_URI}
```

### ローカルテスト（任意）

```bash
docker run --rm ${IMAGE_URI} --n-estimators 50 --max-depth 5
```

---

## Step 3: Vertex AI Custom Training Job の実行

### Python SDK を使う方法

```python
from google.cloud import aiplatform

aiplatform.init(
    project="mlops-dev-a",
    location="asia-northeast1",
    staging_bucket="gs://mlops-dev-a-vertex",
)

IMAGE_URI = "asia-northeast1-docker.pkg.dev/mlops-dev-a/vertex-training/california-housing:v1"

job = aiplatform.CustomContainerTrainingJob(
    display_name="california-housing-rf",
    container_uri=IMAGE_URI,
    model_serving_container_image_uri="asia-northeast1-docker.pkg.dev/mlops-dev-a/vertex-training/california-housing:v1",
)

model = job.run(
    # ハイパーパラメータを引数で渡す
    args=[
        "--n-estimators", "100",
        "--max-depth", "10",
    ],
    # マシンスペック
    replica_count=1,
    machine_type="n1-standard-4",
    # モデル出力先（AIP_MODEL_DIR に自動設定される）
    model_display_name="california-housing-rf",
    base_output_dir="gs://mlops-dev-a-vertex/models/california-housing",
)

print(f"モデルリソース名: {model.resource_name}")
```

### gcloud CLI を使う方法

```bash
gcloud ai custom-jobs create \
    --region=asia-northeast1 \
    --display-name=california-housing-rf \
    --worker-pool-spec=machine-type=n1-standard-4,replica-count=1,container-image-uri=asia-northeast1-docker.pkg.dev/mlops-dev-a/vertex-training/california-housing:v1 \
    --args="--n-estimators=100,--max-depth=10" \
    --project=mlops-dev-a
```

ジョブの状態を確認:

```bash
gcloud ai custom-jobs list --region=asia-northeast1 --project=mlops-dev-a
```

**マシンタイプの目安:**

| データ規模 | マシンタイプ | vCPU | メモリ |
|-----------|-------------|------|--------|
| 小〜中（California Housing 程度） | n1-standard-4 | 4 | 15 GB |
| 中〜大 | n1-standard-8 | 8 | 30 GB |
| GPU 利用時 | n1-standard-4 + NVIDIA_TESLA_T4 | 4+GPU | 15 GB |

> California Housing は約 20,000 件のため `n1-standard-4` で十分。RandomForest は CPU ベースなので GPU は不要。

---

## Step 4: Model Registry への登録

Step 3 で `CustomContainerTrainingJob.run()` を使った場合、`model_display_name` を指定していれば自動的に Model Registry に登録される。

### 手動で登録する場合

```python
from google.cloud import aiplatform

aiplatform.init(project="mlops-dev-a", location="asia-northeast1")

model = aiplatform.Model.upload(
    display_name="california-housing-rf",
    artifact_uri="gs://mlops-dev-a-vertex/models/california-housing/",
    serving_container_image_uri="us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-3:latest",
)

print(f"Model Registry 登録完了: {model.resource_name}")
```

**`serving_container_image_uri` について:**

Vertex AI は scikit-learn 用の事前構築済みサービングコンテナを提供している:

| フレームワーク | コンテナ URI |
|---------------|-------------|
| scikit-learn 1.3 | `us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-3:latest` |
| scikit-learn 1.0 | `us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-0:latest` |
| XGBoost 1.7 | `us-docker.pkg.dev/vertex-ai/prediction/xgboost-cpu.1-7:latest` |

> 事前構築済みコンテナを使うと、自前でサービングコードを書く必要がない。`model.pkl` を GCS に置くだけで推論 API が立ち上がる。

### コンソールで確認

Vertex AI → Model Registry で登録されたモデルの一覧、バージョン、メタデータを確認できる。

---

## Step 5: エンドポイントへのデプロイ

### Python SDK

```python
from google.cloud import aiplatform

aiplatform.init(project="mlops-dev-a", location="asia-northeast1")

# Model Registry から取得
model = aiplatform.Model(model_name="california-housing-rf")

# エンドポイント作成 & デプロイ
endpoint = model.deploy(
    deployed_model_display_name="california-housing-rf-v1",
    machine_type="n1-standard-2",
    min_replica_count=1,
    max_replica_count=3,  # オートスケーリング
    traffic_percentage=100,
)

print(f"エンドポイント: {endpoint.resource_name}")
```

### gcloud CLI

```bash
# エンドポイント作成
gcloud ai endpoints create \
    --display-name=california-housing-endpoint \
    --region=asia-northeast1 \
    --project=mlops-dev-a

# モデル ID を取得
MODEL_ID=$(gcloud ai models list \
    --region=asia-northeast1 \
    --filter="displayName=california-housing-rf" \
    --format="value(name)" \
    --project=mlops-dev-a)

# エンドポイント ID を取得
ENDPOINT_ID=$(gcloud ai endpoints list \
    --region=asia-northeast1 \
    --filter="displayName=california-housing-endpoint" \
    --format="value(name)" \
    --project=mlops-dev-a)

# デプロイ
gcloud ai endpoints deploy-model ${ENDPOINT_ID} \
    --model=${MODEL_ID} \
    --display-name=california-housing-rf-v1 \
    --machine-type=n1-standard-2 \
    --min-replica-count=1 \
    --max-replica-count=3 \
    --region=asia-northeast1 \
    --project=mlops-dev-a
```

> デプロイには 5〜10 分程度かかる。`--min-replica-count=1` にするとエンドポイントは常時稼働するため課金が発生する点に注意。

---

## Step 6: オンライン推論

### Python SDK

```python
from google.cloud import aiplatform

aiplatform.init(project="mlops-dev-a", location="asia-northeast1")

endpoint = aiplatform.Endpoint(endpoint_name="california-housing-endpoint")

# California Housing の 8 特徴量:
# MedInc, HouseAge, AveRooms, AveBedrms, Population, AveOccup, Latitude, Longitude
instances = [
    [8.3252, 41.0, 6.984, 1.024, 322.0, 2.556, 37.88, -122.23],  # サンプル 1
    [3.5, 30.0, 5.0, 1.0, 1500.0, 3.0, 34.05, -118.25],          # サンプル 2（LA 付近）
]

predictions = endpoint.predict(instances=instances)
print(f"予測結果: {predictions.predictions}")
# 出力例: [4.526, 1.854]  （単位: $100,000 → $452,600, $185,400）
```

### curl でリクエスト

```bash
ENDPOINT_ID="YOUR_ENDPOINT_ID"
PROJECT_ID="mlops-dev-a"
REGION="asia-northeast1"

curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://${REGION}-aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/${REGION}/endpoints/${ENDPOINT_ID}:predict" \
  -d '{
    "instances": [
      [8.3252, 41.0, 6.984, 1.024, 322.0, 2.556, 37.88, -122.23]
    ]
  }'
```

### 現行 Cloud Run API との比較

```python
# 現行: Cloud Run FastAPI
# POST /predict  {"features": [8.3252, 41.0, ...]}
# → {"prediction": 4.526, "model_path": "gs://..."}

# Vertex AI Endpoint
# endpoint.predict(instances=[[8.3252, 41.0, ...]])
# → predictions.predictions = [4.526]
```

> Vertex AI Endpoint はオートスケーリング、トラフィック分割（A/B テスト）、モデルモニタリングが組み込まれている。

---

## Step 7: バッチ推論

大量データをまとめて推論する場合はバッチ推論が効率的。

### 入力データの準備（JSONL 形式で GCS に配置）

```jsonl
[8.3252, 41.0, 6.984, 1.024, 322.0, 2.556, 37.88, -122.23]
[3.5, 30.0, 5.0, 1.0, 1500.0, 3.0, 34.05, -118.25]
[5.0, 25.0, 6.0, 1.1, 800.0, 2.8, 36.74, -119.77]
```

```bash
gsutil cp input.jsonl gs://mlops-dev-a-vertex/batch-input/input.jsonl
```

### バッチ推論の実行

```python
from google.cloud import aiplatform

aiplatform.init(project="mlops-dev-a", location="asia-northeast1")

model = aiplatform.Model(model_name="california-housing-rf")

batch_prediction_job = model.batch_predict(
    job_display_name="california-housing-batch",
    instances_format="jsonl",
    predictions_format="jsonl",
    gcs_source="gs://mlops-dev-a-vertex/batch-input/input.jsonl",
    gcs_destination_prefix="gs://mlops-dev-a-vertex/batch-output/",
    machine_type="n1-standard-4",
)

batch_prediction_job.wait()
print(f"出力先: {batch_prediction_job.output_info}")
```

> バッチ推論はエンドポイントのデプロイ不要。一時的にリソースを立ち上げて処理後に自動削除されるため、コスト効率が良い。

---

## Step 8: リソースのクリーンアップ

エンドポイントを放置すると課金が継続するため、不要になったら削除する。

```python
from google.cloud import aiplatform

aiplatform.init(project="mlops-dev-a", location="asia-northeast1")

# エンドポイントからモデルをアンデプロイ → エンドポイント削除
endpoint = aiplatform.Endpoint(endpoint_name="california-housing-endpoint")
endpoint.undeploy_all()
endpoint.delete()

# Model Registry からモデル削除
model = aiplatform.Model(model_name="california-housing-rf")
model.delete()

# GCS のモデルアーティファクト削除
# gsutil rm -r gs://mlops-dev-a-vertex/models/california-housing/
```

```bash
# gcloud CLI で確認
gcloud ai endpoints list --region=asia-northeast1 --project=mlops-dev-a
gcloud ai models list --region=asia-northeast1 --project=mlops-dev-a
```

---

## 補足: Vertex AI Pipelines で自動化

トレーニング→評価→Model Registry 登録→デプロイを Vertex AI Pipelines で自動化できる。

### パイプライン定義

```python
from kfp import dsl
from kfp import compiler
from google_cloud_pipeline_components.v1.custom_job import CustomTrainingJobOp
from google_cloud_pipeline_components.v1.model import ModelUploadOp
from google_cloud_pipeline_components.v1.endpoint import (
    EndpointCreateOp,
    ModelDeployOp,
)


@dsl.pipeline(
    name="california-housing-pipeline",
    pipeline_root="gs://mlops-dev-a-vertex/pipelines/",
)
def pipeline(
    project: str = "mlops-dev-a",
    location: str = "asia-northeast1",
    n_estimators: int = 100,
    max_depth: int = 10,
):
    # Step 1: Custom Training
    training_job = CustomTrainingJobOp(
        display_name="california-housing-training",
        project=project,
        location=location,
        worker_pool_specs=[{
            "machine_spec": {"machine_type": "n1-standard-4"},
            "replica_count": 1,
            "container_spec": {
                "image_uri": "asia-northeast1-docker.pkg.dev/mlops-dev-a/vertex-training/california-housing:v1",
                "args": [
                    f"--n-estimators={n_estimators}",
                    f"--max-depth={max_depth}",
                ],
            },
        }],
        base_output_directory="gs://mlops-dev-a-vertex/models/california-housing/",
    )

    # Step 2: Model Registry 登録
    model_upload = ModelUploadOp(
        project=project,
        location=location,
        display_name="california-housing-rf",
        artifact_uri="gs://mlops-dev-a-vertex/models/california-housing/model/",
        serving_container_image_uri="us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-3:latest",
    ).after(training_job)

    # Step 3: エンドポイント作成
    endpoint_create = EndpointCreateOp(
        project=project,
        location=location,
        display_name="california-housing-endpoint",
    )

    # Step 4: デプロイ
    ModelDeployOp(
        project=project,
        location=location,
        endpoint=endpoint_create.outputs["endpoint"],
        model=model_upload.outputs["model"],
        deployed_model_display_name="california-housing-rf-v1",
        dedicated_resources_machine_type="n1-standard-2",
        dedicated_resources_min_replica_count=1,
        dedicated_resources_max_replica_count=3,
    )


# コンパイル
compiler.Compiler().compile(pipeline, "california_housing_pipeline.yaml")
```

### パイプラインの実行

```python
from google.cloud import aiplatform

aiplatform.init(project="mlops-dev-a", location="asia-northeast1")

job = aiplatform.PipelineJob(
    display_name="california-housing-pipeline-run",
    template_path="california_housing_pipeline.yaml",
    pipeline_root="gs://mlops-dev-a-vertex/pipelines/",
    parameter_values={
        "n_estimators": 100,
        "max_depth": 10,
    },
)

job.run()
```

### パイプラインの可視化

Vertex AI コンソール → パイプライン で DAG と各ステップの実行状況・ログを確認できる。

```
[Training] → [Model Upload] → [Deploy]
                                  ↑
                          [Endpoint Create]
```

---

## まとめ

| ステップ | やること | 主要リソース |
|---------|---------|-------------|
| トレーニング | scikit-learn で学習、GCS にモデル保存 | Custom Training Job |
| モデル管理 | GCS のモデルを Registry に登録 | Model Registry |
| サービング | エンドポイントにデプロイ | Endpoint |
| オンライン推論 | REST API で 1 件〜数件を推論 | Endpoint.predict() |
| バッチ推論 | JSONL で大量データを一括推論 | Batch Prediction Job |
| 自動化 | パイプラインでワークフロー定義 | Vertex AI Pipelines (KFP v2) |

### 現行プロジェクトからの移行ポイント

1. **トレーニングスクリプト**: 既存の `src/batch/` のコードをほぼそのまま流用可能。`AIP_MODEL_DIR` への保存を追加するだけ
2. **モデル管理**: BigQuery metrics テーブルの代わりに Model Registry を使用。バージョン管理・メタデータ管理が組み込み
3. **推論 API**: FastAPI を自前で書く代わりに、事前構築済みサービングコンテナが推論 API を自動提供
4. **モニタリング**: Vertex AI Model Monitoring でデータドリフト・予測ドリフトを自動検知可能
