# 実務でハマる罠まとめ（Pitfalls）（Hanami 2 ＋ Rails との使い分け）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。**Rails の感覚で書くと事故る箇所**を中心に、症状から該当箇所へ素早く飛ぶための索引。

## 役割・なぜ必要か
- Hanami 2 は「明示・疎結合・DI」が前提。Rails の「規約・暗黙・グローバル」発想のまま書くと噛み合わない。**発想の違いが事故ポイント**になる。

## アクション / ビュー
- **アクション = 1クラス**：Rails のような「1コントローラに複数アクション」ではなく `#handle(req, res)` を持つ**1クラス1アクション**。→ [actions.md](./actions.md)
- **View と Action は分離**：アクションは応答準備、描画は独立した View クラス＋テンプレート（exposures でデータ受け渡し）。混ぜない。→ [views_templates.md](./views_templates.md)
- **インスタンス変数が勝手にビューに渡らない**：Rails の `@var` 自動共有は無い。exposures で明示的に渡す。→ [views_templates.md](./views_templates.md)

## 永続化 / データ
- **ORM は ROM／リポジトリで ActiveRecord ではない**：`User.find` のようなモデル直叩きは無い。**Relation で問い合わせ、Repository 経由**で取得。→ [persistence_rom.md](./persistence_rom.md)
- **エンティティは「振る舞いを持たないデータ」寄り**：ActiveRecord 的にエンティティへロジックや永続化を持たせない。操作は Repository / Operation へ。→ [persistence_rom.md](./persistence_rom.md)
- **コールバック地獄が無い代わりに自分で繋ぐ**：`after_save` 的フックは無い。副作用は Operation で明示的に呼ぶ。→ [persistence_rom.md](./persistence_rom.md)

## DI / コンテナ
- **DIコンテナのキー = パス規約**：`app/operations/create_user.rb` → `"operations.create_user"`。パスとキーがズレると解決できない。→ [dependency_injection.md](./dependency_injection.md)
- **依存は `Deps[...]` で注入**：内部で直接 `new` せず `include Deps["..."]`。差し替え可能性とテスト容易性のため。→ [dependency_injection.md](./dependency_injection.md)
- **Rails 感覚でグローバル参照を探さない**：`Rails.application` 的な万能オブジェクトは無い。必要な依存だけを注入する。→ [dependency_injection.md](./dependency_injection.md)

## Slices / 設定 / 検証
- **Slices の境界**：slice はサブアプリ。コンテナ・依存は slice 単位で分離。slice を跨ぐ参照は明示的に。→ [slices.md](./slices.md)
- **dry-validation の schema / rule 分離**：型・必須は `params`（schema）、業務ルールは `rule` で分ける。混ぜると見通しが崩れる。→ [validation.md](./validation.md)
- **settings の型指定漏れ / 必須欠落**：ENV は文字列。`Types::Params::*` で型付けし、必須は default を付けない。参照は `Deps["settings"]`。→ [settings_config.md](./settings_config.md)

## テスト
- **コンテナから取得して DI で差し替え**：`Hanami.app["..."]` で取り出し、`container.stub` でモック化。Rails 流の `type: :model` 発想は通じない。→ [testing.md](./testing.md)

## エコシステム / 学習コスト
- **gem が Rails 前提のことが多い**：ActiveRecord 依存の gem はそのまま使えない場合あり。ROM/dry-rb 対応か要確認。
- **学習コストが高め・コミュニティが小さい**：情報量が Rails より少ない。dry-rb / ROM の理解が前提になる。

## Hanami vs Rails 使い分け
- **規約で最速に立ち上げたい・CRUD中心・人員の Rails 習熟が高い → Rails**：規約優先・暗黙の自動化で開発速度が出る。小〜中規模やプロトタイプに強い。
- **明示・疎結合・DI・大規模長期保守・ドメイン分割（slices）したい → Hanami**：依存が可視化され、テスト容易・変更に強い。チームで設計境界を保ちたい大規模に向く。
- 迷ったら：**速度と既存知見なら Rails、保守性とアーキテクチャ規律なら Hanami**。

## 関連
[actions.md](./actions.md) / [views_templates.md](./views_templates.md) / [persistence_rom.md](./persistence_rom.md) / [dependency_injection.md](./dependency_injection.md) / [slices.md](./slices.md) / [validation.md](./validation.md) / [settings_config.md](./settings_config.md) / [testing.md](./testing.md)
