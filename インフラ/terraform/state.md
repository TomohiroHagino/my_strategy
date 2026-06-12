# State とリモートbackend（Terraform）

## ひとことで言うと
**State（`terraform.tfstate`）** ＝ 「コードに書いたリソース」と「現実のインフラ」を結ぶ**対応表**。Terraformはこれを基に差分を計算する。チームで使うなら、stateを共有・ロックできる **リモートbackend**（S3+DynamoDB / GCS / Terraform Cloud）に置くのが必須。

## 役割・なぜ必要か
- Terraformは `apply` のたびにクラウドAPIを全件問い合わせるのではなく、**前回の結果＝state**と現状を突き合わせて差分を出す。stateが無いと「このリソースは自分が作ったものか」を判断できない。
- stateには**リソースの属性が丸ごと記録される**。そこにはDBパスワード・接続文字列など**秘密情報が平文で含まれうる**。だから **stateはコミット禁止・暗号化必須**。
- ローカルstate（手元の `terraform.tfstate`）は1人なら良いが、**2人が同時に `apply` するとstateが壊れる**。リモートbackend＋**ロック**で「同時に1人だけapplyできる」を保証する。
- 現実とstateがズレることを **drift（ドリフト）** と呼ぶ。誰かがコンソールで手動変更すると発生する。

## 基本の書き方（コード）
リモートbackend（AWS：S3にstate保存 + DynamoDBでロック）：
```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-tfstate"        # stateを置くS3バケット（事前に作成）
    key            = "prod/network/terraform.tfstate"  # バケット内のパス（環境/用途で分ける）
    region         = "ap-northeast-1"
    dynamodb_table = "terraform-locks"          # ロック用DynamoDBテーブル
    encrypt        = true                       # 保存時暗号化（必須）
  }
}
```

GCSの場合（GCP）：
```hcl
terraform {
  backend "gcs" {
    bucket = "mycompany-tfstate"
    prefix = "prod/network"   # オブジェクトのプレフィックス
  }
}
```

backend自体（バケット・ロックテーブル）は「卵が先か」問題があるので、最初だけ手動 or 別の小さなTerraformで用意する：
```hcl
# bootstrap 用（ローカルstateで一度だけ apply して backend の器を作る）
resource "aws_s3_bucket" "tfstate" {
  bucket = "mycompany-tfstate"
}
resource "aws_s3_bucket_versioning" "tfstate" {   # 履歴を残す＝事故復旧用
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration { status = "Enabled" }
}
resource "aws_dynamodb_table" "locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute { name = "LockID"; type = "S" }
}
```

state操作コマンド（直接編集は厳禁、必ずこのコマンド経由）：
```bash
terraform state list                      # 管理下のリソース一覧
terraform state show aws_instance.web     # 1リソースの中身を表示
terraform state mv aws_instance.web \
  module.app.aws_instance.web             # 構成変更時にstate上の住所を引っ越し
terraform state rm aws_instance.web       # 管理対象から外す（現実は消さない）
terraform import aws_instance.web i-0123   # 既存リソースをstateに取り込む

terraform refresh                          # 現実をstateに反映（driftの取り込み。plan -refresh-only推奨）
terraform plan -refresh-only               # driftだけを安全に確認
```

## 実務での使い方・定番パターン
- **チーム開発に入った瞬間にリモートbackend化**。S3+DynamoDB（AWS）か GCS（GCP）が定番。`encrypt = true` とバージョニング有効化はセット。
- **state を環境/用途で分割**（`prod/network`、`prod/app`、`dev/network` …）。`key` を分けて**1ファイルを巨大にしない**。巨大stateは `plan` が遅く、事故範囲も広い。
- **ロック必須**：DynamoDB（S3 backend）や GCSの世代制御で、同時applyを防ぐ。`Error acquiring the state lock` が出たら誰かが実行中。
- **既存インフラの取り込みは `import`**。手で作ったリソースをTerraform管理下に移すときに使う（コード側にも対応するresourceブロックを書いておく）。
- **構成のリファクタ（モジュール化など）でリソースの「住所」が変わる時は `state mv`**。そうしないとTerraformが「古いのを削除して新しいのを作成」と誤判断する。

## ハマりどころ / アンチパターン
- **`*.tfstate` をgitにコミット**：秘密情報の漏洩＋チームで競合。**絶対にgitignore**。リモートbackend＋暗号化が正解。→ [getting_started.md](./getting_started.md)
- **stateを手で書き換え**：JSONを直接編集して整合性崩壊。必ず `terraform state` コマンド経由。
- **ロック無しで同時apply**：stateが破損し、リソースの二重作成や行方不明が起きる。DynamoDB等のロックを必ず設定。
- **手動変更によるdrift放置**：コンソールでSGを書き換え→次の `apply` でTerraformが元に戻す（または競合）。手動変更は避け、変更はコード経由に統一。検知は `plan -refresh-only`。
- **巨大なモノリシックstate**：全社のインフラを1stateに。`plan` が数分かかり、1つの操作ミスが全体に波及。早期に分割。
- **backendバケットを消す/権限喪失**：stateの置き場を失うと管理不能に。バージョニング＋アクセス制御＋バックアップで守る。

## 関連
[getting_started.md](./getting_started.md) / [commands.md](./commands.md) / [environments.md](./environments.md) / [pitfalls.md](./pitfalls.md) / [../aws/s3.md](../aws/s3.md) / [../gcp/cloud_storage.md](../gcp/cloud_storage.md)
