---
title: "[Rails基礎] RackとRackミドルウェア"
---

# RackとRackミドルウェア

## Rack とは

- RackはRubyでWebアプリを作るときの標準仕様
- RailsはRackアプリ(Rack仕様準拠)
- pumaやunicornなどはRackアプリを動作させるためのプログラム

## Rackミドルウェアとは

- Railsアプリでは、一番外側にRackミドルウェア群が配置されている
- つまり routes - controller - view よりも外側の世界があり、それらがRackミドルウェア群
- Railsアプリはデフォルトで20ほどのRackミドルウェアが使われていて、リクエストとレスポンスはその全てを通過する

## RailsもRackアプリ

RailsもRackアプリです。rails serverコマンドをつかわずに、Rackアプリを起動させるrackupコマンドをつかって、rails serverを起動してみましょう。

rackupコマンドを使うときは、config.ruファイルが必要です。rails newした状態では、Railsアプリケーションのルートディレクトリに次のような内容で作成されています。

- config.ru
```ruby
require_relative "config/environment"

run Rails.application
Rails.application.load_server
```

rackupコマンドでrails serverを起動してみましょう。

- $ rackup config.ru

起動後にブラウザから`http://localhost:9292`へアクセスできることを確認してみてください。ポート番号が9292であることに注意してください。

## rails serverコマンドの解説

rails serverコマンドは、Rack::Serverオブジェクトを作成し、Webサーバーを起動します。

rails serverコマンドは、以下のようにRack::Serverのオブジェクトを作成します。

```ruby
Rails::Server.new.tap do |server|
  require APP_PATH
  Dir.chdir(Rails.application.root)
  server.start
end
```

Rails::ServerクラスはRack::Serverクラスを継承しており、以下のようにRack::Server#startを呼び出します。

```ruby
class Server < ::Rack::Server
  def start
    ...
    super
  end
end
```

## 小さなRackアプリづくり

小さなRackアプリをつくって動作を観察してみましょう。空のフォルダに次の内容でminimum_app.ruファイルを作成します。

- minimum_app.ru
```ruby
App = lambda do |env|
  [200, { "content-type" => "text/html"}, ["Hello world!"]]
end
run App
```

Rackアプリはcallメソッドを呼ばれたときに、次の3つの要素を持った配列を返す規約になっています。

- HTTPステータスコードとなるIntegerオブジェクト
- HTTPヘッダとなるHashオブジェクト
- HTTPレスポンスボディとなる文字列を詰めたArrayオブジェクト

rackupコマンドを実行してブラウザからアクセスすると、Hello world!がブラウザに表示されます。

- $ gem install rack
- $ rackup minimum_app.ru

## 小さなRackミドルウェアづくり

次は小さなRackミドルウェアをつくって、Railsでつかってみましょう。

### rails middlewareコマンド

Railsアプリに登録されているRackミドルウェアを一覧表示する`rails middleware`コマンドが用意されています。

```shell
$ rails middleware
use ActionDispatch::HostAuthorization
use Rack::Sendfile
use ActionDispatch::Static
use ActionDispatch::Executor
use ActionDispatch::ServerTiming
use ActiveSupport::Cache::Strategy::LocalCache::Middleware
use Rack::Runtime
use Rack::MethodOverride
use ActionDispatch::RequestId
use ActionDispatch::RemoteIp
use Sprockets::Rails::QuietAssets
use Rails::Rack::Logger
use ActionDispatch::ShowExceptions
use WebConsole::Middleware
use ActionDispatch::DebugExceptions
use ActionDispatch::ActionableExceptions
use ActionDispatch::Reloader
use ActionDispatch::Callbacks
use ActiveRecord::Migration::CheckPending
use ActionDispatch::Cookies
use ActionDispatch::Session::CookieStore
use ActionDispatch::Flash
use ActionDispatch::ContentSecurityPolicy::Middleware
use ActionDispatch::PermissionsPolicy::Middleware
use Rack::Head
use Rack::ConditionalGet
use Rack::ETag
use Rack::TempfileReaper
run Rails703RackAndRackMiddleware::Application.routes
```

### Rackミドルウェアのつくり方

小さなRackミドルウェアの例として、レスポンスボディの文字列を大文字にするRackミドルウェアを書いてみます。

- upcase.ru
```ruby
# Rackミドルウェア
class UpcaseAll
  def initialize(app)
    @app = app
  end

  def call(env)
    status_code, headers, body = @app.call(env)
    body.each{|part| part.upcase! }
    [status_code, headers, body]
  end
end
use UpcaseAll

# テスト用Rackアプリ
App = lambda do |env|
  [200, {"content-type" => "text/html"}, ["Hello, Rack world!"]]
end
run App
```

- $ rackup upcase.ru

Rackミドルウェアをつくるためには、クラスをつくってcallメソッドを実装します。Rackミドルウェア群に組み込まれリクエストが来るとcallメソッドを呼び出してもらえます。callメソッドでRackミドルウェアでやりたいことを書きます。callメソッドの戻り値として `[status_code, headers, body]` を返します。ここで返したオブジェクトが他のRackミドルウェア群を通過して、最終的にレスポンスとしてブラウザへ返ります。

initializeメソッドは呼び出されるときに引数として組み込まれるRackアプリオブジェクトを渡されます。このコードではAppへ代入されたテスト用Rackアプリオブジェクトです。これをインスタンス変数`@app`に代入しておきます。callメソッドが呼ばれたときに `@app.call(env)` でRackアプリを実行して結果を得ています。callメソッドの引数`env`にはリクエストの情報などが入っています。

## つくったRackミドルウェアをRailsへ組み込む

hello と標準出力に表示するRackミドルウェアをつくってRailsへ組み込んでみます。まずはRackミドルウェアをつくって単体で起動確認してみます。

- hello.ru
```ruby
# Rackミドルウェア
class Hello
  def initialize(app)
    @app = app
  end

  def call(env)
    code, headers, body = @app.call(env)
    p "===== Hello! ====="
    return [code, headers, body]
  end
end

# テスト用アプリ（テスト時rackupのときだけ使い、組み込むときはコメントアウト）
App = lambda do |env|
 [200, {"content-type" => "text/html"}, ["Hello, Rack world!"]]
end
use Hello
run App
```

rackupできることを動作確認しておきます。

つづいてRailsへ組み込みます。config/initializers/hello.rbを次のようにつくります。ファイル拡張子をrbにしています。

- config/initializers/hello.rb
```ruby
# Rackミドルウェア
class Hello
  def initialize(app)
    @app = app
  end

  def call(env)
    code, headers, body = @app.call(env)
    p "===== Hello! ====="
    return [code, headers, body]
  end
end

# RailsへRackミドルウェアとして組み込み
Rails.application.config.middleware.use Hello
```

rails serverを起動してブラウザからアクセスし、標準出力に===== Hello! =====が表示されること、rails middlewareコマンドでRackミドルウェアとして表示されることも確認してみてください。

- $ rails middleware

```
...
use Hello
...
```

`Rails.application.config.middleware.use クラス名` で指定したクラスをRailsアプリへ組み込み、Rackミドルウェアとして利用することができます。

Rackミドルウェアは実行される順番が大事になることもあるので、次のように指定したところへ差し込むメソッドも用意されています。

- config.middleware.use(new_middleware)
  - 末尾に新規ミドルウェアnew_middlewareを追加
- config.middleware.insert_before(existing_middleware, new_middleware)
  - 新規ミドルウェアnew_middlewareを既存ミドルウェアexisting_middlewareの直前に追加
- config.middleware.insert_after(existing_middleware, new_middleware)
  - 新規ミドルウェアnew_middlewareを既存ミドルウェアexisting_middlewareの直後に追加
- config.middleware.move_before(target_middleware, moving_middleware)
  - 既存ミドルウェアmoving_middlewareを既存ミドルウェアtarget_middlewareの直前へ移動
- config.middleware.move_after(target_middleware, moving_middleware)
  - 既存ミドルウェアmoving_middlewareを既存ミドルウェアtarget_middlewareの直後へ移動
- config.middleware.swap(out_middleware, in_middleware)
  - 既存ミドルウェアout_middlewareを外して、新規ミドルウェアin_middlewareを追加
- config.middleware.delete(middleware)
  - 既存ミドルウェアmiddlewareを削除

これらのメソッドの説明はRails APIのページが参考になります。

https://api.rubyonrails.org/classes/Rails/Configuration/MiddlewareStackProxy.html

## RailsのRackミドルウェアをつかったエラーハンドリング

Railsアプリにはエラーハンドリングを行うRack Middlware "ActionDispatch::ShowExceptions"（actionpack/lib/action_dispatch/middleware/show_exceptions.rb）がデフォルトで組み込まれています。

たとえば次のような例外をコントローラでrescueしなくても、自動的に適切なステータスコードのレスポンスへと変換してくれます。

- ActionController::RoutingError ➡️ 404
- ActionController::BadRequest ➡️ 400

例外とステータスコードの対応は `ActionDispatch::ExceptionWrapper` (actionpack/lib/action_dispatch/middleware/exception_wrapper.rb) クラスに書かれています。

新しい対応を書くこともできます。通常は500エラーになるFooExceptionを404にするコードは次のようになります。

- config/application.rb
```ruby
class FooException < Exception
end
Rails.application.config.action_dispatch.rescue_responses.merge!({ "FooException" => :not_found })
```

コントローラで`raise FooException.new`してFooException例外を投げます。

development環境だとデバッグ用のエラー表示になってしまうので、productionモードでrails serverを起動します。

```
$ rails s -e production
```

ブラウザからアクセスして404エラーになることを確認してみてください。

## 参考資料

- Rails Guide 「RailsとRack」
    - https://railsguides.jp/rails_on_rack.html
- Rails API Rails::Configuration::MiddlewareStackProxy(Rackミドルウェアを追加するメソッド群)
    - https://api.rubyonrails.org/classes/Rails/Configuration/MiddlewareStackProxy.html
- パーフェクトRails 増補改訂版
    - 8-3-1 エラーハンドリング
    - 8-3-2 Rack Middlewareを利用したエラーハンドリング
- 実録 Let's build a simple Rack compatible server
    - https://speakerdeck.com/coe401_/shi-lu-lets-build-a-simple-rack-compatible-server

## サンプルコード

- https://github.com/igaiga/rails703_rack_and_rack_middleware
