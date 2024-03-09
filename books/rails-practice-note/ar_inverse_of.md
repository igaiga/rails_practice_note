---
title: "[ActiveRecord] 双方向関連付けとinverse_of"
---

# [ActiveRecord] 双方向関連付けとinverse_of

Railsは規約通りの名前の関連があるとき、自動で双方向関連付けがなされています。

## 双方向関連付け機能

双方向関連付けについて、次のような関連を持っているコードで説明します。

```ruby
class User < ApplicationRecord
  has_many :books
end

class Book < ApplicationRecord
  belongs_to :user
end
```

User➡️Book➡️Userと関連をたどるコードを書いて結果を見ると次のようになります。

```ruby
user = User.first
book = user.books.first

user.object_id #=> 30200
book.user.object_id #=> 30200
# userオブジェクトとbookオブジェクトの関連先のuserオブジェクトは同じ

user.name #=> "igaiga"
book.user.name #=> "igaiga"

user.name = "igarashi" # userオブジェクトのnameを変更
user.name #=> "igarashi"
book.user.name #=> "igarashi"
# bookオブジェクトからたどったuserオブジェクトのnameも変更されている
```

つまり、`user.books.first.user`のように関連先Bookオブジェクトから関連をたどったUserオブジェクトも、その前に作成してあったUserオブジェクトと自動的に同一オブジェクトとなるような工夫がなされています。これが双方向関連付けの機能です。

## 双方向関連付けが自動認識されないケース

ActiveRecordは規約通りであればこのように自動で双方向関連付けを行いますが、規約から外れた名前をつかうとき、たとえば:foreign_keyや:throughオプションをつかうときには、双方向関連付けを自動認識しません。規約から外れた名前で関連付けたコードでは、結果は次のようになります。

```ruby
class User < ApplicationRecord
  has_many :books
end

class Book < ApplicationRecord
  belongs_to :author, foreign_key: "user_id", class_name: "User"
end
```

```ruby
user = User.first
book = user.books.first

user.object_id #=> 114540
book.author.object_id #=> 140620
# userオブジェクトとbookオブジェクトの関連先のuserオブジェクトは別のもの

user.name #=> "igaiga"
book.author.name #=> "igaiga"

user.name = "igarashi" # userオブジェクトのnameを変更
user.name #=> "igarashi"
book.author.name #=> "igaiga"
# bookオブジェクトからたどったuserオブジェクトのnameは変更されていない
```

## 双方向関連付けを手動設定する

has_many, has_one, belongs_to で:foreign_keyオプションなどをつかって規約とは違う命名をするときには、:inverse_ofにつづけて関連先モデルからの関連名を書くことで双方向の関連をActiveRecordへ教えることができます。

```ruby
class User < ApplicationRecord
  has_many :books, inverse_of: :author
end

class Book < ApplicationRecord
  belongs_to :author, foreign_key: "user_id", class_name: "User"
end
```

```ruby
user = User.first
book = user.books.first

user.object_id #=> 108560
book.author.object_id #=> 108560
# userオブジェクトとbookオブジェクトの関連先のuserオブジェクトは同じ

user.name #=> "igaiga"
book.author.name #=> "igaiga"

user.name = "igarashi" # userオブジェクトのnameを変更
user.name #=> "igarashi"
book.author.name #=> "igarashi"
# bookオブジェクトからたどったuserオブジェクトのnameも変更されている
```

双方向関連付けがなされていないときには、これに起因するバグや余分なクエリ発行を防ぐために、inverse_ofを書いておいた方が安全です。余分なクエリが発行されているコード例は次のようなコードです。

```ruby
# 双方向関連付け(inverse_of)あり
user = User.first
user.books #クエリ発行
user2 = user.books.first.author #クエリが発行されない(双方向関連付けの効果)
user2.books #クエリが発行されない(前回のuser.booksのキャッシュ利用)

user.object_id  #=> 124660
user2.object_id #=> 124660

# 双方向関連付け(inverse_of)なし
user = User.first
user.books #クエリ発行
user2 = user.books.first.author #クエリ発行(余分)
user2.books #クエリ発行(余分)

user.object_id  #=> 22680
user2.object_id #=> 24120
```

## まとめ

- ActiveRecordでは規約通りの名前でつくられた関連付けでは、双方向関連付けを自動認識する
- 規約から外れたとき、たとえば:foreign_keyや:throughをつかうときには双方向関連付けを自動認識しない
- 自動認識しないケースではinverse_ofオプションを指定することで双方向関連付けを手動設定できる
- 双方向関連付けがなされていないと、バグや余分なクエリ発行の原因となることがある

## 参考資料

- Railsガイド 双方向関連付け
https://railsguides.jp/association_basics.html#%E5%8F%8C%E6%96%B9%E5%90%91%E9%96%A2%E9%80%A3%E4%BB%98%E3%81%91

## サンプルコード
- https://github.com/igaiga/basics_of_active_record/tree/inverse_of
