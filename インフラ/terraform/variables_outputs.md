# variable / output / locals / tfvars（Terraform）

## ひとことで言うと
構成を**パラメータ化**する仕組み。**variable**＝外から渡す入力、**locals**＝内部で計算する中間値、**output**＝外（CLI出力・他モジュール）へ渡す結果、**tfvars**＝variableの実値を書くファイル。これらで「同じコードを環境ごとに使い回す」を実現する。

## 役割・なぜ必要か
- リージョン名・インスタンスサイズ・環境名などを**コードに直書きすると、環境ごとにコピペが増えて破綻**する。`variable` で外出しして1つのコードを使い回す。
- `locals` は「何度も出てくる式」「組み立てた名前」をまとめる場所。DRYに保ち、変更点を1箇所にする。
- `output` は `apply` 後に値（EC2のIP・DBのエンドポイント等）を取り出したり、**モジュールの戻り値**として親に渡すために使う。
- `tfvars` は variable の**実値**をまとめるファイル。dev/stg/prod ごとに別ファイルを用意して切り替える。

## 基本の書き方（コード）
```hcl
# variables.tf — 入力の「定義」（型・デフォルト・説明・検証）
variable "environment" {
  type        = string
  description = "環境名（dev / stg / prod）"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"        # 未指定ならこれ
}

variable "instance_count" {
  type    = number
  default = 1
  validation {                # 入力の検証（境界でのバリデーション）
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "instance_count は 1〜10 で指定してください。"
  }
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# locals — 内部の中間値（再利用する組み立て）
locals {
  name_prefix = "${var.environment}-myapp"
  common_tags = merge(var.tags, {
    Environment = var.environment
    ManagedBy   = "terraform"
  })
}

# 参照は var.x / local.x
resource "aws_instance" "web" {
  instance_type = var.instance_type
  tags = merge(local.common_tags, { Name = "${local.name_prefix}-web" })
}
```

```hcl
# outputs.tf — 結果を外へ
output "instance_id" {
  value       = aws_instance.web.id
  description = "作成したEC2のID"
}

output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true   # ← CLIログに値を表示しない（秘密情報）
}
```

variableに実値を渡す4つの方法（優先順位：下ほど強い）：
```bash
# 1) terraform.tfvars / *.auto.tfvars（自動読み込み）
#    environment = "dev"
#    instance_type = "t3.small"

# 2) 明示的に tfvars を指定
terraform apply -var-file="environments/prod/prod.tfvars"

# 3) コマンドラインで個別指定
terraform apply -var="environment=prod" -var="instance_count=3"

# 4) 環境変数（TF_VAR_ プレフィックス）
export TF_VAR_environment=prod
terraform apply
```

```hcl
# prod.tfvars の例
environment   = "prod"
instance_type = "t3.large"
instance_count = 3
tags = {
  Team = "platform"
}
```

## 実務での使い方・定番パターン
- **variableには `type` と `description` を必ず付ける**。型を書くと誤った値で早期に落ちて安全。
- **環境差分は tfvars に閉じ込める**。`main.tf` は環境非依存にし、`dev.tfvars` / `prod.tfvars` だけ差し替える（→ [environments.md](./environments.md)）。
- **秘密情報の output には `sensitive = true`**。CLIやCIログに平文で出るのを防ぐ。ただし**stateには平文で残る**点は別問題（→ [state.md](./state.md)）。
- **命名の組み立ては locals に集約**（`local.name_prefix`）。命名規則を変えるとき1箇所で済む。
- 秘密値（DBパスワード等）は tfvars にもなるべく書かず、`TF_VAR_*` 環境変数やシークレットマネージャ参照（`data` 経由）で渡す。

## ハマりどころ / アンチパターン
- **秘密値を tfvars に書いてgitコミット**：`*.tfvars` をうっかり追跡してパスワード漏洩。秘密値は `TF_VAR_*` か Secrets Manager。`*.tfvars` はgitignore + `.example` で共有。
- **`output` に `sensitive` を付け忘れ**：DBパスワード等がCI/CDログに平文で残る。秘密の出力は必ず `sensitive = true`。
- **`var.x` と `local.x` の混同**：`var` は外部入力、`local` は内部計算。localは外から上書きできない。
- **デフォルト依存で本番事故**：`instance_type` のデフォルトが `t3.micro` のまま本番に適用、など。本番は明示的に tfvars で指定する。
- **型を書かずに `any` 任せ**：誤った値（数値のつもりが文字列）に気づけない。`type` を明示してバリデーションを効かせる。

## 関連
[providers_resources.md](./providers_resources.md) / [expressions.md](./expressions.md) / [modules.md](./modules.md) / [environments.md](./environments.md) / [state.md](./state.md)
