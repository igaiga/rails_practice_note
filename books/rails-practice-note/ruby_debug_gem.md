---
title: "[Ruby基礎] debug gem"
---

# debug gem

debug gemは2021年に公開されたRuby用のデバッガです。gemをインストールすることでRuby2.6以降で利用可能です。Ruby3.1以降ではデフォルトでRubyに添付されています。また、Rails7.0以降でデフォルトで入っているデバッガとなっています。

同様の機能を持つツールにbyebug gemがありますが、debug gemはRubyに標準添付されていること、Railsにデフォルトで入っているデバッガであること、開発が活発であること、などの理由からRubyデバッガの第一候補の地位を築いています。

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

debug_sample.rb

```ruby
require "debug"

class Foo
  def bar(arg)
    "hi"
    binding.break
  end
end

Foo.new.bar(555)
```

事前にターミナルで `gem install debug` を実行してから、サンプルコードを実行します。


```console
$ gem install debug
$ ruby debug_sample.rb
```

ターミナルで実行すると、次のようにbinding.breakを書いたところで一時停止してデバッグコンソールが起動します。実際の画面では色つきで見やすく表示されます。

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

一時停止したときのメソッド呼び出し履歴が表示されますが、メソッド呼び出し時に渡された引数もあわせて表示されるのが便利です。

デバッグコンソール上ではirbのようにRubyのコードを実行することができます。`next` or `n` コマンドで1行ずつステップ実行、`continue` or `c` コマンドで一時停止を解除してプログラムを再開します。`help`コマンドでデバッグコンソール上で利用できるコマンドの説明が表示されます。

コマンドの一覧や、そのほかのつかい方はdebug gemのGitHubページなどを参考にしてください。

- [debug gem GitHub](https://github.com/ruby/debug) 
- [コマンド一覧](https://github.com/ruby/debug#debug-command-on-the-debug-console)

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

もしも「binding.break + open chrome」相当の処理をRubyコードへ埋め込みたいときは、`binding.break pre: "open chrome"` と書きます。

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

ソースコードの一時停止したい場所にbinding.breakを書きます。次のコードはBooks#indexページへアクセスされたときにデバッグコンソールを起動して一時停止するサンプルコードです。

```ruby
class BooksController < ApplicationController
  def index
    @books = Book.all
    binding.break
  end
...
```

ターミナルでrails sを起動してブラウザから該当ページへアクセスすると、そのターミナルでデバッグコンソールが起動します。

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

デバッグコンソールで`i i`コマンドを実行すると、インスタンス変数と代入されているオブジェクトの一覧を表示できます。ほかの変数を見るコマンドは、`help` コマンドや前述のコマンド一覧のページを参考にしてください。

ここでもChromeデベロッパーツールをつかいたいときには `open chrome` コマンドを実行します。

## Docker中で起動しているRubyプロセスをホスト上のChromeから接続してデバッグ

Docker上でRubyコードを動かしているとき、Docker内のシェルの標準入出力をつかってデバッグコンソールを起動する方法は前に書いた方法と同様です。そのほかの方法として、ホスト上のChromeからTCP/IPで接続してデバッグすることもできます。

次の手順で、Docker中で起動しているRubyプロセスをホスト上のChromeから接続してデバッグすることができます。Dockerコンテナは例としてRubyコアチームが提供しているrubylang/rubyをつかっています。デバッガーがTCP/IPで接続するポートは空いているところであればどこでも良いので、ここでは45555を指定しています。

Dockerコンテナをポート接続オプションをつけて起動します。`-v /Users/igaiga/work:/work` の部分はホスト側の/Users/igaiga/workをDocker上の`/work`としてマウントする設定です。

(Host)

```console
docker run -v /Users/igaiga/work:/work -p 45555:45555 --rm -it rubylang/ruby /bin/bash
```

Docker上でRubyコードを実行します。debug gemがリッスンする接続用ポートと接続元IPアドレスをそれぞれ環境変数で指定します。接続元IPアドレスは任意のものを許可する0.0.0.0を指定しています。

debug_sample.rbはさきほどと同じコードをつかいます。一時停止したいところに`binding.break`が書いてあります。必要であれば事前に`gem install debug`を実行してdebug gemをインストールします。

(Docker)

```console
gem install debug
cd /work
RUBY_DEBUG_PORT=45555 RUBY_DEBUG_HOST=0.0.0.0 bin/rails s
```

デバッグコンソールが起動するするので、open chromeコマンドを実行します。

(デバッグコンソール)

```
open chrome
...
devtools://devtools/bundled/inspector.html?v8only=true&panel=sources&ws=0.0.0.0:45555/06d8bd93-a7b9-406d-9b4a-a3d1a03ac8b2
...
```

ホスト上で起動するChromeから、表示されるURL `devtools://devtools/bundled/inspector.html?v8only=true&panel=sources&ws=0.0.0.0:45555/06d8bd93-a7b9-406d-9b4a-a3d1a03ac8b2` へアクセスすることでデバッグできます。

## Docker上で起動しているRailsアプリをホスト上のChromeからデバッグ

次は、Docker上で動いているRailsアプリをホスト上のChromeからデバッグするときの手順です。先ほどと同様の手順に加えて、rails serverでアクセスする3000番ポートも接続する設定でDockerを起動します。Dockerコンテナは例としてCircleCIが提供しているcimg/rubyをつかっています。マウント設定 `-v /Users/igaiga/work:/work` のホスト側パス `/Users/igaiga/work` の下にRailsアプリのフォルダ(以下の例ではrails_app_name)を置きます。

(ホスト)

```
docker run -v /Users/igaiga/work:/work -p 45555:45555 -p 3000:3000 --rm -it cimg/ruby:3.1.2 /bin/bash
```

rails sを起動するときに`-b 0.0.0.0`オプションを加えて、rails serverに任意のIPアドレスからアクセスできるようにします。

(Docker)

```
cd /work/rails_app_name
bundle install
RUBY_DEBUG_PORT=45555 RUBY_DEBUG_HOST=0.0.0.0 bin/rails s -b 0.0.0.0
```

ほかはRubyコードでの手順と同様です。Railsアプリで一時停止したい場所に `binding.break` を追記して、デバッグコンソールが起動したらopen chromeコマンドを実行、表示されるURL `devtools://...` へホスト上のChromeからアクセスしてデバッグできます。「binding.break + open chrome」相当の処理をRubyコードへ埋め込みできる `binding.break pre: "open chrome"` をつかうのも便利です。

# 参考文献

- ruby/debug GitHub
  - https://github.com/ruby/debug
  - debug gemのGitHubページです
- Ruby: byebugからruby/debugへの移行ガイド（翻訳）
  - https://techracho.bpsinc.jp/hachi8833/2022_09_01/121134
  - Stanさんのdebug gemつかい方記事の日本語訳です
- Introduction of Tools for providing rich user experience in debugger
  - https://www.slideshare.net/NaotoOno1/introduction-of-tools-for-providing-rich-user-experience-in-debugger
  - OnoさんのRubyKaigi2022での講演資料。Chromeデベロッパーツールをつかい方とその開発について話されています。

# 謝辞

本記事を書くにあたり、udzuraさん([@udzura](https://twitter.com/udzura))、笹田さん([@_ko1](https://twitter.com/_ko1))に助けていただきました。ありがとうございます。また、debug gemを熱心に開発してくださっている笹田さん([@_ko1](https://twitter.com/_ko1))、Onoさん([@ono_max7](https://twitter.com/ono_max7))、Stanさん([@_st0012](https://twitter.com/_st0012))に感謝します。
