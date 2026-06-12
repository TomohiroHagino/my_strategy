# Provider / Resource / Data Source（Terraform）

## ひとことで言うと
Terraformの**最小の登場人物3つ**。**Provider**＝どのクラウド/SaaSを操作するかのプラグイン、**Resource**＝Terraformが作成・管理する実体（VM・バケット等）、**Data Source**＝既に存在するものを「読み取って参照する」だけの宣言。

## 役割・なぜ必要か
- **Provider**：Terraform本体はクラウドを直接知らない。各クラウドのAPIを叩く部分はプラグイン（プロバイダ）が担う。`aws` / `google` / `azurerm` / `kubernetes` などを差し替えるだけで、同じ書き方で別の対象を扱える。
- **Resource**：Terraformが**ライフサイクルを管理する**対象。`apply` で作成し、コードを消せば削除され、変更すれば差分適用される。stateに記録されるのはこれ。
- **Data Source**：Terraformが**作らない / 管理しない**既存リソースを参照したい時に使う。「最新のAMI ID」「既存VPCのID」などを取得して、自分のResourceから使う。読むだけなので消えない。

## 基本の書き方（コード）
```hcl
# 1) Provider：何を操作するか＋接続設定
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

# 2) Data Source：既存のもの（最新のAmazon Linux AMI）を「読み取る」
data "aws_ami" "linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

# 3) Resource：Terraformが作って管理する実体
resource "aws_instance" "web" {
  ami           = data.aws_ami.linux.id   # ← Data Source の値を参照
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }
}
```

参照の構文（ここが要）：
```hcl
# resource は     <type>.<name>.<attribute>
aws_instance.web.id            # 作成されたインスタンスのID
aws_instance.web.private_ip    # その属性

# data は data. を先頭に付ける
data.aws_ami.linux.id          # 読み取ったAMIのID

# 例：別リソースから参照して依存関係が自動でつながる
resource "aws_eip" "web_ip" {
  instance = aws_instance.web.id   # webが先に作られる、とTerraformが判断
}
```

複数プロバイダ / 同一プロバイダの別設定は **alias** で使い分ける：
```hcl
provider "aws" {
  region = "ap-northeast-1"   # デフォルト（東京）
}
provider "aws" {
  alias  = "us"
  region = "us-east-1"        # 別リージョン（CloudFront証明書など）
}

resource "aws_acm_certificate" "cdn" {
  provider = aws.us           # ← このリソースだけ us-east-1 で作る
  domain_name = "example.com"
  validation_method = "DNS"
}
```

## 実務での使い方・定番パターン
- **Provider のバージョンは必ず固定**（`version = "~> 5.0"`）。lockファイル（`.terraform.lock.hcl`）とセットでチームの版を揃える。
- **「自分が作るもの」はResource、「既にあるもの・他チーム管理」はData Source** と切り分ける。既存VPCに相乗りするなら `data "aws_vpc"` で参照する。
- AMI・最新イメージ・アカウントID・利用可能AZ一覧などは **Data Sourceで動的に取得**し、ハードコードを避ける（`data.aws_caller_identity.current.account_id` など）。
- 認証情報は **provider ブロックに直書きしない**。AWSなら環境変数 / `~/.aws/credentials` / IAMロールに任せる（→ [../aws/iam.md](../aws/iam.md)）。
- マルチリージョン・マルチアカウントは `alias` で `provider` を複数定義し、リソース側で `provider = aws.xxx` を指定。

## ハマりどころ / アンチパターン
- **Data Source を「作る」ものと勘違い**：Data Sourceは読み取り専用。新規作成はResource。`data` で書いても何も作られない。
- **プロバイダ認証情報のコードへの直書き**：`provider "aws" { access_key = "AKIA..." }` は漏洩の定番。環境変数 / プロファイル / ロールを使う。
- **`version` 未指定で最新が勝手に入る**：プロバイダのメジャー更新で `apply` が突然壊れる。制約を必ず書く。
- **存在しない既存リソースをData Sourceで参照**：フィルタが何にもマッチしない / 複数マッチでエラーになる。`most_recent` や厳密なフィルタで1件に絞る。
- **削除したいのにコードを消すだけ**：Resourceブロックを消せば次の `apply` で削除される（これは正しい挙動）。だが**State上は管理外にしたいだけ**なら `terraform state rm`、本当に消すなら destroy、と区別する。→ [state.md](./state.md)

## 関連
[getting_started.md](./getting_started.md) / [variables_outputs.md](./variables_outputs.md) / [state.md](./state.md) / [expressions.md](./expressions.md) / [../aws/iam.md](../aws/iam.md)
