# Route 53 自動NSレコード作成

## 概要

アカウントAで元のドメインのホストゾーンを管理し、アカウントBにサブドメインのホストゾーンを作成した際に、自動的にNSレコードを作成するシステムです。

## デプロイ手順

### 1. アカウントAにリソースをデプロイ

```bash
aws cloudformation create-stack \
  --stack-name route53-ns-record-account-a \
  --template-body file://account-a-template.yaml \
  --parameters ParameterKey=HostedZoneId,ParameterValue=Z1234456566KX8YR1 \
  --capabilities CAPABILITY_IAM
```

### 2. アカウントBにセットアップリソースをデプロイ（Lambda + EventBridge）

```bash
aws cloudformation create-stack \
  --stack-name route53-ns-record-account-b-setup \
  --template-body file://account-b-setup-template.yaml \
  --capabilities CAPABILITY_IAM
```

### 3. アカウントBにホストゾーンを作成（イベントがトリガーされる）

```bash
aws cloudformation create-stack \
  --stack-name route53-ns-record-account-b-hostedzone \
  --template-body file://account-b-hostedzone-template.yaml
```

## ファイル構成

- `account-a-template.yaml`: アカウントA用CloudFormationテンプレート
- `account-b-setup-template.yaml`: アカウントB用セットアップテンプレート（Lambda + EventBridge）
- `account-b-hostedzone-template.yaml`: アカウントB用ホストゾーン作成テンプレート

## 動作フロー

1. アカウントBでRoute 53ホストゾーンが作成される
2. EventBridgeがイベントを検知してLambda関数を実行
3. Lambda関数がアカウントAのDynamoDBテーブルにレコードを追加（履歴として保存）
4. DynamoDBストリームがアカウントAのLambda関数をトリガー
5. アカウントAのLambda関数が元のホストゾーンにNSレコードを作成
6. ホストゾーン削除時も同様に履歴が記録され、NSレコードが削除される

## 特徴

- **履歴管理**: DynamoDBテーブルにタイムスタンプ付きで全操作を記録
- **クロスアカウントアクセス**: 組織IDベースのリソースポリシーで安全にアクセス
- **自動ドメイン名**: AWSアカウントIDを使用して `{AccountID}.example.com` を自動生成
- **操作追跡**: CREATE/DELETE操作を区別して履歴管理
