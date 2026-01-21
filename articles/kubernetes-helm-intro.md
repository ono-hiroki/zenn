---
title: "Kubernetesをやってみる - Helmでパッケージ管理する"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "helm", "container", "devops"]
published: true
---

## はじめに

この記事では、Kubernetes のパッケージマネージャーである **Helm** について、概念から実践的なハンズオンまで解説します。

Helm を理解することで、複雑なアプリケーションのデプロイ・管理が格段に楽になります。

:::message
本記事では `kubectl` のエイリアスとして `k` を使用しています。
:::

## Helm とは

Helm は、**Kubernetes のパッケージマネージャー** です。

複数の Kubernetes マニフェストをまとめて「Chart（チャート）」という単位で管理し、アプリケーションのインストール・アップグレード・削除を簡単に行えます。

```
┌─────────────────────────────────────────────────────────────────────┐
│  Helm Chart                                                         │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  templates/                                                    │ │
│  │    deployment.yaml   ─┐                                        │ │
│  │    service.yaml      ─┼─→  1つの helm install で全てデプロイ   │ │
│  │    configmap.yaml    ─┤                                        │ │
│  │    ingress.yaml      ─┘                                        │ │
│  └────────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────┐  ┌──────────────────────────────────────┐  │
│  │  Chart.yaml         │  │  values.yaml                         │  │
│  │  (メタ情報)          │  │  (設定値)                            │  │
│  └─────────────────────┘  └──────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

## なぜ Helm を使うのか

### 1. 複数マニフェストの一括管理

通常、アプリケーションをデプロイするには複数のマニフェストが必要です。

```bash
# Helm なし: 個別に apply する必要がある
k apply -f deployment.yaml
k apply -f service.yaml
k apply -f configmap.yaml
k apply -f ingress.yaml
```

```bash
# Helm あり: 1コマンドで全てデプロイ
helm install my-app ./my-chart
```

### 2. 環境ごとの設定切り替え

`values.yaml` で設定値を外出しにすることで、同じ Chart を使いながら環境ごとに異なる設定を適用できます。

```bash
# 開発環境
helm install my-app ./my-chart -f values-dev.yaml

# 本番環境
helm install my-app ./my-chart -f values-prod.yaml
```

### 3. バージョン管理とロールバック

Helm はデプロイ履歴を保持し、問題があれば簡単にロールバックできます。

```bash
# デプロイ履歴を確認
helm history my-app

# 1つ前のバージョンに戻す
helm rollback my-app
```

### 4. 公開 Chart の再利用

Artifact Hub などで公開されている Chart を使えば、複雑なアプリケーション（MySQL、Redis、Prometheus など）を簡単にデプロイできます。

```bash
# 例: Prometheus をインストール
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus
```

## いつ Helm を使うのか

### Helm を使うと便利な場面

| 場面 | 説明 |
|------|------|
| 環境ごとに設定を変えたい | dev / staging / prod で values ファイルを切り替え |
| 公開されているソフトウェアを使いたい | MySQL、Redis、Prometheus などを1コマンドで構築 |
| マニフェストが多くて管理が大変 | Deployment + Service + ConfigMap + ... をまとめて管理 |
| ロールバックしたい | 問題があればすぐ前のバージョンに戻せる |
| チームで構成を共有したい | Chart を共有してデプロイ方法を標準化 |

### Helm を使わなくてもいい場面

| 場面 | 理由 |
|------|------|
| マニフェストが数個だけ | `kubectl apply -f` で十分 |
| 環境差分がほとんどない | テンプレート化するメリットが薄い |
| 学習目的で素の Kubernetes を理解したい | まずは生のマニフェストを書く経験が大事 |

### 判断の目安

```
マニフェストが 3〜4 個以上ある
    → Helm を検討

環境（dev/prod）で設定値が異なる
    → Helm を検討

MySQL や Prometheus など有名なソフトを使う
    → 公開 Chart を使う

シンプルな Pod や Deployment を1つ動かすだけ
    → kubectl apply で OK
```

## Helm の主要概念

| 用語 | 説明 |
|------|------|
| **Chart** | Kubernetes リソースをパッケージ化したもの。テンプレートと設定値のセット |
| **Release** | Chart をクラスタにインストールした「インスタンス」。同じ Chart から複数の Release を作成可能 |
| **Repository** | Chart を公開・共有するための場所 |
| **values.yaml** | Chart の設定値を定義するファイル |

```
        Chart                      Release
    ┌──────────────┐          ┌──────────────┐
    │ nginx-chart  │  ──→     │ web-frontend │  (本番用)
    │              │          └──────────────┘
    │ テンプレート    │  ──→     ┌──────────────┐
    │ + デフォルト値  │          │ web-staging  │  (ステージング用)
    └──────────────┘          └──────────────┘
```

## Chart の構造

```
my-chart/
├── Chart.yaml          # Chart のメタ情報（名前、バージョン、説明）
├── values.yaml         # デフォルトの設定値
├── templates/          # Kubernetes マニフェストのテンプレート
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # テンプレート用のヘルパー関数
│   └── NOTES.txt       # インストール後に表示されるメッセージ
└── charts/             # 依存する Chart（サブチャート）
```

### Chart.yaml の例

```yaml
apiVersion: v2
name: my-app
description: A Helm chart for my application
type: application
version: 0.1.0        # Chart 自体のバージョン
appVersion: "1.0.0"   # アプリケーションのバージョン
```

### values.yaml の例

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.24"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
```

### テンプレートでの値の参照

`templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

| 変数 | 説明 |
|------|------|
| `.Values.*` | values.yaml の値を参照 |
| `.Release.Name` | Release の名前 |
| `.Release.Namespace` | インストール先の Namespace |
| `.Chart.Name` | Chart の名前 |
| `.Chart.Version` | Chart のバージョン |

## よく使うコマンド

| コマンド | 説明 |
|----------|------|
| `helm install <name> <chart>` | Chart をインストール |
| `helm upgrade <name> <chart>` | Release をアップグレード |
| `helm uninstall <name>` | Release を削除 |
| `helm list` | インストール済み Release を一覧表示 |
| `helm rollback <name> [revision]` | 以前のバージョンにロールバック |
| `helm template <chart>` | テンプレートをレンダリング（実際にはインストールしない） |
| `helm repo add <name> <url>` | Repository を追加 |
| `helm search repo <keyword>` | Repository から Chart を検索 |

## 参考リンク

- [Helm | The package manager for Kubernetes](https://helm.sh/)
- [Helm Docs](https://helm.sh/docs/)
- [Artifact Hub](https://artifacthub.io/)

## ハンズオン

ここからは実際に手を動かして Helm の動作を確認します。

### やること

| Step | 内容 |
|------|------|
| Step 1 | Helm をインストールする |
| Step 2 | 公開 Chart を使ってみる（nginx） |
| Step 3 | 自作 Chart を作成する |
| Step 4 | 自作 Chart をデプロイ・アップグレード・ロールバック |
| Step 5 | クリーンアップ |

### 事前準備

```bash
# kind クラスタが起動していることを確認
k cluster-info
```

## Step 1: Helm をインストールする

### macOS（Homebrew）

```bash
brew install helm
```

### その他の OS

```bash
# スクリプトでインストール
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 確認

```bash
helm version
```

```
version.BuildInfo{Version:"v3.x.x", ...}
```

## Step 2: 公開 Chart を使ってみる

まずは公開されている Chart（パッケージ）を使って、Helm の基本的な使い方を体験します。

### 2-1. Repository を追加する

**Repository** は、Chart（パッケージ）を配布している場所です。npm の registry、Homebrew の tap のようなものです。

ここでは、多くの有名ソフトウェアの Chart を公開している **Bitnami** の Repository を追加します。

```bash
# Bitnami の Repository を "bitnami" という名前で登録
helm repo add bitnami https://charts.bitnami.com/bitnami

# Repository の情報を最新化（apt update のようなもの）
helm repo update
```

```
"bitnami" has been added to your repositories
...
Update Complete. ⎈Happy Helming!⎈
```

登録した Repository を確認してみましょう。

```bash
helm repo list
```

```
NAME    URL
bitnami https://charts.bitnami.com/bitnami
```

### 2-2. Chart を検索する

**Chart** は、Kubernetes アプリケーションをパッケージ化したものです。Deployment、Service、ConfigMap などのマニフェストがまとめて入っています。

登録した Repository から nginx の Chart を検索してみましょう。

```bash
helm search repo nginx
```

```
NAME                  CHART VERSION   APP VERSION   DESCRIPTION
bitnami/nginx         ...             ...           NGINX Open Source is a web server...
```

`bitnami/nginx` という Chart が見つかりました。これが nginx をデプロイするためのパッケージです。

### 2-3. Chart の設定値を確認する

Chart には設定値（values）があり、レプリカ数やイメージタグなどをカスタマイズできます。

どんな設定ができるか確認してみましょう。

```bash
# bitnami/nginx Chart の設定値を表示（先頭50行）
helm show values bitnami/nginx | head -50
```

たくさんの設定項目があることがわかります。

### 2-4. Chart をインストールする

いよいよ Chart をクラスタにインストールします。

**Release** は、Chart をインストールした「実体」です。同じ Chart から複数の Release を作成できます（本番用、ステージング用など）。

```bash
# bitnami/nginx Chart を "my-nginx" という名前の Release としてインストール
helm install my-nginx bitnami/nginx --set service.type=ClusterIP
```

```
NAME: my-nginx                    ← Release 名
LAST DEPLOYED: ...
NAMESPACE: default
STATUS: deployed                  ← デプロイ成功
REVISION: 1                       ← リビジョン番号（更新のたびに増える）
```

これだけで nginx がデプロイされました。

### 2-5. 作成されたリソースを確認する

```bash
# インストール済みの Release 一覧
helm list
```

```
NAME      NAMESPACE  REVISION  UPDATED                   STATUS    CHART          APP VERSION
my-nginx  default    1         ...                       deployed  nginx-x.x.x    x.x.x
```

```bash
# この Release が作成した Kubernetes リソースを確認
k get all -l app.kubernetes.io/instance=my-nginx
```

Deployment、Pod、Service などが自動的に作成されていることがわかります。

### 2-6. Release をアンインストールする

```bash
# my-nginx という Release を削除
helm uninstall my-nginx
```

```
release "my-nginx" uninstalled
```

関連する Kubernetes リソースもすべて削除されます。

## Step 3: 自作 Chart を作成する

### Chart のひな形を生成

```bash
helm create my-app
```

以下の構造が生成されます:

```
my-app/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   └── tests/
└── charts/
```

### Chart.yaml を編集

`my-app/Chart.yaml`:

```yaml
apiVersion: v2
name: my-app
description: A simple Helm chart for learning
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### values.yaml を編集

`my-app/values.yaml` を以下の内容に置き換えます（シンプルにするため）:

```yaml
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "stable-alpine"

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

### テンプレートを確認（ドライラン）

実際にインストールせずに、生成されるマニフェストを確認できます。

```bash
helm template my-app ./my-app
```

### Lint で検証

```bash
helm lint ./my-app
```

```
==> Linting ./my-app
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

## Step 4: 自作 Chart をデプロイ・アップグレード・ロールバック

### インストール

```bash
helm install my-app ./my-app
```

```
NAME: my-app
LAST DEPLOYED: ...
NAMESPACE: default
STATUS: deployed
REVISION: 1
```

### 確認

```bash
helm list
k get all -l app.kubernetes.io/instance=my-app
```

### アップグレード（レプリカ数を変更）

```bash
helm upgrade my-app ./my-app --set replicaCount=3
```

```bash
# 変更を確認
k get pods -l app.kubernetes.io/instance=my-app
```

```
NAME                      READY   STATUS    RESTARTS   AGE
my-app-xxx-xxx            1/1     Running   0          10s
my-app-xxx-yyy            1/1     Running   0          10s
my-app-xxx-zzz            1/1     Running   0          10s
```

### 履歴を確認

```bash
helm history my-app
```

```
REVISION    UPDATED                     STATUS      CHART           APP VERSION    DESCRIPTION
1           ...                         superseded  my-app-0.1.0    1.0.0          Install complete
2           ...                         deployed    my-app-0.1.0    1.0.0          Upgrade complete
```

### ロールバック

```bash
helm rollback my-app 1
```

```bash
# レプリカ数が 1 に戻っていることを確認
k get pods -l app.kubernetes.io/instance=my-app
```

### 履歴を再確認

```bash
helm history my-app
```

```
REVISION    UPDATED                     STATUS      CHART           APP VERSION    DESCRIPTION
1           ...                         superseded  my-app-0.1.0    1.0.0          Install complete
2           ...                         superseded  my-app-0.1.0    1.0.0          Upgrade complete
3           ...                         deployed    my-app-0.1.0    1.0.0          Rollback to 1
```

## Step 5: クリーンアップ

```bash
# Release を削除
helm uninstall my-app

# Repository を削除（オプション）
helm repo remove bitnami
```

## よく使うコマンドまとめ

```bash
# Repository 管理
helm repo add <name> <url>
helm repo update
helm repo list
helm repo remove <name>

# Chart 検索・情報
helm search repo <keyword>
helm show chart <chart>
helm show values <chart>

# インストール・アップグレード
helm install <release> <chart>
helm install <release> <chart> -f values.yaml
helm install <release> <chart> --set key=value
helm upgrade <release> <chart>
helm upgrade <release> <chart> --install  # なければインストール

# 確認
helm list
helm status <release>
helm history <release>

# ロールバック・削除
helm rollback <release> [revision]
helm uninstall <release>

# 開発・デバッグ
helm create <name>          # Chart のひな形を生成
helm template <chart>       # テンプレートをレンダリング
helm lint <chart>           # Chart の検証
helm install --dry-run --debug <release> <chart>  # ドライラン
```

## Tips

### values ファイルの優先順位

複数の方法で値を指定した場合、以下の優先順位で上書きされます（下が優先）:

1. Chart の `values.yaml`（デフォルト値）
2. `-f` で指定したファイル（複数指定可、後のものが優先）
3. `--set` で指定した値

```bash
# 例: デフォルト値 → values-prod.yaml → --set の順で上書き
helm install my-app ./my-app \
  -f values-prod.yaml \
  --set replicaCount=5
```

### Namespace を指定

```bash
helm install my-app ./my-app -n my-namespace --create-namespace
```

### Release の状態確認

```bash
helm status my-app
```

## まとめ

この記事では、Helm の基本概念とハンズオンを通じて以下を学びました。

- **Helm とは**: Kubernetes のパッケージマネージャー
- **Chart**: マニフェストをまとめたパッケージ
- **Release**: Chart をインストールした実体
- **values.yaml**: 設定値の外出し
- **基本操作**: install / upgrade / rollback / uninstall

Helm を活用することで、複数環境へのデプロイやバージョン管理が格段に楽になります。
