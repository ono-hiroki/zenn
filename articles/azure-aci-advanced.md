---
title: "Azure Container Instances をやってみる - サイドカーとVNet統合"
emoji: "🫐"
type: "tech"
topics: ["azure", "terraform", "docker", "aws"]
published: true
---

前回の記事に引き続き、Azure Container Instances（ACI）を試していきます。今回は複数コンテナのサイドカーパターンとVNet統合です。

https://zenn.dev/ono_hiroki/articles/azure-aci-introduction

## コード

完全なコードはGitHubに置いています。

https://github.com/ono-hiroki/maitake/tree/main/azure-aci

```bash
git clone https://github.com/ono-hiroki/maitake.git
cd maitake/azure-aci
```

## 1. 複数コンテナのグループ化（サイドカーパターン）

1つの Container Group に複数のコンテナを配置する「サイドカーパターン」を試してみます。

### サイドカーパターンとは

メインコンテナを補助するコンテナを同じ Pod/Group に配置するパターンです。

| 用途 | 例 |
|------|-----|
| ログ収集 | Fluentd, Filebeat |
| プロキシ | Envoy, nginx |
| 監視 | Prometheus exporter |
| 認証 | OAuth2 Proxy |

### 構成図

```
┌─────────────────────────────────────────────┐
│           Container Group                   │
│  ┌─────────────────┐  ┌─────────────────┐  │
│  │     nginx       │  │   log-reader    │  │
│  │   (メイン)       │  │  (サイドカー)    │  │
│  │                 │  │                 │  │
│  │ /var/log/nginx  │  │    /logs        │  │
│  └────────┬────────┘  └────────┬────────┘  │
│           │                    │           │
│           └────────┬───────────┘           │
│                    │                       │
│            ┌───────▼───────┐               │
│            │   emptyDir    │               │
│            │  (logs volume)│               │
│            └───────────────┘               │
│                    │                       │
│              Public IP:80                  │
└─────────────────────────────────────────────┘
```

### AWS との比較

| Azure ACI | AWS ECS |
|-----------|---------|
| Container Group | Task Definition |
| 複数 container ブロック | 複数 containerDefinitions |
| emptyDir | ボリューム (bind mount) |

### Terraform コード

```hcl
# aci.tf
resource "azurerm_container_group" "app" {
  name                = "${var.prefix}-app"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  os_type         = "Linux"
  ip_address_type = "Public"
  dns_name_label  = "${var.prefix}-app"

  # メインコンテナ: nginx (Webサーバー)
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
      name       = "logs"
      mount_path = "/var/log/nginx"
      empty_dir  = true
    }
  }

  # サイドカーコンテナ: busybox (ログ監視)
  container {
    name   = "log-reader"
    image  = "busybox:latest"
    cpu    = "0.25"
    memory = "0.3"

    commands = [
      "/bin/sh", "-c",
      "while true; do if [ -f /logs/access.log ]; then tail -f /logs/access.log; else sleep 1; fi; done"
    ]

    volume {
      name       = "logs"
      mount_path = "/logs"
      empty_dir  = true
    }
  }

  tags = var.tags
}
```

実運用では busybox の代わりに Fluentd や Filebeat などを使うことになります。

### 実行

```bash
cd 04-multi-container
terraform init
terraform apply
```

### 動作確認

```bash
# nginx にアクセス
curl $(terraform output -raw access_url)

# サイドカーのログを確認（nginx のアクセスログが表示される）
az container logs \
  --resource-group aci-multi-rg \
  --name aci-multi-app \
  --container-name log-reader
```

### 各コンテナに入る

```bash
# nginx コンテナ
az container exec \
  --resource-group aci-multi-rg \
  --name aci-multi-app \
  --container-name nginx \
  --exec-command "/bin/sh"

# log-reader コンテナ
az container exec \
  --resource-group aci-multi-rg \
  --name aci-multi-app \
  --container-name log-reader \
  --exec-command "/bin/sh"
```

### Container Group の特徴

- 同じ Group 内のコンテナは **localhost** で通信可能
- ライフサイクルが共有される（一緒に起動・停止）
- リソース（CPU/メモリ）は Group 全体で共有
- パブリック IP は Group に1つ

### 注意点

- ポートは Group 内で重複不可
- 1つのコンテナが失敗すると Group 全体に影響
- Windows コンテナは複数コンテナ非対応

## 2. VNet 統合（プライベートネットワーク）

コンテナを VNet 内のプライベートサブネットに配置してみます。

### 構成図

```
┌─────────────────────────────────────────────────────┐
│                    VNet (10.0.0.0/16)               │
│  ┌───────────────────────────────────────────────┐  │
│  │         ACI Subnet (10.0.1.0/24)              │  │
│  │         delegation: ContainerInstance         │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │        Container Group                  │  │  │
│  │  │  ┌─────────────────────────────────┐    │  │  │
│  │  │  │           nginx                 │    │  │  │
│  │  │  │      Private IP: 10.0.1.x       │    │  │  │
│  │  │  └─────────────────────────────────┘    │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  （インターネットから直接アクセス不可）               │
└─────────────────────────────────────────────────────┘
```

### AWS との比較

| Azure ACI | AWS ECS on Fargate |
|-----------|-------------------|
| VNet + Subnet delegation | VPC + Private Subnet |
| subnet_ids | awsvpc ネットワークモード |
| ip_address_type = "Private" | assignPublicIp = "DISABLED" |

### VNet 統合の要件

1. **専用サブネット** - ACI 専用のサブネットが必要（他リソースと共有不可）
2. **Delegation** - `Microsoft.ContainerInstance/containerGroups` の delegation 設定

### Terraform コード

まず VNet と専用サブネットを作成します。

```hcl
# network.tf
resource "azurerm_virtual_network" "main" {
  name                = "${var.prefix}-vnet"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = ["10.0.0.0/16"]

  tags = var.tags
}

resource "azurerm_subnet" "aci" {
  name                 = "aci-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]

  # ACI 用の delegation（必須）
  delegation {
    name = "aci-delegation"

    service_delegation {
      name    = "Microsoft.ContainerInstance/containerGroups"
      actions = ["Microsoft.Network/virtualNetworks/subnets/action"]
    }
  }
}
```

コンテナを VNet 内に配置します。

```hcl
# aci.tf
resource "azurerm_container_group" "app" {
  name                = "${var.prefix}-app"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  os_type = "Linux"

  # VNet 統合: パブリック IP なし
  ip_address_type = "Private"
  subnet_ids      = [azurerm_subnet.aci.id]

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

### 実行

```bash
cd 05-vnet
terraform init
terraform apply
```

### プライベート IP を確認

```bash
terraform output container_private_ip
```

プライベート IP なので、同じ VNet 内の VM からのみアクセスできます。

```bash
# 同じ VNet 内の VM から
curl http://10.0.1.x
```

### インターネットからアクセスするには

VNet 内の ACI にインターネットからアクセスするには、以下の方法があります。

| 方法 | 説明 |
|------|------|
| Application Gateway | L7 ロードバランサー経由 |
| Azure Load Balancer | L4 ロードバランサー経由 |
| Azure Front Door | グローバル LB + CDN |
| Bastion + VM | 踏み台経由でアクセス |

### 注意点

- VNet 統合した ACI は `dns_name_label` 使用不可
- 一部リージョンでは VNet 統合非対応の場合あり

### ユースケース

- バックエンド API（フロントからのみアクセス）
- 内部バッチ処理
- データベースへのセキュアなアクセス
- マイクロサービス間通信

## クリーンアップ

```bash
terraform destroy
```

## まとめ

### サイドカーパターン

| 項目 | Azure ACI | AWS ECS |
|------|-----------|---------|
| コンテナグループ | Container Group | Task Definition |
| コンテナ間通信 | localhost | localhost |
| 共有ボリューム | emptyDir | bind mount |

### VNet 統合

| 項目 | Azure ACI | AWS ECS on Fargate |
|------|-----------|-------------------|
| プライベート配置 | `ip_address_type = "Private"` | `assignPublicIp = "DISABLED"` |
| ネットワーク指定 | `subnet_ids` | `awsvpcConfiguration` |
| 追加要件 | Subnet delegation | セキュリティグループ |

## ACI と ACA の使い分け

より複雑なコンテナワークロードには Azure Container Apps（ACA）も選択肢になります。

| 機能 | ACI | ACA |
|------|-----|-----|
| スケーリング | 手動 | 自動（KEDA ベース） |
| Ingress | 自前で構築 | 組み込み |
| Key Vault 連携 | アプリ側で実装 | 組み込み |
| マイクロサービス | 難しい | Dapr 統合 |

単発のバッチ処理や開発環境には ACI、本番のサービスには ACA という使い分けが良さそうです。

## 参考リンク

- [Azure Container Instances ドキュメント](https://learn.microsoft.com/ja-jp/azure/container-instances/)
- [Terraform azurerm_container_group](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/container_group)
