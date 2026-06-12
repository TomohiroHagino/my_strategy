# 始め方（Terraform）

## ひとことで言うと
Terraformを使い始めるための初期一式。**インストール → 最小の `.tf` を書く → `terraform init`（プロバイダ取得）→ `plan`（差分確認）→ `apply`（適用）** までの最初のサイクルと、後で破綻しないための **フォルダ構成** を整える工程。

## 役割・なぜ必要か
- Terraformは「`.tf` に書いた状態」を現実のインフラに反映するツール。最初に **init → plan → apply の感覚** を掴むと、以降のすべてはこの繰り返しになる。
- 最初に決めるべきは2つ：**どのプロバイダを使うか**（aws / google など）と、**stateをどこに置くか**（最初はローカル、チームならリモートbackend）。
- フォルダ構成を最初に決めておかないと、ファイルが肥大化したり、環境（dev/stg/prod）が混ざったり、`*.tfstate` をうっかりgitに上げる事故が起きる。土台を先に作る。

## 基本の書き方（コード）
最小構成。`main.tf` 1枚から始められる：
```hcl
# main.tf
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"   # プロバイダの出どころ（Terraform Registry）
      version = "~> 5.0"          # 5.x 系に固定（破壊的更新を避ける）
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"        # 東京
}

# 試しに1つ作ってみる（S3バケット）
resource "aws_s3_bucket" "example" {
  bucket = "my-tf-getting-started-20260612"  # 全世界で一意
}
```

インストール（macOS / Linux 例）と最初のサイクル：
```bash
# インストール（例：tfenvでバージョン管理 / 公式バイナリ / brew など）
brew install terraform        # mac の一例
terraform version             # 入ったか確認

# 認証はプロバイダ任せ。aws の場合は AWS CLI の設定をそのまま使う
aws sts get-caller-identity   # ← ../aws/getting_started.md 参照

# --- 最初のサイクル ---
terraform init      # プロバイダDL・.terraform/ 作成・backend初期化
terraform fmt       # コード整形（インデント等を標準化）
terraform validate  # 構文・参照の静的チェック
terraform plan      # 差分プレビュー（+ create / ~ update / - destroy）
terraform apply     # 適用（yes を入力。-auto-approve で省略可）

terraform destroy   # 後片付け（作ったものを消す）
```

## フォルダ構成（始動直後）
小さく始めて、育ったらモジュール化・環境分けする。**【生成】はTerraformが自動生成（基本さわらない）**、それ以外は自作：
```
my-infra/
├── main.tf                  # リソース本体（resource / module の宣言）★自作
├── providers.tf             # terraform{} ＋ provider{} 設定        ★自作
├── variables.tf             # 入力変数の定義（variable "x" {}）      ★自作
├── outputs.tf               # 出力値の定義（output "y" {}）          ★自作
├── terraform.tfvars         # 変数の実値（環境ごとの設定）★自作・秘密はここに置かない
├── versions.tf              # required_version / required_providers（分けるなら）★自作
│
├── modules/                 # 再利用部品（自作モジュール）          ★自作
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── app/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── environments/            # 環境別の設定（ディレクトリ分け方式の場合）★自作
│   ├── dev/
│   │   ├── main.tf          # modules を dev 用パラメータで呼ぶ
│   │   └── terraform.tfvars # dev の実値
│   ├── stg/
│   │   └── ...
│   └── prod/
│       └── ...
│
├── .terraform/              # 【生成】プロバイダ本体・モジュールキャッシュ ← gitignore
├── .terraform.lock.hcl      # 【生成】プロバイダのバージョン固定（lockファイル）★これはコミットする
├── terraform.tfstate        # 【生成】現実との対応表 ← ローカルbackend時。秘密情報を含む→gitignore
└── terraform.tfstate.backup # 【生成】直前のstateバックアップ ← gitignore
```

`.gitignore` の定番（事故防止の要）：
```gitignore
# Terraform
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfvars            # 秘密値を含む場合。サンプルは terraform.tfvars.example で共有
override.tf
.terraformrc
# ※ .terraform.lock.hcl は逆にコミットする（チームでバージョンを揃えるため）
```

## 実務での使い方・定番パターン
- **`plan` を必ず読んでから `apply`**。`+ create` / `~ update` / `- destroy` の記号を確認。`- destroy` が予想外に並んでいたら一旦止める。
- **`terraform fmt` と `validate` はコミット前 / CIで自動化**。整形と構文崩れを早期に潰す。
- **`.terraform.lock.hcl` はコミットする**。チーム全員のプロバイダ版を揃え、「自分の環境では動く」を防ぐ。
- 最初はローカルstateで十分。**2人以上で触る瞬間にリモートbackendへ移行**する（→ [state.md](./state.md)）。
- 変数の実値や秘密はコードに直書きせず、`tfvars` や環境変数（`TF_VAR_*`）で渡す（→ [variables_outputs.md](./variables_outputs.md)）。

## ハマりどころ / アンチパターン
- **`*.tfstate` をgitにコミット**：最大級の事故。stateには接続情報・パスワード等が平文で入りうる。必ずgitignore。チーム運用ならリモートbackend。→ [state.md](./state.md)
- **プロバイダのバージョン未固定**：`version` を書かずに `init` すると最新が入り、ある日 `apply` が壊れる。`~> 5.0` のように制約を付け、lockファイルをコミットする。
- **`apply` を `plan` 無しで実行**：差分を見ずに適用して意図しない削除。必ず先に `plan`。
- **巨大な単一ディレクトリ**：全リソースを1つの `main.tf` に詰めると `plan` が遅く危険になる。早めにモジュール / 環境分割（→ [modules.md](./modules.md) / [environments.md](./environments.md)）。
- **`.terraform/` をコミット**：プロバイダ本体（数百MB）が混入する。gitignore必須。

## 関連
[providers_resources.md](./providers_resources.md) / [variables_outputs.md](./variables_outputs.md) / [state.md](./state.md) / [commands.md](./commands.md) / [../aws/getting_started.md](../aws/getting_started.md)
