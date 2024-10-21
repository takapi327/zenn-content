---
title: "Laikaを使用したScala製ドキュメントツールのバージョニング対応"
emoji: "♦️"
type: "tech"
topics: ["Scala", "Typelevel", "Laika", "ドキュメント"]
publication_name: nextbeat
published: false
---

# はじめに

この記事では、Scala製のドキュメントツールであるLaikaを使用して、バージョニング対応を行う方法について解説します。

今回作成した内容は以下リポジトリで公開しています。

https://github.com/takapi327/laika-sandbox

また、今回作成したドキュメントは以下で公開しています。

https://takapi327.github.io/laika-sandbox/0.1/

## Laikaとは

Laikaに関しては、以前にも記事を書いていますので詳細はこちらを参照してください。

https://zenn.dev/nextbeat/articles/scala-laika-multi-language-support

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

```scala
laikaTheme := Helium.defaults.site
  .topNavigationBar(
    homeLink = IconLink.internal(Root / "index.md", HeliumIcon.home)
  )
  .build
```
:::

これでプロジェクトの準備が整いました。

## ドキュメントの作成

Laikaを使用したドキュメントの作成方法やカスタマイズ方法に関しては、以下記事の「ドキュメントの作成」を参照してください。

https://zenn.dev/nextbeat/articles/scala-laika-multi-language-support#%E3%83%89%E3%82%AD%E3%83%A5%E3%83%A1%E3%83%B3%E3%83%88%E3%81%AE%E4%BD%9C%E6%88%90

## バージョニング対応

Laikaはデフォルトでドキュメントをバージョン管理する機能を提供しています。

https://typelevel.org/Laika/latest/03-preparing-content/01-directory-structure.html#versioned-documentation

各ディレクトリと個々のドキュメントは、バージョン付きまたはバージョンなしとしてマークすることができます。バージョン管理されたドキュメントはすべて、ルートのサブディレクトリにレンダリングされます。

:::message
今回はLaikaがデフォルトで提供しているHeliumテーマを使用したバージョニング対応のドキュメントを作成します。
他のテーマを使用した場合は、テーマによってはバージョニング対応が異なる場合があります。
:::

### バージョン管理の設定

Laikaのバージョン管理は、`Versions`というオブジェクトを使用して設定します。

`Versions`オブジェクトは、バージョンのリストを保持し、それぞれのバージョンに対してドキュメントのルートディレクトリを指定します。

```scala
import laika.config.{ Version, Versions }
```

次に、`Versions`オブジェクトを作成し、バージョンのリストを設定します。まずは、`forCurrentVersion`メソッドにより、現在のバージョンを指定します。

`Version`は2つの引数を取り、それぞれバージョンの表示名とバージョンのパス(ディレクトリ)名を指定します。

```scala
val versions = Versions
  .forCurrentVersion(Version("0.1.x", "0.1"))
```

次に、作成した`Versions`オブジェクトを`Helium`のテーマに設定します。

```scala
laikaTheme := Helium.defaults.site
  .topNavigationBar(
    homeLink = IconLink.internal(Root / "index.md", HeliumIcon.home)
  )
  .site
  .versions(versions)
  .build
```

これでバージョン管理の設定が完了しました。

`laikaTheme`の設定を行った後`sbt laikaSite`を実行し、`sbt liakaPreview`でプレビューを確認してください。

以下のようにナビゲーションバーにバージョンを切り替えるためのセレクトボックスが表示されているかと思います。

![バージョン切り替え](/images/scala-laika-versioning-support/version-select.png)

表示されたバージョンを選択すると`Version`の引数で渡したバージョンのパスで始まる`/0.1/index.html`に遷移することができます。

しかし、現時点では`0.1`のディレクトリが存在しないため、404エラーが発生してしまいます。`0.1`のディレクトリを作成しても良いのですが、それだとバージョンが増えるたびにディレクトリを作成する必要があり管理が煩雑になってしまいます。

そこで、Laikaでは`laika.versioned`というメタデータを設定することで自動的に`forCurrentVersion`で指定したバージョンのディレクトリに作成したドキュメントを配置することができます。

さっそく`docs/directory.conf`に`laika.versioned`を設定してみましょう。

```conf
laika.versioned = true
```

`laika.versioned`を`true`に設定することで、Laikaは`Versions`で指定したバージョンのディレクトリにドキュメントを配置するようになります。

先ほどと同様に`sbt laikaSite`を実行してみましょう。

`target`ディレクトリに`0.1`のディレクトリが作成され、その中にドキュメントが配置されていることが確認できるかと思います。

```
target
└── docs
    └── site
        └── 0.1
            ├── index.html
            └── ...
```

`sbt laikaPreview`でプレビューを確認すると、対応したバージョンのドキュメントが表示されることが確認できるかと思います。

![バージョン切り替え](/images/scala-laika-versioning-support/version-view.png)

:::message
今回は`docs`ディレクトリのルートに作成したドキュメントをバージョン管理するようにしました。そのため、全てのファイルおよびディレクトリが`0.1`のディレクトリに配置されています。
今の状態ではプレビューでブラウザに表示されるドキュメントは`http://localhost:4242/0.1/`になります。
`http://localhost:4242`にアクセスすると404エラーが発生しますので、注意してください。

ルートのパスでもドキュメントを表示しつつ、バージョン選択時にのみ対応したドキュメントを表示するためには、`laika.versioned`とディレクトリ構成を工夫する必要があります。
:::

### バージョンの追加

先ほどの設定で1つのバージョンを管理することができましたが、複数のバージョンを管理する場合はどうすればよいでしょうか。

`Versions`オブジェクトに複数のバージョンを追加するために、`withNewerVersions`や`withOlderVersions`メソッドが用意されています。

まずは、`withNewerVersions`メソッドを使用して新しいバージョンを追加してみましょう。

```scala
val versions = Versions
  .forCurrentVersion(Version("0.1.x", "0.1"))
  .withNewerVersions(Version("0.2.x", "0.2"))
```

`withNewerVersions`メソッドは、引数に`Version`を取り、新しいバージョンを追加します。

これで`0.2`のバージョンが追加されました。

先ほどと同様に`sbt laikaSite`を実行し、`sbt liakaPreview`でプレビューを確認してください。

ナビゲーションバーに`0.2`のバージョンが追加されていることが確認できるかと思います。

![バージョン切り替え](/images/scala-laika-versioning-support/version-select-02.png)

ただし、現時点では`0.2`のディレクトリが存在しないため、404エラーが発生してしまいます。

Laikaは`forCurrentVersion`で指定したバージョンのディレクトリにドキュメントを配置するため、`0.2`のディレクトリは別途作成する必要があります。

バージョンの設定を以下のように変更することで、`0.2`のディレクトリにドキュメントを配置することができます。

```scala
val versions = Versions
  .forCurrentVersion(Version("0.2.x", "0.2"))
  .withOlderVersions(Version("0.1.x", "0.1"))
```

`withOlderVersions`メソッドは、引数に`Version`を取り、古いバージョンを追加します。

これで`0.2`のバージョンのドキュメントを作成することができます。

`sbt laikaSite`を実行し、`sbt liakaPreview`でプレビューを確認してください。

`0.2`のディレクトリにドキュメントが配置されていることが確認できるかと思います。

![バージョン切り替え](/images/scala-laika-versioning-support/version-view-02.png)

新しいバージョンを追加して、そのバージョンでドキュメントを作成することができましたが今度は`0.1`のバージョンのドキュメントが表示されなくなってしまいました。

これでは、バージョンを切り替えた際に古いバージョンのドキュメントを表示することができず、別のバージョンを表示したい場合はその都度ディレクトリを作成する必要があります。これでは、複数のバージョン用のディレクトリを管理するのと変わらないため、自動でバージョンのディレクトリを作成する旨みがありません。

これをどう解消するかというとGithub Pagesと組み合わせて使用することで、問題を解消することができます。

## Github Pagesへのデプロイ

まずは、Github Pagesを使用してドキュメントを公開するために専用の`gh-pages`ブランチを空で作成します。

```shell
$ git checkout --orphan gh-pages
$ git rm -rf .
$ git commit --allow-empty -m "Create gh-pages branch"
$ git push origin gh-pages
```

ブランチを作成したら、`Workflow`で書き込みを許可するために、`Settings` -> `Actions` -> `General` -> `Workflow permissions`を`Read and write permissions`に変更します。

![Workflow permissions](/images/scala-laika-versioning-support/workflow-permissions.png)

次に、Github Actionsを使用してLaikaで生成したドキュメントをGithub Pagesにデプロイするためのワークフローを作成します。

`.github/workflows`ディレクトリに`deploy.yml`を作成し、以下の内容を記述します。

:::details deploy.yml
```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches:
      - master

env:
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

concurrency:
  group: ${{ github.workflow }} @ ${{ github.ref }}
  cancel-in-progress: true

jobs:
  site:
    name: Generate Site
    strategy:
      matrix:
        os: [ ubuntu-latest ]
        java: [ corretto@11 ]
    runs-on: ${{ matrix.os }}
    steps:
      - name: Install sbt
        uses: sbt/setup-sbt@v1

      - name: Checkout current branch (full)
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Java (corretto@11)
        id: setup-java-corretto-11
        if: matrix.java == 'corretto@11'
        uses: actions/setup-java@v4
        with:
          distribution: corretto
          java-version: 11
          cache: sbt

      - name: sbt update
        if: matrix.java == 'corretto@11' && steps.setup-java-corretto-11.outputs.cache-hit == 'false'
        run: sbt +update

      - name: Generate site
        run: sbt laikaSite

      - name: Publish site
        uses: peaceiris/actions-gh-pages@v4.0.0
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: target/docs/site
          keep_files: true
```
:::

このワークフローは、`master`ブランチにプッシュされた際にLaikaで生成したドキュメントをGithub Pagesにデプロイするものです。

ワークフローを作成したら、Githubにプッシュしてワークフローを実行するのですが、その前に`build.sbt`でバージョンの設定を以下のように変更しておきましょう。

```scala
val versions = Versions
  .forCurrentVersion(Version("0.1.x", "0.1"))
```

変更が完了したら、Githubにプッシュしてワークフローを実行します。

ワークフローが正常に完了すると、Github Pagesにドキュメントが公開されているかと思います。

`gh-pages`ブランチには、Laikaで生成したドキュメントが配置されているかと思います。

```
gh-pages
├── 0.1
│   ├── index.html
│   └── ...
├── laika
│   └── versionInfo.json
└── .nojekyll
```

Github Pagesに公開されたドキュメントを確認するには、`https://<username>.github.io/<repository>/0.1/`にアクセスしてください。

:::message
プライベートリポジトリの場合は、生成されるURLがランダムな文字列になるため、Github Pagesの設定画面からURLを確認してください。
:::

これでGithub PagesにLaikaで生成したドキュメントをデプロイすることができました。

次に、ようやくバージョンを切り替えた際に対応したバージョンのドキュメントを表示する方法について解説します。

といっても難しいことはなく、`build.sbt`でバージョンの設定を以下のように変更するだけです。

```scala
val versions = Versions
  .forCurrentVersion(Version("0.2.x", "0.2"))
  .withOlderVersions(Version("0.1.x", "0.1"))
```

これだけ？と思われるかもしれませんが、Github Pagesにデプロイする設定を見返してみましょう。

ここで使用している`peaceiris/actions-gh-pages`は、`keep_files`を`true`に設定することでデプロイ先 (gh-pagesブランチ) にある既存のファイルを保持したまま新しいファイルをデプロイしてくれ、新しいファイルが既存のファイルと同じ名前の場合は上書きされます。

```yaml
- name: Publish site
  uses: peaceiris/actions-gh-pages@v4.0.0
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: target/docs/site
    keep_files: true
```

つまり、先ほどの設定で`0.1`のディレクトリでドキュメントを配置しデプロイを行いましたが、今度は`forCurrentVersion`で指定した`0.2`のディレクトリにドキュメントを配置しデプロイを行います。
そのため、`0.1`のディレクトリに配置されていたドキュメントは保持されたまま、新しいバージョンのドキュメントのみがデプロイされることで、古いバージョン用のドキュメントは残り続けるため、バージョンを切り替えた際に対応したバージョンのドキュメントを表示することができるようになるということです。

実際にGithub Pagesにデプロイを行い、ドキュメントを確認してみましょう。

再度`build.sbt`でバージョンが以下のようになっていることを確認しましょう。

```scala
val versions = Versions
  .forCurrentVersion(Version("0.2.x", "0.2"))
  .withOlderVersions(Version("0.1.x", "0.1"))
```

また、ちゃんとバージョンごとに異なるドキュメントが表示されているかを確認するために`docs/index.md`のタイトルを以下のように変更しておきましょう。

```md
# Hello, Laika! v2
```

全ての変更が完了したらGithubにプッシュしてワークフローを実行します。

ワークフローが正常に完了すると、Github Pagesに新しいバージョンのドキュメントが公開されているかと思います。

**バージョン 0.1**

![バージョン切り替え](/images/scala-laika-versioning-support/version-view-ghpages-01.png)

**バージョン 0.2**

![バージョン切り替え](/images/scala-laika-versioning-support/version-view-ghpages-02.png)

`gh-pages`ブランチには、最初に作成した`0.1`のディレクトリと新しく作成した`0.2`のディレクトリが存在しているかと思います。

```
gh-pages
├── 0.1
│   ├── index.html
│   └── ...
├── 0.2
│   ├── index.html
│   └── ...
├── laika
│   └── versionInfo.json
└── .nojekyll
```

これでバージョンを切り替えた際に対応したバージョンのドキュメントを表示することができるようになりました。

# おまけ

Laikaではバージョン管理に関する他にも様々な設定が可能です。

## ラベルの設定

Laikaではバージョンにそれぞれラベルを設定することができます。

今回使用している`Helium`テーマでは、以下3つのラベル(デザイン)が用意されています。

- `Stable`: 安定版
- `Dev`: 開発版
- `Eol`: サポート終了

ラベルを設定するには、`Version`に`withLabel`メソッドを使用します。

```scala
val versions = Versions
  .forCurrentVersion(Version("0.2.x", "0.2").withLabel("Stable"))
  .withOlderVersions(Version("0.1.x", "0.1").withLabel("EOL"))
  .withNewerVersions(Version("0.3.x", "0.3").withLabel("Dev"))
```

このように設定することで、ナビゲーションバーにラベルが表示されるかと思います。

![ラベル](/images/scala-laika-versioning-support/version-label.png)

ドキュメントをバージョニングする際に、バージョンの状態をラベルで表示することで、ユーザーに対してバージョンの状態をわかりやすく伝えることができて便利ですね。

上記３つのラベル以外を使用したい場合は、Laikaはcssも独自に追加・カスタマイズすることができるため、自由にラベルを追加することも可能です。

## Canonicalの設定

Laikaでは、`Versions`に`setCanonical`メソッドを使用して、バージョンのCanonicalリンクを設定することができます。

Canonicalリンクは、検索エンジンに対して、同じコンテンツが異なるURLで提供されている場合に、どのURLが優先されるべきかを伝えるために使用されます。

```scala
val versions = Versions
  .forCurrentVersion(Version("0.2.x", "0.2").withLabel("Stable").setCanonical)
  .withOlderVersions(Version("0.1.x", "0.1").withLabel("EOL"))
  .withNewerVersions(Version("0.3.x", "0.3").withLabel("Dev"))
```

このように設定することで、バージョンのCanonicalリンクを設定するこたができます。

複数のバージョンを管理する際に、検索エンジンに対して正しいバージョンのドキュメントを表示するように指定することができるため、SEO対策にも有効ですね。

## フォールバックの設定

Laikaでは、バージョンのフォールバックを設定することができます。

フォールバックは、指定したバージョンが存在しない場合に、代替のバージョンを表示するために使用されます。

```scala
val versions = Versions
  .forCurrentVersion(Version("0.2.x", "0.2").withLabel("Stable").setCanonical)
  .withOlderVersions(Version("0.1.x", "0.1").withLabel("EOL"))
  .withNewerVersions(Version("0.3.x", "0.3").withLabel("Dev").withFallbackLink("/index.html"))
```

:::message
`withFallbackLink`メソッドでパスを指定しなくても、デフォルトでルートのパス (今回だとindex.html) が指定されています。
`withFallbackLink`はデフォルトの挙動を変更したい場合、例えば`/404.html`や`/table-of-contents.html`などのページを指定したい場合に使用すると便利です。
:::

# まとめ

今回は、Scala製のドキュメントツールであるLaikaを使用して、バージョニング対応を行う方法について解説しました。

デフォルトでバージョニングがサポートされているため、比較的簡単にバージョン管理を行うことができました。

ただ、古いバージョンのドキュメントを更新したい場合はどうするのだろうか？であったり、新しいバージョン (開発版)と安定版を同時に運用する場合はどうするのだろうか？など、実際の運用においてはさらに検討が必要かもしれません。

ここは、自身のプロジェクトで引き続き検証していこうかなと思います。
