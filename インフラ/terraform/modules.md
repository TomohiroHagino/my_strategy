# モジュール（Terraform）

## ひとことで言うと
**Module** ＝ 複数のリソースを「入力（variable）と出力（output）を持つ1つの部品」にまとめたもの。`module "x" { source = "..." }` で呼び出して**再利用**する。関数のように、同じ構成（VPC一式など）をパラメータ違いで何度も使い回せる。

## 役割・なぜ必要か
- VPC + サブネット + ルートテーブル… のような**毎回ほぼ同じ塊**を、環境ごとにコピペすると破綻する。モジュール化して `source` で呼べば1箇所で管理できる（DRY）。
- **入力変数と出力だけが外部との接点**になり、内部実装を隠せる（カプセル化）。利用側は「何を渡すと何が返るか」だけ知ればよい。
- 公式の **Terraform Registry** には実績あるモジュール（VPC・EKS等）が公開されており、ゼロから書かずに済む。
- ルート（一番上のディレクトリ）も実は「ルートモジュール」。`module` で呼ぶものは「子モジュール」。すべてモジュールの入れ子。

## 基本の書き方（コード）
子モジュールの中身（`modules/vpc/`）：
```hcl
# modules/vpc/variables.tf — 入力（外から受け取る）
variable "cidr_block" { type = string }
variable "name"       { type = string }

# modules/vpc/main.tf — 中身
resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags = { Name = var.name }
}
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.this.id
  cidr_block = cidrsubnet(var.cidr_block, 8, 0)
}

# modules/vpc/outputs.tf — 出力（親に返す）
output "vpc_id"    { value = aws_vpc.this.id }
output "subnet_id" { value = aws_subnet.public.id }
```

親（ルート）から呼ぶ：
```hcl
# main.tf — モジュールを「呼び出す」
module "vpc" {
  source     = "./modules/vpc"   # ローカルパス
  name       = "prod-vpc"
  cidr_block = "10.0.0.0/16"
}

# モジュールの出力は module.<名前>.<output名> で参照
resource "aws_instance" "web" {
  subnet_id     = module.vpc.subnet_id   # ← 子の output を使う
  instance_type = "t3.micro"
  ami           = "ami-xxxx"
}
```

source の指定方法（ローカル / Registry / Git）：
```hcl
# ローカル
module "vpc" { source = "./modules/vpc" }

# Terraform Registry（公式・実績あり。バージョン固定推奨）
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  name    = "prod-vpc"
  cidr    = "10.0.0.0/16"
}

# Git リポジトリ（社内共通モジュールの共有）
module "vpc" {
  source = "git::https://github.com/myorg/tf-modules.git//vpc?ref=v1.2.0"
}
```

同じモジュールを複数インスタンス化（`for_each` / `count`）：
```hcl
module "service" {
  source   = "./modules/app"
  for_each = toset(["api", "worker", "cron"])
  name     = each.key       # api / worker / cron の3つ分作られる
}
# 参照： module.service["api"].endpoint
```

## 実務での使い方・定番パターン
- **入力と出力を最小限に絞る**。モジュールの「公開インターフェース」が広すぎると再利用しづらい。必要な変数だけ受ける。
- **Registry / Git のモジュールは `version`（または `?ref=`）でバージョン固定**。固定しないと、ある日中身が変わって `apply` が壊れる。
- **環境差分はモジュールの引数で吸収**。`module.app` を dev/stg/prod から別パラメータで呼ぶ（→ [environments.md](./environments.md)）。
- **VPC・IAM・ネットワークなど定型の塊からモジュール化**する。最初は1ディレクトリで書き、繰り返しが見えたら抽出（YAGNI：先に作りすぎない）。
- **公式Registryの実績モジュール（`terraform-aws-modules/*`）を土台に**すると、エッジケースまで作り込まれていて速い。

## ハマりどころ / アンチパターン
- **モジュールの過剰な抽象化**：使う前から汎用化して変数が30個、など。実際の繰り返しが出てから抽出する（YAGNI）。
- **`source` のバージョン未固定**：Registry/Gitモジュールが更新されて突然壊れる。`version` / `?ref=タグ` で固定。
- **モジュール内で `provider` を定義しすぎる**：基本はルートで provider を定義し、モジュールには渡す（`providers = {}`）。モジュール内 provider はマルチリージョン等の特殊時のみ。
- **出力を返し忘れて参照できない**：子で作ったリソースのIDを親で使いたいのに `output` を書いていない。接点は必ず output で公開。
- **巨大な「何でもモジュール」**：1つのモジュールがVPCもDBもアプリも抱える。責務ごとに分割し、小さく組み合わせる。
- **リファクタでモジュール構造を変えてstateがズレる**：リソースの住所が変わるので `terraform state mv` が必要（→ [state.md](./state.md)）。

## 関連
[providers_resources.md](./providers_resources.md) / [variables_outputs.md](./variables_outputs.md) / [expressions.md](./expressions.md) / [environments.md](./environments.md) / [state.md](./state.md)
