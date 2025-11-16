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

### 問題

AWS SAM利用時には、アプリケーションのbuildで`sam build`コマンドが実行されます。

このコマンドは、内部では`npm`を利用しています。

そのため、プロジェクトの構成管理ツールで`pnpm`や`yarn`などを利用している場合、**npmで使えない設定が存在すると、sam buildは失敗します。**

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

sam では、ローカル開発時には`sam local`、deploy時には`sam deploy`コマンドを実行します。

これらのコマンドを実行する前に、`sam build`コマンドが必須となります。

`sam build`コマンドにより生成される、`.aws-sam`ディレクトリ配下のartifactを参照するためです。


公式ドキュメントには以下の通りに記載されています。

> When using sam deploy, the AWS SAM CLI deploys your application’s build artifacts located in the .aws-sam directory. When you make changes to your application's original files, run sam build to update the .aws-sam directory before deploying.

https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/using-sam-cli-deploy.html

> If your application doesn’t have a .aws-sam directory and your function uses an interpreted language, the AWS SAM CLI will automatically update your function by creating a new container and hosting it.
> If your application does have a .aws-sam directory, you need to run sam build to update your function. Then run sam local start-api again to host the function.

https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/using-sam-cli-local-start-api.html

## localでは再現できない機能あり。BinaryDataやAPI Gatewayの認証など
