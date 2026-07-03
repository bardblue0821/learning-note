# ハンズオン②：React サイトを AWS にホスティングしよう

> **作るもの**: React（Vite）で作ったサイトを世界中からアクセスできるようにする
> **使うサービス**: S3 + CloudFront
> **リージョン**: ap-northeast-1（東京）
> **所要時間**: 約 45〜60 分

---

## このハンズオンで学べること

- S3 バケットに静的ファイル（HTML / CSS / JS）を置いて公開する方法
- CloudFront（CDN）で高速・HTTPS 配信する方法
- AWS CLI でデプロイを自動化する方法

---

## 前提知識

- AWS CLI が設定済みであること
- React + Vite の基本がわかること
- Node.js がインストールされていること

---

## 用語解説

| 用語 | ざっくり説明 |
|------|-------------|
| **S3** | ファイルを保存できるストレージサービス。画像や HTML を置ける「ファイル置き場」 |
| **バケット** | S3 の中のフォルダのようなもの。世界で一意な名前が必要 |
| **CloudFront** | 世界中のサーバーにファイルをコピーして、近くのサーバーから配信してくれるサービス（CDN） |
| **CDN** | Content Delivery Network。近くのサーバーから配信するので速い |
| **HTTPS** | 通信を暗号化する仕組み。CloudFront を使うと自動で HTTPS になる |
| **OAC** | Origin Access Control。S3 に直接アクセスさせず、CloudFront 経由のみ許可する設定 |

---

## Step 1：React プロジェクトを作る

```bash
cd aws/02-staticSiteHosting

# Vite で React プロジェクトを作成
npm create vite@latest my-site -- --template react-ts

cd my-site
npm install
```

### 動作確認（ローカル）

```bash
npm run dev
```

ブラウザで `http://localhost:5173` を開いて、React のページが表示されればOK。
確認できたら `Ctrl + C` で停止する。

---

## Step 2：ビルドする

```bash
npm run build
```

`dist/` フォルダが作られる。この中に HTML / CSS / JS が入っている。
**この `dist/` フォルダの中身を S3 にアップロードする。**

---

## Step 3：S3 バケットを作る

バケット名は**世界で一意**でなければならない。自分の名前や日付を入れて重複を避けよう。

```bash
# バケット名を変数に入れておく（自分の名前に変えてね）
BUCKET_NAME="my-react-site-$(date +%Y%m%d)-yourname"

# バケットを作成
aws s3 mb s3://$BUCKET_NAME --region ap-northeast-1

# 確認
aws s3 ls | grep $BUCKET_NAME
```

> 💡 `make_bucket: s3://my-react-site-...` と表示されれば成功！

---

## Step 4：ビルド成果物を S3 にアップロードする

```bash
# dist/ の中身をバケットにアップロード
aws s3 sync dist/ s3://$BUCKET_NAME
```

> `upload: dist/index.html to s3://...` のような表示が出れば OK

---

## Step 5：CloudFront ディストリビューションを作る

CloudFront を使うと、HTTPS で安全にサイトを配信できる。

### 5-1. OAC（Origin Access Control）を作成

```bash
aws cloudfront create-origin-access-control \
  --origin-access-control-config \
    Name=my-site-oac,\
Description="OAC for my React site",\
SigningProtocol=sigv4,\
SigningBehavior=always,\
OriginAccessControlOriginType=s3
```

レスポンスの中の `"Id": "EXXXXXXXX"` をメモしておく。

### 5-2. ディストリビューション設定ファイルを作る

`my-site/` 内に `cf-config.json` というファイルを作成する：

```json
{
  "CallerReference": "my-react-site-unique-ref",
  "Comment": "React site distribution",
  "DefaultCacheBehavior": {
    "TargetOriginId": "s3-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"]
    },
    "ForwardedValues": {
      "QueryString": false,
      "Cookies": { "Forward": "none" }
    },
    "MinTTL": 0,
    "DefaultTTL": 86400,
    "MaxTTL": 31536000
  },
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "s3-origin",
        "DomainName": "<BUCKET_NAME>.s3.ap-northeast-1.amazonaws.com",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        },
        "OriginAccessControlId": "<OAC_ID>"
      }
    ]
  },
  "DefaultRootObject": "index.html",
  "Enabled": true
}
```

**2箇所を書き換える**：
- `<BUCKET_NAME>` → 自分のバケット名
- `<OAC_ID>` → Step 5-1 でメモした OAC の Id

### 5-3. ディストリビューションを作成

```bash
aws cloudfront create-distribution \
  --distribution-config file://cf-config.json
```

レスポンスから以下をメモする：
- `"Id": "EXXXXXXXXXXX"` → ディストリビューション ID
- `"DomainName": "dxxxxxxxxxx.cloudfront.net"` → **これがサイトの URL**

> ⏳ CloudFront の反映には 5〜10 分かかる。

---

## Step 6：S3 バケットポリシーを設定する

CloudFront からのアクセスのみ S3 を許可するポリシーを設定する。

`my-site/` 内に `bucket-policy.json` を作成：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<BUCKET_NAME>/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
        }
      }
    }
  ]
}
```

**3箇所を書き換える**：
- `<BUCKET_NAME>` → 自分のバケット名
- `<ACCOUNT_ID>` → AWS アカウント ID（`aws sts get-caller-identity --query Account --output text` で確認）
- `<DISTRIBUTION_ID>` → Step 5-3 でメモしたディストリビューション ID

```bash
# バケットポリシーを適用
aws s3api put-bucket-policy \
  --bucket $BUCKET_NAME \
  --policy file://bucket-policy.json
```

---

## Step 7：動作確認

ブラウザで `https://dxxxxxxxxxx.cloudfront.net` にアクセスする。

React の初期画面が表示されれば成功！🎉

> もし「Access Denied」が出たら、5〜10 分待ってからリトライしてみよう。
> CloudFront の反映に時間がかかることがある。

---

## Step 8：サイトを更新するとき

コードを変更して再デプロイするときは、この 2 コマンドだけ：

```bash
# ビルドし直す
npm run build

# S3 にアップロード（変更分だけ同期される）
aws s3 sync dist/ s3://$BUCKET_NAME --delete

# CloudFront のキャッシュをクリア（すぐ反映させたい場合）
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths "/*"
```

---

## Step 9：後片付け（重要！）

```bash
# 1. CloudFront ディストリビューションを無効化
aws cloudfront get-distribution-config --id <DISTRIBUTION_ID> > dist-config.json
# dist-config.json の "Enabled" を false に変更し、ETag を確認
# → 無効化してから削除（手順が複雑なので AWS コンソールからの削除も OK）

# 2. S3 バケットの中身を削除してからバケットを削除
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3 rb s3://$BUCKET_NAME

# 3. OAC を削除
aws cloudfront delete-origin-access-control --id <OAC_ID> --if-match <ETAG>
```

> 💡 CloudFront の削除は手順が多いので、AWS コンソール（ブラウザ）から削除しても OK。
> CloudFront コンソール → ディストリビューション選択 → 無効化 → 削除

---

## まとめ

このハンズオンでやったこと：

1. React（Vite）でサイトをビルドした
2. S3 バケットにビルド成果物をアップロードした
3. CloudFront でHTTPS 配信を設定した
4. バケットポリシーでセキュリティを設定した

```
[ユーザー] --HTTPS--> [CloudFront (CDN)] --取得--> [S3 バケット]
                    世界中に配信              HTML/CSS/JS を保管
```

---

## 発展課題

- [ ] 独自ドメインを Route 53 で取得して、CloudFront に紐づけてみよう
- [ ] `aws s3 sync` を npm script に登録して `npm run deploy` で一発デプロイできるようにしよう
- [ ] React Router を使った SPA で、リロード時に 403 にならないよう CloudFront のエラーページ設定を試そう