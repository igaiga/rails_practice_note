---
title: "[Ruby基礎][Rails基礎] debug gem"
---

# debug gem

[debug gem](https://github.com/ruby/debug)は2021年に公開されたRuby用のデバッガです。gemをインストールすることでRuby2.6以降で利用可能です。Ruby3.1以降ではbundled gemとしてRubyにデフォルトで添付されています。また、Rails7.0以降でデフォルトで入っているデバッガとなっています。

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

## デバッグコンソールのつかい方

デバッグコンソール上でつかえる基本的なコマンドを紹介していきます。

### 基本的なコマンド

次のコマンドを知っておくだけでirbと同程度にdebug gemをつかうことができます。デバッグコンソール上ではdebug gemが提供するコマンドのほか、irbと同じくRubyコードを実行することもできます。

- next or n
  - 1行ずつステップ実行
- continue or c
  - 一時停止を解除してプログラムを再開
- help or ?
  - 利用できるコマンドを表示

### info コマンド, ls コマンド

infoコマンド(iと省略可能)で変数一覧とそれらの情報を表示します。lsコマンドも同様に情報を表示する機能です。

- i l
  - ローカル変数と代入されたオブジェクトを表示
- i i
  - selfが持つインスタンス変数と代入されたオブジェクトを表示
- i i <object>
  - objectが持つインスタンス変数と代入されたオブジェクトを表示
- i c
  - アクセス可能な定数と代入されたオブジェクトを表示(トップレベルのものを除く)
- ls
  - 現在のスコープでつかえるメソッド、定数、ローカル変数、インスタンス変数を表示
- ls <object>
  - objectのスコープでつかえるメソッド、インスタンス変数を表示

### trace コマンド

trace コマンドを指定しておくと、以降の処理で指定したイベントが発生したときにその情報を表示します。Rubyの組み込みライブラリであるTracePointに定義されているイベント群に近いイベントが用意されています。

- trace call

メソッド呼び出しとreturnのときにその情報を表示します。returnのときは、その戻り値も表示します。たくさん発生するイベントなので、次の正規表現をつかって絞ると便利です。

- trace call /regexp/

/regexp/ 正規表現にマッチした、メソッド呼び出しとreturnの情報を表示します。たとえば `trace call /each/` ではeachと名前が入ったメソッドに関する情報だけを表示します。/regexp/ はtrace callのほか、全てのtraceコマンドでつかえます。

- trace exception

例外が投げられたときにその情報を表示します。通常、rescueされた例外に気づくことはありませんが、trace exceptionコマンドを有効にしておくことで、どんな例外が投げられてrescueされたのかを観察することができます。また、 catch <ExceptonName> コマンドをつかうと、ExceptonName例外が投げられたときに一時停止してデバッグコンソールを起動できます。

応用例として、`binding.break do: "trace exception"` と `binding.break do: "trace off exception"` で処理を囲むと、その間で投げられた例外が表示されます。binding.break do はそこでコマンド入力待ちにならずに、do以降のコマンドを実行してプログラム実行を再開するため、入力で待つことなく高速に試行できます。

```
binding.break do: "trace exception"
...処理... (この間に投げられた例外が表示される)
binding.break do: "trace off exception"
```

### watchコマンド

- watch <@instance_variable_name>

watchコマンドへインスタンス変数名を指定すると、(selfが持っている)指定されたインスタンス変数が変更されたときに、一時停止してデバッグコンソールを起動できます。

### step backコマンドとrecord onコマンド

nextコマンドで1行ずつコードを実行することができますが、step backコマンドは逆に1行ずつ行ったコード実行を戻っていく機能です。ただし、事前にrecord onコマンドを実行しておく必要があり、step backでさかのぼることができるのはrecord onコマンドを実行した時点までです。

record onコマンドはデバッグコンソールで実行する方法のほか、前に出てきた `binding.break do:` をつかってコード中に書くのも便利です。

```ruby
binding.break do: "record on" # ここまで戻ることが可能
# ...
binding.break # ここでデバッグコンソール起動、step backコマンドで1行戻る
```

### コマンドの調べ方

コマンドの一覧や、そのほかのつかい方はdebug gemのGitHubページなどを参考にしてください。

- [debug gem GitHub](https://github.com/ruby/debug)
  - [debug gem GitHub コマンド一覧](https://github.com/ruby/debug#debug-command-on-the-debug-console)
- [ruby/debug cheatsheet](https://st0012.dev/ruby-debug-cheatsheet)

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

Docker上でRubyコードを実行します。debug gemがリッスンする接続用ポートを環境変数 `RUBY_DEBUG_PORT` で、接続元IPアドレスを環境変数 `RUBY_DEBUG_HOST` で指定します。接続元IPアドレスは任意のものを許可する0.0.0.0を指定します。

debug_sample.rbはさきほどと同じコードをつかいます。一時停止したいところに`binding.break`が書いてあります。必要であれば事前に`gem install debug`を実行してdebug gemをインストールします。

(Docker)

```console
gem install debug
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

ホスト上で起動するChromeから、表示されるURL `devtools://devtools/bundled/inspector.html?v8only=true&panel=sources&ws=0.0.0.0:45555/06d8bd93-a7b9-406d-9b4a-a3d1a03ac8b2` へアクセスすることでデバッグできます。

## Docker上で起動しているRailsアプリをホスト上のChromeからデバッグ

次は、Docker上で動いているRailsアプリをホスト上のChromeからデバッグするときの手順です。先ほどと同様の手順に加えて、rails serverでアクセスする3000番ポートも接続する設定でDockerを起動します。Dockerコンテナは例としてCircleCIが提供しているcimg/rubyをつかっています。マウント設定 `-v /Users/igaiga/work:/work` のホスト側パス `/Users/igaiga/work` の下にRailsアプリを置いておきます。以下の例ではRailsアプリ名はrails_app_nameとしています。

(ホスト)

```
docker run -v /Users/igaiga/work:/work -p 45555:45555 -p 3000:3000 --rm -it cimg/ruby:3.1.2 /bin/bash
```

rails sを起動するときに`-b 0.0.0.0`オプションを加えて、rails serverが任意のIPアドレスからアクセス可能になるよう設定します。

(Docker)

```
cd /work/rails_app_name
bundle install
RUBY_DEBUG_PORT=45555 RUBY_DEBUG_HOST=0.0.0.0 bin/rails s -b 0.0.0.0
```

ほかはRubyコードでの手順と同様です。Railsアプリで一時停止したい場所に `binding.break` を追記して、デバッグコンソールが起動したらopen chromeコマンドを実行、表示されるURL `devtools://...` へホスト上のChromeからアクセスしてデバッグできます。「binding.break + open chrome」相当の処理をRubyコードへ埋め込みできる `binding.break pre: "open chrome"` をつかうのも便利です。

## docker-compose.ymlをつかって起動しているRailsアプリをホスト上のChromeからデバッグ

次は、docker-compose.yml上で起動したRailsアプリをホスト上のChromeからデバッグするときの手順です。先ほどと同様にrails s用の3000番ポートとdebug gem用の45555番ポートを開ける設定でdocker-compose.ymlを書きます。さきほどと同様にDockerコンテナとしてCircleCIが提供しているcimg/rubyをつかうdocker-compose.ymlの例は次のようになります。

```docker-compose.yml
version: '3'

services:
  app:
    image: cimg/ruby:3.1.2
    ports:
      - "45555:45555"
      - "3000:3000"
    volumes:
      - .:/home/circleci/project
    command: sleep infinity
```

docker compose execで接続する想定で、sleep infinity で起動しつづけています。docker-compose.yml をRailsアプリのrootフォルダに置いて、docker compose upコマンドで起動します。-dオプションをつけてバックグラウンドで起動するようにしています。

(ホスト)

```
docker compose up -d
docker compose exec app /bin/bash
```

Docker上でrails sを先ほどと同様に起動します。

(Docker)

```
bundle install
RUBY_DEBUG_PORT=45555 RUBY_DEBUG_HOST=0.0.0.0 bin/rails s -b 0.0.0.0
```

これでコンテナ内のrails serverにホストのChromeデベロッパーツールから接続してデバッグできるようになりました。

## VSCodeでデバッグ

[Visual Studio CodeでRailsアプリを開発](https://zenn.dev/igaiga/books/rails-practice-note/viewer/ruby_rails_vscode) のページで説明しています。

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
- devcontainer for Ruby 3.1 and Rails 7.0
  - https://github.com/saboyutaka/ruby31-rails70-devcontainer
  - さぼさん作のVSCode上でdebug gemほかの環境をつくってRailsアプリを開発するテンプレートです

# 謝辞

本記事を書くにあたり、udzuraさん([@udzura](https://twitter.com/udzura))、笹田さん([@_ko1](https://twitter.com/_ko1))、さぼさん([@saboyutaka](https://twitter.com/saboyutaka))に助けていただきました。ありがとうございます。また、debug gemを熱心に開発してくださっている笹田さん([@_ko1](https://twitter.com/_ko1))、Onoさん([@ono_max7](https://twitter.com/ono_max7))、Stanさん([@_st0012](https://twitter.com/_st0012))に感謝します。
