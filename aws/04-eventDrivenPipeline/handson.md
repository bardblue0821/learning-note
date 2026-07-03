# ハンズオン④：イベント駆動パイプラインを作ろう

> **作るもの**: S3 にファイルをアップロードすると、自動で Lambda が動いてメール通知が届く仕組み
> **使うサービス**: S3 + Lambda（Python） + SNS
> **リージョン**: ap-northeast-1（東京）
> **所要時間**: 約 45〜60 分

---

## このハンズオンで学べること

- 「イベント駆動」とは何か
- S3 のイベント通知で Lambda を自動起動する方法
- SNS でメール通知を送る方法
- Lambda でファイル情報を取得する方法

---

## 前提知識

- AWS CLI が設定済みであること
- Python の基本文法がわかること
- メールアドレスを1つ用意しておくこと（通知の確認用）

---

## 用語解説

| 用語 | ざっくり説明 |
|------|-------------|
| **イベント駆動** | 「何かが起きたら自動で処理が走る」仕組み。人がボタンを押さなくても動く |
| **S3 イベント通知** | S3 にファイルがアップロードされたときに「アップロードされたよ！」と通知する機能 |
| **SNS** | Simple Notification Service。メールや SMS で通知を送れるサービス |
| **トピック** | SNS の「通知チャンネル」。トピックに送ると、登録者全員に届く |
| **サブスクリプション** | 「この通知を受け取ります」という登録のこと |
| **トリガー** | Lambda を起動するきっかけ。今回は「S3 にファイルが置かれたこと」がトリガー |

---

## 全体の流れ

```
[ファイルをアップロード] → [S3] → (イベント通知) → [Lambda] → [SNS] → [メールが届く📧]
```

やることはシンプル。この流れを 1 つずつ作っていく。

---

## Step 1：SNS トピックを作る

まず、通知の「チャンネル」を作る。

```bash
# SNS トピックを作成
aws sns create-topic --name file-upload-notify --region ap-northeast-1
```

レスポンスに `TopicArn` が表示される：

```json
{
    "TopicArn": "arn:aws:sns:ap-northeast-1:123456789012:file-upload-notify"
}
```

**この `TopicArn` をメモしておく！**（以降 `<TOPIC_ARN>` と表記する）

---

## Step 2：メールでサブスクリプション登録する

```bash
aws sns subscribe \
  --topic-arn <TOPIC_ARN> \
  --protocol email \
  --notification-endpoint your-email@example.com \
  --region ap-northeast-1
```

> `your-email@example.com` は**自分の本物のメールアドレス**に変えてね。

実行すると、入力したメールアドレスに **確認メール** が届く。

> 📧 メールの件名: 「AWS Notification - Subscription Confirmation」
> **「Confirm subscription」リンクをクリックする**（これをしないと通知が届かない！）

確認：

```bash
aws sns list-subscriptions-by-topic --topic-arn <TOPIC_ARN> --region ap-northeast-1
```

`"SubscriptionArn"` が `arn:aws:sns:...` のようになっていれば OK（`PendingConfirmation` ならまだ確認メールをクリックしていない）。

---

## Step 3：S3 バケットを作る

```bash
BUCKET_NAME="event-driven-upload-$(date +%Y%m%d)-yourname"

aws s3 mb s3://$BUCKET_NAME --region ap-northeast-1
```

> `yourname` は自分の名前に変えて、重複しないようにしよう。

---

## Step 4：Lambda 関数を作る

### 4-1. Lambda 用のフォルダとコードを用意

```bash
cd aws/04-eventDrivenPipeline
mkdir -p lambda-src
```

`lambda-src/index.py` を作成する：

```python
import json
import os
import boto3
import urllib.parse

sns = boto3.client("sns")


def handler(event, context):
    """
    S3 にファイルがアップロードされたときに自動で呼ばれる関数。
    event の中に「どのバケットの何というファイルか」が入っている。
    """
    # S3 イベントからファイル情報を取り出す
    record = event["Records"][0]
    bucket = record["s3"]["bucket"]["name"]
    key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])
    size = record["s3"]["object"]["size"]

    # ファイルサイズを読みやすい形に変換
    if size < 1024:
        size_str = f"{size} B"
    elif size < 1024 * 1024:
        size_str = f"{size / 1024:.1f} KB"
    else:
        size_str = f"{size / (1024 * 1024):.1f} MB"

    # 通知メッセージを作る
    message = (
        f"📁 ファイルがアップロードされました！\n"
        f"\n"
        f"バケット: {bucket}\n"
        f"ファイル名: {key}\n"
        f"サイズ: {size_str}\n"
        f"\n"
        f"--- 自動通知 by Lambda ---"
    )

    # SNS で通知を送る
    topic_arn = os.environ["TOPIC_ARN"]
    sns.publish(
        TopicArn=topic_arn,
        Subject="S3 ファイルアップロード通知",
        Message=message
    )

    print(f"通知送信完了: {key}")

    return {
        "statusCode": 200,
        "body": json.dumps({"message": "通知を送信しました"})
    }
```

### 4-2. ZIP に圧縮する

Lambda にアップロードするために ZIP にする。

```bash
cd lambda-src
# Windows (PowerShell) の場合
# Compress-Archive -Path index.py -DestinationPath ../function.zip

# bash の場合
zip ../function.zip index.py

cd ..
```

---

## Step 5：Lambda 用の IAM ロールを作る

Lambda が SNS にメッセージを送ったり、S3 からイベントを受け取ったりするための権限が必要。

### 5-1. 信頼ポリシーを作る

`trust-policy.json` を作成する：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### 5-2. IAM ロールを作成する

```bash
aws iam create-role \
  --role-name lambda-s3-sns-role \
  --assume-role-policy-document file://trust-policy.json
```

レスポンスの `"Arn"` をメモする（以降 `<ROLE_ARN>` と表記）。

### 5-3. 必要な権限を付ける

```bash
# Lambda の基本権限（ログ出力など）
aws iam attach-role-policy \
  --role-name lambda-s3-sns-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# SNS への送信権限
aws iam attach-role-policy \
  --role-name lambda-s3-sns-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSNSFullAccess
```

> ⏳ ロールが反映されるまで数秒かかることがある。次のステップで失敗したら少し待ってリトライしよう。

---

## Step 6：Lambda 関数をデプロイする

```bash
aws lambda create-function \
  --function-name s3-upload-notifier \
  --runtime python3.12 \
  --role <ROLE_ARN> \
  --handler index.handler \
  --zip-file fileb://function.zip \
  --environment "Variables={TOPIC_ARN=<TOPIC_ARN>}" \
  --timeout 10 \
  --region ap-northeast-1
```

> `<ROLE_ARN>` と `<TOPIC_ARN>` は自分のものに置き換えてね。

---

## Step 7：S3 から Lambda を呼び出せるように許可する

```bash
aws lambda add-permission \
  --function-name s3-upload-notifier \
  --statement-id s3-trigger \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::$BUCKET_NAME \
  --region ap-northeast-1
```

---

## Step 8：S3 イベント通知を設定する

Lambda 関数の ARN が必要。確認する：

```bash
aws lambda get-function \
  --function-name s3-upload-notifier \
  --query 'Configuration.FunctionArn' \
  --output text \
  --region ap-northeast-1
```

表示された ARN をメモ（以降 `<LAMBDA_ARN>` と表記）。

`notification-config.json` を作成する：

```json
{
  "LambdaFunctionConfigurations": [
    {
      "LambdaFunctionArn": "<LAMBDA_ARN>",
      "Events": ["s3:ObjectCreated:*"]
    }
  ]
}
```

> `<LAMBDA_ARN>` を自分のものに置き換える。

```bash
aws s3api put-bucket-notification-configuration \
  --bucket $BUCKET_NAME \
  --notification-configuration file://notification-config.json
```

---

## Step 9：動作確認！

いよいよテスト。適当なファイルを S3 にアップロードしてみる。

```bash
# テスト用ファイルを作る
echo "Hello from S3 event!" > test-file.txt

# S3 にアップロード
aws s3 cp test-file.txt s3://$BUCKET_NAME/
```

### 確認すること

1. **メールが届くか？** → 1〜2 分以内に届くはず
2. メールの中身に「バケット名」「ファイル名」「サイズ」が表示されているか？

> 📧 メールの件名: 「S3 ファイルアップロード通知」

メールが届けば成功！🎉

### もしメールが届かない場合

```bash
# Lambda のログを確認する
aws logs tail /aws/lambda/s3-upload-notifier --region ap-northeast-1 --since 10m
```

- エラーメッセージが出ていないか確認する
- SNS のサブスクリプション確認メールをクリックしたか確認する

---

## Step 10：もう少し試してみよう

```bash
# 別のファイルもアップロードしてみる
echo '{"data": "test json"}' > data.json
aws s3 cp data.json s3://$BUCKET_NAME/folder/data.json
```

フォルダ付きでアップロードしても、ちゃんとファイル名が `folder/data.json` として通知される。

---

## Step 11：後片付け（重要！）

```bash
# 1. S3 バケットの中身を削除 → バケット削除
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3 rb s3://$BUCKET_NAME

# 2. Lambda 関数を削除
aws lambda delete-function \
  --function-name s3-upload-notifier \
  --region ap-northeast-1

# 3. IAM ロールの権限を外してからロール削除
aws iam detach-role-policy \
  --role-name lambda-s3-sns-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam detach-role-policy \
  --role-name lambda-s3-sns-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSNSFullAccess

aws iam delete-role --role-name lambda-s3-sns-role

# 4. SNS サブスクリプション削除
# まずサブスクリプション ARN を確認
aws sns list-subscriptions-by-topic --topic-arn <TOPIC_ARN> --region ap-northeast-1
# 表示された SubscriptionArn を使って削除
aws sns unsubscribe --subscription-arn <SUBSCRIPTION_ARN> --region ap-northeast-1

# 5. SNS トピック削除
aws sns delete-topic --topic-arn <TOPIC_ARN> --region ap-northeast-1
```

> 💡 後片付けのコマンドが多いのは、今回 SAM や CDK を使わず手動でリソースを作ったから。
> ハンズオン③ で学んだ CDK を使えば `cdk destroy` 一発で済む。その便利さを実感しよう！

---

## まとめ

このハンズオンでやったこと：

1. SNS トピックを作って、メール通知の仕組みを用意した
2. Lambda 関数を Python で書いて、ファイル情報を取得→通知する処理を作った
3. S3 のイベント通知で Lambda を自動起動するよう設定した
4. ファイルをアップロードして、自動でメール通知が届くことを確認した

```
[ファイル UP] → [S3] --イベント--> [Lambda] --publish--> [SNS] --email--> [📧]

  人が操作するのは「ファイルをアップロードする」だけ！
  あとは全部自動で動く = イベント駆動
```

---

## 発展課題

- [ ] 画像ファイル（.jpg, .png）がアップロードされたときだけ通知するようにフィルタを設定してみよう
- [ ] Lambda の中で S3 からファイルをダウンロードして内容を読み取ってみよう
- [ ] この構成をハンズオン③ の CDK で書き直してみよう（後片付けが楽になる！）
- [ ] SNS の代わりに SES（メール送信サービス）を使ってリッチな HTML メールを送ってみよう