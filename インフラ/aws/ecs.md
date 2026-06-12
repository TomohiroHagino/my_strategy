# ECS（Elastic Container Service）（AWS）

## ひとことで言うと
AWS独自のコンテナオーケストレーター。**タスク定義（設計図）**を元に**タスク（コンテナの実体）**を起動し、**サービス**が常時その台数を維持・ロードバランス・オートスケールする。起動方法は **Fargate（サーバ管理不要）** か **EC2起動タイプ** を選ぶ。

## 役割・なぜ必要か
- 「何台・どのスペックで・どう配置し・落ちたら再起動」を自動で面倒みる仕組み。手で `docker run` を維持する運用から解放される。
- Kubernetes（EKS）より学習・運用が軽い。**AWSに閉じるなら ECS が第一候補**。
- **Fargate**：EC2を一切管理しない。「このコンテナをこのCPU/メモリで動かして」と頼むだけ。少人数の既定解。
- **EC2起動タイプ**：自前のEC2クラスタ上で動かす。コスト最適化・GPU等の特殊要件向け（サーバ管理は自分持ち）。

## 構成要素
- **クラスタ（Cluster）**：タスク/サービスを束ねる論理グループ。
- **タスク定義（Task Definition）**：イメージ・CPU/メモリ・ポート・環境変数・ロール・ログ設定を書いたJSON。リビジョン管理される。
- **タスク（Task）**：タスク定義から起動した実体（1〜複数コンテナ）。
- **サービス（Service）**：指定数のタスクを常時維持。ALB連携・ローリング更新・オートスケール。

## 基本の使い方（CLI / コンソール）
タスク定義（Fargate・最小例）：
```json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256", "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [{
    "name": "web",
    "image": "123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/myapp:GIT_SHA",
    "portMappings": [{ "containerPort": 3000 }],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": { "awslogs-group": "/ecs/myapp", "awslogs-region": "ap-northeast-1", "awslogs-stream-prefix": "web" }
    }
  }]
}
```
```bash
# 登録 → サービスを新タスク定義で入れ替え（ローリングデプロイ）
aws ecs register-task-definition --cli-input-json file://taskdef.json
aws ecs update-service --cluster prod --service myapp --force-new-deployment

# 状態確認
aws ecs describe-services --cluster prod --services myapp \
  --query 'services[0].{desired:desiredCount,running:runningCount,deployments:deployments[].rolloutState}'
```
オートスケール（CPUターゲット追跡の例）：
```bash
aws application-autoscaling register-scalable-target --service-namespace ecs \
  --resource-id service/prod/myapp --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 --max-capacity 10
# → target-tracking ポリシーで「平均CPU 60%」を維持するよう増減
```

## 実務の勘所
- **まず Fargate**。サーバ管理が消える分、運用がはるかに軽い。
- **executionRole（イメージpull・ログ書き込み用）と taskRole（アプリがS3等を触る用）は別物**。混同しない。→ [iam.md](./iam.md)
- 前段に **ALB** を置き、サービスのタスクへ振り分け。ヘルスチェックパスを正しく。
- **イメージタグはコミットハッシュ等で固定**（`latest` 固定はロールバック不能）。→ [ecr.md](./ecr.md)
- デプロイはタスク定義リビジョン更新で**ローリング**、または CodeDeploy で **Blue/Green**。→ [../../DevOps/deploy_strategies.md](../../DevOps/deploy_strategies.md)
- ログは `awslogs` で CloudWatch Logs に集約。→ [monitoring.md](./monitoring.md)
- 構成は IaC で管理。→ [iac.md](./iac.md) ／ Terraform実例 [../terraform/example_aws_ecs.md](../terraform/example_aws_ecs.md)

## ハマりどころ / アンチパターン
- **プライベートサブネットで ECR からpullできず起動失敗**：Fargateは `awsvpc` 必須で、NAT Gateway も VPCエンドポイント（ecr.api / ecr.dkr / s3）も無いとイメージを取得できない。**ド定番事故**。→ [vpc.md](./vpc.md)
- **CPU/メモリは固定の組み合わせ**（0.25vCPU/0.5GB 等）。任意値は不可。
- **ヘルスチェック失敗で無限再起動ループ**：パス・ポート・起動猶予（health check grace period）を見直す。
- **デプロイの最小/最大ヘルス％設定ミス**：`minimumHealthyPercent` が高すぎると入れ替えが進まない。
- **タスク定義リビジョンの更新忘れ**：イメージを上げたのにサービスが古いリビジョンを指したまま。

## 関連
[ecr.md](./ecr.md) / [containers.md](./containers.md) / [vpc.md](./vpc.md) / [iam.md](./iam.md) / [monitoring.md](./monitoring.md) / [iac.md](./iac.md)
コンテナ自体は [../docker/](../docker/) ／ IaC実例 [../terraform/example_aws_ecs.md](../terraform/example_aws_ecs.md)
