# Vertex RAG MLOps（Terraform込み案件レベル構成）

---

## 0. 目的

本構成は、**GCP上で実務レベルのRAG（Retrieval-Augmented Generation）をMLOpsとして成立させる最小構成**を定義する。

* シンプル（過剰設計排除）
* Terraformで再現可能
* バッチ主体（Cloud Run Job）

---

## 1. 全体アーキテクチャ

```
[データ投入]
  ↓
GCS / 外部データ
  ↓
Cloud Run Job（前処理・Embedding生成）
  ↓
BigQuery（Vector Store）
  ↓
Cloud Run Service（API）
  ↓
Vertex AI（Gemini）
  ↓
レスポンス
```

---

## 2. コンポーネント構成

### 2.1 データ層

* GCS：原データ格納
* BigQuery：

  * embedding格納
  * メタデータ

### 2.2 ML層（Vertex AI）

* Embedding API
* Gemini（生成）

### 2.3 アプリ層

* Cloud Run Job：

  * 前処理
  * embedding生成
  * BigQuery登録

* Cloud Run Service：

  * クエリ受付
  * 類似検索
  * プロンプト生成
  * LLM呼び出し

### 2.4 検索層

* BigQuery Vector Search

---

## 3. データモデル（BigQuery）

```
table: documents

- id STRING
- content STRING
- embedding ARRAY<FLOAT64>
- created_at TIMESTAMP
```

インデックス：

* VECTOR INDEX（embedding）

---

## 4. 処理フロー

### 4.1 バッチ（インデックス生成）

1. GCSからデータ取得
2. テキスト分割（chunking）
3. Vertex Embedding生成
4. BigQueryに格納

---

### 4.2 リクエスト処理（RAG）

1. ユーザークエリ受信
2. Embedding生成
3. BigQueryで類似検索
4. 上位K件取得
5. プロンプト生成
6. Geminiで回答生成

---

## 5. Terraform構成

### 5.1 リソース一覧

* google_project
* google_storage_bucket
* google_bigquery_dataset
* google_bigquery_table
* google_cloud_run_service
* google_cloud_run_v2_job
* google_service_account
* google_project_iam_binding

---

### 5.2 ディレクトリ構成

```
terraform/
 ├── main.tf
 ├── variables.tf
 ├── outputs.tf
 ├── cloud_run.tf
 ├── bigquery.tf
 ├── gcs.tf
 └── iam.tf
```

---

### 5.3 例：BigQuery

```hcl
resource "google_bigquery_dataset" "rag" {
  dataset_id = "rag_dataset"
  location   = "asia-northeast1"
}

resource "google_bigquery_table" "documents" {
  dataset_id = google_bigquery_dataset.rag.dataset_id
  table_id   = "documents"

  schema = jsonencode([
    { name = "id", type = "STRING" },
    { name = "content", type = "STRING" },
    { name = "embedding", type = "FLOAT64", mode = "REPEATED" }
  ])
}
```

---

### 5.4 例：Cloud Run Job

```hcl
resource "google_cloud_run_v2_job" "batch" {
  name     = "rag-batch-job"
  location = "asia-northeast1"

  template {
    template {
      containers {
        image = "gcr.io/PROJECT_ID/rag-batch:latest"
      }
    }
  }
}
```

---

### 5.5 例：Cloud Run Service

```hcl
resource "google_cloud_run_service" "api" {
  name     = "rag-api"
  location = "asia-northeast1"

  template {
    spec {
      containers {
        image = "gcr.io/PROJECT_ID/rag-api:latest"
      }
    }
  }
}
```

---

## 6. アプリ実装（Python例）

### 6.1 Embedding生成

```python
from vertexai.language_models import TextEmbeddingModel

model = TextEmbeddingModel.from_pretrained("textembedding-gecko")

emb = model.get_embeddings(["sample text"])
```

---

### 6.2 類似検索（BigQuery）

```sql
SELECT *
FROM documents
ORDER BY VECTOR_DISTANCE(embedding, @query_embedding)
LIMIT 5
```

---

### 6.3 LLM呼び出し

```python
from vertexai.generative_models import GenerativeModel

model = GenerativeModel("gemini-pro")

response = model.generate_content(prompt)
```

---

## 7. CI/CD（最低限）

* Cloud Build or GitHub Actions
* Docker build → Artifact Registry
* Cloud Run deploy

---

## 8. 運用設計（最低限）

* Logging：Cloud Logging
* 失敗検知：Error Reporting
* コスト監視：Billing Alert

---

## 9. 拡張ポイント

* リランキング（Cross Encoder）
* キャッシュ（Redis）
* ストリーミング応答
* Feature Store連携

---

## 10. 本質まとめ

* RAGの核心は検索設計
* Vertexは生成エンジン
* BigQueryは現実解のVector DB
* Cloud Runでつなぐ

👉 「1本のパイプラインとして成立させること」が最重要

---

以上。
