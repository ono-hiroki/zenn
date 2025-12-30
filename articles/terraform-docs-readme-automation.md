---
title: "terraform-docsでTerraformモジュールのドキュメント自動生成を始める"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Terraform", "terraformdocs", "IaC", "ドキュメント"]
published: false
---

# terraform-docsでTerraformモジュールのREADMEを自動生成する

## はじめに

Terraformモジュールを作成する際、READMEの作成・更新は面倒な作業です。変数を追加するたびにREADMEを手動で更新するのは手間がかかり、ドキュメントと実装の乖離も起きやすくなります。

[terraform-docs](https://terraform-docs.io/)は、Terraformの設定ファイルから自動的にドキュメントを生成するツールです。この記事では、Markdown形式のREADME生成に焦点を当てて、実際に試した機能を紹介します。

## 環境

- terraform-docs v0.19.0
- macOS (Apple Silicon)

## インストール

macOSの場合、Homebrewでインストールできます。

```bash
brew install terraform-docs
```

その他のインストール方法は[公式ドキュメント](https://terraform-docs.io/user-guide/installation/)を参照してください。

## サンプルプロジェクトの構成

検証用に以下の構成でTerraformモジュールを作成しました。

```
terraform-docs-playground/
├── main.tf              # メイン設定（モジュール説明のコメント含む）
├── variables.tf         # 入力変数
├── outputs.tf           # 出力変数
├── examples/
│   └── basic/
│       └── main.tf      # 使用例
├── .terraform-docs.yml  # terraform-docs設定ファイル
└── README.md            # 自動生成されるドキュメント
```

### main.tf

モジュールの説明は`main.tf`の先頭にコメントとして記述します。このコメントがREADMEのヘッダーとして使用されます。

```hcl
/**
 * # AWS EC2 Instance Module
 *
 * このモジュールはAWS EC2インスタンスを作成します。
 *
 * ## 使用例
 *
 * ```hcl
 * module "ec2" {
 *   source = "./terraform-docs-playground"
 *
 *   instance_name = "my-instance"
 *   instance_type = "t3.micro"
 *   environment   = "dev"
 * }
 * ```
*/

terraform {
required_version = ">= 1.0.0"

required_providers {
aws = {
source  = "hashicorp/aws"
version = ">= 5.0.0"
}
}
}

# ... 以下リソース定義
```

### variables.tf

変数には必ず`description`を記述します。これがドキュメントの説明文になります。

```hcl
variable "instance_name" {
  description = "EC2インスタンスの名前"
  type        = string
}

variable "instance_type" {
  description = "EC2インスタンスタイプ"
  type        = string
  default     = "t3.micro"
}

variable "environment" {
  description = "環境名（dev, stg, prod）"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "stg", "prod"], var.environment)
    error_message = "environmentはdev, stg, prodのいずれかを指定してください"
  }
}

variable "instance_config" {
  description = "インスタンスの詳細設定"
  type = object({
    monitoring              = bool
    disable_api_termination = bool
    ebs_optimized           = bool
  })
  default = {
    monitoring              = false
    disable_api_termination = false
    ebs_optimized           = false
  }
}
```

### outputs.tf

出力にも`description`を記述します。

```hcl
output "instance_id" {
  description = "作成されたEC2インスタンスのID"
  value       = aws_instance.this.id
}

output "instance_public_ip" {
  description = "EC2インスタンスのパブリックIPアドレス"
  value       = aws_instance.this.public_ip
}
```

## 基本的な使い方

### コマンドラインから実行

最もシンプルな使い方は、モジュールのディレクトリでコマンドを実行することです。

```bash
# Markdownテーブル形式で標準出力に表示
terraform-docs markdown table .

# ファイルに出力
terraform-docs markdown table . --output-file README.md
```

### 出力形式の種類

Markdown形式には2種類あります。

#### markdown table

テーブル形式で出力します。コンパクトで見やすいのが特徴です。

```bash
terraform-docs markdown table .
```

**出力例：**

```markdown
## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| instance_name | EC2インスタンスの名前 | `string` | n/a | yes |
| instance_type | EC2インスタンスタイプ | `string` | `"t3.micro"` | no |
```

#### markdown document

リスト形式で詳細に出力します。各変数の情報が見やすくなります。

```bash
terraform-docs markdown document .
```

**出力例：**

```markdown
## Required Inputs

The following input variables are required:

### instance\_name

Description: EC2インスタンスの名前

Type: `string`

## Optional Inputs

The following input variables are optional (have default values):

### instance\_type

Description: EC2インスタンスタイプ

Type: `string`

Default: `"t3.micro"`
```

## 設定ファイル（.terraform-docs.yml）

プロジェクトルートに`.terraform-docs.yml`を配置すると、チーム全体で一貫した設定を共有できます。

### 基本的な設定ファイル

```yaml
formatter: "markdown table"

header-from: main.tf
footer-from: ""

output:
  file: "README.md"
  mode: inject
  template: |-
    <!-- BEGIN_TF_DOCS -->
    {{ .Content }}
    <!-- END_TF_DOCS -->

sort:
  enabled: true
  by: name

settings:
  anchor: true
  color: true
  default: true
  description: true
  escape: true
  hide-empty: false
  html: true
  indent: 2
  lockfile: true
  read-comments: true
  required: true
  sensitive: true
  type: true
```

### 主要な設定項目

| 項目 | 説明 |
|------|------|
| `formatter` | 出力形式（`markdown table`, `markdown document`など） |
| `header-from` | ヘッダーを読み込むファイル（デフォルト: `main.tf`） |
| `output.file` | 出力先ファイル名 |
| `output.mode` | `inject`（マーカー間に挿入）または`replace`（全体置換） |
| `sort.by` | ソート順（`name`, `required`） |

### output.mode: inject

`inject`モードを使用すると、READMEの特定の部分だけを更新できます。手書きの内容とterraform-docsの出力を共存させたい場合に便利です。

README.mdに以下のマーカーを配置しておきます：

```markdown
# My Module

手書きの説明文...

<!-- BEGIN_TF_DOCS -->
この部分がterraform-docsによって自動更新されます
<!-- END_TF_DOCS -->

## License

MIT
```

`terraform-docs .`を実行すると、マーカー間の内容だけが更新されます。

## カスタムテンプレート（content）

`content`オプションを使用すると、出力内容を自由にカスタマイズできます。

### セクションの順序変更

デフォルトの順序を変更したい場合：

```yaml
content: |-
  {{ .Header }}

  {{ .Inputs }}

  {{ .Outputs }}

  {{ .Requirements }}

  {{ .Providers }}

  {{ .Resources }}
```

### 外部ファイルの埋め込み（include）

`{{ include "path/to/file" }}`を使用すると、外部ファイルの内容を埋め込めます。使用例を別ファイルで管理したい場合に便利です。

```yaml
content: |-
  {{ .Header }}
  {{ .Requirements }}
  {{ .Providers }}
  ## Usage
  # ここに ```hcl と ``` で囲んでコードブロックを作成
  {{ include "examples/basic/main.tf" }}
  # コードブロック終了
  {{ .Inputs }}
  {{ .Outputs }}
  {{ .Resources }}
```

実際の設定では、`# ここに...` の部分を ` ```hcl ` と ` ``` ` に置き換えてください。

`examples/basic/main.tf`の内容：

```hcl
module "ec2" {
  source = "../../"

  instance_name = "my-instance"
  instance_type = "t3.micro"
  environment   = "dev"
}
```

これにより、READMEにUsageセクションが追加され、実際に動作するサンプルコードが埋め込まれます。

## セクションの表示/非表示

特定のセクションを非表示にできます。

### コマンドラインで指定

```bash
# modules と providers セクションを非表示
terraform-docs markdown table . --hide modules,providers

# inputs と outputs のみ表示
terraform-docs markdown table . --show inputs,outputs
```

### 設定ファイルで指定

```yaml
sections:
  hide: [modules, providers]
  # または
  show: [inputs, outputs, requirements]
```

## settingsオプション

出力の細かい調整ができます。

### anchor

HTMLアンカーリンクの有効/無効を切り替えます。

```yaml
settings:
  anchor: true  # デフォルト
```

**anchor: true の場合：**
```markdown
| <a name="input_instance_name"></a> [instance\_name](#input\_instance\_name) | ... |
```

**anchor: false の場合：**
```markdown
| instance\_name | ... |
```

GitHubなどでリッチな表示が不要な場合は`false`にするとシンプルになります。

### hide-empty

空のセクション（例：モジュールを使用していない場合の「Modules」）を非表示にします。

```yaml
settings:
  hide-empty: true
```

### その他の設定

```yaml
settings:
  default: true      # デフォルト値を表示
  description: true  # 説明を表示
  required: true     # Required列を表示
  sensitive: true    # sensitive変数にマーク
  type: true         # 型を表示
```

## 実践的な設定例

実際のプロジェクトで使用する設定ファイルの例です。

```yaml
formatter: "markdown table"

header-from: main.tf

recursive:
  enabled: false
  path: modules

sections:
  hide: []
  show: []

content: |-
  {{ .Header }}
  {{ .Requirements }}
  {{ .Providers }}
  ## Usage
  # ここに ```hcl と ``` で囲んでコードブロックを作成
  {{ include "examples/basic/main.tf" }}
  # コードブロック終了
  {{ .Inputs }}
  {{ .Outputs }}
  {{ .Resources }}

output:
  file: "README.md"
  mode: inject
  template: |-
    <!-- BEGIN_TF_DOCS -->
    {{ .Content }}
    <!-- END_TF_DOCS -->

sort:
  enabled: true
  by: name

settings:
  anchor: true
  default: true
  description: true
  hide-empty: true
  required: true
  sensitive: true
  type: true
```

※ `content`内の `# ここに...` の部分は、実際には ` ```hcl ` と ` ``` ` に置き換えてください。

## まとめ

terraform-docsを使用することで：

- **手動更新の手間を削減**：変数やoutputを追加/変更するだけでドキュメントが自動更新される
- **ドキュメントと実装の乖離を防止**：常に最新の状態が反映される
- **チームで一貫したドキュメント**：設定ファイルを共有することで統一されたフォーマットになる

特に`content`オプションと`include`機能を組み合わせることで、使用例を含んだ実用的なREADMEを自動生成できます。

Terraformモジュールを作成する際は、ぜひ導入を検討してみてください。

## 参考リンク

- [terraform-docs公式サイト](https://terraform-docs.io/)
- [terraform-docs GitHub](https://github.com/terraform-docs/terraform-docs)
- [Configuration - terraform-docs](https://terraform-docs.io/user-guide/configuration/)
