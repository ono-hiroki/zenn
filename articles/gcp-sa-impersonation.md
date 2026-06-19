---
title: "個人権限でのterraform applyをやめる — Service Accountインパーソネート最小ハンズオン"
emoji: "🎭"
type: "tech"
topics: ["gcp", "terraform", "iam", "serviceaccount", "security"]
published: false
---

`terraform apply` を**自分の強い権限（ADC）のまま**実行していないだろうか。これを「terraform 実行用の Service Account（SA）を impersonate（なりすまし）して apply する」状態へ移す流れを、素の Terraform + 環境変数 1 つの最小構成でなぞる。

実務では Terragrunt などで provider/backend を生成してこの切替を行うことが多いが、ここでは本質だけを残す。コードは以下にある。

https://github.com/ono-hiroki/maitake/tree/main/gcp-sa-impersonation

```bash
git clone https://github.com/ono-hiroki/maitake.git
cd maitake/gcp-sa-impersonation
```

| ディレクトリ | 実行する権限 | 内容 |
|---|---|---|
| 01-bootstrap-sa | **個人 ADC**（ブートストラップ） | 実行用 SA を作る + 自分に tokenCreator を付与 |
| 02-impersonated-apply | **実行用 SA**（なりすまし） | SA を impersonate してバケットを作る + whoami で確認 |

以降、プロジェクトは `my-sandbox`、個人アカウントは `you@example.com` と表記する。

## なぜ impersonate するのか

`terraform apply` を個人 ADC で実行すると、その人が持つ全権限で全リソースが操作される。これは次の問題を抱える。

- 強い権限を**人間に常時持たせる**ことになる（漏洩時の被害が大きい）
- 「誰が apply できるか」と「apply 時に何ができるか」が個人アカウントに同居し、分離できない

impersonate は、権限を**実行用 SA に集約**し、人間には「その SA を借りてよい」という鍵（`roles/iam.serviceAccountTokenCreator`）だけを渡す。人間自身は強権限を持たず、apply のときだけ短命トークンで SA を借りる。

```
従来:  人間（storage.admin など強権限を直接保持）── apply ──▶ GCP
impersonate:
       人間（tokenCreator だけ）─借用─▶ 実行用SA（storage.admin）── apply ──▶ GCP
```

## トークンの仕組み

impersonate の実体は、`iamcredentials.googleapis.com` の `generateAccessToken` で**短命のアクセストークンを発行してもらう**こと。これを呼べるのは、対象 SA に対して `roles/iam.serviceAccountTokenCreator` を持つ主体だけ。この権限がないと、環境変数を立てても次のエラーになる。

```
Permission denied on iamcredentials.googleapis.com (generateAccessToken)
```

つまり impersonate に必要なのは 3 点セット。

1. なりすます相手（実行用 SA）
2. その SA に付けた作業ロール（ここでは `roles/storage.admin`）
3. 自分に付けた `roles/iam.serviceAccountTokenCreator`（借りる鍵）

## 卵が先か鶏が先か（ブートストラップ）

ここで素朴な疑問が出る。**SA を impersonate して apply したいなら、その SA を作る apply も impersonate すべきでは？**

できない。SA も tokenCreator もまだ存在しない段階では「借りる相手がいない／借りる鍵がない」ので impersonate は失敗する。だから **SA を生む最初の一回だけ**は個人権限で行う。これがブートストラップ問題で、`01-bootstrap-sa` を個人 ADC で apply する理由。

```hcl
# 01-bootstrap-sa/main.tf（抜粋）

# ① なりすます相手
resource "google_service_account" "tf_runner" {
  account_id   = "lab-tf-runner"
  display_name = "Lab Terraform Runner"
}

# ② SA に作業ロール（02 でバケットを作るので storage.admin）
resource "google_project_iam_member" "tf_runner_roles" {
  for_each = toset(["roles/storage.admin"])
  project  = var.project_id
  role     = each.value
  member   = "serviceAccount:${google_service_account.tf_runner.email}"
}

# ③ 自分に「この SA を借りてよい」鍵
resource "google_service_account_iam_member" "token_creator" {
  service_account_id = google_service_account.tf_runner.name
  role               = "roles/iam.serviceAccountTokenCreator"
  member             = "user:you@example.com"
}
```

apply 後、まず**「そもそも借りられるか」**を単体で確認する。成功すればトークン文字列が返る。

```bash
gcloud auth print-access-token \
  --impersonate-service-account=lab-tf-runner@my-sandbox.iam.gserviceaccount.com
```

> `tokenCreator` の付与は反映に数十秒〜数分かかる（IAM は結果整合）。`PERMISSION_DENIED` なら少し待って再実行。

## 切替の確認は data source の whoami で

`02-impersonated-apply` の肝は、**provider の HCL を一切変えない**こと。impersonate を書いていない provider のまま、環境変数だけで実行 ID が切り替わることを `data.google_client_openid_userinfo`（terraform の「今の自分」を返す = whoami）で目視確認する。

```hcl
# 02-impersonated-apply/main.tf（抜粋）
provider "google" {
  project = var.project_id
  region  = var.region
  # impersonate は書かない。切替は環境変数に任せる。
}

# whoami: 環境変数の有無で email が「あなた」⇄「SA」に変わる
data "google_client_openid_userinfo" "me" {}

output "whoami_email" {
  value = data.google_client_openid_userinfo.me.email
}
```

### before / after

```bash
# A) 変数なし（＝今までどおり個人 ADC）
unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
terraform apply
# → whoami_email = "you@example.com"                                    ← あなた個人

# B) 変数あり（＝SA を impersonate）
export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=lab-tf-runner@my-sandbox.iam.gserviceaccount.com
terraform apply
# → whoami_email = "lab-tf-runner@my-sandbox.iam.gserviceaccount.com"   ← SA！
```

`whoami_email` が **個人 → SA** に変われば、「apply 時に個人権限を使わず、SA をなりすまして実行する」状態が達成できている。バケット作成が成功するのは、SA が ① で付けた `storage.admin` を持っているから = SA の権限で動いている証拠でもある。

## impersonate の効かせ方は 3 つある

混乱しやすいので整理する。同じ「impersonate」でも効く相手とツールが違う。

| 方法 | 何に効くか | 切替操作 |
|---|---|---|
| ① gcloud CLI フラグ/config | `gcloud` コマンド（list/describe/create…） | `--impersonate-service-account=...` |
| ② ADC ファイルに焼き込み | ADC を読む全ツール（terraform 含む） | `gcloud auth application-default login --impersonate-service-account=...` |
| ③ 環境変数 | provider と GCS backend（terraform） | `export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=...` |

### ① は terraform には効かない

`gcloud config set auth/impersonate_service_account` などは `gcloud` コマンドを SA で走らせるが、**terraform には効かない**。terraform は gcloud の設定を見ず、ADC・環境変数・provider 設定だけを見るため。

| ツール | ①が効く？ |
|---|---|
| `gcloud` コマンド全般 | ⭕ |
| `terraform apply` / `plan` | ❌（③ or ② を使う） |

①の使いどころは「借用可否の確認」(`gcloud auth print-access-token --impersonate-service-account=...`)。

### ② と ③ の違い、そして戻し方の罠

②も③も terraform に効くが、impersonate 先の**保持場所**が違う。

| | ② ADC 焼き込み | ③ 環境変数 |
|---|---|---|
| 保持場所 | ADC ファイル | 環境変数（ADC は個人のまま） |
| 切替 | SA を変えるたび再ログイン | 変数の値を変えるだけ |
| multi-env(dev/prd) | 都度ログインで手間 | 変数差し替えで楽 |

ここが一番ハマる。**戻し方が違う。**

| 効かせた方法 | 戻し方 |
|---|---|
| ③ 環境変数 | `unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` |
| ② ADC 焼き込み | `gcloud auth application-default login`（フラグなし再ログイン） |

②を使ったのに `unset`（③の戻し方）をしても、ADC ファイルに焼き込まれたままなので**SA のまま戻らない**。今どちらが効いているかは ADC ファイルで確認できる。

```bash
grep -i impersonat ~/.config/gcloud/application_default_credentials.json
# service_account_impersonation_url が出る → ②が効いている = 再ログインで戻す
```

multi-env で SA を頻繁に差し替えるなら、再ログイン不要の **③（環境変数）が扱いやすい**。

## provider だけでなく backend も切り替わる

実運用では state を GCS backend に置く。このとき provider（リソース操作）だけ impersonate しても、**backend（state 読み書き）が個人権限のまま**だと中途半端になる。

環境変数 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` の利点は、**provider と GCS backend の両方が尊重する**こと。変数 1 つで実行 ID を丸ごと切り替えられる。HCL に直書きする方式（`impersonate_service_account = ...`）だと provider にしか効かず、backend には別途設定が要る。

```hcl
# 直書きの場合（provider にしか効かない）
provider "google" {
  impersonate_service_account = "lab-tf-runner@my-sandbox.iam.gserviceaccount.com"
}
# → backend(GCS) 側にも別の impersonate 設定が必要
```

本ラボはローカル state なので backend は対象外だが、実運用に持っていくなら環境変数方式がシンプル。

## Workload Identity Federation との関係

CI/CD（GitHub Actions など）から GCP を触るときは、SA キー（JSON）を使わず Workload Identity Federation（WIF）で OIDC 認証する。この WIF も「principalSet が SA を impersonate する」構造で、本質は同じ短命トークンの借用だ。

- **ローカルの人間** → ADC + tokenCreator で SA を impersonate（本記事）
- **CI/CD** → WIF（OIDC）で SA を impersonate（キーレス）

どちらも「人間/CI に強権限を直接持たせず、SA に集約して借りる」という同じ思想。WIF 側のハンズオンは [maitake/gcp-terraform-handson/workload-identity-federation](https://github.com/ono-hiroki/maitake/tree/main/gcp-terraform-handson/workload-identity-federation) にある。

## まとめ

- `terraform apply` を個人 ADC で実行する状態から、実行用 SA を impersonate する状態へ移す
- 必要なのは 3 点セット（SA / 作業ロール / 自分への tokenCreator）。SA を作る最初の apply だけは個人権限（ブートストラップ）
- impersonate の効かせ方は ①gcloud CLI ②ADC 焼き込み ③環境変数 の 3 つ。terraform に効くのは ②③。①は terraform に効かない
- ②と③は**戻し方が違う**（③は unset、②は再ログイン）。multi-env なら ③ が扱いやすい
- 環境変数方式は provider と GCS backend を一括で切り替えられる

コードは [maitake/gcp-sa-impersonation](https://github.com/ono-hiroki/maitake/tree/main/gcp-sa-impersonation) にある。
