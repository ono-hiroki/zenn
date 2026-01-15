---
title: "自己署名証明書でnginxをHTTPS化する - HTTPからの段階的構築"
emoji: "🔐"
type: "tech"
topics: ["nginx", "https", "docker", "ssl", "mkcert"]
published: false
---

## はじめに

ローカル環境でnginxをHTTPS化したいけど、証明書周りがよくわからない...という方向けに、HTTPのみの状態から段階的にHTTPS化する手順を解説します。

この記事では以下の流れで進めます。

1. HTTPのみのnginxを構築し、HTTPSが使えないことを確認
2. 自己署名証明書を発行してHTTPS化
3. mkcertを使ってブラウザ警告をなくす

## 前提条件

本記事はmacOSを前提としています。以下がインストールされている必要があります。

- **Docker / Docker Compose**
- **Homebrew**

## 前提知識：自己署名証明書と認証局発行証明書の違い

### 自己署名証明書とは

認証局（CA）などの第三者が関与せず、**自分で作成・署名**する証明書です。

- 証明書に含まれる公開鍵に対応する**自身の秘密鍵**で署名する
- issuer（発行者）とsubject（発行先）が同じになる
- 第三者による身元確認がないため、証明書の正当性を検証できない
- 暗号化機能は正常に動作する

### 認証局（CA）発行証明書とは

**信頼された第三者機関**（認証局）が発行する証明書です。

- 認証局は申請者の**身元を確認**（ドメイン所有権、組織の実在性など）してから発行
- 認証局の秘密鍵で署名され、issuerとsubjectが異なる
- ブラウザに認証局の**ルート証明書がプリインストール**されているため自動的に信頼される

### 比較表

| 種類 | 発行者 | 用途 | ブラウザ警告 |
|------|--------|------|-------------|
| 自己署名証明書 | 自分自身 | 開発・テスト環境 | あり |
| 認証局発行証明書 | 信頼された第三者機関（CA） | 本番環境 | なし |

### よくある誤解

> ブラウザで「保護されていない」と表示される = HTTPSではない？

**これは誤解です。**

| 状態 | 暗号化 | ブラウザ表示 |
|------|--------|-------------|
| HTTP | なし | 「保護されていない通信」 |
| HTTPS + 自己署名 | **あり** | 「この接続は安全ではありません」 |
| HTTPS + 認証局発行 | **あり** | 鍵マーク（安全） |

警告は「証明書が信頼されていない」という意味であり、**暗号化自体は行われています**。

## ディレクトリ構成

最終的に以下の構成を作成します。

```
nginx-https-test/
├── http-only/
│   ├── docker-compose.yml
│   └── nginx.conf
├── self-signed/
│   ├── docker-compose.yml
│   ├── nginx.conf
│   ├── server.crt
│   └── server.key
├── mkcert/
│   ├── docker-compose.yml
│   ├── nginx.conf
│   ├── localhost.pem
│   └── localhost-key.pem
└── index.html
```

## Step 1: HTTPのみの状態を確認

まずHTTPだけで動作するnginxを構築し、HTTPSが使えないことを確認します。

### ファイルの作成

```bash
mkdir -p nginx-https-test/http-only
cd nginx-https-test
```

**index.html**

```html
<!DOCTYPE html>
<html>
<head>
    <title>nginx HTTPS Test</title>
</head>
<body>
<h1>Hello from nginx!</h1>
<p>If you can see this page, nginx is working correctly.</p>
</body>
</html>
```

**http-only/nginx.conf**

```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html;
    }
}
```

**http-only/docker-compose.yml**

```yaml
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ../index.html:/usr/share/nginx/html/index.html
```

### 起動と確認

```bash
cd http-only
docker compose up -d
```

**HTTPでアクセス（成功）**

```bash
curl -s http://localhost
```

```html
<!DOCTYPE html>
<html>
<head>
    <title>HTTPS Test</title>
</head>
...
```

**HTTPSでアクセス（失敗）**

```bash
curl -sk https://localhost; echo "Exit code: $?"
```

```
Exit code: 7
```

Exit code 7 = 接続拒否。443ポートでリッスンしていないため、HTTPSは使えません。

```bash
docker compose down
cd ..
```

## Step 2: 自己署名証明書を発行

### opensslコマンドで証明書を作成

```bash
mkdir self-signed
cd self-signed

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout server.key \
  -out server.crt \
  -subj "/CN=localhost"
```

### オプションの説明

| オプション | 説明 |
|-----------|------|
| `-x509` | 自己署名証明書を作成 |
| `-nodes` | 秘密鍵をパスワードで暗号化しない |
| `-days 365` | 有効期限365日 |
| `-newkey rsa:2048` | 2048ビットのRSA鍵を新規作成 |
| `-keyout` | 秘密鍵の出力先 |
| `-out` | 証明書の出力先 |
| `-subj "/CN=localhost"` | 証明書のCommon Name |

### 作成されるファイル

- `server.crt` - 証明書（公開鍵を含む）
- `server.key` - 秘密鍵

## Step 3: HTTPS対応のnginx設定

**self-signed/nginx.conf**

```nginx
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    location / {
        root   /usr/share/nginx/html;
        index  index.html;
    }
}
```

**self-signed/docker-compose.yml**

```yaml
services:
  nginx:
    image: nginx:latest
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./server.crt:/etc/nginx/ssl/server.crt
      - ./server.key:/etc/nginx/ssl/server.key
      - ../index.html:/usr/share/nginx/html/index.html
```

## Step 4: HTTPSで動作確認

```bash
docker compose up -d
```

**HTTPSでアクセス（成功）**

```bash
curl -sk https://localhost
```

```html
<!DOCTYPE html>
<html>
<head>
    <title>HTTPS Test</title>
</head>
...
```

:::message
`-k` オプションは証明書の検証をスキップするオプションです。自己署名証明書は信頼されていないため、このオプションが必要になります。
:::

**TLS接続の詳細を確認**

```bash
curl -kv https://localhost 2>&1 | grep -E "(SSL connection|subject|issuer)"
```

```
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256
*  subject: CN=localhost
*  issuer: CN=localhost
```

- **TLSv1.3** で暗号化されている
- **subject** と **issuer** が同じ = 自己署名証明書

```bash
docker compose down
cd ..
```

## Step 5: mkcertでブラウザ警告をなくす

自己署名証明書だとブラウザで警告が出ます。ローカル開発で警告を出したくない場合は **mkcert** を使います。

mkcertは「ローカル専用の認証局（CA）」を作成し、システムに登録することで、本物のCA発行証明書と同じ扱いにします。

### mkcertのインストール

```bash
brew install mkcert

# ローカルCAをインストール（システムに信頼させる）
mkcert -install
```

```
The local CA is now installed in the system trust store!
```

### 証明書の作成

```bash
mkdir mkcert
cd mkcert
mkcert localhost
```

```
Created a new certificate valid for the following names
 - "localhost"

The certificate is at "./localhost.pem" and the key at "./localhost-key.pem"
```

### nginx設定

**mkcert/nginx.conf**

```nginx
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/nginx/ssl/localhost.pem;
    ssl_certificate_key /etc/nginx/ssl/localhost-key.pem;

    location / {
        root   /usr/share/nginx/html;
        index  index.html;
    }
}
```

**mkcert/docker-compose.yml**

```yaml
services:
  nginx:
    image: nginx:latest
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./localhost.pem:/etc/nginx/ssl/localhost.pem
      - ./localhost-key.pem:/etc/nginx/ssl/localhost-key.pem
      - ../index.html:/usr/share/nginx/html/index.html
```

### 起動と確認

```bash
docker compose up -d
```

**-k オプションなしでアクセス（成功）**

```bash
curl -s https://localhost
```

```html
<!DOCTYPE html>
<html>
<head>
    <title>HTTPS Test</title>
</head>
...
```

自己署名証明書では `-k` が必要でしたが、mkcertでは不要です。

**TLS接続の詳細を確認**

```bash
curl -v https://localhost 2>&1 | grep -E "(SSL connection|subject|issuer)"
```

```
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256
*  subject: O=mkcert development certificate
*  issuer: O=mkcert development CA
```

- **subject** と **issuer** が異なる = CAが発行した証明書
- ローカルCAがシステムに登録されているため、信頼される

```bash
docker compose down
```

## トラブルシューティング

### 「保護されていない通信」と表示される

mkcertでルートCAをインストール後、ブラウザで「保護されていない通信」と表示される場合があります。

**原因**: ブラウザが古い証明書情報をキャッシュしている

**解決方法**:
1. ブラウザを**完全に終了**して再起動（macOSならCmd+Qで終了）
2. シークレットモードで試す

### Firefoxで警告が出る

FirefoxはシステムのトラストストアではなくFirefox独自の証明書ストアを使用します。

**解決方法**:

1. Firefoxで `about:config` を開く
2. `security.enterprise_roots.enabled` を `true` に設定

または、mkcertを再インストール：

```bash
mkcert -install
```

## 自己署名証明書とmkcertの比較

| 項目 | 自己署名証明書 | mkcert |
|------|--------------|--------|
| issuer | 自分自身 (`CN=localhost`) | ローカルCA (`O=mkcert development CA`) |
| curl -k | 必要 | 不要 |
| ブラウザ警告 | あり | なし |
| 用途 | 簡易テスト | ローカル開発 |

## まとめ

| Step | 状態 | HTTPSアクセス | ブラウザ警告 |
|------|------|--------------|-------------|
| Step 1 | HTTPのみ | 失敗（Exit code 7） | - |
| Step 2-4 | 自己署名証明書 | 成功（TLSv1.3） | あり |
| Step 5 | mkcert | 成功（TLSv1.3） | なし |

- **自己署名証明書**を使えば、ローカル環境で簡単にHTTPSを試せる
- **mkcert**を使えば、ブラウザ警告なしでHTTPSをローカル開発できる
- ブラウザの警告は「信頼性」の問題であり「暗号化」の問題ではない
- 本番環境ではACMやLet's Encryptなど認証局発行の証明書を使う

## 参考

- [mkcert - GitHub](https://github.com/FiloSottile/mkcert)
- [nginx SSL/TLS Configuration](https://nginx.org/en/docs/http/configuring_https_servers.html)
