---
title: "sbt-gitを使用したsbtマルチプロジェクトのバージョン管理"
emoji: "♦️"
type: "tech"
topics: ["Scala", "sbt", "sbtgit", "git"]
publication_name: nextbeat
published: true
---

:::message
この記事は、Scala Advent Calendar 2024 5日目の記事です。
:::

# はじめに

今回はsbt-gitというプラグインを使用してsbtプロジェクトのバージョン管理を行う方法について紹介します。

今回のブログは詳細に使い方を説明するというよりも「こんな感じの設定にすればこんな感じで管理できるよ」という内容になります。

:::message
今回は以下環境でプロジェクトを作成しています。

- Scala: 3.5.2
- sbt: 1.10.6
- sbt-git: 2.1.0

sbtプロジェクトは任意のものを使用・作成してください。

以下のコマンドを使用すると、Scalaのシードプロジェクトを作成できます。

```shell
$ sbt new scala/scala3.g8
```
:::

今回紹介したコードは以下に置いてありますので参考にしてください。

https://github.com/takapi327/sbt-git-sandbox

# sbt-gitとは

sbt-gitはsbtプロジェクト内で直接 git コマンドライン機能を提供するだけでなく、他のプラグインが git を利用できるようにするための sbt プラグインです。

https://github.com/sbt/sbt-git

## 設定

sbt-gitはsbtプロジェクトのプラグインとして追加します。

```scala
addSbtPlugin("com.github.sbt" % "sbt-git" % "2.1.0")
```

今回はsbt-gitのGitVersioningプラグインを使用します。このプラグインはgitのタグをバージョンとして使用することができます。

```scala
enablePlugins(GitVersioning)
```

このプラグインを有効にしたプロジェクトでは以下優先順位でバージョンが設定されます。

1. version-propertyの設定(デフォルトはproject.version)を見て、sys.propsに値があるかどうかをチェックし、もしあればそれを使う。
2. Gitのタグを使用。gitTagToVersionNumber の設定に最初にマッチしたものがバージョンの割り当てに使われます。デフォルトでは`v`と数字で始まるタグを探し、その数字をバージョンとして使用します。複数のバージョンタグがある場合は、一番大きなものを選びます。
3. タグも見つからない場合は先頭のコミットを参照し、これを git.baseVersion の設定にアタッチします。`<base-version>.<git commit sha>` となります。
4. ヘッドコミットも存在しない場合は、現在のタイムスタンプをベースバージョンに追加します。`<base-version>.<timestamp>` となります。

実際にプロジェクトに設定してバージョンを確認してみましょう。

```scala 3
lazy val root = project.in(file("."))
  .enablePlugins(GitVersioning)
```

今回使用したプロジェクトは作成したばかりのものなのでまだコミットもタグもない状態です。そのためバージョンは以下のようになります。(上記4番に該当)

```shell
sbt:sbt-git-sandbox> version
[info] version
[info]  027a3e62d43a675254a9cb0d581093ee64c20a7a-SNAPSHOT
```

## Gitタグを使用したバージョン管理

プラグインが有効になっていることは確認できたので実際にタグを打ってバージョンを確認してみましょう。

プラグインのデフォルト設定では`v`で始まるタグをバージョンとして使用します。タグを打つ際は以下のように行います。

```shell
$ git tag v0.1.0
$ git push origin v0.1.0
```

タグを打った後にバージョンを確認してみましょう。

```shell
sbt:sbt-git-sandbox> version
[info] version
[info]  0.1.0
```

バージョンが正しく設定されていることが確認できました。

# プロジェクトごとのバージョン管理

Gitのタグを使用してバージョン管理を行うことができるようになりましたが、これの何が嬉しいのでしょうか？

例えば1つのsbtファイルで複数のプロジェクトを管理するようなモノレポ構成をとっている場合を考えてみましょう。sbtプロジェクトはそれぞれバージョンを設定することができるのでシンプルに書くと以下のような設定になるかと思います。

```scala 3
lazy val project1 = (project in file("project1"))
  .settings(version := "1.0.0")

lazy val project2 = (project in file("project1"))
  .settings(version := "1.0.0")

lazy val root = project.in(file("."))
  .aggregate(project1, project2)
```

これでも問題はないのですが、大抵の場合コードはGithubで管理されているかと思います。その場合リリースする際にはタグを打つことが一般的で、そのタグに対応したリリースノートを書くことが多いですよね。

その場合プロジェクトごとにバージョンを管理するのはちょっと面倒です。sbtのプロジェクト上では別々のプロジェクトが同じバージョンであっても問題ないですが、Github上ではタグが重複してしまいます。

|     | project1 | project2 |
|-----|----------|----------|
| sbt | 1.0.0    | 1.0.0    |
| git | 1.0.0    | ❌        |

同時にリリースする場合は同じタグを共通で使用してリリースノートも共通で管理しても良いかもしれません。しかし、プロジェクトは要件などによっては同じスピードで開発が進むとは限りません。

例えば、`project1`は頻繁に改修が入るが`project2`はあまり改修が入らない場合、`project1`のバージョンが頻繁に上がることになります。

こんな感じに

- `project1`: 5.0.0
- `project2`: 2.0.0

その場合`project2`のバージョン3.0.0をリリースしたいとなった場合、3.0.0のタグはすでに`project1`で使用されているため使用することができません。タグがすでにあるからタグは切らずに対象のリリースノートに追記するという方法もありますが、タグは切った時の内容を確認するためにも重要な情報です。
また、それぞれのプロジェクトのバージョン乖離が大きくなるとバージョン更新の遅いプロジェクトのリリースノートは最新であっても2ページ目3ページ目と過去のリリースノートを探す必要が出てきてしまいます。

これを回避する簡単な方法は`プロジェクト名@バージョン`という形式で専用のバージョンを設定することです。OSSのプロジェクトなどでよく見かける形式ですね。

これであればプロジェクトごとにバージョンを管理することができます。また、リリースノートもプロジェクトごとに管理することができます。

ただし、この方法は手動でバージョンを設定する必要があり少し手間です。最初にスクリプトを書いておけばそれほど手間ではないですがもう少し簡単な方法があります。

それがsbt-gitを使用してバージョンを管理する方法です。

sbt-gitを使用するとGitのタグをバージョンとして使用することができました。つまり`プロジェクト名@バージョン`という形式でタグを作成してしまえば、それをsbt プロジェクトのバージョンとして使用できるということです。

sbt-gitはデフォルトでは`v`で始まるタグをバージョンとして使用しますが、これはカスタマイズすることができます。

`gitTagToVersionNumber` という設定を使用してタグをバージョンに変換する関数を設定することができます。`prefix`の部分がプロジェクト名になります。

```scala
git.gitTagToVersionNumber := { tag =>
  if (tag matches s"""^$prefix@([0-9]+)((?:\\.[0-9]+)+)?([\\.\\-0-9a-zA-Z]*)?""") Some(tag)
  else None
}
```

上記設定を追記したら実際に`プロジェクト名@バージョン`という形式でタグを打ってバージョンを確認してみましょう。

```shell
$ git tag HelloWorld@1.0.0
$ git push origin HelloWorld@1.0.0
```

タグ作成後sbtのバージョンもタグと同じになっていることが確認できます。

```shell
sbt:sbt-git-sandbox> version
[info] helloWorld / version
[info]  HelloWorld@1.0.0
[info] version
[info]  0.1.0
```

ただし、このままではcheckoutしているタグのバージョンしか参照してくれません。 その場合は以下設定を追加することでローカルのタグから最新のタグを参照することができます。

また、`gitDescribePatterns`を使用してタグのパターンを指定することで対象のプロジェクトに関連するタグのみを参照することができます。

```sbt
git.useGitDescribe := true,
git.gitDescribePatterns := Seq(s"$prefix@*"),
```

## sbt-releaseとの併用

さてsbtプロジェクトのリリース方法は様々ですが、`sbt-release`を使用してリリースする方法は様々ある中でもよく使われているのではないでしょうか？

https://github.com/sbt/sbt-release

`sbt-release`を使用するとバージョンの更新やタグの作成などを自動化することができます。`sbt-release`は他のプラグインとの組み合わを簡単に行うことができます。以下ブログなどを参考にしてください。

https://blog.3qe.us/entry/2022/12/22/120000#org708af03

今回の構成を`sbt-release`と併用したい場合はとりあえず以下`releaseSettings`をコピペしてsbtプロジェクトの設定に追加すればOKです。

```scala
lazy val customVersion: SettingKey[CustomVersion] = settingKey[CustomVersion]("Custom version")

def releaseSettings(prefix: String) = Seq(
  git.gitTagToVersionNumber := { tag =>
    if (tag matches s"""^$prefix@([0-9]+)((?:\\.[0-9]+)+)?([\\.\\-0-9a-zA-Z]*)?""") Some(tag)
    else None
  },
  git.gitDescribePatterns := Seq(s"$prefix@*"),
  customVersion := {
    val rawVersion = git.gitDescribedVersion.value.getOrElse((ThisBuild / version).value)
    CustomVersion.build(rawVersion)
      .getOrElse(versionFormatError(rawVersion))
  },
  version := customVersion.value.version,
  releaseVersionBump := Version.Bump.Minor,
  releaseTagName := customVersion.value.tag,
  releaseProcess := Seq[ReleaseStep](
    checkSnapshotDependencies,
    inquireVersions,
    runClean,
    runTest,
    tagRelease,
    publishArtifacts,
    pushChanges
  )
)
```

::::details CustomVersion

:::message
以下ではバージョン更新がマイナーに固定されているため適宜カスタマイズしてください。
:::

```scala
import sbtrelease.Version

import scala.util.matching.Regex
import util.control.Exception.*

case class CustomVersion(
  tag: String,
  version: String,
)

object CustomVersion {
  val pattern: Regex = """(.+)@([0-9]+)((?:\.[0-9]+)+)?([\.\-0-9a-zA-Z]*)?""".r

  def build(str: String): Option[CustomVersion] = {
    allCatch opt {
      val pattern(prefix, maj, subs, qual) = str
      // parse the subversions (if any) to a Seq[Int]
      val subSeq: Seq[Int] = Option(subs) map { str =>
        // split on . and remove empty strings
        str.split('.').filterNot(_.trim.isEmpty).map(_.toInt).toSeq
      } getOrElse Nil
      val version = Version(maj.toInt, subSeq, Option(qual).filterNot(_.isEmpty))
      val newVersion = version.bumpMinor.withoutQualifier.unapply
      CustomVersion(s"$prefix@$newVersion", newVersion)
    }
  }
}
```
::::

やっていることは単純で`releaseTagName`にはsbt-git経由で取得したバージョンをもとにタグを生成し、`releaseVersion`には同じくsbt-git経由で取得したバージョンをもとにsbt-releaseで扱うバージョンを生成しています。

これによって以下のような構成となります。

- Git Tag: `HelloWorld@1.0.0`
- sbt Version: `1.0.0`

今回バージョン管理はGitのタグを使用して行うためsbt-releaseで扱うversion.sbtは使用しません。そのため`releaseProcess`から`setReleaseVersion`や`setNextVersion`などは削除しています。

`releaseSettings`の引数にプロジェクト名を渡すことでプロジェクト名がついたバージョンを管理することができます。

```scala
lazy val helloWorld = (project in file("functions/hello-world"))
  .settings(name := "hello-world")
  .settings(publish / skip := true) // 今回はどこにも公開しないためスキップしています。
  .settings(releaseSettings("HelloWorld")*)
  .enablePlugins(GitVersioning)
```

この設定でGit上ではプロジェクト名がついたバージョンを管理し、sbt上では通常のバージョンを管理することができるのでAWS ECRなどに使用するバージョンにはプロジェクト名などを含めずに使用することができます。

以下コマンドを実行して`sbt-release`を使用してリリースしてみましょう。

:::message
この構成では一番最初は手動で`git tag`を実行してタグを打っておく必要があります。
:::

以下のようにプロジェクト名付きで新しくタグが作成されていることが確認できます。

```shell
sbt 'project helloWorld; release with-defaults'
...
[success] Total time: 0 s, completed 2024/12/05 21:42:59
[info] Everything up-to-date
[info] To github.com:takapi327/sbt-git-sandbox.git
[info]  * [new tag]         HelloWorld@1.1.0 -> HelloWorld@1.1.0
```

sbtプロジェクトのバージョンも確認しておきましょう。以下のように指定した次のバージョンに更新されていて、かつ数値のみで構成されたバージョンとなっていることが確認できます。

```shell
sbt 'project helloWorld; version'
...
[info] 1.2.0
```

以上でsbt-gitを使用してバージョン管理を行う方法について紹介しました。

# まとめ

今回はsbt-gitを使用してGitのタグをバージョンとして使用する方法と、プロジェクトごとにバージョンを分けて管理する方法についても紹介しました。

プロジェクトごとにバージョンを管理することでリリースノートなどをプロジェクトごとに管理することができるようになります。
スクリプトを書いてCI/CDで自動化することもできますが、sbt-gitを使用することで簡単にバージョン管理を行いつつ、sbt-releaseとの併用もできますのでぜひ試してみてください。
