---
title: "Kubernetesをやってみる - manifestファイルでPodを定義する"
emoji: "☸️"
type: "tech"
topics: ["kubernetes", "yaml", "docker"]
published: true
---

## はじめに

Kubernetesを触り始めたので、学んだことを記録していきます。

前回の記事では`kubectl run`コマンドでPodを起動しました。

https://zenn.dev/hono8944/articles/hello-kubernetes

しかし、毎回コマンドを手打ちするのは手間がかかりますし、どんな設定で起動したのか後から確認することも難しくなります。

manifestファイルを使えば、Kubernetesリソースの定義をYAMLで記述し、ファイルとして保存できます。これにより、いつでも同じ構成を簡単に再現できるようになります。

## 環境

- Kubernetes クラスター: Docker Desktop

本記事のコマンド例では`k`を`kubectl`のエイリアスとして使用しています。エイリアスを設定していない場合は`k`を`kubectl`に読み替えてください。

## manifestの基本構造

manifestの構造をnginxのPodを例に見てみましょう。

```yaml
# APIバージョン (<group>/<version> の形式、v1はcoreグループ)
apiVersion: v1
# リソースの種類
kind: Pod
metadata:
  # リソース名
  name: nginx
  # 所属するnamespace
  namespace: dev
# リソースの具体的な設定 (リソースの種類によって異なる)
spec:
  # Pod内のコンテナ一覧
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

- 一部のリソースでは`metadata`や`spec`がない場合もあります
- 指定可能な`kind`や`apiVersion`は`kubectl api-resources`で確認できます

## シンプルなPodを定義する

作業ディレクトリを作成し、その中にmanifestファイルを配置していきます。

```
manifest-practice/
└── pod.yaml
```

```bash
mkdir -p ~/manifest-practice
cd ~/manifest-practice
```

以下の内容で`pod.yaml`を作成します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

| 項目 | 説明 |
|---|---|
| `apiVersion: v1` | APIのバージョン（v1はcoreグループ） |
| `kind: Pod` | リソースの種類 |
| `metadata.name` | Podの名前 |
| `metadata.namespace` | Podを作成するnamespace |
| `spec.containers` | コンテナのリスト |
| `spec.containers[].name` | コンテナの名前 |
| `spec.containers[].image` | 使用するコンテナイメージ |
| `spec.containers[].ports` | コンテナが待ち受けるポート |

## Podをデプロイする

### namespaceの作成

manifestで指定しているnamespace `dev` を事前に作成しておく必要があります。

```bash
k create namespace dev
```

```
namespace/dev created
```

### manifestの適用

`kubectl apply`コマンドでmanifestを適用します。

```bash
k apply -f pod.yaml
```

```
pod/nginx created
```

### 動作確認

Podが正しく動作しているか確認しましょう。

まずPodの状態を確認します。

```bash
k get pod -n dev
```

```
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          10s
```

STATUSが`Running`になっていればOKです。

port-forwardでローカルからアクセスできるようにします。

```bash
k port-forward -n dev pod/nginx 8080:80
```

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

別のターミナルを開いて確認します。

```bash
curl http://localhost:8080
```

nginxのデフォルトページが表示されれば成功です。確認が終わったら`Ctrl+C`でport-forwardを終了します。

### Podの削除

次のセクションに進む前にPodを削除しておきます。

```bash
k delete -f pod.yaml
```

```
pod "nginx" deleted
```

## サイドカーパターンを試す

Podは複数のコンテナを含むことができます。メインのコンテナを補助する形で動作するコンテナを「サイドカー」と呼びます。

```
manifest-practice/
├── pod.yaml
└── pod-sidecar.yaml
```

### ログ転送の例

以下の例では、nginxのアクセスログを共有ボリュームに出力し、サイドカーコンテナがそのログを読み取ります。
実務ではこのパターンでFluentdやFluentbitなどのログ収集ツールにログを転送します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-sidecar
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  - name: log-shipper
    image: busybox:latest
    command:
    - /bin/sh
    - -c
    - |
      # ログファイルが作成されるまで待機
      while [ ! -f /var/log/nginx/access.log ]; do
        sleep 1
      done
      tail -f /var/log/nginx/access.log
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  volumes:
  - name: logs
    emptyDir: {}
```

| 項目 | 説明 |
|---|---|
| `spec.containers` | 2つのコンテナを定義 |
| `spec.volumes` | Pod内で共有するボリュームを定義 |
| `volumeMounts` | 各コンテナでボリュームをマウントする場所を指定 |
| `emptyDir` | Pod起動時に作成される一時的な空のディレクトリ |

この構成では:
- `nginx`コンテナがアクセスログを`/var/log/nginx/access.log`に出力
- `log-shipper`コンテナが同じボリュームをマウントしてログを読み取り

### 適用と動作確認

```bash
k apply -f pod-sidecar.yaml
```

```
pod/nginx-with-sidecar created
```

Podの状態を確認します。READYが`2/2`になっていることを確認してください。

```bash
k get pod -n dev
```

```
NAME                 READY   STATUS    RESTARTS   AGE
nginx-with-sidecar   2/2     Running   0          10s
```

port-forwardでnginxにアクセスできるようにします。

```bash
k port-forward -n dev pod/nginx-with-sidecar 8080:80
```

別のターミナルでリクエストを送ります。

```bash
curl http://localhost:8080
```

### サイドカーのログを確認

`-c`オプションでコンテナを指定し、`-f`オプションでリアルタイムにログを確認します。

```bash
k logs -f -n dev nginx-with-sidecar -c log-shipper
```

この状態で別のターミナルから`curl http://localhost:8080`を実行すると、リアルタイムでログが流れてきます。

```
10.1.0.1 - - [10/Jan/2025:12:00:00 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.0.0" "-"
10.1.0.1 - - [10/Jan/2025:12:00:05 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.0.0" "-"
```

確認が終わったら`Ctrl+C`でログのフォローを終了し、port-forwardも終了してPodを削除します。

```bash
k delete -f pod-sidecar.yaml
```

```
pod "nginx-with-sidecar" deleted
```

## クリーンアップ

最後にnamespaceを削除します。

```bash
k delete namespace dev
```

```
namespace "dev" deleted
```

## まとめ

この記事では、manifestファイルを使ってPodを定義し、起動する方法を学びました。

manifestファイルを使うメリット：
- **再現性**: 同じ構成をいつでも正確に再現できる
- **バージョン管理**: Gitで変更履歴を追跡できる
- **共有**: チームメンバーと設定を簡単に共有できる
- **宣言的管理**: あるべき状態をコードで定義できる

今回学んだこと：
- manifestの基本構造（apiVersion、kind、metadata、spec）
- `kubectl apply -f`でmanifestを適用する方法
- 複数コンテナを持つPod（サイドカーパターン）の定義方法

## 参考資料

- [API概要 | Kubernetes](https://kubernetes.io/ja/docs/reference/using-api/)
