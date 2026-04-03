# Vertex AI 最小構成チュートリアル: カリフォルニア住宅価格推論

Notebook 1 つで Vertex AI の流れを体験する。パイプライン・コンテナビルド不要。

**実装ファイル:** `notebooks/vertex_housing_mvp.ipynb`

---

## 前提

- GCP プロジェクト: `mlops-dev-a`、リージョン: `asia-northeast1`
- Vertex AI API が有効化済み
- Python 3.10+

```bash
gcloud services enable aiplatform.googleapis.com --project=mlops-dev-a
pip install google-cloud-aiplatform scikit-learn pandas
```

---

## 全体像（3 ステップ）

```
① ローカルで学習 → GCS にモデル保存
② Model Registry に登録
③ エンドポイントにデプロイ → 推論
```

所要時間: 約 15〜20 分（デプロイ待ち含む）

---

## ① 学習 & GCS にモデル保存

```python
import pickle
import numpy as np
from sklearn.datasets import fetch_california_housing
from sklearn.ensemble import RandomForestRegressor
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error, mean_absolute_error
from google.cloud import storage

# --- データ取得 ---
housing = fetch_california_housing(as_frame=True)
X_train, X_test, y_train, y_test = train_test_split(
    housing.data, housing.target, test_size=0.2, random_state=42
)

# --- 学習 ---
model = RandomForestRegressor(n_estimators=100, max_depth=10, random_state=42)
model.fit(X_train, y_train)

# --- 評価 ---
y_pred = model.predict(X_test)
print(f"RMSE: {np.sqrt(mean_squared_error(y_test, y_pred)):.4f}")
print(f"MAE:  {mean_absolute_error(y_test, y_pred):.4f}")

# --- GCS に保存 ---
BUCKET = "mlops-dev-a-vertex"
MODEL_PATH = "models/california-housing-mvp/model.pkl"

with open("/tmp/model.pkl", "wb") as f:
    pickle.dump(model, f)

client = storage.Client(project="mlops-dev-a")
bucket = client.bucket(BUCKET)
bucket.blob(MODEL_PATH).upload_from_filename("/tmp/model.pkl")
print(f"保存完了: gs://{BUCKET}/{MODEL_PATH}")
```

> GCS バケットが未作成の場合: `gcloud storage buckets create gs://mlops-dev-a-vertex --location=asia-northeast1`

---

## ② Model Registry に登録

```python
from google.cloud import aiplatform

aiplatform.init(project="mlops-dev-a", location="asia-northeast1")

model = aiplatform.Model.upload(
    display_name="california-housing-rf-mvp",
    artifact_uri=f"gs://{BUCKET}/models/california-housing-mvp/",
    # Vertex AI 提供の sklearn サービングコンテナ（自前コンテナ不要）
    serving_container_image_uri="us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-3:latest",
)

print(f"登録完了: {model.resource_name}")
```

**ポイント:** `serving_container_image_uri` に Vertex AI 事前構築済みコンテナを指定するだけで、`model.pkl` を自動で読み込んで推論 API を提供してくれる。FastAPI 等のサービングコードは不要。

---

## ③ デプロイ & 推論

### デプロイ（5〜10 分待ち）

```python
endpoint = model.deploy(
    deployed_model_display_name="california-housing-rf-mvp-v1",
    machine_type="n1-standard-2",
    min_replica_count=1,
    max_replica_count=1,
)

print(f"エンドポイント: {endpoint.resource_name}")
```

### 推論

```python
# 8 特徴量: MedInc, HouseAge, AveRooms, AveBedrms, Population, AveOccup, Latitude, Longitude
result = endpoint.predict(instances=[
    [8.3252, 41.0, 6.984, 1.024, 322.0, 2.556, 37.88, -122.23],
])

print(f"予測価格: ${result.predictions[0] * 100_000:,.0f}")
# 例: $452,600
```

### テストデータで検証

```python
# テストデータの先頭 3 件で推論
instances = X_test.head(3).values.tolist()
result = endpoint.predict(instances=instances)

for i, (pred, actual) in enumerate(zip(result.predictions, y_test.head(3))):
    print(f"サンプル{i+1}: 予測=${pred * 100_000:,.0f}  実際=${actual * 100_000:,.0f}")
```

---

## クリーンアップ（重要）

エンドポイントは常時課金されるため、確認が終わったら必ず削除する。

```python
endpoint.undeploy_all()
endpoint.delete()
model.delete()
print("クリーンアップ完了")
```

```bash
# GCS のモデルも削除する場合
gcloud storage rm -r gs://mlops-dev-a-vertex/models/california-housing-mvp/
```

---

## 現行プロジェクトとの対応

| 現行（Cloud Run ベース） | この MVP | 備考 |
|-------------------------|----------|------|
| `src/batch/train.py` で学習 | ① のローカル学習 | 同じ sklearn コード |
| GCS に pickle 保存 | ① の GCS 保存 | 同じ |
| BigQuery で最良モデル管理 | ② Model Registry | バージョン管理が組み込み |
| `src/api/main.py` FastAPI | ③ Vertex AI Endpoint | サービングコード不要 |

---

## 次のステップ

この MVP で流れを掴んだら、段階的に拡張できる:

1. **Custom Training Job** — 学習自体も Vertex AI 上で実行（GPU・大規模データ対応）
2. **Vertex AI Experiments** — ハイパーパラメータ・メトリクスの追跡
3. **Vertex AI Pipelines** — 学習→登録→デプロイの自動化
4. **Model Monitoring** — データドリフト・予測ドリフトの自動検知

詳細は [Vertex入門 住宅価格推論.md](Vertex入門%20住宅価格推論.md) を参照。
