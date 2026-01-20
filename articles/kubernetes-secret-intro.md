---
title: "Kubernetesをやってみる - Secretで機密情報を管理する"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "k8s", "secret", "container", "devops"]
published: true
---

## はじめに

この記事では、Kubernetes の **Secret** について、概念から実践的なハンズオンまで解説します。

Secret を理解することで、パスワードやトークンなどの機密情報を安全に管理できるようになります。

:::message
本記事では `kubectl` のエイリアスとして `k` を使用しています。
:::

## Secret とは

Secret は、**パスワードやトークンなどの機密情報を保存する** Kubernetes リソースです。

ConfigMap と似ていますが、機密データを扱うために設計されています。

```
┌─────────────────────────────────────────────────────────┐
│  Secret                                                 │
│  ┌─────────────────┬─────────────────────────────────┐  │
│  │  Key            │  Value (Base64)                 │  │
│  ├─────────────────┼─────────────────────────────────┤  │
│  │  db.user        │  dXNlcg==                       │  │
│  │  db.password    │  cGFzc3dvcmQ=                   │  │
│  └─────────────────┴─────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Pod                                                    │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Container                                      │    │
│  │  環境変数:                                       │    │
│  │    DB_USER=user                                 │    │
│  │    DB_PASSWORD=password                         │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

## Base64 エンコードについて

:::message alert
Secret の値は **Base64 エンコード** されますが、**暗号化ではありません**。
:::

```bash
# エンコード
echo -n "password" | base64
# cGFzc3dvcmQ=

# デコード（簡単に元に戻せる）
echo "cGFzc3dvcmQ=" | base64 -d
# password
```

つまり、Secret の YAML ファイルを Git にコミットすると、パスワードが漏洩します。

## Secret の種類

| タイプ | 用途 |
|--------|------|
| `Opaque` | 汎用的な機密情報（デフォルト） |
| `kubernetes.io/dockerconfigjson` | Docker レジストリの認証情報 |
| `kubernetes.io/tls` | TLS 証明書 |
| `kubernetes.io/basic-auth` | Basic 認証 |

このハンズオンでは `Opaque`（汎用）を使います。

## Secret の使い方

ConfigMap と同じく 4 つの方法がありますが、実務で主に使うのは 2 つです。

| 方法 | 説明 | 使用頻度 |
|------|------|---------|
| 環境変数（個別キー） | `secretKeyRef` で特定のキーを指定 | ★★★ |
| 環境変数（一括） | `envFrom` で全キーを一括設定 | ★☆☆ |
| ボリュームマウント | ファイルとしてマウント | ★★☆ |
| コマンドライン引数 | 環境変数経由で参照 | ★☆☆ |

:::message
実務で主に使うのは **環境変数（個別キー）** と **ボリュームマウント** の 2 つです。
:::

### 環境変数（個別キー）

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret    # Secret の名前
        key: db.password    # キー名
```

### ボリュームマウント

```yaml
volumeMounts:
  - name: secret-volume
    mountPath: /etc/secret
    readOnly: true
volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
```

## 更新時の挙動

| 使い方 | 更新反映 |
|--------|---------|
| 環境変数 | **Pod 再起動が必要** |
| ボリュームマウント | 自動反映（最大1分程度の遅延） |

:::message alert
環境変数で Secret を参照している場合、Secret を更新しても Pod を再起動しないと反映されません。
:::

## ConfigMap vs Secret

| 項目 | ConfigMap | Secret |
|------|-----------|--------|
| 用途 | 一般的な設定値 | 機密情報（パスワード等） |
| データ形式 | プレーンテキスト | Base64 エンコード |
| Git 管理 | OK | **NG**（漏洩の危険） |
| etcd 保存 | プレーンテキスト | 暗号化可能 |

## 本番環境での Secret 管理

Secret を Git で管理しないなら、どうやって管理するのか？

| 方法 | 説明 |
|------|------|
| 外部シークレット管理 | AWS Secrets Manager, HashiCorp Vault などから取得 |
| Sealed Secrets | 暗号化した Secret を Git 管理 |
| SOPS | YAML ファイルを暗号化 |
| External Secrets Operator | 外部サービスと Kubernetes を連携 |

minikube での学習では、これらは使わずに直接 Secret を作成します。

---

## ハンズオン

ここからは実際に手を動かして Secret の動作を確認します。

### やること

実務でよく使う 2 つの方法（環境変数・ボリュームマウント）を順番に試します。

| Step | 内容 |
|------|------|
| Step 1 | Secret を作成する |
| Step 2 | 環境変数で使う（Pod 作成 → 確認 → 更新 → 反映確認） |
| Step 3 | ボリュームマウントで使う（Pod 作成 → 確認 → 更新 → 反映確認） |
| Step 4 | クリーンアップ |

### 事前準備

```bash
# kind クラスターが起動していることを確認
k cluster-info
```

### Step 1: Secret を作成する

Secret の作成方法は 2 つあります。どちらか一方を実行してください。

#### 方法A: コマンドで作成

```bash
k create secret generic app-secret \
  --from-literal=db.user=myuser \
  --from-literal=db.password=mypassword
```

#### 方法B: YAML マニフェストで作成

```yaml:secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  # echo -n "myuser" | base64
  db.user: bXl1c2Vy
  # echo -n "mypassword" | base64
  db.password: bXlwYXNzd29yZA==
```

```bash
k apply -f secret.yaml
```

#### 確認

Secret が作成されたか確認します。

```bash
k get secret
```

```
NAME         TYPE     DATA   AGE
app-secret   Opaque   2      5s      ← DATA=2 で2件のキーがある
```

Secret の中身を確認します。

```bash
k describe secret app-secret
```

```
Name:         app-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
db.password:  10 bytes
db.user:      6 bytes
```

:::message
`describe` では値が表示されません（バイト数のみ）。
:::

値を確認したい場合は `-o yaml` を使います。

```bash
k get secret app-secret -o yaml
```

```yaml
apiVersion: v1
data:
  db.password: bXlwYXNzd29yZA==    ← Base64 エンコードされている
  db.user: bXl1c2Vy
kind: Secret
# ...
```

Base64 デコードして確認できます。

```bash
echo "bXlwYXNzd29yZA==" | base64 -d
# mypassword
```

### Step 2: 環境変数で使う

#### Pod を作成

```yaml:pod-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo-env
spec:
  containers:
    - name: app
      image: busybox:latest
      command: ["sleep", "3600"]
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: db.user
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: db.password
```

```bash
k apply -f pod-env.yaml
```

#### 環境変数を確認

Pod が Running になったら、中に入って確認します。

```bash
k exec -it secret-demo-env -- sh
```

```sh
/ # env | grep DB_
DB_USER=myuser            ← Secret の値が環境変数に入っている
DB_PASSWORD=mypassword

/ # exit
```

Secret で設定した値が `DB_USER`, `DB_PASSWORD` として入っていれば OK です。
Pod 内では Base64 デコードされた状態で参照できます。

#### Secret を更新して反映を確認

```bash
k delete secret app-secret
k create secret generic app-secret \
  --from-literal=db.user=newuser \
  --from-literal=db.password=newpassword

k exec secret-demo-env -- env | grep DB_
# DB_USER=myuser    ← まだ古い値のまま
```

Pod を再作成すると反映されます。

```bash
k delete pod secret-demo-env
k apply -f pod-env.yaml

k exec secret-demo-env -- env | grep DB_
# DB_USER=newuser    ← 更新された！
```

:::message
**ポイント**: 環境変数の場合、Pod を再起動しないと反映されません。
:::

#### クリーンアップ

```bash
k delete -f pod-env.yaml
```

### Step 3: ボリュームマウントで使う

#### Pod を作成

```yaml:pod-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo-volume
spec:
  containers:
    - name: app
      image: busybox:latest
      command: ["sleep", "3600"]
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secret
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: app-secret
```

```bash
k apply -f pod-volume.yaml
```

#### マウントされたファイルを確認

Pod が Running になったら、中に入って確認します。

```bash
k exec -it secret-demo-volume -- sh
```

```sh
/ # ls /etc/secret
db.password
db.user

/ # cat /etc/secret/db.password
newpassword

/ # exit
```

Secret の各キーがファイルとして `/etc/secret/` に存在し、中身が値と一致していれば OK です。

#### Secret を更新して反映を確認

```bash
k delete secret app-secret
k create secret generic app-secret \
  --from-literal=db.user=volumeuser \
  --from-literal=db.password=volumepassword

# 少し待ってから確認（最大1分程度の遅延あり）
k exec secret-demo-volume -- cat /etc/secret/db.password
# volumepassword    ← Pod 再起動なしで更新された！
```

:::message
**ポイント**: ボリュームマウントの場合、Pod を再起動しなくても自動で反映されます。
:::

#### クリーンアップ

```bash
k delete -f pod-volume.yaml
```

### Step 4: クリーンアップ

```bash
# 方法B（YAML）で作成した場合
k delete -f secret.yaml

# 方法A（コマンド）で作成した場合
k delete secret app-secret
```

## よく使うコマンドまとめ

```bash
# 作成
k create secret generic <name> --from-literal=key=value
k apply -f secret.yaml

# 確認
k get secret
k describe secret <name>
k get secret <name> -o yaml

# 値をデコードして確認
k get secret <name> -o jsonpath='{.data.key}' | base64 -d

# 削除
k delete secret <name>
```

## まとめ

- Secret はパスワードやトークンなどの機密情報を管理するリソース
- Base64 エンコードは暗号化ではないため、Git にコミットしない
- 環境変数とボリュームマウントの 2 つの使い方が主流
- 環境変数は Pod 再起動が必要、ボリュームマウントは自動反映
- 本番環境では外部シークレット管理（AWS Secrets Manager 等）を検討する

## 参考資料

- [Secrets | Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Managing Secrets using kubectl](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
