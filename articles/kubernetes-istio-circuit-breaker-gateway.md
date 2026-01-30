---
title: "Istioをやってみる - サーキットブレーカーとGatewayで外部公開する"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "istio", "gateway", "servicemesh"]
published: false
---

## はじめに

この記事では、Istio の**サーキットブレーカー**と **Gateway** を体験します。

サーキットブレーカーで過負荷を防止し、Gateway で外部からメッシュ内のサービスにアクセスできるようにします。

:::message
本記事では `kubectl` のエイリアスとして `k` を使用しています。
:::

## ソースコードの取得

マニフェストファイルを使用するため、リポジトリをクローンします。

```bash
git clone https://github.com/ono-hiroki/maitake.git
cd maitake/kubernetes/15-istio/04-circuit-breaker-gateway
```

## 前提条件

- Kubernetes クラスタが稼働していること
- Istio がインストール済みであること（[サイドカー注入編](https://zenn.dev/ono_hiroki/articles/kubernetes-istio-sidecar) を参照）

## 準備

```bash
k label namespace default istio-injection=enabled --overwrite

# Bookinfo アプリをデプロイ
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/bookinfo/platform/kube/bookinfo.yaml
k get pods -w

# fortio（負荷テストツール）と httpbin をデプロイ
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/httpbin/sample-client/fortio-deploy.yaml
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/httpbin/httpbin.yaml
k get pods -l app=httpbin -w
```

## Part 1: サーキットブレーカー

### サーキットブレーカーとは

Istio のサーキットブレーカーは DestinationRule で設定し、2 つの仕組みで構成されます。

| 仕組み | 動作 |
|--------|------|
| **connectionPool** | 接続数の上限を設定。超えると即座に 503 を返す |
| **outlierDetection** | エラーが続く Pod を一時的にロードバランシング対象から除外する |

```
リクエスト → connectionPool チェック → 上限超え？ → 503
                                     ↓ OK
                               バックエンド Pod
                                     ↓ エラー多発
                               outlierDetection → Pod を一時除外
```

### 1. サーキットブレーカーを設定

接続数を厳しく制限して、サーキットブレーカーの動作を確認しやすくします。

```bash
k apply -f manifests/httpbin-circuit-breaker.yaml
```

```yaml:manifests/httpbin-circuit-breaker.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1           # TCP接続を1に制限
      http:
        http1MaxPendingRequests: 1  # 待機リクエストを1に制限
        maxRequestsPerConnection: 1 # 1接続1リクエスト
    outlierDetection:
      consecutive5xxErrors: 1       # 1回の5xxで除外
      interval: 1s
      baseEjectionTime: 30s
      maxEjectionPercent: 100
```

### 2. サーキットブレーカーの動作確認

同時接続数を超えるリクエストを送信します。

```bash
k exec deploy/fortio-deploy -c fortio -- \
  fortio load -c 3 -qps 0 -n 20 -loglevel Warning \
  http://httpbin:8000/get
```

結果の末尾に以下のような出力が表示されます:

```
Code 200 : 2 (10.0 %)   ← 成功
Code 503 : 18 (90.0 %)  ← サーキットブレーカーで拒否
```

`maxConnections: 1` に対して 3 並列（`-c 3`）でリクエストしたため、上限を超えたリクエストが即座に 503 で拒否されています。

:::message
本番環境では `maxConnections` や `http1MaxPendingRequests` をサービスの処理能力に合わせて調整します。ここではテストのために極端に小さい値を設定しています。
:::

## Part 2: Gateway で外部からアクセス

### Gateway とは

Gateway はメッシュの外部からのトラフィックを受け付ける入口です。Kubernetes の Ingress に相当しますが、Istio の VirtualService と組み合わせることで柔軟なルーティングが可能になります。

```
ブラウザ → Gateway (Ingress Gateway Pod) → VirtualService → バックエンド Pod
```

### 3. Gateway を設定

```bash
k apply -f manifests/bookinfo-gateway.yaml
```

```yaml:manifests/bookinfo-gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway     # Istio の Ingress Gateway Pod を使用
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - "*"
  gateways:
    - bookinfo-gateway        # この Gateway 経由のトラフィックに適用
  http:
    - match:
        - uri:
            exact: /productpage
        - uri:
            prefix: /static
        - uri:
            exact: /login
        - uri:
            exact: /logout
        - uri:
            prefix: /api/v1/products
      route:
        - destination:
            host: productpage
            port:
              number: 9080
```

Gateway で HTTP ポート 80 を受け付け、VirtualService で `/productpage` 等のパスを `productpage` サービスにルーティングしています。

### 4. ポートフォワードでアクセス

```bash
k port-forward -n istio-system svc/istio-ingressgateway 8080:80
```

http://localhost:8080/productpage を開き、Bookinfo アプリの画面が表示されれば成功です。

## クリーンアップ

```bash
k delete gateway bookinfo-gateway
k delete virtualservice bookinfo
k delete destinationrule httpbin
k delete -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/httpbin/httpbin.yaml
k delete -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/bookinfo/platform/kube/bookinfo.yaml
k delete -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/httpbin/sample-client/fortio-deploy.yaml
```

## まとめ

- **サーキットブレーカー**は DestinationRule の `connectionPool`（接続数制限）と `outlierDetection`（異常Pod除外）で構成される
- 接続数の上限を超えたリクエストは即座に 503 で拒否される
- **Gateway** で外部からメッシュ内サービスへのアクセスを制御できる
- Gateway + VirtualService の組み合わせで柔軟なルーティングが可能

## 関連記事

- [Istio サイドカー注入編](https://zenn.dev/ono_hiroki/articles/kubernetes-istio-sidecar)
- [Istio オブザーバビリティ編](https://zenn.dev/ono_hiroki/articles/kubernetes-istio-observability)
- [Istio トラフィック管理編](https://zenn.dev/ono_hiroki/articles/kubernetes-istio-traffic-management)

## ソースコード

https://github.com/ono-hiroki/maitake/tree/main/kubernetes/15-istio/04-circuit-breaker-gateway
