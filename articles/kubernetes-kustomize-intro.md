---
title: "Kubernetesをやってみる - Kustomizeで環境別マニフェストを管理する"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "kustomize", "container", "devops"]
published: true
---

## はじめに

この記事では、Kubernetes マニフェストをテンプレートなしでカスタマイズできる **Kustomize** について解説します。

Kustomize を使うことで、開発環境・ステージング環境・本番環境といった環境ごとの設定差分を、シンプルかつ直感的に管理できるようになります。

:::message
本記事では `kubectl` のエイリアスとして `k` を使用しています。
:::

## Kustomize とは

Kustomize は、**Kubernetes マニフェストをテンプレートなしでカスタマイズするためのツール**です。

kubectl v1.14 以降に組み込まれており、追加インストールなしで使用できます。

## Helm との違い

| 項目 | Kustomize | Helm |
|------|-----------|------|
| アプローチ | パッチベース（既存YAMLを上書き） | テンプレートベース（変数埋め込み） |
| 学習コスト | 低い（純粋なYAML） | やや高い（Go template構文） |
| 依存関係管理 | なし | あり（Chart依存関係） |
| パッケージ配布 | 向いていない | 向いている |
| kubectl統合 | 組み込み (`-k` フラグ) | 別途インストール必要 |
| ユースケース | 環境別の設定差分管理 | 複雑なアプリのパッケージ化・配布 |

### どちらを選ぶべき？

- **Kustomize**: 自社アプリの環境別デプロイ、シンプルな設定管理
- **Helm**: OSSツールのインストール、複雑な依存関係を持つアプリ

## 基本概念

### Base と Overlay

Kustomize の核となる概念は **Base（ベース）** と **Overlay（オーバーレイ）** です。

公式ドキュメントでは以下のように定義されています：

> **Base**: A kustomization that other kustomizations refer to. A base has no knowledge of the overlays that refer to it.
> （他のkustomizationから参照されるkustomization。Baseは自身を参照するoverlayについて関知しない）
>
> **Overlay**: A kustomization depending on another kustomization. An overlay is unusable without its bases.
> （他のkustomizationに依存するkustomization。Overlayは自身のbaseなしでは使用できない）
>
> — 出典: [Kustomize Glossary](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/)

簡潔に言うと：

- **Base**: 共通の基本設定（環境に依存しない部分）
- **Overlay**: 環境固有の設定（dev/staging/prod など）

```
kustomize/
├── base/                    # 共通設定
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/                 # 開発環境用
    │   └── kustomization.yaml
    └── prod/                # 本番環境用
        └── kustomization.yaml
```

### kustomization.yaml

各ディレクトリに配置する設定ファイルです。どのリソースを含めるか、どのような変換を適用するかを定義します。

## インストール確認

kubectl に組み込まれているため、特別なインストールは不要です。

```bash
# バージョン確認
k version --client

# kustomize サブコマンドの確認
k kustomize --help
```

## 実践：サンプルアプリのデプロイ

### 1. Base の作成

まず、環境に依存しない共通設定を作成します。

**base/deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: nginx
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
```

**base/service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
```

**base/kustomization.yaml**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

### 2. Overlay の作成（開発環境）

開発環境では、リソースを少なめに設定します。

**overlays/dev/kustomization.yaml**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

# リソース名に接頭辞を追加
namePrefix: dev-

# 共通ラベルを追加
commonLabels:
  env: development

# レプリカ数を変更
replicas:
  - name: myapp
    count: 1

# イメージタグを指定
images:
  - name: nginx
    newTag: "1.25"
```

### 3. Overlay の作成（本番環境）

本番環境では、レプリカ数を増やし、リソースも多めに設定します。

**overlays/prod/kustomization.yaml**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: prod-

commonLabels:
  env: production

replicas:
  - name: myapp
    count: 3

images:
  - name: nginx
    newTag: "1.25-alpine"

patches:
  - path: resource-patch.yaml
```

**overlays/prod/resource-patch.yaml**（リソース増強用パッチ）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp
          resources:
            requests:
              memory: "128Mi"
              cpu: "200m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

## コマンド

### マニフェストのプレビュー

```bash
# 開発環境の設定を確認
k kustomize overlays/dev

# 本番環境の設定を確認
k kustomize overlays/prod
```

### クラスタへの適用

```bash
# 開発環境にデプロイ
k apply -k overlays/dev

# 本番環境にデプロイ
k apply -k overlays/prod
```

### 削除

```bash
# 開発環境から削除
k delete -k overlays/dev

# 本番環境から削除
k delete -k overlays/prod
```

## よく使う機能

### 1. namePrefix / nameSuffix

リソース名に接頭辞・接尾辞を追加します。

```yaml
namePrefix: dev-
nameSuffix: -v2
# myapp → dev-myapp-v2
```

### 2. commonLabels / commonAnnotations

全リソースに共通のラベル・アノテーションを追加します。

```yaml
commonLabels:
  app.kubernetes.io/managed-by: kustomize
  env: production
```

### 3. images

イメージの名前やタグを変更します。

```yaml
images:
  - name: nginx           # 元のイメージ名
    newName: my-registry/nginx  # 新しいイメージ名（省略可）
    newTag: "2.0"         # 新しいタグ
```

### 4. replicas

Deployment のレプリカ数を変更します。

```yaml
replicas:
  - name: myapp
    count: 5
```

### 5. configMapGenerator / secretGenerator

ConfigMap や Secret を自動生成します。

```yaml
configMapGenerator:
  - name: app-config
    literals:
      - DATABASE_HOST=localhost
      - DATABASE_PORT=5432
    files:
      - config.json

secretGenerator:
  - name: app-secret
    literals:
      - API_KEY=supersecret
```

### 6. patches

Strategic Merge Patch または JSON Patch でリソースを部分的に変更します。

```yaml
# Strategic Merge Patch（ファイル指定）
patches:
  - path: increase-replicas.yaml

# インラインパッチ
patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp
      spec:
        replicas: 10
```

### 7. namespace

全リソースの namespace を変更します。

```yaml
namespace: my-namespace
```

## 推奨されるディレクトリ構造

公式ドキュメントでは、環境ごとのバリエーションを管理するために以下の構造が推奨されています：

> Manage traditional variants of a configuration - like development, staging and production - using overlays that modify a common base.
> （development、staging、production のような従来の構成バリエーションを、共通の base を変更する overlay を使って管理する）
>
> — 出典: [Kustomize Introduction](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/)

```
kustomize/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── config-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── config-patch.yaml
    └── prod/
        ├── kustomization.yaml
        ├── config-patch.yaml
        └── resource-patch.yaml
```

## トラブルシューティング

### よくあるエラー

**1. "resource not found" エラー**

```
Error: accumulating resources: ...
```

→ `resources` に指定したパスが正しいか確認

**2. パッチが適用されない**

→ パッチ内の `metadata.name` が Base のリソース名と一致しているか確認

**3. 重複したリソース**

```
Error: may not add resource with an already registered id
```

→ 同じリソースが複数回含まれていないか確認

### デバッグ方法

```bash
# 生成されるマニフェストを確認
k kustomize overlays/dev

# 詳細なエラーを表示
k kustomize overlays/dev --enable-alpha-plugins 2>&1

# dry-run で適用前に確認
k apply -k overlays/dev --dry-run=client -o yaml
```

## まとめ

- Kustomize は**テンプレートなし**でマニフェストをカスタマイズできる
- **Base + Overlay** 構造で環境ごとの差分を管理
- kubectl に**組み込み**されているため追加インストール不要
- シンプルな設定変更には `namePrefix`、`images`、`replicas` などを使用
- 複雑な変更には `patches` で Strategic Merge Patch を適用

## 参考リンク

- [Kustomize 公式ドキュメント](https://kustomize.io/)
- [Kubernetes公式: Kustomizeでの構成管理](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Kustomize GitHub](https://github.com/kubernetes-sigs/kustomize)
