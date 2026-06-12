# コンテナ（ECS / Fargate / EKS / ECR）（AWS）

## ひとことで言うと
Dockerコンテナ（アプリ＋依存をまとめた箱）をAWS上で動かすための仕組み一式。**ECS**＝AWS独自のコンテナ実行基盤、**EKS**＝マネージドKubernetes、**ECR**＝コンテナイメージの置き場（レジストリ）。「どこで・どう動かすか」を選ぶのが要点。

## 役割・なぜ必要か
- アプリを**コンテナイメージ**にしておけば、ローカルでも本番でも同じものが動く（環境差異をなくす）。その本番側の実行・スケール・配置を担うのがECS/EKS。
- **ECR（Elastic Container Registry）**＝ビルドしたイメージを保管する場所。ECS/EKSはここからイメージを取得して起動する。Docker Hubの自社版。
- **ECS（Elastic Container Service）**＝AWS独自のオーケストレーター。起動方法（**起動タイプ**）を2つから選ぶ：
  - **Fargate**：サーバー（EC2）を**一切管理しない**。「このコンテナをこのCPU/メモリで動かして」と頼むだけ。台数・OS・パッチを意識しない。少人数・運用を軽くしたい時の第一候補。
  - **EC2起動タイプ**：自分で用意したEC2群（クラスタ）の上でコンテナを動かす。**料金最適化やGPU・特殊要件**で細かく制御したい時に。サーバー管理は自分持ち。
- **EKS（Elastic Kubernetes Service）**＝マネージドKubernetes。Kubernetes資産・エコシステム（Helm等）をそのまま使いたい、マルチクラウド前提、という組織向け。**運用の難易度とコストは上がる**。
- **タスク定義（Task Definition）**＝ECSの設計図。使うイメージ・CPU/メモリ・ポート・環境変数・ロール・ログ設定などを記述したJSON。これを元に**タスク（実体）**が起動し、複数タスクを**サービス**が常時維持・ロードバランス・オートスケールする。

## 基本の使い方（CLI / コンソール）
ECR にイメージを push（共通の出発点）：
```bash
ACCOUNT=123456789012; REGION=ap-northeast-1; REPO=myapp
# 1) リポジトリ作成
aws ecr create-repository --repository-name $REPO

# 2) Docker を ECR にログイン
aws ecr get-login-password --region $REGION \
  | docker login --username AWS --password-stdin \
    $ACCOUNT.dkr.ecr.$REGION.amazonaws.com

# 3) ビルド → タグ付け → push
docker build -t $REPO .
docker tag $REPO:latest $ACCOUNT.dkr.ecr.$REGION.amazonaws.com/$REPO:latest
docker push $ACCOUNT.dkr.ecr.$REGION.amazonaws.com/$REPO:latest
```

ECS（Fargate）タスク定義の最小例：
```json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/myapp:latest",
      "portMappings": [{ "containerPort": 3000, "protocol": "tcp" }],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "ap-northeast-1",
          "awslogs-stream-prefix": "web"
        }
      }
    }
  ]
}
```
```bash
# 登録 → サービス更新（新タスク定義で入れ替え）
aws ecs register-task-definition --cli-input-json file://taskdef.json
aws ecs update-service --cluster prod --service myapp --force-new-deployment
```

## 実務での勘所
- **まずは Fargate**。サーバー管理が消える分、運用がはるかに軽い。EC2起動タイプはコスト最適化や特殊要件が出てから検討する。
- **タスク実行ロール**（イメージpullやログ書き込み用＝executionRole）と**タスクロール**（アプリがS3等を触る用＝taskRole）は**別物**。混同しない。→ [iam.md](./iam.md)
- 前段に **ALB（ロードバランサー）** を置き、サービスのタスクへ振り分ける。ヘルスチェックパスを正しく設定。
- ログは `awslogs` ドライバで CloudWatch Logs に集約するのが基本。→ [monitoring.md](./monitoring.md)
- デプロイはタスク定義のリビジョンを更新して**ローリング**（または CodeDeploy で Blue/Green）。イメージタグは `latest` 固定より**コミットハッシュ等で固定**するとロールバックしやすい。
- 構成は IaC（CDK / Terraform）でまとめて管理。→ [iac.md](./iac.md) / 土台のサーバーは [ec2.md](./ec2.md)

## ハマりどころ / アンチパターン
- **Fargate vs EC2 の選択ミス**：とりあえずEC2起動タイプにして**EC2の運用（パッチ・スケール・クラスタ管理）を抱え込む**のは典型的な遠回り。明確な理由がなければFargate。逆に大量・常時稼働でコストが効くならEC2/Savings Planを検討。
- **ネットワークモード（awsvpc）とサブネット/SGの設定漏れ**：Fargateは `awsvpc` 必須で**タスクごとにENI**が付く。プライベートサブネットに置いたのに**NAT Gateway も VPCエンドポイントも無く、ECRからイメージをpullできず起動失敗**、はド定番事故。→ [vpc.md](./vpc.md)
- **CPU/メモリの刻みが固定**：Fargateは「0.25vCPU/0.5GB」等の決まった組み合わせのみ。任意値は不可。
- **EKSの運用コストを甘く見る**：コントロールプレーン料金に加え、Kubernetesの専門知識・アップグレード・アドオン運用が継続的にかかる。**小規模でEKSは過剰**になりがち。要件が無ければECS/Fargateで十分。
- **イメージが肥大**：マルチステージビルドや軽量ベース（alpine/distroless）にしないと、push/pull・コールドな起動が遅くなる。
- **`latest` タグ運用**でどのコミットが本番か分からなくなる／ロールバック不能。タグは固定する。

## 関連
[ec2.md](./ec2.md) / [iam.md](./iam.md) / [vpc.md](./vpc.md) / [monitoring.md](./monitoring.md) / [iac.md](./iac.md)
