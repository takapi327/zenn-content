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

Laikaは、Scala開発者のための強力で柔軟なドキュメント生成ツールです。軽量マークアップ言語を変換し、サイトやe-bookを生成する多機能なツールキットとして設計されています。

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
