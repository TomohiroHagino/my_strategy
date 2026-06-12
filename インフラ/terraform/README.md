# Terraform 実務リファレンス（索引）

> **Terraform** ＝ 宣言的IaC（Infrastructure as Code）ツール。インフラの「あるべき状態」を **HCL（`.tf`）** で書き、現実との差分をTerraformが計算して適用する。AWS / GCP / その他多数のプロバイダを同じ書き方で扱える、**クラウド横断**の標準ツール。
> 各ファイルは「ひとことで言うと → 役割・なぜ → 基本の書き方（HCL）→ 実務での使い方 → ハマり所 → 関連」。

## まず押さえる流れ
```
 init  … プロバイダ取得・backend初期化（最初の一回 / 構成変更時）
   ↓
 plan  … 「今の状態」と「コードのあるべき状態」の差分をプレビュー（何が増減するか）
   ↓
 apply … planの内容を実際に適用（リソースを作成 / 変更 / 削除）
   ↓
 destroy … 管理下のリソースをまとめて削除（検証環境の片付け等）
```
- 宣言的：手順（どう作るか）ではなく **結果（最終状態）** を書く。Terraformが差分を埋める。
- 中核は **State**（`terraform.tfstate`）＝「コード」と「現実のインフラ」を結ぶ対応表。これが全ての要。

## 3つの土台概念
```
 ① Provider / Resource … 何を（どのクラウドの何を）作るかの宣言
 ② State                … 現実との対応表。チームで使うならリモートbackend必須
 ③ Module               … 構成を部品化して再利用（VPC一式など）
```

## 項目（各ファイルへ）

### 基礎 / 始め方
- [getting_started.md](./getting_started.md) … 始め方（インストール / `init`→`plan`→`apply` / フォルダ構成）
- [example_aws_ecs.md](./example_aws_ecs.md) … **実例：AWS ECS構成のファイル分割**（main/variables/outputs/vpc/ecr/ecs/rds/s3.tf を解説）
- [providers_resources.md](./providers_resources.md) … Provider / Resource / Data Source
- [variables_outputs.md](./variables_outputs.md) … variable / output / locals / tfvars

### 状態 / 再利用
- [state.md](./state.md) … State とリモートbackend（S3+DynamoDB / GCS）・ロック・drift
- [modules.md](./modules.md) … モジュール（source / 入出力 / Registry / 再利用）

### 書き方 / 操作 / 運用
- [expressions.md](./expressions.md) … count / for_each / dynamic / 条件 / 関数 / depends_on / lifecycle
- [commands.md](./commands.md) … init / plan / apply / destroy / fmt / validate / import / state / output
- [environments.md](./environments.md) … 環境分け（workspace vs ディレクトリ＋tfvars、dev/stg/prod）
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ（state事故・drift・秘密情報・apply順序）

## クラウド側の具体サービス
Terraformは「どう適用するか」を担う。「何を作るか（VPCやIAMの中身）」はクラウド側の概念を参照：
- AWS … [../aws/README.md](../aws/README.md)（IAM / EC2 / S3 / VPC …）
- GCP … [../gcp/README.md](../gcp/README.md)（IAM / Compute / Cloud Run / GCS …）

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Terraform）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）   ← HCL（.tf）の具体例
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```

> ※ 例は `aws` プロバイダ中心だが、Terraform本体の概念はクラウド非依存。プロバイダを差し替えればGCPでも同じ書き方が通用する。
