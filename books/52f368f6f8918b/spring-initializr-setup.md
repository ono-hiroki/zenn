---
title: "環境構築"
---

# 【初心者向け】Spring InitializrでSpring Bootの開発環境を構築する方法

## はじめに

JavaでWebアプリケーションを開発するなら、**Spring Boot**がとても人気です。  
そして、そのSpring Bootのプロジェクトを簡単に始められるツールが **Spring Initializr** です。

この記事では、Spring Initializrを使って最初のSpring Bootプロジェクトを立ち上げる方法を解説します。

---

## 1. Spring Initializrとは？

[Spring Initializr](https://start.spring.io/) は、Web上でSpring Bootのプロジェクト雛形を簡単に作れる公式ツールです。  
必要な依存関係（ライブラリ）を選ぶだけで、すぐに開発を始められる状態のZIPファイルを生成してくれます。

---

## 2. Spring Bootプロジェクトの作成手順

### ステップ① Spring Initializr にアクセス

以下のURLにアクセスします：

👉 [https://start.spring.io/](https://start.spring.io/)

### ステップ② プロジェクト情報を入力

| 項目           | 設定例                |
|--------------|--------------------|
| Project      | Gradle Project     |
| Language     | Java               |
| Spring Boot  | 3.2.2（最新の安定版）      |
| Group        | `com.example`      |
| Artifact     | `demo`             |
| Name         | `demo`             |
| Package name | `com.example.demo` |
| Packaging    | Jar                |
| Java         | 21                 |

### ステップ③ 依存関係を追加

特に指定しない

### ステップ④ ダウンロード & 解凍

「**Generate**」ボタンをクリックするとZIPファイルがダウンロードされます。  
それを任意の場所に解凍しましょう。

---

## 3. 開発環境でプロジェクトを開く

### IntelliJ IDEAの場合

1. IntelliJを起動
2. 開くで、build.gradleファイルを選択
3. Gradleが自動的に依存関係を解決します

---

## 4. 動作確認してみよう

下記のコマンドでSpring Bootアプリケーションを起動します：

```bash
./gradlew bootRun
```