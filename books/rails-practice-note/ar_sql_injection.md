---
title: "[ActiveRecord] SQLインジェクションを防ぐ"
---

# [ActiveRecord] SQLインジェクションを防ぐ

## SQLインジェクションとは

SQLインジェクションは、外部から文字列を送り込んで攻撃者の意図するSQLを実行する攻撃です。

例としてpackages(荷物)テーブルに16桁英数字のcode(荷物コード)カラムがあり、荷物コードを知っている荷物を検索して情報を表示できるとします。

packagesテーブルをcodeで検索する次のようなコードがコントローラにあり、`params[:code]`をリクエストで送信可能だとします。

```ruby
Package.where("code = '#{params[:code]}'")
# このコードには問題があります
```

想定した動作では次のようなSQLがつくられます。

```sql
SELECT "packages".* FROM "packages" WHERE (code = 'R7aFIgWDrNwNRFL0')
```

ここで悪意のあるユーザーが `' OR 1) --` という文字列を入力すると、次のSQLがつくられます。

```sql
SELECT "packages".* FROM "packages" WHERE (code = '' OR 1) -- ')
```

SQLで `--` 以降はコメントとなるので、`WHERE code = '' OR 1` という必ず成立する条件になってしまいます。この結果、荷物コードを知らない全てのレコードを取得するSQLインジェクション攻撃に成功してしまいます。packagesテーブルに住所など公開してはいけない情報が含まれていれば、情報漏洩となります。

## SQLインジェクションを防ぐ

SQLインジェクションを防ぐためには、文字列をサニタイズ(無害化)する安全な書き方をつかいます。安全な書き方として、位置指定ハンドラや名前付きハンドラなどがあります。たとえば次のようなコード群は安全な書き方です。

```ruby
Package.where(code: params[:code]) # ActiveRecordの自然な書き方。一番読みやすい。
Package.where("code = ?", params[:code]) # 位置指定ハンドラ
Package.where("code = :code", {code: params[:code]}) # 名前付きハンドラ
```

たとえば `Package.where("code = ?", params[:code])` で生成されるSQLは、内部で文字列がサニタイズされて`'`が`''`に置き換わり攻撃を防ぐことができます。

```sql
SELECT "packages".* FROM "packages" WHERE (code = ''' OR 1) -- ')
```

また、SQL片を組み立てて文字列としてActiveRecordへ渡すケースでは特に注意が必要です。このケースでは、[各種サニタイズメソッド](https://api.rubyonrails.org/classes/ActiveRecord/Sanitization/ClassMethods.html)をつかって適切にサニタイズする必要があります。難易度が上がるので、可能な限りActiveRecordの安全な書き方をつかうのがお勧めです。

## LIKEには注意

ほとんどのケースで位置指定ハンドラや名前付きハンドラをつかうことでサニタイズが行われますが、LIKE演算子のケースは例外で自動的にはサニタイズが行われません。正確には、WHERE用のサニタイズはされるのですが、LIKE用のサニタイズが行われないという動作になります。

```ruby
Package.where("code like ?", "%" + params[:str] + "%")
# このコードではLIKE用のサニタイズがされない
```

LIKE用のサニタイズは`ActiveRecord::Base.sanitize_sql_like`メソッドをつかいます。

```ruby
Package.where("code like ?", "%" + ActiveRecord::Base.sanitize_sql_like(params[:str]) + "%")
```

LIKE用のサニタイズでは`%`と`_`がそれぞれ`\\%`と`\\_`に置き換わります。


- Rails API "ActiveRecord::Base.sanitize_sql_like": https://api.rubyonrails.org/classes/ActiveRecord/Sanitization/ClassMethods.html#method-i-sanitize_sql_like

## 参考資料

- Railsガイド 「Rails セキュリティガイド」: https://railsguides.jp/security.html#sql%E3%82%A4%E3%83%B3%E3%82%B8%E3%82%A7%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3
- Rails API 各種サニタイズメソッド: https://api.rubyonrails.org/classes/ActiveRecord/Sanitization/ClassMethods.html
