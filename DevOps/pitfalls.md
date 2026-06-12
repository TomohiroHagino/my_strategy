# DevOps の罠・アンチパターン（DevOps）

## ひとことで言うと
**DevOps導入で繰り返し踏む地雷をまとめた早見表。「手動デプロイ依存・テスト無しCD・flag乱立・計測しない・gate飛ばし・環境差異・secretsベタ書き」が代表格。症状と対処をセットで持っておく。**

## 役割・なぜ必要か
- DevOpsの失敗は「やり方が間違っている」より「**良いことを中途半端にやって逆効果**」が多い（自動化したのにテストが無い等）。
- 各ファイルの「ハマりどころ」を横断で1枚にし、**先回りで避ける/レビュー時に照合する**チェックリストにする。

## 基本（罠の早見表：症状 → なぜ危険 → 対処）

| 罠 | 症状 | なぜ危険 | 対処 |
|---|---|---|---|
| **手動デプロイ依存** | 本番反映を人が手順書を見て実行 | 属人化・再現性なし・夜間/休日に詰む | パイプライン経由に統一（[ci_cd.md](./ci_cd.md)）。緊急時も手動禁止 |
| **テスト無しCD** | 自動デプロイだがテスト/スキャンが薄い | 壊れたものを高速に本番へ届ける | 品質ゲート必須（[automation_quality.md](./automation_quality.md)） |
| **flag乱立** | feature flagが消されず`if`だらけ | コード複雑化・組合せ爆発・誰も状態を把握できない | フラグに期限・棚卸し・使用後削除（[deploy_strategies.md](./deploy_strategies.md)） |
| **計測しない** | 「速くなった気がする」で改善を語る | 効果不明・劣化に気づけない | DORA/SLOを数値化（[dora_metrics.md](./dora_metrics.md) / [observability.md](./observability.md)） |
| **gate飛ばし常態化** | 「急ぎだから赤でもマージ」が普通に | ゲートが形骸化＝無いのと同じ | 迂回は記録＋事後レビュー必須。重大度で扱い分け |
| **環境差異** | staging用とprod用で別ビルド/別設定 | 「stagingでは動いた」が頻発 | artifactは1度だけ作り昇格。設定は環境変数で外出し |
| **secretsベタ書き** | 鍵/トークンをYAML/コード/Gitに直書き | 履歴に残り漏洩・横展開被害 | secret管理機能/Vault。漏れたら即ローテート（[ci_tools.md](./ci_tools.md)） |

## 実務での使い方・定番パターン（避け方を深掘り）

### 手動デプロイ依存
```
 ✕ 「リリース担当者だけがやり方を知っている」
 ◯ ボタン1つ or タグpushで誰でも同じ手順。手順書はパイプラインそのもの
```
- 緊急時こそパイプライン経由を守る（焦った手作業が二次災害を生む）。

### テスト無し/薄いCD
- 自動化と品質ゲートはセット。**「速く出す」前に「壊れたら止まる」を作る**。順序を間違えると自動事故製造機。

### feature flag乱立
```
 各フラグに: 作成日 / 目的 / オーナー / 撤去予定日 を持たせる
 定期棚卸し: 「常時ON/常時OFFで安定したフラグ」は削除
```

### 計測しない / ダッシュボード放置
- 最初の一歩は「デプロイ頻度」を出すだけでもよい。**数字を会話の中心に**（[culture.md](./culture.md) のSharing）。

### 環境差異（works on my machine の本番版）
```
 原因: prod用に再ビルド / 設定がコードに埋まっている / OS・アーキ差
 対処: 同一imageを stg→prod 昇格 / 設定は env で注入 / コンテナで揃える
```
- コンテナで再現性を揃える → [../インフラ/docker/](../インフラ/docker/)。インフラ差は IaC で揃える → [../インフラ/terraform/](../インフラ/terraform/)。

### secretsの扱い
```
 ✕ ci.yml に API_KEY: "sk-xxxx"
 ◯ ${{ secrets.API_KEY }} で注入 / Vault / クラウドのSecret Manager
 GitOpsでマニフェストをGitに置く時は Sealed Secrets / External Secrets
```
- 一度でも平文でcommitしたら、履歴から消えないものとして**必ずローテート**（→ [gitops.md](./gitops.md) / [ci_tools.md](./ci_tools.md)）。

### その他よくある罠
- **パイプラインが遅い（20分超）**：誰も待たずゲートを迂回。並列化・キャッシュ・テスト分割。
- **ロールバック未検証**：障害中に戻し方が分からない。平時にリハーサル（[deploy_strategies.md](./deploy_strategies.md)）。
- **DBの破壊的変更を一発で**：新旧混在中に列削除→旧版が壊れる。後方互換な段階リリース。
- **アラート過多**：麻痺して本物を見逃す。症状/SLOベースに絞る（[observability.md](./observability.md)）。
- **「DevOpsチーム」という新サイロ**：壁を壊すはずが第3の縦割りを作る（[culture.md](./culture.md)）。
- **犯人探し**：情報が隠れ再発する。blamelessで仕組みを直す（[../インフラ/障害対応/ポストモーテム.md](../インフラ/障害対応/ポストモーテム.md)）。

## ハマりどころ / アンチパターン（メタ）
- **全部一気に直そうとする**：罠は多いが、自チームのDORAを見て**効く順に1つずつ**。
- **ツールで解決しようとし続ける**：多くの罠の根は文化（縦割り・責める・小さく出さない）。ツールだけでは消えない。
- **このリストを"完了チェック"にして満足する**：罠は環境で増える。ポストモーテムから新しい罠を足し続ける。

## 関連: [ci_cd.md](./ci_cd.md) / [ci_tools.md](./ci_tools.md) / [deploy_strategies.md](./deploy_strategies.md) / [automation_quality.md](./automation_quality.md) / [observability.md](./observability.md) / [gitops.md](./gitops.md) / [dora_metrics.md](./dora_metrics.md) / [culture.md](./culture.md)
道具側の罠は [../インフラ/docker/pitfalls.md](../インフラ/docker/pitfalls.md) / [../インフラ/terraform/pitfalls.md](../インフラ/terraform/pitfalls.md) / [../インフラ/aws/pitfalls.md](../インフラ/aws/pitfalls.md) / [../インフラ/負荷検証/pitfalls.md](../インフラ/負荷検証/pitfalls.md)
