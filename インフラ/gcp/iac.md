# IaC（Infrastructure as Code）（GCP）

## ひとことで言うと
**Infrastructure as Code＝インフラをコードで定義・管理する**やり方。VM・ネットワーク・IAM・データベースなどを画面のクリックではなく**設定ファイル**で宣言し、そのファイルから自動でリソースを作る・変える・消す。GCPでは **Terraform** が事実上の定番。

## 役割・なぜ必要か
- **手動（gcloud / コンソール）の限界**：誰がいつ何を変えたか残らない、同じ環境（本番/検証）を再現しづらい、設定ミスが起きやすい。
- IaCならインフラ定義が **Gitで履歴管理・レビュー・差し戻し**できる（アプリのコードと同じ運用に乗る）。
- **冪等（べきとう）性**：同じファイルから何度適用しても結果は同じ状態に収束する。「あるべき姿」を書いておけば、それに合わせてくれる。
- 主な選択肢：
  - **Terraform**（HashiCorp）… マルチクラウド対応・情報量が多く、GCPでも最有力。HCLという言語で書く。
  - **Deployment Manager**（GCP純正）… YAML/Python/Jinja。GCP専用で、近年は新規採用が減りTerraformに寄っている。
  - **Config Connector**（GKE上のKubernetes流儀）… GCPリソースをk8sのマニフェストとして宣言・管理する。k8s中心の組織向け。

## 基本の使い方（gcloud / コンソール / HCL）
```bash
# 手動（命令的）: 1つずつ「作れ」と命令する。履歴も再現性も残らない
gcloud compute instances create web-1 \
  --zone=asia-northeast1-a --machine-type=e2-medium
```

```hcl
# Terraform（宣言的）: main.tf に「あるべき姿」を書く
terraform {
  required_providers {
    google = { source = "hashicorp/google", version = "~> 5.0" }
  }
  # state（現状の記録）は共有・ロックできるGCSバックエンドに置くのが定番
  backend "gcs" {
    bucket = "my-tfstate-bucket"
    prefix = "prod"
  }
}

provider "google" {
  project = "my-project"
  region  = "asia-northeast1"
}

resource "google_compute_instance" "web" {
  name         = "web-1"
  machine_type = "e2-medium"
  zone         = "asia-northeast1-a"
  boot_disk { initialize_params { image = "debian-cloud/debian-12" } }
  network_interface { network = "default" }
}
```

```bash
terraform init      # プロバイダ取得・バックエンド初期化
terraform plan      # 差分（これから何を作る/変える/消すか）を確認 ← 必ず見る
terraform apply     # planの内容を実際のGCPに反映
terraform destroy   # 定義したリソースをまとめて削除
```

コンソールでは作れない／作りづらいものはなく、IaCは「同じことを、追跡可能・再現可能にやる」ための土台。運用に乗せたら **手動変更は原則しない**のが鉄則。

## 実務での勘所
- **plan を必ずレビュー**：`terraform plan` の差分をPRに貼り、誰かが目視する。`apply` は差分を理解してから。
- **state はリモート + ロック**：`backend "gcs"` で共有バケットに置き、複数人の同時 `apply` を防ぐ。stateにはIPやシークレットが入りうるので扱いは厳重に。
- **モジュール化**：VPC・GKE・Cloud SQL など単位ごとにモジュール化し、本番/検証で変数だけ差し替えて再利用（DRY）。
- **環境分離**：`prod` / `staging` を workspace か別ディレクトリ + 別state prefix で分ける。
- **CI/CD連携**：`plan` をPRで自動実行、マージで `apply`、という流れ（Atlantis や Cloud Build）で人手ミスを減らす。
- 作ったインフラの稼働は [monitoring.md](./monitoring.md) で監視し、IaC化したリソースに最初からアラートも仕込む。

## ハマりどころ / アンチパターン
- **drift（ドリフト）＝手動変更とのズレ**：コンソールで手動修正すると、コードの「あるべき姿」と実体がずれる。次の `apply` で**手動変更が消される**か、`plan` に謎の差分が出続ける。IaC導入後の手動変更は厳禁。
- **state の取り合い・破損**：stateをローカルや各自のPCに置くと、複数人作業で食い違い・上書きが起きる。リモート + ロック必須。state を手で編集して壊すのも事故の定番。
- **API有効化漏れ**：GCPは使うサービスごとにAPIを有効化する必要がある（例 `compute.googleapis.com`）。有効化前にリソースを作ろうとして `apply` が失敗する。`google_project_service` で有効化もコード化しておく。
- **シークレットを `.tf` や state に平文で**：パスワードやキーをコードに直書きすると履歴に残る。Secret Manager や変数（`TF_VAR_`）経由にし、stateバケットも暗号化・権限制限する。
- **全部入りの巨大 main.tf**：1ファイルに全リソースを詰めると `plan` が読めず事故る。モジュール・ファイル分割（高凝集・低結合）で小さく保つ。
- **`terraform destroy` の誤爆**：本番stateで実行すると一括削除。環境とstateを取り違えないよう workspace 表示を確認する。

## 関連
[monitoring.md](./monitoring.md)
