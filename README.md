# EventBridge\_ECS\_S3\_sample

<br>

以下は「EventBridge Scheduler → ECS(Fargate) タスク → S3コピー」という構成を、実務でそのまま使える粒度でまとめた手順です。

**コンソール手順**と**IaC（Terraform）サンプル**、**アプリ実装例（Python）**、**IAM最小権限**、**ネットワークと運用ポイント**まで一通り網羅します。  
（前提：AWSアカウントがあり、VPC/Subnet/セキュリティグループを用意できること。リージョンは例として ap-northeast-1 ）

* * *

# 1\. 全体像（最小構成）

- **EventBridge Scheduler**（時間指定 or 周期）  
↓（`ecs:RunTask` を実行）
- **ECS Task（Fargate, 1コンテナ）**
    - コンテナ内のスクリプトが
        - `SourceBucket` からテキストファイルを `GetObject`
        - `DestinationBucket` へ `PutObject`

    - ログは **CloudWatch Logs**

- **IAM**
    - タスク実行ロール（`executionRole`）：ECR イメージ取得＋ログ出力
    - タスクロール（`taskRole`）：S3 読み書き
    - Scheduler 実行ロール（`schedulerRole`）：`ecs:RunTask` & `iam:PassRole`

* * *

# 2\. コンテナ実装例（Python + boto3）

### 2.1 アプリコード（`app/main.py`）

```
import os
import boto3

def main():
    src_bucket = os.environ["SOURCE_BUCKET"]
    dst_bucket = os.environ["DEST_BUCKET"]
    src_key    = os.environ["SOURCE_KEY"]   # 例: "input/sample.txt"
    dst_key    = os.environ.get("DEST_KEY", src_key.replace("input/","output/"))

    s3 = boto3.client("s3")

    # 取得
    obj = s3.get_object(Bucket=src_bucket, Key=src_key)
    body = obj["Body"].read()

    # （必要に応じて加工）ここではそのまま
    s3.put_object(Bucket=dst_bucket, Key=dst_key, Body=body, ContentType="text/plain")
    print(f"Copied s3://{src_bucket}/{src_key} -> s3://{dst_bucket}/{dst_key}")

if __name__ == "__main__":
    main()
```

### 2.2 `requirements.txt`

```
boto3==1.34.0
```

### 2.3 Dockerfile（Fargate用, arm/amd どちらでもOK）

```
FROM public.ecr.aws/lambda/python:3.11  AS build
# Lambdaベースでも普通のPythonベースでも可。軽量でpipが使いやすいイメージなら何でもOK。

# 依存インストール
COPY requirements.txt .
RUN pip install -r requirements.txt -t /opt/python

# 実行レイヤ
FROM public.ecr.aws/docker/library/python:3.11-slim
WORKDIR /app
COPY --from=build /opt/python /opt/python
ENV PYTHONPATH=/opt/python
COPY app/ app/
ENTRYPOINT ["python", "app/main.py"]
```

> ビルド → ECR プッシュ（例）

```
aws ecr create-repository --repository-name s3-copy-task
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-northeast-1
IMAGE=$AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/s3-copy-task:latest

aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com

docker build -t s3-copy-task .
docker tag s3-copy-task:latest $IMAGE
docker push $IMAGE
```

* * *

# 3\. IAM（最小権限ポリシー）

### 3.1 タスク実行ロール（execution role）

- 管理ポリシー `AmazonECSTaskExecutionRolePolicy` を付与（ECR 取得 & CW Logs 出力）
- 信頼ポリシー：`ecs-tasks.amazonaws.com`

```
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Principal":{"Service":"ecs-tasks.amazonaws.com"},
      "Action":"sts:AssumeRole"
    }
  ]
}
```

### 3.2 タスクロール（task role）

- **S3最小権限**（バケット名は要置換）

```
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "ReadFromSourceObjects",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::<SourceBucket>/*"]
    },
    { "Sid": "ListSourceBucketIfNeeded",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": ["arn:aws:s3:::<SourceBucket>"]
    },
    { "Sid": "WriteToDestinationObjects",
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": ["arn:aws:s3:::<DestinationBucket>/*"]
    }
  ]
}
```

- 信頼ポリシー：`ecs-tasks.amazonaws.com`（execution role と同様）

### 3.3 Scheduler 実行ロール（EventBridge Scheduler 用）

- `ecs:RunTask`（対象のタスク定義 & クラスター）
- `iam:PassRole`（**タスク実行ロール**と**タスクロール** を ECS に渡せるように）

```
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow",
      "Action": ["ecs:RunTask"],
      "Resource": ["arn:aws:ecs:ap-northeast-1:<ACCOUNT_ID>:task-definition/s3-copy-task:*"] 
    },
    { "Effect": "Allow",
      "Action": ["iam:PassRole"],
      "Resource": [
        "arn:aws:iam::<ACCOUNT_ID>:role/<EcsTaskExecutionRoleName>",
        "arn:aws:iam::<ACCOUNT_ID>:role/<EcsTaskRoleName>"
      ]
    }
  ]
}
```

> ポイント
> 
> - `iam:PassRole` がないと Scheduler から `RunTask` できません。
> - タスク定義名/リビジョン、クラスター名の ARN をきちんと指定しましょう（ワイルドカードでも可ですが最小権限推奨）。

* * *

# 4\. ECS（Fargate）設定

### 4.1 CloudWatch Logs ロググループ

- 例：`/ecs/s3-copy-task` を作成（保持期間 30 日など）

### 4.2 タスク定義（1 vCPU / 0.5–1GB RAM で十分）

- **executionRoleArn**：3.1 のロール
- **taskRoleArn**：3.2 のロール
- 環境変数は **Scheduler の overrides** で渡す方が柔軟（後述）

タスク定義の一部（イメージとロギング）

```
{
  "family": "s3-copy-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/<EcsTaskExecutionRoleName>",
  "taskRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/<EcsTaskRoleName>",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "<ACCOUNT_ID>.dkr.ecr.ap-northeast-1.amazonaws.com/s3-copy-task:latest",
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/s3-copy-task",
          "awslogs-region": "ap-northeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### 4.3 クラスター/サービス

- **サービスは不要**（**スケジューラから `RunTask`** で都度起動するため）
- **クラスター**だけ作成（例：`batch-tools`）

### 4.4 ネットワーク

**選択肢 A（簡易）**：プライベートサブネット + NAT GW

- ECR/ECR DKR/CW Logs へのアウトバウンドに NAT を使用。S3 はインターネット向けでも可。

**選択肢 B（NAT なし, コスト最小）**：プライベートサブネット + **VPC エンドポイント**

- **ゲートウェイ型**：S3
- **インターフェイス型**：ECR（`api.ecr`）、ECR DKR（`dkr.ecr`）、CloudWatch Logs、STS など
- これでインターネットなしで完結（推奨だが初期設定は多め）

**セキュリティグループ**：最小でOK（アウトバウンド開放、インバウンド不要）

* * *

# 5\. EventBridge Scheduler 設定

### 5.1 スケジュール作成

- **スケジュール式**：例）毎日 02:00（日本時間）
    - `cron(0 17 * * ? *)` に **タイムゾーン Asia/Tokyo** を指定  
（EventBridge Scheduler は直接タイムゾーン指定が可能）

- **ターゲット**：**ECS タスク**（RunTask）
    - クラスター：`batch-tools`
    - タスク定義：`s3-copy-task:latest`
    - 起動タイプ：`FARGATE`
    - Platform version：`LATEST`
    - タスクサイズ：`0.5 vCPU / 1GB`（タスク定義に合わせる）
    - サブネット：プライベート（またはパブリックに Public IP 付与）
    - セキュリティグループ：前述の最小 SG
    - **ロール**：前述の **Scheduler 実行ロール** を指定

### 5.2 コンテナ overrides（環境変数で可変化）

- 例）`SOURCE_BUCKET`, `DEST_BUCKET`, `SOURCE_KEY`, `DEST_KEY`

Scheduler の **Payload/Overrides** 例（コンソールなら「コンテナのオーバーライド」）：

```
{
  "ContainerOverrides": [
    {
      "Name": "app",
      "Environment": [
        {"Name": "SOURCE_BUCKET", "Value": "my-source-bucket"},
        {"Name": "DEST_BUCKET",   "Value": "my-destination-bucket"},
        {"Name": "SOURCE_KEY",    "Value": "input/sample.txt"},
        {"Name": "DEST_KEY",      "Value": "output/sample.txt"}
      ]
    }
  ]
}
```

### 5.3 リトライ・DLQ

- Scheduler 側で **最大再試行回数** と **フレックスウィンドウ** を設定可能。
- 失敗検知を厳密にしたい場合、**EventBridge ルール + SQS DLQ**（`ECS Task State Change` イベント）や **CloudWatch Alarms** を組み合わせると良いです。

* * *

# 6\. Terraform（最小一式サンプル）

> 既存の VPC/Subnet/SG がある前提。リソース名/ARN は適宜置換してください。  
> コンテナイメージ（`var.image`）は事前に ECR へ push 済みを想定。

```
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = { source = "hashicorp/aws", version = ">= 5.0" }
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

variable "cluster_name" { default = "batch-tools" }
variable "image"        { description = "ECR image URI" }
variable "subnet_ids"   { type = list(string) }
variable "sg_id"        { type = string }
variable "source_bucket" {}
variable "dest_bucket"   {}

# CW Logs
resource "aws_cloudwatch_log_group" "ecs" {
  name              = "/ecs/s3-copy-task"
  retention_in_days = 30
}

# IAM: execution role
resource "aws_iam_role" "ecs_execution" {
  name = "ecsTaskExecutionRole-s3copy"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Principal = { Service = "ecs-tasks.amazonaws.com" },
      Action = "sts:AssumeRole"
    }]
  })
}
resource "aws_iam_role_policy_attachment" "ecs_exec_attach" {
  role       = aws_iam_role.ecs_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# IAM: task role with least privilege to S3
resource "aws_iam_role" "ecs_task" {
  name = "ecsTaskRole-s3copy"
  assume_role_policy = aws_iam_role.ecs_execution.assume_role_policy
}
resource "aws_iam_policy" "s3_rw" {
  name   = "s3copy-s3-policy"
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Sid: "ReadFromSource",
        Effect: "Allow",
        Action: ["s3:GetObject"],
        Resource: ["arn:aws:s3:::${var.source_bucket}/*"]
      },
      {
        Sid: "ListSource",
        Effect: "Allow",
        Action: ["s3:ListBucket"],
        Resource: ["arn:aws:s3:::${var.source_bucket}"]
      },
      {
        Sid: "WriteToDest",
        Effect: "Allow",
        Action: ["s3:PutObject"],
        Resource: ["arn:aws:s3:::${var.dest_bucket}/*"]
      }
    ]
  })
}
resource "aws_iam_role_policy_attachment" "task_s3" {
  role       = aws_iam_role.ecs_task.name
  policy_arn = aws_iam_policy.s3_rw.arn
}

# ECS Cluster
resource "aws_ecs_cluster" "this" {
  name = var.cluster_name
}

# Task Definition
resource "aws_ecs_task_definition" "s3copy" {
  family                   = "s3-copy-task"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name      = "app",
      image     = var.image,
      essential = true,
      logConfiguration = {
        logDriver = "awslogs",
        options = {
          awslogs-group         = aws_cloudwatch_log_group.ecs.name,
          awslogs-region        = "ap-northeast-1",
          awslogs-stream-prefix = "ecs"
        }
      }
    }
  ])
}

# EventBridge Scheduler role
resource "aws_iam_role" "scheduler" {
  name = "schedulerRole-ecsRunTask-s3copy"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Principal = { Service = "scheduler.amazonaws.com" },
      Action = "sts:AssumeRole"
    }]
  })
}
resource "aws_iam_policy" "scheduler_policy" {
  name   = "scheduler-ecs-runtask-policy"
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect: "Allow",
        Action: ["ecs:RunTask"],
        Resource: [aws_ecs_task_definition.s3copy.arn]
      },
      {
        Effect: "Allow",
        Action: ["iam:PassRole"],
        Resource: [
          aws_iam_role.ecs_execution.arn,
          aws_iam_role.ecs_task.arn
        ]
      }
    ]
  })
}
resource "aws_iam_role_policy_attachment" "scheduler_attach" {
  role       = aws_iam_role.scheduler.name
  policy_arn = aws_iam_policy.scheduler_policy.arn
}

# EventBridge Scheduler (毎日 02:00 JST)
resource "aws_scheduler_schedule" "daily_copy" {
  name       = "daily-s3-copy"
  group_name = "default"

  flexible_time_window {
    mode = "OFF"
  }

  schedule_expression          = "cron(0 17 * * ? *)" # UTC 17:00 = JST 02:00
  schedule_expression_timezone = "Asia/Tokyo"

  target {
    arn      = aws_ecs_cluster.this.arn
    role_arn = aws_iam_role.scheduler.arn

    ecs_parameters {
      task_definition_arn = aws_ecs_task_definition.s3copy.arn
      launch_type         = "FARGATE"
      platform_version    = "LATEST"
      network_configuration {
        awsvpc_configuration {
          subnets         = var.subnet_ids
          security_groups = [var.sg_id]
          assign_public_ip = "DISABLED" # NAT or VPCエンドポイントが前提
        }
      }
      task_count = 1
      # コンテナの環境変数オーバーライド
      tags = {} # 任意
    }

    input = jsonencode({
      ContainerOverrides: [{
        Name: "app",
        Environment: [
          { Name: "SOURCE_BUCKET", Value: var.source_bucket },
          { Name: "DEST_BUCKET",   Value: var.dest_bucket },
          { Name: "SOURCE_KEY",    Value: "input/sample.txt" },
          { Name: "DEST_KEY",      Value: "output/sample.txt" }
        ]
      }]
    })
  }
}
```

* * *

# 7\. 動作確認フロー

1. **S3 事前準備**：`SourceBucket` に `input/sample.txt` を配置。`DestinationBucket` を空でOK。
2. **手動起動テスト**：ECS コンソール → クラスター → タスク → **タスクの実行**（同じタスク定義・ネットワーク・環境変数を上書き）
3. **CloudWatch Logs** を確認：コピーの print ログが出ているか
4. **S3**：`DestinationBucket/output/sample.txt` が作成されているか
5. **スケジュール実行**：Scheduler 時刻を短い間隔で一時的に設定して通電確認

* * *

# 8\. セキュリティ & 運用のポイント

- **最小権限**：S3 アクションは `GetObject`/`PutObject` に限定、バケットリソースも絞る。
- **KMS**：暗号化バケットを使う場合は、**タスクロール**に `kms:Decrypt`（ソース側 CMK）、`kms:Encrypt`（デスティネーション側 CMK）を付与。S3 側のバケットポリシーにもロール許可を反映。
- **ネットワーク**：NAT コストを削りたいなら **VPC エンドポイント（S3 Gateway, ECR/Logs/STS Interface）** を導入。
- **可観測性**：
    - **CloudWatch Logs**（アプリログ）
    - **EventBridge Scheduler のメトリクス/履歴**
    - **ECS Task State Change イベント**を EventBridge で拾って、失敗時 SQS DLQ/メール通知（SNS）

- **再処理設計**：同じ `DEST_KEY` への上書き抑止が必要なら `If-None-Match` 相当の制御や、出力を日付別キーにする（`output/yyyymmdd/sample.txt`）
- **パラメータ管理**：バケットやキーを **SSM パラメータストア**や **Secrets Manager** で管理し、タスク実行時に取得しても良い（タスクロールに `ssm:GetParameter` 付与が必要）。
- **コスト**：Fargate 分課金 + NAT/VPC エンドポイント + S3/Logs 少額。実行時間が短いほど安価。
- **並列実行**：複数ファイルを処理する場合は、入力をプレフィックス指定にし `ListObjectsV2` でループ、または **複数タスク** に分割（SQS キューからメッセージでキーを渡す構成も有効）。
- **リージョン/タイムゾーン**：EventBridge Scheduler は**タイムゾーン指定**が可能（本手順で JST 指定済み）。

* * *

# 9\. よくあるハマりどころ

- **AccessDenied(AccessDeniedException/AccessDenied)**：ほぼ `iam:PassRole` か S3/KMS ポリシー不足。
- **タスクが起動しない**：Fargate キャパ不足やサブネット/SG/ENI 作成不可（権限/クォータ）。
- **ECR から pull 失敗**：NAT なし構成で **ECR/ECR DKR/Logs/STS** の VPC エンドポイント未整備。
- **ENV が反映されない**：Scheduler の **コンテナ override の `Name`** がタスク定義の **container name** と一致していない。

* * *

必要なら、あなたのバケット名やネットワーク条件（NAT あり/なし、既存 VPC ID など）に合わせて **Terraform をあなたの環境に完全一致する形**に書き換えてお渡しします。

<br>

# 10\. 手動で作ったもの
- ECRレジストリ
- ECRイメージ
- セキュリティグループ
- VPCエンドポイント
- CloudWatch log
- S3バケット

