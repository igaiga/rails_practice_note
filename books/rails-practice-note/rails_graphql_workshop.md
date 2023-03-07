---
title: "[Rails基礎] GraphQL基礎講座"
---

GraphQLをつかったRailsアプリづくりを通じて、GraphQLの基礎を説明していきます。

- サンプルコード
  - https://github.com/igaiga/graphql_sample_rails70

# RailsアプリでGraphQLをつかえるようにする

以下の手順で既存のRailsアプリでGraphQLをつかえるようにします。

## Gemfile

Gemfileへ以下を追加します。

```Gemfile
gem 'graphql'
group :development do
  gem 'graphiql-rails'
end
```

graphql gemとgraphql-rails gemの役割は次の通りです。

- graphql gem
    - https://graphql-ruby.org/
    - railsでgraphqlを利用するためのgem
- graphql-rails gem
    - https://github.com/rmosolgo/graphiql-rails
    - railsでGraphiQL IDEを提供するgem

## 準備

次のコマンドを実行します。

- $ bin/rails g graphql:install

追加変更されたファイル群は次の通りです。

- app/controllers/graphql_controller.rb
- app/graphql/graphql_sample_rails70_schema.rb
- app/graphql/mutations/.keep
- app/graphql/mutations/base_mutation.rb
- app/graphql/types/.keep
- app/graphql/types/base_argument.rb
- app/graphql/types/base_connection.rb
- app/graphql/types/base_edge.rb
- app/graphql/types/base_enum.rb
- app/graphql/types/base_field.rb
- app/graphql/types/base_input_object.rb
- app/graphql/types/base_interface.rb
- app/graphql/types/base_object.rb
- app/graphql/types/base_scalar.rb
- app/graphql/types/base_union.rb
- app/graphql/types/mutation_type.rb
- app/graphql/types/node_type.rb
- app/graphql/types/query_type.rb
- config/routes.rb

```config/routes.rb
Rails.application.routes.draw do
  if Rails.env.development?
    mount GraphiQL::Rails::Engine, at: "/graphiql", graphql_path: "/graphql"
  end
  post "/graphql", to: "graphql#execute"
end
```

以降の説明でつかうUserモデルを作成します。

- $ bin/rails g model user name, email
  - nameにnull: false追加

```db/migrate/xxx_create_users.rb
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :name, null: false
      t.string :email

      t.timestamps
    end
  end
end
```

## Type作成

DBテーブルスキーマと対応するようなTypeを作成します。

次のコマンドでTypeファイルを作成します。

```
$ bin/rails g graphql:object User
 create app/graphql/types/user_type.rb
```

コマンドを実行すると、テーブルのスキーマを見て自動でTypeファイルを生成します。

```app/graphql/types/user_type.rb
# frozen_string_literal: true

module Types
  class UserType < Types::BaseObject
    field :id, ID, null: false
    field :name, String, null: false
    field :email, String
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    field :updated_at, GraphQL::Types::ISO8601DateTime, null: false
  end
end
```

以下のように手動指示もできるようですが、既にモデルがあるとカラムが重複します。

- $ bin/rails g graphql:object User id:ID! name:String! email:String
  - ID: GraphQLでのIDの型(String)
  - 末尾 ! あり: null不許可
  - 末尾 ! なし: null許可

## Query

取得系はQueryで処理します。前準備として、usersテーブルにデータ作成しておきます。

- $ bin/rails c
- `User.create!(name: "igaiga", email: "igaiga@example.com")`
- `User.create!(name: "yuri", email: "yuri@example.com")`

### 1つ取得する field :user 

QueryType(app/graphql/type/query_type.rb) へ `field :user` (単数)を追加します。fieldは後述のGraphQLクエリの要素としてつかわれます。field関連コードに、GraphQLクエリにこのfield名が出てきたときの処理を書きます。

```app/graphql/type/query_type.rb
module Types
  class QueryType < Types::BaseObject
    ...
    field :user, Types::UserType, null: false do # GraphQLクエリでのfield名、GraphQL Type、必須かどうか
      argument :id, ID, required: true # GraphQLクエリでの引数、GraphQLでの型、必須かどうか
    end
    def user(id:) # fieldが指定されたとき、どのようにデータ取得するかをfieldと同名メソッドで実装する
      User.find(id)
    end
  end
end
```

動作確認します。

- $ bin/rails s
- http://localhost:3000/graphiql から以下のクエリ発行
    - http://localhost:3000/graphql POST でリクエストが飛ぶ
    - GraphqlController#execute で処理される
        - paramsはクエリJSON

- 入力するクエリ

```graphql
{
  user(id: "1") {
    id
    name
    email
  }
}
```

- 結果

```json
{
  "data": {
    "user": {
      "id": "1",
      "name": "igaiga",
      "email": null
    }
  }
}
```

### 複数取得するfield :users

QueryType(app/graphql/type/query_type.rb) へ `field :users` (複数)を追加します。 `field :users, [Types::UserType], null: false` の `[Types::UserType]` 部分を配列にして複数であることを指定しています

```app/graphql/type/query_type.rb
module Types
  class QueryType < Types::BaseObject
    ...
    field :users, [Types::UserType], null: false
    def users
      User.all
    end
  end
end
```

動作確認します。

- $ bin/rails s
- http://localhost:3000/graphiql から以下のクエリ発行

- 入力するクエリ

```graphql
{
  users {
    id
    name
    email
  }
}
```

- 結果

```json
{
  "data": {
    "users": [
      {
        "id": "1",
        "name": "igaiga",
        "email": "igaiga@example.com"
      },
      {
        "id": "2",
        "name": "yuri",
        "email": "yuri@example.com"
      }
    ]
  }
}
```

## Mutation

更新系はMutationをつくります。以下はUserの新規作成をつくる例です。次のコマンドでmutationファイルを作成します。

```
bin/rails g graphql:mutation CreateUser
  create  app/graphql/mutations/create_user.rb
```

コマンドを実行すると、あわせてapp/graphql/types/mutation_type.rbに変更が加わります。

```app/graphql/types/mutation_type.rb
module Types
  class MutationType < Types::BaseObject
    field :create_user, mutation: Mutations::CreateUser
```

app/graphql/mutations/create_user.rb は上記コマンドでファイルが作成されているので、以下のように実装します。

```ruby
module Mutations
  class CreateUser < BaseMutation
    field :user, Types::UserType, null: true

    argument :name, String, required: true
    argument :email, String, required: false

    def resolve(**args)
      user = User.create!(args)
      {
        user: user
      }
    end
  end
end
```

http://localhost:3000/graphiql で以下のクエリを実行して動作確認します。

- 入力するクエリ

```graphql
mutation {
  createUser(
    input:{
      name: "user1"
      email: "user@example.com"
    }
  ){
    user {
      id
      name 
      email
    }
  }
}
```

- 結果

```json
{
  "data": {
    "createUser": {
      "user": {
        "id": "3",
        "name": "user1",
        "email": "user@example.com"
      }
    }
  }
}
```

## Resolver

QueryTypeに `field :user` などを書いていくとファイルが大きくなっていってしまいます。Resolverをつかうとfieldごとにファイルに分けることができます。

ResolverはActiveRecordをつかってデータを取得する方法と、取得時につかう引数、利用するGraphQLのTypeなどを指定します。QueryTypeに書いていた項目をResolverへ移動するイメージです。user fieldでつかうResolverは以下のようになります。

```app/graphql/resolvers/user_resolver.rb
module Resolvers
  class UserResolver < GraphQL::Schema::Resolver
    description 'Find a user by ID'
    type Types::UserType, null: false

    argument :id, ID, required: true

    def resolve(id:)
      User.find(id)
    end
  end
end
```

QueryTypeを修正してResolverをつかうように変更します。

- app/graphql/types/query_type.rb

```diff
-    field :user, Types::UserType, null: false do
-      argument :id, ID, required: true
-    end
-    def user(id:)
-      User.find(id)
-    end
+    field :user, resolver: Resolvers::UserResolver
```

動作確認して同様に取得できることを確認します。

- 入力するクエリ

```graphql
{
  user(id: "1") {
    id
    name
    email
  }
}
```

- 出力

```json
{
  "data": {
    "user": {
      "id": "1",
      "name": "igaiga",
      "email": null
    }
  }
}
```

複数用の users filedも同様にResolverを作成してそれをつかうように書き換えます。

```app/graphql/resolvers/users_resolver.rb
module Resolvers
  class UsersResolver < GraphQL::Schema::Resolver
    description 'Find users'
    type [Types::UserType], null: false

    def resolve
      User.all
    end
  end
end
```

app/graphql/types/query_type.rb 

```diff
module Types
  class QueryType < Types::BaseObject
-    field :users, [Types::UserType], null: false
-    def users
-      User.all
-    end
+    field :users, resolver: Resolvers::UsersResolver
  end
end

```

```graphql
{
  users {
    id
    name
    email
  }
}
```

```json
{
  "data": {
    "users": [
      {
        "id": "1",
        "name": "igaiga",
        "email": null
      },
      {
        "id": "2",
        "name": "other",
        "email": null
      }
    ]
  }
}
```

## 関連

- 前準備としてUserモデルと1対多でBookモデルを作成します
    - $ bin/rails g model book title user:references
    - Userモデルにhas_many :books関連を追加します

- BookTypeを作成します
    - $ bin/rails g graphql:object book

```app/graphql/types/book_type.rb
# frozen_string_literal: true

module Types
  class BookType < Types::BaseObject
    field :id, ID, null: false
    field :title, String, null: false
    field :user_id, Integer, null: false
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    field :updated_at, GraphQL::Types::ISO8601DateTime, null: false
  end
end
```

- UserTypeにbooks fieldを追加します
    - app/graphql/types/user_type.rb

```diff
# frozen_string_literal: true

module Types
  class UserType < Types::BaseObject
    ...
+   field :books, [BookType], null: false # 追加
  end
end
```

- http://localhost:3000/graphiql からクエリを発行してみます

```graphql
{
  users {
    id
    name
    email
    books {
      id
      title
    }
  }
}
```

```json
{
  "data": {
    "users": [
      {
        "id": "1",
        "name": "igaiga",
        "email": null,
        "books": [
          {
            "id": "1",
            "title": "Ruby超入門"
          },
          {
            "id": "2",
            "title": "パーフェクトRails"
          }
        ]
      }
    ]
  }
}
```

- usersテーブルのレコードが2つ以上あると次のようにN+1になっていることがわかります
    - 次節でGraphQL::Batchをつかって解消します

```sql
SELECT "users".* FROM "users"
SELECT "books".* FROM "books" WHERE "books"."user_id" = ?  [["user_id", 1]]
SELECT "books".* FROM "books" WHERE "books"."user_id" = ?  [["user_id", 2]]
```

- BookとBooksもResolver化しておきます

- app/graphql/types/query_type.rb

```diff
-    field :books, [BookType], null: false
+    field :book, resolver: Resolvers::BookResolver
+    field :books, resolver: Resolvers::BooksResolver
```

```app/graphql/resolvers/book_resolver.rb
module Resolvers
  class BookResolver < GraphQL::Schema::Resolver
    description 'Find a book by ID'
    type Types::BookType, null: false

    argument :id, ID, required: true

    def resolve(id:)
      Book.find(id)
    end
  end
end
```

```app/graphql/resolvers/books_resolver.rb
module Resolvers
  class BooksResolver < GraphQL::Schema::Resolver
    description 'Find books'
    type [Types::BookType], null: false

    def resolve
      Book.all
    end
  end
end
```

## GraphQL::Batch

N+1問題が起きないようにデータ取得をする graphql-batch gem のつかい方です。

- https://github.com/Shopify/graphql-batch

```Gemfile
gem "graphql-batch"
```

`bundle install` を実行します。

app/graphql/ 以下にアプリ名_schema.rbファイルへ `use GraphQL::Batch` を追記します

```app/graphql/graphql_sample_rails70_schema.rb
class GraphqlSampleRails70Schema < GraphQL::Schema
  ...
  use GraphQL::Batch
end
```

### userからbooksを取得するケース(has_many対応)

has_many関連用のGraphQL::Batchを設定します。app/graphql/loaders/association_loader.rb を作成します。graphql-batch gem のexamplesのassociation_loader.rbをコピーします。

-  https://github.com/Shopify/graphql-batch/blob/master/examples/association_loader.rb

Zeitwerkの定めるネームスペースとフォルダ構成をあわせるために、 `module Loaders` でネームスペースを作ります。かんたんに言うと、has_many関連先も含めて過不足なく一度に全データを取得するローダーです。

```app/graphql/loaders/association_loader.rb
module Loaders
  class AssociationLoader < GraphQL::Batch::Loader
    def self.validate(model, association_name)
      new(model, association_name)
      nil
    end

    def initialize(model, association_name)
      super()
      @model = model
      @association_name = association_name
      validate
    end

    def load(record)
      raise TypeError, "#{@model} loader can't load association for #{record.class}" unless record.is_a?(@model)
      return Promise.resolve(read_association(record)) if association_loaded?(record)
      super
    end

    # We want to load the associations on all records, even if they have the same id
    def cache_key(record)
      record.object_id
    end

    def perform(records)
      preload_association(records)
      records.each { |record| fulfill(record, read_association(record)) }
    end

    private

    def validate
      unless @model.reflect_on_association(@association_name)
        raise ArgumentError, "No association #{@association_name} on #{@model}"
      end
    end

    def preload_association(records)
      ::ActiveRecord::Associations::Preloader.new(records: records, associations: @association_name).call
    end

    def read_association(record)
      record.public_send(@association_name)
    end

    def association_loaded?(record)
      record.association(@association_name).loaded?
    end
  end
end
```

次のことを把握すると、中で何をやっているかがなんとなくわかるようになるかもしれません。

- initialize(model, association_name) などのmodelはモデル、association_nameは関連名(has_many books のbooksなど)
- private validateメソッドはその関連が本当にあるか確認
- Promiseは貯めてあとで実行(resolve)してくれて、resolve済み(ロード済み)かどうかを管理してくれる
- preload_association メソッド内でpreloadなどN+1対応をやってくれる
- performメソッドに取得実行時の処理が書かれている
- こちらの記事が詳しいです: https://blog.kymmt.com/entry/graphql-batch-examples

その他のLoaderサンプルコードはgraphql-batchリポジトリのexamplesフォルダにあります。

- https://github.com/Shopify/graphql-batch/blob/master/examples/

上で書いたGraphQL::BatchのAssociationLoaderをつかってデータを取得するようにUserTypeを修正します。

app/graphql/types/user_type.rb

```diff
module Types
  class UserType < Types::BaseObject
...
    field :books, [BookType], null: false
+    def books
+      Loaders::AssociationLoader.for(User, :books).load(object)
+    end
  end
end
```

GraphQLクエリで取得します。

```graphql
{
  users {
    id
    name
    books {
      title
    }
  } 
}
```

Railsのログを確認すると、N+1が解消されていることがわかります。今回はActiveRecord preloadになっているようです。

```SQL
SELECT "users".* FROM "users"
SELECT "books".* FROM "books" WHERE "books"."user_id" IN (?, ?)  [["user_id", 1], ["user_id", 2]]
```

- `Loaders::AssociationLoader.for(User, :books).load(object)` を説明していきます
    - あとで必要になったときにまとめて実行してくれます
    - 戻り値はPromiseオブジェクトを返しています

- forメソッドの引数には取得したい関連をモデル名と関連名で指定します
    - 1つ目の引数は1対多の1側モデルを、2つ目の引数には多側の関連名を書きます
    - forメソッドは引数ごとにLoaderオブジェクトをひとつだけ作って返します
    - このとき、引数が違えば別のオブジェクトを作成しますが、同じ引数で2回呼んだときは前回と同じものを返します

- loadメソッドの引数は、ロードする対象を一意に決めるオブジェクトを渡します
    - ここではUserモデルオブジェクトを渡します
    - Userモデルオブジェクトの取得にはobjectメソッドを利用できます
    - ここでのselfはTypes::UserTypeで、objectメソッドはTypes::UserTypeが持つobjectを取得します
    - ここでobjectメソッドはusersテーブルの特定レコードに対応するUserモデルオブジェクトを返します
    - objectメソッドはgraphql-batch gem内のGraphQL::Schema::Objectクラスに定義されています

### bookからuserを取得するケース(belongs_to対応)

- belongs_to関連用のGraphQL::Batchを設定します
    - app/graphql/loaders/record_loader.rb を作成します
        - graphql-batch gem のexamplesのrecord_loader.rbをコピーします
            -  https://github.com/Shopify/graphql-batch/blob/master/examples/record_loader.rb
        - Zeitwerkの定めるネームスペースとフォルダ構成をあわせるために、 `module Loaders` でネームスペースを作ります。
        - かんたんに言うと、belongs_to関連先も含めて過不足なく一度に全データを取得するローダーです。

```app/graphql/loaders/record_loader.rb
module Loaders
  class RecordLoader < GraphQL::Batch::Loader
    def initialize(model, column: model.primary_key, where: nil)
      super()
      @model = model
      @column = column.to_s
      @column_type = model.type_for_attribute(@column)
      @where = where
    end

    def load(key)
      super(@column_type.cast(key))
    end

    def perform(keys)
      query(keys).each { |record| fulfill(record.public_send(@column), record) }
      keys.each { |key| fulfill(key, nil) unless fulfilled?(key) }
    end

    private

    def query(keys)
      scope = @model
      scope = scope.where(@where) if @where
      scope.where(@column => keys)
    end
  end
end
```

- 上で書いたRecordLoaderをつかってデータを取得するようにBookTypeを修正します

- app/graphql/types/book_type.rb

```diff
+ field :user, UserType, null: false
+ def user
+   Loaders::RecordLoader.for(User).load(object.user_id)
+ end
```

- GraphQLクエリを発行してみます

```graphql
{
  books {
 	title
	user {
  	 name
    }
  } 
}
```

```
SELECT "books".* FROM "books"
SELECT "users".* FROM "users" WHERE "users"."id" IN (?, ?)  [["id", 1], ["id", 2]]
```

- Railsのログを確認すると、N+1が解消されていることがわかります

- `Loaders::RecordLoader.for(User).load(object.user_id)`はPromiseを返します
    - 必要になったときにまとめて実行してくれます

- forメソッドは引数ごとにLoaderオブジェクトをひとつだけ作って返します
    - 引数は取得したいモデルを書きます
    - このとき、引数が違えば別のオブジェクトを作成しますが、同じ引数で2回呼んだときは前回と同じものを返します

- loadメソッドの引数は、ロードする対象を一意に決めるオブジェクトを渡します
    - ここではUserモデルのidを渡しています
    - ここでのselfはTypes::BookTypeで、objectメソッドはTypes::BookTypeが持つobjectを返します
    - ここでobjectメソッドはbooksテーブルの特定レコードに対応するBookモデルオブジェクトを返します

# セキュリティ向上設定

## DOS攻撃対策

- 自由にクエリを受け取れるので、攻撃もしやすい
- タイムアウトさせる(`use GraphQL!::Schema!::Timeout, max_seconds: 2`)
- クエリのASTで制約
    - 深さ制限(max_depth 10)
    - 複雑さで制限(max_complexity 100)

## スキーマ情報取得を本番環境で禁止する

- GraphQLの仕様でスキーマ情報などを取得するIntrospectionが用意されています
- GraphiQLなどでつかわれています
- スキーマ情報を知られづらくするため、production環境ではoffにしておくと良いです
- https://github.com/rmosolgo/graphql-ruby/blob/master/guides/schema/introspection.md#disabling-introspection

```ruby
class MySchema < GraphQL::Schema
  disable_introspection_entry_points if Rails.env.production?
end
```

## 登録したクエリだけを実行させる Persisted Query

- 実行できるクエリを制限する方法です
- クライアントが送信するクエリをハッシュ値と組にしてサーバに登録しておきます
- クライアントからはクエリではなくハッシュ値を送信します

# まとめ

- GraphQLクエリはリクエストをパス /graphqlにてPOSTメソッドで受け付けます
- 開発環境では `http://localhost:3000/graphiql` でGraphQLクエリリクエストを発行できます
- N+1を回避するために、GraphQL::Batchをつかいます

## 新しいGraphQL fieldを作成するときの手順

- Type作成
    - bin/rails g graphql:object Model名
    - 必要ならば各Typeファイルに関連取得コードも記述
- QueryTypeへfield追加
    - GraphQL::Batch設定をしておけば自動でつかってくれる
- Resolver実装
- 取得系ならQuery実装、更新系ならMutation実装

# 参考資料

- GraphQLの高速道路
    - https://speakerdeck.com/cockscomb/graphql-highway
    - GraphQLの基礎と実践ノウハウ
- WEB+DB PRESS Vol.125 GraphQL完全ガイド
    - https://gihyo.jp/magazine/wdpress/archive/2021/vol125
    - Railsとは関係なくGraphQLについて書かれた特集記事
- GraphQLスキーマのfiled名をモデルのカラム名と別の名前にしたいときはmethodオプションを渡す
    - https://graphql-ruby.org/fields/introduction.html
