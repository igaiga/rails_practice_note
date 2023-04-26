---
title: "[ActiveRecord] Delegated Type"
---

# Delegated Type

## Delegated Typeとは

複数モデルのオブジェクト、複数テーブルのレコードを統一的に扱いたいケースがあります。たとえば、コメントモデルとメッセージモデルのレコード群を足し合わせたものから最新レコード3つを取得したいようなケースです。

RailsではこのようなケースでつかえるSTI（Single Table Inheritance: 単一テーブル継承）を提供しています。STIは1つのテーブルに複数モデルのレコードを記録して、取得すると各クラスのオブジェクトとして扱うことができる仕組みです。しかし、1つのテーブルを全てのモデルクラスで共有するため、個々のモデルごとに別のカラムをつくりたいときには、その共有されているテーブルへカラムを追加するしかなく、必要ではないモデルにまでカラムが追加されてしまう問題があります。

Rails6.1で導入されたDelegated Typeはこのような状況でつかえます。

Delegated Typeでは、個々のモデルは一般のモデルと同じくそれぞれのテーブルを持ち、共通部分を統一的に管理するモデルとそのテーブルを別途導入します。このような構成にすると、個々のモデルで必要になったカラムはそれぞれのモデルへ加えれば良く、共通で必要なカラムは共通部分を管理するモデルへ追加することができます。

## どのような設計にするか

次のようなモデル群があるとします。

- Commentモデル(contentカラムを持つ)
- Messageモデル(subject, bodyカラムを持つ)

これらを統一的に扱う統合クラスであるEntryモデルをDelegated Typeをつかって実装します。（「統合クラス」は私の造語で一般的ではないのですが、この資料では統合クラスと呼ぶことにします。）

たとえば、「CommentモデルとMessageモデルのレコード群を足し合わせたものから最新レコード3つを取得」するときに `Entry.last(3)` で取得できるようにします。

## 実装

### マイグレーションファイル

次のようなスキーマを作成します。

```ruby
create_table "comments", force: :cascade do |t|
  t.string "content"
  t.datetime "created_at", null: false
  t.datetime "updated_at", null: false
end

create_table "messages", force: :cascade do |t|
  t.string "subject"
  t.string "body"
  t.datetime "created_at", null: false
  t.datetime "updated_at", null: false
end

create_table "entries", force: :cascade do |t|
  t.string "entryable_type"
  t.integer "entryable_id"
  t.datetime "created_at", null: false
  t.datetime "updated_at", null: false
  t.index ["entryable_id"], name: "index_entries_on_entryable_id"
end
```

統合クラスであるEntryには`entryable_type`カラムと`entryable_id`カラムをつくります。レコードが保存されるときに、entryable_typeには対応するクラス名、entryable_idには対応クラスでのidが自動で保存されます。ここではつくっていませんが、Entryモデルに共通でつかうカラムをつくってもOKです。

マイグレーションファイルとしては次のようになります。

- 20230426053310_create_messages.rb

```ruby
class CreateMessages < ActiveRecord::Migration[7.0]
  def change
    create_table :messages do |t|
      t.string :subject
      t.string :body

      t.timestamps
    end
  end
end
```

- 20230426053318_create_comments.rb

```ruby
class CreateComments < ActiveRecord::Migration[7.0]
  def change
    create_table :comments do |t|
      t.string :content

      t.timestamps
    end
  end
end
```

- 20230426053233_create_entries.rb

```ruby
class CreateEntries < ActiveRecord::Migration[7.0]
  def change
    create_table :entries do |t|
      t.string :entryable_type
      t.references :entryable

      t.timestamps
    end
  end
end
```

### モデル

統合クラスであるEntryモデルには `delegated_type` メソッドでDelegatedTypeである旨を示します。`:entryable` はスキーマで定義した `entryable_type` と `entryable_id` の `_` より前の部分を書きます。`:types` オプションには委譲先クラス群の名前を書きます。

- app/models/entry.rb

```ruby
class Entry < ApplicationRecord
  delegated_type :entryable, types: ["Message", "Comment"]
end
```

委譲先クラスには次のように`has_one`メソッドを書きます。

- app/models/comment.rb

```ruby
class Comment < ApplicationRecord
  has_one :entry, as: :entryable, touch: true
end
```

- app/models/message.rb

```ruby
class Message < ApplicationRecord
  has_one :entry, as: :entryable, touch: true
end
```

今後、他にも委譲先クラスが出てくることを見越してconcernモジュールにしておくのも良いでしょう。

- app/models/concerns/entryable.rb

```ruby
module Entryable
  extend ActiveSupport::Concern

  included do
    has_one :entry, as: :entryable, touch: true
  end
end
```

```ruby
class Comment < ApplicationRecord
  include Entryable
end

class Message < ApplicationRecord
  include Entryable
end
```

## つかい方

### レコードのつくり方

レコードをつくるときは統合クラスと委譲先クラスを同時につくります。次のコードをrails consoleなどで実行してレコードをつくってみてください。

```ruby
Entry.create!(entryable: Comment.new(content: "Hello!"))
Entry.create!(entryable: Message.new(subject: "Hi!", body: "Hello world!"))
```

長くなるので、Factoryパターンをつかってオブジェクトをつくって返すメソッドを実装しても良いでしょう。

```ruby
class Entry < ApplicationRecord
  # ...
  def self.create_with_comment!(content:)
    create!(entryable: Comment.new(content: content))
  end

  def self.create_with_message!(subject:, body:)
    create!(entryable: Message.new(subject: subject, body: body))
  end
end
# Entry.create_with_comment!(content: "Hello!")
```

### レコードの読み出し方

レコードを読み出すときは、統合クラスEntryからいつも通りのActiveRecordメソッドをつかいます。たとえば、CommentまたはMessageのどちらか最新1件を取得するには、`Entry.last` で取得できます。

```
latest_entry = Entry.last
#=>
#<Entry:0x00000001071a00d8
 id: 2,
 entryable_type: "Message",
 entryable_id: 1,
 created_at: Wed, 26 Apr 2023 05:41:25.302092000 UTC +00:00,
 updated_at: Wed, 26 Apr 2023 05:41:25.302092000 UTC +00:00>
```

取得したレコードがCommentまたはMessageのどちらかを調べるために、`comment?`, `message?` メソッドが用意されています。ただ、これらのメソッドで判断して分岐するよりは、次の節で説明する委譲メソッドをつかってポリモーフィズムをつかう設計にした方がコードが読みやすいことが多そうです。

```ruby
last_entry.message? #=> true
last_entry.comment? #=> false
```

Entry#entryableメソッドで対応する委譲先クラスのオブジェクトを取得できます。

```
Entry.find(1).entryable
#=>
#<Comment:0x00000001139be4c0
 id: 1,
 content: "Hello!",
 created_at: Wed, 26 Apr 2023 05:40:29.857794000 UTC +00:00,
 updated_at: Wed, 26 Apr 2023 05:40:29.857794000 UTC +00:00>
Entry.find(2).entryable
#=>
#<Message:0x00000001139fe9d0
 id: 1,
 subject: "Hi!",
 body: "Hello world!",
 created_at: Wed, 26 Apr 2023 05:41:25.299822000 UTC +00:00,
 updated_at: Wed, 26 Apr 2023 05:41:25.299822000 UTC +00:00>
```

また、Messageモデルだけを取得したいときは `Entry.messages` メソッドで取得できます。これはActiveRecord Relationオブジェクトを返すので、ActiveRecordの他のメソッドをつなげることもできます。

## 委譲メソッドの実装

統合クラスEntryにメソッドを実装して、実際の処理を委譲先クラスCommentおよびMessageに任せるサンプルコードです。Entry#titleメソッドでCommentとMessageクラスへ委譲してtitleを返すように実装します。

```ruby
class Entry < ApplicationRecord
  delegated_type :entryable, types: %w[ Message Comment ]
  delegate :title, to: :entryable
end

class Message < ApplicationRecord
  def title
    subject
  end
end

class Comment < ApplicationRecord
  def title
    content.truncate(20)
  end
end
```

Entryモデルオブジェクトのtitleメソッドが呼ばれると、委譲先クラスのオブジェクトのtitleメソッドを実行して、それぞれのクラスごとに定めた結果を返します。

Entryクラスの `delegate :title, to: :entryable` は、 `entryable.title` 相当の動作をします。

## 参考資料

- サンプルコード: https://github.com/igaiga/delegated_type_sample_app
- [ActiveRecord::DelegatedType](https://api.rubyonrails.org/classes/ActiveRecord/DelegatedType.html)
- [Rails: ActiveRecord::DelegatedType APIドキュメント（翻訳）](https://techracho.bpsinc.jp/hachi8833/2022_04_12/112882)

