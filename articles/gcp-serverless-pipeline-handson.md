---
title: "GCPのサーバーレス・イベント駆動構成をTerraformで要素別に構築する"
emoji: "🧩"
type: "tech"
topics: ["gcp", "terraform", "cloudrun", "eventarc", "iam"]
published: false
---

GCP のサーバーレス・イベント駆動な構成（Cloud Storage → Eventarc → Workflows → Cloud Run Job → Cloud SQL / BigQuery、Cloud Run Service + IAP、CI/CD 用の Workload Identity Federation）を、各要素を独立した最小の Terraform root として構築する。本記事は各モジュールの構成と要点、および複数モジュールで共通して現れる GCP の挙動をまとめる。AWS と対応する箇所は対比を併記する。

## コード

各ディレクトリが独立した Terraform root で、単体で `apply` できる。

https://github.com/ono-hiroki/maitake/tree/main/gcp-serverless-pipeline

```bash
git clone https://github.com/ono-hiroki/maitake.git
cd maitake/gcp-serverless-pipeline
```

## アーキテクチャ

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
  Cloud Run Service + IAP認証
  Workload Identity Federation + GitHub Actions
```

8 モジュールに分割し、依存の少ない順に構築する。

| # | ディレクトリ | 学ぶ GCP |
|---|------|---------|
| 01 | network | VPC / Subnet / Firewall |
| 02 | bigquery | BigQuery Dataset / Table |
| 03 | firestore | Firestore（NoSQL） |
| 04 | cloudsql | Cloud SQL / Private Service Access / Secret Manager |
| 05 | cloudrun-job | Cloud Run Job / Artifact Registry / Service Account |
| 06 | cloudrun-service | Cloud Run Service / IAP |
| 07 | workflow | Eventarc / Workflows |
| 08 | cicd | Workload Identity Federation |

各モジュールは独立した root で、state はローカル、必要な API は `google_project_service` で各モジュールが有効化する。

## 01 network — VPC とサブネット

GCP の VPC はグローバル、サブネットはリージョンに紐づく（AWS の VPC はリージョン単位）。

```hcl
resource "google_compute_network" "main" {
  name                    = "sbx-demo"
  auto_create_subnetworks = false  # カスタムモード VPC
}
```

`auto_create_subnetworks = false` はカスタムモード（サブネットを明示定義）、`true` はオートモード（全リージョンに自動作成）。プロジェクト既定の `default` VPC はオートモード。

## 02 bigquery — Dataset / Table

```hcl
resource "google_bigquery_dataset" "main" {
  dataset_id = replace("sbx-demo", "-", "_")  # dataset_id はハイフン不可
  location   = "asia-northeast1"               # 作成後変更不可
}
```

BigQuery はインスタンス管理が不要で、保存量とスキャン量で課金される。`bq query` でテーブルへ INSERT / SELECT できる。

## 03 firestore — ドキュメント操作は REST API

NoSQL ドキュメント DB。`gcloud firestore` のサブコマンドは databases / indexes / export などインフラ操作のみで、ドキュメントの CRUD は含まれない。値の読み書きは REST API か各言語 SDK を使う。

```bash
TOKEN=$(gcloud auth print-access-token)
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  ".../databases/<DB>/documents/documents?documentId=doc-1" \
  -d '{"fields":{"title":{"stringValue":"テスト"}}}'
```

## 04 cloudsql — Private Service Access と秘密の扱い

Cloud SQL は Google 管理の VPC で動く。内部 IP で接続するには Private Service Access を構成する（ピアリング用 IP 範囲を予約 → Service Networking 接続を確立 → `private_network` 指定でインスタンス作成）。

秘密情報の扱いには次の構成を用いる。

- `google_secret_manager_secret`（入れ物）は作るが、`secret_version`（値）は Terraform で作らない
- DB ユーザーのパスワードは初回ダミー値 + `lifecycle { ignore_changes = [password] }`

```hcl
resource "google_sql_user" "main" {
  password = "CHANGE_ME"                       # 初回ダミー
  lifecycle { ignore_changes = [password] }    # 以降 TF は password を追跡しない
}
```

`ignore_changes` を指定した属性は、作成後の差分を Terraform が検知しない。本物のパスワードは apply 後に `gcloud` で投入し、Terraform はダミー値のまま保持する。Terraform を経由した値は tfstate に平文で保存されるため、この構成により本物のパスワードは tfstate に入らない。

## 05 cloudrun-job — Cloud Run Job

Cloud Run Job はリクエストを受けず、実行して完了するバッチ。Google 公式のサンプル Job イメージは環境変数 `FAIL_RATE` / `SLEEP_MS` を読む。

```bash
gcloud run jobs update <job> --update-env-vars FAIL_RATE=0.5 --tasks=4
gcloud run jobs execute <job> --wait
# 結果例: retriedCount: 1, succeededCount: 4
```

`FAIL_RATE` で一定確率でタスクが失敗し、`max_retries` の範囲でリトライされる。`image` と `env` は `ignore_changes` 対象にし、イメージ更新は CI 側に委ねる。

## 06 cloudrun-service — Cloud Run Service と IAP

Cloud Run Service は HTTP リクエストを受け続ける常駐コンテナ。`iap_enabled = true`（google-beta プロバイダ）で前段に IAP が入る。アクセス制御に必要な IAM は 2 つ。

- ユーザー / グループに `roles/iap.httpsResourceAccessor`
- IAP サービスエージェントに `roles/run.invoker`

IAM を正しく設定しても 403 になる場合がある。切り分けには Cloud Audit Logs の `authorizationInfo` を参照する。

```bash
gcloud logging read 'protoPayload.serviceName="iap.googleapis.com"' \
  --format="value(protoPayload.authorizationInfo[0].granted,
    protoPayload.requestMetadata.requestAttributes.auth.principal)"
```

`granted=false` かつ `auth.principal` が空の場合、IAM ロール不足ではなく、リクエストに認証セッションが付いていない（サードパーティ Cookie のブロックや組織のコンテキストアウェアアクセス等、Terraform の管理外の要因）。IAP を無効化して `allUsers` 公開にするとアプリ自体の動作を切り分けられる。

## 07 workflow — Eventarc と Workflows

`GCS → Eventarc → Workflows → Cloud Run Job` を構成する。バケットへファイルを置くと Workflows が起動し、Cloud Run Job が実行される。

Eventarc の宛先（destination）に直接指定できるのは HTTP を受けるもの（Cloud Run Service / Workflows / GKE）。Cloud Run Job は HTTP を受けず、起動には Jobs API（`jobs.run`）の呼び出しが必要なため、Eventarc から直接は起動できない。Workflows を経由して Jobs API を呼ぶ。

```
宛先が Cloud Run Service → Eventarc → Service（HTTP 直送）
宛先が Cloud Run Job     → Eventarc → Workflows →(Jobs API)→ Job
```

Eventarc の GCS トリガーは内部で Pub/Sub を使う。構成には複数の ID が必要（トリガー SA、Eventarc サービスエージェント、GCS サービスエージェントの `pubsub.publisher`）。`components/` に Workflows / Pub/Sub / Eventarc を単体で構築する最小モジュールを置いている。

## 08 cicd — Workload Identity Federation

GitHub Actions が Service Account キー（JSON）を使わず、OIDC トークンで GCP に認証する。AWS の OIDC Identity Provider + AssumeRoleWithWebIdentity に対応する。

| 概念 | AWS | GCP（WIF） |
|---|---|---|
| issuer を信頼する登録 | IAM OIDC Identity Provider | Workload Identity Pool Provider |
| 受け入れ条件 | Role 信頼ポリシーの Condition | attribute_condition + principalSet |
| 成り代わる先の ID + 権限 | IAM Role | Service Account + IAM ロール |
| トークン交換 | sts:AssumeRoleWithWebIdentity | STS（sts.googleapis.com） |

AWS は 1 つの Role に信頼と権限が同居する。GCP は Pool / Provider / SA に分かれ、「誰が impersonate してよいか（principalSet への `workloadIdentityUser`）」と「その SA が何をできるか（project ロール）」が別リソースになる。

GitHub Actions 側は Secrets 不要（Provider パスと SA メールは秘密情報ではない）。

```yaml
permissions:
  id-token: write
steps:
  - uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: <output>
      service_account: <output>
  - run: gcloud auth list   # active が CI/CD SA になる
```

## 複数モジュールで共通する GCP の挙動

### IAM は結果整合（伝播待ち）

権限付与の反映には時間差がある。付与直後はエラーになる場合がある。

| 場面 | 症状 |
|---|---|
| IAP | 付与済みでも 403 |
| Eventarc | サービスエージェント権限の反映前にトリガー作成すると 400 |
| Workflows | logWriter 付与直後の実行で 403 |

Terraform では `time_sleep` で待機を挟む。

```hcl
resource "time_sleep" "wait" {
  depends_on      = [google_project_iam_member.eventarc_service_agent]
  create_duration = "120s"
}
```

### deletion_protection がデフォルト true のリソース

`terraform destroy` の前に `deletion_protection = false` を反映する必要がある。

| リソース | フィールド | 既定 |
|---|---|---|
| Cloud SQL | `deletion_protection` | true |
| Cloud Run Job / Service | `deletion_protection` | true |
| Workflows | `deletion_protection` | true |
| Firestore | `delete_protection_state` | ENABLED |
| BigQuery table | `deletion_protection` | true |

### ignore_changes

`image` / `env` / `password` を `ignore_changes` にすると、Terraform はそれらの差分を検知しない。インフラ定義は Terraform、コンテナイメージや環境変数の更新は CI、という分担に使う。

### terraform graph

`terraform graph | dot -Tpng` で依存を可視化できる。edge 数はモジュールの依存の多さを示す（例: イベント駆動の workflow は 17 edges）。

## まとめ

GCP のサーバーレス・イベント駆動構成を、VPC / BigQuery / Firestore / Cloud SQL / Cloud Run（Job・Service）/ IAP / Eventarc / Workflows / Workload Identity Federation の各要素に分けて Terraform で構築した。コードは [maitake/gcp-serverless-pipeline](https://github.com/ono-hiroki/maitake/tree/main/gcp-serverless-pipeline) にある。
