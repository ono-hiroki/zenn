---
title: "Kubernetesをやってみる - Argo CDでHelm / Kustomize統合"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "argocd", "gitops", "devops"]
published: false
---

## はじめに

この記事では、ArgoCD で Helm チャートと Kustomize を使用する方法を学びます。実運用では環境ごとに設定を変えることが多く、これらのツールとの統合は必須知識です。

:::message
本記事では `kubectl` のエイリアスとして `k` を使用しています。
:::

## 検証する内容

| 項目 | 検証内容 |
|------|----------|
| Helm チャート | 外部 Helm リポジトリからアプリをデプロイ |
| Helm values | values ファイルで設定をカスタマイズ |
| Kustomize | base/overlay パターンで環境別設定を管理 |

## Helm と Kustomize の使い分け

| 項目 | Helm | Kustomize |
|------|------|-----------|
| **用途** | パッケージ化されたアプリのデプロイ | 既存マニフェストのカスタマイズ |
| **設定方法** | values.yaml でパラメータ指定 | patch で差分を適用 |
| **よくある使用例** | nginx-ingress, cert-manager, Prometheus | 自社アプリの環境別設定（dev/prod） |
| **テンプレート** | Go template | なし（純粋な YAML） |

---

# Part 1: Helm 統合

## 0. 前提条件

### Kubernetes クラスタ（kind）

```bash
# クラスタがない場合は作成
kind create cluster --name argocd-demo

# 確認
k get nodes
```

### ArgoCD がインストール・ログイン済み

```bash
# ArgoCD がインストールされていない場合
k create namespace argocd
k apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
k wait --for=condition=Ready pods --all -n argocd --timeout=300s

# port-forward（別ターミナルで実行）
k port-forward svc/argocd-server -n argocd 8080:443

# ログイン
argocd login localhost:8080 --username admin --password $(argocd admin initial-password -n argocd | head -1) --insecure
```

---

## Step 1: 外部 Helm チャートをデプロイ

ArgoCD は外部の Helm リポジトリから直接チャートをデプロイできます。

### 1-1. nginx を Helm チャートでデプロイ

Bitnami の nginx チャートを使用します。

```bash
argocd app create nginx-helm \
  --repo https://charts.bitnami.com/bitnami \
  --helm-chart nginx \
  --revision 18.2.5 \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
```

#### オプションの説明

| オプション | 説明 |
|-----------|------|
| `--repo` | Helm リポジトリの URL |
| `--helm-chart` | チャート名 |
| `--revision` | チャートのバージョン |

### 1-2. 状態を確認

```bash
argocd app get nginx-helm
```

```bash
k get all -l app.kubernetes.io/name=nginx
```

### 1-3. クリーンアップ

```bash
argocd app delete nginx-helm --cascade -y
```

---

## Step 2: values ファイルで設定をカスタマイズ

実際の運用では、values ファイルを Git で管理し、環境ごとに設定を変えます。

### 2-1. GitHub リポジトリを作成

1. GitHub で `argocd-helm-demo` リポジトリを作成（Public）
2. ローカルにクローン

```bash
git clone https://github.com/<your-username>/argocd-helm-demo.git
cd argocd-helm-demo
```

### 2-2. ディレクトリ構成を作成

```bash
mkdir -p helm/nginx
```

### 2-3. values ファイルを作成

`helm/nginx/values.yaml`:

```yaml
# レプリカ数
replicaCount: 2

# Service の設定
service:
  type: ClusterIP
  port: 80

# リソース制限
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

# カスタム設定
serverBlock: |-
  server {
    listen 8080;
    location /health {
      return 200 'healthy\n';
      add_header Content-Type text/plain;
    }
  }
```

### 2-4. Application マニフェストを作成（オプション）

CLI ではなく、マニフェストでも定義できます。

`helm/nginx/application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-helm-custom
  namespace: argocd
spec:
  project: default
  source:
    # Helm リポジトリ
    repoURL: https://charts.bitnami.com/bitnami
    chart: nginx
    targetRevision: 18.2.5
    helm:
      # Git リポジトリ内の values ファイルを参照
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

> **Note**: この方法では values ファイルは Helm リポジトリではなく、Git リポジトリから参照されます。

### 2-5. Git リポジトリ内の values を使う構成

より実践的な構成では、values ファイルを Git で管理し、ArgoCD がそれを参照します。

```
argocd-helm-demo/
├── helm/
│   └── nginx/
│       ├── values.yaml          # デフォルト値
│       ├── values-dev.yaml      # 開発環境用
│       └── values-prod.yaml     # 本番環境用
```

`helm/nginx/values-dev.yaml`:

```yaml
replicaCount: 1

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

`helm/nginx/values-prod.yaml`:

```yaml
replicaCount: 3

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 200m
    memory: 256Mi
```

### 2-6. push して Application 作成

```bash
git add .
git commit -m "Add nginx helm values"
git push origin main
```

CLI で values ファイルを指定してデプロイ:

```bash
# 開発環境
argocd app create nginx-dev \
  --repo https://charts.bitnami.com/bitnami \
  --helm-chart nginx \
  --revision 18.2.5 \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev \
  --helm-set replicaCount=1 \
  --helm-set resources.limits.memory=128Mi \
  --sync-policy automated

# 本番環境
argocd app create nginx-prod \
  --repo https://charts.bitnami.com/bitnami \
  --helm-chart nginx \
  --revision 18.2.5 \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace prod \
  --helm-set replicaCount=3 \
  --helm-set resources.limits.memory=512Mi \
  --sync-policy automated
```

> **Note**: namespace が存在しない場合は事前に作成してください: `k create ns dev && k create ns prod`

### 2-7. クリーンアップ

```bash
argocd app delete nginx-dev --cascade -y
argocd app delete nginx-prod --cascade -y
```

---

# Part 2: Kustomize 統合

Kustomize は Kubernetes に組み込まれたマニフェスト管理ツールです。base（共通設定）と overlay（環境別設定）のパターンで設定を管理します。

## Step 3: Kustomize の基本構成

### 3-1. GitHub リポジトリを作成

1. GitHub で `argocd-kustomize-demo` リポジトリを作成（Public）
2. ローカルにクローン

```bash
git clone https://github.com/<your-username>/argocd-kustomize-demo.git
cd argocd-kustomize-demo
```

### 3-2. ディレクトリ構成を作成

```bash
mkdir -p kustomize/base
mkdir -p kustomize/overlays/dev
mkdir -p kustomize/overlays/prod
```

```
kustomize/
├── base/                    # 共通設定
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/                 # 開発環境
    │   └── kustomization.yaml
    └── prod/                # 本番環境
        ├── kustomization.yaml
        └── replica-patch.yaml
```

---

## Step 4: base マニフェストを作成

### 4-1. Deployment

`kustomize/base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
```

### 4-2. Service

`kustomize/base/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: nginx
```

### 4-3. kustomization.yaml

`kustomize/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  managed-by: argocd
```

---

## Step 5: overlay（環境別設定）を作成

### 5-1. 開発環境（dev）

`kustomize/overlays/dev/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

# リソース名にプレフィックスを追加
namePrefix: dev-

# 共通ラベルを追加
commonLabels:
  env: dev

# namespace を指定
namespace: dev

# イメージタグを変更
images:
  - name: nginx
    newTag: "1.25"
```

### 5-2. 本番環境（prod）

`kustomize/overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: prod-

commonLabels:
  env: prod

namespace: prod

images:
  - name: nginx
    newTag: "1.25-alpine"

# パッチでカスタマイズ
patches:
  - path: replica-patch.yaml
```

`kustomize/overlays/prod/replica-patch.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: nginx
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

---

## Step 6: Git に push

```bash
git add .
git commit -m "Add kustomize base and overlays"
git push origin main
```

---

## Step 7: ArgoCD で Kustomize アプリをデプロイ

### 7-1. namespace を作成

```bash
k create namespace dev
k create namespace prod
```

### 7-2. 開発環境をデプロイ

```bash
argocd app create nginx-kustomize-dev \
  --repo https://github.com/<your-username>/argocd-kustomize-demo.git \
  --path kustomize/overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev \
  --sync-policy automated \
  --self-heal
```

### 7-3. 本番環境をデプロイ

```bash
argocd app create nginx-kustomize-prod \
  --repo https://github.com/<your-username>/argocd-kustomize-demo.git \
  --path kustomize/overlays/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace prod \
  --sync-policy automated \
  --self-heal
```

> **ポイント**: `--path` で overlay ディレクトリを指定するだけで、ArgoCD が自動的に Kustomize を検出して適用します。

### 7-4. 状態を確認

```bash
argocd app list
```

```
NAME                   CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH
nginx-kustomize-dev    https://kubernetes.default.svc  dev        default  Synced  Healthy
nginx-kustomize-prod   https://kubernetes.default.svc  prod       default  Synced  Healthy
```

```bash
# 開発環境: replicas=1, prefix=dev-
k get all -n dev

# 本番環境: replicas=3, prefix=prod-
k get all -n prod
```

開発環境:
```
NAME                             READY   STATUS    RESTARTS   AGE
pod/dev-nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

本番環境:
```
NAME                              READY   STATUS    RESTARTS   AGE
pod/prod-nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
pod/prod-nginx-xxxxxxxxxx-yyyyy   1/1     Running   0          30s
pod/prod-nginx-xxxxxxxxxx-zzzzz   1/1     Running   0          30s
```

---

## Step 8: 設定変更の自動反映を確認

### 8-1. dev の replicas を変更

`kustomize/overlays/dev/kustomization.yaml` に replicas パッチを追加:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: dev-

commonLabels:
  env: dev

namespace: dev

images:
  - name: nginx
    newTag: "1.25"

# replicas を直接指定（Kustomize v4.x+）
replicas:
  - name: nginx
    count: 2
```

### 8-2. push して自動同期を確認

```bash
git add .
git commit -m "Scale dev nginx to 2 replicas"
git push origin main
```

```bash
# 数秒後に確認
k get pods -n dev
```

---

## Step 9: クリーンアップ

### 9-1. Application を削除

```bash
argocd app delete nginx-kustomize-dev --cascade -y
argocd app delete nginx-kustomize-prod --cascade -y
```

### 9-2. namespace を削除

```bash
k delete namespace dev
k delete namespace prod
```

### 9-3. GitHub リポジトリを削除（オプション）

### 9-4. kind クラスタを削除（オプション）

```bash
kind delete cluster --name argocd-demo
```

---

## 補足: ArgoCD が Helm/Kustomize を検出する仕組み

ArgoCD は指定されたパスに以下のファイルがあると自動的にツールを検出します。

| ツール | 検出条件 |
|--------|----------|
| **Helm** | `Chart.yaml` が存在 |
| **Kustomize** | `kustomization.yaml` または `kustomization.yml` が存在 |
| **Plain YAML** | 上記がない場合、`.yaml` / `.yml` ファイルを直接適用 |

### 明示的に指定する場合

Application マニフェストで明示的に指定することもできます:

```yaml
spec:
  source:
    path: my-app
    # Kustomize を明示的に使用
    kustomize:
      namePrefix: prod-
      images:
        - nginx:1.25-alpine
```

```yaml
spec:
  source:
    path: my-chart
    # Helm を明示的に使用
    helm:
      valueFiles:
        - values-prod.yaml
      parameters:
        - name: replicaCount
          value: "3"
```

---

## まとめ

このハンズオンで学んだこと：

| 項目 | 内容 |
|------|------|
| **Helm 外部チャート** | `--helm-chart` で外部リポジトリから直接デプロイ |
| **Helm values** | `--helm-set` や values ファイルで設定をカスタマイズ |
| **Kustomize 自動検出** | `kustomization.yaml` があれば自動的に Kustomize を使用 |
| **overlay パターン** | base + overlays で環境別設定を管理 |

```
【Helm と Kustomize の使い分け】

┌─────────────────────────────────────────────────────────┐
│                    外部チャート                          │
│        (nginx-ingress, cert-manager, etc.)              │
│                         │                               │
│                         ▼                               │
│                 ┌──────────────┐                        │
│                 │     Helm     │                        │
│                 │  + values    │                        │
│                 └──────────────┘                        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    自社アプリ                            │
│              (環境別に設定を変える)                       │
│                         │                               │
│                         ▼                               │
│    ┌─────────┐    ┌───────────┐    ┌───────────┐       │
│    │  base   │───→│ overlay   │───→│ overlay   │       │
│    │ (共通)  │    │   (dev)   │    │  (prod)   │       │
│    └─────────┘    └───────────┘    └───────────┘       │
│                    Kustomize                            │
└─────────────────────────────────────────────────────────┘
```

### 実運用でのベストプラクティス

| 項目 | 推奨 |
|------|------|
| **外部チャート** | Helm を使用。バージョンを固定 |
| **自社アプリ** | Kustomize で base/overlay 管理 |
| **設定ファイル** | Git で管理し、PR でレビュー |
| **機密情報** | values に直接書かず、Sealed Secrets や External Secrets を使用 |

## 関連記事

- [Kubernetesをやってみる - Argo CDでGitOpsを始める](/articles/kubernetes-argocd-intro)
- [Kubernetesをやってみる - Argo CDで自分のGitHubリポジトリから自動デプロイ](/articles/kubernetes-argocd-github-deploy)
- [Kubernetesをやってみる - Argo CDでプライベートリポジトリからデプロイ](/articles/kubernetes-argocd-private-repo)
- [Kubernetesをやってみる - Argo CDのApp of Appsパターン](/articles/kubernetes-argocd-app-of-apps)
