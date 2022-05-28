---
title: "[ActiveRecord] foreign_key, class_name"
---

# [ActiveRecord] foreign_key, class_name

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

# 参考ページ

- マイグレーションファイルのadd_foreign_keyに渡せるオプション
    - https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_foreign_key
- マイグレーションファイルのreferencesに渡せるオプション
    - https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_reference

# サンプルコード

- https://github.com/igaiga/basics_of_active_record/tree/refs/tags/foreign_key_and_class_name

