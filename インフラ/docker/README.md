# Docker 実務リファレンス（索引）

> **Docker** ＝ アプリを「実行に必要なもの一式（コード・ランタイム・ライブラリ・設定）」ごと箱に詰めて、どこでも同じように動かす仕組み。**「自分の環境では動く」を消すための再現性の道具**。
> 流れは一本道：**`Dockerfile`（作り方のレシピ）→ `build` → `image`（読み取り専用テンプレ）→ `run` → `container`（実行中のインスタンス）**。
> 各ファイルは「ひとことで言うと → 役割・なぜ → 基本の書き方（コード）→ 実務の定番 → ハマり所 → 関連」。

## まず押さえる3本柱
```
 ① イメージとコンテナ … Dockerfile→build→image→run→container の関係を体に入れる
 ② Dockerfile          … レイヤ・キャッシュ・マルチステージ。ここで再現性とサイズが決まる
 ③ Compose            … 複数コンテナ（app+db+...）を1ファイルで束ねて起動する
```

## まず手を動かす（実習）
- [ハンズオン.md](./ハンズオン.md) … **0からコピペで動く実習**。小さなアプリに Dockerfile→`build`→`run -p`→`curl`→`compose`でapp+DBの2コンテナ→`logs`/`exec`。わざと「ポート競合 / CMD間違い / COPY漏れ」を起こして直す

## 項目（各ファイルへ）

### 基礎
- [getting_started.md](./getting_started.md) … 始め方（インストール / hello-world / build→run の最小例 / フォルダ構成）
- [images_containers.md](./images_containers.md) … イメージとコンテナの違い・レイヤ・tag・レジストリ

### イメージを作る / 束ねる
- [dockerfile.md](./dockerfile.md) … Dockerfile命令一覧・レイヤキャッシュ・マルチステージビルド
- [compose.md](./compose.md) … Compose（compose.yaml / 複数サービス / depends_on / サービス名通信）

### 永続化 / 接続 / 操作
- [volumes_networks.md](./volumes_networks.md) … Volume vs Bind mount（永続化）/ Network（コンテナ間通信）
- [commands.md](./commands.md) … よく使うコマンド早見（build/run/ps/logs/exec/stop/rm/prune …）

### 品質 / 運用
- [best_practices.md](./best_practices.md) … イメージ最小化・.dockerignore・非root・レイヤ順・tag戦略
- [production.md](./production.md) … 本番運用（restart / resource limits / ログ / レジストリ / 脆弱性スキャン）
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ（latest / キャッシュ / データ消失 / アーキ差 …）

> ※ 本番の**オーケストレーション**（多数コンテナの自動配置・自己修復・スケール）は Docker 単体の範囲外。
> マネージドに寄せるなら → [../aws/containers.md](../aws/containers.md)（ECS / Fargate / EKS）

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Docker）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
