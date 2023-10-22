---
title: "Scala 3におけるinlineを用いたコンパイル時の様々なパターンマッチングと条件分岐"
emoji: "♦️"
type: "tech"
topics: ["Scala", "Scala3", "dotty"]
published: true # 公開設定（falseにすると下書き）
---

## はじめに

Scala 3では言語設計が一新され多くの新機能が追加されました。その一環として、Scala 3ではメタプログラミングの全面的な見直しが行われました。
見直しが行われた機能の中にはScala 2から存在していた`inline`も含まれていました。
今回はScala 3から大きく機能が拡張された`inline`修飾子に関しての話であり、特に`inline`を使用したパターンマッチングと条件分岐の使用方法に関しての話となります。

まずScala3のinlineとinlineと組み合わせて使用できるAPIの紹介を行い、その後にそれぞれを組み合わせて行うパターンマッチングと条件分岐の使用方法に関して説明できればと思います。

:::message
本記事で使用するScalaのバージョンは`3.3.1`を使用しています。
:::

https://docs.scala-lang.org/scala3/reference/metaprogramming/inline.html

## Scala3のinline

Scala 3で見直された`inline`修飾子は、Scala 2の時から`@inline`アノテーションとして存在していました。しかし、Scala 2での`inline`修飾子は、単なるヒントでありコードがインライン化されることを保証することはできませんでした。
Scala 3で見直されたことによって、`inline`修飾子はコードがインライン化されていることを保証し、インライン化できない場合はコンパイルエラーを出すことができるようになりました。

:::message
インライン化とは、指定されたコードがコンパイル時にのみ評価または展開されることを意味します。
:::

インライン化の簡単な例として以下のようなコードがあるとします。

```scala 3
inline val debugLogEnabled = true
@main def hello: Unit = 
  if debugLogEnabled then
    println("Hello world!")
```

`inline val debugLogEnabled = true`はインライン定数であり、`debugLogEnabled`への参照は全てコンパイル時に評価された値となります。
つまり`debugLogEnabled`という名前の変数はコンパイル時に評価が行われて、右辺のtrueに置き換えられるようになります。

上記コードをコンパイルすると`debugLogEnabled`は右辺値であるtrueに置き換えられるので、コンパイル時には以下のようになります。

```scala 3
@main def hello: Unit = 
  if true then // debugLogEnabledは変数への参照ではなく定数である値に置き換えられる
    println("Hello world!")
```

最終的にはコンパイラによって、true値は定数であるため、if条件は単純化されて以下のようになります。

```scala 3
@main def hello: Unit = 
  println("Hello world!")
```

`inline`修飾子を使用したインライン化は、このようにコンパイル時に評価を行うことによってコードを最適化します。

## compiletimeパッケージ

Scala 3では新しいパッケージscala.compiletimeが導入され、インラインコードを評価してコンパイル時にユーザー独自のエラー生成や条件判定ができるようになりました。

https://docs.scala-lang.org/scala3/reference/metaprogramming/compiletime-ops.html

### constValue/constValueOpt

`constValue`は型によって表される定数値を生成する関数であり、型が定数型でない場合はコンパイル時にエラーとなります。

つまり型パラメーターから実際に使用できる値を生成できるということです。

以下に定義されている`showTypeParamInt`は型パラメーターでInt型を受け取りIntを返す関数です。
`scala.compiletime.constValue[N]`を見ると受け取った型パラメーターを`constValue`に渡しそのまま戻り値として返しています。

```scala 3
inline def showTypeParamInt[N <: Int]: Int = scala.compiletime.constValue[N]
```

`showTypeParamInt`を呼び出すと以下のようになります。

```shell
scala> showTypeParamInt[1]
val res1: Int = 1
                                                                                                                                                                                                                                                                                                        
scala> showTypeParamInt[2]
val res2: Int = 2
                                                                                                                                                                                                                                                                                                        
scala> showTypeParamInt[4]
val res3: Int = 4
```

`constValue`は定数値しか受け取ることができないため定数値ではない値を渡すとコンパイルエラーとなります。

```shell
scala> val int: Int = 1
val int: Int = 1
                                                                                                                                                                                                                                                                                                        
scala> showInt[int.type]
-- Error: ----------------------------------------------------------------------
1 |showInt[int.type]
  |^^^^^^^^^^^^^^^^^
  |not a constant type: (int : Int); cannot take constValue
  |-----------------------------------------------------------------------------
  |Inline stack trace
  |- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  |This location contains code that was inlined from rs$line$7:1
1 |inline def showInt[N <: Int]: Int = scala.compiletime.constValue[N]
  |                                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   -----------------------------------------------------------------------------
1 error found
```

`constValueOpt`は`constValue`と同じように型によって表される定数値を生成する関数ですが、`constValue`と違い型が定数型でない場合はコンパイル時にエラーとはならずOption型であるNoneを返します。

先ほどの`showTypeParamInt`を`constValueOpt`に変更してみると戻り値がOption型になっていることがわかります。

```scala 3
inline def showTypeParamInt[N <: Int]: Option[Int] = scala.compiletime.constValueOpt[N]
```

再度`showTypeParamInt`を呼び出すと以下のようになります。

```shell
scala> showTypeParamInt[4]
val res4: Option[Int] = Some(4)
```

先ほどと同じように定数値ではない値を渡した場合はNoneが返却されます。

```shell
scala> showTypeParamInt[int.type]
val res5: Option[Int] = None
```

`constValue/constValueOpt`を使用することで型のみでプログラミングを行えるようになりました。
このように型のみでプログラミングを行うことを型レベル関数と言います。

`constValue`を使用することで定数値である型パラメーターを値として扱えるようになったので、値をそのまま返すだけではなくパターンマッチングを型だけで行えるようにもなりました。

以下は`constValue`を使用して月にマッチする数字が型パラメーターに渡された時にその英名を返すようなメソッドです。

```scala 3
inline def renameMonth[N <: Int]: String =
  inline scala.compiletime.constValue[N] match
    case 1  => "January"
    case 2  => "February"
    case 3  => "March"
    case 4  => "April"
    case 5  => "May"
    case 6  => "June"
    case 7  => "July"
    case 8  => "August"
    case 9  => "September"
    case 10 => "October"
    case 11 => "November"
    case 12 => "December"
```

対応する定数値を渡すとそれぞれの月にマッチした英名が返却されます。

```shell
scala> renameMonth[1]
val res1: String = January

scala> renameMonth[12]
val res2: String = December
```

定数値を渡してもパターンマッチに存在しない値であった場合コンパイルエラーとなります。

```shell
scala> renameMonth[13]
-- Error: ----------------------------------------------------------------------
 1 |renameMonth[13]
   |^^^^^^^^^^^^^^^
   |cannot reduce inline match with
   | scrutinee:  13 : (13 : Int)
   | patterns :  case 1
   |             case 2
   |             case 3
   |             case 4
   |             case 5
   |             case 6
   |             case 7
   |             case 8
   |             case 9
   |             case 10
   |             case 11
   |             case 12
   |----------------------------------------------------------------------------
   |Inline stack trace
   |- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   |This location contains code that was inlined from rs$line$3:2
 2 |  inline scala.compiletime.constValue[N] match
   |         ^
 3 |    case 1  => "January"
 4 |    case 2  => "February"
 5 |    case 3  => "March"
 6 |    case 4  => "April"
 7 |    case 5  => "May"
 8 |    case 6  => "June"
 9 |    case 7  => "July"
10 |    case 8  => "August"
11 |    case 9  => "September"
12 |    case 10 => "October"
13 |    case 11 => "November"
14 |    case 12 => "December"
    ----------------------------------------------------------------------------
1 error found
```

### erasedValue

`erasedValue`は先ほどの`constValue`とは違い渡された型をあたかも値として存在するかのように振る舞うことができる関数です。

どういうことかというとScalaでは渡された型でパターンマッチングなどの条件分岐を行いたい場合実際に値を使用しなければいけませんでした。
しかし`erasedValue`を使うことで、値を使用することなく型のみで条件分岐を行い、条件分岐に応じた処理の結果を返すことができるようになります。

例えば受け取った型の情報がOptionかどうかを判定する場合に`erasedValue`が存在する場合としない場合でコードを見比べてみます。

`erasedValue`が存在しない場合

```scala 3
def isOptional[T](value: T): Boolean =
  value match
    case _: Option[?] => true
    case _            => false
```

型がOptionかどうか判定するためには実際に値を渡さなければ判定することはできません。

```shell
scala> isOptional[Int](1)
val res1: Boolean = false
                                                                                                                                                                                                                                                                                                        
scala> isOptional[Option[Int]](Some(1))
val res2: Boolean = true
```

`erasedValue`が存在する場合

```shell
inline def isOptional[T]: Boolean =
  inline scala.compiletime.erasedValue[T] match
    case _: Option[?] => true
    case _            => false
```

型レベル関数で評価が行われるので実際に値を渡すことなく型のみで判定を行うことができます。

```shell
scala> isOptional[Int]
val res3: Boolean = false

scala> isOptional[Option[Int]]
val res4: Boolean = true
```

:::message
`erasedValue`は渡された型をあたかも値として存在するかのように振る舞うという特性であるため値を使用するとエラーになるということに注意する必要があります。
```scala 3
inline def isOptional[T]: Boolean =
  inline scala.compiletime.erasedValue[T] match
    case v: Option[?] =>
      println(v) // 値として使用する
      true
    case _ => false
```

呼び出すとエラーとなる。

```shell
scala> isOptional[Option[Int]]
-- Error: ----------------------------------------------------------------------
1 |isOptional[Option[Int]]
  |^^^^^^^^^^^^^^^^^^^^^^^
  |method erasedValue is declared as `erased`, but is in fact used
  |-----------------------------------------------------------------------------
  |Inline stack trace
  |- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  |This location contains code that was inlined from rs$line$25:2
2 |  inline scala.compiletime.erasedValue[T] match
  |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   -----------------------------------------------------------------------------
1 error found
```
:::

筆者はScala 3のUnion型を使用して、標準の型もしくはそのOption型のどちらかをとる型パラメーターの関数をOSSの開発時に使用しており、Option型であればOption型であるというプロパティを持つようなモデルを作成していました。
しかし、そのモデルは値を持つことはなく型パラメーターのみ受け取るようなものでした。その場合だと以下のように標準の型を受け取ってモデルを生成する関数とOption型を受け取ってモデルを生成する関数を2つ用意しないといけませんでした。
1つや2つであればこれでもよかったのですが、このようなものがいくつもあったり他に型パラメーターだけで判定を行わなければいけない箇所があったので結構同じようなコードが増えて大変でした。

```scala 3
def INT[T <: Int | Long]: Integer[T] = Integer(None, false)
def INT[T <: Option[Int | Long]]: Integer[T] = Integer(None, true)
```

`erasedValue`を知ってからは単純にコードが半分になったり表現できる幅が増えたりとすごく重宝してます。

```scala 3
inline def INT[T <: Int | Long | Option[Int | Long]]: Integer[T] = Integer(None, isOptional[T])
```

公式サンプルコードの`defaultValue`の方が実際に使用する場面をイメージしやすいと思います。
https://docs.scala-lang.org/scala3/reference/metaprogramming/compiletime-ops.html

### error/codeOf

`error`はインライン展開時にユーザー独自のエラーメッセージを発生させることができるものです。

```scala 3
inline def fail() = scala.compiletime.error("failed for a reason")
```

呼び出してみると「failed for a reason」という自身で定義したエラーメッセージが確認できます。

```shell
scala> fail()
-- Error: ----------------------------------------------------------------------
1 |fail()
  |^^^^^^
  |failed for a reason
1 error found
```

`constValue`の時に作成した関数に`error`を導入して独自のエラーメッセージを表示してみます。

```scala 3
inline def renameMonth[N <: Int]: String =
  inline scala.compiletime.constValue[N] match
    case 1  => "January"
    case 2  => "February"
    case 3  => "March"
    case 4  => "April"
    case 5  => "May"
    case 6  => "June"
    case 7  => "July"
    case 8  => "August"
    case 9  => "September"
    case 10 => "October"
    case 11 => "November"
    case 12 => "December"
```

これは月にマッチする数字が型パラメーターに渡された時にその英名を返す関数でした。
この関数にパターンマッチにマッチしない定数を渡すとエラーにすることはできましたが、そのエラーメッセージは少しわかりにくいものとなっていました。

:::details エラーメッセージ
```shell
scala> renameMonth[13]
-- Error: ----------------------------------------------------------------------
 1 |renameMonth[13]
   |^^^^^^^^^^^^^^^
   |cannot reduce inline match with
   | scrutinee:  13 : (13 : Int)
   | patterns :  case 1
   |             case 2
   |             case 3
   |             case 4
   |             case 5
   |             case 6
   |             case 7
   |             case 8
   |             case 9
   |             case 10
   |             case 11
   |             case 12
   |----------------------------------------------------------------------------
   |Inline stack trace
   |- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   |This location contains code that was inlined from rs$line$3:2
 2 |  inline scala.compiletime.constValue[N] match
   |         ^
 3 |    case 1  => "January"
 4 |    case 2  => "February"
 5 |    case 3  => "March"
 6 |    case 4  => "April"
 7 |    case 5  => "May"
 8 |    case 6  => "June"
 9 |    case 7  => "July"
10 |    case 8  => "August"
11 |    case 9  => "September"
12 |    case 10 => "October"
13 |    case 11 => "November"
14 |    case 12 => "December"
    ----------------------------------------------------------------------------
1 error found
```
:::

この関数のエラー文を`error`を使用して見やすくします。
パターンマッチングでどれにも一致しない場合に`error`を使用するよう変更を加えます。

```scala 3
inline def renameMonth[N <: Int]: String =
  inline scala.compiletime.constValue[N] match
    case 1  => "January"
    case 2  => "February"
    case 3  => "March"
    case 4  => "April"
    case 5  => "May"
    case 6  => "June"
    case 7  => "July"
    case 8  => "August"
    case 9  => "September"
    case 10 => "October"
    case 11 => "November"
    case 12 => "December"
    case _  => scala.compiletime.error("月は1 ~ 12までの範囲しかありません。")
```

定数値を渡してもパターンマッチに存在しない値であった場合コンパイルエラーとなり、独自のエラーメッセージが表示されるようになったことで非常にわかりやすくなりました。

```shell
scala> renameMonth[13]
-- Error: ----------------------------------------------------------------------
1 |renameMonth[13]
  |^^^^^^^^^^^^^^^
  |月は1 ~ 12までの範囲しかありません。
1 error found
```

`error`はこのようにコンパイル時に独自のエラーを表示することができるので、今まで以上に型安全なコードを書けるようになったのではないでしょうか？
少し用途は違うかもしれませんが、Scala 2で使用していた`require`などはコンパイル時ではなく実行時に例外を吐くだけであったため、定数を取り扱っている場合であれば`error`に載せ替えることでより型安全にできるかもしれません。

`error`は受け取るメッセージが定数でなければいけません。そのため動的に変わるようなメッセージを出すことができないのです。
先ほどのコードでパターンマッチングに一致しない場合にその値がなんであったかをエラーに含めたい時があると思います。

```scala 3
inline def renameMonth[N <: Int]: String =
  inline scala.compiletime.constValue[N] match
    case 1  => "January"
    case 2  => "February"
    case 3  => "March"
    case 4  => "April"
    case 5  => "May"
    case 6  => "June"
    case 7  => "July"
    case 8  => "August"
    case 9  => "September"
    case 10 => "October"
    case 11 => "November"
    case 12 => "December"
    case v  => scala.compiletime.error(s"$vは1 ~ 12のどれにも一致しません") // このように一致しない値をエラーに含めたい
```

しかし、`error`は受け取るメッセージが定数でなければいけないためこのコードのエラーメッセージは意図したものとはなりません。

```shell
scala> renameMonth[13]
-- Error: ----------------------------------------------------------------------
1 |renameMonth[13]
  |^^^^^^^^^^^^^^^
  |A literal string is expected as an argument to `compiletime.error`. Got v.+("は1 ~ 12のどれにも一致しません")
1 error found
```

エラーメッセージを動的なものにしたい場合は、`codeOf`を使用すると実現することができます。

```scala 3
inline def renameMonth[N <: Int]: String =
  inline scala.compiletime.constValue[N] match
    case 1  => "January"
    case 2  => "February"
    case 3  => "March"
    case 4  => "April"
    case 5  => "May"
    case 6  => "June"
    case 7  => "July"
    case 8  => "August"
    case 9  => "September"
    case 10 => "October"
    case 11 => "November"
    case 12 => "December"
    case v  => scala.compiletime.error(scala.compiletime.codeOf(v) + "は1 ~ 12のどれにも一致しません")
```

ただ`codeOf`は受け取ったコードをそのまま表示する？だけのようなので上記だと受け取った`v`はそのまま`v`として表示されてしまいます。

```shell
scala> renameMonth[13]
-- Error: ----------------------------------------------------------------------
1 |renameMonth[13]
  |^^^^^^^^^^^^^^^
  |vは1 ~ 12のどれにも一致しません
1 error found
```

そのため以下のようにすることで求めていたエラーメッセージを表示することができるようになります。

```scala 3
case _ => scala.compiletime.error(scala.compiletime.codeOf(scala.compiletime.constValue[N]) + "は1 ~ 12のどれにも一致しません")
```

```shell
scala> renameMonth[13]
-- Error: ----------------------------------------------------------------------
1 |renameMonth[13]
  |^^^^^^^^^^^^^^^
  |13は1 ~ 12のどれにも一致しません
1 error found
```

`codeOf`に関しては公式ドキュメントなどでもあまり詳しく記載がなかったのでここら辺の挙動は理解できておりません。
有識者の方がいれば教えていただけると嬉しいです。

### ops

## パターンマッチングと条件分岐

### inline条件
### inlineマッチング

## まとめ

## 参考文献

https://docs.scala-lang.org/ja/scala3/new-in-scala3.html
https://docs.scala-lang.org/scala3/guides/migration/compatibility-metaprogramming.html
https://www.baeldung.com/scala/inline-modifier
https://www.educative.io/answers/what-is-inline-in-scala-3
