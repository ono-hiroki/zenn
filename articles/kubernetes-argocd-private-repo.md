---
title: "Kubernetesをやってみる - Argo CDでプライベートリポジトリからデプロイ"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "argocd", "gitops", "devops"]
published: true
---

## はじめに

この記事では、GitHub のプライベートリポジトリから ArgoCD でデプロイする方法を学びます。

この記事のソースは [GitHub リポジトリ](https://github.com/ono-hiroki/maitake/tree/main/kubernetes/14-argocd-introduction/hands-on) で公開しています。

:::message
本記事では `kubectl` のエイリアスとして `k` を使用しています。
:::

## 検証する内容

| 項目 | 検証内容 |
|------|----------|
| リポジトリ認証 | ArgoCD にプライベートリポジトリの認証情報を登録する |
| 認証付きデプロイ | 認証が必要なリポジトリから自動デプロイできることを確認 |

## 認証方法

ArgoCD でプライベートリポジトリにアクセスする方法は主に 3 つあります。

| 方法 | 特徴 |
|------|------|
| **HTTPS + Personal Access Token** | 設定が簡単。このハンズオンで使用 |
| **SSH Key** | PAT より安全。鍵の管理が必要 |
| **GitHub App** | 組織向け。自動ローテーション対応 |

このハンズオンでは最もシンプルな **HTTPS + Personal Access Token (PAT)** を使用します。

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

## Step 1: GitHub で Private リポジトリを作成

### 1-1. 新しいリポジトリを作成

1. [GitHub](https://github.com) にログイン
2. 右上の「+」ボタン → 「New repository」をクリック
3. 以下の設定でリポジトリを作成:

| 項目 | 設定値 |
|------|--------|
| Repository name | `argocd-private-demo`（任意の名前） |
| Visibility | **Private** |
| Initialize this repository with | 「Add a README file」にチェック |

4. 「Create repository」をクリック

---

## Step 2: Personal Access Token (PAT) を作成

### 2-1. GitHub で PAT を生成

1. GitHub にログイン
2. 右上のアイコン → 「Settings」
3. 左メニューの一番下「Developer settings」
4. 「Personal access tokens」→「Tokens (classic)」
5. 「Generate new token」→「Generate new token (classic)」

### 2-2. トークンの設定

| 項目 | 設定値 |
|------|--------|
| Note | `argocd-demo`（任意の名前） |
| Expiration | 「7 days」など短めに設定 |
| Select scopes | **`repo`** にチェック（Full control of private repositories） |

6. 「Generate token」をクリック
7. **表示されたトークンをコピーして安全な場所に保存**（この画面を閉じると二度と表示されません）

```
ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

---

## Step 3: ArgoCD にリポジトリを登録

### 3-1. 認証情報付きでリポジトリを登録

```bash
# <your-username> を自分の GitHub ユーザー名に置き換え
# <your-token> を Step 2 で取得した PAT に置き換え
argocd repo add https://github.com/<your-username>/argocd-private-demo.git \
  --username <your-username> \
  --password <your-token>
```

成功すると以下のように表示されます:

```
Repository 'https://github.com/<your-username>/argocd-private-demo.git' added
```

### 3-2. 登録を確認

```bash
argocd repo list
```

```
TYPE  NAME  REPO                                                        INSECURE  OCI    LFS    CREDS  STATUS      MESSAGE  PROJECT
git         https://github.com/<your-username>/argocd-private-demo.git  false     false  false  true   Successful
```

`CREDS` が `true` になっていれば、認証情報が登録されています。

### 3-3. Web UI で確認（オプション）

ブラウザで https://localhost:8080 を開き:

1. 左メニュー「Settings」→「Repositories」
2. 登録したリポジトリが表示されていることを確認
3. 「CONNECTION STATUS」が「Successful」になっていることを確認

---

## Step 4: マニフェストを作成して push

### 4-1. リポジトリをクローン

```bash
# <your-username> を自分の GitHub ユーザー名に置き換え
git clone https://github.com/<your-username>/argocd-private-demo.git
cd argocd-private-demo
```

### 4-2. マニフェストを作成

```bash
mkdir -p manifests/nginx
```

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

### 4-3. push

```bash
git add .
git commit -m "Add nginx manifests"
git push origin main
```

---

## Step 5: ArgoCD で Application を作成

### 5-1. Application を作成

```bash
# <your-username> を自分の GitHub ユーザー名に置き換え
argocd app create nginx-private \
  --repo https://github.com/<your-username>/argocd-private-demo.git \
  --path manifests/nginx \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --self-heal \
  --auto-prune
```

> **Note**: リポジトリは Step 3 で既に登録済みなので、Application 作成時に認証情報を再度指定する必要はありません。

### 5-2. 状態を確認

```bash
argocd app get nginx-private
```

```
Sync Status:        Synced to  (xxxxxxx)
Health Status:      Healthy
```

### 5-3. デプロイを確認

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

### 5-4. ブラウザで確認（オプション）

```bash
# nginx Service を port-forward
k port-forward svc/nginx 8081:80
```

ブラウザで http://localhost:8081 を開き、nginx のウェルカムページが表示されることを確認します。

---

## Step 6: 自動デプロイを確認

### 6-1. replicas を変更

`manifests/nginx/deployment.yaml` の replicas を 1 → 3 に変更:

```yaml
spec:
  replicas: 3    # 1 → 3 に変更
```

### 6-2. push

```bash
git add .
git commit -m "Scale nginx to 3 replicas"
git push origin main
```

### 6-3. 自動同期を確認

```bash
# Pod が 3 つに増えていることを確認
k get pods -l app=nginx -w
```

---

## Step 7: クリーンアップ

### 7-1. ArgoCD Application を削除

```bash
argocd app delete nginx-private --cascade -y
```

### 7-2. ArgoCD からリポジトリ登録を削除

```bash
argocd repo rm https://github.com/<your-username>/argocd-private-demo.git
```

### 7-3. GitHub PAT を削除

1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. 作成したトークンの「Delete」をクリック

### 7-4. GitHub リポジトリを削除（オプション）

1. GitHub でリポジトリを開く
2. 「Settings」→「Delete this repository」

### 7-5. kind クラスタを削除（オプション）

```bash
kind delete cluster --name argocd-demo
```

---

## 補足: SSH Key で認証する場合

PAT の代わりに SSH Key を使用する場合の手順です。

### SSH Key を生成

```bash
# ArgoCD 用の鍵を生成
ssh-keygen -t ed25519 -C "argocd" -f ~/.ssh/argocd_ed25519 -N ""
```

### GitHub に公開鍵を登録

1. GitHub → Settings → SSH and GPG keys → New SSH key
2. 公開鍵を登録:

```bash
cat ~/.ssh/argocd_ed25519.pub
```

### ArgoCD にリポジトリを登録

```bash
argocd repo add git@github.com:<your-username>/argocd-private-demo.git \
  --ssh-private-key-path ~/.ssh/argocd_ed25519
```

### Application 作成時の URL

SSH の場合は URL 形式が異なります:

```bash
argocd app create nginx-private \
  --repo git@github.com:<your-username>/argocd-private-demo.git \
  --path manifests/nginx \
  ...
```

---

## まとめ

このハンズオンで学んだこと：

| 項目 | 内容 |
|------|------|
| **リポジトリ登録** | `argocd repo add` で認証情報付きでリポジトリを登録 |
| **PAT 認証** | `--username` と `--password` で Personal Access Token を指定 |
| **SSH 認証** | `--ssh-private-key-path` で SSH 秘密鍵を指定 |
| **認証の分離** | リポジトリ登録と Application 作成は別。一度登録すれば Application 作成時に認証不要 |

```
【プライベートリポジトリのフロー】

1. リポジトリ登録（認証情報を ArgoCD に保存）
   argocd repo add <url> --username <user> --password <token>

2. Application 作成（認証情報は自動で使用される）
   argocd app create <name> --repo <url> ...

3. 自動デプロイ
   Git push → ArgoCD が検知 → 登録済みの認証情報でアクセス → デプロイ
```

### 本番環境での考慮事項

| 項目 | ローカル | 本番 |
|------|---------|------|
| 認証情報の保存 | ArgoCD に直接登録 | Secret Manager + External Secrets Operator |
| PAT の有効期限 | 短め（7日など） | 定期的なローテーション or GitHub App |
| アクセス権限 | 個人の PAT | 専用のサービスアカウント or GitHub App |

## 関連記事

- [Kubernetesをやってみる - Argo CDでGitOpsを始める](https://zenn.dev/hono8944/articles/kubernetes-argocd-intro)
- [Kubernetesをやってみる - Argo CDで自分のGitHubリポジトリから自動デプロイ](https://zenn.dev/hono8944/articles/kubernetes-argocd-github-deploy)
- [Kubernetesをやってみる - Argo CDのApp of Appsパターン](https://zenn.dev/hono8944/articles/kubernetes-argocd-app-of-apps)
- [Kubernetesをやってみる - Argo CDでHelm / Kustomize統合](https://zenn.dev/hono8944/articles/kubernetes-argocd-helm-kustomize)
