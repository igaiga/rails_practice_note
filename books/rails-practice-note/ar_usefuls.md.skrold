---
title: "[ActiveRecord] ActiveRecordの知識"
---
# [ActiveRecord] ActiveRecordの知識

## 関連先を指定するjoinsメソッド、whereメソッドの書き方

User has_many books (User 1-* Book) という関連があるとき、UserモデルからBookモデルを指定するwhereメソッドの書き方は次のようになります。

```ruby
User.joins(:books).where(books: {title: "Ruby"})
# SQL: SELECT "users".* FROM "users" INNER JOIN "books" ON "books"."user_id" = "users"."id" WHERE "books"."title" = ?  [["title", "Ruby"]]
```

関連名、ここではhas_manyである`books`をwhereメソッドのキーワード引数のキーワードに指定して、引数の値にはHashオブジェクトでキーにカラム名を、値に検索語を書きます。

多段の関連も指定可能です。User has_many books, Book has_many novelties (User 1-* Book 1-* Novelty) のように2つ先の関連があるとき、UserモデルからNoveltyモデルを指定するwhereメソッドの書き方は次のようになります。

```ruby
User.joins(books: :novelties).where(novelties: {name: "postcard"})
# SQL: SELECT "users".* FROM "users" INNER JOIN "books" ON "books"."user_id" = "users"."id" INNER JOIN "novelties" ON "novelties"."book_id" = "books"."id" WHERE "novelties"."name" = ?  [["name", "postcard"]]
```

joinsメソッドにはキーワード引数のキーワードに最初の関連名、引数の値に2つ目の関連名を指定します。whereメソッドの書き方は1段の関連のときと同じ書き方で、関連名`novelties`をキーワード引数のキーワードに、引数の値にはHashオブジェクトでキーにカラム名を、値に検索語を書きます。

また、 `User.joins(books: :novelties).where(books: {novelties: {name: "postcard"}})` とwhereメソッドに関連名booksから書き始めることもできます。発行されるSQLは同じです。

belongs_to関連を多段で指定するときは次のようになります。whereメソッドのキーワード引数のキーワードには関連名またはテーブル名を指定することができます。

```ruby
Novelty.joins(book: :user).where(user: {name: "igaiga"})
# SQL: SELECT "novelties".* FROM "novelties" INNER JOIN "books" ON "books"."id" = "novelties"."book_id" INNER JOIN "users" "user" ON "user"."id" = "books"."user_id" WHERE "user"."name" = ?  [["name", "igaiga"]]

Novelty.joins(book: :user).where(users: {name: "igaiga"})
# SQL: SELECT "novelties".* FROM "novelties" INNER JOIN "books" ON "books"."id" = "novelties"."book_id" INNER JOIN "users" ON "users"."id" = "books"."user_id" WHERE "users"."name" = ?  [["name", "igaiga"]]

Novelty.joins(book: :user).where(book: {users: {name: "igaiga"}})
# SQL: SELECT "novelties".* FROM "novelties" INNER JOIN "books" "book" ON "book"."id" = "novelties"."book_id" INNER JOIN "users" ON "users"."id" = "book"."user_id" WHERE "users"."name" = ?  [["name", "igaiga"]]

Novelty.joins(book: :user).where(books: {users: {name: "igaiga"}})
# SQL: SELECT "novelties".* FROM "novelties" INNER JOIN "books" ON "books"."id" = "novelties"."book_id" INNER JOIN "users" ON "users"."id" = "books"."user_id" WHERE "users"."name" = ?  [["name", "igaiga"]]
```

参考資料欄の「Railsガイド Active Record クエリインターフェイス テーブルを結合する」には、より複雑なパターンのjoinsメソッドの書き方が紹介されています。

### 参考資料

- Railsガイド Active Record クエリインターフェイス テーブルを結合する
  - https://railsguides.jp/active_record_querying.html

### サンプルコード

- https://github.com/igaiga/basics_of_active_record/tree/3step_relations

## mergeメソッド

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

## selectとpluckの違い

ActiveRecordのselectメソッドとpluckメソッドは次のように動きます。

- pluckは呼び出すとすぐにSQL発行し、結果の値の配列が取れる
- selectはActiveRecord::Relationを返し、to_aメソッドなどでSQLが発行され、ActiveRecordオブジェクトの配列が取れる
- `select(Arel.sql("... as foo"))` を実行すると、取得したActiveRecordオブジェクトにSQLの結果を取得するfooメソッド(attributes)がつくられる

pluckの結果はActiveRecordオブジェクトではなく値になるため、結果レコード数が多いときにつかうとメモリ使用量を抑えることができます。結果レコード数がおおよそ1000レコードを超えてくると、ActiveRecordオブジェクト群をつくるためにメモリをたくさん確保してパフォーマンスが悪くなりはじめることがあります。そのケースではpluckをつかうとメモリ消費量を抑えて高速に動かすことができます。ただし、ActiveRecordオブジェクトの機能がつかえない制限があります。StringやIntegerなどの値と、ActiveRecordオブジェクトとのメモリ使用量を比較すると、ActiveRecordオブジェクトが数倍から数百倍大きくなります。

どちらのメソッドもattributes(カラム)のシンボルを渡したり、SQLの一部を書くこともできます。

```ruby
irb> Book.pluck(:title)
#=> SELECT "books"."title" FROM "books"
=> ["Ruby"] # 結果の値が取れる

irb> Book.select(:title)
#=> SELECT "books"."title" FROM "books"
=> [#<Book:0x000000010a206628 id: nil, title: "Ruby">] # 結果を入れたActiveRecordオブジェクトが取れる
```

selectをつかう場合、取得できるのはActiveRecordオブジェクトなので、attributesにないものを取得しようとしても一見そのメソッドがないように見えます。SQLを書いてas some_nameを書くと、ActiveRecordがattributesとしてsome_nameメソッドをつくる機能があるので、これをつかって取得できます。

```ruby
irb> Book.pluck(Arel.sql("count(*)"))
#=> SELECT count(*) FROM "books"
=> [1]

irb> Book.select(Arel.sql("count(*) as count")).first
=> #<Book:0x0000000129e5b350 id: nil>

irb> Book.select(Arel.sql("count(*) as count")).first.count
#=> Book Load (0.5ms)  SELECT count(*) as count FROM "books" ORDER BY "books"."id" ASC LIMIT ?  [["LIMIT", 1]]
=> 1 # モデルに格納する場所がなさそうに見えるが、as countが実行されてcountメソッド(attributes)で取り出せる。

irb> Book.select(Arel.sql("count(*) as count")).first.attributes
=> {"count"=>1, "id"=>nil} # attributesとしてcountが追加されている

irb> Book.select(Arel.sql("count(*) as foo")).first.foo
=> 1 # as xxx の名前はなんでもOK
```

### サンプルコード

- https://github.com/igaiga/basics_of_active_record/tree/select_pluck

## group, having, count

SQLで同じ値のデータ群をまとめるための道具としてgroup byがあります。ActiveRecordではgroupメソッドでつかうことができます。havingメソッド、countメソッドと組み合わせてつかうこともできます。

### groupメソッド

Bookモデルのtitleカラムでgroup byでまとめるには `Book.group(:title)` と書きます。次のコードは、同じtitleカラムのレコードは1つだけ取得されるようなActiveRecord Relationを取得しています。

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

booksテーブルの全カラムを取得する必要がないときは、pluckメソッドやselectメソッドで絞ることもできます。2つのメソッドの違いは「pluckとselectの違い」を参照してください。次のコードはgroup byをつかってbooksテーブルの全titleを重複なく取得しています。

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

### groupメソッドとhavingメソッド

group byの条件を加えるSQLのhaving句を書くにはgroupメソッドとhavingメソッドをつかいます。

```ruby
Book.having("SUM(price) > 1000").group("user_id")
#=> SELECT "books".* FROM "books" GROUP BY "books"."user_id" HAVING (SUM(price) > 1000)
```

### groupメソッドとcountメソッド

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

### 参考資料

- Railsガイド Active Record クエリインターフェイス: https://railsguides.jp/active_record_querying.html
- groupメソッド: https://api.rubyonrails.org/classes/ActiveRecord/QueryMethods.html#method-i-group
- havingメソッド: https://api.rubyonrails.org/classes/ActiveRecord/QueryMethods.html#method-i-having
- countメソッド: https://api.rubyonrails.org/classes/ActiveRecord/Calculations.html#method-i-count

### サンプルコード

- https://github.com/igaiga/basics_of_active_record/tree/refs/tags/group_by

## foreign_key, class_name

Railsは規約通りの名付けをしているときには多くのオプションを省略できます。

User が1対多で Book を持っているときのコード例は次のようになります。

```ruby
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :name

      t.timestamps
    end
  end
end

class CreateBooks < ActiveRecord::Migration[7.0]
  def change
    create_table :books do |t|
      t.string :title
      t.references :user, null: false, foreign_key: true

      t.timestamps
    end
  end
end

class User < ApplicationRecord
  has_many :books
end

class Book < ApplicationRecord
  belongs_to :user
end
```

一方で、規約から外れた命名をしたいときもあります。

User が1対多で Book を持っていて、ただしこれは著作の関係であるので、Bookからは `book.user` ではなく、 `book.author` でUserを取得したいというケースを考えます。booksテーブルで規約名以外の外部キーカラム名(author_id)をつかうケースです。

そのときのコードは次のようになります。

```ruby
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :name

      t.timestamps
    end
  end
end

class CreateBooks < ActiveRecord::Migration[7.0]
  def change
    create_table :books do |t|
      t.string :title
      t.references :author, null: false, foreign_key: { to_table: :users }

      t.timestamps
    end
  end
end

class User < ApplicationRecord
  has_many :books, foreign_key: "author_id" #TODO: inverse_of
end

class Book < ApplicationRecord
  belongs_to :author, class_name: "User" #TODO: inverse_of
end
```

差分を順に説明します。

### 規約名以外の外部キーカラム名をつかうときのマイグレーションファイル

DBテーブル構造は、booksテーブルからusersテーブルへのforeign_keyとしてuser_idではなく、author_idをつかうことにします。

マイグレーションファイルでは外部キー制約に外部キーカラム名を指定する必要があります。referencesの:foreign_keyオプションへ:to_table オプションを追加で書きます。

```ruby
t.references :author, null: false, foreign_key: { to_table: :users }
```

結果、schema.rbでは次のように外部キー制約が記述されます。

```ruby
add_foreign_key "books", "users", column: "author_id"
```

### 規約名以外の外部キーカラム名をつかうときのモデル

規約名以外の外部キーカラム名をつかうときは、モデルでhas_many, belongs_toを書くときにオプションを追加する必要があります。

Userモデルにhas_manyを書くときに、 :foreign_key オプションで外部キーカラム名を指定します。

```ruby
class User < ApplicationRecord
  has_many :books, foreign_key: "author_id"
end
```

Bookモデルにbelongs_toを書くときに、 :class_name オプションで関連先モデルを指定します。

```ruby
class Book < ApplicationRecord
  belongs_to :author, class_name: "User"
end
```

ここでは、外部キーカラム名として関連名authorから規約通りのカラム名author_idがつかわれます。外部キーカラム名が規約通りでないときは、 :foreign_key オプションも追加して外部キーカラム名を指定します。

ここまでで関連付けは動作しますが、規約通りでない命名のときには双方向関連付けが自動で行われません。双方向関連付けを指定するためにinverse_ofオプションをモデルに追記します。双方向関連付けについては「[ActiveRecord] 双方向関連付けとinverse_of」で詳しく説明します。

```ruby
class User < ApplicationRecord
  has_many :books, foreign_key: "author_id", inverse_of: :author
end

class Book < ApplicationRecord
  belongs_to :author, class_name: "User", inverse_of: :books
end
```

### 参考資料

- マイグレーションファイルのadd_foreign_keyに渡せるオプション
    - https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_foreign_key
- マイグレーションファイルのreferencesに渡せるオプション
    - https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_reference
- Railsガイド Active Recordの関連付け: https://railsguides.jp/association_basics.html

### サンプルコード

- https://github.com/igaiga/basics_of_active_record/tree/refs/tags/foreign_key_and_class_name

## through, source

多対多の関連をつくるときは、中間テーブルおよびモデルと、has_many :through をつかいます。

たとえば、UserとBookとその所有権を記録する中間モデルOwnershipの例は以下のようになります。

```ruby
class User < ApplicationRecord
  has_many :ownerships
  has_many :books, through: :ownerships
end

class Ownership < ApplicationRecord
  belongs_to :user
  belongs_to :book
end
# 中間テーブルのマイグレーションファイル例
# class CreateOwnerships < ActiveRecord::Migration[7.0]
#   def change
#     create_table :ownerships do |t|
#       t.references :user, null: false, foreign_key: true
#       t.references :book, null: false, foreign_key: true
#       t.timestamps
#     end
#   end
# end

class Book < ApplicationRecord
  has_many :ownerships
  has_many :users, through: :ownerships
end
```

命名が規約通りなので、少ないコード量で記述できます。では、名前が規約から外れたときはどのようになるでしょうか。

UserとBook、中間モデルAuthorship(著作している)の例を考えてみます。

```ruby
class User < ApplicationRecord
  has_many :authorships
  has_many :authored_books, through: :authorships, source: :book
end

class Authorship < ApplicationRecord
  belongs_to :user
  belongs_to :book
end
# 中間テーブルのマイグレーションファイル例
# class CreateAuthorships < ActiveRecord::Migration[7.0]
#   def change
#     create_table :authorships do |t|
#       t.references :user, null: false, foreign_key: true
#       t.references :book, null: false, foreign_key: true
#
#       t.timestamps
#     end
#   end
# end

class Book < ApplicationRecord
  has_many :authorships
  has_many :authors, through: :authorships, source: :user
end
```

先ほどと違うのは、Bookモデルのauthors関連の名前が規約から外れているので、has_many, :through に加えて :source オプションを書いていることです。 :source オプションには:throughで指定したモデルから、取得したいモデルへたどるための関連名を書きます。もしも `has_many :authors, through: :authorships` と:sourceを書かずに実行すると、次のようなエラーになり、関連を取得できない旨が表示されます。

```shell
Could not find the source association(s) "author" or :authors in model Authorship. Try 'has_many :authors, :through => :authorships, :source => <name>'.
```

エラーメッセージの指示通りに :source オプションを書きます。:source には「:throughで指定した関連先モデルから、取得したいモデルへたどるための関連名」を書きます。ここではAuthorshipモデルに書かれたbelongs_to :userでUserモデルへたどりたいので、source: :user を加えて、 `has_many :authors, through: :authorships, source: :user` となります。取得されるオブジェクトとSQLを確認して、意図通り書けたかどうかを確認します。

:sourceオプションはhas_many関連と、has_one関連で:throughオプションを書いていて、かつ、規約から外れた命名をしているときに指定します。また、ポリモフィック関連で:sourceを書くときは、あわせて :source_type も書く必要があります。詳しくはRailsガイドを読んでみてください。

私は:sourceオプションにどの関連名を書けばいいのかよく分からないことが多かったのですが、「SQLのfrom相当をARの関連として書くためのオプションがsourceである」と解釈をしてから覚えられるようになってきました。

### サンプルコード

- https://github.com/igaiga/rails7023_ruby311_ar_through_source_app

