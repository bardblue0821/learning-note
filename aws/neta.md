# AWS ハンズオンネタ帳

> Docker を使わず、VSCode 内で完結できるものをピックアップ

---

## 1. サーバーレス Web API（Lambda + API Gateway + DynamoDB）

### 概要
- REST API を Lambda で作成し、DynamoDB でデータを永続化する
- 無料枠内で収まりやすく、最初の題材として最適

### 学べること
- Lambda 関数の作成・デプロイ
- API Gateway による HTTP エンドポイント公開
- DynamoDB のテーブル設計と CRUD 操作
- IAM ロール・ポリシーの基本

### 環境構築
1. AWS CLI インストール・設定（`aws configure`）
2. VSCode 拡張「AWS Toolkit」インストール
3. SAM CLI インストール（ローカルテスト用、Docker 不要モードあり）
4. Node.js or Python（Lambda ランタイム）

### ハンズオン手順（例：TODO API）
1. `sam init` でプロジェクト作成（`--runtime python3.12` など）
2. `template.yaml` で Lambda + API Gateway + DynamoDB を定義
3. Lambda 関数に CRUD ハンドラーを実装
4. `sam deploy --guided` でデプロイ
5. API Gateway のエンドポイントに curl でリクエスト確認
6. DynamoDB コンソールまたは AWS Toolkit でデータ確認

### 注意
- SAM CLI のローカル実行（`sam local invoke`）は Docker が必要だが、直接デプロイ（`sam deploy`）は Docker 不要
- ローカルテストは単体テスト（pytest / jest）で代替可能

### コスト目安
- Lambda: 月100万リクエスト無料
- DynamoDB: 25GB ストレージ・25 WCU/RCU 無料
- API Gateway: 月100万 API コール無料（12ヶ月）

---

## 2. 静的サイトホスティング（S3 + CloudFront）

### 概要
- React / Vite でビルドした静的サイトを S3 にホスティング
- CloudFront で HTTPS 配信

### 学べること
- S3 バケットの作成・静的ウェブサイトホスティング設定
- CloudFront ディストリビューションの設定
- AWS CLI によるデプロイ自動化
- バケットポリシー・OAC の設定

### 環境構築
1. AWS CLI インストール・設定済み
2. Node.js（React / Vite 用）
3. VSCode のみで完結

### ハンズオン手順
1. `npm create vite@latest my-site -- --template react-ts`
2. `npm run build` で静的ファイル生成
3. S3 バケット作成: `aws s3 mb s3://my-site-bucket-xxxx`
4. 静的ウェブサイトホスティング有効化
5. ビルド成果物アップロード: `aws s3 sync dist/ s3://my-site-bucket-xxxx`
6. CloudFront ディストリビューション作成（CLI or コンソール）
7. HTTPS でアクセス確認

### コスト目安
- S3: 5GB 無料（12ヶ月）
- CloudFront: 1TB/月 転送無料（12ヶ月）

---

## 3. Infrastructure as Code（AWS CDK with TypeScript）

### 概要
- TypeScript で AWS インフラを定義・デプロイ
- 上記 1, 2 の構成を CDK でコード化すると実践的

### 学べること
- CDK のプロジェクト構造と Construct の概念
- TypeScript によるインフラ定義
- `cdk deploy` / `cdk destroy` によるライフサイクル管理
- スタック・リソース間の依存関係

### 環境構築
1. AWS CLI インストール・設定済み
2. Node.js + npm
3. `npm install -g aws-cdk`
4. `cdk bootstrap`（初回のみ、AWS アカウントに CDK 用リソースを作成）

### ハンズオン手順
1. `cdk init app --language typescript`
2. `lib/` 配下にスタック定義を記述（例: S3 バケット + Lambda）
3. `cdk synth` で CloudFormation テンプレート生成・確認
4. `cdk deploy` でデプロイ
5. リソースを変更して `cdk diff` → `cdk deploy` で差分更新
6. `cdk destroy` で後片付け

### コスト目安
- CDK 自体は無料（作成されるリソースの料金のみ）

---

## 4. イベント駆動パイプライン（S3 + Lambda + SNS）

### 概要
- S3 にファイルをアップロードすると Lambda がトリガーされ、処理結果を SNS でメール通知

### 学べること
- S3 イベント通知の設定
- Lambda のイベントソースマッピング
- SNS トピック・サブスクリプション
- イベント駆動アーキテクチャの基本

### 環境構築
1. AWS CLI インストール・設定済み
2. Python or Node.js（Lambda 用）

### ハンズオン手順
1. SNS トピック作成 → メールサブスクリプション登録
2. Lambda 関数作成（S3 イベントを受けて SNS に通知）
3. S3 バケット作成 → イベント通知で Lambda を設定
4. ファイルアップロード: `aws s3 cp test.txt s3://my-bucket/`
5. メール通知を確認

### コスト目安
- SNS: 月100万パブリッシュ無料
- 他は無料枠内

---

## 共通：環境構築チェックリスト

- [ ] AWS アカウント（個人）作成済み
- [ ] IAM ユーザー作成（ルートアカウントは使わない）
- [ ] MFA 有効化
- [ ] AWS CLI v2 インストール
- [ ] `aws configure` で credentials 設定
- [ ] VSCode 拡張「AWS Toolkit」インストール
- [ ] Node.js（LTS）インストール
- [ ] SAM CLI インストール（題材1で使用）
- [ ] AWS CDK インストール（題材3で使用）

---

## 後片付け（コスト防止）

各ハンズオン終了後、必ずリソースを削除する:

```bash
# SAM でデプロイしたもの
sam delete --stack-name <stack-name>

# CDK でデプロイしたもの
cdk destroy

# S3 バケット削除（中身を先に空にする）
aws s3 rm s3://my-bucket --recursive
aws s3 rb s3://my-bucket

# CloudFront ディストリビューション削除（先に無効化が必要）
```
