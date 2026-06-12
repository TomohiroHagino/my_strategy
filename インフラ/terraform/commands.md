# コマンド（Terraform）

## ひとことで言うと
Terraform CLIの主要コマンド一覧。中核は **init → plan → apply → destroy** のライフサイクル。加えて整形 `fmt`、検証 `validate`、既存取り込み `import`、状態操作 `state`、出力参照 `output` を日常的に使う。

## 役割・なぜ必要か
- Terraformの操作は「コードを書く」だけでは完結せず、**プロバイダ取得（init）→ 差分確認（plan）→ 適用（apply）** という決まった手順を回す。各コマンドの役割を知ることが安全な運用の前提。
- `plan` で**適用前に必ず差分を確認**できるのがTerraform最大の安全装置。`apply` の前に何が増減するかを読む習慣が事故を防ぐ。
- `fmt` / `validate` はCIに組み込み、整形崩れ・構文ミスを早期に潰す。

## 基本の書き方（コード）
```bash
# === 初期化 ===
terraform init                      # プロバイダDL・backend初期化・modules取得
terraform init -upgrade             # プロバイダ/モジュールを制約内で更新
terraform init -reconfigure         # backend設定を変えた時に再初期化

# === 差分プレビュー ===
terraform plan                      # 差分表示（+ create / ~ update / - destroy）
terraform plan -out=tfplan          # plan結果をファイル保存（applyで使い回す）
terraform plan -var-file=prod.tfvars
terraform plan -target=aws_instance.web  # 特定リソースだけ（緊急時のみ。乱用禁止）
terraform plan -refresh-only        # driftだけを確認（現実とstateの差）

# === 適用 ===
terraform apply                     # plan→確認(yes)→適用
terraform apply tfplan              # 保存したplanをそのまま適用（差分の取り違え防止）
terraform apply -auto-approve       # 確認を省略（CI向け。手元では非推奨）

# === 削除 ===
terraform destroy                   # 管理下を全削除（検証環境の片付け）
terraform destroy -target=aws_instance.web  # 一部だけ削除

# === 整形・検証 ===
terraform fmt                       # コード整形（-recursive で配下も）
terraform fmt -check                # 整形済みかチェック（CIで使う。差分あれば失敗）
terraform validate                  # 構文・参照の静的チェック（apply不要）

# === 既存リソースの取り込み ===
terraform import aws_instance.web i-0abc123   # 既存をstateに取り込む
                                              # ※ コード側にも resource ブロックが必要

# === 状態操作（直接編集は禁止。必ずコマンド経由） ===
terraform state list                # 管理下のリソース一覧
terraform state show aws_instance.web
terraform state mv  <from> <to>     # state上の住所変更（リファクタ時）
terraform state rm  aws_instance.web # 管理対象から外す（現実は消さない）

# === 出力 ===
terraform output                    # 全outputを表示
terraform output instance_id        # 特定の値だけ
terraform output -json              # JSONで取得（スクリプト連携）
terraform output -raw db_endpoint   # 生の値（クォート無し。シェル変数に入れる時）
```

CI/CDでの定番フロー：
```bash
terraform fmt -check               # 整形チェック
terraform init -input=false        # 対話無効化
terraform validate                 # 構文チェック
terraform plan -out=tfplan -input=false   # plan保存
# （レビュー / 承認ゲート）
terraform apply -input=false tfplan       # 承認後、保存したplanを適用
```

## 実務での使い方・定番パターン
- **`plan -out=tfplan` → `apply tfplan`** の2段がCIの定番。planとapplyの間にレビューを挟みつつ、**apply時に差分が変わらない**ことを保証できる。
- **`fmt -check` と `validate` はCI必須**。整形崩れ・未定義参照をマージ前に弾く。
- **`-target` は緊急避難用**。依存を飛ばして部分適用するためstateが不整合になりやすい。常用しない。
- **既存インフラのコード化は `import`**。手作りリソースをTerraform管理下へ。新しめのバージョンでは `import {}` ブロックで宣言的にも書ける。
- **`output -raw` / `-json`** でTerraformの結果をシェルや別ツールに渡す（DBエンドポイントをデプロイスクリプトへ等）。
- `apply` 時は必ず `plan` の `- destroy` 行を目視確認。意図しない削除が混ざっていないか見る。

## ハマりどころ / アンチパターン
- **`apply -auto-approve` を手元で常用**：差分を見ずに適用して事故。CI以外では確認を挟む。
- **`-target` の常用**：部分適用でstateと現実がズレる。原則は全体を `plan`→`apply`。
- **`state` コマンドを使わず `.tfstate` を手編集**：JSON破損で復旧不能。必ず `terraform state` 経由。→ [state.md](./state.md)
- **`init` し忘れ / backend変更後の `-reconfigure` 忘れ**：`Backend initialization required` エラー。構成変更後は再init。
- **`plan` を保存せず時間を置いて `apply`**：その間に誰かが変更し、planとapplyの差分が食い違う。`-out` で固定して `apply tfplan`。
- **`destroy` を本番で気軽に**：管理下を全削除する。本番には `prevent_destroy`（→ [expressions.md](./expressions.md)）と権限制御で多重に防御。

## 関連
[getting_started.md](./getting_started.md) / [state.md](./state.md) / [expressions.md](./expressions.md) / [environments.md](./environments.md) / [pitfalls.md](./pitfalls.md)
