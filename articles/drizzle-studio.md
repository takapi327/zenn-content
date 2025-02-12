---
title: "Drizzle Studio"
emoji: "🔥"
type: "tech"
topics: ["drizzleorm", "drizzlestudio", "Typescitpt", "MySQL"]
publication_name: nextbeat
published: false
---

## はじめに

TypeScriptでデータベース操作を行うためのライブラリであるDrizzle ORMを気に入って使用しているのですが、Drizzle ORMにはGUIでDBの操作ができるDrizzle Studioというツールがあります。

今まではなんやかんや開発時にDBに対して直接操作をしたい場合は、使い慣れたMySQL ShellやWorkbenchを使用していました。しかし、Drizzle Studioを使用することでGUIでDBのレコード操作だけではなくスキーマの編集等もできるということを最近知ったため試しに使ってみました。

使ってみると思ったよりも使い勝手がよかったので、今回はDrizzle Studioについて紹介したいと思います。

:::message
今回はDrizzle ORM自体の紹介は軽く触れる程度です。詳細な内容や構築方法については公式ドキュメントや他の方が書いたブログ等を参照してください。
:::

## Drizzle ORM

Drizzle ORMは、TypeScriptでデータベース操作を行う際の優れた選択肢として注目を集めています。その最大の魅力は型安全性と開発体験の高さにあります。開発者はスキーマから自動的に生成される型定義により、コーディング時からデータベース操作の誤りを防ぐことができます。

https://orm.drizzle.team/

個人的にはDrizzle ORMのAPI設計がSQLに近い形で設計されている点が気に入っています。 クエリビルダーの構文がSQL文に非常に近い形で設計されているため、学習コストを抑えながら直感的にコードを書くことができスキーマ管理やクエリ構築などもSQLの知識をそのまま活かして実装できます。
また、SQLに近い形で設計されているためAIによるコード生成やコード補完が効きやすく、開発効率を向上させることができる点も嬉しい点です。

**テーブルスキーマの例**

```typescript
export const city = mysqlTable('city', {
  id: int().notNull().primaryKey().autoincrement(),
  name: char({ length: 35 }).notNull().default(''),
  countryCode: char({ length: 3 }).notNull().default('').references(() => country.code),
  district: char({ length: 20 }).notNull().default(''),
  population: int().notNull().default(0),
})
```

**データ取得の例**

```typescript
const cities = await db.select().from(city).where(eq(city.name, 'Tokyo'))
```

**データ挿入の例**

```typescript
await db.insert(city).values({
  name: 'Tokyo',
  countryCode: 'JPN',
  district: 'Kanto',
  population: 13929286,
})
```

> If you know SQL — you know Drizzle.

公式が「SQLを知っていれば、Drizzleを知っている。」と言うようにSQLを知っていればDrizzleのAPIもすぐに使いこなせると思います。

更にDrizzleのスキーマを定義することでzodのバリデーションスキーマが生成される点も最近のzodの流行に合わせているため、開発者にとっては非常に使いやすいライブラリだと思います。

Drizzle ORMに関しては公式ドキュメントだけでなく、youtubeにも紹介動画があるため見てみると良いかもしれません。

https://youtu.be/i_mAHOhpBSA

## Drizzle Studio

Drizzle Studioは、GUIでDBに対して操作をすることができるツールです。Drizzle Studioを使用することでスキーマの追加や削除、データの追加や削除、クエリの実行などがGUIで行うことができます。

起動自体は簡単で、Drizzleの設定さえあれば以下のコマンドだけで起動し操作することができます。

```bash
npx drizzle-kit studio
```

起動すると以下のような画面が表示されます。サイドバーにはテーブル一覧が表示され、テーブルを選択することでテーブルのデータを一覧で表示することができます。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO.png)

レコードの追加は指定したカラムのフィールドに値を入力するだけで簡単に追加できます。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_ADD.png)

個人的に良かった点は外部キー制約がついているレコードがJOINなどをしなくても表示することができる点です。

外部キーが設定されている場合、テーブルのセルに関連するテーブル名が表示され、クリックすることで関連するテーブルのデータを表示することができます。 画像で言うと「cities」と「countrylanguages」が対象のセルです。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_FOREIGN_KEY.png)

以下だと国 (`country`) テーブルに関連する都市 (`city`) テーブルのデータを一覧で表示しています。表示自体はアコーディオンのような形式でクリックすることで表示を切り替えることができます。これによって見たいデータだけを表示することができるため、見やすくなっています。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_JOIN.png)

他のツールだとJOINを書く必要がありますしJOINを行うことでレコードはフラットに表示されるためどのテーブルに属しているのかがわかりにくいですが、Drizzle Studioであれば完全に分離されているためどのテーブルに属しているのかが一目でわかります。

表示しているテーブルを直接表示 (メインとして表示) することもできるため、データを追う際にも便利です。

レコードのフィルタリングに関しても画面を操作するだけで簡単に行うことができます。条件文もあらかじめ用意してくれているため、条件を選択するだけでフィルタリングができる点も使いやすいです。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_FILTER.png)

Drizzle Studioはテーブルのレコード一覧とスキーマ表示を切り替えることができます。スキーマを表示に切り替えると、スキーマに関連するいくつかの操作が表示されるので任意の操作を行うことができます。

これによって、スキーマの変更に関してもGUIで行うことができるため開発者はSQLを書くことなくスキーマの変更を行うことができます。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_SCHEMA.png)

例えば、テーブルに新しいカラムを追加したい場合以下のような操作で追加処理を行うことができます。

まずは追加するカラム名を入力します。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_SET_COLUMN_NAME.png)

次にカラムの型を選択します。カラムの方は選択肢から選択することができます。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_SET_COLUMN_TYPE.png)

次にカラムの制約を設定します。カラムの制約は選択肢からラジオボタンで選択することができます。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_SET_COLUMN_CONSTRAINT.png)

必要であればカラムのデフォルト値を設定することもできます。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_SET_COLUMN_DEFAULT.png)

最後に「Review and commit」ボタンをクリックすることで実際にカラム追加を行うALTER文が表示されます。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_REVIEW.png)

実行されるALTER文を確認し、問題がなければ「Commit changes」ボタンをクリックすることでカラム追加を反映することができます。

削除する場合も簡単でカラムを選択して「Delete selected」ボタンをクリックするだけで削除ができます。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_DELETE.png)

ただし、この方法で変更したスキーマはアプリケーションコードには反映されないため、スキーマの変更を行った場合はアプリケーションコードにも反映する必要があります。

カラムの追加・削除だけではなくテーブルおよびビュー自体の作成や削除も同様の操作で行うことができます。Truncateなどもボタンクリックで簡単に行うことができるので便利ですね。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_CREATE_TABLE.png)

また、直接SQLを書いて処理を行いたい場合は「SQL runner」を選択することで任意のSQLを実行することができます。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_SQL.png)

Drizzle Studioはこれに加えてGUI上でDrizzle ORMのスキーマを使用してクエリを実行することもできます。そのため、Studio上でコードを試して動作を確認しそのままアプリケーションコードに反映することができます。

以下はDrizzle ORMのスキーマを使用してクエリを実行する例です。アプリケーション開発時にクエリ実行の部分だけ先に動作検証できるのはとても便利で良いですね。

![Drizzle Studio](/images/drizzle-studio/DRIZZLE_STUDIO_QUERY.png)

これだけの機能がライブラリに付属して使えるのは非常にありがたいです。

ただし、Drizzle Studioは現在ベータ版であるため意図しない動作やバグが発生する可能性がある点に関しては注意が必要です。

## まとめ

Drizzle Studio自体を触るのは初めてでしたが、使い勝手がよくGUIでスキーマの操作ができるため非常に便利だと感じました。

また、今まで別々のツールを用意したり設定したりする必要がありましたが、Drizzle Studioを使用することでアプリケーションで使用しているものと同じ環境と設定でスキーマの操作ができるため手間が省ける点も良かったです。

> We’re going to massively improve and extend Drizzle Studio in the upcoming months!

現在ベータ版ということもあり公式によると今後もDrizzle Studioは改善されていくとのことなので今後のアップデートが楽しみです。

個人的にはDrizzle Studio上で行なったスキーマ変更をアプリケーションコードに反映する機能があればなあと思ったので今後のアップデートに期待しています。ここら辺はDrizzle ORMのマイグレーション機能と連携することで実現できるのかもしれません。近いうちにこちらも試してみようかなと思います。

Drizzle StudioはDrizzle ORMを使用している方にはおすすめのツールだと思います。Drizzle ORMを使用している方はぜひ一度使ってみてください。
