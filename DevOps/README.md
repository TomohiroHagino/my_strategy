# DevOps 実務リファレンス（索引）

> **DevOps ＝ 開発(Dev)と運用(Ops)の壁を壊し、「自動化」と「文化」で、速く・安全に・継続的にリリースし続ける取り組み。**
> 個々の道具（コンテナ・IaC・クラウド・監視・負荷試験）は既に別フォルダにある。**このフォルダはそれらを"つなぐ"上位レイヤ**＝「コードを書いてから本番に届け、運用し、計測して改善する」までの一連の流れ（パイプラインと文化）を扱う。
> 各ファイルは「ひとことで言うと → 役割・なぜ → 基本（手順・コード・図）→ 実務での使い方 → ハマりどころ → 関連」。

## 全体像（コード→CI→CD→運用→計測のループ）
```
        ┌──────────────────────────────────────────────┐
        │                                                          │
        ▼                                                          │
   ① コード      ② CI            ③ CD              ④ 運用        ⑤ 計測
  commit/PR ─▶ build/test/scan ─▶ deploy(stg→prod) ─▶ monitor/incident ─▶ DORA/SLO
  小さく頻繁    自動で品質を担保   戦略的に本番反映     可観測性で見る   速さ/安定を数値化
        │                                                          │
        └────────────── 計測結果を次の改善へ戻す ───────────────────┘
```
**ポイント：これは一回きりの手順ではなく「回し続けるループ」**。計測（⑤）の結果を①〜④の改善に戻すのがDevOpsの肝。

## CALMS（DevOpsの5本柱）
DevOpsは「ツール」ではなく、ツールと文化の組み合わせ。判断に迷ったらこの5つに立ち返る。

| 頭文字 | 意味 | 中身 |
|---|---|---|
| **C** | Culture（文化） | サイロ解消・責任の共有・blameless。**まずここ**。ツールだけ入れても変わらない → [culture.md](./culture.md) |
| **A** | Automation（自動化） | ビルド/テスト/デプロイ/インフラを手作業から外す → [ci_cd.md](./ci_cd.md) / [ci_tools.md](./ci_tools.md) |
| **L** | Lean（無駄を削る） | 小さく頻繁に流す・仕掛り(WIP)を減らす・リードタイム短縮 |
| **M** | Measurement（計測） | 推測でなく数字で改善する（DORA・SLO） → [dora_metrics.md](./dora_metrics.md) / [observability.md](./observability.md) |
| **S** | Sharing（共有） | 知見・ポストモーテム・ダッシュボードをチーム横断で開く |

## 繋ぎ先の地図（既存リファレンスとの関係）
DevOpsは「流れ」を扱い、具体的な道具は既存フォルダに任せる。**重複は書かず、ここから辿る。**

```
              DevOps（このフォルダ＝つなぐ流れ・文化・計測）
                          │
   ┌──────────┬──────────┼──────────┬──────────────┐
   ▼          ▼          ▼          ▼              ▼
 コンテナ     IaC        クラウド    監視/障害       性能
 docker/    terraform/  aws/       障害対応/       負荷検証/
 何で運ぶか  何で土台を  どこで動か  壊れたとき/    限界を知る
 (image)    作るか      すか        見るとき       (負荷試験)
```

| DevOpsの関心 | 具体的な道具は… | リンク |
|---|---|---|
| パイプラインで何を「build/run」するか | コンテナ（image/Dockerfile/Compose） | [../インフラ/docker/](../インフラ/docker/) |
| インフラを「コードで」用意・変更する | Terraform（init/plan/apply・state・module） | [../インフラ/terraform/](../インフラ/terraform/) |
| デプロイ先・実行基盤・マネージドサービス | AWS（ECS/Fargate/Lambda/RDS…） | [../インフラ/aws/](../インフラ/aws/) |
| デプロイ後に「見る」「壊れたら戻す」 | 監視・障害対応（SEV・止血・ポストモーテム） | [../インフラ/障害対応/](../インフラ/障害対応/) |
| リリース前に「壊れる前の限界を知る」 | 負荷検証（p95/p99・飽和点・k6） | [../インフラ/負荷検証/](../インフラ/負荷検証/) |

## 項目（各ファイルへ）

### 流れ（CI/CD・デプロイ）
- [ci_cd.md](./ci_cd.md) … CI/CDの概念。CI＝頻繁な統合＋自動build/test、CD＝自動でリリース可能/本番反映。パイプライン段階（commit→build→test→scan→artifact→deploy）
- [ci_tools.md](./ci_tools.md) … CIツール。**GitHub Actions の完全なworkflow YAML例**（`on`/`jobs`/`steps`/`uses`/`run`）＋ GitLab CI / Jenkins / CircleCI 比較
- [deploy_strategies.md](./deploy_strategies.md) … デプロイ戦略。rolling / blue-green / canary / feature flag ＋ ロールバック（各図と使い分け）

### 計測・運用
- [dora_metrics.md](./dora_metrics.md) … DORA 4 Keys（デプロイ頻度／リードタイム／変更失敗率／復旧時間）の定義・測り方・Elite〜Lowの目安
- [observability.md](./observability.md) … 可観測性の3本柱（metrics/logs/traces）・SLI/SLO/エラーバジェット
- [gitops.md](./gitops.md) … GitOps（Gitを信頼の源に宣言的デプロイ・Argo CD/Flux・push型/pull型）

### 品質・文化
- [automation_quality.md](./automation_quality.md) … 品質ゲート（lint/型/test/coverage/SAST/SCA/コンテナスキャン）・シフトレフト
- [culture.md](./culture.md) … CALMS・サイロ解消・トランクベース開発・blameless・You build it you run it
- [pitfalls.md](./pitfalls.md) … 罠まとめ（手動デプロイ依存・テスト無しCD・flag乱立・計測しない・gate飛ばし・環境差異・secretsベタ書き）

## 各ファイルの書式（テンプレ）
```markdown
# {テーマ名}（DevOps）
## ひとことで言うと
## 役割・なぜ必要か
## 基本（手順・考え方・コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```

## 関連
[ci_cd.md](./ci_cd.md) / [ci_tools.md](./ci_tools.md) / [deploy_strategies.md](./deploy_strategies.md) / [dora_metrics.md](./dora_metrics.md) / [observability.md](./observability.md) / [gitops.md](./gitops.md) / [automation_quality.md](./automation_quality.md) / [culture.md](./culture.md) / [pitfalls.md](./pitfalls.md)
道具は [../インフラ/docker/](../インフラ/docker/) / [../インフラ/terraform/](../インフラ/terraform/) / [../インフラ/aws/](../インフラ/aws/) / [../インフラ/障害対応/](../インフラ/障害対応/) / [../インフラ/負荷検証/](../インフラ/負荷検証/)
