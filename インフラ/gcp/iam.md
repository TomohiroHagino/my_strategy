# IAM（GCP）

## ひとことで言うと
IAM（Identity and Access Management）＝**認証認可の仕組み**。「**誰が（メンバー）**・**何をできるか（ロール）**・**どのリソースに対して**」の3点セットで、GCP上の操作権限を管理する。

## 役割・なぜ必要か
- GCPはあらゆる操作に権限チェックが入る。IAMは「この人/このアプリに、このリソースで、この操作を許す」を宣言する場所。
- 基本構造は **メンバー（principal）＋ロール（role）＋リソース** の組み合わせで「**ポリシーバインディング**」を作ること。
  - **メンバー**：人間ユーザー（Googleアカウント）、グループ、そして **サービスアカウント**（アプリやVMが名乗る非人間アカウント）。
  - **ロール**：許される操作の束。**事前定義ロール**（`roles/storage.objectViewer` 等、用途別に細かい）と、自分で必要権限だけ集めた **カスタムロール** がある。
- ねらいは **最小権限（least privilege）**。必要な操作だけを必要なメンバーに与えることで、漏洩・誤操作の被害を最小化する。
- アプリ（VMやCloud Run）が他サービスを呼ぶときは、人間ではなく **サービスアカウント** に権限を付けて使わせるのが基本形。

## 基本の使い方（gcloud / コンソール）
```bash
# プロジェクトに対して人間ユーザーへロールを付与
gcloud projects add-iam-policy-binding my-app-123456 \
  --member="user:alice@example.com" \
  --role="roles/storage.objectViewer"

# 現在のポリシー（誰に何を）を確認
gcloud projects get-iam-policy my-app-123456

# 付与を外す
gcloud projects remove-iam-policy-binding my-app-123456 \
  --member="user:alice@example.com" \
  --role="roles/storage.objectViewer"
```

```bash
# サービスアカウントを作り、ロールを付与
gcloud iam service-accounts create my-app-sa \
  --display-name="My App Service Account"

gcloud projects add-iam-policy-binding my-app-123456 \
  --member="serviceAccount:my-app-sa@my-app-123456.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"
```

- **メンバー指定の接頭辞**が重要：人間は `user:`、グループは `group:`、サービスアカウントは `serviceAccount:`。
- **コンソール**では「IAMと管理 → IAM」で一覧・追加・編集。「ロールを推奨してくれる」機能（過剰権限の指摘）もある。
- VMやCloud Runには「実行時に名乗るサービスアカウント」を割り当てられる。アプリのコードはキー無しでそのSAの権限を使える（**キーレス**）。

## 実務での勘所
- **最小権限を徹底**：まず狭い事前定義ロールを探す。`roles/storage.objectViewer` のように「サービス.対象.動詞」で命名されているので用途を絞りやすい。
- 個人ではなく **グループ単位** で権限管理すると、入退社・異動時の付け替えが楽。
- アプリの認証は **キーレスを最優先**。VM/Cloud Run/GKE に紐付けたサービスアカウントを使えば、鍵ファイルを持たずに済む。
- 権限は **組織 → フォルダ → プロジェクト → リソース** と上位から **継承** される。プロジェクトに付けた権限は中の全リソースに効く。
- カスタムロールは「事前定義では広すぎ/狭すぎ」のときの最終手段。メンテコストがかかるのでまず事前定義で足りないか確認。

## ハマりどころ / アンチパターン
- **基本ロール（Owner/Editor/Viewer）の乱用**：`roles/editor` は広すぎて最小権限に反する。本番では事前定義ロールへ置き換える。→ [getting_started.md](./getting_started.md)
- **サービスアカウントキーの漏洩**：JSON鍵ファイルをコミット/共有して流出する事故が多い。可能なら**鍵を作らずキーレス**にする。作るなら定期ローテーションと厳格な保管を。
- **権限の継承を見落とす**：「このプロジェクトの誰も触れないはず」が、組織/フォルダ階層で付いた権限で実は触れる、という事故。上位のポリシーも確認する。
- **付けっぱなし権限**：検証で付けた強権限を剥がし忘れる。`get-iam-policy` で棚卸しする習慣を。
- サービスアカウントを「ユーザー」と混同して `user:` で指定し、付与が効かない。接頭辞 `serviceAccount:` を使う。

## 関連
[getting_started.md](./getting_started.md)
