---
title: "Azureをやってみる - AWSのEC2相当のサーバーをTerraformで立ち上げる"
emoji: "🫐"
type: "tech"
topics: ["azure", "terraform", "aws"]
published: false
---

AWSは触ったことがあるけどAzureは初めて、という方向けに、AWSとの対比を交えながらAzure上にWebサーバーを構築する手順を紹介します。

## AWSとAzureの用語対応

まず、AWSとAzureで対応するリソース名を押さえておきます。

| AWS | Azure | 説明 |
|-----|-------|------|
| VPC | VNet (Virtual Network) | 仮想ネットワーク |
| Subnet | Subnet | サブネット |
| Security Group | NSG (Network Security Group) | ファイアウォール |
| EC2 | VM (Virtual Machine) | 仮想サーバー |
| Elastic IP | Public IP | 固定パブリックIP |
| ENI | NIC (Network Interface) | ネットワークインターフェース |
| IAM | Microsoft Entra ID (旧Azure AD) | ID管理 |
| Region | Region | リージョン |
| AZ | Availability Zone | 可用性ゾーン |

大きな違いとして、**AzureではNICとPublic IPを明示的に作成**する必要があります。

AWSの場合:
- EC2を作成すると、ENI（ネットワークインターフェース）が暗黙的に作られる
- `associate_public_ip_address = true` でパブリックIPが自動付与される

Azureの場合:
- Public IP → NIC → VM の順で**それぞれ個別にリソースを作成**してアタッチする
- VMはNICなしでは作成できない

## 作るもの

VNet上にパブリックサブネットを作成し、nginxが動作するLinux VMを公開します。

```
┌─────────────────────────────────────────────────────────────┐
│ Resource Group                                              │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ VNet: 10.0.0.0/16                                     │  │
│  │                                                       │  │
│  │  ┌─────────────────────┐  ┌─────────────────────┐    │  │
│  │  │ Public Subnet       │  │ Private Subnet      │    │  │
│  │  │ 10.0.1.0/24         │  │ 10.0.2.0/24         │    │  │
│  │  │                     │  │                     │    │  │
│  │  │  ┌─────────────┐    │  │                     │    │  │
│  │  │  │ VM (nginx)  │    │  │                     │    │  │
│  │  │  │ + Public IP │    │  │                     │    │  │
│  │  │  └─────────────┘    │  │                     │    │  │
│  │  │                     │  │                     │    │  │
│  │  │  [NSG: SSH/HTTP]    │  │  [NSG: VNet only]   │    │  │
│  │  └─────────────────────┘  └─────────────────────┘    │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## コード

https://github.com/ono-hiroki/maitake/tree/main/azure-network

## リソースグループ

Azureには**リソースグループ**という概念があります。すべてのリソースは必ずどこかのリソースグループに所属します。

AWSでいうと「タグでグルーピングしたもの」に近いですが、Azureではリソースグループを削除するとその中のリソースがすべて削除されるため、環境のクリーンアップが簡単です。

```hcl
resource "azurerm_resource_group" "main" {
  name     = "${var.prefix}-rg"
  location = var.location
  tags     = var.tags
}
```

## VNetとサブネット

AWSのVPCに相当するのがVNet（Virtual Network）です。

```hcl
resource "azurerm_virtual_network" "main" {
  name                = "${var.prefix}-vnet"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "public" {
  name                 = "${var.prefix}-public-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}
```

AWSと違い、Azureのサブネットはデフォルトでインターネットへのアウトバウンド通信が可能です。Internet Gatewayやルートテーブルを明示的に作成する必要はありません。

## NSG（Network Security Group）

AWSのセキュリティグループに相当するのがNSGです。

```hcl
resource "azurerm_network_security_group" "public" {
  name                = "${var.prefix}-public-nsg"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "AllowSSH"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "AllowHTTP"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}
```

AWSとの違い:
- **priority（優先度）** が必須。数字が小さいほど優先
- デフォルトで**アウトバウンドは許可**、**インバウンドは拒否**

NSGはサブネットまたはNICにアタッチできます。

```hcl
resource "azurerm_subnet_network_security_group_association" "public" {
  subnet_id                 = azurerm_subnet.public.id
  network_security_group_id = azurerm_network_security_group.public.id
}
```

## Public IPとNIC

VMにパブリックIPを付与するには、まずPublic IPリソースを作成し、NICにアタッチします。

```hcl
resource "azurerm_public_ip" "web" {
  name                = "${var.prefix}-web-pip"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_network_interface" "web" {
  name                = "${var.prefix}-web-nic"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.public.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.web.id
  }
}
```

AWSではEC2作成時に `associate_public_ip_address = true` とするだけでしたが、Azureでは明示的にPublic IP → NIC → VMという依存関係を作ります。

## VM（Virtual Machine）

AWSのEC2に相当するのがVMです。

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  name                = "${var.prefix}-web-vm"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  size                  = "Standard_D2s_v3"
  admin_username        = "azureuser"
  network_interface_ids = [azurerm_network_interface.web.id]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  custom_data = base64encode(<<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl enable nginx
    systemctl start nginx
    echo "<h1>Hello from Azure VM!</h1>" > /var/www/html/index.html
  EOF
  )
}
```

AWSとの対比:

| 項目 | AWS (EC2) | Azure (VM) |
|------|-----------|------------|
| インスタンスタイプ | `instance_type = "t3.micro"` | `size = "Standard_D2s_v3"` |
| AMI | `ami = "ami-xxx"` | `source_image_reference` ブロック |
| ユーザーデータ | `user_data` | `custom_data`（base64エンコード必須） |
| キーペア | `key_name` | `admin_ssh_key` ブロック |
| EBS | `root_block_device` | `os_disk` ブロック |

## 実行

```bash
# Azure CLIでログイン
az login

# Terraform実行
terraform init
terraform apply
```

## 動作確認

```bash
# Webサーバーにアクセス
$ curl http://$(terraform output -raw vm_public_ip)
<h1>Hello from Azure VM!</h1><p>Hostname: network-basic-web-vm</p>

# SSH接続
$ ssh azureuser@$(terraform output -raw vm_public_ip)
Welcome to Ubuntu 22.04.5 LTS
```

## クリーンアップ

```bash
terraform destroy
```

リソースグループごと削除されるので、AWSよりクリーンアップが簡単です。

## ハマりポイント

### VMサイズの在庫切れ

日本リージョンでは `Standard_B1s` などの小さいサイズが在庫切れになることがあります。

```
Error: SkuNotAvailable: Standard_B1s is currently not available in location 'japaneast'
```

事前に利用可能なサイズを確認しておくと安心です。

```bash
az vm list-skus --location japanwest --size Standard_D --output table
```

### Gen1/Gen2イメージの互換性

VMサイズによってはGen2イメージが使えません。古いサイズ（Aシリーズなど）を使う場合は `sku = "22_04-lts"`（Gen1）を指定してください。

## まとめ

| AWS | Azure | 備考 |
|-----|-------|------|
| VPC | VNet | ほぼ同じ |
| Security Group | NSG | priorityが必要 |
| EC2 | VM | NICを明示的に作成 |
| (暗黙) | Resource Group | リソースのグルーピング |
| IGW + Route Table | (不要) | デフォルトでインターネット接続可能 |

AWSに慣れていれば、用語の違いを押さえるだけでAzureも同じ感覚で使えます。
