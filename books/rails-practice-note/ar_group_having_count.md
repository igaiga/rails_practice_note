---
title: "[ActiveRecord] group, having, count"
---

# [ActiveRecord] groupメソッド、havingメソッド、countメソッド

SQLで同じ値のデータ群をまとめるための道具としてgroup byがあります。ActiveRecordではgroupメソッドでつかうことができます。havingメソッド、countメソッドと組み合わせてつかうこともできます。

## groupメソッド

Bookモデルのtitleカラムでgroup byでまとめるには `Book.group(:title)` と書きます。

```ruby
Book.group(:title)
#=> SELECT "books".* FROM "books" GROUP BY "books"."title"
#=>
[#<Book:0x000000010b0ffc88
  id: 1,
  title: "book0",
  memo: nil,
  user_id: 1,
  created_at: Sat, 23 Apr 2022 23:55:17.478897000 UTC +00:00,
  updated_at: Sat, 23 Apr 2022 23:55:17.478897000 UTC +00:00>,
 #<Book:0x000000010b0ffbc0
  id: 2,
  title: "book1",
  memo: nil,
  user_id: 1,
  created_at: Sat, 23 Apr 2022 23:55:17.486179000 UTC +00:00,
  updated_at: Sat, 23 Apr 2022 23:55:17.486179000 UTC +00:00>,
...
```

booksテーブルの全カラムを取得する必要がないときは、pluckメソッドやselectメソッドで絞ることもできます。2つのメソッドの違いは「pluckとselectの違い」を参照してください。

```ruby
Book.group(:title).pluck(:title)
#=> SELECT "books"."title" FROM "books" GROUP BY "books"."title"
#=> ["book0", "book1", "book2", "book3", "book4"]
```

groupメソッドに複数カラムを指定してgroup byすることもできます。

```ruby
Book.group(:title, :memo)
SELECT "books".* FROM "books" GROUP BY "books"."title", "books"."memo"
```

## groupメソッドとhavingメソッド

group byの条件を加えるSQLのhaving句を書くにはgroupメソッドとhavingメソッドをつかいます。

```ruby
Book.having("SUM(price) > 1000").group("user_id")
#=> SELECT "books".* FROM "books" GROUP BY "books"."user_id" HAVING (SUM(price) > 1000)
```

## groupメソッドとcountメソッド

group byがよくつかわれるケースとして、同じ値のデータをまとめてそれを数えたいケースがあります。

booksテーブルの同じtitleの本がそれぞれ何冊ずつあるかを数えるには `Book.group(:title).count` と書きます。

```ruby
Book.group(:title).count
#=> SELECT COUNT(*) AS "count_all", "books"."title" AS "books_title" FROM "books" GROUP BY "books"."title"
#=> {"book0"=>2, "book1"=>2, "book2"=>3, "book3"=>1, "book4"=>1}```
```

SQLを見ると `AS "count_all"` が指定されているので、attributesとしてcount_allを扱うことができます。attributesについて詳しくは「selectとpluckの違い」を参照してください。

たとえば結果順でorderするときは次のように書けます。

```ruby
Book.group(:title).order(count_all: :desc).count
#=> SELECT COUNT(*) AS "count_all", "books"."title" AS "books_title" FROM "books" GROUP BY "books"."title" ORDER BY "count_all" DESC
#=> {"book2"=>3, "book1"=>2, "book0"=>2, "book4"=>1, "book3"=>1}

Book.group(:title).order(count_all: :desc).limit(3).count
#=> SELECT COUNT(*) AS "count_all", "books"."title" AS "books_title" FROM "books" GROUP BY "books"."title" ORDER BY "count_all" DESC LIMIT ?  [["LIMIT", 3]]
#=> {"book2"=>3, "book1"=>2, "book0"=>2}
```

## 資料

- Railsガイド「クエリインターフェイス」: https://railsguides.jp/active_record_querying.html
- groupメソッド: https://api.rubyonrails.org/classes/ActiveRecord/QueryMethods.html#method-i-group
- havingメソッド: https://api.rubyonrails.org/classes/ActiveRecord/QueryMethods.html#method-i-having
- countメソッド: https://api.rubyonrails.org/classes/ActiveRecord/Calculations.html#method-i-count

## サンプルコード

- https://github.com/igaiga/basics_of_active_record/tree/refs/tags/group_by
