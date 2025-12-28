---
title: "Terragrunt公式ガイド「Terralith to Terragrunt」ディレクトリ構成まとめ"
emoji: "🏗️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Terraform", "Terragrunt", "AWS", "IaC", "OpenTofu"]
published: true
---

## はじめに

この記事は、Terragrunt公式ドキュメントの「[Terralith to Terragrunt](https://terragrunt.gruntwork.io/docs/guides/terralith-to-terragrunt/)」を実践し、各Stepのディレクトリ構成をまとめたものです。

**Terralith**とは「Terraform + Monolith」の造語で、単一のStateファイルで大規模なインフラを管理している状態を指します。このガイドでは、Terralithの課題を段階的に解消し、スケーラブルなTerragrunt構成へ移行する過程のディレクトリ構成の変化を追っていきます。

### 構築するアプリケーション

猫画像に投票するWebアプリケーション「Best Cat」を題材に、以下のAWSリソースをデプロイします：

| リソース | 用途 |
|----------|------|
| S3 | 猫画像の保存 |
| DynamoDB | 投票メタデータの保存 |
| IAM Role | Lambda用の権限 |
| Lambda + Function URL | サーバーレスWebアプリ |

---

## Step 1: Starting the Terralith

### 概要

単一ディレクトリに全てのTerraformファイルを配置する「ナイーブな」構成からスタートします。

### ディレクトリ構成

```
terralith-to-terragrunt/
├── live/
│   ├── backend.tf        # S3バックエンド設定
│   ├── providers.tf      # AWSプロバイダー
│   ├── versions.tf       # Terraform/OpenTofu バージョン
│   ├── data.tf           # データソース
│   ├── s3.tf             # S3バケット定義
│   ├── ddb.tf            # DynamoDBテーブル定義
│   ├── iam.tf            # IAMロール・ポリシー定義
│   ├── lambda.tf         # Lambda関数定義
│   ├── outputs.tf        # 出力値
│   ├── vars-required.tf  # 必須変数
│   ├── vars-optional.tf  # オプション変数
│   └── terraform.auto.tfvars
├── app/
│   └── best-cat/         # Lambdaアプリケーション
└── dist/
    ├── best-cat.zip      # デプロイ用ZIP
    └── static/           # 猫画像
```

### 課題

| 課題 | 説明 |
|------|------|
| 再利用性がない | 全リソースが1ディレクトリに直書きで、他プロジェクトで再利用できない |
| 環境分離ができない | dev/prodを分けるには全ファイルをコピーする必要がある |
| 影響範囲が大きい | `apply`で全リソースが対象になる |

---

## Step 2: Refactoring

### 概要

リソースを再利用可能なモジュールに分割します。`moved`ブロックでStateのリソースアドレスを変更し、リソースの再作成を防ぎます。

### ディレクトリ構成

```
terralith-to-terragrunt/
├── catalog/
│   └── modules/
│       ├── ddb/
│       │   ├── main.tf
│       │   ├── vars-required.tf
│       │   └── outputs.tf
│       ├── s3/
│       │   ├── main.tf
│       │   ├── vars-required.tf
│       │   ├── vars-optional.tf
│       │   └── outputs.tf
│       ├── iam/
│       │   ├── main.tf
│       │   ├── data.tf
│       │   ├── vars-required.tf
│       │   └── outputs.tf
│       └── lambda/
│           ├── main.tf
│           ├── vars-required.tf
│           ├── vars-optional.tf
│           └── outputs.tf
└── live/
    ├── main.tf           # module呼び出し
    ├── moved.tf          # State移行定義
    ├── backend.tf
    ├── providers.tf
    ├── versions.tf
    └── outputs.tf
```

### 改善点

- リソース定義がモジュール化され、再利用可能に
- 各モジュールにinput/outputインターフェースが定義された
- `tofu plan`で**0 changes**を確認できる（リソース再作成なし）

### 学んだこと

```hcl
# moved.tf の例
moved {
  from = aws_s3_bucket.static_assets
  to   = module.s3.aws_s3_bucket.static_assets
}
```

`moved`ブロックにより、Stateファイル内のリソースアドレスを安全に変更できます。

---

## Step 3: Adding Dev

### 概要

モジュールを2回インスタンス化して、dev/prod両環境を単一Stateで管理します。

### ディレクトリ構成

```
terralith-to-terragrunt/
├── catalog/
│   └── modules/
│       ├── best_cat/     # 新規：オーケストレーションモジュール
│       │   ├── main.tf   # s3, ddb, iam, lambdaを呼び出し
│       │   ├── vars-required.tf
│       │   ├── vars-optional.tf
│       │   └── outputs.tf
│       ├── ddb/
│       ├── s3/
│       ├── iam/
│       └── lambda/
└── live/
    ├── main.tf           # module.dev, module.prod を定義
    ├── moved.tf          # module.prod.module.* への移動
    └── ...
```

### main.tf の例

```hcl
module "prod" {
  source          = "../catalog/modules/best_cat"
  name            = "my-best-cat-prod"
  lambda_zip_file = "../dist/best-cat.zip"
}

module "dev" {
  source          = "../catalog/modules/best_cat"
  name            = "my-best-cat-dev"
  lambda_zip_file = "../dist/best-cat.zip"
  force_destroy   = true
}
```

### 課題

| 課題 | 説明 |
|------|------|
| 単一Stateの危険性 | dev環境の変更がprodに影響するリスク |
| 爆発半径が大きい | `apply`で両環境が対象になる |
| 権限分離ができない | 開発者にprod変更権限を与えざるを得ない |

---

## Step 4: Breaking the Terralith

### 概要

環境ごとにディレクトリとStateファイルを分離し、爆発半径を縮小します。

### ディレクトリ構成

```
terralith-to-terragrunt/
├── catalog/
│   └── modules/
│       └── ...
└── live/
    ├── dev/
    │   ├── backend.tf    # key = "dev/tofu.tfstate"
    │   ├── main.tf       # module.main として best_cat を呼び出し
    │   ├── moved.tf      # module.dev.module.* → module.main.module.*
    │   ├── removed.tf    # prod リソースを State から forget
    │   └── ...
    └── prod/
        ├── backend.tf    # key = "prod/tofu.tfstate"
        ├── main.tf
        ├── moved.tf      # module.prod.module.* → module.main.module.*
        ├── removed.tf    # dev リソースを State から forget
        └── ...
```

### 改善点

- **爆発半径の縮小**: devディレクトリでの`apply`はdevリソースのみに影響
- **権限分離が可能**: IAMポリシーで環境別のアクセス制御が可能に

### 課題

- ボイラープレートの重複（backend.tf, providers.tf, versions.tfなど）
- 環境追加のたびに同じファイルをコピーする必要がある

### 学んだこと

```hcl
# removed.tf の例（prod環境側）
removed {
  from = module.dev.module.s3.aws_s3_bucket.static_assets
  lifecycle {
    destroy = false  # 実リソースは削除せず、Stateから除外のみ
  }
}
```

`removed`ブロックでStateからリソース参照を削除しつつ、実リソースは残せます。

---

## Step 5: Adding Terragrunt

### 概要

Terragruntを導入し、ボイラープレートコードを削減します。

### ディレクトリ構成

```
terralith-to-terragrunt/
├── catalog/
│   └── modules/
│       └── ...
└── live/
    ├── root.hcl          # 共通設定（providers, versions, backend生成）
    ├── dev/
    │   ├── terragrunt.hcl
    │   └── moved.tf      # 初回適用時のみ
    └── prod/
        ├── terragrunt.hcl
        └── moved.tf
```

### root.hcl

```hcl
locals {
  aws_region = "ap-northeast-1"
}

generate "providers" {
  path      = "providers.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${local.aws_region}"
}
EOF
}

generate "versions" {
  path      = "versions.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
terraform {
  required_version = ">= 1.10"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
EOF
}

remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket       = "my-best-cat-tfstate"
    key          = "${path_relative_to_include()}/tofu.tfstate"
    region       = local.aws_region
    encrypt      = true
    use_lockfile = true
  }
}
```

### dev/terragrunt.hcl

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "${find_in_parent_folders("catalog/modules")}//best_cat"
}

inputs = {
  name            = "my-best-cat-dev"
  lambda_zip_file = "${find_in_parent_folders("dist")}/best-cat.zip"
  force_destroy   = true
}
```

### 改善点

| 改善点 | 説明 |
|--------|------|
| DRY原則 | providers.tf, versions.tf, backend.tf が自動生成される |
| 環境追加が容易 | terragrunt.hcl を作成するだけで新環境を追加可能 |
| 一括操作 | `terragrunt run-all apply` で全環境を一括デプロイ |

---

## Step 6: Breaking the Terralith Further

### 概要

環境内をさらにコンポーネント（S3, DDB, IAM, Lambda）ごとに分割し、Stateの粒度を細かくします。

### ディレクトリ構成

```
terralith-to-terragrunt/
├── catalog/
│   └── modules/
│       ├── ddb/
│       ├── s3/
│       ├── iam/
│       └── lambda/
└── live/
    ├── root.hcl
    ├── dev/
    │   ├── s3/
    │   │   └── terragrunt.hcl
    │   ├── ddb/
    │   │   └── terragrunt.hcl
    │   ├── iam/
    │   │   └── terragrunt.hcl    # dependency: s3, ddb
    │   └── lambda/
    │       └── terragrunt.hcl    # dependency: s3, ddb, iam
    └── prod/
        └── (同構造)
```

### dependency ブロックの例

```hcl
# lambda/terragrunt.hcl
dependency "s3" {
  config_path = "../s3"
  mock_outputs = {
    name = "mock-bucket-name"
  }
}

dependency "iam" {
  config_path = "../iam"
  mock_outputs = {
    lambda_role_arn = "arn:aws:iam::123456789012:role/mock-role"
  }
}

inputs = {
  s3_bucket_name  = dependency.s3.outputs.name
  lambda_role_arn = dependency.iam.outputs.lambda_role_arn
}
```

### 改善点

| 改善点 | 説明 |
|--------|------|
| 爆発半径の最小化 | Lambda変更がS3/DynamoDBに影響しない |
| 並列デプロイ | 依存関係のないリソースは並列でapply可能 |
| 変更頻度による分離 | 頻繁に変わるLambdaと、ほぼ変わらないS3/DDBを分離 |

### 課題

- 各環境でs3, ddb, iam, lambdaの4つのterragrunt.hclが必要
- 環境追加時のコピー量が増加

---

## Step 7: Terragrunt Stacks

### 概要

Terragrunt Stacks（実験的機能）を使い、ユニット定義をカタログ化して`terragrunt.stack.hcl`で環境を定義します。

### ディレクトリ構成

```
terralith-to-terragrunt/
├── catalog/
│   ├── modules/
│   │   ├── ddb/
│   │   ├── s3/
│   │   ├── iam/
│   │   └── lambda/
│   └── units/              # 新規：ユニット定義のカタログ
│       ├── ddb/
│       │   └── terragrunt.hcl
│       ├── s3/
│       │   └── terragrunt.hcl
│       ├── iam/
│       │   └── terragrunt.hcl
│       └── lambda/
│           └── terragrunt.hcl
└── live/
    ├── root.hcl
    ├── dev/
    │   ├── terragrunt.stack.hcl   # Stack定義
    │   └── .terragrunt-stack/     # 生成されるディレクトリ
    │       ├── ddb/
    │       ├── s3/
    │       ├── iam/
    │       └── lambda/
    └── prod/
        ├── terragrunt.stack.hcl
        └── .terragrunt-stack/
```

### catalog/units/lambda/terragrunt.hcl

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "${find_in_parent_folders("catalog/modules")}//lambda"
}

dependency "s3" {
  config_path = values.s3_path    # values で外部から注入
  mock_outputs = { name = "mock-bucket" }
}

dependency "iam" {
  config_path = values.iam_path
  mock_outputs = { lambda_role_arn = "arn:aws:iam::..." }
}

inputs = {
  name            = values.name
  lambda_zip_file = values.lambda_zip_file
  s3_bucket_name  = dependency.s3.outputs.name
  lambda_role_arn = dependency.iam.outputs.lambda_role_arn
}
```

### dev/terragrunt.stack.hcl

```hcl
locals {
  name            = "my-best-cat-dev"
  aws_region      = "ap-northeast-1"
  units_path      = "${find_in_parent_folders("catalog/units")}"
  lambda_zip_file = "${find_in_parent_folders("dist")}/best-cat.zip"
}

unit "ddb" {
  source = "${local.units_path}/ddb"
  path   = "ddb"
  values = { name = local.name }
}

unit "s3" {
  source = "${local.units_path}/s3"
  path   = "s3"
  values = { name = local.name, force_destroy = true }
}

unit "iam" {
  source = "${local.units_path}/iam"
  path   = "iam"
  values = {
    name       = local.name
    aws_region = local.aws_region
    s3_path    = "../s3"
    ddb_path   = "../ddb"
  }
}

unit "lambda" {
  source = "${local.units_path}/lambda"
  path   = "lambda"
  values = {
    name            = local.name
    lambda_zip_file = local.lambda_zip_file
    s3_path         = "../s3"
    ddb_path        = "../ddb"
    iam_path        = "../iam"
  }
}
```

### 改善点

| 改善点 | 説明 |
|--------|------|
| 究極のDRY | ユニット定義は1箇所、環境差分はvaluesで注入 |
| 環境追加が簡単 | terragrunt.stack.hcl 1ファイルで環境定義完了 |
| 依存関係の可視化 | `terragrunt graph`で依存グラフを生成可能 |

### 実行コマンド

```bash
# Stack の生成と適用
cd live/dev
terragrunt --experiment stacks stack generate
terragrunt --experiment stacks --experiment cli-redesign run --all apply
```

---

## Step 8: Refactoring state with Terragrunt Stacks

### 概要

`.terragrunt-stack/`ディレクトリを使用するTerragrunt標準構造に移行します。

### 最終ディレクトリ構成

```
terralith-to-terragrunt/
├── catalog/
│   ├── modules/
│   │   ├── ddb/
│   │   │   ├── main.tf
│   │   │   ├── vars-required.tf
│   │   │   └── outputs.tf
│   │   ├── s3/
│   │   │   ├── main.tf
│   │   │   ├── vars-required.tf
│   │   │   ├── vars-optional.tf
│   │   │   └── outputs.tf
│   │   ├── iam/
│   │   │   ├── main.tf
│   │   │   ├── data.tf
│   │   │   ├── vars-required.tf
│   │   │   └── outputs.tf
│   │   └── lambda/
│   │       ├── main.tf
│   │       ├── vars-required.tf
│   │       ├── vars-optional.tf
│   │       └── outputs.tf
│   └── units/
│       ├── ddb/
│       │   └── terragrunt.hcl
│       ├── s3/
│       │   └── terragrunt.hcl
│       ├── iam/
│       │   └── terragrunt.hcl
│       └── lambda/
│           └── terragrunt.hcl
├── live/
│   ├── root.hcl
│   ├── dev/
│   │   └── terragrunt.stack.hcl
│   └── prod/
│       └── terragrunt.stack.hcl
├── app/
│   └── best-cat/
└── dist/
    ├── best-cat.zip
    └── static/
```

### Stateファイルの配置

```
S3: my-best-cat-tfstate/
├── dev/.terragrunt-stack/ddb/tofu.tfstate
├── dev/.terragrunt-stack/s3/tofu.tfstate
├── dev/.terragrunt-stack/iam/tofu.tfstate
├── dev/.terragrunt-stack/lambda/tofu.tfstate
├── prod/.terragrunt-stack/ddb/tofu.tfstate
├── prod/.terragrunt-stack/s3/tofu.tfstate
├── prod/.terragrunt-stack/iam/tofu.tfstate
└── prod/.terragrunt-stack/lambda/tofu.tfstate
```

---

## まとめ：各Stepの課題と改善

| Step | 構成 | 課題 | 改善点 |
|------|------|------|--------|
| 1 | 単一ディレクトリ | 再利用不可、環境分離不可 | - |
| 2 | モジュール化 | まだ単一State | コード再利用可能に |
| 3 | 複数環境（単一State） | 爆発半径大、権限分離不可 | dev/prod両対応 |
| 4 | 環境分離 | ボイラープレート重複 | 爆発半径縮小 |
| 5 | Terragrunt導入 | まだ環境単位のState | DRY、自動生成 |
| 6 | コンポーネント分離 | terragrunt.hclが多い | 最小爆発半径 |
| 7 | Stacks導入 | 実験的機能 | 究極のDRY |
| 8 | 標準構造 | - | ベストプラクティス準拠 |

---

## トラブルシューティング

### Lambda Function URL で 403 Forbidden

Lambda Function URLを使用する場合、以下の**両方**のpermissionが必要です：

```hcl
resource "aws_lambda_permission" "function_url_public" {
  statement_id           = "FunctionURLAllowPublicAccess"
  action                 = "lambda:InvokeFunctionUrl"
  function_name          = aws_lambda_function.main.function_name
  principal              = "*"
  function_url_auth_type = "NONE"
}

resource "aws_lambda_permission" "function_invoke_public" {
  statement_id  = "FunctionInvokeAllowPublicAccess"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.main.function_name
  principal     = "*"
}
```

---

## 参考リンク

- [公式ガイド: Terralith to Terragrunt](https://terragrunt.gruntwork.io/docs/guides/terralith-to-terragrunt/)
- [Terragrunt Stacks ドキュメント](https://terragrunt.gruntwork.io/docs/features/stacks/)
