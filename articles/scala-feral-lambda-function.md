---
title: "FeralとAWS SAMを使ってScalaでLambda関数を作成する"
emoji: "♦️"
type: "tech"
topics: ["Scala", "Typelevel", "aws", "awslambda", "awssam"]
published: false # 公開設定（falseにすると下書き）
---

# はじめに

2023年10月26日に、Serverless Framework v4が発表され、新しくライセンスが導入されることが発表されました。v4自体は2024年5月21日からベータ版としてリリースされ、利用できるようになりました。

https://www.serverless.com/blog/serverless-framework-v4-a-new-model

Serverless Framework v4は、Serverless Framework v3とは異なり、オープンソースのライセンスがなくなり、有料のライセンスが導入されることになり、これによってServerless Framework v4を使用するためには、有料のライセンスが必要になりました。

前年度に年間収益が200万ドルを超えた組織がv4を利用する場合に対象となるみたいですが、200万ドル以下だったかどうかの審査はないため、自己申告のようです。

リリースブログとCLI上では個人の開発者や小規模企業、非営利団体は無料で利用できると記載されていましたが今回の変更でServerless Framework意外の選択肢も試してみようと思いこの記事を書きました。

この記事では、TypelevelのライブラリであるFeralを使用してScalaでAWS Lambda関数を作成する方法を紹介します。また、AWS SAMを使って作成した関数のデプロイを行います。

本記事では以下の内容についての構築と説明を行います。

- Feralでの複数のLambda関数の作成
- AWS SAMを使ったデプロイ

:::message
Scalaを使用してLambda関数を作成しますが、実行環境はJVMではなくScala.jsを使用したNode環境を使用します。

Scalaのクロスプラットフォーム性(JVM, JS, Native)を活かして、JVMとJavaScriptの両方のランタイムをターゲットにしているTypelevelのプロジェクトであるFeralを使用してLambda関数を作成します。
:::

## Feralとは

Feralは、Cats Effectを使ってScalaでサーバーレス関数を書き、クラウドにデプロイするためのフレームワークで、JVMとJavaScriptの両方のランタイムをターゲットにしているTypelevelのプロジェクトです。

Feralの特徴や使い方については、以下の記事に詳しく記載されておりますので、参考にしてください。

https://blog.3qe.us/entry/2024/08/05/000939

## AWS SAMとは

AWS SAM（Serverless Application Model）は、AWSのサーバーレスアプリケーションを構築、テスト、デプロイするためのフレームワークです。AWS SAM CLIを使うことで、ローカルでLambda関数をテストすることができます。

AWS SAMに関しては、以下の記事に詳しく記載されておりますので、参考にしてください。

https://zenn.dev/k_tana/articles/2023-08_how-to-use-of-aws-sam

# 構築

以下の手順でFeralとAWS SAMを使ってScalaでLambda関数を作成します。

1. AWS SAMを使用したプロジェクトの作成
2. Feralを使用したLambda関数の作成
3. AWS SAMを使用したデプロイ

## AWS SAMを使用したプロジェクトの作成

まずはAWS SAM CLIをインストールします。

```shell
brew install aws-sam-cli
```

インストールが完了したら、以下のコマンドでバージョンを確認します。

```shell
sam --version
```

次に、SAMを使用してプロジェクトを作成します。

```shell
sam init
```

まず、`1 - AWS Quick Start Templates`を選択し、テンプレートの種類は`1 - Hello World Example`を選択します。

```shell
Which template source would you like to use?
        1 - AWS Quick Start Templates
        2 - Custom Template Location
Choice: 1

Choose an AWS Quick Start application template
        1 - Hello World Example
        2 - Data processing
        3 - Hello World Example with Powertools for AWS Lambda
        4 - Multi-step workflow
        5 - Scheduled task
        6 - Standalone function
        7 - Serverless API
        8 - Infrastructure event management
        9 - Lambda Response Streaming
        10 - Serverless Connector Hello World Example
        11 - Multi-step workflow with Connectors
        12 - GraphQLApi Hello World Example
        13 - Full Stack
        14 - Lambda EFS example
        15 - DynamoDB Example
        16 - Machine Learning
Template: 1
```

次に、パッケージのランタイムを選択します。ここでは`11 - nodejs20.x`を選択します。

```shell
Use the most popular runtime and package type? (Python and zip) [y/N]: N

Which runtime would you like to use?
        1 - dotnet8
        2 - dotnet6
        3 - go (provided.al2)
        4 - go (provided.al2023)
        5 - graalvm.java11 (provided.al2)
        6 - graalvm.java17 (provided.al2)
        7 - java21
        8 - java17
        9 - java11
        10 - java8.al2
        11 - nodejs20.x
        12 - nodejs18.x
        13 - nodejs16.x
        14 - python3.9
        15 - python3.8
        16 - python3.12
        17 - python3.11
        18 - python3.10
        19 - ruby3.3
        20 - ruby3.2
        21 - rust (provided.al2)
        22 - rust (provided.al2023)
Runtime: 11
```

次に、パッケージの種類を選択します。ここでは`1 - Zip`を選択します。

```shell
What package type would you like to use?
        1 - Zip
        2 - Image
Package type: 1
```

ランライムに`Node`を選択したため、`npm`のテンプレートを選択する必要があります。ここでは`1 - Hello World Example`を選択します。

※ 今回はScalaでLambda関数を作成するため、`npm`のテンプレートを選択しても使用しません。

```shell
Based on your selections, the only dependency manager available is npm.
We will proceed copying the template using npm.

Select your starter template
        1 - Hello World Example
        2 - Hello World Example TypeScript
Template: 1
```

`X-Ray`トレースを有効にするかどうかを尋ねられます。今回は使用しないため`N`を選択します。

```shell
Would you like to enable X-Ray tracing on the function(s) in your application?  [y/N]: N
```

次に、`CloudWatch Application Insights`を有効にするかどうかを尋ねられます。今回は使用しないため`N`を選択します。

```shell
Would you like to enable monitoring using CloudWatch Application Insights?
For more info, please view https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-application-insights.html [y/N]: N
```

最後に、Lambda関数のログをJSON形式で出力するかどうかを尋ねられます。今回は使用しないため`N`を選択します。

```shell
Would you like to set Structured Logging in JSON format on your Lambda functions?  [y/N]: N
```

最後にプロジェクト名を入力します。

```shell
Project name [sam-app]: feral-example

Cloning from https://github.com/aws/aws-sam-cli-app-templates (process may take a moment)                                                                                                                                      

    -----------------------
    Generating application:
    -----------------------
    Name: feral-example
    Runtime: nodejs20.x
    Architectures: x86_64
    Dependency Manager: npm
    Application Template: hello-world
    Output Directory: .
    Configuration file: feral-example/samconfig.toml
    
    Next steps can be found in the README file at feral-example/README.md
        

Commands you can use next
=========================
[*] Create pipeline: cd feral-example && sam pipeline init --bootstrap
[*] Validate SAM template: cd feral-example && sam validate
[*] Test Function in the Cloud: cd feral-example && sam sync --stack-name {stack-name} --watch
```

プロジェクトが作成されたら、プロジェクト構成は以下のようになります。

```text
.
├── README.md
├── events
│   └── event.json
├── hello-world
│   ├── app.mjs
│   ├── package.json
│   ├── .npmignore
│   └── tests
│       └── unit
│           └── test-handler.mjs
├── samconfig.toml
└── template.yaml
```

今回はScalaでLambda関数を作成するため、`hello-world`ディレクトリは削除してしまいましょう。

```shell
rm -rf hello-world
```

## Feralを使用したLambda関数の作成

次に、Feralを使用してScalaでLambda関数を作成します。

:::message
以下のコマンドでFeralのテンプレートを作成できるのですが、構築されたアプリケーションのバージョンが古いため今回は使用しません。

```shell
sbt new typelevel/feral.g8 -o .
```
:::

そのため、手動でテンプレートを作成します。

まずは、プロジェクトにFeralのプラグインを追加します。

```sbt
addSbtPlugin("org.typelevel" %% "sbt-feral-lambda" % "0.3.0")
```

そして、`build.sbt`を以下の内容で作成します。

:::message
現在Feralは1つのsbtプロジェクトに1つのLambda関数しかサポートしていないため、複数のLambda関数を作成する場合は複数のsbtプロジェクトを作成する必要があります。
:::

```scala
ThisBuild / organization := "com.example"
ThisBuild / scalaVersion := "3.3.3"

lazy val helloWorld = (project in file("functions/hello-world"))
  .enablePlugins(LambdaJSPlugin)
  .settings(
    name := "hello-world",
    libraryDependencies ++= Seq(
      "org.typelevel" %% "feral-lambda" % "0.3.0"
    )
  )

lazy val root = (project in file("."))
  .settings(name := "feral-example")
  .aggregate(helloWorld)
```

次に、`functions/hello-world`ディレクトリを作成し、その中に`src/main/scala`ディレクトリを作成します。

```shell
mkdir -p functions/hello-world/src/main/scala
```

`functions/hello-world/src/main/scala`ディレクトリに`HelloWorld.scala`を作成します。

```scala
import cats.effect.*
import feral.lambda.*

case class Param(message: String) derives io.circe.Decoder

object HelloWorld extends IOLambda.Simple[Param, Unit]:

  def apply(event: Param, context: Context[IO], init: Init): IO[Option[Unit]] =
    IO.println(s"got message: [${event.message}]").as(Some(()))
```

次に、このLambda関数をjsファイルとしてビルドします。

```shell
sbt npmPackage
```

すると`target/scala-(Scalaバージョン)/npm-package`以下にファイルが生成されます。

```text
.
├── functions
│   └── hello-world
│       ├── target
│       │   └── scala-3.3.3
│       │       ├── npm-package
│       │       │   ├── index.js
│       │       │   └── package.json
```

このファイルをAWS SAMを使ってデプロイします。

## AWS SAMを使用したデプロイ

まずは、`template.yaml`を以下の内容に修正します。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  feral-example

  Sample SAM Template for feral-example
  
Globals:
  Function:
    Timeout: 3

Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: functions/hello-world/target/scala-3.3.3/npm-package/
      Handler: index.HelloWorld
      Runtime: nodejs20.x
      Architectures:
        - x86_64

Outputs:
  HelloWorldFunction:
    Description: "Hello World Lambda Function ARN"
    Value: !GetAtt HelloWorldFunction.Arn
  HelloWorldFunctionIamRole:
    Description: "Implicit IAM Role created for Hello World function"
    Value: !GetAtt HelloWorldFunctionRole.Arn
```

AWS SAMは、`CodeUri`にLambda関数のコードが格納されているディレクトリを指定します。今回はFeralによって生成された`functions/hello-world/target/scala-3.3.3/npm-package/`を指定しています。

そして、`Handler`にはLambda関数のエントリーポイントを指定します。Feralによって生成されたLambda関数のハンドラーは`index.{オブジェクト名}`となるため、`index.HelloWorld`を指定しています。

デプロイを行う前に変更した`template.yaml`が正しいかを確認します。

```shell
sam validate
```

問題がなければ、次にビルドを行います。

```shell
sam build
```

ビルドが完了したので、デプロイを行う前にローカル環境でテストを行います。

まず、今回は単純なJSONをイベントとして使用します。`events/event.json`を以下の内容に修正します。

```json
{
  "message": "Hello from Lambda!"
}
```

次に、ローカル環境でテストを行います。

```shell
sam local invoke HelloWorldFunction --event events/event.json
```

:::message
ローカル環境でテストを行う際には、Dockerが必要です。事前にDockerをインストールしておいてください。
また、Macの場合にDockerを起動しているのにエラーが出る場合は、以下のコマンドを実行する。
```shell
sudo ln -s ~/.docker/run/docker.sock /var/run/docker.sock
```

もしくは、Docker Desktopの設定で「Enable default Docker socket」を有効にする設定に変更してください。

https://github.com/lando/lando/issues/3533
:::

テストが成功すると、以下のようなログが出力されます。

```shell
Invoking index.HelloWorld (nodejs20.x)                                                                                                                                                                                                             
Local image is up-to-date                                                                                                                                                                                                                          
Using local image: public.ecr.aws/lambda/nodejs:20-rapid-x86_64.                                                                                                                                                                                   
                                                                                                                                                                                                                                                   
Mounting /Users/takapi327/Development/scala-app/feral-example/.aws-sam/build/HelloWorldFunction as /var/task:ro,delegated, inside runtime container                                                                                                
START RequestId: 4f6990f4-e447-4f74-95fe-4fe0b28dd47c Version: $LATEST
got message: [Hello from Lambda!]
END RequestId: bb986cc1-6dee-4cc6-9050-7073d386767d
REPORT RequestId: bb986cc1-6dee-4cc6-9050-7073d386767d  Init Duration: 0.44 ms  Duration: 966.21 ms     Billed Duration: 967 ms Memory Size: 128 MB     Max Memory Used: 128 MB 
{}
```

テストが成功したら、デプロイを行います。

```shell
sam deploy
```

デプロイ準備が完了すると、CloudFormation スタックの変更確認を行い、問題なければスタックの作成/更新を行う。

```shell
Configuring SAM deploy
======================

        Looking for config file [samconfig.toml] :  Found
        Reading default arguments  :  Success

        Setting default arguments for 'sam deploy'
        =========================================
        Stack Name [feral-example]: 
        AWS Region [ap-northeast-1]: 
        #Shows you resources changes to be deployed and require a 'Y' to initiate deploy
        Confirm changes before deploy [Y/n]: Y
        #SAM needs permission to be able to create roles to connect to the resources in your template
        Allow SAM CLI IAM role creation [Y/n]: Y
        #Preserves the state of previously provisioned resources when an operation fails
        Disable rollback [y/N]: N
        Save arguments to configuration file [Y/n]: Y
        SAM configuration file [samconfig.toml]: 
        SAM configuration environment [default]: 

        Looking for resources needed for deployment:
        Creating the required resources...
        Successfully created!

        Managed S3 bucket: aws-sam-cli-managed-default-samclisourcebucket-cqiynjym6997
        A different default S3 bucket can be set in samconfig.toml and auto resolution of buckets turned off by setting resolve_s3=False
                                                                                                                                                                                                                               
        Parameter "stack_name=feral-example" in [default.deploy.parameters] is defined as a global parameter [default.global.parameters].                                                                                      
        This parameter will be only saved under [default.global.parameters] in /Users/takapi327/Development/scala-app/feral-example/samconfig.toml.                                                                            

        Saved arguments to config file
        Running 'sam deploy' for future deployments will use the parameters saved above.
        The above parameters can be changed by modifying samconfig.toml
        Learn more about samconfig.toml syntax at 
        https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-config.html

        Uploading to feral-example/0d2dbd7484c39ac6c6903b307a7acde4  898080 / 898080  (100.00%)

        Deploying with following values
        ===============================
        Stack name                   : feral-example
        Region                       : ap-northeast-1
        Confirm changeset            : True
        Disable rollback             : False
        Deployment s3 bucket         : aws-sam-cli-managed-default-samclisourcebucket-cqiynjym6997
        Capabilities                 : ["CAPABILITY_IAM"]
        Parameter overrides          : {}
        Signing Profiles             : {}

Initiating deployment
=====================

        Uploading to feral-example/4ead5ba28965c83603d8f72db1c5216d.template  865 / 865  (100.00%)


Waiting for changeset to be created..

CloudFormation stack changeset
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Operation                                              LogicalResourceId                                      ResourceType                                           Replacement                                          
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+ Add                                                  HelloWorldFunctionRole                                 AWS::IAM::Role                                         N/A                                                  
+ Add                                                  HelloWorldFunction                                     AWS::Lambda::Function                                  N/A                                                  
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


Changeset created successfully. arn:aws:cloudformation:ap-northeast-1:573320908463:changeSet/samcli-deploy1723288838/e3b907b1-f311-47fa-b078-9505ebe18f54


Previewing CloudFormation changeset before deployment
======================================================
Deploy this changeset? [y/N]: y
```

デプロイが完了すると、Lambda関数が作成されます。

![](/images/scala-feral-lambda-function/lambda-function.png)

テスト実行のイベント JSONを以下のように変更して、テストを実行します。

```json
{
  "message": "Hello from Lambda!"
}
```

テストを実行すると、CloudWatch Logsにログが出力されます。

![](/images/scala-feral-lambda-function/test-result.png)

すみません。こちらご存知だったら教えていただけると嬉しいのですが、Feralを使用した場合と`cats.effect.unsafe.implicits.global`を使用てして`unsafeRunSync`や`unsafeRunAndForget`を使用した場合の違いって何でしょうか？
FeralのIOLambdaを見るとランタイムは同様に`IORuntime.global`を使用していたのでどう違うのかなあと思った感じです！
