---
title: "Scodecでパフォーマンスに問題があると感じたらこう使え！"
emoji: "♦️"
type: "tech"
topics: ["Scala", "scodec"]
publication_name: nextbeat
published: false
---

# はじめに

自分の作成しているライブラリでパフォーマンスに問題があり、その調査をしていると[scodec](https://github.com/scodec/scodec)の使用方法で大きくパフォーマンスに違いが現れるということがわかりました。

そのため、今回は同じようにscodecを使用していてパフォーマンスに課題を感じた場合どのように対処すればよいかについて紹介したいと思います。

この記事は、Scalaのバイナリデータを扱うためのライブラリであるscodecの使い方やパフォーマンスに関する内容を解説しています。scodecを使ってバイナリデータを効率的に処理する方法について知りたい方は、ぜひ参考にしてください。

:::message
パフォーマンスに違いがあると記載していますが、これは`ms`単位での差異です。自分のライブラリはデータベースのデータを扱っており、大量のデータを処理するためにパフォーマンスが重要となります。そのため、ms単位での差異が大きくなると、処理時間が大幅に増加してしまいます。
他の用途であればms単位での差異は問題にならないかもしれません。
:::

# scodecとは

scodecはバイナリデータを扱うためのScalaライブラリです。ビットやバイトを扱うためのシンプルで高性能なデータ構造から、ストリーミングのエンコードやデコードまで幅広くサポートしているTypelevelのプロジェクトです。

https://github.com/scodec/scodec

https://typelevel.org/

## 基本の使い方

scodecは主に3つのパッケージで構成されています。

- `scodec-bits`: バイナリを扱うための永続的なデータ構造、scodecのメインとなるBitVectorとByteVectorを提供するライブラリ。
- `scodec-core`: BitVectorとByteVectorからのエンコーダとデコーダを定義するためのライブラリ。
- `scodec-stream`: ストリーミングのエンコードとデコードをサポートするためのライブラリ。

scodecを使うためには、まず`scodec-bits`を使ってビットやバイトのデータ構造を作成し、その後`scodec-core`を使ってエンコーダとデコーダを定義します。

scodecには`ByteVector`と`BitVector`という2つの主要なデータ構造があります。どちらも不変のコレクションで、他のscodecモジュールで使用するために最適化された性能特性を持っています。scodecモジュールは、これらのデータ構造を使用してビットやバイトのデータをエンコード・デコード操作するための関数を提供します。

scodecモジュールを使用しなくても汎用的に使用できるように設計されているため、例えばByteVectorは不変バイト配列の代わりとして安全に使用したりもできます。


scodecを使うためには、以下の依存関係を`build.sbt`に追加します。

```sbt
libraryDependencies += "org.scodec" %% "scodec-bits" % "1.2.1"
```

Scala CLIなどでは以下のように依存関係を追加します。

```shell
//> using dep org.scodec::scodec-bits:1.2.1
```

**ByteVector**

ByteVector型は名前から想像できるように`scala.collection.immutable.Vector[Byte]`と同型ですが、より優れたパフォーマンス特性を持ちます。ByteVectorは、チャンクからなるバランスの取れたバイナリツリーとして表現されています。

以下は、ByteVectorを構築する基本的な使い方の例です。

```scala 3
import scodec.bits.*

val hexString = "0000000c48656c6c6f20576f726c6421" // "Hello World!"
val byteVector = ByteVector.fromValidHex(hexString)

println(byteVector)
// ByteVector(16 bytes, 0x0000000c48656c6c6f20576f726c6421)
```

上記の例では、16バイトのバイナリデータを16進数の文字列から作成しています。`fromValidHex`メソッドを使うことで、16進数の文字列をバイトベクトルに変換することができます。

このByteVectorを使って、バイト列から特定のデータを取り出すことができます。以下は、バイト列から長さと文字列を取り出す例です。

```scala 3
// Extract the length (first 4 bytes)
val length = byteVector.take(4).toInt(signed = false)

// Extract the string (remaining bytes)
val stringBytes = byteVector.drop(4)
val decodedString = stringBytes.decodeUtf8.right.get

println(s"Length: $length, String: $decodedString")
// Output: Length: 12, String: Hello World!
```

この例では、バイト列から長さと文字列を取り出しています。`take`メソッドを使ってバイト列の先頭から4バイトを取り出し、`toInt`メソッドを使って整数に変換しています。また、`drop`メソッドを使って残りのバイト列を取り出し、`decodeUtf8`メソッドを使ってUTF-8でデコードしています。

**BitVector**

BitVector型はByteVectorに似ていますが、バイトの代わりにビットをインデックスする点が異なっています。これによって、特定のビットへのアクセスと更新、および8で均等に割り切れないビット数の保存が可能になります。

以下は、BitVectorの基本的な使い方の例です。

```scala 3
import scodec.bits.*

val hexString = "0000000c48656c6c6f20576f726c6421" // "Hello World!"
val byteVector = ByteVector.BitVector(hexString)

println(bitVector)
// BitVector(128 bits, 0x0000000c48656c6c6f20576f726c6421)
```

上記の例では、128ビットのバイナリデータを16進数の文字列から作成しています。`BitVector`コンパニオンオブジェクトの`apply`メソッドを使うことで、16進数の文字列をビットベクトルに変換することができます。

このBitVectorを使って、ビット列から特定のデータを取り出すことができます。以下は、ビット列から長さと文字列を取り出す例です。

```scala 3
// Extract the length (first 32 bits)
val (lengthBits, stringBits) = bitVector.splitAt(32)
val length = lengthBits.toInt(signed = false)

// Extract the string (remaining bits)
val decodedString = stringBits.bytes.decodeUtf8.right.get

println(s"Length: $length, String: $decodedString")
// Output: Length: 12, String: Hello World!
```

`ByteVector`と`BitVector`は、それぞれ変換することができます。`ByteVector`から`BitVector`に変換するには、`bits`メソッドを使います。逆に`BitVector`から`ByteVector`に変換するには、`bytes`メソッドを使います。

```scala 3
val bitVector = byteVector.bits
println(bitVector)
// BitVector(128 bits, 0x0000000c48656c6c6f20576f726c6421)

// Convert BitVector to ByteVector
val convertedByteVector = bitVector.bytes
println(convertedByteVector)
// ByteVector(16 bytes, 0x0000000c48656c6c6f20576f726c6421)
```

このように、scodecを使ってバイナリデータを効率的に処理することができます。

### Scodec Core

scodecでバイナリデータの処理方法が分かったと思いますが、特定の位置にあるデータを切り出して処理をするのは色々とめんどくさいです。

scodecには先ほど紹介したようにエンコーダとデコーダを定義するための`scodec-core`があります。これを使うことで、バイナリデータを簡単にエンコード・デコードすることができます。

以下は、基本的な使い方の例です。

```scala 3
def decode(): Attempt[DecodeResult[String]] =
  (for
    length <- uint32
    str <- bytes(length.toInt)
  yield str.decodeUtf8Lenient).decode(bitVector)
```

この例では、`Hello World!`という文字列をエンコードしてデコードしています。まず、文字列の長さをビットベクトルにエンコードし、その後文字列をUTF-8でエンコードしています。最後に、文字列の長さと文字列を連結してビットベクトルを作成し、エンコードとデコードを行っています。

このように、`scodec-core`を使うことで簡単にデコード処理を行うことができます。`map`や`flatMap`を使って組み合わせられるのも使い勝手が良いですね。

また、この方法ではバリデーションも行なっており戻り値は`Attempt`型となるため、データが正しくデコードされているかどうかを確認することもできます。

```scala 3
decode() match
  case Attempt.Successful(DecodeResult(value, remainder)) =>
    println(s"Decoded: $value, Remaining: $remainder")
  case Attempt.Failure(cause) =>
    println(s"Failed to decode: $cause")
```

このように、scodecを使ってバイナリデータを効率的に処理することができます。

## パフォーマンスに問題がある？

`scodec-core`を使うと簡単にデコード処理を行うことができますが、実際にはパフォーマンスに問題がある場合があります。どれぐらいの差があるのかは後続のベンチマークで調査しますが、パフォーマンスに問題がある場合どのように対処すればよいかについて紹介します。

結論から言うと`scodec-core`を使っていてパフォーマンスに問題があるなと感じたら、`scodec-core`の代わりに`scodec-bits`をそのまま使うことでパフォーマンスを向上させることができます。

先ほどの例で`scodec-core`を使ってデコード処理を行っていましたが、同じ処理を`scodec-bits`だけで行うと以下のようになります。

```diff
def decode(): String =
-  (for
-    length <- uint32
-    str <- bytes(length.toInt)
-  yield str.decodeUtf8Lenient).decode(bits).require.value
+  val (lengthBits, postLengthBits) = bits.splitAt(32)
+  val length = lengthBits.toInt(false)
+  val strBits = postLengthBits.take(length * 8L)
+  new String(strBits.toByteArray, "UTF-8")
```

この方法は、`scodec-core`を使うよりもパフォーマンスが向上します。

`map`や`flatMap`を使っているからその分遅くなっているのでは？と思われるかもしれませんが、以下のように`for`式を使わずに`scodec-core`を使ってデコード処理を行っても同じように`scodec-bits`を使うよりも遅くなります。

```scala 3
def decode(): String =
  val decoded = uint32.decode(bits).require
  val length = decoded.value
  val str = bytes(length).decode(decoded.remainder).require.value
  new String(str.toArray, "UTF-8”)
```

つまり、`scodec-core`を使っている場合`map`や`flatMap`を使っているから遅くなるというわけではなく、内部での処理が遅いためにパフォーマンスが低下していると考えられます。

## ベンチマーク

パフォーマンスに問題がある場合、`scodec-core`の代わりに`scodec-bits`をそのまま使うことでパフォーマンスを向上させることができると記載しましたが、実際にはどれぐらいの差があるのでしょうか？

以下は、`scodec-core`と`scodec-bits`のパフォーマンスを比較した例です。

単純な実行時間を計測するために、以下のコードで比較を行いました。bitsは上記の例で作成した「Hello World!」のビットベクトルです。

```scala 3
def decodeByCodecs(): String =
  (for
    length <- uint8
    str <- bytes(length)
  yield new String(str.toArray, "UTF-8")).decode(bits).require.value

def decodeBySplitAt(): String =
  val (lengthBits, postLengthBits) = bits.splitAt(8)
  val length = lengthBits.toInt(false)
  val strBits = postLengthBits.take(length * 8L)
  new String(strBits.toByteArray, "UTF-8")
```

:::message
単純なデコード速度を見るために文字列変換は同じ処理方法にしています。
:::

全体のコードは以下Gistにあります。

https://gist.github.com/takapi327/233176b326b8019e55bf6bb1c289987c#file-benchmark-sc

上記ベンチマークの結果は以下の通りで、およそ2倍の差が出ました。

```shell
scodec-core: 15 ms
scodec-bits: 6 ms
```

[@xuwei-k](https://x.com/xuwei_k?ref_src=twsrc%5Egoogle%7Ctwcamp%5Eserp%7Ctwgr%5Eauthor)さんがプロファイリングしてくれた結果を見ても、`scodec-core`のコードを使用するとおよそ2倍の差が出ることがわかりました。

```shell
scodec-core: 61.22%
scodec-bits: 29.25%
```

https://gist.github.com/xuwei-k/d3fda077d85ef70a643375777b2914a8

# まとめ

今回は、scodecを使っていてパフォーマンスに問題がある場合どのように対処すればよいかについて紹介しました。

ベンチマークを取ると`scodec-core`を使っている場合、`scodec-bits`をそのまま使うことでおよそ2倍の差が出ることがわかりました。流石に2倍の差が出るとは思いませんでしたが、ベンチマークを取得してみると意外な結果が出ることもありますね。

`scodec-bits`をそのまま使うことで今回遭遇したパフォーマンスの問題を解決することができましたが、`scodec-bits`をそのまま使う方法は`scodec-core`を使う方法に比べると読み取ったバイト列の管理など少し冗長になってしまいます。

個人的には`scodec-core`を使う方法の方が好きなので、余裕ができたら調査して改善できるといいなと思っています。

scodecを使っていてパフォーマンスに問題がある場合、`scodec-core`を使っているなら`scodec-bits`をそのまま使うことでパフォーマンスを向上させることができるので、ぜひ参考にしてみてください。

それでは、Scalaとscodecを使ってバイナリデータを効率的に処理していきましょう！
