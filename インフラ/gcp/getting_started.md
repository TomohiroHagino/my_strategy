# 始め方（GCP）

## ひとことで言うと
Google Cloud Platform（GCP）を使い始めるための土台。**アカウント → プロジェクト → API有効化 → 課金/予算**の順に整える。GCPの全リソースは必ず「**プロジェクト**」に属する、という大原則がすべての出発点。

## 役割・なぜ必要か
- GCPでは VM もストレージも DB も、すべて何らかの **プロジェクト** の中に作られる。プロジェクトは「課金・権限・リソースをまとめる箱」であり、料金請求も IAM 権限もプロジェクト単位で効く。
- プロジェクトには人間向けの **プロジェクト名** と、世界で一意な **プロジェクトID（project_id）** がある。CLI・API・課金はこの **project_id** で対象を指定するため、ここを取り違えると「別プロジェクトに作ってしまった」事故が起きる。
- GCPは「使いたい機能ごとに **API を有効化** する」設計。Compute Engine を使うなら Compute Engine API、と都度ONにしないと、認証が通っていてもコマンドが弾かれる。
- 個人検証では **無料枠（Free Tier）** と **予算アラート** を最初に押さえると、想定外の高額請求を防げる。

## 基本の使い方（gcloud / コンソール）
```bash
# 1) CLIの初期設定（ログイン＋プロジェクト/リージョン選択をまとめて）
gcloud init

# 2) 認証だけ単体でやり直す場合
gcloud auth login                 # 人間ユーザーとしてログイン
gcloud auth application-default login   # アプリ（SDK）が使う既定認証

# 3) プロジェクトの作成・確認・切り替え
gcloud projects create my-app-123456 --name="My App"
gcloud projects list
gcloud config set project my-app-123456     # 以降の操作対象を固定
gcloud config get-value project              # 今どのプロジェクトか確認（最重要）

# 4) 使うAPIを有効化（サービスごとに必要）
gcloud services enable compute.googleapis.com        # Compute Engine
gcloud services enable run.googleapis.com            # Cloud Run
gcloud services list --enabled                       # 有効なAPI一覧
```

```bash
# 5) 課金アカウントの確認とプロジェクトへの紐付け
gcloud billing accounts list
gcloud billing projects link my-app-123456 \
  --billing-account=XXXXXX-XXXXXX-XXXXXX
```

- **コンソール**（console.cloud.google.com）では、画面上部のプロジェクト選択プルダウンが「今どのプロジェクトを触っているか」を表す。ここを常に意識する。
- **API有効化**はコンソールの「APIとサービス → ライブラリ」から検索してON、でもよい。
- **予算アラート**は「お支払い → 予算とアラート」で金額としきい値（例: 50%/90%/100%）を設定。メール通知が飛ぶ。

## 実務での勘所
- 環境ごとにプロジェクトを分ける（`my-app-dev` / `my-app-stg` / `my-app-prod`）。課金・権限・誤操作の影響範囲を分離できる。
- `gcloud config configurations` で「dev用」「prod用」など設定セットを切り替えると、project / アカウントの取り違えを減らせる。
- 新しいサービスを触る前に「**このサービスのAPIは有効か**」を確認する癖をつける。`gcloud services list --enabled` が早い。
- 無料枠は「Always Free（小さな常時無料枠）」と「初回クレジット（期間限定）」が別物。期限切れ後に課金へ移る点に注意。
- 予算アラートは**課金を止める機能ではない**（あくまで通知）。本当に止めたいなら別途プログラムで対処が必要。

## ハマりどころ / アンチパターン
- **プロジェクトの取り違え**：`gcloud config get-value project` を確認せず作業し、別プロジェクトにVMやバケットを作ってしまう。作業前の確認を徹底。→ [iam.md](./iam.md)
- **APIの有効化し忘れ**：`PERMISSION_DENIED` や `API has not been used...` のエラーは権限ではなくAPI未有効が原因のことが多い。まず `services enable` を疑う。
- **課金/予算アラート未設定**：検証用VMやログ転送が裏で動き続け、月末に高額請求。最初に予算アラートを置く。
- `gcloud auth login` と `application-default login` の混同：前者はCLI操作用、後者はアプリ（SDKコード）用。アプリから認証エラーが出るときは後者を確認。
- project_id は**作成後に変更不可**。命名規約（環境サフィックス等）を最初に決めておく。

## 関連
[iam.md](./iam.md) / [compute_engine.md](./compute_engine.md)
