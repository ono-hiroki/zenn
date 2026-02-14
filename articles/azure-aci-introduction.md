---
title: "Azure Container Instances をやってみる - 基本・環境変数・ボリューム"
emoji: "🫐"
type: "tech"
topics: ["azure", "terraform", "docker", "aws"]
published: true
---

AWSは使ったことがあるけどAzureは初めて、という状況で Azure Container Instances（ACI）を試してみました。AWSとの対比を交えながら書いていきます。

## Azure Container Instances とは

ACIは、サーバーレスでコンテナを実行できるサービスです。VMやクラスターの管理不要で、コンテナイメージを直接実行できます。

| 特徴 | 説明 |
|------|------|
| サーバーレス | インフラ管理不要 |
| 高速起動 | 数秒でコンテナ起動 |
| 従量課金 | vCPU/メモリ × 秒単位 |
| パブリックIP | 直接割り当て可能 |

### AWS との比較

- ACI ≒ **ECS on Fargate**（タスク単発実行）
- ECR不要 → Docker Hub等のパブリックイメージをそのまま使用可能

## 前提条件

- Azure CLI インストール済み
- `az login` で認証済み
- Terraform インストール済み

## コード

完全なコードはGitHubに置いています。

https://github.com/ono-hiroki/maitake/tree/main/azure-aci

```bash
git clone https://github.com/ono-hiroki/maitake.git
cd maitake/azure-aci
```

## 1. 基本的なコンテナ起動

まずは nginx コンテナを起動してみます。

### 構成図

```
┌─────────────────────────────────────┐
│         Resource Group              │
│  ┌───────────────────────────────┐  │
│  │   Azure Container Instance    │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │   nginx container       │  │  │
│  │  │   (Docker Hub image)    │  │  │
│  │  └─────────────────────────┘  │  │
│  │         :80                   │  │
│  └───────────────────────────────┘  │
│              │                      │
│        Public IP                    │
└──────────────┼──────────────────────┘
               │
          Internet
```

### Terraform コード

```hcl
# aci.tf
resource "azurerm_container_group" "nginx" {
  name                = "${var.prefix}-nginx"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  os_type         = "Linux"
  ip_address_type = "Public"
  dns_name_label  = "${var.prefix}-nginx"

  container {
    name   = "nginx"
    image  = "nginx:latest"
    cpu    = "0.5"
    memory = "0.5"

    ports {
      port     = 80
      protocol = "TCP"
    }
  }

  tags = var.tags
}
```

`dns_name_label` を設定すると `<label>.<region>.azurecontainer.io` でアクセスできます。

### AWSとの比較

| パラメータ | Azure ACI | AWS ECS (Fargate) |
|-----------|-----------|-------------------|
| コンテナグループ | `azurerm_container_group` | Task Definition |
| OS | `os_type = "Linux"` | `runtimePlatform` |
| CPU/メモリ | `cpu = "0.5"` / `memory = "0.5"` | `cpu = 256` / `memory = 512` |
| パブリックIP | `ip_address_type = "Public"` | `assignPublicIp = "ENABLED"` |

### 実行

```bash
cd 01-basic
terraform init
terraform apply
```

適用完了後、出力されるURLにブラウザでアクセスすると nginx のウェルカムページが表示されます。

### コンテナ操作

AWS ECS の `execute-command` に相当する操作です。

```bash
# コンテナ内に入る
az container exec \
  --resource-group aci-demo-rg \
  --name aci-demo-nginx \
  --container-name nginx \
  --exec-command "/bin/sh"

# ログを見る（docker logs相当）
az container logs \
  --resource-group aci-demo-rg \
  --name aci-demo-nginx

# リアルタイムでログを追う（-f相当）
az container attach \
  --resource-group aci-demo-rg \
  --name aci-demo-nginx
```

ECSと違って、ACIはexecを使うための事前設定（SSM Agentなど）が不要なのが楽です。

## 2. 環境変数の設定

次に、コンテナに環境変数を渡してみます。

### Terraform コード

```hcl
container {
  name   = "app"
  image  = "nginx:latest"
  cpu    = "0.5"
  memory = "0.5"

  ports {
    port     = 80
    protocol = "TCP"
  }

  # 通常の環境変数
  environment_variables = {
    APP_ENV     = "development"
    APP_DEBUG   = "true"
    APP_NAME    = "ACI Demo App"
    SERVER_PORT = "80"
  }

  # 機密情報用の環境変数（terraform plan/applyの出力に表示されない）
  secure_environment_variables = {
    DB_PASSWORD = var.db_password
    API_KEY     = var.api_key
  }
}
```

### AWS との比較

| Azure ACI | AWS ECS |
|-----------|---------|
| `environment_variables` | `environment` |
| `secure_environment_variables` | Secrets Manager / Parameter Store 連携 |

### 実行

```bash
cd 02-env-vars
terraform init
terraform apply
```

### 環境変数の確認

```bash
az container exec \
  --resource-group aci-env-rg \
  --name aci-env-app \
  --container-name app \
  --exec-command "/bin/sh"

# コンテナ内で
env | grep -E "APP_|DB_|API_"
```

### 注意: tfstate への機密情報の残存

`secure_environment_variables` を使っても、**tfstate には値が平文で残ります**。

`sensitive = true` は Terraform の出力（plan/apply）を隠すだけで、tfstate には保存されてしまいます。

**本番での対策:**
- アプリ側で Managed Identity + Key Vault SDK を使う
- Azure Container Apps（ACA）に移行する（Key Vault 直接参照が可能）

## 3. ボリュームのマウント

Azure File Share をコンテナにマウントして、永続的なストレージを使ってみます。

### 構成図

```
┌─────────────────────────────────────────────┐
│              Resource Group                 │
│  ┌─────────────────┐  ┌─────────────────┐  │
│  │ Storage Account │  │ Container Group │  │
│  │  ┌───────────┐  │  │  ┌───────────┐  │  │
│  │  │File Share │◄─┼──┼─►│  nginx    │  │  │
│  │  │ (aci-data)│  │  │  │           │  │  │
│  │  └───────────┘  │  │  └───────────┘  │  │
│  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────┘
```

### AWS との比較

| Azure | AWS |
|-------|-----|
| Azure File Share | EFS (Elastic File System) |
| Storage Account | S3 + EFS の組み合わせ |

### Terraform コード

まず Storage Account と File Share を作成します。

```hcl
# storage.tf
resource "azurerm_storage_account" "main" {
  name                     = replace("${var.prefix}storage", "-", "")
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"

  tags = var.tags
}

resource "azurerm_storage_share" "data" {
  name               = "aci-data"
  storage_account_id = azurerm_storage_account.main.id
  quota              = 1 # GB
}
```

コンテナにマウントします。

```hcl
# aci.tf
container {
  name   = "nginx"
  image  = "nginx:latest"
  cpu    = "0.5"
  memory = "0.5"

  ports {
    port     = 80
    protocol = "TCP"
  }

  volume {
    name       = "data-volume"
    mount_path = "/usr/share/nginx/html"
    read_only  = false
    share_name = azurerm_storage_share.data.name

    storage_account_name = azurerm_storage_account.main.name
    storage_account_key  = azurerm_storage_account.main.primary_access_key
  }
}
```

### 実行

```bash
cd 03-volumes
terraform init
terraform apply
```

### 動作確認

```bash
# ストレージアカウント名を取得
STORAGE_ACCOUNT=$(terraform output -raw storage_account_name)

# index.html をアップロード
echo "<h1>Hello from Azure File Share!</h1>" > index.html

az storage file upload \
  --account-name $STORAGE_ACCOUNT \
  --share-name aci-data \
  --source index.html \
  --path index.html

# ブラウザでアクセス
terraform output access_url
```

アップロードした index.html が表示されます。

### ボリュームの種類

ACI でサポートされるボリュームは以下の4種類です。

| 種類 | 用途 |
|------|------|
| Azure File Share | 永続的な共有ストレージ |
| emptyDir | 一時的なストレージ（コンテナ間共有） |
| gitRepo | Git リポジトリのクローン |
| secret | 機密情報のマウント |

## 料金の目安（Japan East）

- vCPU: 約 0.005円/秒
- メモリ: 約 0.0006円/GB/秒

例: 1 vCPU + 1.5GB メモリで1時間 ≒ 約22円

## クリーンアップ

```bash
terraform destroy
```

## まとめ

| 項目 | Azure ACI | AWS ECS on Fargate |
|------|-----------|-------------------|
| コンテナグループ | Container Group | Task Definition |
| 環境変数（通常） | `environment_variables` | `environment` |
| 環境変数（機密） | `secure_environment_variables` | Secrets Manager連携 |
| ボリューム | Azure File Share | EFS |
| exec | `az container exec`（設定不要） | `execute-command`（SSM設定必要） |

AWSに慣れていれば、用語の違いを押さえるだけで同じ感覚で使えます。

## 次の記事

サイドカーパターンとVNet統合についてはこちら。

https://zenn.dev/ono_hiroki/articles/azure-aci-advanced
