# 実務でハマる罠まとめ（Terraform）

## ひとことで言うと
Terraform運用で「やらかしがち」な事故と対策を一覧化したもの。多くは **state・drift・秘密情報・apply順序・削除事故** の5系統に集約される。個別の詳細は各トピックへリンクする。

## 役割・なぜ必要か
- Terraformは強力だが、操作が**現実のインフラを直接・大量に変更・削除する**ため、ミスの被害が大きい。事故パターンを先に知っておくことが最大の防御。
- 特に **state破損・秘密情報漏洩・意図しないdestroy** は、一度起きると復旧が困難。先回りでガードを入れる。

## 1. State の事故
- **`*.tfstate` をgitコミット**：秘密情報の漏洩＋チームで競合。→ `.gitignore` に必ず追加し、リモートbackend（暗号化＋バージョニング）へ。
- **ロック無しで同時apply**：stateが壊れリソースが二重作成・行方不明に。→ DynamoDB（S3 backend）等でロック必須。
- **stateの手編集**：JSON破損で復旧不能。→ 必ず `terraform state mv / rm / import` 経由。
- **巨大モノリシックstate**：planが激遅、1ミスが全体波及。→ 用途・環境でbackendの `key` を分割。
詳細 → [state.md](./state.md)

## 2. Drift（手動変更とのズレ）
- **コンソールで手動変更**：次の `apply` でTerraformが元に戻す / 競合エラー。→ 変更はコード経由に統一。検知は `terraform plan -refresh-only`。
- **`ignore_changes` でdriftを隠す**：差分が出ないが現実と乖離して把握不能に。→ オートスケールのdesired_count等、本当に管理外の属性に限定。
詳細 → [state.md](./state.md) / [expressions.md](./expressions.md)

## 3. 秘密情報
- **tfvars / コードに秘密値を直書きしてコミット**：DBパスワード等が漏洩。→ `TF_VAR_*` 環境変数 / Secrets Manager（`data` で参照）。`*.tfvars` はgitignore。
- **outputに `sensitive` 付け忘れ**：CI/CDログに平文出力。→ 秘密の出力は `sensitive = true`。
- **stateに秘密が平文で残る**：`sensitive` を付けてもstate本体には平文。→ backend暗号化＋アクセス制御で守る。
詳細 → [variables_outputs.md](./variables_outputs.md) / [state.md](./state.md)

## 4. apply順序・依存
- **暗黙の依存を書き忘れ**：IAMポリシー反映前にインスタンスが起動して失敗等。→ 参照で繋がらない依存は `depends_on` で明示。
- **`-target` の常用で部分適用**：依存を飛ばしてstate不整合。→ 緊急時のみ。原則は全体 plan→apply。
- **`for_each` に確定しない値**：`Invalid for_each argument`。→ `toset()` 等で確定値を渡す。
詳細 → [expressions.md](./expressions.md) / [commands.md](./commands.md)

## 5. 削除事故（destroy / 再作成）
- **本番リソースをうっかりdestroy / 再作成**：DBやバケットが消失。→ 本番DB・stateバケットに `lifecycle { prevent_destroy = true }`。
- **`count` の途中要素を削除→後続が全部作り直し**：インデックスずれ。→ 集合は `for_each`（キー安定）を使う。
- **属性変更で「強制再作成（forces replacement）」を見落とす**：`plan` の `# forces replacement` を見ずにapplyしてダウンタイム。→ planを精読。必要なら `create_before_destroy`。
- **モジュール構造のリファクタでstateの住所がズレて削除→再作成**：→ `terraform state mv` で住所を引っ越す。
詳細 → [expressions.md](./expressions.md) / [modules.md](./modules.md) / [state.md](./state.md)

## 6. バージョン / 初期化まわり
- **プロバイダ/モジュールの `version` 未固定**：ある日 `init`/`apply` が突然壊れる。→ `~> 5.0` 等で固定し、`.terraform.lock.hcl` をコミット。
- **`.terraform/` をコミット**：プロバイダ本体（巨大）が混入。→ gitignore。
- **backend変更後に再init忘れ**：`Backend initialization required`。→ `terraform init -reconfigure`。
詳細 → [getting_started.md](./getting_started.md) / [providers_resources.md](./providers_resources.md)

## 7. 環境分けの事故
- **dev/prodをworkspaceで切替→ prodのまま destroy**：→ 本番は別ディレクトリ＋別権限で物理分離。
- **環境でstateを分けていない**：dev applyがprodを触る。→ backendの `key` / workspaceで必ず分離。
詳細 → [environments.md](./environments.md)

## 実務での使い方・定番パターン（事故を減らす運用）
- **必ず `plan` を読んでから `apply`**。`- destroy` と `# forces replacement` を目視確認。
- **`fmt -check` / `validate` / `plan` をCIで自動化**し、マージ前に弾く。
- **本番は多重防御**：`prevent_destroy` ＋ 最小権限のIAM ＋ 手動承認ゲート ＋ stateバックアップ。
- **秘密はコードに入れない**を徹底（`TF_VAR_*` / シークレットマネージャ）。
- **小さく分割**（state・モジュール・環境）して、1操作の影響範囲を狭く保つ。

## 関連
[state.md](./state.md) / [expressions.md](./expressions.md) / [variables_outputs.md](./variables_outputs.md) / [commands.md](./commands.md) / [environments.md](./environments.md) / [modules.md](./modules.md) / [../aws/pitfalls.md](../aws/pitfalls.md) / [../gcp/pitfalls.md](../gcp/pitfalls.md)
