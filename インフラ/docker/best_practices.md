# ベストプラクティス（Docker）

## ひとことで言うと
イメージを**小さく・安全に・再現可能に**作るための定石。**.dockerignore で不要物を弾く・小さいベース・マルチステージ・レイヤ順でキャッシュを効かす・非rootで動かす・1コンテナ1プロセス・tagを固定・HEALTHCHECK・秘密を焼かない**。

## 役割・なぜ必要か
- イメージが大きいと、ビルド・push・pull・起動が遅くなり、含まれるパッケージが多いほど**脆弱性の数も増える**。小さく保つことは速度とセキュリティの両方に効く。
- レイヤ順を最適化するとビルドキャッシュが効き、CI が速くなる。逆に順序が悪いと毎回フルビルドになる。
- root 実行・秘密の焼き込み・`latest` 依存は、いずれも**いつか事故になる**典型。最初から避ける設計にしておくと後で困らない。

## 基本の書き方（コード）
小さく安全な Dockerfile（マルチステージ＋非root＋固定tag）：
```dockerfile
# 固定tag（digest 固定ならさらに堅い：node:20.11-slim@sha256:...）
FROM node:20.11-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20.11-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app/dist ./dist
USER node                      # 非rootで実行
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"] # exec形式（シグナルが正しく届く）
```

`.dockerignore`（build context を絞る＝高速＆秘密混入防止）：
```dockerignore
.git
node_modules
.env
*.log
dist
coverage
Dockerfile
.dockerignore
.github
**/*.md
```

ビルド時の秘密は焼かず BuildKit secret で渡す（履歴に残さない）：
```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npmtoken \
    NPM_TOKEN=$(cat /run/secrets/npmtoken) npm ci
```
```bash
DOCKER_BUILDKIT=1 docker build --secret id=npmtoken,src=./npm_token.txt -t myapp:1.0 .
```

apt のレイヤを膨らませない（1レイヤで入れて残骸を消す）：
```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl \
 && rm -rf /var/lib/apt/lists/*
```

## 実務での使い方・定番パターン
- **ベースは用途で最小に**：`alpine`（極小・muslで稀に互換問題）、`slim`（debian系の軽量・無難）、`distroless`（シェルすら無い・最小攻撃面・本番向け）。Go/Rust は static バイナリ＋`scratch`も選択肢。
- **レイヤ順は「変わりにくい順に上」**：システム依存 → 言語依存（lockファイル）→ アプリコード。下ほど頻繁に変わる物を置く。
- **tag は固定、できれば digest 固定**。`node:20`より`node:20.11-slim`、再現性最優先なら `@sha256:...` まで固定。`latest` は使わない。
- **1コンテナ1プロセス**。app と DB を1コンテナに同居させない。プロセス管理は compose / オーケストレータに任せる。
- **ログは stdout/stderr へ**。ファイルに書かず標準出力に流すと、Docker・compose・本番基盤がそのまま集約できる（→ [production.md](./production.md)）。
- **イメージを脆弱性スキャン**：`docker scout cve myapp:1.0` や Trivy を CI に入れる（→ [production.md](./production.md)）。

## ハマりどころ / アンチパターン
- **`.dockerignore` 無しで巨大 context**：`node_modules`・`.git`・`.env` まで送られ、遅い・大きい・秘密混入。必ず置く。
- **root 実行のまま**：脱獄時の被害が大きい。`USER` で非root。書き込みが要るパスの権限も合わせる。
- **秘密を `ENV`/`ARG` に焼く**：`docker history`/`inspect` で露出。BuildKit secret か実行時注入に。
- **`latest` 依存**：別マシン・別時刻で中身が変わり再現しない。固定tag/digest。
- **巨大シングルレイヤ or キャッシュ無視の順序**：`COPY . .` を先頭に置いてキャッシュを殺す、`apt` 残骸を消さずレイヤ肥大。順序と後始末を整える。
- **`CMD` をシェル形式**で書きシグナルが届かない：exec形式 `["...","..."]` を使う。

## 関連
[dockerfile.md](./dockerfile.md) / [production.md](./production.md) / [images_containers.md](./images_containers.md) / [pitfalls.md](./pitfalls.md)
