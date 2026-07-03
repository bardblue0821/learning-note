# ハンズオン③：AWS CDK でインフラをコードで管理しよう

> **作るもの**: Lambda + API Gateway + DynamoDB の構成を TypeScript のコードで定義する
> **使うサービス**: AWS CDK（CloudFormation）
> **リージョン**: ap-northeast-1（東京）
> **所要時間**: 約 60〜90 分

---

## このハンズオンで学べること

- Infrastructure as Code（IaC）とは何か
- CDK を使って TypeScript でインフラを定義する方法
- `cdk deploy` でリソースを作成し、`cdk destroy` で削除する方法
- ハンズオン① でやった構成を「コードで再現」するという考え方

---

## 前提知識

- AWS CLI が設定済みであること
- Node.js + npm がインストールされていること
- TypeScript の基本がわかること（型、クラスなど）
- ハンズオン① の内容を理解していると理想的

---

## 用語解説

| 用語 | ざっくり説明 |
|------|-------------|
| **IaC** | Infrastructure as Code。インフラ（サーバーやDB）をコードで定義する手法 |
| **CDK** | Cloud Development Kit。TypeScript や Python でインフラを書ける AWS のツール |
| **CloudFormation** | AWS のインフラ管理サービス。CDK は内部でこれを使っている |
| **スタック** | CloudFormation でまとめて管理するリソースの単位。「プロジェクト」みたいなもの |
| **Construct** | CDK の部品。Lambda や DynamoDB などの AWS リソースを表すオブジェクト |
| **Bootstrap** | CDK が動くための土台を AWS アカウントに用意すること（初回だけ） |

---

## なぜ IaC が必要なの？

手動（AWS コンソール画面）でリソースを作ると…

- 何を作ったか忘れる 😱
- 同じ環境をもう一度作れない 😱
- チームで共有できない 😱

→ **コードで書けば、Git で管理でき、何度でも同じ環境を再現できる！**

---

## Step 0：CDK をインストールする

```bash
# CDK CLI をグローバルインストール
npm install -g aws-cdk

# 確認
cdk --version
```

> `2.x.x` と表示されれば OK！

---

## Step 1：CDK プロジェクトを作る

```bash
cd aws/03-IaC

# CDK プロジェクトを初期化
mkdir todo-infra && cd todo-infra
cdk init app --language typescript
```

以下のような構造が作られる：

```
todo-infra/
├── bin/
│   └── todo-infra.ts        ← アプリのエントリーポイント
├── lib/
│   └── todo-infra-stack.ts   ← ★ ここにインフラを定義する
├── package.json
├── tsconfig.json
└── cdk.json
```

---

## Step 2：Bootstrap する（初回のみ）

CDK が使う S3 バケットなどを AWS アカウントに用意する。**これは AWS アカウントにつき1回だけ**やればいい。

```bash
cdk bootstrap aws://<ACCOUNT_ID>/ap-northeast-1
```

> `<ACCOUNT_ID>` は `aws sts get-caller-identity --query Account --output text` で確認できる。

> ⏳ 1〜2 分かかる。`Environment aws://...  bootstrapped.` と出れば OK。

---

## Step 3：必要な CDK ライブラリをインストールする

```bash
npm install
```

> CDK v2 では `aws-cdk-lib` に全サービスが入っているので追加インストールは不要。

---

## Step 4：Lambda 関数のコードを用意する

Lambda 関数は Python で書く。プロジェクト内に Lambda 用のフォルダを作る。

```bash
mkdir -p lambda
```

`lambda/index.py` を作成する：

```python
import json
import uuid
import os
import boto3

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(os.environ["TABLE_NAME"])


def handler(event, context):
    """API Gateway からのリクエストを処理する"""
    method = event["httpMethod"]
    path = event.get("pathParameters")

    try:
        if method == "GET":
            result = table.scan()
            return make_response(200, result["Items"])

        elif method == "POST":
            body = json.loads(event["body"])
            item = {
                "id": str(uuid.uuid4()),
                "title": body["title"],
                "done": False
            }
            table.put_item(Item=item)
            return make_response(201, item)

        elif method == "DELETE":
            todo_id = path["id"]
            table.delete_item(Key={"id": todo_id})
            return make_response(200, {"message": "削除しました"})

        else:
            return make_response(400, {"message": "対応していないメソッドです"})

    except Exception as e:
        print(f"エラー: {e}")
        return make_response(500, {"message": "サーバーエラーが発生しました"})


def make_response(status_code, body):
    return {
        "statusCode": status_code,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
        },
        "body": json.dumps(body, ensure_ascii=False)
    }
```

> ハンズオン① とほぼ同じコード。今回は **インフラの定義方法** がメイン。

---

## Step 5：インフラを TypeScript で定義する（メインパート！）

`lib/todo-infra-stack.ts` を開いて、以下に**丸ごと置き換える**：

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';

export class TodoInfraStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ============================================
    // 1. DynamoDB テーブルを定義
    // ============================================
    const table = new dynamodb.Table(this, 'TodoTable', {
      tableName: 'TodoTableCDK',
      partitionKey: {
        name: 'id',            // 主キーの名前
        type: dynamodb.AttributeType.STRING,  // 文字列型
      },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,  // 使った分だけ課金
      removalPolicy: cdk.RemovalPolicy.DESTROY,  // cdk destroy で削除する
    });

    // ============================================
    // 2. Lambda 関数を定義
    // ============================================
    const todoFunction = new lambda.Function(this, 'TodoFunction', {
      runtime: lambda.Runtime.PYTHON_3_12,  // Python 3.12
      handler: 'index.handler',             // ファイル名.関数名
      code: lambda.Code.fromAsset('lambda'), // lambda/ フォルダのコードを使う
      environment: {
        TABLE_NAME: table.tableName,  // 環境変数で DynamoDB テーブル名を渡す
      },
    });

    // Lambda に DynamoDB の読み書き権限を付ける
    table.grantReadWriteData(todoFunction);

    // ============================================
    // 3. API Gateway を定義
    // ============================================
    const api = new apigateway.RestApi(this, 'TodoApi', {
      restApiName: 'Todo Service (CDK)',
    });

    // /todos エンドポイント
    const todos = api.root.addResource('todos');
    todos.addMethod('GET', new apigateway.LambdaIntegration(todoFunction));
    todos.addMethod('POST', new apigateway.LambdaIntegration(todoFunction));

    // /todos/{id} エンドポイント
    const singleTodo = todos.addResource('{id}');
    singleTodo.addMethod('DELETE', new apigateway.LambdaIntegration(todoFunction));

    // ============================================
    // 4. 出力（デプロイ後に URL を表示する）
    // ============================================
    new cdk.CfnOutput(this, 'ApiUrl', {
      value: api.url ?? 'URL が取得できませんでした',
      description: 'API Gateway のエンドポイント URL',
    });
  }
}
```

### ここがポイント！

```
手動でやると…               CDK で書くと…
─────────────────────      ─────────────────────
DynamoDB コンソールで       → new dynamodb.Table(...)
テーブル作成

Lambda コンソールで         → new lambda.Function(...)
関数作成

API Gateway コンソールで    → new apigateway.RestApi(...)
API 作成

IAM で権限設定             → table.grantReadWriteData(fn)
                             たった1行！
```

---

## Step 6：デプロイ前に確認する

```bash
# CloudFormation テンプレートを生成して確認
cdk synth
```

大量の YAML が表示される。これが CDK から自動生成された CloudFormation テンプレート。

```bash
# 現在の AWS 環境との差分を確認
cdk diff
```

「何が新しく作られるか」が表示される。

---

## Step 7：デプロイする

```bash
cdk deploy
```

`Do you wish to deploy these changes?` と聞かれたら `y` と答える。

> ⏳ 2〜5 分かかる。

デプロイが完了すると、API の URL が表示される：

```
Outputs:
TodoInfraStack.ApiUrl = https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/prod/
```

---

## Step 8：動作確認

ハンズオン① と同じように curl で確認する。

```bash
# TODO を作成
curl -X POST <API_URL>todos \
  -H "Content-Type: application/json" \
  -d '{"title": "CDK でデプロイできた！"}'

# 一覧取得
curl <API_URL>todos
```

---

## Step 9：変更を体験する（CDK の真価）

CDK のすごいところは、**コードを変えてデプロイするだけで差分が反映される**こと。

例えば、Lambda のタイムアウトを変更してみよう。

`lib/todo-infra-stack.ts` の Lambda 定義に `timeout` を追加：

```typescript
const todoFunction = new lambda.Function(this, 'TodoFunction', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
  timeout: cdk.Duration.seconds(30),  // ← これを追加！
  environment: {
    TABLE_NAME: table.tableName,
  },
});
```

```bash
# 差分を確認
cdk diff

# 差分だけデプロイ
cdk deploy
```

> Lambda の設定だけが更新される。DynamoDB や API Gateway は変わらない。便利！

---

## Step 10：後片付け（重要！）

```bash
# CDK で作ったリソースを全部削除
cdk destroy
```

`Are you sure you want to delete...?` と聞かれたら `y`。

> 💡 `cdk destroy` 一発で、DynamoDB・Lambda・API Gateway がすべて消える。
> これも IaC の大きなメリット。手動で一個ずつ消す必要がない。

---

## まとめ

このハンズオンでやったこと：

1. CDK プロジェクトを作った
2. DynamoDB / Lambda / API Gateway を **TypeScript のコード**で定義した
3. `cdk deploy` でデプロイし、`cdk diff` で差分を確認した
4. `cdk destroy` で後片付けした

### 手動 vs CDK 比較

| 項目 | 手動（コンソール） | CDK |
|------|---------------------|-----|
| 再現性 | ❌ 手順を忘れる | ✅ コードで再現可能 |
| バージョン管理 | ❌ できない | ✅ Git で管理 |
| チーム共有 | ❌ 手順書が必要 | ✅ コードを共有 |
| 削除 | ❌ 一個ずつ手動 | ✅ `cdk destroy` 一発 |

---

## 発展課題

- [ ] 新しいリソース（SNS トピックなど）を追加してみよう
- [ ] `cdk.json` の `context` でステージ（dev / prod）を切り替えられるようにしてみよう
- [ ] ハンズオン② の S3 + CloudFront 構成も CDK で書いてみよう