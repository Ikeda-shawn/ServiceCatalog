# AWS Service Catalog - Network Infrastructure

検証用のAWS Service Catalogポートフォリオとネットワークインフラストラクチャテンプレート集です。

## 概要

このリポジトリには、以下のAWSリソースをService Catalog製品として管理するためのCloudFormationテンプレートが含まれています。

### 製品一覧

1. **VPC** - VPCとインターネットゲートウェイ
2. **Subnets** - パブリックサブネット2つ(AZ-a/c)、プライベートサブネット6つ(AZ-a/c)
3. **Route Tables** - ルートテーブルとNATゲートウェイ
4. **VPC Endpoints** - AWSサービス用のVPCエンドポイント
5. **Security Groups** - Web、アプリケーション、データベース、管理用セキュリティグループ

## ディレクトリ構造

```
.
├── templates/
│   ├── vpc.yaml                  # VPCテンプレート
│   ├── subnets.yaml              # サブネットテンプレート
│   ├── route-tables.yaml         # ルートテーブルテンプレート
│   ├── vpc-endpoints.yaml        # VPCエンドポイントテンプレート
│   └── security-groups.yaml      # セキュリティグループテンプレート
└── service-catalog-portfolio.yaml # Service Catalogポートフォリオ定義
```

## デプロイ手順

### 1. S3バケットの作成とテンプレートのアップロード

```bash
# S3バケット名を環境変数に設定
export BUCKET_NAME="your-service-catalog-bucket"
export REGION="ap-northeast-1"

# S3バケットの作成
aws s3 mb s3://${BUCKET_NAME} --region ${REGION}

# テンプレートをS3にアップロード
aws s3 cp templates/ s3://${BUCKET_NAME}/service-catalog/templates/ --recursive
```

### 2. Service Catalogポートフォリオのデプロイ

```bash
aws cloudformation create-stack \
  --stack-name network-service-catalog \
  --template-body file://service-catalog-portfolio.yaml \
  --parameters \
    ParameterKey=S3BucketName,ParameterValue=${BUCKET_NAME} \
    ParameterKey=S3KeyPrefix,ParameterValue=service-catalog/templates/ \
  --region ${REGION}
```

### 3. ポートフォリオへのアクセス権限付与

Service Catalogコンソールから、ポートフォリオにIAMユーザー/ロール/グループを追加します。

## リソースのプロビジョニング順序

Service Catalog製品は以下の順序でプロビジョニングする必要があります。

1. **VPC** (最初)
2. **Security Groups** (VPCの後)
3. **Subnets** (VPCの後)
4. **Route Tables** (Subnetsの後)
5. **VPC Endpoints** (Security Groups、Subnets、Route Tablesの後)

### 各製品のパラメータ例

#### 1. VPC
- `SystemName`: myapp
- `Env`: dev
- `VPCCidr`: 10.0.0.0/16

#### 2. Security Groups
- `SystemName`: myapp
- `Env`: dev
- `VPCStackName`: (VPCスタック名を指定)

#### 3. Subnets
- `SystemName`: myapp
- `Env`: dev
- `VPCStackName`: (VPCスタック名を指定)

#### 4. Route Tables
- `SystemName`: myapp
- `Env`: dev
- `VPCStackName`: (VPCスタック名を指定)
- `SubnetStackName`: (Subnetsスタック名を指定)
- `CreateNATGateway`: true

#### 5. VPC Endpoints
- `SystemName`: myapp
- `Env`: dev
- `VPCStackName`: (VPCスタック名を指定)
- `SubnetStackName`: (Subnetsスタック名を指定)
- `RouteTableStackName`: (Route Tablesスタック名を指定)
- `SecurityGroupStackName`: (Security Groupsスタック名を指定)

## 命名規則

全てのリソースは以下の命名規則に従います。

```
{SystemName}-{Env}-{リソース名}
```

例: `myapp-dev-vpc`, `myapp-dev-public-subnet-a`

## ネットワーク構成

### サブネット構成

- **パブリックサブネット**: 2つ (AZ-a, AZ-c)
  - インターネットゲートウェイ経由でインターネットアクセス可能

- **プライベートサブネット**: 6つ (AZ-aに3つ, AZ-cに3つ)
  - NATゲートウェイ経由でインターネットアクセス可能

### セキュリティグループ

1. **VPC Endpoint SG**: VPC内からのHTTPS(443)を許可
2. **Web SG**: インターネットからのHTTPS(443)を許可
3. **App SG**: WebレイヤーからのTCP/8080を許可
4. **DB SG**: Appレイヤーからのデータベースポートを許可、アウトバウンド通信なし
5. **Management SG**: Session Manager用、インバウンドなし

### VPC Endpoints

以下のAWSサービス用のVPCエンドポイントが作成されます。

- S3 (Gateway)
- DynamoDB (Gateway)
- EC2 (Interface)
- EC2 Messages (Interface)
- SSM (Interface)
- SSM Messages (Interface)
- CloudWatch Logs (Interface)

## 注意事項

- これは検証用のテンプレートです
- 本番環境で使用する場合は、適切なセキュリティレビューを実施してください
- NAT Gatewayは料金が発生するため、不要な場合は`CreateNATGateway`パラメータを`false`に設定してください
- DBセキュリティグループはアウトバウンド通信を許可していません

## クリーンアップ

```bash
# Service Catalogポートフォリオの削除
aws cloudformation delete-stack --stack-name network-service-catalog --region ${REGION}

# プロビジョニングした製品を先に削除してください
# 削除順序: VPC Endpoints → Route Tables → Subnets → Security Groups → VPC

# S3バケットの削除
aws s3 rb s3://${BUCKET_NAME} --force
```
