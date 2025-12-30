---
title: "ecspresso と Terragrunt を組み合わせたらサービス名が２重管理になりそう"
emoji: "🔀"
type: "tech"
topics: ["ecspresso", "terragrunt", "ecs", "aws"]
published: false
---

## やりたかったこと

Terragrunt と ecspresso を組み合わせて ECS 環境を構築しています。

できれば設定値は1箇所で管理したい。特にサービス名のような基本的な値は、Terraform 側で定義して ecspresso から参照できると嬉しいなと思っていました。

## ecspresso の tfstate プラグインを試してみた

ecspresso には tfstate プラグインがあり、Terraform の state から値を取得できます。これを使えば一元管理できそう！

```yaml
# ecspresso.yml
plugins:
  - name: tfstate
    config:
      url: s3://bucket/path/terraform.tfstate
```

タスク定義の Jsonnet 内では、こんな感じで tfstate から値を取れます。

```jsonnet
// ecs-task-def.jsonnet
local tfstate = std.native('tfstate');

{
  executionRoleArn: tfstate('output.task_execution_role_arn'),
  // ...
}
```

IAM ロールの ARN やセキュリティグループ ID などはこれでうまくいきました。

## ただ、ecspresso.yml 自体のサービス名には使えなかった

ここで困ったのが、`ecspresso.yml` のトップレベル設定には tfstate プラグインが使えないこと。

```yaml
# ecspresso.yml
region: ap-northeast-1
cluster: dev-cluster
service: dev-nginx        # ← ここは静的に書くしかない
```

Jsonnet の中では `tfstate()` が使えるけど、この設定ファイル自体には使えないみたいです。

| 設定箇所 | tfstate |
|----------|---------|
| `ecspresso.yml` の `service` など | 使えない |
| Jsonnet 内 | 使える |

## 結局こうなった

### Terragrunt 側

```hcl
# live/dev/nginx-service.hcl
locals {
  service_name = "dev-nginx"
}
```

### ecspresso 側

```yaml
# ecspresso.yml
service: dev-nginx
```

同じ `dev-nginx` が2箇所に...。

## どうするか考えた

### 案1: 共通の設定ファイル + Taskfile で環境変数を渡す

サービス名を1箇所にまとめた設定ファイルを用意して、Taskfile などから環境変数として渡す方法。

一元管理はできるけど、Taskfile を経由しないと動かなくなるのと、設定ファイルの管理が増えるのが少し手間かなと。

### 案2: Terragruntとecspressoで別々に管理する

サービス名って一度決めたらそんなに変わらないので、Terragrunt と ecspresso で別々に管理する方法。


## 今回は2箇所管理にした

ecspresso を単独で実行できる柔軟性を残したかったので、今回は2箇所で管理することにしました。

サービス名の変更頻度を考えると、そこまで困らないかなという判断です。

## もっといい方法あるかな？

ecspresso.yml のサービス名も動的に取得できる方法があれば知りたいです。

## 参考

- [ecspresso](https://github.com/kayac/ecspresso)
- [Terragrunt](https://terragrunt.gruntwork.io/)
