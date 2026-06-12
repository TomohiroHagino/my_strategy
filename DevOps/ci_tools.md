# CIツール（GitHub Actions ほか）（DevOps）

## ひとことで言うと
**CIツール＝「commitしたら自動でビルド/テスト/デプロイを流す」エンジン。GitHub Actions は GitHubリポジトリ内の `.github/workflows/*.yml` に、いつ(`on`)・何を(`jobs`/`steps`)実行するかを書くだけで動く。**

## 役割・なぜ必要か
- [ci_cd.md](./ci_cd.md) の「パイプライン」を実際に動かす実行基盤。**どのツールでも概念（トリガ→ジョブ→ステップ）は同じ**。
- GitHub Actions はリポジトリに統合されていて導入が軽く、`uses:` で既製のアクションを再利用できる（個人開発〜中規模の第一候補）。

### GitHub Actions の用語（YAMLの骨格）
| 用語 | 意味 |
|---|---|
| **workflow** | `.github/workflows/ci.yml` の1ファイル＝1つの自動化 |
| **`on`** | いつ動かすか（push / pull_request / tag / schedule / workflow_dispatch=手動） |
| **`jobs`** | 並列に走る作業単位。`needs:` で依存（順序）を作れる |
| **runner** | jobを実行するマシン（`runs-on: ubuntu-latest`） |
| **`steps`** | job内の手順。`uses:`（既製アクション）か `run:`（シェル）で書く |
| **`uses:`** | 既製アクションを呼ぶ（例 `actions/checkout@v4`） |
| **`run:`** | 任意のコマンドを実行（例 `npm test`） |
| **secrets** | APIキー等。`${{ secrets.XXX }}` で参照。コードに直書きしない |

## 基本（手順・考え方・コード）

### 完全な workflow YAML 例（build→test→scan→deploy）
`.github/workflows/ci.yml`:
```yaml
name: CI/CD

# いつ動かすか
on:
  pull_request:            # PRごとに検査（本番には出さない）
    branches: [main]
  push:
    branches: [main]       # mainへのマージで staging へ
    tags: ['v*']           # v1.2.3 タグで prod へ

# secretsの権限を絞る（最小権限）
permissions:
  contents: read

jobs:
  # ① テスト（PR・push 両方で実行）
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'                 # 依存をキャッシュして高速化
      - run: npm ci
      - run: npm run lint              # 静的解析（品質ゲート）
      - run: npm test -- --coverage    # テスト＋カバレッジ
      - name: Coverage gate
        run: npx nyc check-coverage --lines 80   # 80%未満で失敗

  # ② 脆弱性スキャン（testと並列）
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high          # 依存の脆弱性(SCA)
      - name: SAST
        uses: github/codeql-action/analyze@v3       # コードの脆弱性(SAST)

  # ③ イメージをビルドして保存（test と scan が通ったら）
  build:
    needs: [test, scan]              # ← 依存。両方緑でないと走らない
    runs-on: ubuntu-latest
    if: github.event_name == 'push'  # PRでは作らない
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}  # SHAで一意に

  # ④ デプロイ（staging or prod）
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: ${{ startsWith(github.ref, 'refs/tags/') && 'production' || 'staging' }}
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}   # secretsから注入
        run: ./scripts/deploy.sh                       # rolling/blue-green等は戦略次第
```

### 読み方のポイント
- `needs: [test, scan]` で「テストとスキャンが緑のときだけビルド」という**順序と前提**を作る。これがパイプラインの段階([ci_cd.md](./ci_cd.md))に対応。
- `${{ github.sha }}` でイメージにコミットハッシュのタグを付け、**同じ成果物を staging→prod に昇格**させる（`latest`は使わない → [pitfalls.md](./pitfalls.md)）。
- `environment:` を使うと「production には承認を必須」などの保護ルールを掛けられる（手動承認＝Continuous Delivery）。
- secretsは必ず `${{ secrets.XXX }}`。**YAMLに鍵を直書きしない**（→ [pitfalls.md](./pitfalls.md)）。

## 実務での使い方・定番パターン
- **PR用とデプロイ用を1ファイルに `on:` で分岐**（上の例）。小規模ならこれで足りる。
- **キャッシュ（`cache:` / actions/cache）で高速化**：依存インストールが毎回数分かかるとパイプラインが遅くなる。
- **matrix ビルド**：複数バージョン/OSを並列検査（`strategy.matrix`）。
- **再利用可能ワークフロー**（`workflow_call`）で複数リポジトリの共通CIをDRYに保つ。
- イメージのビルドは [../インフラ/docker/](../インフラ/docker/) のDockerfileをCIから叩く形。インフラ自体の適用は [../インフラ/terraform/](../インフラ/terraform/) を CI から `plan`/`apply` する（→ [gitops.md](./gitops.md) の push型）。

### ツール比較（どれを選ぶか）
| ツール | 設定場所 | 特徴 | 向く場面 |
|---|---|---|---|
| **GitHub Actions** | `.github/workflows/*.yml` | GitHubに統合・`uses:`で再利用・無料枠あり | GitHub使用・個人〜中規模（第一候補） |
| **GitLab CI** | `.gitlab-ci.yml` | GitLab統合・stages/needsが明快・組込みレジストリ | GitLab使用・セルフホスト志向 |
| **Jenkins** | `Jenkinsfile`(Groovy) | 自前ホスト・プラグイン膨大・何でもできる | 大規模/オンプレ/複雑なレガシー |
| **CircleCI** | `.circleci/config.yml` | 並列実行・orbs(再利用)・高速 | 速度重視・GitHub/Bitbucket |

> どれも「トリガ→jobs→steps」は同じ。GitHub Actionsの`uses`/`run`が分かれば他も読める。

## ハマりどころ / アンチパターン
- **secretsをYAML/コードに直書き**：履歴に残り漏洩する。必ずsecrets機能を使い、漏れたら即ローテート。
- **`actions/xxx@main` のように可変参照で固定**：アクション側の変更で突然壊れる/乗っ取りリスク。**タグかSHAで固定**（`@v4`）。
- **`permissions` を絞らない**：デフォルト権限が広いと、汚染されたPRがリポジトリを書き換えうる。`read`基本＋必要なjobだけ昇格。
- **遅いCIを放置**：キャッシュ無し・直列実行で20分超。誰もマージ前に待たなくなる。並列化・キャッシュ・テスト分割。
- **runnerに状態を持たせる前提**：runnerは毎回まっさら。永続化したいものはartifact/cache/外部ストレージへ。

## 関連: [ci_cd.md](./ci_cd.md) / [automation_quality.md](./automation_quality.md) / [deploy_strategies.md](./deploy_strategies.md) / [gitops.md](./gitops.md) / [pitfalls.md](./pitfalls.md)
道具は [../インフラ/docker/](../インフラ/docker/)（イメージbuild）/ [../インフラ/terraform/](../インフラ/terraform/)（CIからplan/apply）
