# Hanami 2 実務リファレンス（索引）

> **対象 = Hanami 2.x（Ruby 3.x、dry-rb / ROM）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> Hanami 2 は v1 から全面刷新。核は「**DIコンテナ＋明示的なコンポーネント分割**：アクション・ビュー・永続化を分け、依存は `Deps[...]` で注入」。

## この版のポイント（Hanami 2）
- **`app/` 中心**（2.1+）＋ **`slices/`**（サブアプリ）。
- **DIコンテナ**：`app/` 配下のクラスが自動でコンテナに登録され、`include Deps["..."]` で注入。
- **アクション = 1クラス**（`#handle(request, response)`）。
- **ビューは独立**（View クラス＋テンプレート、exposures でデータ受け渡し）。
- **永続化は ROM**（Relations / Repositories）。2.2 で DB 層が統合。
- **dry-validation** の Contract でパラメータ検証。

## 項目（各ファイルへ）

### はじめに / 構成
- [getting_started.md](./getting_started.md) … 始め方（hanami new / server / console）
- [request_flow.md](./request_flow.md) … リクエスト/処理の流れ・各層は何を返すか（全体像）
- [project_structure.md](./project_structure.md) … 構成（app / slices / コンテナ自動登録）とは
- [routing.md](./routing.md) … ルーティング（config/routes.rb）とは

### MVC相当（Hanami流）
- [actions.md](./actions.md) … アクション（1アクション=1クラス）とは
- [views_templates.md](./views_templates.md) … ビュー / テンプレート（exposures）とは
- [dependency_injection.md](./dependency_injection.md) … DIコンテナ（Deps[]）とは ← Hanamiの核

### データ・検証・分割
- [persistence_rom.md](./persistence_rom.md) … 永続化（ROM / Repository）とは
- [slices.md](./slices.md) … Slices（サブアプリ）とは ← Hanami固有
- [validation.md](./validation.md) … バリデーション（dry-validation Contract）とは

### 設定・運用
- [settings_config.md](./settings_config.md) … 設定（Hanami::Settings / dry-types）とは
- [testing.md](./testing.md) … テスト（RSpec / コンテナからの取得）とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ（＋Rails との使い分け）

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Hanami 2）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
