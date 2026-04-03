# Terraform化チートシート
## Elastic Cloud + GCP Cloud Run Job

---

## 概念マップ (今回の構成)

```
今回 手動でやったこと          → Terraform resourceに対応
──────────────────────────────────────────────────────
Elastic Cloud デプロイメント作成 → ec_deployment
Artifact Registry リポジトリ作成 → google_artifact_registry_repository
Secret Manager シークレット作成  → google_secret_manager_secret
                                   google_secret_manager_secret_version
Cloud Run Job 作成               → google_cloud_run_v2_job
IAM 権限付与                     → google_secret_manager_secret_iam_member
```

---

## ディレクトリ構成

```
terraform/
├── main.tf          # リソース定義
├── variables.tf     # 変数定義
├── outputs.tf       # 出力値
├── providers.tf     # provider設定
└── terraform.tfvars # 変数の実値 (gitignore必須)
```

---

## providers.tf

```hcl
terraform {
  required_providers {
    ec = {
      source  = "elastic/ec"
      version = "~> 0.10"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "ec" {
  apikey = var.elastic_cloud_api_key
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

---

## variables.tf

```hcl
variable "project_id" {}
variable "region" { default = "asia-northeast1" }
variable "elastic_cloud_api_key" { sensitive = true }  # Elastic CloudのOrg APIキー
variable "elastic_api_key"       { sensitive = true }  # ESのAPIキー (既存流用可)
```

---

## main.tf

```hcl
# Elastic Cloud デプロイメント
resource "ec_deployment" "hello" {
  name                   = "my-deployment"
  region                 = "gcp-asia-northeast1"
  version                = "9.3.2"
  deployment_template_id = "gcp-storage-optimized"

  elasticsearch = {
    hot = {
      autoscaling = {}
      size        = "1g"
      zone_count  = 1
    }
  }

  kibana = {
    zone_count = 1
  }
}

# Artifact Registry
resource "google_artifact_registry_repository" "hello" {
  repository_id = "hello-elastic"
  location      = var.region
  format        = "DOCKER"
}

# Secret Manager
resource "google_secret_manager_secret" "elastic_api_key" {
  secret_id = "elastic-api-key"
  replication {
    user_managed {
      replicas { location = var.region }
    }
  }
}

resource "google_secret_manager_secret_version" "elastic_api_key" {
  secret      = google_secret_manager_secret.elastic_api_key.id
  secret_data = var.elastic_api_key
}

# IAM: Cloud Run → Secret Manager
resource "google_secret_manager_secret_iam_member" "hello" {
  secret_id = google_secret_manager_secret.elastic_api_key.id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${data.google_project.project.number}-compute@developer.gserviceaccount.com"
}

data "google_project" "project" {}

# Cloud Run Job
resource "google_cloud_run_v2_job" "hello" {
  name     = "es-hello"
  location = var.region

  template {
    template {
      containers {
        image = "${var.region}-docker.pkg.dev/${var.project_id}/hello-elastic/es-hello:latest"

        env {
          name  = "ELASTIC_CLOUD_URL"
          value = ec_deployment.hello.elasticsearch[0].https_endpoint
        }
        env {
          name = "ELASTIC_API_KEY"
          value_source {
            secret_key_ref {
              secret  = google_secret_manager_secret.elastic_api_key.secret_id
              version = "latest"
            }
          }
        }
      }
    }
  }

  depends_on = [google_secret_manager_secret_iam_member.hello]
}
```

---

## outputs.tf

```hcl
output "elasticsearch_endpoint" {
  value = ec_deployment.hello.elasticsearch[0].https_endpoint
}

output "cloud_run_job_name" {
  value = google_cloud_run_v2_job.hello.name
}

output "artifact_registry_url" {
  value = "${var.region}-docker.pkg.dev/${var.project_id}/hello-elastic"
}
```

---

## terraform.tfvars (gitignore必須)

```hcl
project_id            = "mlops-dev-a"
region                = "asia-northeast1"
elastic_cloud_api_key = "YOUR_ELASTIC_CLOUD_ORG_API_KEY"  # ※後述
elastic_api_key       = "WXU5T1Va..."
```

---

## Elastic Cloud Org APIキーの発行 (providers.tf用)

```
https://cloud.elastic.co/
  → 左メニュー: Organization
  → API keys
  → Create API key (Org level)
```

> 今回発行した `hello-elastic-job` キーはES接続用。  
> Terraform provider用には **Org レベル** の別キーが必要。

---

## 実行コマンド

```bash
cd terraform/

# 初期化 (provider download)
terraform init

# 差分確認
terraform plan -var-file="terraform.tfvars"

# 適用
terraform apply -var-file="terraform.tfvars"

# 削除 (課金停止)
terraform destroy -var-file="terraform.tfvars"
```

---

## .gitignore

```
terraform/.terraform/
terraform/terraform.tfvars
terraform/.terraform.lock.hcl
terraform/terraform.tfstate
terraform/terraform.tfstate.backup
```

---

## 作成・削除フロー

```
作業開始: terraform apply   → Elastic Cloud + Cloud Run Job が5分で再現
作業終了: terraform destroy → 全リソース削除 → 課金ゼロ
```