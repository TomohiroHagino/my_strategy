# 式・メタ引数（Terraform）

## ひとことで言うと
HCLで「動的に・繰り返し・条件付きで」構成を書くための道具一式。**count / for_each**（リソースを繰り返し作る）、**for 式**（リストやマップを変換）、**dynamic ブロック**（ネストブロックを動的生成）、**条件式 `? :`**、**関数**、**depends_on**（依存の明示）、**lifecycle**（作成・削除の挙動制御）。

## 役割・なぜ必要か
- 「サブネットを3つ」「環境ごとに有無を切替」のような**繰り返し・条件**を、コピペせず1つの定義で表現するため。
- Terraformは参照（`a.b.id`）から依存関係を自動推論するが、**推論できない依存は `depends_on` で明示**する必要がある。
- リソースの**削除を禁止したい / 入れ替え時に先に作ってから消したい**等の繊細な制御は `lifecycle` で行う。誤った削除事故を防ぐ要。

## 基本の書き方（コード）
**count**（個数で繰り返す。インデックスで参照）：
```hcl
resource "aws_instance" "web" {
  count         = 3                    # 0,1,2 の3つ
  ami           = "ami-xxxx"
  instance_type = "t3.micro"
  tags = { Name = "web-${count.index}" }
}
# 参照： aws_instance.web[0].id  /  aws_instance.web[*].id（全部）
```

**for_each**（キー付きで繰り返す。要素の増減に強い＝こちらが基本）：
```hcl
resource "aws_iam_user" "team" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.key                   # each.key / each.value
}

# マップでも回せる
variable "instances" {
  default = {
    web = "t3.micro"
    db  = "t3.large"
  }
}
resource "aws_instance" "srv" {
  for_each      = var.instances
  instance_type = each.value
  ami           = "ami-xxxx"
  tags = { Name = each.key }            # web / db
}
# 参照： aws_instance.srv["web"].id
```

**for 式**（リスト/マップを変換）と **条件式**：
```hcl
locals {
  names    = ["web", "db", "cache"]
  upper    = [for n in local.names : upper(n)]            # ["WEB","DB","CACHE"]
  name_map = { for n in local.names : n => "prod-${n}" }  # { web="prod-web", ... }
}

# 三項演算子（条件で値を切り替え）
resource "aws_instance" "web" {
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"
}

# 条件付きで作る/作らない（count を 0 か 1 に）
resource "aws_instance" "bastion" {
  count = var.enable_bastion ? 1 : 0
}
```

**dynamic ブロック**（ネストブロックを動的に生成。SGルール等）：
```hcl
variable "ingress_ports" { default = [80, 443, 22] }

resource "aws_security_group" "web" {
  name = "web-sg"
  dynamic "ingress" {                  # ingress ブロックを繰り返し生成
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

**関数**（組み込み。文字列・コレクション・暗号化など）：
```hcl
locals {
  bucket   = lower("My-App")                 # "my-app"
  merged   = merge({ a = 1 }, { b = 2 })     # { a=1, b=2 }
  first_az = element(data.aws_availability_zones.all.names, 0)
  cidr     = cidrsubnet("10.0.0.0/16", 8, 1) # "10.0.1.0/24"
  json     = jsonencode({ key = "value" })
}
```

**depends_on**（参照に表れない依存を明示）と **lifecycle**：
```hcl
resource "aws_instance" "app" {
  # ... IAMロールのポリシー反映を待ちたい等、暗黙依存を明示
  depends_on = [aws_iam_role_policy.app]

  lifecycle {
    prevent_destroy       = true   # 誤って destroy されるのを禁止（本番DB等）
    create_before_destroy = true   # 入れ替え時、先に新を作ってから旧を消す（ダウンタイム回避）
    ignore_changes        = [tags] # この属性の差分は無視（手動運用するタグ等）
  }
}
```

## 実務での使い方・定番パターン
- **繰り返しは原則 `for_each`**。`count` はインデックス管理なので「途中の要素を消すと後続が全部作り直し」になりがち。`for_each` はキー単位で安定。
- **条件付き生成は `count = cond ? 1 : 0`**（または `for_each` を空にする）。bastionやprod限定リソースの切替に多用。
- **`prevent_destroy = true` を本番DB・stateバケット等に**付け、`apply` 事故での消失を防ぐ。
- **`create_before_destroy`** で、置き換えが必要なリソース（AMI更新時のインスタンス等）のダウンタイムを減らす。
- **`ignore_changes`** は「Terraform管理外で変わる属性」（オートスケールのdesired_count、運用タグ）に限定して使う。乱用するとdriftを隠して危険。
- 関数は公式リファレンスで都度確認。`merge` / `lookup` / `coalesce` / `cidrsubnet` / `jsonencode` あたりが頻出。

## ハマりどころ / アンチパターン
- **`count` 配列の途中削除で全部作り直し**：`[0,1,2]` の真ん中を消すとインデックスがずれ、後続が破棄→再作成。集合的なものは `for_each` を使う。
- **`for_each` に未知の値（applyしないと決まらない値）を渡す**：`Invalid for_each argument` エラー。`toset()` で確定値を渡す。動的すぎる値はNG。
- **`depends_on` の付けすぎ**：参照で繋がる依存に手動 `depends_on` を足すと冗長＆遅延。本当に暗黙の依存だけに使う。
- **`ignore_changes` でdriftを隠す**：差分が出るのが面倒だからと多用すると、現実とコードが乖離して把握不能に。最小限に。
- **`prevent_destroy` のまま消したい**：本当に消す時は一旦 `lifecycle` を外してから destroy。付けっぱなしだとapplyが失敗し続ける。
- **三項演算子で型不一致**：`cond ? "a" : 0`（文字列と数値）はエラー。両辺の型を揃える。

## 関連
[providers_resources.md](./providers_resources.md) / [variables_outputs.md](./variables_outputs.md) / [modules.md](./modules.md) / [pitfalls.md](./pitfalls.md)
