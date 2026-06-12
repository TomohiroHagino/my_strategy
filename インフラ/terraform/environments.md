# 環境分け（Terraform）

## ひとことで言うと
dev / stg / prod のように**同じ構成を環境ごとに分けて管理**する方法。大きく2方式：**workspace**（1コードで複数stateを切り替える）と、**ディレクトリ分け＋環境別tfvars**（環境ごとにディレクトリとstateを分ける）。実務では後者（ディレクトリ分け）が主流。

## 役割・なぜ必要か
- dev で試した構成を stg → prod へ同じコードで昇格させたい。だが**stateとパラメータは環境ごとに完全に分離**しないと、dev の `apply` が prod を巻き込む大事故になる。
- 「コードは共通、値だけ環境差分」を実現する。共通部分はモジュール、差分は tfvars に閉じ込めるのが基本設計。
- 環境ごとに**権限・backend・承認フロー**も分けたい（prodは厳格に）。ディレクトリ分けだと分離しやすい。

## 基本の書き方（コード）
### 方式A：ディレクトリ分け＋環境別tfvars（推奨）
```
├── modules/                 # 共通の部品（環境非依存）
│   └── app/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    ├── dev/
    │   ├── main.tf          # module を dev 用に呼ぶ
    │   ├── backend.tf       # state: dev/terraform.tfstate
    │   └── terraform.tfvars # dev の値
    ├── stg/
    │   └── ...
    └── prod/
        ├── main.tf
        ├── backend.tf       # state: prod/terraform.tfstate（別物）
        └── terraform.tfvars # prod の値
```
```hcl
# environments/prod/main.tf
module "app" {
  source         = "../../modules/app"
  environment    = "prod"
  instance_type  = "t3.large"
  instance_count = 3
}
```
```hcl
# environments/prod/backend.tf — 環境ごとに state を完全分離
terraform {
  backend "s3" {
    bucket = "mycompany-tfstate"
    key    = "prod/app/terraform.tfstate"   # dev とは別キー
    region = "ap-northeast-1"
    dynamodb_table = "terraform-locks"
    encrypt = true
  }
}
```
```bash
# 各環境のディレクトリで操作（stateが物理的に別なので混ざらない）
cd environments/prod
terraform init
terraform plan
terraform apply
```

### 方式B：workspace（1コードで複数state）
```bash
terraform workspace new dev      # workspace作成
terraform workspace new prod
terraform workspace list         # 一覧（* が現在）
terraform workspace select prod  # 切り替え

# state は workspace ごとに自動分離される（terraform.tfstate.d/<ws>/...）
terraform apply -var-file="prod.tfvars"
```
```hcl
# コード側で現在のworkspace名を参照して切り替え
locals {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
}
```

## 実務での使い方・定番パターン
- **本番を含むなら方式A（ディレクトリ分け）が安全**。環境ごとにbackend・権限・承認を分離でき、`cd prod` という物理的な区切りが「今どの環境を触っているか」を明確にする。
- **共通ロジックはモジュールへ、差分はtfvarsへ**。`modules/app` を全環境から呼び、`environment` / `instance_type` などだけ変える（→ [modules.md](./modules.md) / [variables_outputs.md](./variables_outputs.md)）。
- **workspaceは「ほぼ同一・軽量な環境の量産」向き**（レビュー用の使い捨て環境、開発者ごとのサンドボックス等）。本番と開発を同じコード・同じ権限で切り替えるのは事故リスクが高い。
- **prodは `prevent_destroy` と厳格な権限・手動承認ゲート**を重ねる（→ [expressions.md](./expressions.md)）。
- CIは「ブランチ→環境」を対応づける（develop→dev、main→prod 等）と、誤環境への適用を防げる。

## ハマりどころ / アンチパターン
- **workspaceで本番と開発を切り替え、prodのまま `destroy`**：`select` を間違えて本番を破壊。本番は別ディレクトリ＋別権限で物理分離するのが安全。
- **環境ごとにstateを分けていない**：dev の `apply` が prod のリソースを触る。backendの `key` / workspace でstateを必ず分離。
- **tfvarsに秘密値を入れて環境別ファイルをコミット**：prod.tfvars にパスワード→漏洩。秘密は `TF_VAR_*` / Secrets Manager（→ [state.md](./state.md)）。
- **環境ごとにコードを丸ごとコピペ**：dev/stg/prod の `main.tf` が少しずつ違う「コピペ地獄」に。共通部分はモジュール化してDRYに保つ。
- **`terraform.workspace` 依存の条件分岐が複雑化**：workspace名で大量の三項分岐を書くと読めなくなる。差分が多いならディレクトリ分けに移行。

## 関連
[modules.md](./modules.md) / [variables_outputs.md](./variables_outputs.md) / [state.md](./state.md) / [commands.md](./commands.md) / [expressions.md](./expressions.md)
