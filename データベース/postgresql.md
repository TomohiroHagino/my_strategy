# PostgreSQL

## ひとことで言うと
**世界で最も先進的なオープンソースRDB**を標榜するリレーショナルデータベース。SQL標準準拠が厳格で、多機能かつ拡張可能。「とりあえずこれを選んでおけば外さない」近年のデファクト。

> "The World's Most Advanced Open Source Relational Database" を公式キャッチコピーに掲げる。単なるデータ置き場ではなく、型・関数・拡張機能まで含めた「拡張できるDBエンジン」というのが核心。

## 役割・立ち位置
- 主戦場は **OLTP**（1行ずつの読み書き：Webアプリの裏など）だが、ウィンドウ関数やCTE、全文検索といった**分析寄りの機能も強い**ため、軽い集計ならそのままこなせる。
- よく **MySQL** と比較される。ざっくり言うと「MySQLは速さと手軽さ／PostgreSQLは多機能と厳密さ」。
- **コミュニティ主導**のプロジェクトで、特定企業の所有物ではない（MySQLがOracle傘下なのと対照的）。この中立性も採用される理由のひとつ。
- 近年はクラウドのマネージドサービス（RDS / Cloud SQL / Aurora 等）と相まって、新規プロジェクトの**デフォルト選択肢**になりつつある。

## 多機能さ（MySQLとの最大の差）
PostgreSQLの真価は**型と機能の豊かさ**。MySQLにない／弱い部分が多い。

- リッチな型：**JSONB**（インデックスを張れるバイナリJSON）・**配列**・**range型**（範囲）・**ENUM**・独自型（CREATE TYPE）まで作れる。
- SQL機能：**CTE（WITH句）**・**再帰CTE**・**ウィンドウ関数**・**全文検索**が標準搭載。
- **拡張（EXTENSION）**で機能を後付けできる：地理空間なら **PostGIS**、ベクトル検索なら **pgvector**（RAG/類似検索で定番）など。

```sql
-- JSONB: JSONを列に入れ、キーで検索・インデックスもできる
SELECT * FROM events WHERE payload->>'type' = 'click';

-- 配列: 1列に複数値。要素検索も可能
SELECT * FROM posts WHERE 'postgres' = ANY(tags);

-- CTE(WITH): 中間結果に名前を付けて読みやすく組み立てる
WITH recent AS (
  SELECT user_id, COUNT(*) AS n
  FROM events
  WHERE created_at > now() - interval '7 days'
  GROUP BY user_id
)
SELECT * FROM recent WHERE n > 100;
```

## MVCC とトランザクション
- **追記型MVCC**を採用。更新は既存行を書き換えるのではなく、**新しいバージョンの行を書き、古い行を残す**。これにより読み取りは書き込みをブロックせず、同時実行性が高い。
- **デフォルトの分離レベルは `READ COMMITTED`**。ここは要注意で、**MySQL（InnoDB）のデフォルトである `REPEATABLE READ` とは違う**。移植時にトランザクションの見え方が変わるので意識する。
- `SERIALIZABLE` も用意されており、PostgreSQLのそれは **SSI（Serializable Snapshot Isolation）** による「本物のシリアライズ可能性」を提供する（直列実行と等価な結果を保証）。

## VACUUM（運用の肝・重要）
追記型MVCCの代償として、**削除・更新で「不要になった古い行（dead tuple）」が物理的に溜まる**。これを回収するのが **VACUUM**。

- `autovacuum` が自動でバックグラウンド実行され、dead tupleの領域を再利用可能にする（＋プランナ用の統計も更新）。
- ただし**放置すると危険**。書き込みが多いテーブルでautovacuumが追いつかないと、テーブル肥大化（bloat）→ スキャン量増 → **性能劣化**を招く。
- 最悪は **トランザクションID周回（wraparound）**。古い行の凍結（freeze）が間に合わないとDBが書き込みを止めて自衛する事態になる。

> VACUUMは **MySQLには存在しない、PostgreSQL特有の運用ポイント**。「入れたら終わり」ではなく、書き込みの多いテーブルはautovacuumの効きを監視するのが実務の常識。

## インデックスの勘所
B-treeだけでなく、用途別に複数のインデックス型を持つのが強み。

| 種類 | 得意なもの |
|---|---|
| B-tree | 等価・範囲・ソート（デフォルト・基本これ） |
| GIN | JSONB・全文検索・配列の「中身」検索 |
| GiST | 地理空間（PostGIS）・範囲型 |
| BRIN | 巨大な追記専用テーブル（時系列ログ等）を軽量に |
| Hash | 等価検索のみ |

- **部分インデックス**（条件付き）や**式インデックス**（関数の結果に張る）も使える。
- 効いているかは **`EXPLAIN (ANALYZE)`** で実行計画を確認するのが鉄則。

```sql
-- 通常のB-tree
CREATE INDEX idx_events_user ON events (user_id);

-- JSONBにはGIN
CREATE INDEX idx_events_payload ON events USING GIN (payload);

-- 部分インデックス: 有効ユーザーだけに張る
CREATE INDEX idx_active_users ON users (email) WHERE is_active;
```

## 接続モデルの注意（重要）
- PostgreSQLは **1接続 = 1 OSプロセス**。接続ごとにプロセスがforkされる。
- このため**接続そのものが重く**、接続数が増えるほどメモリを食う。数千接続を素で張ると簡単に枯渇する。
- **多数の短命接続（Webアプリ・サーバーレス等）には PgBouncer などのコネクションプーラがほぼ必須**。アプリとDBの間に挟み、少数の実接続を使い回す。

> MySQLが「1接続 = 1スレッド」で接続が比較的軽いのとは対照的。PostgreSQLでは「接続をプールする」前提で設計する。

## 基本操作
```bash
# psql で接続
psql "postgresql://user:password@localhost:5432/mydb"

# ホスト・DB・ユーザーを個別指定でも可
psql -h localhost -p 5432 -U user -d mydb
```

```sql
-- テーブル作成。主キーの自動採番は今風の IDENTITY を使う
CREATE TABLE users (
  id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  uuid       UUID DEFAULT gen_random_uuid(),
  name       VARCHAR(100) NOT NULL,
  bio        TEXT,
  profile    JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()  -- タイムゾーン付き推奨
);

-- 基本CRUD
INSERT INTO users (name, bio) VALUES ('alice', 'hello');
SELECT id, name FROM users WHERE name = 'alice';
UPDATE users SET bio = 'updated' WHERE name = 'alice';
DELETE FROM users WHERE name = 'alice';
```

よく使う型：`INT` / `BIGINT`（整数）、`TEXT` / `VARCHAR`（文字列）、`TIMESTAMPTZ`（タイムゾーン付き時刻）、`JSONB`（JSON）、`UUID`（識別子）。

## 強み / 弱み
**強み**
- 多機能（JSONB・配列・全文検索・ウィンドウ関数など）で、別ミドルウェアを足さずに済む場面が多い。
- SQL標準準拠が厳格で、**データ整合性に厳しい**（型・制約をきっちり効かせる）。
- 拡張（PostGIS / pgvector 等）でDB自体を育てられる。
- 複雑なクエリ・JOIN・集計に強い。

**弱み**
- **VACUUMという固有の運用負荷**がある（autovacuum監視が要る）。
- **接続数に弱く**、プーラ前提の設計が必要。
- **レプリケーションがMySQLより設定が重め**（物理＝ストリーミング / 論理＝logical replication の2系統があり、選定と運用に知識が要る）。
- 機能が多いぶん**学習要素が多い**。

## ハマりどころ / アンチパターン
- **autovacuumを軽視してbloat**：書き込みの多いテーブルで放置すると肥大化・性能劣化、最悪 wraparound。監視と必要なら手動 `VACUUM` / 設定調整を。
- **接続プール無しで接続枯渇**：1接続1プロセスを忘れてアプリから大量接続を張ると、メモリ不足で倒れる。PgBouncer等を挟む。
- **`TIMESTAMP`（タイムゾーン無し）と `TIMESTAMPTZ` の混同**：素の `TIMESTAMP` を使うと時差バグの温床。基本は **`TIMESTAMPTZ`** を選ぶ。
- **`SERIAL` を今でも使う**：`SERIAL` は古い書き方。今風は **`GENERATED ALWAYS AS IDENTITY`**（SQL標準）。
- **大文字を含む識別子**：`CREATE TABLE "Users"` のように作ると、以後**常にダブルクオートが必須**になり事故る。識別子は小文字スネークケースで統一する。
- **JSONBに何でも放り込む**：柔軟さに甘えて正規化を捨てると、検索・整合性・JOINで苦しむ。構造が決まっているものは普通の列に分ける。

## 関連
- MySQLとの比較は [mysqlとpostgresqlの違い.md](./mysqlとpostgresqlの違い.md)
- マネージド運用は [../インフラ/aws/](../インフラ/aws/) や [../インフラ/gcp/cloud_sql.md](../インフラ/gcp/cloud_sql.md)
- SQL・スキーマの設計判断は [../設計/](../設計/)
