# CI/CD（DevOps）

## ひとことで言うと
**CI＝コードを頻繁に統合し、その都度自動でビルド/テストして「壊れていない」を保つ仕組み。CD＝そこから先を自動化し、いつでもリリースできる状態に保つ（CDeli）／実際に本番へ反映する（CDep）仕組み。**

## 役割・なぜ必要か
- **手作業のリリースは事故る・遅い・属人化する**。「金曜の夜にベテランが手順書を見ながら30分かけてデプロイ」は再現性が無く、ミスが本番に直結する。
- CI/CDは「commitしたら、あとは自動で品質チェックと配布が流れる」状態にすることで、**小さな変更を・速く・安全に・何度でも**出せるようにする。
- 効果は数字で出る → 速さ（デプロイ頻度・リードタイム）と安定（変更失敗率・復旧時間）の両方が良くなる。測り方は [dora_metrics.md](./dora_metrics.md)。

### CI と CD の用語整理（混同しやすい）
| 略 | 正式 | 意味 | どこまで自動か |
|---|---|---|---|
| **CI** | Continuous Integration | 頻繁な統合＋自動build/test | mainへの統合とテストまで |
| **CD** | Continuous **Delivery** | 「いつでも本番に出せる」状態を自動で保つ | 本番直前まで自動。**本番反映は人がボタンを押す** |
| **CD** | Continuous **Deployment** | テストを全部通ったら**本番反映まで自動** | 人の承認なしで本番へ |

> 同じ「CD」でも Delivery（人が最後にOKを出す）と Deployment（全自動）は別物。組織の成熟度・規制で使い分ける。多くのチームはまず Delivery から。

## 基本（手順・考え方・コード）

### パイプラインの段階（左から右へ流れる）
```
 commit ─▶ build ─▶ test ─▶ scan ─▶ artifact ─▶ deploy
   │        │        │       │        │           │
  PR/push  コンパイル 単体/結合 lint    成果物を    staging ─▶ prod
  をトリガ  ・依存解決 ・型・E2E SAST/SCA  保存       自動で    戦略的に
                              脆弱性    (image等)   検証       反映
```
- **左ほど速く・安く・頻繁に。右ほど重く・本番に近い**。失敗は左で早く落とすほど安い（シフトレフト → [automation_quality.md](./automation_quality.md)）。
- **artifact（成果物）は一度だけ作る**。同じものをstaging→prodへ昇格させる。「prod用に再ビルド」は環境差異の原因（→ [pitfalls.md](./pitfalls.md)）。
- ここで作る artifact の中身が**コンテナイメージ**になることが多い → [../インフラ/docker/](../インフラ/docker/)。

### 各段階で何をするか
| 段階 | やること | 失敗したら |
|---|---|---|
| **build** | コンパイル・依存解決・イメージ作成 | 即停止（先へ進めない） |
| **test** | 単体/結合/E2E・カバレッジ確認 | 即停止。テスト無しのCDは禁物 |
| **scan** | lint/型・SAST(コード脆弱性)・SCA(依存脆弱性)・コンテナスキャン | 重大度で止める/警告（品質ゲート） |
| **artifact** | image/jar等を**バージョン付きで**レジストリへ保存 | リトライ |
| **deploy** | staging→（検証）→prod。戦略はrolling/blue-green/canary | ロールバック（→ [deploy_strategies.md](./deploy_strategies.md)） |

### トリガ（いつ動かすか）の典型
```
 PR作成/更新   → build + test + scan（マージ前にチェック。本番には出さない）
 mainへマージ  → 上記 + artifact作成 + stagingへ自動deploy
 tag(v1.2.3)  → prodへdeploy（リリースはタグ駆動にすると履歴が綺麗）
 定期(cron)    → 依存の脆弱性再スキャン・夜間E2E
```

## 実務での使い方・定番パターン
- **PRごとに必ずCIを回し、緑(pass)でなければマージできない**ようブランチ保護をかける。これが品質の最低ライン。
- **trunk-based（短命ブランチ・1日以内にmainへ）** とCIは相性が良い。大きな長寿命ブランチはマージ地獄とCIの意味低下を招く → [culture.md](./culture.md)。
- **staging を本番相当に保つ**。staging で問題なし→prod、の流れが効くのはstagingが本番に近いとき。性能面の確認は [../インフラ/負荷検証/](../インフラ/負荷検証/)。
- **デプロイとリリースを分離**：コードを本番に置く(deploy)のと、ユーザーに機能を見せる(release)を分ける。feature flag で実現 → [deploy_strategies.md](./deploy_strategies.md)。
- ツールの具体（YAML）は [ci_tools.md](./ci_tools.md)。

## ハマりどころ / アンチパターン
- **テスト無し・スキャン無しのCD**：自動で本番に壊れたものを高速に届けるだけ。速さは事故も速くする。
- **パイプラインが遅すぎる（20分超）**：誰もマージ前に待たなくなり、CIを迂回し始める。並列化・キャッシュ・テスト分割で短く。
- **環境ごとに別々にビルド**：staging用とprod用で別image＝「stagingでは動いたのに」の温床。artifactは作り直さない。
- **mainが赤いまま放置**：CIが赤いのが常態化すると、本物の故障に気づけなくなる。赤は最優先で直す（止める文化）。
- **手動デプロイへの逆戻り**：「今回は急ぎだから手で」が積もって自動化が形骸化する。緊急時こそパイプライン経由を守る。

## 関連: [ci_tools.md](./ci_tools.md) / [deploy_strategies.md](./deploy_strategies.md) / [automation_quality.md](./automation_quality.md) / [dora_metrics.md](./dora_metrics.md)
道具は [../インフラ/docker/](../インフラ/docker/)（build成果物）/ [../インフラ/負荷検証/](../インフラ/負荷検証/)（リリース前の性能確認）
