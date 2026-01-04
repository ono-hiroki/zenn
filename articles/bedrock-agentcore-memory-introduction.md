---
title: "AWS CLIで学ぶAmazon Bedrock AgentCore Memory"
emoji: "🧠"
type: "tech"
topics: ["aws","agentcore"]
published: true
---


## はじめに

AWS Bedrock AgentCore Memory は、AIエージェントに「記憶」を持たせるためのマネージドサービスです。会話履歴の保存だけでなく、会話から重要な情報を自動抽出し、後からセマンティック検索で取り出すことができます。

この記事では、AWS CLI を使って AgentCore Memory の基本操作を一通り試してみます。

## AgentCore Memory の基本概念

### Short-term Memory と Long-term Memory

AgentCore Memory には2種類のメモリがあります。

| 種類 | 説明 | 用途 |
|------|------|------|
| **Short-term Memory** | 生の会話データをそのまま保存 | セッション内のコンテキスト維持 |
| **Long-term Memory** | 会話から抽出・要約された情報 | パーソナライズ、過去の知識検索 |

```
会話データ → [Short-term Memory] → [自動抽出] → [Long-term Memory]
                  （生データ）                      （構造化された知識）
```

### Memory の設計単位

Memory リソースは **エージェント（アプリケーション）単位** で作成し、その中で `actorId`（ユーザー）と `sessionId`（セッション）で論理的に分離します。

```
Memory（エージェント単位）
├── actorId: user-001（ユーザーA）
│   ├── sessionId: session-001
│   └── sessionId: session-002
└── actorId: user-002（ユーザーB）
    └── sessionId: session-003
```

## 4つの Memory Strategy

Long-term Memory への抽出方法を決めるのが **Strategy（戦略）** です。Built-in で4種類用意されています。

### 1. Semantic（セマンティック）

会話から **事実・知識** を抽出します。

**抽出例：**
- 注文番号 `#XYZ-123` がサポートケースに関連
- プロジェクト締切が10月25日
- ユーザーがソフトウェア v2.1 を使用中

### 2. User Preference（ユーザー設定）

ユーザーの **好み・選択パターン** を抽出します。

**抽出例：**
- 配送業者は FedEx を希望
- コーディングスタイルはスネークケース派
- フォーマルな口調を好む

### 3. Summary（サマリー）

セッション内の会話を **要約** します。コンテキストウィンドウの節約に有効です。

**抽出例：**
- 「ユーザーは注文#XYZ-123の問題を報告し、エージェントが交換を手配した」

### 4. Episodic（エピソード）

意味のある「エピソード」を記録し、**Reflection（振り返り）** で学習します。最も高度な戦略です。

**抽出例：**
- コードデプロイでエラー発生 → 代替アプローチで解決した経緯
- どのツール組み合わせが成功しやすいかのパターン

### 戦略の選び方

| ユースケース | 推奨戦略 |
|-------------|---------|
| カスタマーサポート | Semantic + UserPreference + Summary |
| コーディングエージェント | Semantic + Episodic |
| パーソナルアシスタント | UserPreference + Summary |

## 実践：AWS CLI で Memory を操作する

### 環境準備

```bash
export AWS_REGION=us-east-1
```

### 1. Memory の作成

Semantic 戦略を持つ Memory を作成します。

```bash
aws bedrock-agentcore-control create-memory \
  --region $AWS_REGION \
  --name "my_memory" \
  --description "AWS CLIで作成したMemory" \
  --event-expiry-duration 7 \
  --memory-strategies '[
    {
      "semanticMemoryStrategy": {
        "name": "semantic_strategy",
        "description": "Semantic memory for facts",
        "namespaces": ["facts"]
      }
    }
  ]'
```

**レスポンス（抜粋）：**

```json
{
    "memory": {
        "id": "my_memory-GU1PAu70oU",
        "status": "CREATING",
        "strategies": [
            {
                "strategyId": "semantic_strategy-39KW5dFpFw",
                "type": "SEMANTIC",
                "namespaces": ["facts"],
                "status": "CREATING"
            }
        ]
    }
}
```

### 2. Memory の一覧・詳細確認

```bash
# 一覧
aws bedrock-agentcore-control list-memories --region $AWS_REGION

# 詳細
export MEMORY_ID="my_memory-GU1PAu70oU"
aws bedrock-agentcore-control get-memory \
  --region $AWS_REGION \
  --memory-id "$MEMORY_ID"
```

ステータスが `ACTIVE` になれば準備完了です。

### 3. 会話データの保存（Short-term Memory）

`create-event` で会話を保存します。

```bash
export ACTOR_ID="user_001"
export SESSION_ID="session_001"

aws bedrock-agentcore create-event \
  --region $AWS_REGION \
  --memory-id "$MEMORY_ID" \
  --actor-id "$ACTOR_ID" \
  --session-id "$SESSION_ID" \
  --event-timestamp "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --payload '[
    {
      "conversational": {
        "role": "USER",
        "content": {"text": "私の名前は田中太郎です"}
      }
    },
    {
      "conversational": {
        "role": "ASSISTANT",
        "content": {"text": "はじめまして、田中太郎さん！"}
      }
    }
  ]'
```

### 4. Short-term Memory の確認

```bash
aws bedrock-agentcore list-events \
  --region $AWS_REGION \
  --memory-id "$MEMORY_ID" \
  --actor-id "$ACTOR_ID" \
  --session-id "$SESSION_ID" \
  --include-payloads
```

**レスポンス：**

```json
{
    "events": [
        {
            "eventId": "0000001767482726000#0b93904f",
            "payload": [
                {
                    "conversational": {
                        "content": {"text": "私の名前は田中太郎です"},
                        "role": "USER"
                    }
                },
                {
                    "conversational": {
                        "content": {"text": "はじめまして、田中太郎さん！"},
                        "role": "ASSISTANT"
                    }
                }
            ]
        }
    ]
}
```

生の会話データがそのまま保存されています。

### 5. Long-term Memory の確認

Long-term Memory への抽出は **非同期** で行われます。数秒〜数十秒待ってから確認します。

```bash
aws bedrock-agentcore list-memory-records \
  --region $AWS_REGION \
  --memory-id "$MEMORY_ID" \
  --namespace "facts"
```

**最初は空：**

```json
{
    "memoryRecordSummaries": []
}
```

**しばらく待つと抽出完了：**

```json
{
    "memoryRecordSummaries": [
        {
            "memoryRecordId": "mem-f4d9b704-02cc-448f-bcaf-ba025e28d6aa",
            "content": {
                "text": "ユーザーの名前は田中太郎である。"
            },
            "memoryStrategyId": "semantic_strategy-39KW5dFpFw",
            "namespaces": ["facts"]
        }
    ]
}
```

会話から「ユーザーの名前は田中太郎である」という **事実** が自動抽出されました！

### 6. セマンティック検索

自然言語で Long-term Memory を検索できます。

```bash
aws bedrock-agentcore retrieve-memory-records \
  --region $AWS_REGION \
  --memory-id "$MEMORY_ID" \
  --namespace "facts" \
  --search-criteria '{"searchQuery": "このユーザーの名前は？"}' \
  --max-results 5
```

**レスポンス：**

```json
{
    "memoryRecordSummaries": [
        {
            "memoryRecordId": "mem-f4d9b704-02cc-448f-bcaf-ba025e28d6aa",
            "content": {
                "text": "ユーザーの名前は田中太郎である。"
            },
            "score": 0.4250299
        }
    ]
}
```

`score` は類似度スコアで、クエリとの関連度を示しています。

### 7. セッション・アクターの管理

```bash
# セッション一覧
aws bedrock-agentcore list-sessions \
  --region $AWS_REGION \
  --memory-id "$MEMORY_ID" \
  --actor-id "$ACTOR_ID"

# アクター一覧
aws bedrock-agentcore list-actors \
  --region $AWS_REGION \
  --memory-id "$MEMORY_ID"
```

## 実践：複数 Strategy を使ったカスタマーサポート

1つの Memory に複数の Strategy を設定して、同じ会話から異なる観点で情報を抽出できます。

### 1. 複数 Strategy を持つ Memory の作成

```bash
aws bedrock-agentcore-control create-memory \
  --region $AWS_REGION \
  --name "customer_support_memory" \
  --description "カスタマーサポート用Memory" \
  --event-expiry-duration 30 \
  --memory-strategies '[
    {
      "semanticMemoryStrategy": {
        "name": "facts_extractor",
        "description": "事実の抽出",
        "namespaces": ["/facts/{actorId}"]
      }
    },
    {
      "userPreferenceMemoryStrategy": {
        "name": "preference_learner",
        "description": "ユーザーの好みを学習",
        "namespaces": ["/preferences/{actorId}"]
      }
    },
    {
      "summaryMemoryStrategy": {
        "name": "session_summarizer",
        "description": "セッションの要約",
        "namespaces": ["/summaries/{actorId}/{sessionId}"]
      }
    }
  ]'
```

3つの Strategy が作成されます：

| Strategy | Type | Namespace |
|----------|------|-----------|
| facts_extractor | SEMANTIC | `/facts/{actorId}` |
| preference_learner | USER_PREFERENCE | `/preferences/{actorId}` |
| session_summarizer | SUMMARIZATION | `/summaries/{actorId}/{sessionId}` |

### 2. 顧客ごとの会話を保存

**田中さんの会話：**

```bash
export SUPPORT_MEMORY_ID="customer_support_memory-CjBhIu59lj"

aws bedrock-agentcore create-event \
  --region $AWS_REGION \
  --memory-id "$SUPPORT_MEMORY_ID" \
  --actor-id "customer_tanaka" \
  --session-id "2026-01-04-001" \
  --event-timestamp "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --payload '[
    {
      "conversational": {
        "role": "USER",
        "content": {"text": "注文番号 ABC-123 の荷物が届きません。あと、次回からは必ずFedExで送ってください。"}
      }
    },
    {
      "conversational": {
        "role": "ASSISTANT",
        "content": {"text": "ご不便をおかけして申し訳ございません。注文番号 ABC-123 を確認いたします。また、今後の配送はFedExをご希望とのこと、承知いたしました。"}
      }
    }
  ]'
```

**鈴木さんの会話：**

```bash
aws bedrock-agentcore create-event \
  --region $AWS_REGION \
  --memory-id "$SUPPORT_MEMORY_ID" \
  --actor-id "customer_suzuki" \
  --session-id "2026-01-04-001" \
  --event-timestamp "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --payload '[
    {
      "conversational": {
        "role": "USER",
        "content": {"text": "先日購入した商品Xを返品したいです。サイズが合いませんでした。あと、私はメールより電話で連絡してほしいです。"}
      }
    },
    {
      "conversational": {
        "role": "ASSISTANT",
        "content": {"text": "商品Xの返品を承ります。返品ラベルをお送りしますね。ご連絡方法は電話をご希望とのこと、記録いたしました。"}
      }
    }
  ]'
```

### 3. Actor 一覧の確認

```bash
aws bedrock-agentcore list-actors \
  --region $AWS_REGION \
  --memory-id "$SUPPORT_MEMORY_ID"
```

```json
{
    "actorSummaries": [
        {"actorId": "customer_suzuki"},
        {"actorId": "customer_tanaka"}
    ]
}
```

### 4. 各 Strategy の抽出結果を確認

**田中さんの事実（Semantic）：**

```bash
aws bedrock-agentcore list-memory-records \
  --region $AWS_REGION \
  --memory-id "$SUPPORT_MEMORY_ID" \
  --namespace "/facts/customer_tanaka"
```

```json
{
    "memoryRecordSummaries": [
        {
            "content": {
                "text": "ユーザーの注文番号 ABC-123 の荷物が届いていない。"
            }
        },
        {
            "content": {
                "text": "ユーザーは今後の配送をFedExで行うことを希望している。"
            }
        }
    ]
}
```

**田中さんの好み（UserPreference）：**

```bash
aws bedrock-agentcore list-memory-records \
  --region $AWS_REGION \
  --memory-id "$SUPPORT_MEMORY_ID" \
  --namespace "/preferences/customer_tanaka"
```

```json
{
    "memoryRecordSummaries": [
        {
            "content": {
                "text": "{\"context\":\"The user explicitly requested that future deliveries must be sent via FedEx.\",\"preference\":\"Prefers FedEx for shipping\",\"categories\":[\"shipping\",\"delivery\"]}"
            }
        }
    ]
}
```

**田中さんのセッション要約（Summary）：**

```bash
aws bedrock-agentcore list-memory-records \
  --region $AWS_REGION \
  --memory-id "$SUPPORT_MEMORY_ID" \
  --namespace "/summaries/customer_tanaka/2026-01-04-001"
```

```json
{
    "memoryRecordSummaries": [
        {
            "content": {
                "text": "<topic name=\"配達状況の問い合わせ\">\n顧客が注文番号ABC-123の荷物が届いていないと報告している。\n</topic>\n<topic name=\"配送方法の要望\">\n顧客は今後の配送についてFedExを使用するよう要望し、アシスタントはこの要望を承諾した。\n</topic>"
            }
        }
    ]
}
```

### 5. 各 Strategy の出力形式の違い

同じ会話から、Strategy ごとに異なる形式・観点で情報が抽出されます：

| Strategy | 出力形式 | 抽出内容 |
|----------|----------|----------|
| **Semantic** | プレーンテキスト | 事実（何が起きたか） |
| **UserPreference** | JSON（context, preference, categories） | 好み（今後どうしてほしいか） |
| **Summary** | XML（topic タグで構造化） | 会話全体の概要 |

```
┌─────────────────────────────────────────────────────────────────┐
│ Semantic（事実抽出）                                             │
│ → シンプルなテキスト                                             │
│   "ユーザーの注文番号 ABC-123 の荷物が届いていない。"              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ UserPreference（好み抽出）                                       │
│ → JSON形式                                                      │
│   {                                                             │
│     "context": "The user explicitly requested...",              │
│     "preference": "Prefers FedEx for shipping",                 │
│     "categories": ["shipping", "delivery"]                      │
│   }                                                             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Summary（セッション要約）                                        │
│ → XML形式                                                       │
│   <topic name="配達状況の問い合わせ">                            │
│     顧客が注文番号ABC-123の荷物が届いていないと報告...            │
│   </topic>                                                      │
└─────────────────────────────────────────────────────────────────┘
```

## 全体像

```
Memory: customer_support_memory
│
├── Strategies:
│   ├── facts_extractor (SEMANTIC)
│   ├── preference_learner (USER_PREFERENCE)
│   └── session_summarizer (SUMMARIZATION)
│
├── Actor: customer_tanaka
│   │
│   ├── Session: 2026-01-04-001
│   │   └── Event: "注文番号 ABC-123 の荷物が届きません..."
│   │
│   └── Memory Records (自動抽出):
│       ├── /facts/customer_tanaka
│       │   ├── "注文番号 ABC-123 の荷物が届いていない"
│       │   └── "今後の配送をFedExで希望"
│       ├── /preferences/customer_tanaka
│       │   └── {"preference": "Prefers FedEx for shipping"}
│       └── /summaries/customer_tanaka/2026-01-04-001
│           └── <topic name="配達状況の問い合わせ">...</topic>
│
└── Actor: customer_suzuki
    │
    ├── Session: 2026-01-04-001
    │   └── Event: "商品Xを返品したいです..."
    │
    └── Memory Records (自動抽出):
        ├── /facts/customer_suzuki
        │   ├── "商品Xを返品希望（サイズが合わない）"
        │   └── "電話での連絡を希望"
        ├── /preferences/customer_suzuki
        │   └── {"preference": "メールより電話での連絡"}
        └── /summaries/customer_suzuki/2026-01-04-001
            └── ...
```

## API 一覧

### Control Plane（リソース管理）

| API | 用途 |
|-----|------|
| `create-memory` | Memory 作成 |
| `get-memory` | Memory 詳細取得 |
| `list-memories` | Memory 一覧 |
| `update-memory` | Memory 更新 |
| `delete-memory` | Memory 削除 |

### Data Plane（データ操作）

| API | 用途 |
|-----|------|
| `create-event` | 会話を保存（Short-term） |
| `list-events` | 生の会話履歴を取得 |
| `get-event` | 特定のイベント取得 |
| `list-memory-records` | 抽出された記録を一覧 |
| `retrieve-memory-records` | **セマンティック検索** |
| `list-sessions` | セッション一覧 |
| `list-actors` | アクター一覧 |

## まとめ

AWS Bedrock AgentCore Memory を使うと：

1. **会話の保存**（Short-term Memory）と**知識の抽出**（Long-term Memory）を自動化できる
2. **4つの戦略**（Semantic, UserPreference, Summary, Episodic）で抽出内容をカスタマイズできる
3. **複数の戦略を組み合わせて**、同じ会話から異なる観点で情報を抽出できる
4. **セマンティック検索**で自然言語クエリから関連情報を取得できる
5. `actorId` / `sessionId` / `namespace` でデータを論理的に分離できる

RAG システムの「ユーザーごとの記憶」部分を簡単に実装できるサービスですね。

### ID / Namespace の整理

| 概念 | 役割 | 例 |
|------|------|-----|
| **memory-id** | アプリケーション単位 | `customer_support_memory-xxx` |
| **actor-id** | ユーザー単位 | `customer_tanaka` |
| **session-id** | 会話単位 | `2026-01-04-001` |
| **namespace** | Long-term Memory の保存先 | `/facts/{actorId}` |

Namespace で `{actorId}` や `{sessionId}` を使うと、自動的に展開されてユーザーごと・セッションごとにデータが分離されます。

## 参考リンク

- [Amazon Bedrock AgentCore Memory - Developer Guide](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html)
- [Memory strategies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-strategies.html)
- [Built-in strategies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/built-in-strategies.html)
