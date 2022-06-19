---
title: "[ActiveRecord] through, source"
---

# [ActiveRecord] through, source

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

## 参考資料

- Railsガイド Active Recordの関連付け: https://railsguides.jp/association_basics.html

## サンプルコード

- https://github.com/igaiga/rails7023_ruby311_ar_through_source_app
