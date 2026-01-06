---
title: "読み取りは自由でいい"
emoji: "🍞"
type: "tech"
topics: [ "php","architecture","cqrs" ]
published: true
---

## はじめに

以前、私はこう思っていました。

「データの取得も書き込みと同じ経路を通すのが正しい設計だ」

結果、画面に表示するだけのデータでも、毎回エンティティを組み立てて、DTOに詰め替えて...という処理を書いていました。

この記事では、**読み取りは自由に書いていい**という気づきを得て、設計に対する考え方が柔軟になった経験を共有します。

## 以前の自分：すべてをRepository経由で

当時の私のコードはこんな感じでした。

```
// ユーザー一覧を取得するUseCase
class GetUserListUseCase {
    constructor(userRepository)

    execute() {
        // Repositoryからエンティティを取得
        users = userRepository.findAll()

        // DTOに詰め替えて返す
        return users.map(user => {
            id: user.id,
            name: user.name,
            email: user.email,
        })
    }
}
```

一見きれいに見えます。でも、この設計には問題がありました。

## 感じていた違和感

### 1. 画面ごとに必要なデータが違う

一覧画面では `id`, `name`, `email` だけでいい。
詳細画面では関連データも含めて全部ほしい。
管理画面では集計値も必要。

なのに、毎回同じ `User` エンティティを組み立てていました。

### 2. 「取得するだけなのに、なぜこんなに複雑？」

画面に表示するだけ。
ビジネスロジックは一切通さない。
なのに、エンティティを組み立てて、DTOに詰め替えて...

**「これ、本当に必要？」**

## 気づき：書き込みと読み取りは責務が違う

転機は、CQRSという考え方に触れたときでした。

**CQRS（Command Query Responsibility Segregation）** は、読み取りと書き込みのモデルを分離するアーキテクチャパターンです。本格的に導入すると別DBを用意したりと大掛かりになりますが、その根底にある考え方はシンプルでした。

| 操作 | 目的 | 必要なこと |
|------|------|------------|
| **書き込み** | データを変更する | ビジネスルールを守る必要がある |
| **読み取り** | データを表示する | 表示に最適化されていればいい |

つまり：

- **書き込み** → ビジネスルールを通して、整合性を守る
- **読み取り** → 必要なデータを、必要な形で、効率よく取得すればいい

**読み取りでエンティティを経由する必要はなかった**のです。

## 変化後：読み取りは自由に

気づいてからは、読み取り専用のQueryServiceを作るようになりました。

```
// 読み取り専用のQueryService
class UserListQueryService {
    constructor(db)

    findForList() {
        // 必要なカラムだけSELECT
        // JOINも自由に
        // エンティティを経由しない
        return db.query(
            "SELECT id, name, email FROM users WHERE deleted_at IS NULL"
        )
    }
}
```

### 得られたメリット

1. **シンプル**：必要なデータを直接取得するだけ
2. **高速**：必要なカラムだけSELECT、最適なJOIN
3. **柔軟**：画面ごとに最適化したQueryを書ける
4. **明確**：「これは読み取り専用」と意図が伝わる

## 書き込み側は変わらず厳格に

誤解のないように補足すると、**書き込み側は今まで通り厳格に**設計しています。

```
// 書き込みはビジネスルールを通す
class PlaceOrderUseCase {
    constructor(orderRepository, inventoryService)

    execute(command) {
        // 在庫チェック（ビジネスルール）
        if (!inventoryService.hasStock(command.productId, command.quantity)) {
            throw new Error("在庫不足")
        }

        // 注文を作成
        order = Order.create(command.productId, command.quantity)

        // 保存
        orderRepository.save(order)
    }
}
```

書き込みでは：
- 在庫の確認
- 上限チェック
- 整合性の担保

など、**ビジネスルールを守る必要がある**からです。

## まとめ：書き込みは厳格に、読み取りは実用的に

| | 書き込み (Command) | 読み取り (Query) |
|---|---|---|
| **経由** | Repository → エンティティ | QueryService（直接SQL可） |
| **理由** | ビジネスルールを守るため | 表示に最適化するため |
| **設計方針** | 厳格に | 実用的に |

「すべてを同じ経路で処理すべき」という思い込みを捨てたら、設計がぐっと楽になりました。

読み取りは、**用途に合わせて最適な方法で取得していい**。

この気づきが、誰かの設計の参考になれば幸いです。
