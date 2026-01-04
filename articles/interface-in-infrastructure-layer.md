---
title: "インフラ層にインターフェースを置いても良いと気づいた話"
emoji: "🧅"
type: "tech"
topics: ["cleanarchitecture"]
published: false
---

## はじめに

クリーンアーキテクチャはまだまだ勉強中ですが、最近まで私はこう理解していました。

> インターフェースはドメイン層やアプリケーション層に置き、インフラ層は実装だけを持つべき

しかしあるとき、この理解が**半分しか正しくない**ことに気づきました。

- ✅ 正しい部分: Application層やDomain層が依存するインターフェースはその層に置くべき
- ❌ 間違っていた部分: すべてのインターフェースを内側の層に置く必要がある

## 依存性逆転の原則（DIP）の本質

DIPが解決したい問題は何でしょうか？

```
❌ 避けたいこと
UseCase → EloquentUserRepository（具象クラス）

✅ 目指すこと
UseCase → UserRepositoryInterface ← EloquentUserRepository
           （UseCaseは抽象に依存）    （実装も抽象に依存）

```

※ `→` は「依存する」を意味します

**Application層やDomain層がInfrastructure層の実装詳細に依存しないこと**。これがDIPの本質です。

「すべてのインターフェースを内側の層に置くこと」ではありません。

## インターフェースの配置基準

判断の軸は**「誰がそのインターフェースに依存するか？」**です。

| 依存元 | インターフェースの配置 |
|--------|----------------------|
| Application層/Domain層 | その層に置く |
| Infrastructure層のみ | Infrastructure層でOK |

言い換えると、「Application層やDomain層にどこまで公開するか」「Infrastructure層にどこまで隠蔽するか」という境界設計の問題です。この境界に唯一の正解はなく、プロジェクトやチームの方針に委ねられます。

## 具体例

外部サービスと連携するUseCaseを例に考えます。

```
src/
├── Application/
│   └── SyncExternalUser/
│       ├── SyncExternalUserUseCase.php
│       └── ExternalUserApiInterface.php  ← アプリケーション層が依存
│
└── Infrastructure/
    └── ExternalApi/
        ├── ExternalUserApiClient.php          ← 実装
        ├── M2MTokenProviderInterface.php      ← インフラ内部専用
        └── Auth0M2MTokenProvider.php
```

#### ExternalUserApiInterface（アプリケーション層）

```php
// src/Application/SyncExternalUser/ExternalUserApiInterface.php
namespace App\Application\SyncExternalUser;

interface ExternalUserApiInterface
{
    public function fetchUser(string $externalUserId): ExternalUser;
}
```

UseCaseは「外部サービスからユーザーを取得する」ことだけを知っていれば十分です。認証方式がOAuthなのかAPIキーなのか、トークンをどう管理するのかは関心外です。

#### M2MTokenProviderInterface（インフラ層）

```php
// src/Infrastructure/ExternalApi/M2MTokenProviderInterface.php
namespace App\Infrastructure\ExternalApi;

interface M2MTokenProviderInterface
{
    public function getAccessToken(): string;
}
```

M2Mトークンの取得・キャッシュ・リフレッシュといった認証の実装詳細は、インフラ層に閉じ込めます。UseCaseからは見えなくて良いため、インフラ層に配置します。

このように、**UseCaseが知りたいこと**と**インフラ層で隠蔽してよいこと**を分けて考えると、インターフェースの配置が自然と決まります。

## 補足: Infrastructure層同士の依存について

同じ層内での依存は問題ありません。

先ほどの例では、`ExternalUserApiClient`が`M2MTokenProviderInterface`に依存しています。どちらもInfrastructure層に属していますが、これは許容されます。

クリーンアーキテクチャの制約は「内側の層が外側の層に依存してはいけない」であり、同じ層内の依存は制限されていません。

## まとめ

- DIPの目的は「Application層やDomain層がInfrastructure層の実装詳細に依存しないこと」
- インターフェースの配置は「誰が依存するか」で決める
- Infrastructure層内部でのみ使うインターフェースはInfrastructure層に置いてOK

**この気づきを得てから、より柔軟に設計を考えられるようになりました。**