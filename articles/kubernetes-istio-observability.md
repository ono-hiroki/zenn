---
title: "Istioをやってみる - Kiali/Jaegerでサービスメッシュを可視化する"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "istio", "kiali", "observability"]
published: false
---

## はじめに

この記事では、Istio のオブザーバビリティ機能を体験します。

Istio を導入すると、**アプリのコード変更なし**でメトリクスと分散トレースが自動収集されます。Kiali でサービス間のトラフィックフローを可視化し、Jaeger でリクエストの経路を追跡します。

:::message
本記事では `kubectl` のエイリアスとして `k` を使用しています。
:::

## なぜコード変更なしで可視化できるのか

すべての通信が Envoy サイドカーを経由するため、Envoy がリクエスト数・レイテンシ・エラー率・トレース情報を自動で収集します。

```
┌─────────────────────────────────────────────────┐
│ Pod                                             │
│                                                 │
│  アプリ ←──────→ Envoy (istio-proxy)            │
│                    │                            │
│                    ├─ リクエスト数をカウント    │
│                    ├─ レイテンシを計測          │
│                    ├─ エラー率を記録            │
│                    ├─ トレース情報を送信        │
│                    │                            │
│                    ↓                            │
│              :15090/stats/prometheus            │
└─────────────────────────────────────────────────┘
```

| 観点 | Istio なし | Istio あり |
|------|-----------|------------|
| メトリクス収集 | アプリに `/metrics` を実装 | Envoy が自動収集 |
| 分散トレーシング | アプリにライブラリを組み込み | Envoy が自動収集 |
| コード変更 | 必要 | 不要 |

## ツールの役割

```
                Envoy (istio-proxy)
                       │
       ┌───────────────┴───────────────┐
       ↓                               ↓
    メトリクス                      トレース
       │                               │
       ↓                               ↓
  Prometheus                        Jaeger
       │                               │
       └───────────┬───────────────────┘
                   ↓
                Kiali
```

| ツール | 役割 |
|--------|------|
| **Prometheus** | メトリクス収集・保存（Envoy の `/stats` をスクレイプ） |
| **Jaeger** | トレース収集・保存（Envoy からスパンを受信） |
| **Kiali** | 可視化ダッシュボード（Prometheus + Istio 設定を読み取り） |

## 前提条件

- Kubernetes クラスタが稼働していること
- Istio がインストール済みであること（[サイドカー注入編](https://zenn.dev/ono_hiroki/articles/kubernetes-istio-sidecar) を参照）

## 1. Bookinfo アプリをデプロイ

可視化には複数サービス間の通信が必要なため、Istio 公式の Bookinfo サンプルを使用します。

```
productpage → details
           → reviews → ratings
```

```bash
k label namespace default istio-injection=enabled --overwrite

k apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/bookinfo/platform/kube/bookinfo.yaml

# Pod が全て 2/2 で Running になるまで待つ
k get pods -w
```

## 2. Envoy のメトリクスを確認

アドオンをインストールする前に、Envoy が自動でメトリクスを収集していることを確認します。

```bash
k exec deploy/productpage-v1 -c istio-proxy -- \
  curl -s localhost:15090/stats/prometheus | grep istio_requests_total | head -5
```

`istio_requests_total` などのメトリクスが表示されます。これが Prometheus がスクレイプするデータです。

## 3. Prometheus / Kiali をインストール

Kiali は Prometheus のメトリクスを使用するため、両方インストールします。

```bash
# Prometheus（メトリクス収集）
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/addons/prometheus.yaml

# Kiali（サービスメッシュ可視化）
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/addons/kiali.yaml

# 起動を待つ
k get pods -n istio-system -w
```

`kiali` と `prometheus` の Pod が `Running` になったら次へ進みます。

## 4. トラフィックを発生させる

Kiali でトラフィックを可視化するため、リクエストを送信します。

**別のターミナル**で以下を実行:

```bash
while true; do
  k exec deploy/productpage-v1 -c istio-proxy -- \
    curl -s http://productpage:9080/productpage > /dev/null
  sleep 1
done
```

## 5. Kiali ダッシュボードを開く

```bash
istioctl dashboard kiali
```

ブラウザが自動で開きます。

1. 左メニューから **「Graph」** を選択
2. Namespace を **「default」** に変更
3. Display で **「Traffic Animation」** をオンにする

| 確認項目 | 説明 |
|----------|------|
| サービス間の依存関係 | どのサービスがどのサービスを呼んでいるか |
| リクエストの成功率 | 緑 = 成功、赤 = エラー |
| レイテンシ | 各サービスの応答時間 |
| トラフィック量 | 線の太さでリクエスト数を表現 |

## 6. Jaeger で分散トレーシング

```bash
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/addons/jaeger.yaml
k get pods -n istio-system -l app=jaeger -w

istioctl dashboard jaeger
```

1. **Service** ドロップダウンから `productpage.default` を選択
2. **Find Traces** をクリック

トレースをクリックすると、リクエストがどのサービスを経由したかが可視化されます:

```
productpage ─────────────────────────────────────→
    │
    ├─→ details ──────→
    │
    └─→ reviews ───────────────→
           │
           └─→ ratings ──→
```

各バーの長さが処理時間を表します。どのサービスがボトルネックかを特定できます。

## メトリクス vs トレースの使い分け

| 種類 | 内容 | 用途 |
|------|------|------|
| **メトリクス** | 集計データ（数値） | 「過去1分間に100リクエスト、平均50ms」 |
| **トレース** | 個別リクエストの追跡 | 「このリクエストは A→B→C を経由し、B で 30ms かかった」 |

| 状況 | 使うツール |
|------|-----------|
| 全体的なトラフィック傾向を見たい | Kiali（メトリクス） |
| 特定のリクエストがなぜ遅いか調べたい | Jaeger（トレース） |
| どのサービスでエラーが起きているか | Kiali（メトリクス） |
| エラーの詳細な原因を追跡したい | Jaeger（トレース） |

## クリーンアップ

```bash
k delete -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/bookinfo/platform/kube/bookinfo.yaml
k delete -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/addons/kiali.yaml
k delete -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/addons/prometheus.yaml
k delete -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/addons/jaeger.yaml
```

## まとめ

- Envoy サイドカーがメトリクスとトレースを自動収集する（アプリのコード変更不要）
- Kiali でサービス間のトラフィックフロー・成功率・レイテンシを可視化できる
- Jaeger で個別リクエストの経路と各サービスの処理時間を追跡できる

## 関連記事

- [Istio サイドカー注入編](https://zenn.dev/ono_hiroki/articles/kubernetes-istio-sidecar)
- Istio トラフィック管理編（VirtualService / Canary Deployment）
- Istio サーキットブレーカー / Gateway 編

## ソースコード

https://github.com/ono-hiroki/maitake/tree/main/kubernetes/15-istio
