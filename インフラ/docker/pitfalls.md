# 実務でハマる罠まとめ（Docker）

## ひとことで言うと
Docker で繰り返し踏まれる地雷の一覧と対処。**latest tag・キャッシュでコードが反映されない・volume忘れでデータ消失・build context肥大・root実行・arm/amdアーキ差・ポート競合・秘密のベタ書き**。原因と直し方をセットで覚える。

## 役割・なぜ必要か
- これらは「動くけど後で事故る」「環境を変えると壊れる」類の罠。先に知っておくと、レビューやトラブル時に一発で原因に当たれる。
- 多くは**再現性・データ保全・セキュリティ**のどれかを壊す。Docker を使う動機そのものを潰しかねないので、最初に潰しておく価値が高い。

## 基本の書き方（コード）
### 1. `latest` tag で再現しない
別マシン・別時刻で `latest` が別バージョンになり挙動が変わる。
```dockerfile
FROM node:latest        # ✗ いつ pull したかで中身が変わる
FROM node:20.11-slim    # ◎ 固定。digest固定ならさらに堅い
```

### 2. キャッシュでコードが反映されない
`COPY . .` を先頭に置くとキャッシュが効いてコード変更が無視される、または逆に毎回フルビルド。
```dockerfile
# ◎ 依存を先・コードを後（キャッシュが正しく効く）
COPY package*.json ./
RUN npm ci
COPY . .
```
```bash
docker build --no-cache -t myapp:1.0 .   # どうしても疑わしい時はキャッシュ無効化
```

### 3. volume 忘れでデータ消失
コンテナの書き込みは `rm` で消える。DBデータ等は volume に逃がす。
```bash
docker run -d --name db postgres:16                       # ✗ rm でデータ消滅
docker run -d --name db -v db-data:/var/lib/postgresql/data postgres:16  # ◎
```

### 4. build context 肥大
`.dockerignore` が無いと巨大ディレクトリ・秘密まで送られる。
```dockerignore
.git
node_modules
.env
dist
*.log
```

### 5. root 実行
既定 root のままだと脱獄時の被害が大きい。
```dockerfile
RUN useradd -m app   # 必要なら作成
USER app             # 非rootに切り替え
```

### 6. arm / amd アーキ差（Apple Silicon ⇄ x86サーバ）
Mac(arm64) でビルドしたイメージが x86 本番で動かない（`exec format error`）。
```bash
# マルチアーキでビルドして push（buildx）
docker buildx build --platform linux/amd64,linux/arm64 \
  -t ghcr.io/org/myapp:1.0 --push .

# 単発で本番アーキに合わせる
docker build --platform linux/amd64 -t myapp:1.0 .
```

### 7. ポート競合
`port is already allocated` ＝ ホスト側ポートが使用中。
```bash
docker run -p 8080:3000 myapp     # 8080 が空いていなければ衝突
lsof -i :8080                      # 誰が使っているか調べて
docker run -p 8081:3000 myapp     # 別ポートに変える
```

### 8. 秘密のベタ書き
`ENV`/`ARG` やイメージに焼くと `history`/`inspect` で露出する。
```dockerfile
ENV DB_PASSWORD=secret    # ✗ イメージ層に残り消えない
```
```bash
# ◎ 実行時に注入（イメージに焼かない）
docker run --env-file .env myapp:1.0
# ◎ ビルド時の秘密は BuildKit secret（→ best_practices.md）
```

## 実務での使い方・定番パターン
- **疑わしい挙動はまず `--no-cache` と固定tagで再現確認**。キャッシュとtag揺れが原因の大半。
- **「消えて困るもの」を起動前に洗い出す**。DB・アップロード・生成物は volume へ。`compose down -v` を共有環境で打たない。
- **CIでビルドするアーキを本番に合わせる**。Apple Silicon 開発・x86本番なら `--platform linux/amd64` か buildx マルチアーキ。
- **秘密は3か所に書かない**：Dockerfile・イメージ・git。実行時注入（`--env-file`/シークレットマネージャ）に統一。

## ハマりどころ / アンチパターン（要点再掲）
- **`latest` 依存** → 固定tag/digest。
- **`COPY . .` 先頭でキャッシュ事故** → 依存を先、コードを後。
- **volume 無しのDB** → 永続パスを volume に。
- **`.dockerignore` 無し** → context肥大・秘密混入。必ず置く。
- **root 実行** → `USER` で非root。
- **アーキ不一致（`exec format error`）** → `--platform` / buildx。
- **ポート競合** → ホスト側ポートを変える / 使用プロセスを止める。
- **秘密の焼き込み** → 実行時注入・BuildKit secret。
- **`localhost` でコンテナ間通信** → サービス名/コンテナ名で繋ぐ（→ [volumes_networks.md](./volumes_networks.md)）。
- **ログが `docker logs` に出ない** → アプリを stdout 出力にする（→ [best_practices.md](./best_practices.md)）。

## 関連
[best_practices.md](./best_practices.md) / [dockerfile.md](./dockerfile.md) / [volumes_networks.md](./volumes_networks.md) / [compose.md](./compose.md) / [production.md](./production.md) / [commands.md](./commands.md)
