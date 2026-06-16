# データベース（DBエンジン）

> リレーショナルDB（RDB）の**エンジンそのもの**——SQL・型・トランザクション・インデックス・運用——を製品別にまとめる場所。
> 「クラウドでマネージドに動かす」話（RDS / Cloud SQL / Aurora）は [../インフラ/](../インフラ/) 側、「大量データをSQLで殴る分析基盤」は [../インフラ/gcp/bigquery.md](../インフラ/gcp/bigquery.md)。ここは**DBソフトの中身**のレイヤ。

## まず立ち位置（OLTP の2大OSS）
```
 MySQL      … 世界で最も普及。Web(LAMP)の定番。枯れた運用・読み取り速い・情報量が多い
 PostgreSQL … 最も先進的を標榜。多機能・SQL標準準拠が厳格・拡張可能。近年デファクト化
```
どちらも**1行ずつの読み書き（OLTP）** が主戦場。全件集計の分析（OLAP）は BigQuery のような専用基盤の役目。

## 項目（各ファイルへ）
- [mysql.md](./mysql.md) … MySQL（8.0系）。InnoDB・分離レベル既定 REPEATABLE READ・utf8mb4 の罠・インデックス・レプリケーション。
- [postgresql.md](./postgresql.md) … PostgreSQL（16系）。JSONB/配列/拡張・分離レベル既定 READ COMMITTED・**VACUUM**・1接続1プロセス（プーラ必須）。
- [mysqlとpostgresqlの違い.md](./mysqlとpostgresqlの違い.md) … **どっちを選ぶか**。10観点の比較表・性能/運用の傾向・選定判断・移行時のハマりどころ。
- [redis.md](./redis.md) … Redis（インメモリKVS）。**直接書き込む方法（redis-cli・各クライアント）** ・データ型（Hash/List/Set/Sorted Set）・TTL・永続化・`KEYS *` の罠。

## 各ファイルの書式（テンプレ）
```markdown
# {製品名}
## ひとことで言うと
## 役割・立ち位置
## （ストレージエンジン / MVCC / トランザクション / インデックス / 文字コード …トピック節）
## 強み / 弱み
## ハマりどころ / アンチパターン
## 関連
```

## 今後足せる候補
- SQLite（組み込み・1ファイルDB・モバイル/エッジ）
- MongoDB（ドキュメント指向NoSQL）
- インデックス設計の深掘り（B-tree/複合/カバリング/部分・実行計画の読み方）を製品横断で
- トランザクション分離レベル横断（READ系の比較・ファントム・MVCCの実装差）

## 関連
- マネージド運用 → [../インフラ/aws/](../インフラ/aws/) / [../インフラ/gcp/cloud_sql.md](../インフラ/gcp/cloud_sql.md)
- データウェアハウス（分析・OLAP） → [../インフラ/gcp/bigquery.md](../インフラ/gcp/bigquery.md)
- SQL設計の判断（リポジトリ・正規化・N+1） → [../設計/](../設計/)
