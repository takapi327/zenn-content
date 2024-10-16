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

これでプロジェクトの準備が整いました。

## ドキュメントの作成

デフォルトではLaika sbtプラグインは`src/docs`をドキュメントディレクトリだと認識するため、`src/docs`ディレクトリを作成します。

```shell
$ mkdir -p src/docs
```

`src/docs`ディレクトリに`index.md`を作成し、以下のように記述します。

```markdown
# Hello, Laika!
```

実際にドキュメントを生成してみましょう。

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

:::message
新しくMarkdownファイルを作成した場合は、一度`sbt laikaSite`を実行してドキュメントを生成する必要があります。
:::

先ほど`src/docs/index.md`に記述した内容がHTML形式に変換されましたが、Laikaはファイルの拡張子によって処理が異なります。

- **マークアップ・ファイル**: 拡張子が`.md`、`.markdown`、または`.rst`のファイルは、ファイル名はそのままに`.html`へ(出力フォーマットに応じて)レンダリングされます。
- **設定ファイル**: 各ディレクトリには、ナビゲーションの順番や章のタイトルなどを指定するためのオプションの`directory.conf`ファイルを含めることができます。
- **テンプレートファイル**: `default.template.<suffix>`という名前で、サフィックスが出力形式 (例えば`.html`) に一致するデフォルトのテンプレートをディレクトリごとに提供することができます。
- **静的ファイル**: CSS、JavaScript、画像など、他のすべてのファイルは、同じディレクトリ構造と同じファイル名でターゲットにコピーされます。

また、Laikaは`src/docs/`ディレクトリに別のディレクトリを作成することで、章分けを行うこともできます。

```shell
$ mkdir -p src/docs/chapter1
$ touch src/docs/chapter1/index.md
```

試しに`src/docs/chapter1/index.md`に以下の内容を記述してみましょう。

```markdown
# はじめに
```

再度`sbt laikaSite`を実行すると、`src/docs/chapter1/index.md`がHTML形式に変換され、`target/docs/site/chapter1/index.html`に出力されます。

```shell
$ sbt laikaSite
```

プレビューサーバーを起動して、`http://localhost:4242/chapter1`にアクセスすることで、`src/docs/chapter1/index.md`の内容を確認することができます。

```shell
$ sbt laikaPreview
```

![](/images/scala-laika-multi-language-support/laika-chapter1.png)

Laikaはディレクトリ構造からサイドナビゲーションを自動生成するため、章分けを行うことでドキュメントのナビゲーションを行いやすくすることができます。

しかし、このサイドナビゲーションはデフォルトでは、ディレクトリ名を大文字にしてそのまま表示し、そこに含まれるコンテンツはドキュメントのタイトルが表示されるだけです。
また、表示される順番もデフォルトではディレクトリ名のアルファベット順になります。

> - **設定ファイル**: 各ディレクトリには、ナビゲーションの順番や章のタイトルなどを指定するためのオプションの`directory.conf`ファイルを含めることができます。

このようなデフォルトの挙動を変更するためには、`directory.conf`ファイルを使用します。

`src/docs/chapter1/directory.conf`を作成し、以下のように記述します。

```conf
laika.title = チャプター1
```

すると、サイドナビゲーションに`チャプター1`と表示されるようになります。

![](/images/scala-laika-multi-language-support/laika-chapter1-title.png)

このように、ディレクトリ構造を活用してドキュメントを章分けし、設定ファイルを使用することでカスタマイズを行いながらドキュメントを作成するこが可能です。

Laikaは他にもテーマを自作したり、バージョニングを行ったりと、さまざまな機能を提供しています。しかし、今回の主題である多言語対応については、筆者の調べた限りではLaikaの公式ドキュメントには記載がなく、おそらくサポートされていない機能のようです。

そこで、ようやくLaikaを使用して多言語対応のドキュメントを作成する方法について説明します。

# 多言語対応のドキュメントを作成

Laikaは拡張性が高くテーマなども自作できるため、おそらく多言語化に対応したテーマなどを自作することで多言語対応のドキュメントを作成することが可能だと思われます。しかし、今回はなるべく自作をせずにLaika標準の機能を使用して多言語対応のドキュメントを作成したいので、以下の方法で多言語対応のドキュメントを作成します。

- 言語ごとにドキュメントを生成する
- Metadataを使用して言語を指定する
- テンプレートを使用して言語を切り替える
- ナビゲーションを言語別に分ける

## 言語ごとにドキュメントを生成する

Laikaはディレクトリを分割することで、章ごとに区切ってドキュメントを生成することができると説明しましたが、この章ごとの区切りを言語ごとに分けることで多言語対応のドキュメントを作成します。

今回は日本語と英語の2言語に対応したドキュメントを作成します。

`src/docs/en`ディレクトリを作成し、`src/docs/en/index.md`に以下の内容を記述します。

```markdown
# Hello, Laika!
```

同様に、`src/docs/ja`ディレクトリを作成し、`src/docs/ja/index.md`に以下の内容を記述します。

```markdown
# こんにちは、Laika!
```

これで準備が整いました。`sbt laikaSite`を実行すると、`src/docs/en/index.md`と`src/docs/ja/index.md`がそれぞれHTML形式に変換され、`target/docs/site/en/index.html`と`target/docs/site/ja/index.html`に出力されます。

```shell
$ sbt laikaSite
```

プレビューサーバーを起動して、`http://localhost:4242/en`と`http://localhost:4242/ja`にアクセスすることで、それぞれの言語のドキュメントを確認することができます。

```shell
$ sbt laikaPreview
```

![](/images/scala-laika-multi-language-support/laika-en-ja.png)

言語ごとにディレクトリを分けることで、それぞれの言語ごとにドキュメントを作成することができました。

しかし、現状ではサイドナビゲーションに`EN`と`JA`と表示されてしまい、ドキュメントの数が増えると対応する言語の数だけサイドナビゲーションが2倍3倍と増えていってしまいます。

言語は最初に選択して、その後はその言語のドキュメントのみを表示するようにしたいですよね。次に、Metadataを使用して言語をそれぞれ指定する方法について説明します。

## Metadataを使用して言語を指定する

LaikaはMarkdownファイルのメタデータを使用して、ドキュメントの設定を行うことができます。メタデータは特殊な(`{%...%}`で囲まれた)形式で記述し、ファイルの先頭に記述します。

`src/docs/en/index.md`に以下のように記述します。

```markdown
{%
laika.metadata.language = en
%}

# Hello, Laika!
```

同様に、`src/docs/ja/index.md`に以下のように記述します。

```markdown
{%
laika.metadata.language = ja
%}

# こんにちは、Laika!
```

これでドキュメントごとにメタデータを指定することができました。現時点ではこのメタデータはLaikaによって特別な処理をされるわけではないため、このメタデータを使用して言語を切り替えるための処理を追加する必要があります。

次に、テンプレートを使用して言語を切り替える方法について説明します。

## テンプレートを使用して言語を切り替える

> - **テンプレートファイル**: `default.template.<suffix>`という名前で、サフィックスが出力形式 (例えば`.html`) に一致するデフォルトのテンプレートをディレクトリごとに提供することができます。

Laikaはテンプレートを使用して、ドキュメントのレイアウトをカスタマイズすることができます。テンプレートは出力形式ごとに用意されており、HTML形式の場合は`default.template.html`が使用されます。

`src/docs/default.template.html`を作成し、以下の内容を記述します。

:::details default.template.html
```html
<!DOCTYPE html>
<html lang="${?laika.metadata.language}">

@:include(helium.site.templates.head)

<body>

@:include(helium.site.templates.topNav)

<nav id="sidebar">

  <div class="row">
    @:for(helium.site.topNavigation.phoneLinks)
    ${_}
    @:@
  </div>

  @:navigationTree {
  entries = ${helium.site.mainNavigation.prependLinks} [
  { target = "/", excludeRoot = true }
  ] ${helium.site.mainNavigation.appendLinks}
  }

</nav>

<div id="container">

  @:include(helium.site.templates.pageNav)

  <main class="content">

    ${cursor.currentDocument.content}

    @:include(helium.site.templates.footer)

  </main>

</div>

</body>

</html>
```
:::

これで、`sbt laikaSite`を実行しプレビューサーバーを起動してみてください。先ほどまでと同じ画面が表示されるはずです。

```shell
$ sbt laikaSite
$ sbt laikaPreview
```

しかし、このテンプレートには一箇所だけ言語を変更している箇所があります。
    
```html
<html lang="${?laika.metadata.language}">
```

この`${?laika.metadata.language}`はLaikaのメタデータを参照して言語を取得しています。このようにしてメタデータを使用することで、言語を切り替えることができます。

実際に、`http://localhost:4242/en`と`http://localhost:4242/ja`にアクセスし、開発ツールを開きHTMLを確認すると、`<html lang="en">`と`<html lang="ja">`がそれぞれ表示されていることが確認できるはずです。

**英語**

![](/images/scala-laika-multi-language-support/laika-en-html.png)

**日本語**

![](/images/scala-laika-multi-language-support/laika-ja-html.png)

このようにして、Laikaではメタデータの値をテンプレート内で参照することができ、ドキュメントごとにそれぞれ異なった値を設定することができます。

これで言語を切り替えることができました。しかし、現状ではサイドナビゲーションは言語ごとに分かれていないため、次にナビゲーションを言語別に分ける方法について説明します。

## ナビゲーションを言語別に分ける