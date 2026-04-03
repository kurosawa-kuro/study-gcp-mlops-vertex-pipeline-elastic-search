# Cloud Run Job デプロイ手順書
## Elasticsearch Hello World (Python → GCR → Cloud Run Job)

---

## 前提条件

| 項目 | 状態 |
|---|---|
| GCP ログイン済み | `gcloud auth list` で確認 |
| プロジェクト設定済み | `mlops-dev-a` |
| Elastic Cloud 起動済み | GCP Tokyo (asia-northeast1) |
| API Key 取得済み | `ELASTIC_API_KEY` |
| Docker インストール済み | `docker --version` で確認 |
| ソースコード準備済み | `src/elastic-search/` に3ファイル |

---

## ステップ1: 環境変数を設定

```bash
export PROJECT_ID=mlops-dev-a
export REGION=asia-northeast1
export IMAGE_NAME=es-hello
export REPO_NAME=hello-elastic
export ELASTIC_CLOUD_URL="https://e2ca746771d94d2781a841170364167c.asia-northeast1.gcp.cloud.es.io:443"
export ELASTIC_API_KEY="YOUR_API_KEY"  # 実際のキーに置き換える
```

> **なぜ環境変数で管理するか**: APIキーをコードにハードコードするとGitリポジトリに漏洩するリスクがある。環境変数として外部から注入することでコードとシークレットを分離する。

---

## ステップ2: Artifact Registry リポジトリを作成

```bash
gcloud artifacts repositories create ${REPO_NAME} \
  --repository-format=docker \
  --location=${REGION} \
  --description="Hello Elastic Docker images"
```

> **Artifact Registry とは**: GCPのDockerイメージ保管場所。旧来のContainer Registry (GCR) の後継。Cloud Runはここからイメージを取得して実行する。

確認:
```bash
gcloud artifacts repositories list --location=${REGION}
```

---

## ステップ3: Docker認証を設定

```bash
gcloud auth configure-docker ${REGION}-docker.pkg.dev
```

> **なぜ必要か**: DockerクライアントがArtifact Registryにpushするためにはgcloudの認証情報を使う必要がある。このコマンドで`~/.docker/config.json`にcredential helperが登録される。

---

## ステップ4: Dockerイメージをビルド

```bash
cd ~/repos/study-gcp-mlops-vertex-elastic-search/src/elastic-search

docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:latest .
```

> **イメージ名の構造**:
> ```
> asia-northeast1-docker.pkg.dev / mlops-dev-a / hello-elastic / es-hello:latest
> └─ registry host ─────────────┘ └─ project ─┘ └─ repo ──────┘ └─ image:tag ─┘
> ```

確認:
```bash
docker images | grep es-hello
```

---

## ステップ5: イメージを Artifact Registry に push

```bash
docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:latest
```

> **pushとは**: ローカルで作ったDockerイメージをGCPのレジストリにアップロードする操作。Cloud RunはこのURLを参照してコンテナを起動する。

確認:
```bash
gcloud artifacts docker images list \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}
```

---

## ステップ6: Secret Manager にAPIキーを登録

```bash
echo -n "${ELASTIC_API_KEY}" | \
  gcloud secrets create elastic-api-key \
  --data-file=- \
  --replication-policy=user-managed \
  --locations=${REGION}
```

> **なぜSecret Managerを使うか**: Cloud Runの環境変数に直接APIキーを書くと、GCPコンソールで誰でも見えてしまう。Secret Managerに格納することで、アクセス制御・ローテーション・監査ログが利用できる。

確認:
```bash
gcloud secrets list
```

---

## ステップ7: Cloud Run Job を作成

```bash
gcloud run jobs create ${IMAGE_NAME} \
  --image=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:latest \
  --region=${REGION} \
  --set-env-vars="ELASTIC_CLOUD_URL=${ELASTIC_CLOUD_URL}" \
  --set-secrets="ELASTIC_API_KEY=elastic-api-key:latest" \
  --task-timeout=60s \
  --max-retries=0
```

> **Cloud Run Job とは**: HTTPリクエストを待たずに1回実行して終了するコンテナ。APIサーバーとして常時起動するCloud Run Serviceと異なり、バッチ処理に適している。今回のようなデータ投入・検索確認に最適。

> **オプション解説**:
> - `--set-secrets`: Secret ManagerのシークレットをコンテナのEnvironment Variableとしてマウント
> - `--task-timeout`: タスクの最大実行時間
> - `--max-retries=0`: 失敗時に再試行しない（デバッグ時は0が見やすい）

確認:
```bash
gcloud run jobs list --region=${REGION}
```

---

## ステップ8: Cloud Run Job を実行

```bash
gcloud run jobs execute ${IMAGE_NAME} \
  --region=${REGION} \
  --wait
```

> **`--wait` オプション**: 実行完了までコマンドがブロックされる。完了後に成功/失敗が表示される。

---

## ステップ9: ログを確認

```bash
gcloud logging read \
  "resource.type=cloud_run_job AND resource.labels.job_name=${IMAGE_NAME}" \
  --limit=50 \
  --format="value(textPayload)" \
  --project=${PROJECT_ID}
```

期待される出力:
```
=== 1. info ===
cluster_name: e2ca746771d94d2781a841170364167c
version: 9.3.2

=== 2. index document ===
result: created
_id: xxxxxxxxxxxxxxxx

=== 3. search ===
total hits: 1
  -> {'message': 'hello world', 'tag': 'test'}

=== 4. cleanup ===
index deleted
```

> **Cloud Loggingとは**: GCPのログ集約サービス。Cloud Runのstdout/stderrが自動的に転送される。`gcloud logging read`でCLIから検索できる。

---

## トラブルシューティング

### permission denied (Artifact Registry push)
```bash
# サービスアカウントに権限付与
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/artifactregistry.writer"
```

### Secret Manager アクセスエラー
```bash
# Cloud Run のサービスアカウントに Secret アクセス権を付与
PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)")

gcloud secrets add-iam-policy-binding elastic-api-key \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

### Cloud Run APIs が未有効
```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  secretmanager.googleapis.com
```

---

## クリーンアップ (作業終了後)

```bash
# Cloud Run Job 削除
gcloud run jobs delete ${IMAGE_NAME} --region=${REGION} --quiet

# Artifact Registry イメージ削除
gcloud artifacts docker images delete \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:latest --quiet

# Secret 削除
gcloud secrets delete elastic-api-key --quiet

# Elastic Cloud デプロイメント削除 (課金停止)
# → Elastic Cloud Console > My deployment > Actions > Delete deployment
```

---

## 全体フロー図

```
[WSL]                    [GCP]
  │
  ├─ docker build
  │
  ├─ docker push ──────► Artifact Registry
  │                           │
  ├─ gcloud run jobs ────────►│
  │   create                  │
  │                           ▼
  ├─ gcloud run jobs     Cloud Run Job
  │   execute ──────────► (コンテナ起動)
  │                           │
  │                           ▼
  │                      Elastic Cloud
  │                      (ES接続・検索)
  │                           │
  └─ gcloud logging ◄─────────┘
      read (ログ確認)
```