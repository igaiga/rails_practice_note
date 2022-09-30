---
title: "[Ruby基礎] debug gem"
---

# debug gem

debug gemは2021年に公開されたRuby用のデバッガです。gemをインストールすることでRuby2.6以降で利用可能です。Ruby3.1以降ではデフォルトでRubyに添付されています。また、Rails7.0以降で標準のデバッガとなっています。

同様の機能を持つツールにbyebug gemがありますが、debug gemはRubyに標準添付されていること、Railsのデフォルトとなっていること、開発が活発であることなどの理由からRubyデバッガの第一候補の地位を築いています。

## Rubyコードでのつかい方

いくつかの起動方法がありますが、ここでは `binding.irb` と同様にコードを書いた場所で一時停止してデバッグコンソールを起動する方法を説明します。

3つの手順でデバッグコンソールを起動することができます。

- gem install debug
  - ターミナルからdebug gemをインストールします
  - Ruby3.1以降では省略することもできますが、実行すると最新のdebug gemを利用できます
- require "debug"
  - コード中にrequire "debug"を書きます
- binding.break
  - コード中の一時停止したい場所でbinding.breakを書きます
  - binding.b や debugger と書いてもOKです

debug gemをつかうサンプルコードは次のようになります。

```ruby
require "debug"

class Foo
  def bar(arg)
    "hi"
    binding.break
  end
end

Foo.new.bar
```

ターミナルで実行すると、次のようにbinding.breakを書いたところで一時停止してデバッグコンソールが起動します。

```console
$ ruby debug_sample.rb
[1, 10] in debug_sample.rb
     1| require "debug"
     2|
     3| class Foo
     4|   def bar(arg)
     5|     "hi"
=>   6|     binding.break
     7|   end
     8| end
     9|
    10| Foo.new.bar(555)
=>#0	Foo#bar(arg=555) at debug_sample.rb:6
  #1	<main> at debug_sample.rb:10
(rdbg)
```

デバッグコンソール上ではirbのようにRubyのコードを実行することができます。`next`コマンドで1行ずつステップ実行、`continue`コマンドで一時停止を解除してプログラムを再開します。`help`コマンドでデバッグコンソール上で利用できるコマンドの説明が表示されます。また、実際の画面では色付きで見やすく表示されます。

一時停止したときのメソッド呼び出し履歴が表示されますが、メソッド呼び出し時に渡された引数もあわせて表示されるのが便利です。

そのほかのつかい方はdebug gemのGitHubページなどを参考にしてください。

- [debug gem GitHub](https://github.com/ruby/debug) 

## Chromeデベロッパーツールをつかってデバッグ

debug gemはターミナルの標準入出力をつかってデバッグコンソールを起動しますが、そのほかにChromeデベロッパーツールから接続してデバッグすることもできます。binding.breakで起動しているデバッグコンソールで`open chrome`コマンドを実行すると、Chromeが起動してデベロッパーツールの画面になります。

```console
$ ruby debug_sample.rb
...
=>   6|     binding.break
...
(rdbg) open chrome
DEBUGGER: wait for debugger connection...
DEBUGGER: Debugger can attach via TCP/IP (127.0.0.1:60024)
DEBUGGER: Connected.
```

![](/images/rails_practice_note/ruby_debug_gem/chrome_dev_tools_with_ruby.png)

ソースコードはSourcesタブに表示され、ConsoleタブをつかうとRubyコードを実行できます。右側のボタンで再開やステップ実行を行えます。

![](/images/rails_practice_note/ruby_debug_gem/chrome_dev_tools_with_ruby_add_comments.png)

## Railsコードでのつかい方

2つの手順でデバッグコンソールを起動することができます。

- Gemfile にdebug gem追加
- binding.break をコードの一時停止したい場所に書く

Gemfileへの追加例です。Rails7.0以降ではデフォルトで追加されています。

```ruby
group :development, :test do
  gem "debug"
end
```

一時停止したい場所にbinding.breakを書きます。次のコードはBooks#indexページへアクセスされたときにデバッグコンソールを起動して一時停止するサンプルコードです。

```ruby
class BooksController < ApplicationController
  def index
    @books = Book.all
    binding.break
  end
...
```

rails sしてブラウザから該当ページへアクセスすると、rails sしているターミナルでデバッグコンソールが起動します。

```console
Started GET "/books" for ::1 at 2022-09-30 09:50:10 +0900
Processing by BooksController#index as HTML
[2, 11] in ~/work/rails7031_ruby312_books_app_debugger/app/controllers/books_controller.rb
     2|   before_action :set_book, only: %i[ show edit update destroy ]
     3|
     4|   # GET /books or /books.json
     5|   def index
     6|     @books = Book.all
=>   7|     binding.break
     8|   end
     9|
    10|   # GET /books/1 or /books/1.json
    11|   def show
=>#0	BooksController#index at ~/work/rails7031_ruby312_books_app_debugger/app/controllers/books_controller.rb:7
  #1	ActionController::BasicImplicitRender#send_action(method="index", args=[]) at ~/.rbenv/versions/3.1.2/lib/ruby/gems/3.1.0/gems/actionpack-7.0.3.1/lib/action_controller/metal/basic_implicit_render.rb:6
  # and 73 frames (use `bt' command for all frames)
(rdbg)
```

デバッグコンソールで`i i`コマンドを実行すると、インスタンス変数のと代入されているオブジェクトの一覧を表示できます。

## Docker中で起動しているRubyプロセスをホスト上のChromeから接続してデバッグ

Docker上でRubyコードを動かしているとき、Docker内のシェルからデバッグコンソールを起動する方法は前に書いた方法と同様です。そのほかの方法として、ホスト上のChromeからTCP/IPで接続してデバッグすることもできます。

次の手順で、Docker中で起動しているRubyプロセスをホスト上のChromeから接続してデバッグすることができます。Dockerコンテナは例としてRubyコアチームでメンテナンスされているrubylang/rubyをつかっています。デバッガーがTCP/IPで接続するポートは空いているところであればどこでも良いので、ここでは45555を指定しています。

Dockerコンテナをポート接続オプションをつけて起動します。`-v /Users/igaiga/work:/work` の部分はホスト側の/Users/igaiga/workをDocker上の`/work`としてマウントする設定です。

(Host)

```console
docker run -v /Users/igaiga/work:/work -p 45555:45555 --rm -it rubylang/ruby /bin/bash
```

Docker上でRubyコードを実行します。debug gemがリッスンする接続用ポートと接続元IPアドレスをそれぞれ環境変数で指定します。接続元IPアドレスは任意のものを許可する0.0.0.0を指定しています。

debug_sample.rbはさきほどと同じコードをつかいます。一時停止したいところに`binding.break`が書いてあります。必要であれば事前に`gem i debug`を実行してdebug gemをインストールします。

(Docker)

```console
gem i debug
cd /work
RUBY_DEBUG_PORT=45555 RUBY_DEBUG_HOST=0.0.0.0 ruby debug_sample.rb
```

デバッグコンソールが起動するするので、open chromeコマンドを実行します。

(デバッグコンソール)

```
open chrome
...
devtools://devtools/bundled/inspector.html?v8only=true&panel=sources&ws=0.0.0.0:45555/06d8bd93-a7b9-406d-9b4a-a3d1a03ac8b2
...
```

表示されるURL `devtools://devtools/bundled/inspector.html?v8only=true&panel=sources&ws=0.0.0.0:45555/06d8bd93-a7b9-406d-9b4a-a3d1a03ac8b2` へホスト上で起動するChromeからアクセスすることでデバッグできます。

## Docker上で起動しているRailsアプリをホスト上のChromeからデバッグ

TODO: 書きかけです

Dockerの中でopen chromeしてネイティブのChromeから接続する

- $ docker pull cimg/ruby:3.1.2
    - CircleCIが提供するイメージをつかう

(ホスト)

- /Users/igaiga/work/rails_app_name にデバッグ対象のRailsアプリを置いておく
- ポートはrails server用に3000、debug用に45555を指定しておく

```
docker run -v /Users/igaiga/work:/work -p 45555:45555 -p 3000:3000 --rm -it cimg/ruby:3.1.2 /bin/bash
```

(docker)

```
cd /work/rails_app_name
bundle install
RUBY_DEBUG_PORT=45555 RUBY_DEBUG_HOST=0.0.0.0 bin/rails s -b 0.0.0.0
```

- native Chrome から localhost:3000 へアクセス

(Rails code)

```
binding.break
```

(debug)
```
open chrome
...
devtools://devtools/bundled/inspector.html?v8only=true&panel=sources&ws=0.0.0.0:45555/da94b5bd-bb1d-4244-aa65-9046eae9a841
...
```

- ホスト上 ChromeからURL `devtools://devtools/bundled/inspector.html?v8only=true&panel=sources&ws=0.0.0.0:45555/da94b5bd-bb1d-4244-aa65-9046eae9a841` で接続

# 謝辞

本記事を書くにあたり、udzuraさん、笹田さんに助けていただきました。ありがとうございます。また、debug gemを熱心に開発してくださっている笹田さん、Onoさん、Stanさんに感謝します。
