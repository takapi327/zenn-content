---
title: "Laikaを使用したScala製ドキュメントツールの多言語対応"
emoji: "♦️"
type: "tech"
topics: ["Scala", "Typelevel", "Laika", "ドキュメント"]
publication_name: nextbeat
published: false
---

# はじめに

LaikaはScala製のドキュメントツールで、Markdownを使用してドキュメントを記述し、HTMLやPDFなどの形式に変換することができます。Laikaは多言語対応も可能で、この記事ではLaikaを使用して多言語対応したドキュメントを作成する方法について説明します。

## Laikaとは

Laikaは、Scala開発者のための強力で柔軟なドキュメント生成ツールです。軽量マークアップ言語を変換し、サイトや`e-book` (EPUB & PDF) を生成する多機能なツールキットとして設計されています。

https://typelevel.org/Laika/

LaikaはTypelevelプロジェクトの一部であり、TypelevelはさまざまなScalaのライブラリを提供するコミュニティです。

https://typelevel.org/

Laikaには、次のような特徴があります。

**豊富なフォーマットサポート**

- Markdown（GitHub Flavor含む）
- reStructuredText
- HTML、EPUB、PDFなど多彩な出力形式

**高度な機能**

- 内部リンクの自動検証
- ナビゲーションツリーや目次の自動生成
- 多言語シンタックスハイライト

**柔軟なカスタマイズ**

- テーマやテンプレートのカスタマイズ
- 独自のマークアップ言語や出力形式の追加が可能

Laikaは[API](https://typelevel.org/Laika/latest/02-running-laika/02-library-api.html#)も提供しており、ScalaのコードからLaikaを使用してドキュメントを生成することもできます。

```scala 3
import laika.api.*
import laika.format.*

val transformer = Transformer
  .from(Markdown)
  .to(HTML)
  .using(Markdown.GitHubFlavor)
  .build
```

このように、Laikaを使用することで、Scalaのコードからドキュメントを生成することも可能です。

```scala 3
val result = transformer.transform("hello *there*")
// result: Either[errors.TransformationError, String] = Right(
//   "<p>hello <em>there</em></p>"
// )
```

また、Laikaはsbtプラグインも提供しており、sbtプロジェクトからLaikaを使用してドキュメントを生成することもできます。

今回はこのsbtプラグインを使用して、Laikaを使った多言語対応のドキュメントを作成する方法について説明します。

Laikaの使い方は以下ブログ記事も参考になります。

https://blog.3qe.us/entry/2024/02/15/220718

# プロジェクトの作成

まず、[公式ドキュメント](https://typelevel.org/Laika/latest/02-running-laika/01-sbt-plugin.html#adding-the-plugin-to-the-build)を参考にLaikaを使用して多言語対応のドキュメントを作成するためのプロジェクトを作成します。

:::message
今回は以下環境でプロジェクトを作成しています。

- Scala: 3.5.1
- sbt: 1.10.2
- Laika: 1.2.0
:::

sbtプロジェクトの`project/plugins.sbt`にLaikaのsbtプラグインを追加します。

```scala
addSbtPlugin("org.typelevel" % "laika-sbt" % "1.2.0")
```

次に、プロジェクトのbuild.sbtでプラグインを有効にします。

```scala
enablePlugins(LaikaPlugin)
```

デフォルトではLaika sbtプラグインは`src/docs`をドキュメントディレクトリだと認識するため、`src/docs`ディレクトリを作成します。

```shell
$ mkdir -p src/docs
```

`src/docs`ディレクトリに`index.md`を作成し、以下のように記述します。

```markdown
# Hello, Laika!
```

これでプロジェクトの準備が整いました。実際にドキュメントを生成してみましょう。

```shell
$ sbt laikaSite
```

`target/docs/site`ディレクトリにHTML形式のドキュメントが生成されます。生成されたドキュメントをブラウザで開くことでドキュメントの内容を確認することができますが、Laikaにはプレビューサーバーも用意されています。

```shell
$ sbt laikaPreview
```

このコマンドを実行すると、`http://localhost:4242`でプレビューサーバーが起動します。ブラウザで`http://localhost:4242`にアクセスすることでドキュメントを確認することができます。

![](/images/scala-laika-multi-language-support/laika-preview.png)

このプレビューサーバーは、ドキュメントの変更を検知して自動的に再生成するため、ドキュメントの作成を効率的に行うことができます。

先ほど`src/docs/index.md`に記述した内容がHTML形式に変換されましたが、Laikaはファイルの拡張子によって処理が異なります。

- **マークアップ・ファイル**: 拡張子が`.md`、`.markdown`、または`.rst`のファイルは、ファイル名はそのままに`.html`へ(出力フォーマットに応じて)レンダリングされます。
- **設定ファイル**: 各ディレクトリには、ナビゲーションの順番や章のタイトルなどを指定するためのオプションの`directory.conf`ファイルを含めることができます。
- **テンプレートファイル**: `default.template.<suffix>`という名前で、サフィックスが出力形式 (例えば`.html`) に一致するデフォルトのテンプレートをディレクトリごとに提供することができます。
- **静的ファイル**: CSS、JavaScript、画像など、他のすべてのファイルは、同じディレクトリ構造と同じファイル名でターゲットにコピーされます。
