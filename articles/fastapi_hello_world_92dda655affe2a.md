---
title: "DockerでFastAPIの開発環境を構築して「Hello World」する"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["FastAPI", "Docker"]
published: true
---

Hello Worldします。

# 🛠 使用した技術

- FastAPI
- Docker
- Docker Compose

# 📁 ディレクトリ構成

まずは以下のような構成でディレクトリとファイルを用意します。

```plain text
fastapi-docker/
├── app/
│   └── main.py
├── Dockerfile
└── docker-compose.yml
```

# ✏️ 各ファイルの中身

## 1️⃣ FastAPIアプリ作成

app/main.py

FastAPIアプリ本体です。ルートにアクセスしたときにメッセージを返します。

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

## 2️⃣ Dockerfile作成

Python環境＋FastAPI＋Uvicornを用意するためのDockerfileです。

```Docker
FROM python:3.11-slim

WORKDIR /app

COPY ./app /app

RUN pip install --no-cache-dir fastapi uvicorn
```

## 3️⃣ docker-compose.yml作成

```yaml
version: "3.9"

services:
  fastapi:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./app:/app
    command: >
      uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

# ▶️ サーバー起動

以下のコマンドを実行するだけ

```bash
docker-compose up --build
```

するとログにこう表示されます：

```bash
Uvicorn running on http://127.0.0.1:8000
```

ブラウザで http://localhost:8000 にアクセスすると：

```Plain Text
{"message":"Hello World"}
```

# 🧪 Swagger UIも自動生成

FastAPIはSwagger UIを自動生成する機能があります。以下のURLにアクセスすると、APIの仕様書を確認できます。これは驚き。すごい。

- http://localhost:8000/docs → Swagger UI
- http://localhost:8000/redoc → ReDoc

# 🧪 OAS（OpenAPI Specification）のJSONも取得可能

以下のURLにアクセスすると、APIの仕様書をJSON形式で取得できます。これもすごい。yamlは生成してくれないみたい🥺

- http://localhost:8000/openapi.json

# 🧹 終了・削除コマンド

サーバーを停止するときは、以下のコマンドを実行します。

```bash
docker-compose down
```

# ✍️ 感想
わりと簡単でした。自動生成されるSwagger UIも便利です。ただ、「OASを自動生成 = コードが正しい」となるので気をつけたいかも。

