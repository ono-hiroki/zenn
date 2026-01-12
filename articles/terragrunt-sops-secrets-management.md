---
title: "Terragrunt + SOPS でシークレットを安全に Git 管理する"
emoji: "🏗️"
type: "tech"
topics: ["terragrunt", "sops", "terraform", "aws"]
published: true
---

IaC でシークレット（DB パスワード、API キーなど）をどう管理していますか？

この記事では、**SOPS** で暗号化したシークレットを Git 管理し、**Terragrunt** でシームレスに復号して使う方法を紹介します。

## よくある課題

| 方法 | 問題点 |
|------|--------|
| `.tfvars` に平文で記載 | Git に機密情報が残る |
| 環境変数で渡す | CI/CD の設定が複雑になる |
| Secrets Manager 等を直接参照 | Terraform 外で事前設定が必要 |

## 解決策：SOPS + Terragrunt

**SOPS**（Secrets OPerationS）は暗号化ツールです。YAML/JSON のキー名は平文のまま、**値だけを暗号化**できるため、Git の差分が見やすいのが特徴です。

Terragrunt は `sops_decrypt_file()` 関数をネイティブサポートしており、追加ツールなしで復号できます。

```
secrets.yaml（暗号化済み）
        ↓
    Terragrunt: sops_decrypt_file()
        ↓
    AWS KMS で自動復号
        ↓
    Terraform の inputs に渡す
```

## 前提条件

- Terragrunt / Terraform がインストール済み
- AWS KMS キーが作成済み（暗号化・復号に使用）
- 実行環境に KMS の `kms:Decrypt` 権限がある

## ディレクトリ構成例

```
live/
├── .sops.yaml              # SOPS 設定（どの KMS キーで暗号化するか）
└── dev/
    └── app/
        ├── terragrunt.hcl
        └── secrets.yaml    # 暗号化済みシークレット
```

## セットアップ手順

### 1. SOPS のインストール

```bash
brew install sops
```

### 2. .sops.yaml の作成

リポジトリルートに配置します。パスのパターンごとに使用する KMS キーを指定できます。

```yaml
creation_rules:
  - path_regex: secrets\.yaml$
    kms: arn:aws:kms:ap-northeast-1:123456789012:alias/my-key
```

### 3. シークレットファイルの作成と暗号化

まず平文で作成します。

```yaml
# secrets.yaml
db_password: my-secret-password
api_key: sk-1234567890abcdef
```

暗号化を実行します。

```bash
sops --encrypt --in-place secrets.yaml
```

暗号化後のファイルは以下のようになります（キー名は平文のまま）。

```yaml
db_password: ENC[AES256_GCM,data:xxx,iv:xxx,tag:xxx]
api_key: ENC[AES256_GCM,data:yyy,iv:yyy,tag:yyy]
sops:
    kms:
        - arn: arn:aws:kms:ap-northeast-1:123456789012:alias/my-key
          ...
```

### 4. Terragrunt での利用

```hcl
# terragrunt.hcl
locals {
  secrets = yamldecode(sops_decrypt_file("${get_terragrunt_dir()}/secrets.yaml"))
}

inputs = {
  db_password = local.secrets.db_password
  api_key     = local.secrets.api_key
}
```

`terragrunt plan` や `terragrunt apply` 実行時に自動で復号されます。

## 日常の操作

```bash
# 編集（エディタで開き、保存時に自動で再暗号化）
sops secrets.yaml

# 復号して内容を確認
sops --decrypt secrets.yaml
```

## メリットまとめ

| 特徴 | 説明 |
|------|------|
| Git で履歴管理 | 暗号化されているので安全にコミット可能 |
| 差分が見やすい | キー名が平文なので、どの値が変わったか分かる |
| 権限分離 | 環境ごとに異なる KMS キーを使い、アクセス制御 |
| シンプル | Terragrunt ネイティブ対応で追加ツール不要 |

## 参考リンク

- [SOPS 公式リポジトリ](https://github.com/getsops/sops)
- [Terragrunt sops_decrypt_file ドキュメント](https://terragrunt.gruntwork.io/docs/reference/built-in-functions/#sops_decrypt_file)
