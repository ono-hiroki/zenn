---
title: "Strands AgentsをAmazon Bedrock AgentCore Runtimeにデプロイする"
emoji: "🚀"
type: "tech"
topics: ["strandsagents","aws","bedrock","python","agentcore"]
published: false
---


# Strands AgentsをAmazon Bedrock AgentCore Runtimeにデプロイしてみた

## はじめに

本記事は、Strands Agents公式ドキュメントの[Python Deployment to Amazon Bedrock AgentCore Runtime](https://strandsagents.com/latest/documentation/docs/user-guide/deploy/deploy_to_bedrock_agentcore/python/)に沿って、実際にデプロイを試した記録です。

Amazon Bedrock AgentCore Runtimeは、AIエージェントをAWS上でホスティングするためのマネージドサービスです。Strands Agentsフレームワークを使用してシンプルなエージェントを作成し、AgentCore Runtimeにデプロイする手順を紹介します。

## 前提条件

- Python 3.10以上
- AWSアカウントと適切なIAM権限
- AWS CLI設定済み
- uv（Pythonパッケージマネージャー）

## デプロイ方法の選択

AgentCore Runtimeへのデプロイには2つのアプローチがあります：

| 方法 | 特徴 | 用途 |
|------|------|------|
| **Option A: SDK Integration** | 自動HTTPサーバー設定、デプロイツール内蔵 | シンプルなエージェント、プロトタイピング |
| **Option B: Custom Agent** | FastAPIで完全制御、カスタムルーティング | 複雑なエージェント、本番システム |

今回は**Option A: SDK Integration**を使用します。

## セットアップ〜デプロイ

### Step 1: プロジェクトの作成

```bash
mkdir strands-agentcore && cd strands-agentcore
uv init --python 3.11
```

### Step 2: 依存関係のインストール

```bash
uv add strands-agents bedrock-agentcore bedrock-agentcore-starter-toolkit
```

インストールされる主要パッケージ：
- `strands-agents`: Strands Agentsフレームワーク（LLM呼び出し）
- `bedrock-agentcore`: AgentCore Runtime SDK（HTTPサーバー）
- `bedrock-agentcore-starter-toolkit`: デプロイ自動化ツール

### Step 3: エージェントコードの作成

`agent_example.py`を作成します：

```python
from strands import Agent
from bedrock_agentcore.runtime import BedrockAgentCoreApp

agent = Agent()
app = BedrockAgentCoreApp()


@app.entrypoint
def invoke(payload):
    """Process user input and return a response"""
    user_message = payload.get("prompt", "Hello")
    response = agent(user_message)
    return str(response)


if __name__ == "__main__":
    app.run()
```

### Step 4: `__init__.py`の作成

Pythonパッケージとして認識させるため、空の`__init__.py`を作成します：

```bash
touch __init__.py
```

### Step 5: ローカルテスト

エージェントを起動：

```bash
AWS_PROFILE=your_profile uv run python agent_example.py
```

別ターミナルでテスト：

```bash
curl -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"prompt":"Hello"}'
```

レスポンス例：
```
"Hello! How are you doing today? Is there anything I can help you with?"
```

### Step 6: agentcore configure

```bash
agentcore configure --entrypoint agent_example.py
```

対話形式で以下を設定：

```
Agent name [agent_example]: （Enter）
Select deployment type:
  1. Direct Code Deploy (recommended)
  2. Container
Choice [1]: 1

Select Python runtime version:
Choice [2]: 2  # PYTHON_3_11

Execution role ARN/name: （Enter）  # 自動作成
S3 URI/path: （Enter）              # 自動作成

Memory Configuration:
Your choice: （Enter）              # 新規作成
Enable long-term memory? [no]: （Enter）
```

### Step 7: agentcore launch

```bash
agentcore launch
```

デプロイには数分かかります。完了時の出力：
```
✅ Deployment completed successfully
Agent ARN: arn:aws:bedrock-agentcore:ap-northeast-1:XXXXXXXXXXXX:runtime/agent_example-XXXXXXXXXX
```

### Step 8: デプロイ後のテスト

```bash
agentcore invoke '{"prompt": "こんにちは"}'
```

レスポンス：
```
Response:
こんにちは! Nice to meet you. Are you interested in practicing Japanese,
or is there something specific I can help you with today?
```

## 運用コマンド

```bash
# ステータス確認
agentcore status

# エージェント呼び出し
agentcore invoke '{"prompt": "Hello"}'

# ログ確認（リアルタイム）
aws logs tail /aws/bedrock-agentcore/runtimes/agent_example-XXXXXXXXXX-DEFAULT \
  --log-stream-name-prefix "$(date +%Y/%m/%d)/[runtime-logs" --follow

# リソース削除
agentcore destroy
```

## Observabilityダッシュボード

CloudWatchのGenAI Observabilityダッシュボードで監視できます：

https://console.aws.amazon.com/cloudwatch/home?region=ap-northeast-1#gen-ai-observability/agent-core

※ データが表示されるまで最大10分かかる場合があります。

---

## 補足: SDKアーキテクチャ

`bedrock-agentcore`と`strands-agents`は異なる役割を持っています：

```
┌─────────────────────────────────────────────────────────────┐
│  クライアント (curl / agentcore invoke)                      │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│  bedrock-agentcore SDK (BedrockAgentCoreApp)                │
│  - POST /invocations  ← エージェント呼び出し                 │
│  - GET /ping          ← ヘルスチェック                       │
│  - WebSocket /ws      ← WebSocket通信                       │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│  strands-agents (Agent)                                     │
│  - Amazon Bedrockモデル呼び出し                              │
│  - ツール実行                                                │
└─────────────────────────────────────────────────────────────┘
```

### Strandsなしでも動作可能

`bedrock-agentcore`はHTTPサーバーのラッパーなので、Strandsなしでもエージェントを作成できます：

```python
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    user_message = payload.get("prompt", "")
    return {"message": f"あなたは「{user_message}」と言いました"}

if __name__ == "__main__":
    app.run()
```

| パターン | 使用ライブラリ |
|---------|--------------|
| 固定値を返す | `bedrock-agentcore`のみ |
| 外部API呼び出し | `bedrock-agentcore` + `requests` |
| LLM（Bedrock）を使う | `bedrock-agentcore` + `strands-agents` |

## 補足: @app.entrypointの詳細

### 単一エンドポイント設計

AgentCore SDKは「1エージェント = 1エンドポイント」の設計です。`@app.entrypoint`を複数定義すると、最後に定義した関数だけが有効になります。

### 引数

```python
# パターン1: payloadのみ（基本）
@app.entrypoint
def invoke(payload):
    user_message = payload.get("prompt")
    return {"response": "..."}

# パターン2: payload + context（拡張）
@app.entrypoint
def invoke(payload, context):
    session_id = context.session_id
    headers = context.request_headers
    return {"response": "..."}
```

第2引数は必ず`context`という名前にする必要があります。

### 他のデコレータ

| デコレータ | 用途 |
|-----------|------|
| `@app.entrypoint` | メインエンドポイント（/invocations） |
| `@app.ping` | カスタムヘルスチェック（/ping） |
| `@app.websocket` | WebSocketハンドラー（/ws） |
| `@app.async_task` | 非同期タスク追跡 |

複数のルーティングが必要な場合はOption B（FastAPI）を使う必要があります。

## 補足: 設定ファイル

`agentcore configure`実行後、`.bedrock_agentcore.yaml`が生成されます：

```yaml
agents:
  agent_example:
    deployment_type: direct_code_deploy
    runtime_type: PYTHON_3_11
    aws:
      region: ap-northeast-1
      network_configuration:
        network_mode: PUBLIC
    memory:
      mode: STM_ONLY
      event_expiry_days: 30
```

| 項目 | 説明 |
|------|------|
| `deployment_type` | `direct_code_deploy`（Dockerなし）または`container` |
| `runtime_type` | Python 3.10〜3.13から選択 |
| `network_mode` | `PUBLIC`または`VPC` |
| `memory.mode` | `STM_ONLY`（短期記憶）または`STM_AND_LTM`（長期記憶含む） |

設定を変更する場合は、YAMLを編集後に`agentcore launch`を実行します。

## 補足: メモリ機能

AgentCore Memoryは2層構造のメモリシステムを提供します。

### STM（短期記憶）とLTM（長期記憶）

```
┌─────────────────────────────────────────────────────────────┐
│  Short-Term Memory (STM) - 短期記憶                         │
│  • セッション内の会話履歴を保持                              │
│  • 即座に利用可能、30日間保持                                │
└─────────────────────────────────────────────────────────────┘
                  ↓ 非同期抽出（LTM有効時）
┌─────────────────────────────────────────────────────────────┐
│  Long-Term Memory (LTM) - 長期記憶                          │
│  • セッション間で永続化                                     │
│  • ユーザーの好み・事実・パターンを自動抽出                 │
└─────────────────────────────────────────────────────────────┘
```

### セッションIDの仕組み

```bash
# セッションID未指定 → 自動生成
agentcore invoke '{"prompt": "私は田中です"}'
# → Session: ef57535e-b8b5-4f40-9bf1-5226fee4ff3d

# 同じセッションIDで継続 → 会話を覚えている
agentcore invoke '{"prompt": "私の名前は？"}' --session-id ef57535e-...
# → 「田中さんですね」

# 別のセッションID → 覚えていない
agentcore invoke '{"prompt": "私の名前は？"}' --session-id new-session-xxx...
# → 「お名前をお聞きしていません」
```

セッションIDは33文字以上が必要です（UUID形式推奨）。

※ 本記事ではLTM（長期記憶）は試していません。詳細は[公式ドキュメント](https://aws.amazon.com/blogs/machine-learning/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/)を参照してください。

## 補足: リソース管理

### agentcore CLIはCloudFormationを使用しない

`agentcore` CLIは直接AWS APIを呼び出してリソースを作成します。状態管理はローカルの`.bedrock_agentcore.yaml`ファイルで行われます。

作成されるリソース：
- IAM Role
- S3 Bucket
- AgentCore Runtime
- Memory
- CloudWatch Logs

### カスタムドメインの設定

デフォルトではエージェント作成時にユニークなIDが付与されます。固定のカスタムドメインを使用したい場合は、**CloudFront + Route 53 + ACM**で設定できます。

詳細は[AWS公式ブログ](https://aws.amazon.com/blogs/machine-learning/set-up-custom-domain-names-for-amazon-bedrock-agentcore-runtime-agents/)を参照してください。

## まとめ

Strands AgentsをAmazon Bedrock AgentCore Runtimeにデプロイする手順を紹介しました。

**メリット：**
- 数コマンドでデプロイ可能
- Dockerなしで直接コードをデプロイ
- メモリ機能・Observabilityが組み込み

**デメリット：**
- CloudFormation/Terraformでの管理ができない
- プロトタイピング向け

## 参考リンク

### 公式ドキュメント
- [Strands Agents Documentation](https://strandsagents.com/)
- [Amazon Bedrock AgentCore Runtime Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore.html)
- [Deploy to Bedrock AgentCore (Python)](https://strandsagents.com/latest/documentation/docs/user-guide/deploy/deploy_to_bedrock_agentcore/python/)

### メモリ機能
- [Amazon Bedrock AgentCore Memory: Building context-aware agents](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-agentcore-memory-building-context-aware-agents/)
- [Building smarter AI agents: AgentCore long-term memory deep dive](https://aws.amazon.com/blogs/machine-learning/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/)

### カスタムドメイン
- [Set up custom domain names for Amazon Bedrock AgentCore Runtime agents](https://aws.amazon.com/blogs/machine-learning/set-up-custom-domain-names-for-amazon-bedrock-agentcore-runtime-agents/)
