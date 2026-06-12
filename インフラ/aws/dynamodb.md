# DynamoDB（AWS）

## ひとことで言うと
**フルマネージドなNoSQL（キーバリュー型 / KVS）データベース**。テーブルにスキーマ（列定義）はほぼ無く、**キーでアイテムを引く**ことに特化。サーバー管理不要で、規模に応じて自動でスケールする。

## 役割・なぜ必要か
- RDB（RDS）は「自由なSQLで柔軟に検索」が得意な一方、**超大量アクセスでの安定した低レイテンシ**や**運用の手離れ**は苦手な面がある。DynamoDBはそこを埋める。
- 「**ユーザーIDで引く**」「**セッションIDで引く**」のように**アクセスパターンが決まっている**用途（セッション・カート・IoT・ゲームのスコア・通知履歴）で強い。
- ミリ秒級の応答を**規模が増えても維持**でき、サーバー台数を意識しなくてよいのが本質的な価値。
- キー構造：
  - **パーティションキー（PK）**：データを物理的に分散させる軸。必須。
  - **ソートキー（SK）**：同じPK内での並び替え・範囲検索の軸（任意）。PK+SKで**複合キー**になる。

## 基本の使い方（CLI / コンソール）
```bash
# テーブル作成（PK=userId, SK=createdAt、オンデマンド課金）
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
    AttributeName=userId,AttributeType=S \
    AttributeName=createdAt,AttributeType=S \
  --key-schema \
    AttributeName=userId,KeyType=HASH \
    AttributeName=createdAt,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST
```
```bash
# 書き込み（PutItem）
aws dynamodb put-item --table-name Orders \
  --item '{"userId":{"S":"u-001"},"createdAt":{"S":"2026-06-11T10:00:00Z"},"amount":{"N":"1200"}}'

# 読み取りはまず Query（キーで引く＝速い・安い）
aws dynamodb query --table-name Orders \
  --key-condition-expression "userId = :u" \
  --expression-attribute-values '{":u":{"S":"u-001"}}'

# 範囲検索（SKで「この日付以降」など）
aws dynamodb query --table-name Orders \
  --key-condition-expression "userId = :u AND createdAt > :d" \
  --expression-attribute-values '{":u":{"S":"u-001"},":d":{"S":"2026-06-01"}}'
```
- **GSI（グローバルセカンダリインデックス）**：本来のキー以外の属性でも引けるよう、**別のキー構造を持つ"影テーブル"**を追加する仕組み。「メールアドレスで引きたい」等を後付けできる。
- コンソール：DynamoDB → 「テーブルの作成」→ PK/SK指定 → 容量モード（オンデマンド/プロビジョンド）。

## 実務での勘所
- **アクセスパターンを先に洗い出してからキーを設計する**。RDBの「とりあえず正規化」とは逆で、**どう引くかが決まらないとテーブルが設計できない**。
- 課金モードは2つ：
  - **オンデマンド（PAY_PER_REQUEST）**：リクエスト数に応じた従量。トラフィックが読めない/波があるなら基本これ。
  - **プロビジョンド**：あらかじめ **WCU/RCU（書き込み/読み取り容量）** を確保。常時安定した負荷ならこちらが安い。Auto Scaling併用可。
- **単一テーブル設計**：複数の種類のデータ（ユーザー・注文・商品）を**1テーブルにまとめ、PK/SKの値の付け方で表現**するDynamoDB特有の設計。JOINが無い世界で関連データを1回のQueryで取るための定石。慣れないうちは無理に1テーブルにせず、用途別に分けてよい。
- **TTL** で「一定時間後に自動削除」（セッション・一時データに便利）。

## ハマりどころ / アンチパターン
- **クエリはキー設計が前提**：DynamoDBは「キーで引く」前提の作り。**後から「別の条件で検索したい」が増えると詰む**（RDBのように `WHERE 任意カラム` ができない）。GSIで救えるが、設計段階でアクセスパターンを固めるのが正攻法。
- **`Scan` の乱用**：`Scan` は**全アイテムを総なめ**する操作。データが増えるほど遅く・高くなる。本番の検索を `Scan` で実装したら黄信号。**`Query`（キー指定）で引ける設計**にする。
- **ホットパーティション**：PKの値が偏る（例：PKを `"all"` 固定）と、**特定パーティションにアクセスが集中**してスロットリング（throttle）が起きる。PKは**カーディナリティ（値の種類）が高い**ものを選ぶ。
- **容量（WCU/RCU）の見積もりミス**：プロビジョンドで容量を低く設定すると**スロットリングでエラー**。逆に高すぎると無駄な課金。読めないならオンデマンドが無難。
- **大きいアイテム**：1アイテム最大400KB。巨大データは本体をS3に置きキーだけ保存する。
- **RDBの感覚で使う**：トランザクション/結合/集計を多用したいならRDS（RDB）の方が素直。「**何でもDynamoDB**」は失敗のもと。→ [rds.md](./rds.md)

## 関連
[rds.md](./rds.md) / [s3.md](./s3.md) / [lambda.md](./lambda.md)
