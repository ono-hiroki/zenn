---
title: "Kubernetesをやってみる - ConfigMapで設定を管理する"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "configmap", "container", "devops"]
published: false
---

# はじめに

この記事では、Kubernetes の **ConfigMap** について、概念から実践的なハンズオンまで解説します。

ConfigMap を理解することで、アプリケーションの設定管理がより柔軟になります。

# ConfigMap とは

ConfigMap は、**非機密な設定データをキーバリューペアで保存する** Kubernetes リソースです。

アプリケーションの設定情報をコンテナイメージから分離することで、環境ごとに異なる設定を柔軟に管理できます。

```
┌─────────────────────────────────────────────────────────┐
│  ConfigMap                                              │
│  ┌─────────────────┬─────────────────────────────────┐  │
│  │  Key            │  Value                          │  │
│  ├─────────────────┼─────────────────────────────────┤  │
│  │  db.host        │  db-server                      │  │
│  │  db.port        │  3306                           │  │
│  │  log.level      │  info                           │  │
│  └─────────────────┴─────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Pod                                                    │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Container                                      │    │
│  │  環境変数:                                       │    │
│  │    DB_HOST=db-server                            │    │
│  │    DB_PORT=3306                                 │    │
│  │    LOG_LEVEL=info                               │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

# なぜ ConfigMap を使うのか

## 1. 設定の一元管理

環境変数の値を複数の場所に直接書いていると、変更時に抜け漏れが発生しやすくなります。

```yaml:Bad - 直接記述（管理が大変）
env:
  - name: DB_HOST
    value: "db-server"  # 変更時に全箇所を修正する必要がある
```

```yaml:Good - ConfigMap から参照（一箇所で管理）
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: db.host
```

## 2. 環境ごとの設定切り替え

開発環境と本番環境で異なる ConfigMap を用意することで、同じマニフェストを使いつつ設定だけを切り替えられます。

## 3. イメージの再利用性向上

設定をイメージに埋め込まないことで、同じイメージを異なる環境で利用できます。

# ConfigMap の使い方

ConfigMap を Pod から参照する方法は 4 つあります。

| 方法 | 説明 | 使用頻度 |
|------|------|---------|
| 環境変数（個別キー） | `configMapKeyRef` で特定のキーを指定 | ★★★ |
| 環境変数（一括） | `envFrom` で全キーを一括設定 | ★☆☆ |
| ボリュームマウント | ファイルとしてマウント | ★★☆ |
| コマンドライン引数 | 環境変数経由で `$(VAR_NAME)` で参照 | ★☆☆ |

:::message
実務で主に使うのは **環境変数（個別キー）** と **ボリュームマウント** の 2 つです。
:::

## 環境変数（個別キー）← 最もよく使う

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config    # ConfigMap の名前
        key: db.host        # キー名
```

## ボリュームマウント

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /etc/config
volumes:
  - name: config-volume
    configMap:
      name: app-config
```

設定ファイル（nginx.conf 等）を渡したいときに使います。

# 更新時の挙動

| 使い方 | 更新反映 |
|--------|---------|
| 環境変数 | **Pod 再起動が必要** |
| ボリュームマウント | 自動反映（最大1分程度の遅延） |

:::message alert
環境変数で ConfigMap を参照している場合、ConfigMap を更新しても Pod を再起動しないと反映されません。
:::

# 制限事項

- データサイズ: **1 MiB** まで
- 機密情報は保存しない（Secret を使う）
- Pod と同じ Namespace に存在する必要がある

# ConfigMap vs Secret

| 項目 | ConfigMap | Secret |
|------|-----------|--------|
| 用途 | 一般的な設定値 | 機密情報（パスワード等） |
| データ形式 | プレーンテキスト | Base64 エンコード |
| Git 管理 | OK | 非推奨 |

---

# ハンズオン編

ここからは実際に手を動かして ConfigMap の動作を確認します。

## やること

実務でよく使う 2 つの方法を、それぞれ独立して試します。

| Step | 内容 |
|------|------|
| Step 1 | ConfigMap を作成する |
| Step 2 | 環境変数で使う（Pod 作成 → 確認 → 更新 → 反映確認） |
| Step 3 | ボリュームマウントで使う（Pod 作成 → 確認 → 更新 → 反映確認） |
| Step 4 | クリーンアップ |

## 事前準備

```bash
# kind クラスタが起動していることを確認
kubectl cluster-info
```

# Step 1: ConfigMap を作成する

## 方法A: kubectl コマンドで作成

```bash
kubectl create configmap app-config \
  --from-literal=db.host=mysql-server \
  --from-literal=db.port=3306 \
  --from-literal=app.environment=development
```

## 方法B: YAML マニフェストで作成（推奨）

```yaml:configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  db.host: "mysql-server"
  db.port: "3306"
  app.environment: "development"
```

```bash
kubectl apply -f configmap.yaml
```

## 確認

ConfigMap が作成されたか確認します。

```bash
kubectl get configmap
```

```
NAME               DATA   AGE
app-config         3      9s      ← DATA=3 で3件のキーがある
kube-root-ca.crt   1      52m     ← Kubernetes が自動作成（無視してOK）
```

ConfigMap の中身を確認します。

```bash
kubectl describe configmap app-config
```

```
Name:         app-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
app.environment:
----
development

db.host:
----
mysql-server

db.port:
----
3306


BinaryData
====

Events:  <none>
```

`Data` セクションに3つのキーと値が表示されていれば OK です。

# Step 2: 環境変数で使う

## Pod を作成

```yaml:pod-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-env
spec:
  containers:
    - name: app
      image: busybox:latest
      command: ["sleep", "3600"]
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db.host
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db.port
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: app.environment
```

```bash
kubectl apply -f pod-env.yaml
```

## 環境変数を確認

Pod が Running になったら、中に入って確認します。

```bash
kubectl exec -it configmap-demo-env -- sh
```

```sh
/ # env | grep -E "DB_|APP_"
DB_HOST=mysql-server      ← ConfigMap の値が環境変数に入っている
DB_PORT=3306
APP_ENV=development

/ # exit
```

## ConfigMap を更新して反映を確認

```bash
kubectl patch configmap app-config --patch '{"data":{"db.host":"new-mysql-server"}}'

kubectl exec configmap-demo-env -- env | grep DB_HOST
# DB_HOST=mysql-server    ← まだ古い値のまま
```

Pod を再作成すると反映されます。

```bash
kubectl delete pod configmap-demo-env
kubectl apply -f pod-env.yaml

kubectl exec configmap-demo-env -- env | grep DB_HOST
# DB_HOST=new-mysql-server    ← 更新された！
```

:::message
**ポイント**: 環境変数の場合、Pod を再起動しないと反映されません。
:::

## クリーンアップ

```bash
kubectl delete -f pod-env.yaml
```

# Step 3: ボリュームマウントで使う

## Pod を作成

```yaml:pod-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-volume
spec:
  containers:
    - name: app
      image: busybox:latest
      command: ["sleep", "3600"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

```bash
kubectl apply -f pod-volume.yaml
```

## マウントされたファイルを確認

Pod が Running になったら、中に入って確認します。

```bash
kubectl exec -it configmap-demo-volume -- sh
```

```sh
/ # ls /etc/config
app.environment
db.host
db.port

/ # cat /etc/config/db.host
new-mysql-server

/ # exit
```

## ConfigMap を更新して反映を確認

```bash
kubectl patch configmap app-config --patch '{"data":{"db.host":"auto-updated-server"}}'

# 少し待ってから確認（最大1分程度の遅延あり）
kubectl exec configmap-demo-volume -- cat /etc/config/db.host
# auto-updated-server    ← Pod 再起動なしで更新された！
```

:::message
**ポイント**: ボリュームマウントの場合、Pod を再起動しなくても自動で反映されます。
:::

## クリーンアップ

```bash
kubectl delete -f pod-volume.yaml
```

# Step 4: クリーンアップ

```bash
kubectl delete -f configmap.yaml
```

# よく使うコマンドまとめ

```bash
# 作成
kubectl create configmap <name> --from-literal=key=value
kubectl apply -f configmap.yaml

# 確認
kubectl get configmap
kubectl describe configmap <name>
kubectl get configmap <name> -o yaml

# 更新
kubectl apply -f configmap.yaml
kubectl patch configmap <name> --patch '{"data":{"key":"new-value"}}'

# 削除
kubectl delete configmap <name>
```

# まとめ

- ConfigMap は非機密な設定データを管理するリソース
- 環境変数とボリュームマウントの 2 つの使い方が主流
- 環境変数は Pod 再起動が必要、ボリュームマウントは自動反映
- 機密情報は Secret を使う

# 参考リンク

- [ConfigMaps | Kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
