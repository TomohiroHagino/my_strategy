# IaC（CloudFormation / CDK / Terraform）（AWS）

## ひとことで言うと
**Infrastructure as Code**＝インフラ構成（VPC・EC2・S3・IAM…）を**コードで宣言して管理する**こと。コンソールで手作業せず、**コードを実行すれば同じ環境が再現**でき、差分・履歴・レビューが効く。

## 役割・なぜ必要か
- 手作業（クリックポチポチ）は**再現不能・属人化・設定漏れ**の温床。コード化すれば **Git で履歴管理・PR レビュー・環境複製**ができる。
- `dev` / `stg` / `prod` を**同じコードからパラメータ違いで量産**でき、環境差異による事故が減る。
- 代表的な3つ：
  - **CloudFormation**：AWS 純正。**YAML/JSON で宣言**。スタック単位で作成/更新/削除。学習コスト中、AWS 限定。
  - **CDK**：**TypeScript/Python など普通のコードで書き**、内部で CloudFormation テンプレへ変換。ループ・条件・抽象化が効き大規模向き。
  - **Terraform**：HashiCorp 製。**マルチクラウド**（AWS 以外も同じ書き方）。**state（状態ファイル）**で現状を管理。エコシステムが巨大。
- 「スタック（stack）」＝まとめてライフサイクルを共にするリソースの束。作成・削除はこの単位。

## 基本の使い方（CLI / コンソール）
CloudFormation（YAML テンプレ）：
```yaml
# template.yaml — S3 バケットを1つ作る最小例
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-iac-bucket-2026
      VersioningConfiguration:
        Status: Enabled
Outputs:
  BucketName:
    Value: !Ref MyBucket
```
```bash
# デプロイ（スタック作成/更新）と削除
aws cloudformation deploy --template-file template.yaml --stack-name my-stack
aws cloudformation delete-stack --stack-name my-stack
```
CDK（TypeScript）：
```bash
npx cdk init app --language typescript
npx cdk diff      # 変更差分を確認（適用前に必ず見る）
npx cdk deploy    # CloudFormation スタックとして反映
npx cdk destroy   # 破棄
```
Terraform：
```bash
terraform init     # プロバイダ取得・state 初期化
terraform plan     # 実行計画（差分）を表示
terraform apply    # 反映（state を更新）
terraform destroy  # 破棄
```
```hcl
# main.tf — S3 バケットを1つ作る最小例
resource "aws_s3_bucket" "my" {
  bucket = "my-iac-bucket-2026"
}
```

## 実務での勘所
- **適用前に必ず差分を見る**：`cdk diff` / `terraform plan` / CloudFormation の**変更セット（change set）**。何が変わるか確認せず apply は事故のもと。
- **状態（state）管理はチームの最重要**：Terraform は state を **S3 バックエンド + DynamoDB ロック**で共有・排他する（ローカル state を Git に置くのは厳禁）。CloudFormation/CDK は状態を AWS 側が持つので楽。
- **環境分離**：ワークスペース/パラメータ/スタック分割で `dev/stg/prod` を切る。**本番だけ手で変えない**を徹底。
- **シークレットを直書きしない**：パスワード・キーは Secrets Manager / SSM Parameter Store を参照。state やテンプレに平文で残さない。→ [iam.md](./iam.md)
- **小さく分割**：1スタックに全部詰めず、ネットワーク層・アプリ層・データ層で分けると影響範囲とデプロイ時間を抑えられる。
- 作ったリソースは**監視と一緒にコード化**しておく（アラーム/ダッシュボードも IaC で）。→ [monitoring.md](./monitoring.md)

## ハマりどころ / アンチパターン
- **手動変更とのドリフト（drift / 差分）**：コンソールで直接いじると IaC の認識と実体がズレ、次の apply で**手動変更が上書き/破壊**される。`terraform plan` で意図しない差分が出る、CloudFormation の `detect-drift` で乖離が見える。**変更は必ずコード経由**を鉄則に。
- **state ファイルの破損・喪失・競合**：Terraform で state をローカル放置/共有なしだと、2人同時 apply で壊れる。**リモート state + ロック**を最初に用意する。state には平文の機微情報が入りうるので**暗号化とアクセス制限**も必須。
- **削除順序・依存でスタックが消えない/壊れる**：他スタックから参照（Export/Import や `depends_on`）されているリソースは消せず `delete-stack` が `DELETE_FAILED`。**依存の逆順で削除**、循環参照を作らない。空でない S3 バケットや ENI 残留で削除が止まるのも頻出。
- **巨大モノリススタック**：全リソースを1スタックにすると、些細な変更で全体が再評価され遅く・危険。**境界で分割**する。
- **`terraform destroy` / `delete-stack` の事故**：本番に向けて誤実行で全消し。**本番には削除保護（termination protection / `prevent_destroy`）**を付ける。
- **プロバイダ/CDK のバージョン固定漏れ**：未固定だと「昨日通ったのに今日壊れる」。バージョンを lock する。

## 関連
[monitoring.md](./monitoring.md) / [iam.md](./iam.md) / [vpc.md](./vpc.md)
