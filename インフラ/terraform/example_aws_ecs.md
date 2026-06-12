# 実例：AWS ECS 構成のファイル分割（Terraform）

## ひとことで言うと
1つの巨大な `main.tf` に全部書かず、**リソース種別ごとに `.tf` を分割**するのが実務の定番。Terraform は同じディレクトリの `*.tf` を**全部マージして1つの構成として読む**ので、ファイルを分けても動作は変わらない（人間が見やすくするための分割）。下は AWS で「VPC + ECR + ECS(Fargate) + RDS + S3」を組む典型例。

## 全体構成
```
.
├── main.tf                    # provider・backend・共通設定
├── variables.tf               # 入力変数の「定義」
├── terraform.tfvars.example   # 変数の実値サンプル（コピーして terraform.tfvars に）
├── outputs.tf                 # 出力（URL・エンドポイント等）
├── vpc.tf                     # ネットワーク（VPC・subnet・SG）
├── ecr.tf                     # コンテナイメージ置き場（ECR）
├── ecs.tf                     # コンテナ実行（クラスタ・タスク・サービス）
├── rds.tf                     # DB（RDS）
└── s3.tf                      # ストレージ（S3）
```
> ファイル名に技術的な意味はない。`vpc.tf` の resource を `ecs.tf` から参照してもよい（Terraform が依存関係を自動で解決する）。

---

## main.tf — provider・backend・共通設定
```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {                    # state をS3で共有（チーム運用の定番）
    bucket         = "my-tfstate"
    key            = "ecs-app/terraform.tfstate"
    region         = "ap-northeast-1"
    dynamodb_table = "tf-lock"      # 同時applyを防ぐロック
  }
}

provider "aws" {
  region = var.region
}
```

## variables.tf — 入力変数の「定義」
```hcl
variable "region"  { type = string, default = "ap-northeast-1" }
variable "project" { type = string }                 # 名前のprefixに使う
variable "db_password" {
  type      = string
  sensitive = true                                   # ログ/出力に出さない
}
variable "container_image_tag" { type = string, default = "latest" }
```

## terraform.tfvars.example — 実値のサンプル
```hcl
# 使い方: cp terraform.tfvars.example terraform.tfvars して実値を入れる
# （terraform.tfvars は秘密を含むので .gitignore する。example だけコミット）
project     = "myapp"
region      = "ap-northeast-1"
db_password = "CHANGE_ME"
```

## outputs.tf — 出力（apply後に表示・他から参照）
```hcl
output "alb_dns_name"       { value = aws_lb.app.dns_name }
output "ecr_repository_url" { value = aws_ecr_repository.app.repository_url }
output "rds_endpoint"       { value = aws_db_instance.app.address, sensitive = true }
output "s3_bucket"          { value = aws_s3_bucket.assets.bucket }
```

## vpc.tf — ネットワーク
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "${var.project}-vpc" }
}
resource "aws_subnet" "public" {
  count                   = 2                          # 2AZに分散
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index}.0/24"
  map_public_ip_on_launch = true
}
resource "aws_security_group" "app" {
  vpc_id = aws_vpc.main.id
  ingress { from_port = 80, to_port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0,  to_port = 0,  protocol = "-1",  cidr_blocks = ["0.0.0.0/0"] }
}
# 実際は internet_gateway / route_table も要る
```

## ecr.tf — コンテナイメージ置き場
```hcl
resource "aws_ecr_repository" "app" {
  name = "${var.project}-app"
  image_scanning_configuration { scan_on_push = true }   # push時に脆弱性スキャン
}
```

## ecs.tf — コンテナ実行（Fargate）
```hcl
resource "aws_ecs_cluster" "main" {
  name = "${var.project}-cluster"
}
resource "aws_ecs_task_definition" "app" {
  family                   = "${var.project}-app"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "256"
  memory                   = "512"
  container_definitions = jsonencode([{
    name         = "app"
    image        = "${aws_ecr_repository.app.repository_url}:${var.container_image_tag}"  # ← ecr.tf を参照
    portMappings = [{ containerPort = 80 }]
  }])
}
resource "aws_ecs_service" "app" {
  name            = "${var.project}-svc"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2
  launch_type     = "FARGATE"
  network_configuration {
    subnets          = aws_subnet.public[*].id              # ← vpc.tf を参照
    security_groups  = [aws_security_group.app.id]
    assign_public_ip = true
  }
}
```

## rds.tf — DB
```hcl
resource "aws_db_subnet_group" "app" {
  name       = "${var.project}-db"
  subnet_ids = aws_subnet.public[*].id                  # ← vpc.tf を参照
}
resource "aws_db_instance" "app" {
  identifier             = "${var.project}-db"
  engine                 = "postgres"
  instance_class         = "db.t3.micro"
  allocated_storage      = 20
  username               = "appuser"
  password               = var.db_password               # ← variables.tf の sensitive 変数
  db_subnet_group_name   = aws_db_subnet_group.app.name
  vpc_security_group_ids = [aws_security_group.app.id]
  skip_final_snapshot    = true                          # 開発用。本番は false
}
```

## s3.tf — ストレージ
```hcl
resource "aws_s3_bucket" "assets" {
  bucket = "${var.project}-assets"
}
resource "aws_s3_bucket_public_access_block" "assets" {   # 公開事故を防ぐ（必須級）
  bucket                  = aws_s3_bucket.assets.id
  block_public_acls       = true
  block_public_policy      = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## 実務での使い方・定番パターン
- **ファイル間は自由に参照できる**：`ecs.tf` から `aws_ecr_repository.app...`（ecr.tf）や `aws_subnet.public`（vpc.tf）を参照してOK。Terraform が依存グラフを組んで**正しい順序で作る**（VPC→RDS→ECS など）。
- **`terraform.tfvars` は除外、`.example` だけコミット**：秘密（`db_password` 等）を入れる実ファイルは `.gitignore`。チームには `.example` で「何を入れるか」だけ共有。
- **命名は `${var.project}-xxx`** で prefix を揃え、環境ごとの衝突を防ぐ。
- 大きくなったら `modules/` に切り出す（vpc・ecs を再利用可能に）。→ [modules.md](./modules.md)

## ハマりどころ / アンチパターン
- **`*.tf` は全部マージされる**：別ファイルでも**同じ名前の resource は重複エラー**。`resource "aws_vpc" "main"` を2か所に書かない。
- **`terraform.tfvars` をコミットして秘密漏洩**：必ず `.gitignore`。漏れたら鍵をローテーション。
- **削除順序の事故**：手動で消したリソースがあると drift で `destroy`/`apply` が詰まる。変更は Terraform 経由に統一。
- **S3 の公開設定漏れ**：`aws_s3_bucket_public_access_block` を忘れると意図せず公開。

## 関連
[getting_started.md](./getting_started.md) / [variables_outputs.md](./variables_outputs.md) / [state.md](./state.md) / [modules.md](./modules.md) / [../aws/containers.md](../aws/containers.md)
