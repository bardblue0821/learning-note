# ハンズオン①：サーバーレス Web API を作ろう

> **作るもの**: TODO（やることリスト）を管理する Web API
> **使うサービス**: Lambda（Python） + API Gateway + DynamoDB
> **リージョン**: ap-northeast-1（東京）
> **所要時間**: 約 60〜90 分

---

## このハンズオンで学べること

- 「サーバーレス」とは何か
- Lambda 関数を Python で書いてデプロイする方法
- API Gateway で HTTP のエンドポイント（URL）を作る方法
- DynamoDB にデータを保存・取得する方法

---

## 前提知識

- AWS CLI が設定済みであること（`aws configure` が完了していること）
- Python の基本文法（関数、辞書、if 文など）がわかること
- ターミナル（コマンドプロンプト / bash）の基本操作ができること

---

## 用語解説（最初に読んでね）

| 用語 | ざっくり説明 |
|------|-------------|
| **Lambda** | 「関数だけ」をクラウドに置いて動かせるサービス。サーバーを借りなくていい |
| **API Gateway** | インターネットからのリクエスト（URL アクセス）を受け取って Lambda に渡す「受付窓口」 |
| **DynamoDB** | AWS が提供するデータベース。Excel の表みたいにデータを保存できる |
| **SAM** | Lambda や API Gateway をまとめて設定・デプロイできるツール |
| **デプロイ** | 作ったプログラムを AWS 上に配置して動かせるようにすること |

---

## Step 0：SAM CLI をインストールする

SAM CLI は Lambda アプリを簡単に作成・デプロイするためのツール。

### Windows の場合

```bash
# winget でインストール（推奨）
winget install Amazon.SAM-CLI

# インストール確認
sam --version
```

> 💡 `sam --version` で `SAM CLI, version 1.x.x` と表示されれば OK！

---

## Step 1：プロジェクトを作成する

VSCode のターミナルで以下を実行する。

```bash
# aws ディレクトリ内で作業する
cd aws/01-serverlessWeb

# SAM プロジェクトを作成
sam init
```

対話式で聞かれるので、以下のように答える：

| 質問 | 選択 |
|------|------|
| Which template source would you like to use? | `1 - AWS Quick Start Templates` |
| Choose an AWS Quick Start application template | `1 - Hello World Example` |
| Use the most popular runtime and package type? | `N` |
| Which runtime would you like to use? | `python3.12`（または最新の Python） |
| What package type would you like to use? | `1 - Zip` |
| Would you like to enable X-Ray tracing...? | `N` |
| Would you like to enable monitoring...? | `N` |
| Would you like to set Structured Logging...? | `N` |
| Project name | `todo-api`（そのまま Enter でも OK） |

```
todo-api/ が作成される
```

---

## Step 2：プロジェクトの中身を確認する

```
todo-api/
├── hello_world/          ← Lambda 関数のコードが入っている
│   ├── app.py            ← ★ ここにメインのコードを書く
│   └── requirements.txt  ← 使う Python ライブラリの一覧
├── template.yaml         ← ★ AWS リソースの設計図
└── ...
```

> **template.yaml** が設計図、**app.py** がプログラム本体、と覚えよう。

---

## Step 3：DynamoDB テーブルを設計図に追加する

`todo-api/template.yaml` を開いて、中身を以下に**丸ごと置き換える**。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: TODO API - Serverless Web API Handson

Globals:
  Function:
    Timeout: 10

Resources:
  # ---- DynamoDB テーブル ----
  TodoTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: TodoTable
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S       # S = 文字列型
      KeySchema:
        - AttributeName: id
          KeyType: HASH          # HASH = 主キー（一意な識別子）
      BillingMode: PAY_PER_REQUEST  # 使った分だけ課金（無料枠あり）

  # ---- Lambda 関数 ----
  TodoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: hello_world/
      Handler: app.lambda_handler
      Runtime: python3.12
      Architectures:
        - x86_64
      Environment:
        Variables:
          TABLE_NAME: TodoTable
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TodoTable
      Events:
        # GET /todos → 一覧取得
        GetTodos:
          Type: Api
          Properties:
            Path: /todos
            Method: get
        # POST /todos → 新規作成
        CreateTodo:
          Type: Api
          Properties:
            Path: /todos
            Method: post
        # DELETE /todos/{id} → 削除
        DeleteTodo:
          Type: Api
          Properties:
            Path: /todos/{id}
            Method: delete

Outputs:
  ApiUrl:
    Description: "API Gateway のエンドポイント URL"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/"
```

### この設計図のポイント

- **TodoTable**: `id` を主キーにした DynamoDB テーブル
- **TodoFunction**: 1つの Lambda 関数で GET / POST / DELETE の3つの API を処理する
- **Policies**: Lambda が DynamoDB を読み書きする権限を自動で付ける

---

## Step 4：Lambda 関数を書く（Python）

`todo-api/hello_world/app.py` を開いて、以下に**丸ごと置き換える**。

```python
import json
import uuid
import os
import boto3

# DynamoDB に接続する
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(os.environ["TABLE_NAME"])


def lambda_handler(event, context):
    """
    API Gateway からのリクエストを受け取って処理する関数。
    event の中に「どの HTTP メソッドか」「どんなデータが送られたか」が入っている。
    """
    method = event["httpMethod"]       # GET, POST, DELETE のどれか
    path = event.get("pathParameters") # URL に含まれるパラメータ（例: /todos/123 の 123）

    try:
        # ----- GET /todos → TODO 一覧を返す -----
        if method == "GET":
            result = table.scan()  # テーブルの中身を全部取得
            return response(200, result["Items"])

        # ----- POST /todos → 新しい TODO を作る -----
        elif method == "POST":
            body = json.loads(event["body"])  # リクエストの中身を取り出す
            item = {
                "id": str(uuid.uuid4()),      # ランダムな ID を生成
                "title": body["title"],        # TODO のタイトル
                "done": False                  # 最初は未完了
            }
            table.put_item(Item=item)  # DynamoDB に保存
            return response(201, item)

        # ----- DELETE /todos/{id} → TODO を削除する -----
        elif method == "DELETE":
            todo_id = path["id"]
            table.delete_item(Key={"id": todo_id})
            return response(200, {"message": "削除しました"})

        else:
            return response(400, {"message": "対応していないメソッドです"})

    except Exception as e:
        print(f"エラー発生: {e}")  # CloudWatch Logs に出力される
        return response(500, {"message": "サーバーエラーが発生しました"})


def response(status_code, body):
    """API Gateway に返すレスポンスを組み立てるヘルパー関数"""
    return {
        "statusCode": status_code,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"  # どこからでもアクセス可能にする
        },
        "body": json.dumps(body, ensure_ascii=False)  # 日本語が文字化けしないように
    }
```

### コードの流れ

```
ユーザー → API Gateway → Lambda（この app.py）→ DynamoDB
                                ↓
                        結果を JSON で返す
```

---

## Step 5：デプロイする

```bash
cd todo-api

# 初回デプロイ（対話式で設定）
sam deploy --guided
```

対話式で聞かれるので、以下のように答える：

| 質問 | 入力 |
|------|------|
| Stack Name | `todo-api`（そのまま Enter） |
| AWS Region | `ap-northeast-1` |
| Confirm changes before deploy | `Y` |
| Allow SAM CLI IAM role creation | `Y` |
| Disable rollback | `N` |
| TodoFunction has no authentication... | `y`（警告だけなので OK） |
| Save arguments to configuration file | `Y` |
| SAM configuration file | そのまま Enter |
| SAM configuration environment | そのまま Enter |

変更内容が表示されたら `y` で確定。

> ⏳ デプロイには 2〜3 分かかる。コーヒーでも飲んで待とう。

デプロイが完了すると、以下のような URL が表示される：

```
Outputs
-------------------------------------------
Key                 ApiUrl
Description         API Gateway のエンドポイント URL
Value               https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/Prod/
```

**この URL をメモしておく！**（以降 `<API_URL>` と表記する）

---

## Step 6：動作確認する

VSCode のターミナルで curl コマンドを使って API をテストする。

### 6-1. TODO を作成する（POST）

```bash
curl -X POST <API_URL>todos \
  -H "Content-Type: application/json" \
  -d '{"title": "AWS の勉強をする"}'
```

成功すると、こんなレスポンスが返る：

```json
{
  "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "title": "AWS の勉強をする",
  "done": false
}
```

### 6-2. TODO 一覧を取得する（GET）

```bash
curl <API_URL>todos
```

さっき作った TODO が返ってくるはず。

### 6-3. TODO を削除する（DELETE）

```bash
curl -X DELETE <API_URL>todos/<上で取得した id>
```

### 6-4. もう一度一覧を取得して、消えていることを確認

```bash
curl <API_URL>todos
```

空の配列 `[]` が返れば成功！🎉

---

## Step 7：後片付け（重要！）

ハンズオンが終わったら、**必ずリソースを削除する**。放置すると課金される可能性がある。

```bash
# SAM で作ったリソースを全部削除
sam delete --stack-name todo-api
```

`Are you sure you want to delete...?` と聞かれたら `y` と答える。

---

## まとめ

このハンズオンでやったこと：

1. **SAM CLI** で Lambda プロジェクトを作った
2. **template.yaml** に DynamoDB テーブルと Lambda 関数を定義した
3. **app.py** に Python で API のロジックを書いた
4. `sam deploy` で AWS にデプロイした
5. **curl** で API の動作確認をした

```
[ユーザー] --HTTP--> [API Gateway] --イベント--> [Lambda] --読み書き--> [DynamoDB]
                         ↑                         |
                     URL を提供              結果を JSON で返す
```

---

## 発展課題（余裕があればやってみよう）

- [ ] PUT /todos/{id} で TODO の完了状態を更新する機能を追加してみよう
- [ ] title が空のときにエラーを返すバリデーションを追加してみよう
- [ ] AWS Toolkit で Lambda のログ（CloudWatch Logs）を確認してみよう