---
title: "ecspresso と Terragrunt を組み合わせたらサービス名が２重管理になりそう"
emoji: "🔀"
type: "tech"
topics: ["ecspresso", "terragrunt", "ecs", "aws"]
published: false
---

## やりたかったこと

Terragrunt と ecspresso を組み合わせて ECS 環境を構築しています。

設定値はできるだけ1箇所で管理したい。特にサービス名のような基本的な値は、Terraform 側で定義して ecspresso から参照できると理想的です。

## tfstate プラグインを試してみた

ecspresso には tfstate プラグインがあり、Terraform の state から値を取得できます。

```yaml
# ecspresso.yml
plugins:
  - name: tfstate
    config:
      url: s3://bucket/path/terraform.tfstate
```

タスク定義の Jsonnet 内では、このように tfstate から値を取得できます。

```jsonnet
// ecs-task-def.jsonnet
local tfstate = std.native('tfstate');

{
  executionRoleArn: tfstate('output.task_execution_role_arn'),
  // ...
}
```

IAM ロールの ARN やセキュリティグループ ID などはこれで一元管理できました。

## ecspresso.yml 自体には tfstate が使えない

問題は、`ecspresso.yml` のトップレベル設定には tfstate プラグインが使えないことでした。

```yaml
# ecspresso.yml
region: ap-northeast-1
cluster: dev-cluster
service: dev-nginx  # ← ここは静的に書くしかない
```

Jsonnet 内では動的に値を取得できますが、サービス名やクラスタ名といった基本設定は静的に記述する必要があります。

## 検討した代替案

### 案1: 共通設定ファイル + 環境変数

サービス名を1箇所にまとめた設定ファイル（YAML や JSON）を用意し、Taskfile などから環境変数として渡す方法です。

ecspresso.yml は環境変数を展開できるため、以下のように書けます。

```yaml
# ecspresso.yml
service: "{{ must_env `SERVICE_NAME` }}"
```

一元管理は実現できますが、Taskfile などを経由しないと動作しなくなる点と、設定ファイルの管理が増える点がデメリットです。

### 案2: 別々に管理する

サービス名は一度決めたら変更頻度が低いため、Terragrunt と ecspresso で別々に管理する方法です。

シンプルですが、値がずれるリスクは残ります。

## 今回の選択: 2箇所管理

ecspresso を単独で実行できる柔軟性を残したかったため、2箇所で管理することにしました。

```hcl
# Terragrunt 側: live/dev/nginx-service.hcl
locals {
  service_name = "dev-nginx"
}
```

```yaml
# ecspresso 側: ecspresso.yml
service: dev-nginx
```

サービス名の変更頻度を考えると、実運用上は大きな問題にならないという判断です。

## おわりに

ecspresso.yml のトップレベル設定も tfstate から動的に取得できる方法があれば、ぜひ教えてください。

## 参考

- [ecspresso](https://github.com/kayac/ecspresso)
- [Terragrunt](https://terragrunt.gruntwork.io/)
