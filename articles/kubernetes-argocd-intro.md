---
title: "Kubernetesをやってみる - Argo CDでGitOpsを始める"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "argocd", "gitops", "devops"]
published: true
---

## はじめに

この記事では、Kubernetes 向けの GitOps 継続的デリバリーツールである **Argo CD** について、概念から基礎的なハンズオンまで解説します。

:::message
本記事では `kubectl` のエイリアスとして `k` を使用しています。
:::

## 前提知識

この記事を読む前に、以下を理解しておくと良いです：

- Kubernetes の基本（Pod, Service, Deployment, Namespace）
- Git の基本操作
- マニフェスト管理ツール（Helm, Kustomize）の基礎

## GitOps とは

### 従来の CD（継続的デリバリー）との違い

```
【従来の Push 型 CD】

  Developer        CI/CD Pipeline       Kubernetes
      │                  │                  │
      │  git push        │                  │
      │─────────────────→│                  │
      │                  │  k apply   │
      │                  │─────────────────→│
      │                  │                  │

問題点:
  ・CI/CD パイプラインにクラスタへのアクセス権限が必要
  ・実際のクラスタ状態と Git の状態がずれやすい
  ・手動でのデプロイも可能（一貫性の欠如）
```

```
【GitOps（Pull 型 CD）】

  Developer          Git             Argo CD         Kubernetes
      │               │                 │                │
      │  git push     │                 │                │
      │──────────────→│                 │                │
      │               │   監視・検知     │                │
      │               │←────────────────│                │
      │               │                 │   sync         │
      │               │                 │───────────────→│
      │               │                 │                │

メリット:
  ・Git が唯一の信頼できる情報源
  ・クラスタ状態と Git 状態が常に同期
  ・変更履歴は Git に完全に記録
  ・ロールバックは git revert で完了
```

### GitOps の原則

| 原則 | 説明 |
|------|------|
| **宣言的** | システムの望ましい状態を宣言的に記述（マニフェスト） |
| **バージョン管理** | 全ての変更を Git で管理し、履歴を保持 |
| **自動適用** | 承認された変更は自動的にシステムに適用 |
| **継続的調整** | ソフトウェアエージェントが望ましい状態との差分を検知・修正 |

## Argo CD とは

### 概要

Argo CD は、Kubernetes クラスタ上で動作する GitOps オペレーターです。

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Argo CD                                   │
│                                                                     │
│    ┌─────────────┐     ┌─────────────┐     ┌─────────────────────┐ │
│    │   Web UI    │     │     CLI     │     │       API           │ │
│    └──────┬──────┘     └──────┬──────┘     └──────────┬──────────┘ │
│           │                   │                       │            │
│           └───────────────────┼───────────────────────┘            │
│                               ▼                                    │
│    ┌─────────────────────────────────────────────────────────────┐ │
│    │                    Argo CD Server                           │ │
│    │  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐ │ │
│    │  │ API Server │  │ Repository │  │   Application          │ │ │
│    │  │            │  │   Server   │  │   Controller           │ │ │
│    │  └────────────┘  └─────┬──────┘  └──────────┬─────────────┘ │ │
│    └────────────────────────┼────────────────────┼───────────────┘ │
│                             │                    │                 │
└─────────────────────────────┼────────────────────┼─────────────────┘
                              │                    │
                              ▼                    ▼
                      ┌──────────────┐    ┌──────────────────┐
                      │     Git      │    │   Kubernetes     │
                      │  Repository  │    │    Cluster       │
                      └──────────────┘    └──────────────────┘
```

### 主要コンポーネント

| コンポーネント | 役割 |
|--------------|------|
| **API Server** | Web UI、CLI、CI/CD システムからの API リクエストを処理 |
| **Repository Server** | Git リポジトリのクローン、マニフェストの生成を担当 |
| **Application Controller** | アプリケーションの状態を監視し、望ましい状態との差分を検知・同期 |
| **Dex** | 外部認証プロバイダー（OIDC、LDAP 等）との連携 |
| **Redis** | キャッシュと状態の保持 |

### 対応するマニフェスト形式

Argo CD は様々なマニフェスト形式をサポートしています：

- 素の Kubernetes マニフェスト（YAML/JSON）
- Helm Chart
- Kustomize
- Jsonnet
- 任意のカスタムツール（Config Management Plugin）

## Argo CD の主要概念

### Application

**Application** は Argo CD の中核となるリソースで、以下を定義します：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default          # 所属するプロジェクト

  source:                   # マニフェストの取得元
    repoURL: https://github.com/example/my-repo.git
    targetRevision: main
    path: manifests/

  destination:              # デプロイ先
    server: https://kubernetes.default.svc
    namespace: my-app

  syncPolicy:               # 同期ポリシー
    automated:              # 自動同期を有効化
      prune: true           # Git から削除されたリソースを削除
      selfHeal: true        # 手動変更を自動で元に戻す
```

### Application の状態

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Application Status                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Sync Status（Git との同期状態）                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                          │
│  │  Synced  │  │OutOfSync │  │ Unknown  │                          │
│  │ (同期済) │  │(未同期)  │  │  (不明)  │                          │
│  └──────────┘  └──────────┘  └──────────┘                          │
│                                                                     │
│  Health Status（アプリケーションの健全性）                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ Healthy  │  │Progressing│ │ Degraded │  │ Missing  │           │
│  │  (正常)  │  │(処理中)  │  │ (劣化)   │  │  (欠落)  │           │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

| 状態 | 説明 |
|------|------|
| **Synced** | Git の状態とクラスタの状態が一致 |
| **OutOfSync** | Git の状態とクラスタの状態が不一致 |
| **Healthy** | 全てのリソースが正常に動作 |
| **Progressing** | リソースがまだ起動中 |
| **Degraded** | 一部のリソースに問題あり |
| **Missing** | リソースがクラスタに存在しない |

### Project

**Project** は、複数の Application をグループ化し、アクセス制御を行うためのリソースです。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-project
  namespace: argocd
spec:
  description: "My team's project"

  # 許可する Git リポジトリ
  sourceRepos:
    - https://github.com/my-org/*

  # 許可するデプロイ先
  destinations:
    - namespace: 'my-*'
      server: https://kubernetes.default.svc

  # 許可するクラスタリソース
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
```

### Sync と Sync Policy

| 設定 | 説明 |
|------|------|
| **Manual Sync** | ユーザーが明示的に同期を実行 |
| **Automated Sync** | Git の変更を自動検知して同期 |
| **Prune** | Git から削除されたリソースをクラスタからも削除 |
| **Self Heal** | クラスタ上で手動変更があっても Git の状態に戻す |

```
【Self Heal の動作】

  Git          Argo CD        Kubernetes
   │              │               │
   │   replicas:3 │               │
   ├─────────────→│   sync        │
   │              │──────────────→│ replicas: 3
   │              │               │
   │              │               │ 誰かが k で
   │              │               │ replicas: 5 に変更
   │              │               │
   │              │   監視・検知   │
   │              │←──────────────│
   │              │               │
   │              │   self heal   │
   │              │──────────────→│ replicas: 3 に戻す
   │              │               │
```

## アーキテクチャ詳細

### デプロイのフロー

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          Argo CD Workflow                                │
│                                                                          │
│  1. Git Push                                                             │
│     ┌────────┐     ┌─────────────┐                                      │
│     │Developer├────→│ Git Repo    │                                      │
│     └────────┘     └──────┬──────┘                                      │
│                           │                                              │
│  2. Repository Server が変更を検知                                        │
│                           ▼                                              │
│     ┌─────────────────────────────────────────┐                         │
│     │           Repository Server             │                         │
│     │  ・Git clone                            │                         │
│     │  ・Helm template / Kustomize build      │                         │
│     │  ・マニフェスト生成                       │                         │
│     └─────────────────────┬───────────────────┘                         │
│                           │                                              │
│  3. Application Controller が差分を検知                                   │
│                           ▼                                              │
│     ┌─────────────────────────────────────────┐                         │
│     │        Application Controller           │                         │
│     │  ・望ましい状態と実際の状態を比較          │                         │
│     │  ・Sync Status, Health Status を更新     │                         │
│     └─────────────────────┬───────────────────┘                         │
│                           │                                              │
│  4. Sync（手動または自動）                                                │
│                           ▼                                              │
│     ┌─────────────────────────────────────────┐                         │
│     │            Kubernetes Cluster           │                         │
│     │  ・k apply 相当の処理              │                         │
│     │  ・リソースの作成/更新/削除              │                         │
│     └─────────────────────────────────────────┘                         │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### マルチクラスタ対応

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│                          ┌──────────────┐                              │
│                          │   Argo CD    │                              │
│                          │  (管理クラスタ)│                              │
│                          └──────┬───────┘                              │
│                                 │                                      │
│            ┌────────────────────┼────────────────────┐                │
│            │                    │                    │                │
│            ▼                    ▼                    ▼                │
│    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│    │  開発クラスタ  │    │ ステージング  │    │  本番クラスタ │          │
│    │              │    │   クラスタ    │    │              │          │
│    └──────────────┘    └──────────────┘    └──────────────┘          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

1つの Argo CD インスタンスで複数のクラスタを管理できます。

---

## ハンズオン編

ここからは実際に手を動かして Argo CD の動作を確認します。

### やること

| Step | 内容 |
|------|------|
| Step 1 | Argo CD をインストールする |
| Step 2 | Argo CD にログインする |
| Step 3 | Application を作成してデプロイする |
| Step 4 | 自動同期を試す |
| Step 5 | クリーンアップ |

### 事前準備

```bash
# kind クラスタが起動していることを確認
k cluster-info
```

---

## Step 1: Argo CD をインストールする

### Namespace を作成してインストール

```bash
# argocd namespace を作成
k create namespace argocd

# Argo CD をインストール（公式マニフェスト）
k apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Pod が起動するまで待機

```bash
k wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

### インストールされたリソースを確認

```bash
k get all -n argocd
```

```
NAME                                                   READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0                    1/1     Running   0          2m
pod/argocd-applicationset-controller-xxx               1/1     Running   0          2m
pod/argocd-dex-server-xxx                              1/1     Running   0          2m
pod/argocd-notifications-controller-xxx                1/1     Running   0          2m
pod/argocd-redis-xxx                                   1/1     Running   0          2m
pod/argocd-repo-server-xxx                             1/1     Running   0          2m
pod/argocd-server-xxx                                  1/1     Running   0          2m
```

### Argo CD CLI をインストール

```bash
# macOS (Homebrew)
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

```bash
# バージョン確認
argocd version --client
```

---

## Step 2: Argo CD にログインする

### Web UI にアクセスする

```bash
# argocd-server を port-forward
k port-forward svc/argocd-server -n argocd 8080:443
```

ブラウザで https://localhost:8080 を開きます（証明書の警告は無視して進む）。

### 初期パスワードを取得

```bash
# admin ユーザーの初期パスワードを取得
argocd admin initial-password -n argocd
```

```
<パスワードが表示される>

This password must be only used for first time login. ...
```

### ログイン

**Web UI の場合:**
- Username: `admin`
- Password: 上記で取得したパスワード

**CLI の場合:**

```bash
# CLI でログイン
argocd login localhost:8080 --username admin --password <パスワード> --insecure
```

### パスワードを変更（推奨）

```bash
argocd account update-password
```

---

## Step 3: Application を作成してデプロイする

### サンプルアプリケーションのリポジトリ

Argo CD の公式サンプルリポジトリを使います：

- リポジトリ: `https://github.com/argoproj/argocd-example-apps`
- パス: `guestbook`（シンプルな Web アプリ）

### Application を CLI で作成

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

このコマンドは Argo CD に「どこからマニフェストを取得し、どこにデプロイするか」を教えています。

#### 各オプションの詳細説明

```
argocd app create guestbook
                  ─────────
                      │
                      └─ Application の名前
                         Argo CD 内でこのアプリを識別する名前
                         (k8s リソース名とは別)
```

```
--repo https://github.com/argoproj/argocd-example-apps.git
       ───────────────────────────────────────────────────
                              │
                              └─ マニフェストが置いてある Git リポジトリの URL
                                 Argo CD はここを監視して変更を検知する
```

```
--path guestbook
       ─────────
           │
           └─ リポジトリ内のディレクトリパス
              このディレクトリ内のマニフェストがデプロイ対象
```

```
--dest-server https://kubernetes.default.svc
              ──────────────────────────────
                          │
                          └─ デプロイ先の Kubernetes クラスタ
                             "https://kubernetes.default.svc" は
                             Argo CD が動いている同じクラスタを指す
```

```
--dest-namespace default
                 ───────
                    │
                    └─ デプロイ先の Namespace
                       マニフェストで namespace が指定されていない
                       リソースはここにデプロイされる
```

#### リポジトリの構造と対応関係

```
argocd-example-apps.git （--repo で指定）
│
├── guestbook/            ← --path で指定したディレクトリ
│   ├── guestbook-ui-deployment.yaml   ─┐
│   └── guestbook-ui-svc.yaml          ─┴─ これらがデプロイされる
│
├── helm-guestbook/       ← 別の --path を指定すれば
│   ├── Chart.yaml           こちらをデプロイできる
│   └── ...
│
└── kustomize-guestbook/  ← Kustomize 形式の例
    ├── kustomization.yaml
    └── ...
```

#### 実際にデプロイされるマニフェストの中身

`guestbook/` ディレクトリには以下の2つのマニフェストがあります：

**guestbook-ui-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: guestbook-ui
  template:
    metadata:
      labels:
        app: guestbook-ui
    spec:
      containers:
      - image: gcr.io/heptio-images/ks-guestbook-demo:0.2
        name: guestbook-ui
        ports:
        - containerPort: 80
```

**guestbook-ui-svc.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: guestbook-ui
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: guestbook-ui
```

#### 図解: Application が繋ぐもの

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         argocd app create                               │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      Application "guestbook"                    │   │
│   │                                                                 │   │
│   │   Source（取得元）              Destination（デプロイ先）        │   │
│   │   ┌─────────────────────┐      ┌─────────────────────────────┐ │   │
│   │   │ --repo              │      │ --dest-server               │ │   │
│   │   │ Git リポジトリ URL   │ ───→ │ Kubernetes API Server URL   │ │   │
│   │   │                     │      │                             │ │   │
│   │   │ --path              │      │ --dest-namespace            │ │   │
│   │   │ ディレクトリパス     │      │ Namespace 名                │ │   │
│   │   └─────────────────────┘      └─────────────────────────────┘ │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

        │                                      │
        ▼                                      ▼

┌─────────────────────┐              ┌─────────────────────────────┐
│ GitHub              │              │ Kubernetes Cluster          │
│                     │              │                             │
│ argocd-example-apps │              │  Namespace: default         │
│ └── guestbook/      │   同期       │  ┌─────────────────────┐    │
│     ├── deployment  │ ─────────→   │  │ Deployment          │    │
│     └── service     │              │  │ guestbook-ui        │    │
│                     │              │  ├─────────────────────┤    │
└─────────────────────┘              │  │ Service             │    │
                                     │  │ guestbook-ui        │    │
                                     │  └─────────────────────┘    │
                                     └─────────────────────────────┘
```

#### よく使う追加オプション

| オプション | 説明 | 例 |
|-----------|------|-----|
| `--revision` | 特定のブランチ/タグ/コミットを指定 | `--revision main` |
| `--project` | 所属する Project を指定 | `--project my-project` |
| `--sync-policy` | 同期ポリシーを設定 | `--sync-policy automated` |
| `--helm-set` | Helm の values を上書き | `--helm-set replicas=3` |

#### 作成される Application リソース（YAML 形式）

上記のコマンドは、内部的に以下の Kubernetes リソースを作成します：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook           # argocd app create の引数
  namespace: argocd         # Application は argocd namespace に作成される
spec:
  project: default          # デフォルトの Project

  source:                   # --repo, --path に対応
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    path: guestbook
    targetRevision: HEAD    # デフォルトは HEAD（最新）

  destination:              # --dest-server, --dest-namespace に対応
    server: https://kubernetes.default.svc
    namespace: default
```

#### 実際の運用では CLI を使わない

ハンズオンでは `argocd app create` コマンドを使いましたが、**実際の運用現場では CLI で手動作成することはほとんどありません**。

理由は単純で、「Application の設定も Git で管理したい」からです。

```
【ハンズオン（学習用）】

  運用者 ──→ argocd app create ──→ Argo CD ──→ Kubernetes
                  ↑
                  手動実行
                  設定が Git に残らない


【実際の運用】

  運用者 ──→ Git push ──→ Git リポジトリ ──→ Argo CD ──→ Kubernetes
                              ↑
                              Application の YAML も
                              Git で管理される
```

**運用で使われるパターン:**

| パターン | 説明 |
|---------|------|
| **YAML を kubectl apply** | Application マニフェストを Git 管理し、`k apply -f` でデプロイ |
| **App of Apps** | Application を管理する親 Application を作成（後述） |
| **ApplicationSet** | 複数 Application をテンプレートから自動生成 |

##### App of Apps パターン

「Application を管理する Application」を作成するパターンです。

```
apps-repo/                          ← このリポジトリを Argo CD が監視
├── apps/                           ← 親 Application が見るディレクトリ
│   ├── guestbook.yaml              ← 子 Application の定義
│   ├── prometheus.yaml
│   └── nginx.yaml
└── app-of-apps.yaml                ← 親 Application（最初だけ手動 apply）
```

**app-of-apps.yaml（親 Application）:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/apps-repo.git
    path: apps          # ← apps/ ディレクトリを監視
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd   # ← Application リソースは argocd namespace に作成
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

**apps/guestbook.yaml（子 Application）:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    path: guestbook
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      selfHeal: true
```

```
【App of Apps の動作】

1. 運用者が app-of-apps.yaml を一度だけ apply
   k apply -f app-of-apps.yaml

2. Argo CD が apps/ ディレクトリを監視

3. apps/ に新しい Application YAML を追加（Git push）
   └─→ Argo CD が検知して Application を自動作成

4. 各 Application がそれぞれのアプリをデプロイ

┌─────────────────────────────────────────────────────────────────┐
│ Argo CD                                                         │
│                                                                 │
│  ┌──────────────┐                                              │
│  │ app-of-apps  │ ← apps/ ディレクトリを監視                    │
│  └──────┬───────┘                                              │
│         │ 作成・管理                                            │
│         ▼                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐               │
│  │ guestbook  │  │ prometheus │  │   nginx    │  ← 子 App     │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘               │
│        │               │               │                       │
└────────┼───────────────┼───────────────┼───────────────────────┘
         ▼               ▼               ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │guestbook │   │prometheus│   │  nginx   │  ← 実際のワークロード
   │  Pods    │   │  Pods    │   │  Pods    │
   └──────────┘   └──────────┘   └──────────┘
```

このパターンのメリット:

- **全ての設定が Git に残る** - 誰がいつ何を変更したか追跡可能
- **PR ベースのワークフロー** - Application の追加/変更をレビューできる
- **自動化** - Git push だけで新しいアプリをデプロイ可能
- **再現性** - クラスタを作り直しても app-of-apps を apply するだけで復元

### Application の状態を確認

```bash
argocd app get guestbook
```

```
Name:               argocd/guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://localhost:8080/applications/guestbook
Repo:               https://github.com/argoproj/argocd-example-apps.git
Target:
Path:               guestbook
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        OutOfSync from  (53e28ff)  ← まだ同期されていない
Health Status:      Missing                     ← リソースがない
```

### 手動で Sync を実行

```bash
argocd app sync guestbook
```

または Web UI で「SYNC」ボタンをクリック。

### 同期後の状態を確認

```bash
argocd app get guestbook
```

```
Sync Status:        Synced to  (53e28ff)  ← 同期済み
Health Status:      Healthy               ← 正常
```

```bash
# Kubernetes リソースを確認
k get all -l app=guestbook-ui
```

```
NAME                               READY   STATUS    RESTARTS   AGE
pod/guestbook-ui-xxx               1/1     Running   0          30s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/guestbook-ui   ClusterIP   10.96.xxx.xx   <none>        80/TCP    30s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/guestbook-ui   1/1     1            1           30s
```

### YAML で Application を作成する場合

CLI の代わりに YAML マニフェストで Application を作成することもできます。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

```bash
k apply -f guestbook-app.yaml
```

---

## Step 4: 自動同期を試す

### 自動同期と Self Heal を有効化

```bash
# 自動同期を有効化
argocd app set guestbook --sync-policy automated

# Self Heal を有効化（手動変更を自動で元に戻す）
argocd app set guestbook --self-heal

# Prune を有効化（Git から削除されたリソースをクラスタからも削除）
argocd app set guestbook --auto-prune
```

または YAML で設定:

```yaml
spec:
  syncPolicy:
    automated:
      prune: true      # Git から削除されたリソースをクラスタからも削除
      selfHeal: true   # 手動変更を自動で Git の状態に戻す
```

設定を確認:

```bash
argocd app get guestbook -o json | jq '.spec.syncPolicy'
```

```json
{
  "automated": {
    "prune": true,
    "selfHeal": true
  }
}
```

### Self Heal を確認

手動でレプリカ数を変更してみます。

```bash
# レプリカ数を 3 に変更
k scale deployment guestbook-ui --replicas=3

# すぐに確認（3つに増えている）
k get pods -l app=guestbook-ui
```

Self Heal が有効な場合、数秒〜十数秒で Argo CD が自動的に元の状態（1 レプリカ）に戻します。

```bash
# 少し待ってから確認（1つに戻っている）
k get pods -l app=guestbook-ui
```

### Application の詳細を Web UI で確認

https://localhost:8080 で guestbook Application を開くと：

- リソースの依存関係がグラフで可視化される
- 各リソースの状態（Sync Status, Health）が確認できる
- ログやイベントも確認可能

---

## Step 5: クリーンアップ

### Application を削除

```bash
# Application を削除（関連する Kubernetes リソースも削除される）
argocd app delete guestbook --cascade
```

`--cascade` オプションにより、Application が管理していた Deployment, Service なども削除されます。

### Argo CD をアンインストール

```bash
k delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
k delete namespace argocd
```

---

## よく使うコマンドまとめ

```bash
# ログイン
argocd login <server> --username admin --password <password>

# Application 管理
argocd app create <name> --repo <url> --path <path> --dest-server <server> --dest-namespace <ns>
argocd app list
argocd app get <name>
argocd app sync <name>
argocd app delete <name>

# 同期ポリシー設定
argocd app set <name> --sync-policy automated
argocd app set <name> --self-heal
argocd app set <name> --auto-prune

# 履歴とロールバック
argocd app history <name>
argocd app rollback <name> <revision>

# クラスタ管理
argocd cluster add <context-name>
argocd cluster list

# リポジトリ管理
argocd repo add <url> --username <user> --password <password>
argocd repo list
```

---

## 発展的なトピック

### Helm Chart のデプロイ

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-helm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: nginx
    targetRevision: 15.0.0
    helm:
      values: |
        replicaCount: 2
        service:
          type: ClusterIP
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

### Kustomize のデプロイ

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-kustomize-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/my-repo.git
    path: overlays/production
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

### ApplicationSet

複数の Application を一括で管理するためのリソースです。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-apps
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: dev
            namespace: dev
          - cluster: staging
            namespace: staging
          - cluster: prod
            namespace: prod
  template:
    metadata:
      name: 'my-app-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/example/my-repo.git
        path: 'overlays/{{cluster}}'
        targetRevision: main
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
```

---

## トラブルシューティング

### よくある問題と対処法

| 問題 | 原因 | 対処法 |
|------|------|--------|
| Sync が OutOfSync のまま | Git との差分がある | `argocd app diff <name>` で差分を確認 |
| Health が Degraded | Pod が起動失敗 | `k describe pod` でエラーを確認 |
| リポジトリに接続できない | 認証エラー | `argocd repo add` で認証情報を設定 |
| 同期が遅い | キャッシュが古い | `argocd app get <name> --hard-refresh` |

### デバッグ用コマンド

```bash
# Application の詳細ログ
argocd app logs <name>

# Application のマニフェストを表示
argocd app manifests <name>

# Git と Kubernetes の差分を表示
argocd app diff <name>

# Argo CD サーバーのログ
k logs -n argocd deployment/argocd-server
k logs -n argocd deployment/argocd-repo-server
k logs -n argocd statefulset/argocd-application-controller
```

---

## 参考リンク

- [Argo CD 公式ドキュメント](https://argo-cd.readthedocs.io/)
- [Argo CD GitHub リポジトリ](https://github.com/argoproj/argo-cd)
- [GitOps とは - Weaveworks](https://www.weave.works/technologies/gitops/)
- [Argo CD Example Apps](https://github.com/argoproj/argocd-example-apps)

## 関連記事

- [Kubernetesをやってみる - Argo CDで自分のGitHubリポジトリから自動デプロイ](/articles/kubernetes-argocd-github-deploy)
- [Kubernetesをやってみる - Argo CDでプライベートリポジトリからデプロイ](/articles/kubernetes-argocd-private-repo)
- [Kubernetesをやってみる - Argo CDのApp of Appsパターン](/articles/kubernetes-argocd-app-of-apps)
- [Kubernetesをやってみる - Argo CDでHelm / Kustomize統合](/articles/kubernetes-argocd-helm-kustomize)
