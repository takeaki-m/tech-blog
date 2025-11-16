---
title: "AWS SAMをTypeScriptにて利用する際のつまづきポイント"
emoji: "🚧" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["sam","aws-sam","TypeScript" ] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

AWS Lambdaでアプリケーションを構築する際に、構成管理ツールとしてAWS SAMを利用するシーンは少ないくないと思います。

今回AWS SAMを本格的に利用する機会がありました。その際につまづくポイントがいくつかありました。

問題の直接的な解決となる記事も多くなかったため、まとめて残しておきたいと思います。

なお記事執筆時点(2025/11)時点の情報ですので、将来的に課題が解決されている可能性ももちろんあります。

以下、私が遭遇した問題点と、その解決方法を列挙します。
TypeScriptが前提となりますが、他のプログラミング言語でも問題の構造は同じかと思います。

## AWS SAM は内部でnpmを利用している。そのため、pnpmやyarnに特化した設定でビルドが失敗する可能性がある。

### 仕様
AWS SAM利用時には、アプリケーションのbuildで`sam build`コマンドが実行されます。

このコマンドは、内部では`npm`を利用しています。

そのため、プロジェクトの構成管理ツールで`pnpm`や`yarn`などを利用している場合、**npmで使えない設定が存在すると、sam buildは失敗します。**

### 問題
私のケースでは、pnpmでmonorepoを構成していました。
pnpmでの共通パッケージを定義し、その共通パッケージを読み込む場合、以下のように`package.json`に記載します。 (共通packageとして、foo,barを、アプリケーションにて読み込む場合)

この時`workspace`という設定は、`npm`では存在しないため、ビルドに失敗します

```json
{
  "dependencies": {
    "foo": "workspace:*",
    "bar": "workspace:*",
  }
}```

https://pnpm.io/workspaces

### 対処方法

AWS SAMの公式ドキュメントに設定が記載されていますが、build方法にて、`makefile`の方式を採用します。
内部でnpmを利用するAWS SAMのbuild方法ではなく、独自でビルド方式を定義するということです。

https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/building-custom-runtimes.html

`Makefile`を準備します。Makefileの設定には命名規則があり、`build-${template.ymlで定義した関数名}`とする必要があります。

```template.yml
Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: hello_world/
      Handler: app.lambda_handler
      Runtime: python3.12
    Metadata:
      BuildMethod: makefile
```

```Makefile
build-HelloWorldFunction: #←命名を揃える必要あり
	cp *.py $(ARTIFACTS_DIR)
	cp requirements.txt $(ARTIFACTS_DIR)
	python -m pip install -r requirements.txt -t $(ARTIFACTS_DIR)
	rm -rf $(ARTIFACTS_DIR)/bin
```


## sam buildのartifactがローカルに存在すると、sam localコマンドで起動した際にそのartifacnを参照する

### 仕様
sam では、ローカル開発時には`sam local`、deploy時には`sam deploy`コマンドを実行します。

これらのコマンドを実行する前に、`sam build`コマンドが必須となります。

`sam build`コマンドにより生成される、`.aws-sam`ディレクトリ配下のartifactを参照するためです。(sam localについては、このディレクトリが存在しない場合には、アプリリソースそのものを参照します。)

公式ドキュメントには以下の通りに記載されています。

> When using sam deploy, the AWS SAM CLI deploys your application’s build artifacts located in the .aws-sam directory. When you make changes to your application's original files, run sam build to update the .aws-sam directory before deploying.

https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/using-sam-cli-deploy.html

> If your application doesn’t have a .aws-sam directory and your function uses an interpreted language, the AWS SAM CLI will automatically update your function by creating a new container and hosting it.
> If your application does have a .aws-sam directory, you need to run sam build to update your function. Then run sam local start-api again to host the function.

https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/using-sam-cli-local-start-api.html

### 問題

利用開始当初、この参照先リソースの関係性が理解できておらず、`sam local`実行前の`sam build`実行を忘れておりました。

そのため、アプリケーションソースコードに変更反映->`sam local`で起動をしても、ソースコードに加えた変更が反映されません。
この課題は当初本当に原因がわからず、`aws sam`の利用をやめようかと考えました笑

### 対処方法

結局、上記の`.aws-sam`ディレクトリを削除することで対応しました。

また、`Makefile`方式に切り替えたことで、この問題は発生しなくなりました。


## localでは再現できないAWS本番の機能がある。API GatewayへのBinaryData送信や、API GatewayでのCognitoの認証検証など

### 仕様

AWSクラウド上で動作する機能のうち、aws samでのローカルでは利用できない機能があります。。どのような機能が利用できないのか全体はわかっていません。
ですが、**AWS SAMでは、AWSクラウドの機能全てをローカルで利用できるわけではない**と頭の片隅に置いておくだけでも、スタックする可能性が低くなると思います。

私の場合には、以下のような機能が、ローカルでは利用できませんでした。

#### API Gatewayでの認証機能

Cognitoと連携し、Cognito未認証のユーザーにはAPI Gatewayにてリクエストを拒否する機能です。

Ga
#### Binary Dataの処理

API Gatewayに対して、`multipart/form-data` を指定して、音声データをアップロードしようとしましたが、ローカルでは、502のエラーが発生します。 w

```sh

$ sam local start-api
[snipped]
UnicodeDecodeError while processing HTTP request: 'utf-8' codec can't decode byte 0xff in position 5782: invalid start byte
2023-05-16 07:15:50 127.0.0.1 - - [16/May/2023 07:15:50] "POST /whatever HTTP/1.1" 502 -
```

※一般的にはAPI GatewayS3にアップロードが推奨されていますが、処理時間と)


  * 補足
    * 私が利用していたHTTP APIでは、
  * https://aws.amazon.com/jp/blogs/news/handling-binary-data-using-amazon-api-gateway-http-apis/

https://stackoverflow.com/questions/68821753/managing-files-inside-multipart-form-data-api-gateway-request-body
