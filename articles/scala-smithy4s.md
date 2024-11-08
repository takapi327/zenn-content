---
title: "smithy4sを使用したScala.jsでAWS クライアントを使う"
emoji: "♦️"
type: "tech"
topics: ["Scala", "Typelevel", "AWS", "Smithy", "smithy4s"]
publication_name: nextbeat
published: false
---

# smithy4sとは

smithy4sは、AWSが開発したAPI定義言語であるSmithyをScalaで扱うためのライブラリです。

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

## smithy4sの主な特徴

smithy4sには以下のような特徴があります

1. 型安全性
2. 簡単な統合
3. 開発効率の向上

また、smithy4sはScala.js, Scala Native, Scala JVMのマルチプラットフォームをサポートしています。

現在Smithy4sは以下の用途に使用できます。

- http/restサーバーとクライアントの生成
- pure-scalaなAWSクライアントの生成
- CLIツールの生成

今回は上記用途の中から、Scala.jsでAWS クライアントを使う方法を紹介します。

# Scala.jsでAWS クライアントを使う

まず、なぜScala.jsでAWS クライアントを使うのか？という疑問があるかもしれません。実際、AWSに対しての処理はScalaであればJava用のSDKが提供されているため、こちらを使用するのが一般的です。

しかし、昨今のScalaはJVMだけでなく、Scala.jsやScala Nativeといったプラットフォームをサポートしており、これらのプラットフォームではJava用のAWS SDKを利用することができません。そのため、Scala.jsやScala NativeでAWSを利用するためには、別の方法を考える必要があります。

smithy4sは、Smithyを使用してAWSのAPIを定義し、ScalaでAWSクライアントを生成することができます。このため、Scala.jsやScala NativeでもAWSを利用することができます。

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

これを回避するためには、定期的にLambda関数を呼び出す必要があります。HeartBeat用のイベントを作成して、定期的にLambda関数を呼び出すことで、コンテナベースのLambda関数をスリープ状態にさせないようにすることができますが、これもまたコストがかかります。
また、AWS公式ではどのくらいの期間でスリープ状態になるかは公開されていないため、適切な設定が難しいです。

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

少し古いですが、Datadogの記事でもLambdaで最も使われている言語は`Python`で、続いて`Node.js`とこの2つがほとんどの割合を占めています。

このようにサーバーレスアーキテクチャを採用する際には、Nodeのような

https://www.datadoghq.com/ja/state-of-serverless-2021/

:::message
この調査はDatadogを使用しているユーザーのデータを元にしているため、全体のデータとは異なる可能性があります。
:::

では、なぜここでScala.jsを採用するのでしょうか？

Scala.jsを使用することで、Node.jsの優れた実行環境とScalaの強力な型システムや関数型プログラミングの利点を組み合わせることができます：

- 型安全性の確保
    - Scalaの強力な型システムにより、実行時エラーを防ぎ、コードの品質を向上させることができます
    - TypeScriptと比較してより強力な型推論と型チェックが可能です

- コード共有の実現
    - フロントエンド（Scala.js）とバックエンド（Lambda/Scala.js）で同じScalaコードを共有できます
    - モデルやバリデーションロジックを共通化し、保守性を高めることができます

- 開発効率の向上
    - Scalaの表現力豊かな構文と関数型プログラミングの特徴を活かした開発が可能です
    - コンパイル時の型チェックにより、バグの早期発見が可能です

このように、Node.js環境の軽量さとScala.jsの型安全性を組み合わせることで、サーバーレスアプリケーションの課題を解決しつつ、高品質なコードベースを維持することができます。特に、フロントエンドもScala.jsで開発している場合、コードの共有や型の一貫性の面で大きなメリットが得られます。

:::message
一応実行速度だけであれば、昨今はAWSのLambda SnapStartやGraalVM Native Imageなどの技術が進化しており、JVMの起動時間を短縮することができるためJVMを使用しても問題ないのかもしれません。

筆者はこの2つをちゃんと使用したことがないので、実際の性能差などについてはわかりません。
:::

https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/snapstart.html

https://www.graalvm.org/

上記2つは外部？の技術でScalaで書いたコードをいい感じに動かすものなので、今回はScala標準の機能だけでどこまでできるかやScalaでもこんな感じで使えるのかということを知ってもらうためにScala.jsでAWS クライアントを使う方法を紹介します。

:::message
今回の構成はScalaをすでに採用しているプロダクトでLambdaだけ別の言語で書きたくないよ〜であったり同じようなモデル定義などを2重管理したくないよ〜という場合に採用を検討すると良いかと思います。
:::

## プロジェクトの作成

Scala.jsを使用してAWS Lambdaを作成するのですが、ScalaにはFeralというCats Effectを使ってScalaでサーバーレス関数を書き、FaaS(Function as a Service)基盤に適合させる便利なライブラリがあります。

今回はこのFeralを使用してScala.jsでAWS Lambdaを作成します。また、作成したアプリケーションをAWS Lambdaにデプロイするにも設定が必要です。今回はAWS SAMを使用してデプロイします。

FeralとAWS SAMについては以下の記事を参考にしてプロジェクトの作成を行ってください。

https://zenn.dev/nextbeat/articles/scala-feral-lambda-function#feral%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%97%E3%81%9Flambda%E9%96%A2%E6%95%B0%E3%81%AE%E4%BD%9C%E6%88%90

:::message
今回は以下環境でプロジェクトを作成しています。

- Scala: 3.5.2
- sbt: 1.10.4
- Feral: 0.3.1
- smithy4s: 0.18.25
- Node.js: 20.x

sbtプロジェクトは任意のものを使用・作成してください。

以下のコマンドを使用すると、Scalaのシードプロジェクトを作成できます。

```shell
$ sbt new scala/scala3.g8
```
:::

これでプロジェクトの準備ができました。

## smithy4sの設定

プロジェクトの準備ができたら次は、smithy4sの[公式ドキュメント](https://disneystreaming.github.io/smithy4s/docs/overview/quickstart/)を参考にsmithy4sをプロジェクトで使えるように設定を追加します。

sbtプロジェクトの`project/plugins.sbt`にsmithy4sのsbtプラグインを追加します。

```scala
addSbtPlugin("com.disneystreaming.smithy4s" % "smithy4s-sbt-codegen" % "0.18.25")
```

次に、プロジェクトの`build.sbt`でプラグインを有効にします。

```scala
enablePlugins(Smithy4sCodegenPlugin)
```

これでsmithy4sの設定が完了しました。

次に、smithy4sでAWS SDKを使用するための設定を行います。

smithy4sを使用したAWS SDKの公式ドキュメントは以下を参照してください。

https://disneystreaming.github.io/smithy4s/docs/protocols/aws/aws

smithy4sはAWS SDKを生成する際に内部で`http4s`のクライアントを使用しているため、AWS SDKを使用する際には`http4s`を追加する必要があります。

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

smithy4sはプラグイン内部でロードするAWSモジュールのリストに基づいて、AWS SDKを生成します。
:::message
モジュールのリストはいくつかあって、`smithy4sAwsSpecs`はそのうちの1つです。
:::

`libraryDependencies`に`% Smithy4s`を追加しているものは、プラグイン内部でこのキーが付与されているものをコンパイル時にフィルタリングしてモジュールのリストに追加しています。

一方で、`AWS.dynamodb`は直接モジュールのリストに追加しています。

この`AWS`オブジェクトはsmithy4sのcodegenパッケージ内で自動的に生成されたものであり、実態は以下のようになっています。

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

また、`AwsSpecs`は自動生成されたものであるためsmithy4sのGithubを確認しても見つけることはできません。実装内容を知りたい場合は、smithy4s.codegenプロジェクトの`target`配下に生成されたソースコードを確認する必要があります。
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

このように、`smithy4sAwsSpecs`に直接モジュールを追加する場合も、`libraryDependencies`に追加する場合も最終的にはsmithy4sがコード生成するために使用する配列の中に同じ形式で追加されるため、どちらを使用しても問題ないということです。
::::

smithy4sで使用できるAWSサービスの一覧は以下を参照してください。

https://disneystreaming.github.io/smithy4s/docs/protocols/aws/aws#service-summary

:::message
[公式](https://disneystreaming.github.io/smithy4s/docs/protocols/aws/aws#what-is-missing-)にも記載されていますが、smithy4sでAWS SDKを使用する際は以下に注意してください。

- ストリーミング操作（S3のputObject、getObject、KinesisのsubscribeToShardなど）は現在サポートされていない
- サービス固有のカスタマイズは現在サポートされていない
- AWS S3にデータを入出力するためにsmithy4sを使用すべきではない

S3をなぜ使用してはいけないのについて筆者は詳しくないため、もしわかる方がいれば教えていただけると幸いです。
:::

## AWS クライアントの生成

smithy4sの設定が完了したら、AWS クライアントを生成します。今回はDynamoDBを使用してレコードの作成と取得を行うサンプルを作成します。

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

smithy4sでAWS SDKを使用する際は、以下のようにAWS クライアントを生成する必要があります。

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

object Handler extends IOLambda.Simple[Unit, Unit]:

  override type Init = DynamoDB[IO]

  override def init: Resource[IO, Init] =
    for
      httpClient <- EmberClientBuilder.default[IO].build
      awsEnv <- AwsEnvironment.default(httpClient, AwsRegion.AP_NORTHEAST_1)
      dynamodb <- AwsClient(DynamoDB.service, awsEnv)
    yield dynamodb

  override def apply(event: Unit, context: Context[IO], init: Init): IO[Option[Unit]] =
    ??? // DynamoDBのレコードを作成する処理を記述
```

### smithy4sはどのようにAWS クライアントを生成しているのか

## クライアントの使用

AWS クライアントを生成したら、DynamoDBにレコードを作成する処理を記述します。

```scala 3
override def apply(event: Unit, context: Context[IO], init: Init): IO[Option[Unit]] =
  init.putItem(
    TableName("test"),
    Map(
      AttributeName("id") -> AttributeValue.n(NumberAttributeValue("1")),
      AttributeName("name") -> AttributeValue.s(StringAttributeValue("Alice"))
    )
  ).as(None)
```

上記のコードは、`test`テーブルに`id`と`name`のレコードを作成する処理です。

このように、AWS クライアントを使用することで、DynamoDBに対して操作を行うことができます。

## LocalStackでのテスト

AWS クライアントを使用する際には、実際のAWSアカウントを使用することもできますが、テスト時にはLocalStackを使用することをおすすめします。

LocalStackは、AWSのローカルエミュレータで、AWSの主要なサービスをローカルで実行することができます。

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
    configs:
      - source: aws_profile
        target: /root/.aws
    environment:
      - DEFAULT_REGION=ap-northeast-1
      - DEBUG=${DEBUG:-0}
      - LAMBDA_EXECUTOR=${LAMBDA_EXECUTOR:-docker}
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"

configs:
  aws_profile:
    file: ./localstack/profile
```

```shell
mkdir localstack
mkdir localstack/profile
```

```shell
touch localstack/profile/config
```

```text
[profile localstack]
region = ap-northeast-1
output = json
```

```shell
touch localstack/profile/credentials
```

```text
[localstack]
aws_access_key_id = dummy
aws_secret_access_key = dummy
```
::::

LocalStackを使用することで、実際のAWSアカウントを使用せずに、ローカルでAWSサービスをテストすることができます。

smithy4sはLocalStackをサポートしているため、LocalStackを使用してAWS クライアントをテストすることができます。

https://disneystreaming.github.io/smithy4s/docs/protocols/aws/localstack

```scala 3
object LocalstackProxy:
  def apply[F[_]: Async: Compression](client: Client[F]): Client[F] = Client { req =>
    val request = req
      .withUri(
        req.uri.copy(
          scheme = Some(Uri.Scheme.http),
          authority = req.uri.authority.map(
            _.copy(
              host = Uri.RegName("localstack"),
              port = Some(4566)
            )
          )
        )
      )
      .putHeaders(Header.Raw(ci"host", "localstack"))

    client.run(request)
  }
```

:::message alert

一点注意が必要な点があります。現在 (2024年11月時点)では特定のサービスを使用する際にリクエストヘッダーから`X-Amz-Target`を削除しないと正常に動作しない場合があります。

筆者が遭遇したのは、SQSを使用する際に`X-Amz-Target`を削除しないとレスポンス型をXML形式で期待しているのにJSONで帰ってきてしまいパースを行うことができずエラーになるというものでした。

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

### DynamoDBのリソースを作成

# まとめ
