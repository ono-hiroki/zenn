---
title: "Kubernetesをやってみる - Argo CDで自分のGitHubリポジトリから自動デプロイ"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "argocd", "gitops", "devops"]
published: true
---

## はじめに

この記事では、自分の GitHub リポジトリにマニフェストを push し、ArgoCD が変更を検知して自動デプロイされることを確認します。

:::message
本記事では `kubectl` のエイリアスとして `k` を使用しています。
:::

## 検証する内容

| 項目 | 検証内容 |
|------|----------|
| GitOps ワークフロー | Git push をトリガーにデプロイが自動実行されることを確認 |
| 自動同期 | マニフェストの変更（replicas など）が自動的にクラスタに反映されることを確認 |

## 構成

```
┌─────────────────┐      ┌─────────────┐      ┌────────────────┐
│   ローカル PC    │      │   GitHub    │      │  Kubernetes    │
│                 │      │             │      │   Cluster      │
│  マニフェスト作成 │─────→│  リポジトリ  │←─────│   ArgoCD       │
│  git push       │      │             │      │                │
└─────────────────┘      └─────────────┘      └────────────────┘
                              │                      │
                              │   監視・検知          │
                              └──────────────────────┘
                                       │
                                       ▼
                              ┌────────────────┐
                              │  nginx Pod     │
                              │  nginx Service │
                              └────────────────┘
```

---

## 0. 前提条件

### Kubernetes クラスタ（kind）

クラスタがない場合は kind で作成します。

```bash
# kind がインストールされていない場合
brew install kind

# クラスタを作成
kind create cluster --name argocd-demo

# 確認
k get nodes
```

### ArgoCD のインストール

```bash
# namespace を作成
k create namespace argocd

# ArgoCD をインストール
k apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Pod が起動するまで待機
k wait --for=condition=Ready pods --all -n argocd --timeout=300s

# 確認
k get pods -n argocd
```

### ArgoCD CLI のインストール

```bash
# macOS
brew install argocd

# バージョン確認
argocd version --client
```

### ArgoCD にログイン

```bash
# 別のターミナルで port-forward を実行
k port-forward svc/argocd-server -n argocd 8080:443

# 初期パスワードを取得
argocd admin initial-password -n argocd

# CLI でログイン（パスワードは上記で取得したもの）
argocd login localhost:8080 --username admin --password <password> --insecure

# 確認
argocd app list
```

### GitHub アカウント

GitHub アカウントを持っていることを前提とします。

---

## Step 1: GitHub で Public リポジトリを作成

### 1-1. GitHub で新しいリポジトリを作成

1. [GitHub](https://github.com) にログイン
2. 右上の「+」ボタン → 「New repository」をクリック
3. 以下の設定でリポジトリを作成:

| 項目 | 設定値 |
|------|--------|
| Repository name | `argocd-demo`（任意の名前） |
| Visibility | **Public**（ArgoCD から認証なしでアクセスするため） |
| Initialize this repository with | 「Add a README file」にチェック |

4. 「Create repository」をクリック

> **Note**: Private リポジトリの場合は ArgoCD にリポジトリの認証情報を登録する必要があります。このハンズオンでは簡単のため Public リポジトリを使用します。

---

## Step 2: ローカルにクローン & マニフェスト作成

### 2-1. リポジトリをクローン

```bash
# <your-username> を自分の GitHub ユーザー名に置き換えてください
git clone https://github.com/<your-username>/argocd-demo.git
cd argocd-demo
```

### 2-2. マニフェスト用ディレクトリを作成

```bash
mkdir -p manifests/nginx
```

### 2-3. Deployment マニフェストを作成

`manifests/nginx/deployment.yaml`:

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
```

### 2-4. Service マニフェストを作成

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

### 2-5. ディレクトリ構造を確認

```bash
tree .
```

```
argocd-demo/
├── README.md
└── manifests/
    └── nginx/
        ├── deployment.yaml
        └── service.yaml
```

---

## Step 3: 初回 push

### 3-1. Git にコミット & push

```bash
git add .
git commit -m "Add nginx manifests"
git push origin main
```

### 3-2. GitHub で確認

ブラウザで GitHub リポジトリを開き、`manifests/nginx/` ディレクトリとファイルが表示されていることを確認します。

---

## Step 4: ArgoCD で Application を作成

### 4-1. Application を CLI で作成

```bash
# <your-username> を自分の GitHub ユーザー名に置き換えてください
argocd app create nginx-demo \
  --repo https://github.com/<your-username>/argocd-demo.git \
  --path manifests/nginx \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --self-heal \
  --auto-prune
```

#### オプションの説明

| オプション | 説明 |
|-----------|------|
| `--repo` | マニフェストが置いてある GitHub リポジトリの URL |
| `--path` | リポジトリ内のマニフェストのパス |
| `--dest-server` | デプロイ先の Kubernetes クラスタ（同一クラスタ） |
| `--dest-namespace` | デプロイ先の Namespace |
| `--sync-policy automated` | Git の変更を自動で同期 |
| `--self-heal` | 手動変更を Git の状態に自動で戻す |
| `--auto-prune` | Git から削除されたリソースをクラスタからも削除 |

### 4-2. Application の状態を確認

```bash
argocd app get nginx-demo
```

```
Name:               argocd/nginx-demo
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://localhost:8080/applications/nginx-demo
Repo:               https://github.com/<your-username>/argocd-demo.git
Target:
Path:               manifests/nginx
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced to  (xxxxxxx)
Health Status:      Healthy
```

`Sync Status: Synced` と `Health Status: Healthy` になっていれば成功です。

### 4-3. デプロイされたリソースを確認

```bash
k get all -l app=nginx
```

```
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          30s

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/nginx   ClusterIP   10.96.xxx.xxx   <none>        80/TCP    30s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           30s
```

### 4-4. ブラウザで確認（オプション）

#### ArgoCD Web UI

```bash
# 別のターミナルで port-forward（前提条件で実行済みの場合は不要）
k port-forward svc/argocd-server -n argocd 8080:443
```

ブラウザで https://localhost:8080 を開き、`nginx-demo` Application を確認します。

#### nginx の動作確認

```bash
# 別のターミナルで nginx Service を port-forward
k port-forward svc/nginx 8081:80
```

ブラウザで http://localhost:8081 を開き、nginx のウェルカムページが表示されることを確認します。

```
Welcome to nginx!
If you see this page, the nginx web server is successfully installed and working.
...
```

---

## Step 5: 自動デプロイを確認（replicas 変更）

Git push によって変更が自動的にクラスタに反映されることを確認します。

### 5-1. replicas を変更

`manifests/nginx/deployment.yaml` を編集して、replicas を 1 から 3 に変更します。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3    # 1 → 3 に変更
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

### 5-2. 変更を Git push

```bash
git add .
git commit -m "Scale nginx to 3 replicas"
git push origin main
```

### 5-3. 自動同期を確認

ArgoCD は数秒〜数十秒で Git の変更を検知し、自動的にクラスタに反映します。

```bash
# Pod が 3 つに増えていることを確認
k get pods -l app=nginx -w
```

```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          5m
nginx-xxxxxxxxxx-yyyyy   1/1     Running   0          10s
nginx-xxxxxxxxxx-zzzzz   1/1     Running   0          10s
```

### 5-4. ArgoCD の同期履歴を確認

```bash
argocd app history nginx-demo
```

```
ID  DATE                           REVISION
0   2024-01-01 12:00:00 +0900 JST  xxxxxxx (HEAD)
1   2024-01-01 12:05:00 +0900 JST  yyyyyyy (HEAD)
```

### 5-5. Self Heal の確認（オプション）

手動で replicas を変更しても、ArgoCD が自動的に Git の状態に戻すことを確認します。

```bash
# 手動で replicas を 1 に変更
k scale deployment nginx --replicas=1

# すぐに確認
k get pods -l app=nginx

# 数秒〜数十秒後に確認（3 に戻っている）
k get pods -l app=nginx
```

---

## Step 6: クリーンアップ

### 6-1. ArgoCD Application を削除

```bash
# Application を削除（関連する Kubernetes リソースも削除される）
argocd app delete nginx-demo --cascade -y
```

### 6-2. 削除を確認

```bash
# Application が削除されたことを確認
argocd app list

# Kubernetes リソースも削除されたことを確認
k get all -l app=nginx
```

### 6-3. GitHub リポジトリを削除（オプション）

不要であれば、GitHub で作成した `argocd-demo` リポジトリを削除します。

1. GitHub でリポジトリを開く
2. 「Settings」タブ → 一番下の「Delete this repository」

### 6-4. kind クラスタを削除（オプション）

```bash
kind delete cluster --name argocd-demo
```

---

## まとめ

このハンズオンで学んだこと：

| 項目 | 内容 |
|------|------|
| **GitOps ワークフロー** | Git リポジトリが唯一の信頼できる情報源（Single Source of Truth）となる |
| **自動同期** | `--sync-policy automated` により、Git push するだけで自動的にデプロイされる |
| **Self Heal** | `--self-heal` により、手動変更を Git の状態に自動的に戻す |
| **Prune** | `--auto-prune` により、Git から削除されたリソースはクラスタからも削除される |

```
【GitOps のフロー】

  Developer          GitHub           ArgoCD           Kubernetes
      │                 │                │                  │
      │  git push       │                │                  │
      │────────────────→│                │                  │
      │                 │   監視・検知    │                  │
      │                 │←───────────────│                  │
      │                 │                │   auto sync      │
      │                 │                │─────────────────→│
      │                 │                │                  │
      │                 │                │                  │ replicas: 3
      │                 │                │                  │
```

これにより、以下のメリットが得られます：

- **変更履歴の追跡**: すべての変更が Git に記録される
- **レビュープロセス**: PR を使って変更をレビューできる
- **ロールバック**: 問題があれば `git revert` で簡単に戻せる
- **一貫性**: 手動変更が自動的に修正され、Git と常に同期される

## 関連記事

- [Kubernetesをやってみる - Argo CDでGitOpsを始める](/articles/kubernetes-argocd-intro)
- [Kubernetesをやってみる - Argo CDでプライベートリポジトリからデプロイ](/articles/kubernetes-argocd-private-repo)
- [Kubernetesをやってみる - Argo CDのApp of Appsパターン](/articles/kubernetes-argocd-app-of-apps)
- [Kubernetesをやってみる - Argo CDでHelm / Kustomize統合](/articles/kubernetes-argocd-helm-kustomize)
