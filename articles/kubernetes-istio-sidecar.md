---
title: "Istioをやってみる - サイドカー注入でEnvoy経由の通信を確認する"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "istio", "envoy", "servicemesh"]
published: false
---

## はじめに

この記事では、Istio の最も基本的な機能である**サイドカー注入**を体験します。

Pod に Envoy プロキシが自動で注入され、アプリを変更せずにすべての通信が Envoy を経由するようになることを確認します。

:::message
本記事では `kubectl` のエイリアスとして `k` を使用しています。
:::

## Istio とは

Istio は、マイクロサービス間の通信を管理する**サービスメッシュ**の実装です。

マイクロサービスアーキテクチャでは、各サービスでリトライ、サーキットブレーカー、認証、メトリクス収集などを個別に実装する必要があります。Istio はこれらをインフラ層で提供し、アプリケーションコードから分離します。

```
┌──────────────────────────────────────────────────────────────┐
│  Istio Service Mesh                                          │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ Pod A           │  │ Pod B           │  │ Pod C        │ │
│  │ ┌─────┐ ┌─────┐ │  │ ┌─────┐ ┌─────┐ │  │ ┌─────┐┌───┐│ │
│  │ │App A│←│Envoy│←┼──┼→│Envoy│→│App B│ │──┼→│Envoy││App││ │
│  │ └─────┘ └─────┘ │  │ └─────┘ └─────┘ │  │ └─────┘└───┘│ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
│           ▲                   ▲                   ▲         │
│           └───────────────────┼───────────────────┘         │
│                               │                             │
│                        ┌──────┴──────┐                      │
│                        │   istiod    │ ← Control Plane      │
│                        └─────────────┘                      │
└──────────────────────────────────────────────────────────────┘
```

各 Pod にサイドカーとして Envoy プロキシが注入され、すべての通信を中継します。Control Plane（istiod）が各 Envoy の設定を一元管理します。

## 前提条件

- Kubernetes クラスタが稼働していること
- istioctl がインストール済みであること

```bash
# istioctl のインストール
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# demo プロファイルでインストール（学習用）
istioctl install --set profile=demo -y

# istiod が起動しているか確認
k get pods -n istio-system
```

## 1. nginx と curl をデプロイ（サイドカーなし）

まず、サイドカー注入を**有効化せずに**デプロイします。

```bash
# nginx サーバーをデプロイ
k run nginx --image=nginx --port=80
k expose pod nginx --port=80

# curl クライアントをデプロイ
k run curl --image=curlimages/curl --command -- sleep infinity

# Pod が起動するまで待つ
k get pods -w
```

`READY` 列が **`1/1`** になっていることを確認します。

```
NAME    READY   STATUS    RESTARTS   AGE
curl    1/1     Running   0          30s
nginx   1/1     Running   0          30s
```

**`1/1` = アプリコンテナのみ**（サイドカーなし）です。

### サイドカーなしでリクエスト

```bash
k exec curl -- curl -sI http://nginx | grep -E "^HTTP|^[Ss]erver:"
```

```
HTTP/1.1 200 OK
Server: nginx/1.x.x
```

`Server: nginx` = nginx が直接応答しています（Envoy を経由していない）。

## 2. サイドカー自動注入を有効化

Namespace にラベルを付けると、以降そこに作成される Pod に自動で Envoy サイドカーが注入されます。

```bash
k label namespace default istio-injection=enabled

# 確認
k get namespace -L istio-injection
```

:::message
ラベルは**新しく作成される Pod** に対して適用されます。既存の Pod には何も起きません。
:::

## 3. Pod を再作成してサイドカーを注入

既存の Pod を削除して再作成します。

```bash
k delete pod nginx curl

k run nginx --image=nginx --port=80
k run curl --image=curlimages/curl --command -- sleep infinity

k get pods -w
```

`READY` 列が **`2/2`** になることを確認します。

```
NAME    READY   STATUS    RESTARTS   AGE
curl    2/2     Running   0          30s
nginx   2/2     Running   0          30s
```

**`2/2` = アプリコンテナ + istio-proxy（Envoy サイドカー）** です。

## 4. サイドカー注入の確認

### コンテナの構成を確認

```bash
k get pod nginx -o custom-columns="\
INIT_CONTAINERS:.spec.initContainers[*].name,\
CONTAINERS:.spec.containers[*].name"
```

Istio 1.28 + Kubernetes 1.28 以降では、**ネイティブサイドカー**機能を使用しています。`istio-proxy` は `initContainers` に定義されていますが、`restartPolicy: Always` によりメインコンテナと一緒に動き続けます。

```bash
# istio-proxy の restartPolicy を確認
k get pod nginx -o jsonpath='{.spec.initContainers[?(@.name=="istio-proxy")].restartPolicy}'
# → Always
```

### サイドカー注入で何が起きたか

```
Before (1/1):
┌─────────────────────────────────────┐
│ Pod                                 │
│ spec.containers:                    │
│ ┌─────────────────────────────────┐ │
│ │ nginx (アプリ)                  │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘

After (2/2):
┌─────────────────────────────────────┐
│ Pod                                 │
│ spec.initContainers:                │
│ ┌─────────────────────────────────┐ │
│ │ istio-init (iptables設定後終了) │ │
│ ├─────────────────────────────────┤ │
│ │ istio-proxy (restartPolicy:     │ │
│ │   Always で継続実行)            │ │
│ └─────────────────────────────────┘ │
│ spec.containers:                    │
│ ┌─────────────────────────────────┐ │
│ │ nginx (アプリ)                  │ │
│ └─────────────────────────────────┘ │
│ iptables により全通信が             │
│ istio-proxy を経由                  │
└─────────────────────────────────────┘
```

アプリは何も知らずに通信していますが、iptables によってすべてのトラフィックが istio-proxy を経由します。

## 5. Envoy 経由の通信を確認

```bash
k exec curl -- curl -sI http://nginx | grep -E "^HTTP|^[Ss]erver:|^x-envoy"
```

```
HTTP/1.1 200 OK
server: envoy
x-envoy-upstream-service-time: 1
```

| 状態 | Server ヘッダー | 説明 |
|------|-----------------|------|
| サイドカーなし | `Server: nginx` | nginx が直接応答 |
| サイドカーあり | `server: envoy` | Envoy 経由で応答 |

nginx 自体は `Server: nginx` を返しますが、istio-proxy（Envoy）が間に入ることで `server: envoy` に書き換わります。

## クリーンアップ

```bash
k delete pod nginx curl
k delete svc nginx
```

## まとめ

- Namespace にラベルを付けると、Pod に Envoy サイドカーが自動注入される（`1/1` → `2/2`）
- アプリを変更せずに、すべての通信が Envoy を経由するようになる
- Envoy が通信を中継することで、トラフィック管理やオブザーバビリティなどの機能が使えるようになる

## 関連記事

- Istio オブザーバビリティ編（Kiali / Jaeger による可視化）
- Istio トラフィック管理編（VirtualService / Canary Deployment）
- Istio サーキットブレーカー / Gateway 編

## ソースコード

https://github.com/ono-hiroki/maitake/tree/main/kubernetes/15-istio
