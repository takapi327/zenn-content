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

ではどういうユースケースでScala.jsでAWSを利用する必要があるのでしょうか？

今回筆者はScalaを使用してサーバレスアプリケーションを開発する際に使用しました。サーバレスアプリケーションでは、バックエンドの処理をAWS Lambdaで行うことが一般的かと思います。この場合、Lambda関数内でAWSの他のサービスを利用する必要がある場合があります。例えば、Lambda関数内でDynamoDBにアクセスする場合などです。

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
また、AWS公式ではどれぐらいの期間でスリープ状態になるかは公開されていないため、適切な設定が難しいです。

実行速度だけであれば、昨今はAWSのLambda SnapStartやGraalVM Native Imageなどの技術が進化しており、JVMの起動時間を短縮することができるためJVMを使用しても問題ないのかもしれません。

https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/snapstart.html

https://www.graalvm.org/

:::message
筆者はこの2つをちゃんと使用したことがないので、実際の性能差などについてはわかりません。
:::

上記2つは外部？の技術でScalaで書いたコードをいい感じ動かすものなので、今回はScala標準の機能だけでどこまでできるかやScalaでもこんな感じで使えるのかということを知ってもらうためにScala.jsでAWS クライアントを使う方法を紹介します。

:::message
今回の構成はScalaをすでに採用しているプロダクトでLambdaだけ別の言語で書きたくないよ〜であったり同じようなモデル定義などを2重管理したくないよ〜という場合に採用を検討すると良いかと思います。
:::

## プロジェクトの作成

## smithy4sのインストール

### smithy4sはどのようにAWS クライアントを生成しているのか

## AWS クライアントの生成

## クライアントの使用

## LocalStackでのテスト

# まとめ
