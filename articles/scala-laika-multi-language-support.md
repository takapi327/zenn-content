---
title: "Laikaを使用したScala製ドキュメントツールの多言語対応"
emoji: "♦️"
type: "tech"
topics: ["Scala", "Typelevel", "Laika", "ドキュメント"]
publication_name: nextbeat
published: true
---

# はじめに

今回は筆者が作成しているOSSプロジェクトのドキュメントをLaikaに書き換えた際に、多言語対応のドキュメントを作成する方法を調査した結果を共有します。

多言語対応と記載していますが、`i18n`などを使用して自動的に多言語対応を行うわけではなく、Laikaの標準機能を使用してドキュメントを言語ごとに作成・管理する方法についての説明となりますので、ご了承ください。

:::message
今回の方法はLaikaが多言語対応の機能を提供しているわけではなく、Laikaの標準機能を使用して多言語対応のドキュメントを作成しています。
そのため、今後バージョン更新によって本ブログで紹介している機能は動作しなくなる可能性があることに注意してください。
:::

今回作成した内容は以下リポジトリで公開しています。

https://github.com/takapi327/laika-sandbox

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

sbtプロジェクトは任意のものを使用・作成してください。

以下のコマンドを使用すると、Scalaのシードプロジェクトを作成できます。

```shell
$ sbt new scala/scala3.g8
```
:::

sbt自体初めての方は以下の記事も参考になるかと思います。

https://blog.3qe.us/entry/2024/04/17/213142

sbtプロジェクトの`project/plugins.sbt`にLaikaのsbtプラグインを追加します。

```scala
addSbtPlugin("org.typelevel" % "laika-sbt" % "1.2.0")
```

次に、プロジェクトのbuild.sbtでプラグインを有効にします。

```scala
enablePlugins(LaikaPlugin)
```

:::message alert
公式の[ドキュメント](https://typelevel.org/Laika/latest/02-running-laika/01-sbt-plugin.html)ではこの設定で動作すると記載されていますが、実際には以下のエラーが発生しました。

```
[error] stack trace is suppressed; run last laikaSite for the full output
[error] (laikaSite) laika.api.errors.InvalidDocuments: One or more invalid documents:
[error] /index.md
[error] 
[error]   [1]: No target for home link found - for options see 'Theme Settings / Top Navigation Bar' in the manual
[error] 
[error]   
[error]   ^
```

このエラーはLaikaのテーマ設定に関するエラーのようで、どうも以下のようにテーマを指定しナビゲーションバーに表示するホームのリンクを設定する必要があるみたいです。

※ `build.sbt`で`LaikaPlugin`を有効化したプロジェクトに設定

```sbt
laikaTheme := Helium.defaults.site
  .topNavigationBar(
    homeLink = IconLink.internal(Root / "index.md", HeliumIcon.home)
  )
  .build
```
:::

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

Laikaはナビゲーションを自動生成する機能を提供していますが、そのナビゲーションは先ほどの`default.template.html`でカスタマイズすることができます。

ナビゲーションを言語別に分けるためには、言語ごとにナビゲーションを生成する必要があります。Laikaはナビゲーションを生成する際に、`helium.site.mainNavigation`という変数を使用しています。

この`helium.site.mainNavigation`はナビゲーションのエントリーを保持する変数で、この変数を使用してナビゲーションを生成しています。

現在の`src/docs/default.template.html`ではナビゲーション部分は以下のようになっています。

```html
<nav id="sidebar">
  ...

  @:navigationTree {
  entries = ${helium.site.mainNavigation.prependLinks} [
  { target = "/", excludeRoot = true }
  ] ${helium.site.mainNavigation.appendLinks}
  }

</nav>
```

この`@:navigationTree`はナビゲーションを生成するためのマクロで、`entries`にナビゲーションのエントリーを指定することでナビゲーションを生成しています。

今回は言語用のドキュメントは何らかの方法で言語を選択したら表示されるようにし、サイドナビゲーションは言語に応じて表示されるようにします。

そのため、言語選択前はサイドナビゲーションに言語それぞれのナビゲーションは表示されないようにしたいです。

言語用のメタデータに応じて出しわけを行ってもいいのですが、今回は言語選択前のドキュメントは言語別に用意するものとは別でかつ英語としたいため、言語用のメタデータとは別にルート用のメタデータ (`isRootPath`)を使用することにします。

先ほどの、`default.template.html`を以下のように修正します。

```html
<nav id="sidebar">
  ...

  @:if(laika.metadata.isRootPath)
  @:navigationTree {
  entries = ${helium.site.mainNavigation.prependLinks} [
  { target = "/", excludeRoot = true, depth = 1 }
  ] ${helium.site.mainNavigation.appendLinks}
  }
  @:else
  @:navigationTree {
  entries = ${helium.site.mainNavigation.prependLinks} [
  { target = /${laika.metadata.language}/, excludeRoot = true, excludeSections = ${helium.site.mainNavigation.excludeSections}, depth = ${helium.site.mainNavigation.depth} },
  ] ${helium.site.mainNavigation.appendLinks}
  }
  @:@
</nav>
```

Laikaはテンプレートファイル内で条件分岐を行うためのマクロを提供しており、`@:if`と`@:else`を使用することで条件分岐を行うことができます。

この機能を利用して、今回は以下のような設定でナビゲーションを生成しています。

- `laika.metadata.isRootPath`が`true`の場合は、ルート用のナビゲーションを生成
  - ルート用のナビゲーションは深さを1にして、ルート以外のエントリーを表示しないようにしています
- `laika.metadata.isRootPath`が`false`の場合は、言語用のナビゲーションを生成
  - 言語用のナビゲーションはLaikaデフォルトのメタデータを使用して深さなどの設定を行っています
  - `target = /${laika.metadata.language}/`として言語ごとにナビゲーションを生成しています

:::message
`target = /${laika.metadata.language}/`の部分は、言語ごとに書かれたドキュメントはそれぞれ異なるディレクトリの中に格納されています。Laikaではディレクトリを分割するとそのディレクトリがパスになるため、`en`というディレクトリに格納されたドキュメントは`/en/`というパスになります。
そのため、`target`を`/${laika.metadata.language}/`として言語ごとにナビゲーションを生成しないとサイドナビゲーションのパスがルート直下のパスになってしまい、正しくナビゲーションが生成されないためメタデータを使用して言語を指定しています。
:::

テンプレートファイルを修正したら、ルート用のメタデータを`src/docs/index.md`に追加します。

```markdown
{%
laika.metadata {
  language = en
  isRootPath = true
}
%}

# Hello, Laika!
```

これで、言語選択前のドキュメントは言語別に用意するものとは別で、サイドナビゲーションは言語に応じて表示されるようになりました。

再度、`sbt laikaSite`を実行しプレビューサーバーを起動してみてください。

```shell
$ sbt laikaSite
$ sbt laikaPreview
```

サイドナビゲーションから言語の項目が消えています。

![](/images/scala-laika-multi-language-support/laika-en-html-nav.png)

それぞれの言語のパスにアクセスすると、言語ごとのナビゲーションが表示されていることが確認できるはずです。

**英語**

`http://localhost:4242/en`

![](/images/scala-laika-multi-language-support/laika-en-html-nav-en.png)

**日本語**

`http://localhost:4242/ja`

![](/images/scala-laika-multi-language-support/laika-en-html-nav-ja.png)

このままでは、パスを知っていないと言語を切り替えることができないため、それぞれの言語に対応したリンクを用意する必要があります。

ルート用のドキュメントに以下のようなリンクを追加します。

```markdown
{%
laika.metadata {
  language = en
  isRootPath = true
}
%}

# Hello, Laika!

- [English](en/index.md)
- [Japanese](ja/index.md)
```

これで、言語を切り替えるためのリンクが表示されるようになり言語ごとにドキュメントを切り替えることができるようになりました。

![](/images/scala-laika-multi-language-support/laika-en-html-nav-lang.png)

# まとめ

今回の方法だと言語ごとにそれぞれドキュメントを作成し管理する必要がありますが、Laikaの柔軟性を活かして多言語対応のドキュメントを作成することができました。

ただ現状の構成だと、一度言語を選択するとその言語のドキュメントしか表示されないため、それぞれのドキュメントで言語を切り替えるためのリンクを用意する必要があったりと、ユーザビリティの面で改善の余地があると感じました。

Laikaはバージョニングなどの機能を提供しており以下のようにヘッダーでバージョンを切り替えることができるため、テーマなどを自作してみるとよりユーザビリティの高いドキュメントを作成することができるかもしれません。

![](/images/scala-laika-multi-language-support/laika-version.png)

まだまだLaikaの機能は多岐にわたり、今回紹介した機能以外にもさまざまな機能が提供されています。ドキュメントのバージョニングは自身のOSSでも導入してみたい機能の一つであるため、こちらも調査が終わり対応出来次第ブログにて紹介したいと考えています。

PS. LaikaはTypelevelコミュニティのプロジェクトだと説明しましたが、Typelevelには`sbt-typelevel`というプロジェクトもあり、このプロジェクトはLaikaを使用してドキュメントを生成する機能も提供しています。このプロジェクトを使用するとTypelevel用のテーマやテンプレートを使用することができるため、Typelevelプロジェクトを開発している場合は`sbt-typelevel`を使用することでより効率的にドキュメントを作成することができるかもしれません。

https://typelevel.org/sbt-typelevel/
