---
title: "GCPのサーバーレス構成を「1モジュールずつ手元で再現」してキャッチアップする"
emoji: "🧩"
type: "tech"
topics: ["gcp", "terraform", "cloudrun", "eventarc", "iam"]
published: false
---

GCP をほとんど触ったことがない状態から、「ファイルを置くと自動で処理が走って DB に保存される」サーバーレス・イベント駆動な構成（Cloud Run / Eventarc / Workflows / Cloud SQL / BigQuery / IAP / Workload Identity Federation …）を理解したい。いきなり全体を読んでも頭に入らないので、取った方針が:

> **構成の各要素を「独立した最小の Terraform root」として、自分の sandbox プロジェクトに1つずつ作りながら学ぶ**

です。本記事はその記録と、GCP 初学者がハマった点・横断的に効いた学びのまとめです。AWS 経験はある前提で、ところどころ AWS と対比します。

## コード

完全なコードは GitHub に置いてあります。各ディレクトリが独立した Terraform root で、単体で `apply` できます。

https://github.com/ono-hiroki/maitake/tree/main/gcp-serverless-pipeline

```bash
git clone https://github.com/ono-hiroki/maitake.git
cd maitake/gcp-serverless-pipeline
```

## 題材アーキテクチャ

```
                ┌─────────────┐
  ファイル投入 → │Cloud Storage│
                └──────┬──────┘
                       │ オブジェクト作成イベント
                       ▼
                  ┌─────────┐      ┌───────────┐      ┌──────────────┐
                  │ Eventarc│ ───► │ Workflows │ ───► │ Cloud Run Job│
                  └─────────┘      └───────────┘      └──────┬───────┘
                                                             ├──► Cloud SQL
                                                             └──► BigQuery
  可視化デモ: Cloud Run Service + IAP認証
  CI/CD: Workload Identity Federation + GitHub Actions
```

これを 8 モジュールに分割し、依存の少ない方から積み上げました。

| # | 要素 | 学ぶ GCP |
|---|------|---------|
| 01 | network | VPC / Subnet / Firewall |
| 02 | bigquery | BigQuery Dataset / Table |
| 03 | firestore | Firestore（NoSQL） |
| 04 | cloudsql | Cloud SQL / Private Service Access / Secret Manager |
| 05 | application | Cloud Run Job / Artifact Registry / Service Account |
| 06 | ui-demo | Cloud Run Service / IAP |
| 07 | workflow | Eventarc / Workflows |
| 08 | cicd | Workload Identity Federation |

## 学習の進め方（ルール）

- **1ディレクトリ = 1つの独立した Terraform root**（`init/plan/apply` が単体完結）
- **state はローカル**（学習用なので GCS バックエンドは使わない）
- **API 有効化も Terraform 管理**（`google_project_service`）。どのサービスがどの API を要るかが見える
- **`terraform graph` で依存を可視化**（複雑さが edge 数で分かる）

## モジュールごとの学び

### 01 network — VPC はグローバル

AWS との最初の違い。**GCP の VPC はグローバル、サブネットがリージョンに紐づく**。

```hcl
resource "google_compute_network" "main" {
  name                    = "sbx-demo"
  auto_create_subnetworks = false  # ← カスタムモード VPC
}
```

`false` で「カスタムモード」（サブネットを自分で定義、本番推奨）、`true` だと全リージョンに自動作成（オートモード）。プロジェクト既定の `default` VPC がオートモードなので、並べると違いが一目瞭然です。

### 02 bigquery — サーバレス DWH、ID にハイフン不可

```hcl
resource "google_bigquery_dataset" "main" {
  dataset_id = replace("sbx-demo", "-", "_")  # dataset_id はハイフン不可
  location   = "asia-northeast1"               # 作成後変更不可
}
```

`bq query` でサンプルテーブルに INSERT/SELECT すると、インスタンス管理不要なサーバレス DWH の手触りが掴めます。

### 03 firestore — `gcloud firestore documents create` は無い

NoSQL ドキュメント DB。ハマったのは **ドキュメントの CRUD が gcloud に無い** こと。`gcloud firestore` のサブコマンドは databases / indexes / export などインフラ操作だけ。値の読み書きは REST API か SDK です。

```bash
TOKEN=$(gcloud auth print-access-token)
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  ".../databases/<DB>/documents/documents?documentId=doc-1" \
  -d '{"fields":{"title":{"stringValue":"テスト"}}}'
```

### 04 cloudsql — 秘密を tfstate に残さない設計

技術的には **Private Service Access（VPC ピアリング）** が肝。Cloud SQL は Google 管理 VPC で動くので、内部 IP で繋ぐには「ピアリング用 IP 範囲を予約 → Service Networking 接続を確立」します。

そして一番の学びが **「秘密を tfstate に持ち込まない」設計**:

- `google_secret_manager_secret`（入れ物）は作るが、**`secret_version`（値）は作らない**
- DB ユーザーのパスワードは初回ダミー値 + `lifecycle { ignore_changes = [password] }`

```hcl
resource "google_sql_user" "main" {
  password = "CHANGE_ME"                       # 初回ダミー
  lifecycle { ignore_changes = [password] }    # 以降TFは追跡しない
}
```

`ignore_changes` は「新しい値を追跡する」ではなく **「その項目をもう一切見ない」**。本物は apply 後に `gcloud` で手動投入し、TF はダミーを上書きしようとしない。結果として **本物のパスワードは一度も Terraform / tfstate を通らない**。

> 大原則: **Terraform を通った秘密は、リソース種別を問わず tfstate に平文で残る**。だから「本物を TF に渡さない」工夫が要る。

### 05 application — Cloud Run Job と FAIL_RATE でリトライ観察

Cloud Run **Job**（実行して完了するバッチ）。Google 公式サンプル Job イメージは `FAIL_RATE` / `SLEEP_MS` を読むので、**確率失敗 → 自動リトライ** を観察できます。

```bash
gcloud run jobs update <job> --update-env-vars FAIL_RATE=0.5 --tasks=4
gcloud run jobs execute <job> --wait
# → retriedCount: 1, succeededCount: 4（1タスク失敗→リトライで成功）
```

`image` と `env` は `ignore_changes` 対象にして「中身は CI に任せ、TF はインフラの骨組みだけ」を表現。

### 06 ui-demo — IAP の 403 を audit log で切り分ける

Cloud Run **Service** + **IAP**。ここが一番ハマりました。`iap_enabled = true`（google-beta 専用）＋ ユーザーに `roles/iap.httpsResourceAccessor` ＋ IAP サービスエージェントに `run.invoker`。設定は正しいのに **403「You don't have access」** が消えない。

効いたのが **Cloud Audit Logs の `authorizationInfo`**:

```bash
gcloud logging read 'protoPayload.serviceName="iap.googleapis.com"' \
  --format="value(protoPayload.authorizationInfo[0].granted,
    protoPayload.requestMetadata.requestAttributes.auth.principal)"
# granted=false, principal=(空)
```

`auth.principal` が**空**＝ IAP に認証セッションが届いていない（IAM 不足ではない）。IAP を一時的に外して `allUsers` 公開にすると 200 で表示できたので、**アプリ・IAM は正常で、止めていたのは IAP の認証セッション層**（サードパーティ Cookie ブロック / 組織のコンテキストアウェアアクセス等、Terraform の外）と確定。

> 「設定したのに 403」はまず audit log の `authorizationInfo` と `auth.principal` を見る。principal が空なら IAM ではなく認証セッション層を疑う。

### 07 workflow — なぜ Workflows を挟むのか

`GCS → Eventarc → Workflows → Cloud Run Job`。実際にバケットへファイルを置くと Job が起動するところまで動かしました。

**なぜ Eventarc から直接 Job を呼ばず Workflows を挟むのか?** ——

- Eventarc が直接呼べる宛先は **HTTP を受けるもの**（Cloud Run **Service** / Workflows / GKE …）
- Cloud Run **Job** は HTTP を受けない。起動には Jobs API（`jobs.run`）を**呼ぶ**必要がある
- だから「Jobs API を呼ぶ係」として Workflows を挟む

```
宛先が Service → Eventarc → Service（HTTP直送、Workflows不要）
宛先が Job     → Eventarc → Workflows →(Jobs API)→ Job
```

イベント駆動は**登場人物が多い**（トリガー SA・Eventarc サービスエージェント・GCS サービスエージェント…）。要素が多すぎるので、Workflows / Pub/Sub / Eventarc を**単体**で試す最小モジュールも `components/` に用意して、段階的に理解しました。

### 08 cicd — 鍵レス認証（WIF）、AWS との対応

**Workload Identity Federation**。GitHub Actions が **SA キー(JSON) を一切持たずに** OIDC で GCP 認証する仕組み。AWS の OIDC + AssumeRole とほぼ同じ発想です。

| 概念 | AWS | GCP（WIF） |
|---|---|---|
| issuer を信頼する登録 | IAM OIDC Identity Provider | Workload Identity Pool Provider |
| 受け入れ条件 | Role 信頼ポリシーの Condition | attribute_condition + principalSet |
| 成り代わる先の ID + 権限 | IAM Role | Service Account + IAM ロール |
| トークン交換 | sts:AssumeRoleWithWebIdentity | STS（sts.googleapis.com） |

違いは **AWS は 1 つの Role に信頼+権限が同居、GCP は Pool / Provider / SA に分割** される点。

最小の GitHub Actions で実動確認できます（**Secrets 設定ゼロ**。Provider パスと SA メールは秘密ではないのでベタ書きでよい）:

```yaml
permissions:
  id-token: write          # OIDC 発行に必須
steps:
  - uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: <output>
      service_account: <output>
  - run: gcloud auth list   # active が cicd SA なら成功
```

## 横断的に効いた「GCP のクセ」

モジュールを跨いで繰り返し出たので、これが GCP 全般の知識として一番の収穫でした。

### 1. IAM は結果整合（伝播待ち）

「権限を付けた直後はまだ効かない」場面に何度も遭遇:

| 場面 | 症状 | 対処 |
|---|---|---|
| IAP | 付与済みなのに 403 | 伝播待ち |
| Eventarc | サービスエージェント権限が間に合わず作成失敗 (400) | `time_sleep` |
| Workflows | logWriter 付与直後に 403 | 数分待って再実行 |

Terraform で「待つ」を表現するなら:

```hcl
resource "time_sleep" "wait" {
  depends_on      = [google_project_iam_member.eventarc_service_agent]
  create_duration = "120s"
}
# trigger 側で depends_on = [time_sleep.wait]
```

### 2. `deletion_protection` がデフォルト true のリソースが多い

`terraform destroy` が「`deletion_protection=false` にして apply してから」と弾く系。

| リソース | フィールド | 既定 |
|---|---|---|
| Cloud SQL | `deletion_protection` | true |
| Cloud Run Job/Service | `deletion_protection` | true |
| Workflows | `deletion_protection` | true |
| Firestore | `delete_protection_state` | ENABLED |
| BigQuery table | `deletion_protection` | true |

### 3. `ignore_changes` で「TF と運用の役割分担」

`image` / `env` / `password` を `ignore_changes` にして、**TF はインフラの骨組み、中身は CI や手動運用に任せる**。04 のパスワード、05/06 のイメージで同じパターンが出ます。

### 4. 依存グラフで複雑度を見える化

`terraform graph | dot -Tpng` で各モジュールの依存を可視化。edge 数がそのまま複雑度の目安に。イベント駆動（workflow, 17 edges）が突出して複雑＝「登場人物が多い」が数字で見えます。

## まとめ

- **実在しうる構成を題材に、1要素ずつ独立 root で再現**するのは、全体を一度に読むより圧倒的に頭に入った
- **手を動かして実機で確認**（FAIL_RATE のリトライ、IAP の 403、GCS 投入で Job 起動、WIF で実認証）すると、ドキュメントだけでは曖昧な挙動が確信に変わる
- **ハマりどころ（IAM 伝播・deletion_protection・秘密と tfstate・IAP のセッション層）こそ横断的な財産**。別サービスでも同じ顔で出てくる

知らないクラウドをキャッチアップするとき、いきなり全体と格闘せず **要素分解 → 最小再現 → 実機確認 → 統合理解** の順で登るのはおすすめです。コードは [maitake/gcp-serverless-pipeline](https://github.com/ono-hiroki/maitake/tree/main/gcp-serverless-pipeline) にあるので、手を動かして試してみてください。
