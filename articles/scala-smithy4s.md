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

## プロジェクトの作成

## smithy4sのインストール

### smithy4sはどのようにAWS クライアントを生成しているのか

## AWS クライアントの生成

## クライアントの使用

## LocalStackでのテスト

# まとめ
