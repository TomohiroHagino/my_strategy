# GitOps（DevOps）

## ひとことで言うと
**「あるべき状態」をGitに宣言的に書き、Gitを唯一の信頼の源(Single Source of Truth)として、その通りに本番を自動で一致させ続けるやり方。デプロイ＝Gitにマージすること、になる。**

## 役割・なぜ必要か
- 従来の「CIサーバから本番へコマンドを送り込む(push)」は、誰がいつ何を打ったかが追いにくく、本番の実状態とコードがズレやすい。
- GitOpsは**「本番の状態＝Gitの中身」を常に一致させる**。Gitを見れば本番が分かり、変更履歴・レビュー・ロールバックが全部Gitの仕組みに乗る。
- 思想は Terraform の「宣言的に状態を書く」と同じ → [../インフラ/terraform/](../インフラ/terraform/)。GitOpsはそれをKubernetes等のデプロイに広げ、**継続的に自動同期**する点が特徴。

## 基本（手順・考え方・コード）

### 4原則
```
 ① 宣言的       : 望む状態をマニフェスト(YAML)で書く（手順でなく結果を書く）
 ② バージョン管理: そのマニフェストをGitに置く（履歴・レビュー・戻しが付く）
 ③ 自動適用     : Gitの状態を環境に自動で反映する
 ④ 継続的に収束  : 実状態がズレたら検知し、Gitの状態へ戻す（self-heal）
```

### push型 と pull型
```
【push型】CIが外から本番へ送り込む（従来のCD）
  Git ─▶ CI(GitHub Actions) ─── kubectl/terraform apply ──▶ 本番(クラスタ)
                                （CIが本番への認証情報を持つ）

【pull型】本番側のエージェントが自分でGitを見に行き、自分を合わせる（GitOpsの主流）
  Git ◀──── 監視/取得 ──── Argo CD / Flux（クラスタ内に常駐）
                            └─ Gitと実状態の差分を検知 ▶ 自動で適用・自己修復
```
| | push型 | pull型(GitOps) |
|---|---|---|
| 適用の主体 | 外部のCI | クラスタ内エージェント |
| 認証情報 | CIが本番権限を持つ（漏れると危険） | エージェントが内側から。**外部に本番鍵を渡さない** |
| ドリフト検知 | 基本なし | あり（ズレを自動で戻す） |
| 代表ツール | GitHub Actions＋kubectl/terraform | **Argo CD / Flux** |

### 代表ツール
- **Argo CD**：GitのマニフェストとKubernetesの実状態を比較し、差分をUIで見せて同期。「どこがズレているか」が見やすい。
- **Flux**：同じくpull型でGitと同期。GitOpsの草分け。軽量でコントローラ志向。

### 典型の構成（リポジトリを分ける）
```
 app-repo（アプリのコード）       ──CIでimageをbuild──▶ レジストリ(image:sha)
        │                                                    │
        │ 新しいimageタグを更新                                │
        ▼                                                    │
 config-repo（k8sマニフェスト/Gitが本番の真実）◀─────────────────┘
        │
        ▼
 Argo CD/Flux が config-repo を監視 ─▶ 差分があればクラスタへ適用
```
> アプリのコードと「デプロイ設定」を分けるのが定番。設定リポジトリへのマージ＝本番反映、になる。

## 実務での使い方・定番パターン
- **本番変更は必ずGit経由**にする。`kubectl edit` で直接いじるのは禁止（Gitとズレ、すぐ戻される）。
- **ロールバック＝Gitをrevert**。「前のコミットに戻す」だけで本番も戻る。履歴が監査ログになる。
- **環境ごとにディレクトリ/ブランチを分ける**（dev/staging/prod）。Terraformの環境分けと同じ発想 → [../インフラ/terraform/](../インフラ/terraform/) の environments。
- **CIはimageを作るところまで、CDはGitOpsエージェントが担当**、と役割分担すると綺麗（CIに本番鍵を持たせない）。
- 動かす基盤（クラスタ/コンテナ）は [../インフラ/aws/containers.md](../インフラ/aws/containers.md) / [../インフラ/docker/](../インフラ/docker/)。

## ハマりどころ / アンチパターン
- **手で本番を直す（ドリフト）**：GitOpsツールが「Gitと違う」と判断して勝手に戻し、変更が消える。緊急時も原則Git経由。
- **secretを平文でGitに入れる**：マニフェストをGit管理する性質上やりがち。**Sealed Secrets / External Secrets** 等で暗号化・外部参照する（→ [pitfalls.md](./pitfalls.md)）。
- **app-repoとconfig-repoを混ぜる**：アプリのcommitごとに本番が動いて制御しづらい。設定は分離。
- **同期を自動(auto-sync)にしたまま検証不足**：マージ即本番。重要環境は手動同期や承認を挟む。
- **小さく始めない**：いきなり全環境GitOps化で詰まる。1サービスから。

## 関連: [ci_cd.md](./ci_cd.md) / [ci_tools.md](./ci_tools.md) / [deploy_strategies.md](./deploy_strategies.md) / [pitfalls.md](./pitfalls.md)
宣言的IaCの土台は [../インフラ/terraform/](../インフラ/terraform/)。動かす基盤は [../インフラ/aws/containers.md](../インフラ/aws/containers.md) / [../インフラ/docker/](../インフラ/docker/)
