---
title: "Terragrunt × ecspresso 構成で初回デプロイ時に発生する依存関係問題と5つの解決策"
emoji: "🐣"
type: "tech"
topics: ["AWS", "ECS", "Terraform", "Terragrunt", "ecspresso"]
published: true
---

## はじめに

Terragrunt でインフラを管理し、ecspresso で ECS サービスをデプロイする構成は、それぞれのツールの強みを活かせる組み合わせです。しかし、**初回デプロイ時に依存関係の循環的な問題**が発生します。

この記事では、この問題の本質と、5つの解決策のメリット・デメリットを整理します。

## 問題：初回デプロイ時の依存関係

### 依存関係の構造

```
┌─────────────────────────────────────────────────────────────────┐
│                      依存関係の流れ                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Terraform/Terragrunt が管理                                   │
│   ┌─────────┐    ┌─────────┐    ┌──────────────────┐           │
│   │ Network │───▶│   ALB   │───▶│ ECS Cluster      │           │
│   └─────────┘    └─────────┘    │ IAM Roles        │           │
│                                 │ Security Groups  │           │
│                                 │ CloudWatch Logs  │           │
│                                 └────────┬─────────┘           │
│                                          │                      │
│                                          ▼                      │
│   ┌──────────────────────────────────────────────────────┐     │
│   │              ecspresso が管理                         │     │
│   │         ECS Task Definition / Service                │     │
│   └────────────────────────┬─────────────────────────────┘     │
│                            │                                    │
│                            ▼                                    │
│   ┌──────────────────────────────────────────────────────┐     │
│   │           Terraform/Terragrunt が管理                 │     │
│   │      Auto Scaling / CloudWatch Alarms                │     │
│   │      （ECS Service の存在が前提）                      │     │
│   └──────────────────────────────────────────────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### なぜ問題になるのか

1. **ecspresso は Terraform の出力に依存**
    - IAM ロールの ARN
    - セキュリティグループ ID
    - サブネット ID
    - ターゲットグループ ARN

2. **Auto Scaling は ECS サービスに依存**
    - `aws_appautoscaling_target` は既存の ECS サービスを参照
    - ECS サービスは ecspresso が作成するため、Terraform からは見えない

3. **初回デプロイ時の鶏と卵問題**
   ```
   terragrunt run-all apply を実行すると...

   ✅ Network, ALB, ECS Cluster, IAM ... 成功
   ❌ Auto Scaling ... 失敗（ECS サービスがまだ存在しない）

   その後 ecspresso deploy を手動実行...

   ✅ ECS Service 作成

   再度 terragrunt run-all apply ...

   ✅ Auto Scaling ... 成功（ECS サービスが存在する）
   ```

**つまり、初回だけ手順が複雑になる。**

## 解決策の比較

### 解決策 1: null_resource + local-exec で Terraform に統合

Terraform の `null_resource` と `local-exec` provisioner を使い、`terraform apply` 時に ecspresso を自動実行する。

```hcl
resource "null_resource" "ecspresso" {
  triggers = {
    cluster_name = var.cluster_name
    # ... 依存する値をすべて含める
  }

  provisioner "local-exec" {
    command     = "ecspresso deploy --config ecspresso.yml"
    working_dir = var.ecspresso_dir
    environment = {
      EXECUTION_ROLE_ARN = var.task_execution_role_arn
      # ... 環境変数で値を渡す
    }
  }

  provisioner "local-exec" {
    command = "ecspresso scale --tasks 0 && ecspresso delete --force"
    when    = destroy
  }
}
```

| メリット | デメリット |
|----------|------------|
| `terragrunt run-all apply` 一発で完結 | 実装が複雑（triggers の管理、環境変数の受け渡し） |
| `terraform destroy` で ECS サービスも削除される | ecspresso の設定を2種類用意する必要がある（tfstate 版と環境変数版） |
| 依存関係が Terragrunt で明示的に管理される | `null_resource` は状態管理が難しい（再実行の制御など） |
| CI/CD パイプラインがシンプル | ローカルに ecspresso のインストールが必須 |

### 解決策 2: シェルスクリプトで順序制御

デプロイ用のシェルスクリプトを作成し、実行順序を明示的に制御する。

```bash
#!/bin/bash
# deploy.sh

set -e

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

# Phase 1: インフラ（Auto Scaling 以外）
terragrunt run-all apply \
  --terragrunt-working-dir "${SCRIPT_DIR}/live/dev" \
  --terragrunt-exclude-dir "**/ecs-autoscaling" \
  --terragrunt-exclude-dir "**/ecs-alarms"

# Phase 2: ECS サービス
ecspresso deploy --config "${SCRIPT_DIR}/ecspresso/dev-nginx/ecspresso.yml"

# Phase 3: Auto Scaling / Alarms
terragrunt run-all apply \
  --terragrunt-working-dir "${SCRIPT_DIR}/live/dev"
```

| メリット | デメリット |
|----------|------------|
| 実装がシンプルで理解しやすい | 手動実行が必要（または CI/CD で複数ステップ） |
| 各ツールを独立して使える | `--terragrunt-exclude-dir` の管理が煩雑 |
| ecspresso の設定は1種類で済む | 削除時も同様のスクリプトが必要 |
| 既存の構成への影響が少ない | チームメンバー全員がスクリプトの存在を知る必要がある |

### 解決策 3: 初回だけ手動でコメントアウト

Auto Scaling など、ECS サービスに依存するモジュールを初回は除外し、後から追加する。

```hcl
# live/dev/ecs-autoscaling/terragrunt.hcl

# 初回デプロイ時はこのファイル全体をコメントアウト
# または skip = true を設定

# skip = true  # 初回のみ有効化

include "root" {
  path = find_in_parent_folders("root.hcl")
}
# ...
```

| メリット | デメリット |
|----------|------------|
| 追加の実装が不要 | 手動作業が必要（忘れやすい） |
| 既存の構成を変更しない | コメントのコミット忘れでチームが混乱 |
| 理解しやすい | 環境ごとに手順が必要 |
| | 自動化しにくい |

### 解決策 4: Terragrunt の skip + 環境変数

環境変数でモジュールの skip を制御する。

```hcl
# live/dev/ecs-autoscaling/terragrunt.hcl

skip = get_env("SKIP_ECS_DEPENDENT", "false") == "true"
```

```bash
# 初回デプロイ
SKIP_ECS_DEPENDENT=true terragrunt run-all apply
ecspresso deploy --config ecspresso.yml
terragrunt run-all apply

# 2回目以降
terragrunt run-all apply
```

| メリット | デメリット |
|----------|------------|
| コードの変更なしで制御可能 | 環境変数の存在を知る必要がある |
| CI/CD で制御しやすい | 初回かどうかの判断が必要 |
| コメントアウトより安全 | 複数モジュールの skip 管理が煩雑になりうる |

### 解決策 5: ECS サービスも Terraform で管理

ecspresso を使わず、ECS サービスも Terraform で管理する。

```hcl
resource "aws_ecs_service" "this" {
  name            = var.service_name
  cluster         = var.cluster_id
  task_definition = aws_ecs_task_definition.this.arn
  # ...
}

resource "aws_ecs_task_definition" "this" {
  family                = var.service_name
  container_definitions = jsonencode([...])
  # ...
}
```

| メリット | デメリット |
|----------|------------|
| 依存関係の問題が発生しない | タスク定義の変更が Terraform の state に影響 |
| ツールが1つで済む | ecspresso の便利機能が使えない（ロールバック、diff など） |
| `run-all apply` で完結 | タスク定義の JSON 管理が煩雑 |
| | アプリデプロイのたびに `terraform apply` が必要 |

## 比較表

| 解決策 | 初回の手間 | 運用の手間 | 実装コスト | 自動化 | ユースケース |
|--------|-----------|-----------|-----------|--------|-------------|
| 1. null_resource 統合 | ◎ 少ない | ◎ 少ない | △ 高い | ◎ | チームで長期運用 |
| 2. シェルスクリプト | ○ 普通 | ○ 普通 | ◎ 低い | ○ | バランス重視 |
| 3. 手動コメントアウト | △ 多い | △ 多い | ◎ 低い | × | 個人開発・検証環境 |
| 4. skip + 環境変数 | ○ 普通 | ○ 普通 | ○ 中程度 | ○ | CI/CD 重視 |
| 5. Terraform 一本化 | ◎ 少ない | △ 多い | ○ 中程度 | ◎ | ecspresso 不要な場合 |

## どれを選ぶべきか

### null_resource 統合（解決策 1）を選ぶ場合

- チームで長期的に運用する
- CI/CD パイプラインをシンプルにしたい
- 初期の実装コストを許容できる
- `terraform destroy` で完全にクリーンアップしたい

### シェルスクリプト（解決策 2）を選ぶ場合

- シンプルさを重視
- 既存の構成を大きく変えたくない
- ecspresso と Terraform を明確に分離したい

### 手動コメントアウト（解決策 3）を選ぶ場合

- 個人開発や検証環境
- 初回デプロイの頻度が低い
- 追加の実装を避けたい

### skip + 環境変数（解決策 4）を選ぶ場合

- CI/CD での制御を重視
- コードを変更せずに挙動を変えたい
- 複数環境で同じ設定を使いたい

### Terraform 一本化（解決策 5）を選ぶ場合

- ecspresso の機能（diff、ロールバックなど）が不要
- ツールを増やしたくない
- アプリデプロイの頻度が低い

## まとめ

Terragrunt と ecspresso を組み合わせる場合、**初回デプロイ時の依存関係問題**は避けられません。

- **完全な自動化**を目指すなら `null_resource` 統合
- **シンプルさ**を重視するならシェルスクリプトや手動対応
- **ecspresso の機能が不要**なら Terraform 一本化

プロジェクトの規模、チームの人数、運用頻度に応じて最適な解決策を選んでください。

## 参考

- [ecspresso 公式ドキュメント](https://github.com/kayac/ecspresso)
- [Terragrunt 公式ドキュメント](https://terragrunt.gruntwork.io/)
