---
title: "[ActiveRecord] フォームオブジェクト"
---

# フォームオブジェクト

## フォームオブジェクトとは

フォームオブジェクトはフォームからの入力を検証したり、form_withメソッドにモデルのように渡すことができるオブジェクトです。

## フォームオブジェクトのつくりかた

ActiveModel::API, ActiveModel::Attributes をつかうとRailsが提供する道具でフォームオブジェクトをつくれます。ActiveRecordを継承せずにつくるのが一般的です。

- ActiveModel::Attributes
  - 型(cast type)を指定したattributesをつくれる
- ActiveModel::API
  - form_withに渡せたり、validationできたり、newメソッドでattributesと一緒に初期化できる機能などを提供

ここでつくるサンプルコードは以下からダウンロードできます。

https://github.com/igaiga/rails_form_object_sample_app

## ActiveModel::Attributesモジュール

型を持つattributes(カラム的なもの)をかんたんに定義できるようにします。attributeメソッドに名前と型(cast type)を指定して定義できます。

```ruby
class FooFormObject
  include ActiveModel::Attributes

  attribute :name, :string
  attribute :email, :string
  attribute :terms_of_service, :boolean
end
```

## ActiveModel::APIモジュール

モデルのように振る舞う機能各種をつかえるようになります。ActiveModel::APIはRails7.0からつかえるようになりました。それより前のバージョンでは、ActiveModel::Modelをつかうことで同様の機能をつかうことができます。

- バリデーションを設定して実行できる機能
- form_withとやりとりする機能
- newメソッドでattributesと一緒に初期化する機能 ほか
  - `FooFormObject.new(name: "iga", email: "iga@example.com", terms_of_service: true)`

```ruby
class FooFormObject
  include ActiveModel::API
  include ActiveModel::Attributes

  attribute :name, :string
  attribute :email, :string
  attribute :terms_of_service, :boolean

  validates :name, presence: true
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
  # URI::MailTo::EMAIL_REGEXPはRubyに定義されてるemail検証正規表現
  validates :terms_of_service, acceptance: { allow_nil: false }
  # acceptanceはチェックボックス確認用 https://railsguides.jp/active_record_validations.html#acceptance
```

## 今回のフォームオブジェクトのサンプルアプリ仕様

Userモデルのattributesである name, emailを入力するフォームが既にあるところに、新しくnameをひらがなでusersテーブルに保存するフォームが追加されたときを想定しています。nameをひらがなでusersテーブルに保存するフォーム用のフォームオブジェクトとしてUserNameFormをつくっていきます。

![](/images/rails_practice_note/ar_form_object/form_object_app_specs.png)

## Userモデルのattributesとバリデーション

app/models/user.rb

```ruby
# DB schema
# create_table "users" do |t|
#   t.string "name", null: false
#   t.string "email"
#
class User < ApplicationRecord
  validates :name, presence: true
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }, allow_blank: true
end
```

Userモデル

- nameは必須
- emailはblank可で、フォーマットの確認をする

`URI::MailTo::EMAIL_REGEXP`はRubyに定義されてるemail検証正規表現です。

## その1 基礎工事

- 名前は **UserNameForm** とします
- app/forms/user_name_form.rb
  - 置くフォルダは app/forms とます
- attributesとして`attribute :name, :string` を持ちます
- バリデーション `validates :name, presence: true`

app/forms/user_name_form.rb

```ruby
class UserNameForm
  include ActiveModel::API
  include ActiveModel::Attributes
  attribute :name, :string

  # このフォームオブジェクトのバリデーション
  validates :name, presence: true
end
```

## その2 Userモデルをフォームオブジェクトへ渡す

- Userモデルに仕事を委譲するために`attr_accessor :user`で設定取得可能に
  - DB保存機能を委譲するためにUserモデルを設定可能に
  - URLヘルパーで `xxx_path(user)` のようにつかいたいので取得も可能に
- transfer_attributesメソッドでフォームオブジェクトからモデルへattributesセット

app/forms/user_name_form.rb

```ruby
class UserNameForm
  # ...(略)...

  attr_accessor :user

  # フォームオブジェクトからモデルへattributesをセット
  def transfer_attributes
    user.name = name
  end
end
```

## その3 saveメソッド

- DB保存機能をsaveメソッドとして実装
  - モデルと似た書き味を目指します

app/forms/user_name_form.rb

```ruby
def save(...) # ... は全引数を引き渡す記法
  transfer_attributes # フォームオブジェクトからモデルへattributesをセット
  if valid? # フォームオブジェクトのバリデーション実行
    user.save(...) # モデルのsaveメソッドへ委譲
  else
    false # これがないとvalid?失敗時にnilが返る
  end
  # valid? && user.save(...) # 短く書いても良い
end
```

## その4 initializeメソッド

- `UserNameForm.new(name_params)` フォームからのparamsで初期化する想定
- 追加実装しなくてもnewメソッドでattributesを設定可能
  - `UserNameForm.new(name: "iga")`
- 今回はnewメソッドへattributesのほかにUserモデルも渡したい
  - initializeメソッドをオーバーライドして機能追加

app/forms/user_name_form.rb

```ruby
def initialize(model: nil, **attrs) # `**`はキーワード引数をHashで受け取る文法
  attrs.symbolize_keys! # StringとSymbolの両対応
  if model
    @user = model
    attrs = {name: @user.name}.merge(attrs) # attrsがあれば優先
  end
  super(**attrs) # もともとのinitializeメソッドを呼び出し
  # `**`はHashをキーワード引数で渡す文法
end
```

## その5 コントローラの変更

### その5-1 コントローラの変更 StrongPrameters

- NamesControllerとそのroutes, viewのベース部分はscaffoldでつくったコードの想定
- StrongPrametersを変更
  - フォームからのparamsにあわせて`require(:user_name_form)`へ

app/controllers/names_controller.rb

```ruby
class NamesController < ApplicationController

  private

  def name_params
    params.require(:user_name_form).permit(:name)
  end
end
```

### その5-2 コントローラの変更 new & createアクション

- `@user_name_form` 変数へUserNameFormオブジェクトを代入
- リダイレクト先のURL取得を変更

app/controllers/names_controller.rb

```ruby
class NamesController < ApplicationController
  def new
    @user_name_form = UserNameForm.new(model: User.new)
  end

  def create
    @user_name_form = UserNameForm.new(model: User.new, **name_params)

    if @user_name_form.save
      redirect_to user_url(@user_name_form.user), notice: "User was successfully created."
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```

### その5-3 コントローラの変更 edit & updateアクション

- `@user_name_form` 変数へUserNameFormオブジェクトを代入
- リダイレクト先のURL取得を変更

app/controllers/names_controller.rb
```ruby
class NamesController < ApplicationController
  def edit
    @user_name_form = UserNameForm.new(model: User.find(params[:id]))
  end

  def update
    @user_name_form = UserNameForm.new(model: User.find(params[:id]), **name_params)

    if @user_name_form.save
      redirect_to user_url(@user_name_form.user), notice: "User was successfully updated."
    else
      render :edit, status: :unprocessable_entity
    end
  end
end
```

## その6 viewの変更

### その6-1 form_withでフォームオブジェクトをつかう

- form_withのmodelオプションにはフォームオブジェクトを渡すことにします
- コントローラの変更で変数user_name_formにフォームオブジェクトが入っている

app/views/names/_form.html.erb
```erb
<%= form_with(model: user_name_form) do |form| %>
```

- 次のエラーが出ます

```
undefined method `user_name_forms_path' for an instance of #<Class:0x...>
```

- フォームのリクエスト先パスがわからない旨のエラー
- 今回はform_withへurl, methodオプションでリクエスト先を指定する方法で対応

### その6-2 form_withへurl, methodオプションを指定

- form_withのurl, methodオプションでリクエスト先を指定
  - モデルが**未保存**のときは**createアクション**へ
  - モデルが**保存済み**のときは**updateアクション**へ

app/views/names/_form.html.erb

```erb
<%# 実際は改行なし %>
<% form_with_options = user_name_form.user.persisted? ?
  { url: name_path(user_name_form.user), method: :patch } :
  { url: names_path, method: :post } %>
<%= form_with(model: user_name_form, **form_with_options) do |form| %>
```

- `model.persisted?` メソッドはDB保存済みかどうかを判定
- `**form_with_options`の`**`はHashをキーワード引数で渡す文法
- ビューに書くと読みづらいのでフォームオブジェクトに移動します

### その6-3 form_withのオプションをフォームオブジェクトから取得

app/forms/user_name_form.rb

```ruby
  def form_with_options
    if user.persisted?
      # update用
      { url: Rails.application.routes.url_helpers.name_path(user), method: :patch }
    else
      # create用
      { url: Rails.application.routes.url_helpers.names_path, method: :post }
    end
  end
```

- `Rails.application.routes.url_helpers.names_path`
  - ビュー以外でURLヘルパーメソッド(names_pathなど)をつかう方法

app/views/names/_form.html.erb
```erb
<%= form_with(model: user_name_form, **user_name_form.form_with_options) do |form| %>
```

これで完成です！ 🎉🎊🎂

## フォームオブジェクト最終形

```ruby
class UserNameForm
  include ActiveModel::API
  # バリデーション機能、form_withに渡せる機能、
  # new(name: "xxx", ...)のようにattributesとあわせて初期化する機能などを足す
  include ActiveModel::Attributes # 型を持つattributesをかんたんに定義できるようにする

  attribute :name, :string

  # このフォームオブジェクトのバリデーション
  validates :name, presence: true

  # DB保存などの機能を委譲するためにUserモデルをセット可能に
  # redirect_to @user のときなどUserモデルを取りたいので取得もできるようにする
  attr_accessor :user

  def initialize(model: nil, **attrs) # `**`はキーワード引数をHashで受け取る文法
    attrs.symbolize_keys! # StringとSymbolの両対応
    if model
      @user = model
      attrs = {name: @user.name}.merge(attrs) # attrsがあれば優先
    end
    super(**attrs) # もともとのinitializeメソッドを呼び出し
    # `**`はHashをキーワード引数で渡す文法
  end

  def save(...) # ... は全引数を引き渡す記法
    transfer_attributes # フォームオブジェクトからモデルへattributesをセット
    if valid? # フォームオブジェクトのバリデーション実行
      user.save(...)  # モデルのsaveメソッドへ委譲
    else
      false # これがないとvalid?失敗時にnilが返る
    end
    # valid? && user.save(...) # 短く書いても良い
  end

  def form_with_options
    if user.persisted?
      # update用
      { url: Rails.application.routes.url_helpers.name_path(user), method: :patch }
    else
      # create用
      { url: Rails.application.routes.url_helpers.names_path, method: :post }
    end
  end

  private

  # フォームオブジェクトからモデルへattributesをセット
  def transfer_attributes
    user.name = name
  end
end
```

## 参考資料

- サンプルコード: https://github.com/igaiga/rails_form_object_sample_app

- Rails API
  - [ActiveModel::API](https://api.rubyonrails.org/classes/ActiveModel/API.html)
  - [ActiveModel::Attributes](https://api.rubyonrails.org/classes/ActiveModel/Attributes.html)
  - [ActiveModel::Model](https://api.rubyonrails.org/classes/ActiveModel/Model.html)
- [パーフェクトRails増補改訂版](https://gihyo.jp/book/2020/978-4-297-11462-6) 12-3 フォームオブジェクト節
