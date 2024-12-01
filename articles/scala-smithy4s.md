---
title: "Smithy4sを使用したScala.jsでAWS クライアントを使う"
emoji: "♦️"
type: "tech"
topics: ["Scala", "AWS", "Smithy", "Smithy4s", "awslambda"]
publication_name: nextbeat
published: false
---

# Smithy4sとは

Smithy4sは、AWSが開発したAPI定義言語であるSmithyをScalaで扱うためのライブラリです。

https://disneystreaming.github.io/smithy4s/

## Smithyって何？

まず、Smithyについて簡単に説明しましょう。SmithyはAWSが開発したAPI定義言語で、REST APIやgRPCなどのAPIを設計するために使用されます。従来のOpenAPI (Swagger)と同様の目的を持ちますが、より型安全で拡張性の高い設計が可能です。

https://smithy.io/2.0/index.html

例えば、以下のように直感的な構文でAPIを定義できます

```smithy
namespace example.hello

service HelloWorldService {
  version: "1.0.0"
  operations: [SayHello]
}

operation SayHello {
  input: SayHelloInput
  output: SayHelloOutput
}

structure SayHelloInput {
  name: String
}

structure SayHelloOutput {
  message: String
}
```

Smithyに関しては以下の記事なども参考にしてください。

https://dev.classmethod.jp/articles/smihty-document-reading/

https://zenn.dev/jins/articles/6a5ca8c6a67017

https://zenn.dev/seumo/articles/d33581c111a6d7

## Smithy4sの主な特徴

Smithy4sには以下のような特徴があります

1. 型安全性
2. 簡単な統合
3. 開発効率の向上

また、Smithy4sはScala.js, Scala Native, JVMのマルチプラットフォームをサポートしています。

現在Smithy4sは以下の用途に使用できます。

- http/restサーバーとクライアントの生成
- pure-scalaなAWSクライアントの生成
- CLIツールの生成

今回は上記用途の中から、Scala.jsでAWS クライアントを使う方法を紹介します。

# なぜScala.jsでAWS クライアントを使うのか

:::message
少し長くなるので使い方だけ知りたいよ〜って方は「プロジェクトの作成 」まで読み飛ばしてください。
:::

まず、なぜScala.jsでAWS クライアントを使うのか？という疑問があるかもしれません。実際、AWSに対しての処理はScalaであればJava用のSDKが提供されているため、こちらを使用するのが一般的です。

しかし、昨今のScalaはJVMだけでなく、Scala.jsやScala Nativeといったプラットフォームをサポートしており、これらのプラットフォームではJava用のAWS SDKを利用することができません。そのため、Scala.jsやScala NativeでAWSを利用するためには、別の方法を考える必要があります。

Smithy4sは、Smithyを使用してAWSのAPIを定義し、ScalaでAWSクライアントを生成することができます。このため、Scala.jsやScala NativeでもAWSを利用することができます。

:::message
Scala.jsではTypeScriptへのfacadeを用意することでもAWS SDKを利用することができます。
:::

ではどのようなユースケースでScala.jsでAWSを利用する必要があるのでしょうか？

今回筆者はScalaを使用してサーバレスアプリケーションを開発する際に使用しました。サーバレスアプリケーションでは、バックエンドの処理をAWS Lambdaで行うことが一般的かと思います。この場合、Lambda関数内でAWSの他のサービスを利用する必要がある場合があります。例えば、Lambda関数内でDynamoDBにアクセスするなどです。

Scalaを使ったことがあるもしくは知っている人は「LambdaはJarファイルもサポートしてるし別にJVMでもよくないか？」と思うかもしれません。
確かにLambdaはJarファイルをサポートしていますが、Lambda関数のサイズには制限があります。
また、Lambda関数の起動時間も重要な要素です。

サーバレスアーキテクチャであるLambdaを使用してアプリケーションを構築する際、課題となるのは「コールドスタートにおける処理遅延」だと思います。 「JVMの特性とLambdaの特性の相性が悪い」という話もあります。
これは、JVMの場合フットプリントも大きく、またJIT(Just-In-Time)等アプリケーションアーキテクチャの特性上初動が遅くなってしまうためです。

JVMの特徴に対してLambdaの特徴は以下のようなものがあります。

- ワークロードの同時リクエストが増えてきたら Lambda 関数がスケールアウトし、落ち着いてきたらスケールインする。つまり、大量インスタンスが頻繁に起動、停止を繰り返す
- 使用料金は設定したメモリ確保量ｘ処理時間で決まる。つまり、処理時間が同じならメモリ確保料が多ければ多いほど料金が高くなる

このようにJVMはひとたびコードが最適化されると他の言語よりも高速になりますが、初動時からスピードが求められるサーバレスアーキテクチャとの相性はあまり良くないです。

また、Jarファイルを使用した場合Jarファイルはサイズが大きくなりがちです。Lambda関数のサイズには制限があるため、Jarファイルだとすぐにサイズ制限に引っかかってしまいます。

「じゃあDocker Image化してコンテナベースのLambda関数にすればいいんじゃないの？」と思った方もいるでしょう。確かにDocker Image化してコンテナベースのLambda関数にすれば、Lambda関数のサイズ制限に引っかかることはないですが、コンテナベースのLambda関数は一定期間使用していないとスリープ状態になり、再起動時に初動が遅くなったり呼び出し元でエラーになる可能性があります。

これを回避するためには定期的にLambda関数を呼び出す必要があります。HeartBeat用のイベントを作成して、定期的にLambda関数を呼び出すことでコンテナベースのLambda関数をスリープ状態にさせないようにすることができますが、これもまたコストがかかります。
また、AWS公式ではどのくらいの期間でスリープ状態になるかは公開されていないため適切な設定が難しいです。

このような課題に対して、Node.js環境でLambda関数を実行する方法を検討してみましょう。Node.jsには以下のようなメリットがあります：

- 軽量な実行環境
    - Node.jsはV8エンジンベースの軽量な実行環境で、JVMと比較して起動が高速です
    - メモリフットプリントが小さく、Lambda実行時のコストを抑えることができます
    - コールドスタート時の遅延が少なく、サーバーレスアーキテクチャとの相性が良好です

- 小さなデプロイメントパッケージ
    - JVMベースのJARファイルは、ランタイムライブラリや依存関係を含めると数十MBから数百MB規模になることがあります
    - 一方、Node.jsのデプロイメントパッケージは通常数MB程度に収まります
        - 例えば、基本的なAWS SDKの操作を含むLambda関数の場合：
            - JARファイル: 約50-100MB
            - Node.js: 約2-5MB
    - Lambdaの関数サイズ制限（250MB）を考慮すると、Node.jsの方が余裕を持ってデプロイが可能です

少し古いですが、Datadogの記事でもLambdaで最も使われているランタイムは`Python`で、続いて`Node`とこの2つがほとんどの割合を占めています。

このようにサーバーレスアーキテクチャを採用する際には、Nodeのような軽量な実行環境を選択することが一般的です。

https://www.datadoghq.com/ja/state-of-serverless-2021/

:::message
この調査はDatadogを使用しているユーザーのデータを元にしているため、全体のデータとは異なる可能性があります。
:::

では、なぜここでScala.jsを採用するのでしょうか？

Scala.jsを使用することで、Node.jsの優れた実行環境とScalaの強力な型システムや関数型プログラミングの利点を組み合わせることができるからです。

- 型安全性の確保
    - Scalaの強力な型システムにより、実行時エラーを防ぎ、コードの品質を向上させることができます
    - TypeScriptと比較してより強力な型推論と型チェックが可能です

- コード共有の実現
    - フロントエンド（Scala.js）とバックエンド（Lambda/Scala.js）で同じScalaコードを共有できます
    - モデルやバリデーションロジックを共通化し、保守性を高めることができます

- 開発効率の向上
    - Scalaの表現力豊かな構文と関数型プログラミングの特徴を活かした開発が可能です
    - コンパイル時の型チェックにより、バグの早期発見が可能です

このように、Node.js環境の軽量さとScalaの型安全性を組み合わせることでサーバーレスアプリケーションの課題を解決しつつ、高品質なコードベースを維持することができます。特にAPIサーバー等のバックエンドをScalaで開発している場合は、コードの共有や型の一貫性の面で大きなメリットが得られます。

:::message
一応実行速度だけであれば昨今はAWSのLambda SnapStartやGraalVM Native Imageなどの技術が進化しており、JVMの起動時間を短縮することができるためJVMを使用しても問題ないのかもしれません。

筆者はこの2つをちゃんと使用したことがないので、実際の性能差などについてはわかりません。
:::

https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/snapstart.html

https://www.graalvm.org/

上記2つは外部？の技術でScalaで書いたコードをいい感じに動かすものなので、今回はScala標準の機能だけでどこまでできるかやScalaでもこんな感じで使えるのかということを知ってもらうためにScala.jsでAWS クライアントを使う方法を紹介します。

:::message
今回の構成はScalaをすでに採用しているプロダクトでLambdaだけ別の言語で書きたくないよ〜であったり同じようなモデル定義などを2重管理したくないよ〜という場合に採用を検討すると良いかと思います。
:::

# プロジェクトの作成

Scala.jsを使用してAWS Lambdaを作成するのですが、ScalaにはFeralというCats Effectを使ってScalaでサーバーレス関数を書き、FaaS(Function as a Service)基盤に適合させる便利なライブラリがあります。

今回はこのFeralを使用してScala.jsでAWS Lambdaを作成します。また、作成したアプリケーションをAWS Lambdaにデプロイするにも設定が必要です。今回はAWS SAMを使用してデプロイします。

FeralとAWS SAMについては以下の記事を参考にしてプロジェクトの作成を行ってください。

https://zenn.dev/nextbeat/articles/scala-feral-lambda-function#feral%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%97%E3%81%9Flambda%E9%96%A2%E6%95%B0%E3%81%AE%E4%BD%9C%E6%88%90

:::message
今回は以下環境でプロジェクトを作成しています。

- Scala: 3.5.2
- sbt: 1.10.4
- Feral: 0.3.1
- Smithy4s: 0.18.25
- Node.js: 20.x

sbtプロジェクトは任意のものを使用・作成してください。

以下のコマンドを使用すると、Scalaのシードプロジェクトを作成できます。

```shell
$ sbt new scala/scala3.g8
```
:::

これでプロジェクトの準備ができました。

今回作成したプロジェクトは以下で公開しておりますので参考にしてください。

https://github.com/takapi327/smithy4s-sandbox

## Smithy4sの設定

プロジェクトの準備ができたら次は、Smithy4sの[公式ドキュメント](https://disneystreaming.github.io/smithy4s/docs/overview/quickstart/)を参考にSmithy4sをプロジェクトで使えるように設定を追加します。

sbtプロジェクトの`project/plugins.sbt`にSmithy4sのsbtプラグインを追加します。

```scala
addSbtPlugin("com.disneystreaming.smithy4s" % "smithy4s-sbt-codegen" % "0.18.25")
```

次に、プロジェクトの`build.sbt`でプラグインを有効にします。

```scala
enablePlugins(Smithy4sCodegenPlugin)
```

これでSmithy4sの設定が完了しました。

次に、Smithy4sでAWS SDKを使用するための設定を行います。

Smithy4sを使用したAWS SDKの公式ドキュメントは以下を参照してください。

https://disneystreaming.github.io/smithy4s/docs/protocols/aws/aws

Smithy4sはAWS SDKを生成する際に内部で`http4s`のクライアントを使用しているため、AWS SDKを使用する際には`http4s`も依存関係に追加する必要があります。

```scala
libraryDependencies ++= Seq(
  "com.disneystreaming.smithy4s" %%% "smithy4s-aws-http4s" % smithy4sVersion.value,
  "org.http4s" %%% "http4s-ember-client" % "0.23.29"
)
```

:::message
`smithy4s-aws-http4s`は`http4s`のクライアントを使用してAWS SDK用のクライアントを生成するためのライブラリです。
`http4s-ember-client`は[http4s](https://http4s.org/)のクライアントライブラリです。
:::

これでAWS SDKを使用するための設定が完了しました。

ここからは、アプリケーションでどのAWSサービスを使用するかに応じて設定を追加していきます。

例えば、DynamoDBを使用する場合は以下のように設定を追加します。

```scala
smithy4sAwsSpecs ++= Seq(AWS.dynamodb)
// OR
// libraryDependencies ++= Seq(
//   "com.disneystreaming.smithy" % "aws-dynamodb-spec" % "2023.09.22" % Smithy4s
// )
```

上記設定は全然違うように見えますが、実際はどちらも同じことをしているため好きな方を選んでも問題ありません。

::::details 詳細な説明

`smithy4sAwsSpecs`はロードするAWSモジュールのリストを保持しています。

Smithy4sはプラグイン内部でロードするAWSモジュールのリストに基づいて、AWS SDKを生成します。
:::message
モジュールのリストはいくつかあって、`smithy4sAwsSpecs`はそのうちの1つです。
:::

`libraryDependencies`に`% Smithy4s`を追加しているものは、プラグイン内部でこのキーが付与されているものをコンパイル時にフィルタリングしてモジュールのリストに追加しています。

一方で、`AWS.dynamodb`は直接モジュールのリストに追加しています。

この`AWS`オブジェクトはSmithy4sのcodegenパッケージ内で自動的に生成されたものであり、実態は以下のようになっています。

```scala
object AwsSpecs {
  val org = "com.disneystreaming.smithy"
  val knownVersion = "2023.09.22"
  ...
  val dynamodb = "aws-dynamodb-spec"
}
```

:::message
AWSオブジェクトはAwsSpecsのエイリアスです。
プラグインの設定を簡潔にするために、`autoImport`を使用してAWSオブジェクトを定義しています。
```scala
object autoImport {
  val AWS = smithy4s.codegen.AwsSpecs
}
```

また、`AwsSpecs`は自動生成されたものであるためSmithy4sのGithubを確認しても見つけることはできません。実装内容を知りたい場合は、smithy4s.codegenプロジェクトの`target`配下に生成されたソースコードを確認する必要があります。
:::

そして、プラグイン内部で以下のように`libraryDependencies`に追加していたものと同じ形式でモジュールのリストに追加しています。

```scala
smithy4sAwsSpecsVersion := smithy4s.codegen.AwsSpecs.knownVersion,
Compile / smithy4sAwsSpecDependencies := {
  val version = (smithy4sAwsSpecsVersion).value
  (smithy4sAwsSpecs).value.map { case artifactName =>
    smithy4s.codegen.AwsSpecs.org % artifactName % version
    // "com.disneystreaming.smithy" % "aws-dynamodb-spec" % "2023.09.22" こうなっているということ
  }
}
```

このように、`smithy4sAwsSpecs`に直接モジュールを追加する場合も、`libraryDependencies`に追加する場合も最終的にはSmithy4sがコード生成するために使用する配列の中に同じ形式で追加されるため、どちらを使用しても問題ないということです。
::::

Smithy4sで使用できるAWSサービスの一覧は以下を参照してください。

https://disneystreaming.github.io/smithy4s/docs/protocols/aws/aws#service-summary

:::message
[公式](https://disneystreaming.github.io/smithy4s/docs/protocols/aws/aws#what-is-missing-)にも記載されていますが、Smithy4sでAWS SDKを使用する際は以下に注意してください。

- ストリーミング操作（S3のputObject、getObject、KinesisのsubscribeToShardなど）は現在サポートされていない
- サービス固有のカスタマイズは現在サポートされていない
- AWS S3にデータを入出力するためにSmithy4sを使用すべきではない

S3をなぜ使用してはいけないのについて筆者は詳しくないため、もしわかる方がいれば教えていただけると幸いです。
:::

## AWS クライアントの生成

Smithy4sの設定が完了したら、AWS クライアントを生成します。今回はDynamoDBを使用してレコードの作成と取得を行うサンプルを作成します。

まずは、DynamoDBのAWS SDKを生成できるように以下設定を追加します。

```scala
smithy4sAwsSpecs ++= Seq(AWS.dynamodb)
```

追加したら、以下のコマンドを実行してAWS SDKを生成します。

```shell
$ sbt compile
```

コンパイルを実行すると以下ディレクトリにAWS SDKが生成されます。

```
target/scala-x.x.x/src_managed/main/smithy4s/com.amazonaws.dynamodb
```

生成されたパッケージを使用することで、DynamoDBにアクセスすることができます。

```scala
import com.amazonaws.dynamodb.*
```

Smithy4sでAWS SDKを使用する際は、以下のようにAWS クライアントを生成する必要があります。

`AwsEnvironment`はリージョンや、認証情報を保持するためのクラスです。

`AwsClient`は引数で受け取ったサービスに対して操作を行うクラスを生成します。

```scala 3
for
  httpClient <- EmberClientBuilder.default[IO].build
  awsEnv <- AwsEnvironment.default(httpClient, AwsRegion.AP_NORTHEAST_1)
  dynamodb <- AwsClient(DynamoDB.service, awsEnv)
yield dynamodb
```

これらは`cats.effect`の`Resource`を使用しているため、リソースの解放を自動的に行うことができます。

今回は`Feral`を使用しているため、以下のように`IOLambda.Simple`を継承してLambda関数を作成し、初期化処理でDynamoDBのクライアントを生成します。

```scala 3
import cats.effect.*
import fs2.io.compression.*
import feral.lambda.*
import org.http4s.ember.client.EmberClientBuilder
import smithy4s.aws.*
import com.amazonaws.dynamodb.*

object Handler extends IOLambda.Simple[Unit, String]:

  override type Init = DynamoDB[IO]

  override def init: Resource[IO, Init] =
    for
      httpClient <- EmberClientBuilder.default[IO].build
      awsEnv <- AwsEnvironment.default(httpClient, AwsRegion.AP_NORTHEAST_1)
      dynamodb <- AwsClient(DynamoDB.service, awsEnv)
    yield dynamodb

  override def apply(event: Unit, context: Context[IO], init: Init): IO[Option[String]] =
    ??? // DynamoDBのレコードを作成する処理を記述
```

## クライアントの使用

AWS クライアントを生成したら、DynamoDBにレコードを作成する処理を記述します。

```scala 3
override def apply(event: Unit, context: Context[IO], init: Init): IO[Option[String]] =
  init.putItem(
    TableName("Smithy4sSandboxTable"),
    Map(
      AttributeName("id") -> AttributeValue.s(StringAttributeValue("1")),
      AttributeName("name") -> AttributeValue.s(StringAttributeValue("Alice"))
    )
  ).as(Some("Success"))
```

上記のコードは、`Smithy4sSandboxTable`テーブルの`id`と`name`フィールドにレコードを作成する処理です。

このように、AWS クライアントを使用することで、DynamoDBに対して操作を行うことができます。

試しにJavaのAWS SDKを使用したコードと比較してみましょう。

```scala 3
private val dynamodb = AmazonDynamoDBClientBuilder.defaultClient()

dynamodb.putItem(
  "Smithy4sSandboxTable",
  Map(
    "id" -> new AttributeValue("2"),
    "name" -> new AttributeValue("Bob")
  ).asJava
)
```

どうでしょうか？ほぼ同じですよね？これはSmithy4sがAWS SDKを生成する際に、Smithyの定義に基づいて生成しているためです。

Smithy4sはここから単にString型を受け取るのではなく、[Newtype](https://github.com/disneystreaming/smithy4s/blob/series/0.18/modules/core/src-2/Newtype.scala)という独自の型を使用しているためより型安全性を保ちながらAWS SDKを使用することができます。

上記の設定を追加したら、以下のコマンドを実行してScalaで作成したLambda関数をjsファイルとしてビルドします。

```shell
$ sbt npmPackage
```

すると`target/scala-(Scalaバージョン)/npm-package`にファイルが生成されます。

```
.
├── functions
│   └── insert-dynamodb
│       ├── target
│       │   └── scala-3.5.2
│       │       ├── npm-package
│       │       │   ├── index.js
│       │       │   └── package.json
```

このファイルをAWS SAMを使ってデプロイします。

## AWSへのデプロイ

AWS Lambdaにデプロイするためには、AWS SAMを使用します。

AWS SAMを使用して`DynamoDB`のリソースを作成するためには、以下のように`Type`を`AWS::Serverless::SimpleTable`に設定して`TableName`を指定することで作成できます。

```yaml
  Smithy4sSandboxTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      TableName: Smithy4sSandboxTable
```

`DynamoDB`はこの設定だけでリソースを作成できます。

次は上記`DynamoDB`にレコードを挿入するLambda関数を作成します。
作成する`DynamoDB`にはLambda関数からアクセスするため、作成した`DynamoDB`のテーブルにアクセスできるようにLambda関数にポリシーを設定する必要があります。

```yaml
  InsertDynamoDB:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: functions/insert-dynamodb/target/scala-3.5.2/npm-package/
      Handler: index.Handler
      Runtime: nodejs20.x
      Architectures:
        - x86_64
      Policies:
        DynamoDBCrudPolicy:
          TableName: !Ref Smithy4sSandboxTable
```

::::details template.yamlの全体

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  smithy4s-sandbox

  Sample SAM Template for smithy4s-sandbox
  
# More info about Globals: https://github.com/awslabs/serverless-application-model/blob/master/docs/globals.rst
Globals:
  Function:
    Timeout: 3

Resources:
  InsertDynamoDB:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: functions/insert-dynamodb/target/scala-3.5.2/npm-package/
      Handler: index.Handler
      Runtime: nodejs20.x
      Architectures:
        - x86_64
      Policies:
        DynamoDBCrudPolicy:
          TableName: !Ref Smithy4sSandboxTable
  Smithy4sSandboxTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      TableName: Smithy4sSandboxTable
```
::::

今回はCodeUriにLambda関数のコードが格納されているディレクトリを指定してデプロイを行います。Feralによって生成された`functions/insert-dynamodb/target/scala-3.5.2/npm-package/`を指定しています。ここは各自のプロジェクトに合わせて変更してください。

設定が完了したら、以下のコマンドを実行してデプロイします。

AWSへデプロイする際には認証情報が必要です。認証情報の設定は各自の好きな方法で設定してください。

```shell
$ sam deploy
```

デプロイが完了したら、AWS環境のDynamoDBにテーブルが作成されているはずなのでAWS CLIを使用してレコードを取得してみましょう。

```shell
aws dynamodb scan --table-name Smithy4sSandboxTable
{
    "Items": [],
    "Count": 0,
    "ScannedCount": 0,
    "ConsumedCapacity": null
}
```

::::details AWS コンソールでの確認
AWSコンソールでDynamoDBのテーブルが作成されていることを確認できます。
![AWS DynamoDB](/images/scala-smithy4s/aws-dynamodb.png)

:::message
AWS コンソールの画面は2024年12月時点のものです。画面が変更されている可能性があるため、最新の画面を確認してください。
:::
::::

Lambda関数も同様にAWS CLIを使用して確認できます。

```shell
aws lambda list-functions
{
    "Functions": [
        {
            "FunctionName": "smithy4s-sandbox-InsertDynamoDB-ITah9d1K2ULS",
            "FunctionArn": "arn:aws:lambda:ap-northeast-1:xxxxxxxxxxxx:function:smithy4s-sandbox-InsertDynamoDB-ITah9d1K2ULS",
            "Runtime": "nodejs20.x",
            "Role": "arn:aws:iam::xxxxxxxxxxxx:role/smithy4s-sandbox-InsertDynamoDBRole-6BkmoajkQy4O",
            "Handler": "index.Handler",
            "CodeSize": 4061116,
            "Description": "",
            "Timeout": 3,
            "MemorySize": 128,
```

### Lambda関数の実行

Lambda関数を実行するには、以下のコマンドを使用します。

```shell
aws lambda invoke --function-name smithy4s-sandbox-InsertDynamoDB-ITah9d1K2ULS --payload '{}' output.json
```

::::details AWSコンソールでの実行
AWSコンソールからもLambda関数を実行することができます。

以下画像のように空のJSONを入力してテスト実行してみましょう。

![AWS Lambda](/images/scala-smithy4s/aws-lambda.png)

:::message
AWS コンソールの画面は2024年12月時点のものです。画面が変更されている可能性があるため、最新の画面を確認してください。
:::
::::

Lambda関数を実行すると以下のようなレスポンスが帰ってきます。

```json
{
  "StatusCode": 200,
  "ExecutedVersion": "$LATEST"
}
```

ステータスコードが200で返ってきているため、正常に実行されていることがわかります。

DynamoDBのテーブルを確認すると、レコードが追加されていることが確認できます。

```shell
aws dynamodb scan --table-name Smithy4sSandboxTable
{
    "Items": [
        {
            "id": {
                "S": "1"
            },
            "name": {
                "S": "Alice"
            }
        }
    ],
    "Count": 1,
    "ScannedCount": 1,
    "ConsumedCapacity": null
}
```

これでScala.jsでAWS クライアントを使用してDynamoDBにレコードを追加するLambda関数を作成し、実際に動作確認を行うことができました。

## Jarファイルとの比較

Smithy4sとScala.jsを使用することでNode環境でAWS SDKを使用することができました。ここでJarファイルとの比較を行ってみましょう。

![Jarファイルとの比較](/images/scala-smithy4s/jar-vs-node.png)

上記画像からJarファイルは単純な処理であっても**32.6MB**となっており、Node.jsの**3.9MB**の**約10倍**のサイズとなっています。

:::message
メモリサイズはJavaの環境だとデフォルトの128MBでは足らないためNode, Javaともに1024MBに設定しています。
Node.jsの場合はデフォルトの128MBでも十分な場合が多いです。
:::

また、実行時間についても以下のようになりました。

**Java 実行時間**

![Java 実行時間](/images/scala-smithy4s/java-execution-time.png)

**Node 実行時間**

![Node 実行時間](/images/scala-smithy4s/node-execution-time.png)

上記画像からもわかるように、Node.jsの方がJavaよりも実行時間が短くなっていることがわかります。

:::message
今回は両方とも初回実行のみ計測しています。Lambda関数を連続で実行する場合や定期的に実行するような場合ではこの結果は変わる可能性があります。

初回実行が遅くなることは重々承知しておりますが、サーバレスアプリケーションを採用する場合は初回実行が重要な場合が多いためこの結果を参考にしていただければと思います。
:::

このようにNode.jsの方がJarファイルよりも軽量で実行時間も短いため、サーバーレスアプリケーションを開発する際にはNode.jsを使用するのが適していると言えます。

Scala.jsを使用することで、Node.jsの軽量さとScalaの型安全性を組み合わせることができるため、サーバーレスアプリケーションの開発においても有用であると言えるのではないでしょうか。

この計測で実際に使用したJarファイルを生成しているコードは以下に置いてあります。

https://github.com/takapi327/smithy4s-sandbox/blob/master/functions/insert-dynamodb-jar/src/main/scala/Handler.scala

:::message
環境はScala.jsを使用したものと同じで、Jarファイルは[sbt assembly](https://github.com/sbt/sbt-assembly)を使用して生成しています。
:::

## (おまけ) LocalStackでのテスト

AWS クライアントを使用する際には実際のAWSアカウントを使用することもできますが、テスト時やローカル環境での開発時にはLocalStackを使用することをおすすめします。

LocalStackはAWSのローカルエミュレータで、AWSの主要なサービスをローカルで実行することができます。

https://www.localstack.cloud/

LocalStackはDockerコンテナが提供されているため、Dockerを使用して構築しておきましょう。

::::details LocalStackの構築

```yaml
services:
  localstack:
    container_name: localstack
    platform: ${DOCKER_PLATFORM:-linux/amd64}
    image: localstack/localstack:latest
    ports:
      - "127.0.0.1:4566:4566"
      - "127.0.0.1:4510-4559:4510-4559"
    environment:
      - DEFAULT_REGION=ap-northeast-1
      - DEBUG=${DEBUG:-0}
      - LAMBDA_EXECUTOR=${LAMBDA_EXECUTOR:-docker}
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    extra_hosts:
      - "host.docker.internal:host-gateway"
```
::::

LocalStackを使用することで実際のAWSアカウントを使用せずに、ローカル環境でAWSサービスをテストすることができます。

Smithy4sはLocalStackをサポートしているため、LocalStackを使用してAWS クライアントをテストすることができます。

https://disneystreaming.github.io/smithy4s/docs/protocols/aws/localstack

Smithy4sで構築したAWSクライアントをローカル環境に適用するには、リクエストをLocalstackのホストとポートにリダイレクトするミドルウェア`(Client[F] => Client[F]関数)`を作成する必要があります。

ミドルウェアの例は以下となります。

:::message
ホストはLambdaをどこで実行するかによって変わります。以下参考にしていただければと思います。

| 実行場所             | ホスト                  | 備考                                   |
|------------------|----------------------|--------------------------------------|
| ローカル             | host.docker.internal | `samlocal local invoke`コマンドなどを使用する場合 |
| docker container | localstack           | `SQS`や`SNS`などを経由して実行する場合             |
:::

```scala 3
import org.typelevel.ci.*
import fs2.compression.Compression
import cats.effect.*
import org.http4s.*
import org.http4s.client.Client

object LocalstackProxy:
  def apply[F[_]: Async: Compression](client: Client[F]): Client[F] = Client { req =>
    val request = req
      .withUri(
        req.uri.copy(
          scheme = Some(Uri.Scheme.http),
          authority = req.uri.authority.map(
            _.copy(
              host = Uri.RegName("host.docker.internal"),
              port = Some(4566)
            )
          )
        )
      )
      .putHeaders(Header.Raw(ci"host", "host.docker.internal"))

    client.run(request)
  }
```

:::message alert

一点注意が必要な点があります。現時点(2024年11月時点)では特定のサービスを使用する際にリクエストヘッダーから`X-Amz-Target`を削除しないと正常に動作しない場合があります。

筆者が遭遇したのはSQSを使用する際に`X-Amz-Target`を削除しないとレスポンス型をXML形式で期待しているのにJSONで帰ってきてしまいパースを行うことができずエラーになるというものでした。

この情報がSmithy4s公式の[Discord](https://discord.gg/3SERA46Z)上でしか見つけられなかったため、もし同じような問題に遭遇した場合は以下のようにリクエストヘッダーから`X-Amz-Target`を削除してリクエストを送信してみてください。

他のサービスではこのヘッダー情報を削除すると逆にエラーになる場合もあるため、対象のサービスに絞って対応するのが良いかと思います。

```scala 3
val isSQS = req.headers.headers
  .find(_.name == ci"X-Amz-Target")
  .exists(_.value.contains("AmazonSQS"))

val base = req
  .withUri(...)
  ...

val request =
  if isSQS then base.removeHeader(ci"X-Amz-Target")
  else base
  
client.run(request)
```
:::

次にこのミドルウェアを使用したAWS クライアントを生成します。

`AwsEnvironment`では、LocalStackで使用するクレデンシャル情報を設定します。

そして受け取る`Client[F]`に対してLocalStackのホストとポートにリダイレクトするミドルウェアを適用することで、LocalStackでAWS クライアントを使用することができます。

```scala 3
import cats.syntax.all.*
import cats.effect.*
import fs2.compression.Compression
import org.http4s.client.Client
import smithy4s.*
import smithy4s.aws.*

object LocalstackClient:

  private def env[F[_]: Async: Compression](
    client: Client[F],
    region: AwsRegion
  ): AwsEnvironment[F] =
    AwsEnvironment.make[F](
      client,
      Async[F].pure(region),
      Async[F].pure(AwsCredentials.Default("dummy", "dummy", None)),
      Async[F].realTime.map(_.toSeconds).map(Timestamp(_, 0))
    )

  def apply[F[_]: Async: Compression, Alg[_[_, _, _, _, _]]](
    client:  Client[F],
    region:  AwsRegion,
    service: Service[Alg]
  ): Resource[F, service.Impl[F]] =
    AwsClient(service, env[F](LocalstackProxy[F](client), region))
```

公式のサンプルコードでは特定のサービス専用にクライアントを作成していましたが、今回は汎用的なクライアントを作成するためにSmithy4sの`Service`を型パラメーター(`Alg`)を指定して受け取るようにしました。

これで使用したいサービスに対してクライアントを生成できます。

先ほど作成した`DynamoDB`用のクライアントを作成した`LocalstackClient`を使用したコードに書き換えておきましょう。

```scala 3
  override def init: Resource[IO, Init] =
    for
      httpClient <- EmberClientBuilder.default[IO].build
      dynamodb <- LocalstackClient(httpClient, AwsRegion.AP_NORTHEAST_1, DynamoDB.service)
    yield dynamodb
```

これでSmithy4sを使用して`LocalStack`でAWS クライアントを検証するための設定が完了しました。

AWS SAMと`LocalStack`を使用する場合は[aws-sam-cli-local](https://github.com/localstack/aws-sam-cli-local)を使用することで`samlocal deploy`を実行するだけで`localstack`へデプロイができるため今回はこちらを使用します。

`aws-sam-cli-local`を使用するためにはPythonの環境が必要なので以下記事などを参考に用意しておきましょう。

https://qiita.com/y-tsutsu/items/54c10e0b2c6b565c887a

Localstack上のリソース確認などは`awscli-local`を使用するので合わせてインストールをしておきます。

```shell
$ pipenv install aws-sam-cli-local
$ pipenv install awscli-local
```

```shell
# samlocalコマンドを利用するので、pipenvのshellに入る
$ pipenv shell

# localstackにデプロイ
$ samlocal deploy
```

`LocalStack`上に`DynamoDB`のリソースが作成されたらまずは、以下コマンドを実行して`DynamoDB`のテーブルにレコードが存在していないことを確認しておきます。

```shell
$ awslocal dynamodb scan --table-name Smithy4sSandboxTable
{
    "Items": [],
    "Count": 0,
    "ScannedCount": 0,
    "ConsumedCapacity": null
}
```

次に`LocalStack`のホストとポートにリダイレクトするミドルウェアを適用したAWS クライアントを使用して、`DynamoDB`にレコードを作成する処理を実行してみましょう。

```shell
$ samlocal local invoke InsertDynamoDB
```

コマンドを実行すると以下のようにLambdaの処理で返却していた`Success`が表示されるためレコードが作成されたことが確認できます。

```shell
START RequestId: 87926c3f-b3be-426b-a4b8-61eb99ad7ecf Version: $LATEST
END RequestId: d0573e25-cdfa-4f53-94df-2c2c507e515b
REPORT RequestId: d0573e25-cdfa-4f53-94df-2c2c507e515b  Init Duration: 0.28 ms  Duration: 2327.89 ms    Billed Duration: 2328 ms        Memory Size: 128 MB     Max Memory Used: 128 MB 
"Success"
```

先ほど実行したコマンドを再度実行して`DynamoDB`のテーブルにレコードが作成されたことを確認してみましょう。

以下のように作成したLambda関数内でputした内容と同じレコードが作成されていることが確認できます。

```shell
$ awslocal dynamodb scan --table-name Smithy4sSandboxTable
{
  "Items": [
    {
      "name": {
        "S": "Alice"
      },
      "id": {
        "S": "1"
      }
    }
  ],
  "Count": 1,
  "ScannedCount": 1,
  "ConsumedCapacity": null
}
```

これでLocalStackを使用してもSmithy4sを使用したAWS クライアントをテストすることができました。

# まとめ

今回はScala.jsとSmithy4sを使用してAWS SDKを生成し、DynamoDBにレコードを追加するLambda関数を作成しました。

最近JVMだけではなくScala JSやScala Nativeに対応しているライブラリが増えてきていると感じており、Scala.jsを使用してLambda関数を使用したサーバレスアプリケーションを開発しやすくなってきていると感じていました。しかし、Lambda関数を使用する際にはAWS SDKを使用することが多いため、その他のライブラリが使えたとしてもAWS SDKが使えないと採用するのは難しいかなとも感じていました。
Scala.jsではJS用のAWS SDKを使用してfacadeを用意することもできますが、facadeを作成するのは手間がかかるためあまり積極的には使用していませんでした。

そんな中今回のSmithy4sというライブラリをTypelevelのコミュニティで知り、このライブラリがAWS SDKを生成することができるということであったため実際に触ってみることにしました。
初めはSmithy定義書からコードを生成しているということで結構イマイチなのでは？と思っていましたが、実際に触ってみると本家？のJava AWS SDKとほとんど同じようなコードが生成されておりScala.jsで使用することもできるということでかなり使いやすいと感じました。

LocalStackを使用してローカル環境でも簡単に動作させられる点も個人的に良かったです。また、AWS SDKのSmithy定義からコードを生成しているためJavaのAWS SDKと比較してもほぼ同じコードなっており、これであればAPIサーバーのようなバックエンドアプリケーションでも共通してSmithy4sのAWS SDKを使用することができるのではないかと感じました。

今現状で難点があるとすればよくわからないエラー(ブログ内でも書いた)がたまにあるということと、Smithy4側ではAWS SDKのコードを事前に生成していないため使用側でコード生成が発生しコンパイル時間が長くなってしまうという点があります。

まあ、この問題自体は事前にコンパイルしておいたプロジェクトを公開しておくなりすれば解消できる問題なのでそこまで気にする必要はないかなとも思いました。

まだまだSmithy4sは開発途中であるため今後も機能追加やバグ修正が行われると思います。筆者はScalaが好きでできるならなんでもScalaで書きたいマンなので、Scala.jsやJVMでAWS SDKを使用できるこのライブラリを今後も使っていこうかなと思っています。

長くなりましたが、最後まで読んでいただきありがとうございました。