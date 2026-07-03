# ハンズオン⓪：AWS 学習環境を構築しよう

> **目的**: ハンズオン①〜④ を実行するための環境をすべて整える
> **前提**: Windows / 社用PC（exe・msi インストール不可）/ Git Bash / VSCode
> **所要時間**: 約 60〜90 分

---

## 環境の全体像

```
必要なもの            インストール方法         用途
──────────────────────────────────────────────────────────────
Git for Windows       ✅ インストール済み       pyenv-win の導入・バージョン管理
Node.js (v24)         ✅ インストール済み       CDK / React / Vite
Python (3.12+)        pyenv-win で導入         Lambda 関数の開発
AWS CLI               pip で導入               AWS の操作全般
SAM CLI               pip で導入               Lambda のデプロイ（ハンズオン①）
AWS CDK               npm で導入               IaC（ハンズオン③）
AWS Toolkit           VSCode 拡張で導入         VSCode から AWS を操作
```

---

## 目次

1. [AWS アカウントの安全設定（IAM ユーザー作成）](#step-1aws-アカウントの安全設定iam-ユーザー作成)
2. [pyenv-win で Python をインストール](#step-2pyenv-win-で-python-をインストール)
3. [AWS CLI をインストール](#step-3aws-cli-をインストール)
4. [AWS CLI を設定する（aws configure）](#step-4aws-cli-を設定するaws-configure)
5. [SAM CLI をインストール](#step-5sam-cli-をインストール)
6. [AWS CDK をインストール](#step-6aws-cdk-をインストール)
7. [VSCode 拡張機能をインストール](#step-7vscode-拡張機能をインストール)
8. [動作確認](#step-8動作確認)

---

## Step 1：AWS アカウントの安全設定（IAM ユーザー作成）

### なぜ IAM ユーザーが必要？

AWS アカウントを作ると「ルートユーザー」ができる。これは**全権限を持つ最強のアカウント**。
日常的にルートユーザーを使うのは危険（パスワードが漏れたら全リソースが乗っ取られる）なので、
**権限を限定した IAM ユーザーを作って、普段はそちらを使う**。

```
ルートユーザー = 家の大家さん（全部の鍵を持っている）
IAM ユーザー   = 住人（自分の部屋の鍵だけ持っている）
```

### 1-1. ルートアカウントに MFA を設定する

MFA（多要素認証）= パスワード＋スマホアプリの確認コードで二重ロック。

1. ブラウザで [AWS コンソール](https://console.aws.amazon.com/) にルートユーザーでログイン
2. 右上のアカウント名 → 「セキュリティ認証情報」
3. 「MFA デバイスの割り当て」→「MFA デバイスを追加」
4. 「認証アプリケーション」を選択
5. スマホに **Google Authenticator** か **Microsoft Authenticator** をインストール
6. QR コードをスキャンして、表示される 6 桁のコードを 2 回入力
7. 「MFA を追加」をクリック

> ⚠️ これをやらないと、AWS アカウントが不正利用されて高額請求されるリスクがある！

### 1-2. IAM ユーザーを作成する

1. AWS コンソール上部の検索バーで「IAM」と検索 → IAM ダッシュボードを開く
2. 左メニュー「ユーザー」→「ユーザーを作成」

| 設定項目 | 入力値 |
|---------|--------|
| ユーザー名 | `admin-user`（好きな名前で OK） |
| AWS マネジメントコンソールへのアクセスを提供 | ✅ チェックする |
| パスワード | 自分で設定（忘れないように！） |

3. 「次へ」→ 許可を設定

| 設定項目 | 選択 |
|---------|------|
| 許可のオプション | 「ポリシーを直接アタッチする」 |
| ポリシー | `AdministratorAccess` を検索してチェック |

> 💡 学習用なので `AdministratorAccess`（全権限）を付ける。本番では必要最小限の権限にするのが原則。

4. 「次へ」→「ユーザーの作成」

### 1-3. IAM ユーザー用のアクセスキーを作成する

CLI から AWS を操作するために「アクセスキー」が必要。

1. 作成した IAM ユーザーの詳細ページを開く
2. 「セキュリティ認証情報」タブ → 「アクセスキーを作成」
3. ユースケース: 「コマンドラインインターフェイス（CLI）」を選択
4. 確認チェックを入れて「次へ」→「アクセスキーを作成」

**以下の 2 つを安全な場所にメモする（二度と表示されない！）**：

```
アクセスキー ID:     AKIAXXXXXXXXXXXXXXXX
シークレットアクセスキー: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

> ⚠️ **絶対に Git にコミットしたり、他人に共有しないこと！**
> 漏洩すると AWS リソースを不正利用される。

### 1-4. 今後は IAM ユーザーでログインする

ルートユーザーでの日常ログインはやめて、IAM ユーザー用のログイン URL を使う：

```
https://<アカウントID>.signin.aws.amazon.com/console
```

> アカウント ID は AWS コンソール右上のアカウント名をクリックすると確認できる。

---

## Step 2：pyenv-win で Python をインストール

### pyenv-win とは？

Python のバージョンを複数管理できるツール。
exe インストーラーを使わずに、Git Bash 上で Python をインストールできる。

### 2-1. pyenv-win をインストールする

VSCode のターミナル（Git Bash）で以下を実行：

```bash
# pyenv-win を ~/.pyenv にインストール
git clone https://github.com/pyenv-win/pyenv-win.git "$HOME/.pyenv"
```

### 2-2. 環境変数（PATH）を設定する

`~/.bashrc` に以下を追記する。Git Bash 起動時に自動で読み込まれる。

```bash
# ~/.bashrc に追記（なければ新規作成される）
cat << 'EOF' >> ~/.bashrc

# === pyenv-win 設定 ===
export PYENV="$HOME/.pyenv/pyenv-win"
export PYENV_ROOT="$HOME/.pyenv/pyenv-win"
export PYENV_HOME="$HOME/.pyenv/pyenv-win"
export PATH="$PYENV/bin:$PYENV/shims:$PATH"
EOF
```

設定を反映する：

```bash
source ~/.bashrc
```

### 2-3. 確認

```bash
pyenv --version
```

> `pyenv 3.x.x` のように表示されれば OK。

### 2-4. Python 3.12 をインストールする

```bash
# インストール可能なバージョン一覧を確認
pyenv install --list | grep "3.12"

# Python 3.12 系の最新をインストール（例: 3.12.8）
pyenv install 3.12.8

# グローバルで使うバージョンに設定
pyenv global 3.12.8

# 確認
python --version
```

> `Python 3.12.8` のように表示されれば OK！

### 2-5. pip も確認

```bash
pip --version
```

> `pip 24.x.x from ...` のように表示されれば OK。

> ⚠️ もし `python` コマンドが認識されない場合、**VSCode のターミナルを一度閉じて開き直す**と反映される。

---

## Step 3：AWS CLI をインストール

社用 PC で exe / msi インストーラーが使えないので、**pip（Python パッケージ管理）** で AWS CLI をインストールする。

```bash
# AWS CLI v1 を pip でインストール
pip install awscli

# 確認
aws --version
```

> `aws-cli/1.x.x Python/3.12.x ...` のように表示されれば OK！

> 💡 pip でインストールできるのは AWS CLI **v1** です。
> v2 は MSI インストーラーが必要ですが、v1 でもハンズオン①〜④には十分対応できます。
> 機能差はほぼありません。

---

## Step 4：AWS CLI を設定する（aws configure）

Step 1-3 でメモしたアクセスキーを使って、CLI から AWS を操作できるようにする。

```bash
aws configure
```

対話式で聞かれるので、以下のように入力する：

```
AWS Access Key ID [None]: AKIAXXXXXXXXXXXXXXXX      ← Step 1-3 のアクセスキー ID
AWS Secret Access Key [None]: XXXXXXXXXXXXXXXX...    ← Step 1-3 のシークレットアクセスキー
Default region name [None]: ap-northeast-1           ← 東京リージョン
Default output format [None]: json
```

### 設定の確認

```bash
# 自分の AWS アカウント情報が表示されれば OK
aws sts get-caller-identity
```

成功するとこんな感じ：

```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/admin-user"
}
```

> ⚠️ エラーが出た場合、アクセスキーの入力ミスが多い。`aws configure` をやり直そう。

### 設定ファイルの場所

```bash
# 認証情報
cat ~/.aws/credentials

# リージョン等の設定
cat ~/.aws/config
```

> ⚠️ `~/.aws/credentials` には秘密情報が入っている。**Git に含めないよう注意**。

---

## Step 5：SAM CLI をインストール

SAM CLI はハンズオン① で Lambda アプリをデプロイするために使う。

```bash
# pip でインストール
pip install aws-sam-cli

# 確認
sam --version
```

> `SAM CLI, version 1.x.x` と表示されれば OK！

> ⏳ インストールに数分かかることがある。

### SAM CLI のポイント

| コマンド | 用途 | Docker 必要？ |
|---------|------|--------------|
| `sam init` | プロジェクト作成 | ❌ 不要 |
| `sam build` | ビルド | ❌ 不要（`--use-container` を付けなければ） |
| `sam deploy` | AWS にデプロイ | ❌ 不要 |
| `sam local invoke` | ローカルで Lambda テスト | ✅ 必要（今回は使わない） |

> 💡 `sam local invoke` は Docker が必要だが、今回のハンズオンでは**使わない**。
> 直接 AWS にデプロイしてテストする方式で進める。

---

## Step 6：AWS CDK をインストール

CDK はハンズオン③ で使う。Node.js の npm でインストールする。

```bash
# CDK CLI をグローバルインストール
npm install -g aws-cdk

# 確認
cdk --version
```

> `2.x.x` と表示されれば OK！

---

## Step 7：VSCode 拡張機能をインストール

VSCode の拡張機能タブ（`Ctrl + Shift + X`）で以下を検索してインストールする。

### 必須

| 拡張機能 | 検索キーワード | 用途 |
|---------|--------------|------|
| AWS Toolkit | `aws toolkit` | Lambda のログ確認、リソース閲覧 |
| Python | `python ms-python` | Python コードの補完・デバッグ |

### あると便利

| 拡張機能 | 検索キーワード | 用途 |
|---------|--------------|------|
| YAML | `redhat yaml` | template.yaml の編集が楽になる |
| Thunder Client | `thunder client` | REST API のテスト（curl の代わり） |

### AWS Toolkit の初期設定

1. サイドバーに AWS アイコンが表示される → クリック
2. 「Connect to AWS」→ 「Use IAM Credentials」を選択
3. Profile: `default`（aws configure で設定したもの）
4. 接続成功すると、サイドバーに Lambda や S3 などのリソースが表示される

---

## Step 8：動作確認

すべてのツールが正しくインストールされているか確認する。

```bash
echo "=== 環境確認 ==="
echo "--- Git ---"
git --version

echo "--- Node.js ---"
node --version

echo "--- npm ---"
npm --version

echo "--- Python ---"
python --version

echo "--- pip ---"
pip --version

echo "--- AWS CLI ---"
aws --version

echo "--- SAM CLI ---"
sam --version

echo "--- CDK ---"
cdk --version

echo "--- AWS 接続確認 ---"
aws sts get-caller-identity
```

### 期待される出力（例）

```
=== 環境確認 ===
--- Git ---
git version 2.53.0.windows.2
--- Node.js ---
v24.14.0
--- npm ---
10.x.x
--- Python ---
Python 3.12.8
--- pip ---
pip 24.x.x
--- AWS CLI ---
aws-cli/1.x.x Python/3.12.8 ...
--- SAM CLI ---
SAM CLI, version 1.x.x
--- CDK ---
2.x.x
--- AWS 接続確認 ---
{
    "UserId": "AIDAXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/admin-user"
}
```

> すべて表示されれば環境構築完了！🎉

---

## チェックリスト

- [ ] AWS ルートアカウントに MFA を設定した
- [ ] IAM ユーザー（admin-user）を作成した
- [ ] IAM ユーザーのアクセスキーを取得・安全に保管した
- [ ] pyenv-win をインストールして Python 3.12 を導入した
- [ ] AWS CLI を pip でインストールした
- [ ] `aws configure` でアクセスキー・リージョンを設定した
- [ ] SAM CLI を pip でインストールした
- [ ] AWS CDK を npm でインストールした
- [ ] VSCode に AWS Toolkit 拡張をインストールした
- [ ] 動作確認がすべて通った

---

## トラブルシューティング

### `python` コマンドが見つからない

```bash
# pyenv の shims が PATH に入っているか確認
echo $PATH | tr ':' '\n' | grep pyenv

# 入っていなければ .bashrc を再読み込み
source ~/.bashrc

# それでもダメなら VSCode ターミナルを閉じて開き直す
```

### `aws: command not found`

```bash
# pip でインストールした先のパスを確認
pip show awscli | grep Location

# pip のインストール先が PATH に入っているか確認
python -m awscli --version

# PATH に追加が必要な場合
echo 'export PATH="$(python -m site --user-base)/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### `aws sts get-caller-identity` でエラー

```
An error occurred (InvalidClientTokenId)
```

→ アクセスキーが間違っている。`aws configure` をやり直す。

```
An error occurred (SignatureDoesNotMatch)
```

→ シークレットアクセスキーが間違っている。`aws configure` をやり直す。

### `sam --version` でエラー

```bash
# パスが通っていない場合
python -m samcli --version

# PATH に Scripts ディレクトリを追加
echo 'export PATH="$(python -m site --user-base)/Scripts:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### pyenv install でエラーが出る

```bash
# pyenv-win を最新版に更新
cd ~/.pyenv && git pull

# 再度インストールを試す
pyenv install 3.12.8
```

---

## セキュリティ上の注意（必ず読んでね）

### やってはいけないこと ❌

1. **アクセスキーを Git にコミットしない**
   ```bash
   # .gitignore に追記しておく
   echo ".aws/" >> ~/.gitignore
   ```

2. **ルートユーザーで日常作業しない**
   - 必ず IAM ユーザーでログインする

3. **アクセスキーを他人に教えない**
   - Slack やメールで送らない

### やるべきこと ✅

1. **請求アラートを設定する**
   - AWS コンソール → 「Billing」→「請求設定」→「無料利用枠アラートを受け取る」を ON
   - → 無料枠を超えそうになったらメールが届く

2. **使い終わったリソースはすぐ削除する**
   - 各ハンズオンの「後片付け」セクションを必ず実行する

3. **月に一度は請求ダッシュボードを確認する**
   - AWS コンソール → 「Billing and Cost Management」→ 今月の請求額を確認

---

## 次のステップ

環境構築が完了したら、以下の順番でハンズオンを進めよう：

| 順番 | ハンズオン | 内容 |
|------|-----------|------|
| 1st | [01-serverlessWeb](../01-serverlessWeb/handson.md) | Lambda + API Gateway + DynamoDB で TODO API |
| 2nd | [04-eventDrivenPipeline](../04-eventDrivenPipeline/handson.md) | S3 アップロード → Lambda → メール通知 |
| 3rd | [02-staticSiteHosting](../02-staticSiteHosting/handson.md) | React サイトを S3 + CloudFront でホスティング |
| 4th | [03-IaC](../03-IaC/handson.md) | CDK で ① の構成をコード化 |