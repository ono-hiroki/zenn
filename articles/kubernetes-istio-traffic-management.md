---
title: "Istioをやってみる - VirtualServiceでトラフィック管理とCanary Deploymentを試す"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "istio", "canary", "servicemesh"]
published: true
---

## はじめに

この記事では、Istio の **VirtualService** と **DestinationRule** を使ってトラフィックルーティングを制御します。

特定バージョンへのルーティング固定や、Canary Deployment（重み付けルーティング）を体験します。

:::message
本記事では `kubectl` のエイリアスとして `k` を使用しています。
:::

## ソースコードの取得

マニフェストファイルを使用するため、リポジトリをクローンします。

```bash
git clone https://github.com/ono-hiroki/maitake.git
cd maitake/kubernetes/15-istio/03-traffic-management
```

## 前提条件

- Kubernetes クラスタが稼働していること
- Istio がインストール済みであること（[サイドカー注入編](https://zenn.dev/ono_hiroki/articles/kubernetes-istio-sidecar) を参照）

## VirtualService と DestinationRule

Istio のトラフィック管理は主に 2 つの CRD で行います。

| CRD | 役割 |
|-----|------|
| **DestinationRule** | サービスのバージョンを **subset** としてグループ化する |
| **VirtualService** | リクエストをどの subset にルーティングするか制御する |

```
リクエスト → VirtualService（どこに送る？）→ DestinationRule（どのPod群？）→ Pod
```

## 1. Bookinfo アプリをデプロイ

Bookinfo の reviews サービスには v1, v2, v3 の 3 つのバージョンがあります。

```
reviews-v1: 評価なし
reviews-v2: 黒い星の評価
reviews-v3: 赤い星の評価
```

```bash
k label namespace default istio-injection=enabled --overwrite

k apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/bookinfo/platform/kube/bookinfo.yaml
k get pods -w

# fortio（負荷テストツール）をデプロイ
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/httpbin/sample-client/fortio-deploy.yaml
k get pods -l app=fortio -w
```

## 2. DestinationRule で subset を定義

reviews サービスの 3 つのバージョンを subset として定義します。

```bash
k apply -f manifests/reviews-destination-rule.yaml
```

```yaml:manifests/reviews-destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3
```

Pod の `version` ラベルで v1 / v2 / v3 をグループ化しています。VirtualService からこの subset 名を参照してルーティング先を指定します。

## 3. VirtualService で v1 に固定

すべてのトラフィックを reviews-v1 にルーティングします。

```bash
k apply -f manifests/reviews-v1-only.yaml
```

```yaml:manifests/reviews-v1-only.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
```

### 確認

```bash
for i in {1..5}; do
  k exec deploy/fortio-deploy -c fortio -- \
    fortio curl -quiet http://productpage:9080/productpage 2>&1 \
    | grep -oE "reviews-v[0-9]+" | head -1
done
```

すべて `reviews-v1` と表示されれば成功です。

## 4. Canary Deployment（重み付けルーティング）

v1 に 80%、v3 に 20% のトラフィックを流します。

```bash
k apply -f manifests/reviews-canary.yaml
```

```yaml:manifests/reviews-canary.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 80
        - destination:
            host: reviews
            subset: v3
          weight: 20
```

### 確認

```bash
for i in {1..10}; do
  k exec deploy/fortio-deploy -c fortio -- \
    fortio curl -quiet http://productpage:9080/productpage 2>&1 \
    | grep -oE "reviews-v[0-9]+" | head -1
done
```

おおよそ 8:2 の割合で `reviews-v1` と `reviews-v3` が表示されます。

### Canary Deployment の活用

この仕組みを使うと、段階的なリリースが可能になります。

1. 新バージョンを **10%** だけリリース
2. 問題がなければ徐々に増やす（20% → 50% → 100%）
3. 問題があれば即座に **0%** に戻す

Deployment のローリングアップデートでは「全 Pod が新バージョンに切り替わる」のに対し、Istio の Canary Deployment では**新旧バージョンを同時に稼働させたままトラフィック比率を制御**できます。

## クリーンアップ

```bash
k delete virtualservice reviews
k delete destinationrule reviews
k delete -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/bookinfo/platform/kube/bookinfo.yaml
k delete -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/httpbin/sample-client/fortio-deploy.yaml
```

## まとめ

- **DestinationRule** で Pod のラベルをもとに subset（バージョン群）を定義する
- **VirtualService** でリクエストのルーティング先を制御する
- **Canary Deployment** で新バージョンへのトラフィックを段階的に増やせる

## 関連記事

- [Istio サイドカー注入編](https://zenn.dev/ono_hiroki/articles/kubernetes-istio-sidecar)
- [Istio オブザーバビリティ編](https://zenn.dev/ono_hiroki/articles/kubernetes-istio-observability)
- Istio サーキットブレーカー / Gateway 編

## ソースコード

https://github.com/ono-hiroki/maitake/tree/main/kubernetes/15-istio/03-traffic-management
