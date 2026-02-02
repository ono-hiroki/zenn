---
title: "Kubernetesをやってみる - Argo CDのApp of Appsパターン"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "argocd", "gitops", "devops"]
published: true
---

## はじめに

この記事では、ArgoCD の「App of Apps」パターンを学びます。親アプリケーションが子アプリケーションを管理し、すべてのアプリケーション定義を Git で一元管理する方法を実践します。

:::message
本記事では `kubectl` のエイリアスとして `k` を使用しています。
:::

## 検証する内容

| 項目 | 検証内容 |
|------|----------|
| App of Apps | 親 Application が子 Application を自動作成することを確認 |
| 宣言的管理 | アプリの追加・削除が Git 経由で管理されることを確認 |
| 一括デプロイ | 親をデプロイするだけで複数アプリが展開されることを確認 |

## App of Apps パターンとは

通常の ArgoCD 運用では、Application を一つずつ `argocd app create` で作成します。しかしアプリケーションが増えると管理が大変になります。

**App of Apps パターン**では、Application のマニフェスト（Application CRD）自体を Git で管理し、それを監視する「親アプリ」を作成します。

```
┌─────────────────────────────────────────────────────────────┐
│                        Git Repository                        │
│                                                              │
│  apps/                          manifests/                   │
│  ├── nginx-app.yaml             ├── nginx/                   │
│  ├── redis-app.yaml       →     │   ├── deployment.yaml      │
│  └── postgres-app.yaml          │   └── service.yaml         │
│       │                         ├── redis/                   │
│       │                         │   └── ...                  │
│       │                         └── postgres/                │
│       │                             └── ...                  │
└───────┼─────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────┐
│   Root App        │ ← argocd app create で作成
│   (親アプリ)       │
│   path: apps/     │
└───────┬───────────┘
        │ 自動作成
        ▼
┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐
│   nginx-app       │  │   redis-app       │  │   postgres-app    │
│   (子アプリ)       │  │   (子アプリ)       │  │   (子アプリ)       │
└───────────────────┘  └───────────────────┘  └───────────────────┘
```

### メリット

| 従来の方法 | App of Apps |
|-----------|-------------|
| `argocd app create` を手動で実行 | Git push するだけでアプリが追加される |
| どのアプリがあるか把握しづらい | Git を見ればすべてわかる |
| チームで共有しづらい | PR でレビュー・承認できる |

---

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

### GitHub アカウント

GitHub アカウントを持っていることを前提とします。

---

## Step 1: GitHub リポジトリを作成

### 1-1. 新しいリポジトリを作成

1. [GitHub](https://github.com) にログイン
2. 右上の「+」ボタン → 「New repository」をクリック
3. 以下の設定でリポジトリを作成:

| 項目 | 設定値 |
|------|--------|
| Repository name | `argocd-app-of-apps`（任意の名前） |
| Visibility | **Public** |
| Initialize this repository with | 「Add a README file」にチェック |

4. 「Create repository」をクリック

### 1-2. ローカルにクローン

```bash
# <your-username> を自分の GitHub ユーザー名に置き換え
git clone https://github.com/<your-username>/argocd-app-of-apps.git
cd argocd-app-of-apps
```

---

## Step 2: ディレクトリ構成を作成

App of Apps パターンでは、以下の 2 種類のディレクトリを作成します。

| ディレクトリ | 内容 |
|-------------|------|
| `apps/` | 子 Application のマニフェスト（Application CRD） |
| `manifests/` | 各アプリの Kubernetes マニフェスト |

### 2-1. ディレクトリを作成

```bash
mkdir -p apps
mkdir -p manifests/nginx
mkdir -p manifests/redis
```

---

## Step 3: Kubernetes マニフェストを作成

まず、子アプリがデプロイする実際の Kubernetes リソースを作成します。

### 3-1. nginx マニフェスト

`manifests/nginx/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 2
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
```

`manifests/nginx/service.yaml`:

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

### 3-2. redis マニフェスト

`manifests/redis/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7
          ports:
            - containerPort: 6379
```

`manifests/redis/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
spec:
  type: ClusterIP
  ports:
    - port: 6379
      targetPort: 6379
  selector:
    app: redis
```

---

## Step 4: 子 Application マニフェストを作成

ここが App of Apps の核心部分です。ArgoCD の Application リソース自体を YAML ファイルとして作成します。

### 4-1. nginx Application

`apps/nginx.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
  # 親 Application による削除時に子も削除されるようにする
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    # <your-username> を自分の GitHub ユーザー名に置き換え
    repoURL: https://github.com/<your-username>/argocd-app-of-apps.git
    targetRevision: HEAD
    path: manifests/nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 4-2. redis Application

`apps/redis.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    # <your-username> を自分の GitHub ユーザー名に置き換え
    repoURL: https://github.com/<your-username>/argocd-app-of-apps.git
    targetRevision: HEAD
    path: manifests/redis
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 4-3. ディレクトリ構造を確認

```bash
tree .
```

```
argocd-app-of-apps/
├── README.md
├── apps/
│   ├── nginx.yaml      # nginx Application の定義
│   └── redis.yaml      # redis Application の定義
└── manifests/
    ├── nginx/
    │   ├── deployment.yaml
    │   └── service.yaml
    └── redis/
        ├── deployment.yaml
        └── service.yaml
```

---

## Step 5: Git に push

```bash
git add .
git commit -m "Add app-of-apps structure with nginx and redis"
git push origin main
```

---

## Step 6: 親 Application（Root App）を作成

ここで唯一 `argocd app create` を実行します。この親アプリが `apps/` ディレクトリを監視し、子アプリを自動作成します。

### 6-1. Root App を作成

```bash
# <your-username> を自分の GitHub ユーザー名に置き換え
argocd app create root-app \
  --repo https://github.com/<your-username>/argocd-app-of-apps.git \
  --path apps \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd \
  --sync-policy automated \
  --self-heal \
  --auto-prune
```

> **ポイント**: `--path apps` で `apps/` ディレクトリを指定しています。ここに Application CRD が置いてあるため、ArgoCD はそれらを「デプロイ」= 子 Application として作成します。

### 6-2. Sync を実行（初回）

```bash
argocd app sync root-app
```

### 6-3. 状態を確認

```bash
argocd app list
```

```
NAME       CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS
nginx      https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  <none>
redis      https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  <none>
root-app   https://kubernetes.default.svc  argocd     default  Synced  Healthy  Auto-Prune  <none>
```

`nginx` と `redis` が自動的に作成されています。

### 6-4. Kubernetes リソースを確認

```bash
# nginx のリソース
k get all -l app=nginx

# redis のリソース
k get all -l app=redis
```

```
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
pod/nginx-xxxxxxxxxx-yyyyy   1/1     Running   0          30s

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/nginx   ClusterIP   10.96.xxx.xxx   <none>        80/TCP    30s
...
```

---

## Step 7: 新しいアプリを Git で追加

App of Apps の真価を確認します。新しいアプリを CLI ではなく **Git push だけ**で追加します。

### 7-1. httpbin マニフェストを作成

```bash
mkdir -p manifests/httpbin
```

`manifests/httpbin/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  labels:
    app: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
        - name: httpbin
          image: kennethreitz/httpbin
          ports:
            - containerPort: 80
```

`manifests/httpbin/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: httpbin
```

### 7-2. httpbin Application を作成

`apps/httpbin.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: httpbin
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    # <your-username> を自分の GitHub ユーザー名に置き換え
    repoURL: https://github.com/<your-username>/argocd-app-of-apps.git
    targetRevision: HEAD
    path: manifests/httpbin
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 7-3. Git push

```bash
git add .
git commit -m "Add httpbin application"
git push origin main
```

### 7-4. 自動作成を確認

```bash
# 数秒〜数十秒待ってから確認
argocd app list
```

```
NAME       CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS
httpbin    https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  <none>
nginx      https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  <none>
redis      https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  <none>
root-app   https://kubernetes.default.svc  argocd     default  Synced  Healthy  Auto-Prune  <none>
```

`httpbin` が自動的に追加されました。`argocd app create` は実行していません。

```bash
# httpbin の Pod も起動している
k get pods -l app=httpbin
```

---

## Step 8: アプリを Git で削除

同様に、アプリの削除も Git で管理できます。

### 8-1. redis Application を削除

```bash
# redis の Application マニフェストを削除
rm apps/redis.yaml

# マニフェストも削除（オプション）
rm -rf manifests/redis

git add .
git commit -m "Remove redis application"
git push origin main
```

### 8-2. 自動削除を確認

```bash
# 数秒〜数十秒待ってから確認
argocd app list
```

```
NAME       CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS
httpbin    https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  <none>
nginx      https://kubernetes.default.svc  default    default  Synced  Healthy  Auto-Prune  <none>
root-app   https://kubernetes.default.svc  argocd     default  Synced  Healthy  Auto-Prune  <none>
```

`redis` が自動的に削除されました。

```bash
# redis の Pod も削除されている
k get pods -l app=redis
```

```
No resources found in default namespace.
```

---

## Step 9: Web UI で確認（オプション）

ブラウザで https://localhost:8080 を開くと、App of Apps の階層構造が視覚的に確認できます。

1. 「Applications」画面で `root-app`、`nginx`、`httpbin` が表示される
2. `root-app` をクリックすると、子アプリ（nginx、httpbin の Application リソース）が表示される
3. `nginx` をクリックすると、実際の Deployment や Service が表示される

---

## Step 10: クリーンアップ

### 10-1. Root App を削除

親アプリを削除すると、`finalizers` の設定により子アプリも連鎖的に削除されます。

```bash
argocd app delete root-app --cascade -y
```

### 10-2. 削除を確認

```bash
# すべての Application が削除されたことを確認
argocd app list

# Kubernetes リソースも削除されたことを確認
k get pods -l app=nginx
k get pods -l app=httpbin
```

### 10-3. GitHub リポジトリを削除（オプション）

1. GitHub でリポジトリを開く
2. 「Settings」→「Delete this repository」

### 10-4. kind クラスタを削除（オプション）

```bash
kind delete cluster --name argocd-demo
```

---

## 補足: ApplicationSet との違い

ArgoCD には「ApplicationSet」という似た機能もあります。

| 機能 | App of Apps | ApplicationSet |
|------|-------------|----------------|
| **方式** | Application CRD を Git で管理 | テンプレートから Application を生成 |
| **柔軟性** | アプリごとに個別設定可能 | 同じパターンのアプリを大量生成向け |
| **用途** | 異なる設定の複数アプリを管理 | 複数クラスタ・環境への一括デプロイ |
| **学習コスト** | 低い（Application を書くだけ） | やや高い（Generator の理解が必要） |

### 使い分けの目安

- **App of Apps**: アプリごとに設定が異なる場合、少数〜中規模のアプリ管理
- **ApplicationSet**: 同じパターンで大量のアプリを管理、マルチクラスタ環境

---

## まとめ

このハンズオンで学んだこと：

| 項目 | 内容 |
|------|------|
| **App of Apps** | Application CRD を Git で管理し、親アプリが子アプリを自動作成 |
| **宣言的管理** | `argocd app create` を使わず、Git push だけでアプリを追加・削除 |
| **finalizers** | 親アプリ削除時に子アプリも連鎖削除するための設定 |
| **一元管理** | すべてのアプリ定義が Git に集約され、履歴・レビューが可能 |

```
【App of Apps のフロー】

  Developer          GitHub           ArgoCD
      │                 │                │
      │  apps/新app.yaml │                │
      │  を追加          │                │
      │─────────────────→│                │
      │                 │   root-app     │
      │                 │   が検知       │
      │                 │←───────────────│
      │                 │                │
      │                 │   子 App を    │
      │                 │   自動作成     │
      │                 │───────────────→│
      │                 │                │
      │                 │                │ K8s リソース
      │                 │                │ をデプロイ
```

### ベストプラクティス

| 項目 | 推奨 |
|------|------|
| **finalizers** | 子 Application には必ず `resources-finalizer.argocd.argoproj.io` を設定 |
| **ディレクトリ構成** | `apps/` と `manifests/` を分離し、役割を明確に |
| **Namespace** | 環境ごとに Namespace を分ける（`apps-dev/`、`apps-prod/`） |
| **Project** | 本番では ArgoCD Project を使ってアクセス制御 |

## 関連記事

- [Kubernetesをやってみる - Argo CDでGitOpsを始める](/articles/kubernetes-argocd-intro)
- [Kubernetesをやってみる - Argo CDで自分のGitHubリポジトリから自動デプロイ](/articles/kubernetes-argocd-github-deploy)
- [Kubernetesをやってみる - Argo CDでプライベートリポジトリからデプロイ](/articles/kubernetes-argocd-private-repo)
- [Kubernetesをやってみる - Argo CDでHelm / Kustomize統合](/articles/kubernetes-argocd-helm-kustomize)
