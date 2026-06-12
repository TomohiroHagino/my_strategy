# Django

## 一言で
Pythonで最も定番のフルスタックWebフレームワーク。「**電池付属（batteries included）**」を掲げ、ORM・**自動生成される管理画面**・認証・フォームまで標準で揃い、堅実に速く作れる。

## 特徴
- **MTV アーキテクチャ**: Model（データ）/ Template（見た目）/ View（処理）。**Djangoの「View」は他FWのコントローラ役**で、テンプレートが「ビュー」相当。ここが用語の罠。
- **強力なORM**: モデル＝テーブル。`Article.objects.filter(...)` で直感的にDB操作。マイグレーション標準。
- **管理画面（admin）**: モデルを登録するだけでCRUD管理UIが自動生成される。Django最大の武器。
- **forms / ModelForm**: 入力検証とHTML生成を一体で扱う。
- **project と app の二層構造**: 1つのプロジェクトに機能単位の「app」を複数ぶら下げる。
- **セキュリティ既定が堅実**: CSRF・XSS・SQLi対策が標準でオン。

## どういう使い方をするのか
- **DB駆動のWebアプリ／業務システム／CMS**（特に管理画面が要るもの）。
- **API バックエンド**（Django REST Framework が定番）。
- 大きめ・堅めの組織システムで安定運用したいとき。

## 強み
- admin / 認証 / forms が標準で「よくある要件」が即揃う。
- セキュリティとドキュメントが堅実・安定（LTSあり）。
- ORM・マイグレーションが成熟。

## 弱み・注意点
- 規約・全部入りゆえ重め。小さなAPIだけならFastAPI/Flaskが軽い。
- 非同期/リアルタイムは後付け感（成長中）。フロントは別途。
- ORMの手軽さでN+1やfat modelになりやすい。

## エコシステム・周辺ツール
- API: **Django REST Framework（DRF）**
- 非同期タスク: **Celery**（＋Redis/RabbitMQ）/ Django-Q
- 設定: django-environ / python-dotenv
- テスト: pytest-django / factory_boy
- 実行: Gunicorn / Uvicorn（ASGI）＋ nginx、静的配信 WhiteNoise

## ひとことメモ（自分の実感）
- （現役として触った所感を後から追記）

## このフォルダの構成
- [django5/](./django5/) … **Django 5 実務リファレンス（フラッグシップ）**。プロジェクトの始め方〜MTV〜ORM〜admin〜DRF〜テスト〜罠まで、項目=1ファイル。
- （5.2 が LTS。差分は django5 の各「この版のポイント」に補記。他バージョンは一旦作らない）
