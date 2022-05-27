---
title: "[ActiveRecord] mergeメソッド"
---

# [ActiveRecord] mergeメソッド

AR::ReleationオブジェクトにmergeメソッドをつかってAR::Releationを渡すと、AR::Releation同士を結合することができます。

次のように、Userが1対多でBookを持つような関連を考えます。

```ruby
class User < ApplicationRecord
  has_many :books 
end

class Book < ApplicationRecord
  belongs_to :user
  scope :ruby ->{ where(title: "Ruby") }
end
```

mergeメソッドにはAR::Relationを渡せるので、scopeを渡すこともできます。これらのコードは同じSQLを発行します。

```ruby
user.books.merge(Book.ruby)
user.books.merge(Book.where(title: "Ruby"))
user.books.ruby
user.books.where(title: "Ruby")
#=> SELECT "books".* FROM "books" WHERE "books"."user_id" = 1 AND "books"."title" = 'Ruby'
```

mergeメソッドが便利なのは、たとえば `user.books.merge(User.where(name: "igaiga"))` のように、user.booksの後ろにBookモデルではない別のモデルに関する条件を追加したいときです。

`user.books.where(name: "igaiga")` とmergeメソッドをつかわずにそのまま書くと、where条件の対象モデルはBookモデルとなるので、 `books.name = 'igaiga'` という条件になってしまいます。

```ruby
user.books.where(name: "igaiga").to_sql
#=> "SELECT "books".* FROM "books" WHERE "books"."user_id" = 1 AND "books"."name" = 'igaiga'"
```

`user.books.merge(User.where(name: "igaiga"))` と書くことでwhere条件にusers.name = 'igaiga' と書けます。

```ruby
user.books.merge(User.where(name: "igaiga")).to_sql
#=> "SELECT "books".* FROM "books" WHERE "books"."user_id" = 1 AND "users"."name" = 'igaiga'"
```

また、joinsをつかうときにテーブルを指定して条件を書きたいときがあります。`User.joins(:books).where(title: "Ruby")` と書くと、where条件の対象はBookモデルではなくUserモデルなので、`WHERE users.title = 'Ruby'` となります。

```ruby
User.joins(:books).where(title: "Ruby").to_sql
#=> "SELECT "users".* FROM "users" INNER JOIN "books" ON "books"."user_id" = "users"."id" WHERE "users"."title" = 'Ruby'"
```

join先のテーブルへAR::Releation条件をつなげたいときに、mergeメソッドをつかえば次のように書けます。

```ruby
User.joins(:books).merge(Book.where(title: "Ruby"))
#=> "SELECT "users".* FROM "users" INNER JOIN "books" ON "books"."user_id" = "users"."id" WHERE "books"."title" = 'Ruby'"
```

このようにmergeメソッドをつかうと、対象モデルを指定してARメソッド群をつなげることができます。ここではwhere1つだけをmergeしていますが、scopeを呼び出したり、もっと複雑なAR::Releationをつなげるときに便利です。

ちなみに、whereメソッドは次のように対象テーブルを指定することもできます。

```ruby
User.joins(:books).where(books: {title: "Ruby"})
```

また、mergeメソッドの引数にはAR::Releationのほか、ARオブジェクトの配列（アンド集合を返す）、またはProc（複数のAR::Relationで共有してつかうときに便利）を渡すこともできます。

```ruby
recent_books = Book.order(created_at: :desc).first(5)
Book.where(title: "Ruby").merge(recent_books)
```

```ruby
join_books = -> { joins(:books) } # この条件を複数箇所でつかいたい
User.merge(join_books)
#=> SELECT "users".* FROM "users" INNER JOIN "books" ON "books"."user_id" = "users"."id"
```

### 参考資料

- https://api.rubyonrails.org/classes/ActiveRecord/SpawnMethods.html#method-i-merge

### サンプルコード

- https://github.com/igaiga/basics_of_active_record/tree/merge
