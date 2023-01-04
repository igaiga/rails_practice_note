---
title: "[Ruby基礎] デバッグに便利な道具"
---

# デバッグに便利な道具

## binding.irb

`binding.irb` を書いてプログラムを動作させると、その行が実行されたときにirbが起動し、コード実行を一時停止してデバッグすることができます。任意のRubyの式を実行できます。`exit`を実行すると、コード実行が再開されます。再開ではなく、プログラムを終了させるときは`exit!`を実行します。

binding.irbはgemをインストールせずに実行でき、またrequireを書く必要もありません。どんな環境でもつかえるデバッグの道具としてたいへん便利です。

また、ステップ実行や変数情報の取得などより多くの機能を持ったdebug gemもRubyの標準添付ライブラリとして提供されています。debug gemについては[別のページ](../viewer/ruby_debug_gem)で説明しているのでそちらも参照してください。

## methodメソッドとsource_locationメソッド

メソッドが定義されているソースコードのパスを調べるときには、methodメソッドとsource_locationメソッドを組み合わせてつかいます。

最初にmethodメソッドにシンボルでメソッド名を渡して、Methodオブジェクトを取得します。

```ruby
require "csv"
CSV.method(:read) #=> Methodオブジェクト
```

Methodオブジェクトで呼び出せるsource_locationメソッドをつかうと、そのメソッドが定義されたソースコードのパスと行数を表示できます。

```ruby
require "csv"
CSV.method(:read).source_location #=> ["/Users/igaiga/.rbenv/versions/3.1.2/lib/ruby/3.1.0/csv.rb", 1668] 
```

Ruby2.7からはMethodオブジェクトのinspectメソッドが同様の情報を返すので、source_locationメソッドの代わりにpメソッドでも表示できます。

```ruby
require "csv"
p CSV.method(:read) #=> #<Method: CSV.read(path, **options) /Users/igaiga/.rbenv/versions/3.1.2/lib/ruby/3.1.0/csv.rb:1668>
```

source_locationメソッドでnilが返ってきたときは、Rubyで定義されていないメソッドです。たとえば組み込みライブラリのクラスではC言語で定義されているメソッドもあるので、そのケースではnilを返します。

ソースコードのパスがわかれば、Gemで定義されていても、自分たちのアプリで定義されていても、ソースコードを読むことができるので調査や動作の理解を深めるのに役立ちます。

Methodオブジェクトには他にもownerメソッド(メソッドが定義されているクラスまたはモジュールを返す)、original_nameメソッド(aliasがつかわれているときにalias先のメソッド名を返す)、super_methodメソッド(superを呼んだときに呼び出されるメソッドオブジェクトを返す)など便利なメソッドが用意されています。また、メソッドの継承ツリー呼び出し順を調べるときはModule#ancestorsメソッドが便利です。

- Rubyリファレンスマニュアル Methodクラス: https://docs.ruby-lang.org/ja/latest/class/Method.html

## privateメソッドを呼び出す

デバッグのときなど、privateなメソッドをNoMethodErrorにならないように呼び出ししたいときはeval族のメソッドをつかうのが便利です。

privateなインスタンスメソッドを呼び出すときは、instance_evalメソッドをつかいます。

```ruby
class Foo
  private
  def bar
    "private bar method"
  end
end

foo = Foo.new
# foo.bar #=> private method `bar' called for #<Foo ...> (NoMethodError)
p foo.instance_eval{ bar } #=> "private bar method"
```

privateなクラスメソッドを呼び出すときは、class_evalメソッドをつかいます。

```ruby
class Foo
  private_class_method def self.bar
    "private bar class method"
  end
end

#Foo.bar #=> private method `bar' called for Foo:Class (NoMethodError)
p Foo.class_eval{ bar } #=> "private bar class method"
```

また、sendメソッドをつかう方法もありますが、引数を加えた呼び出しの形が通常時と少し異なるのでeval族をつかう方が覚えやすいかもしれません。

```ruby
class Foo
  private
  def bar(x,y)
  end
end

foo = Foo.new
foo.send(:bar, 1, 2)
```

## インスタンス変数を取得する

オブジェクトがどんなインスタンス変数を持っているかをinstance_variablesメソッドとinstance_variable_getメソッドをつかって調べることができます。

```ruby
class Foo
  def initialize
    @x = 1
    @y = 2
  end
end

foo = Foo.new
p foo.instance_variables #=> [:@x, :@y]
p foo.instance_variable_get(:@x) #=> 1
```

インスタンス変数を変更したいときはinstance_variable_setメソッドをつかいます。

```ruby
class Foo
  def initialize
    @x = 1
  end
end

foo = Foo.new
p foo.instance_variable_get(:@x) #=> 1
foo.instance_variable_set(:@x, 3)
p foo.instance_variable_get(:@x) #=> 3
```

## ObjectSpace

Rubyの組込ライブラリであるObjectSpaceをつかうとオブジェクトに関するいろいろな情報を得たり、オブジェクトを探すことができます。ObjectSpaceにある各種メソッドについて説明します。

### ObjectSpace.each_object

- https://docs.ruby-lang.org/ja/latest/class/ObjectSpace.html#M_EACH_OBJECT

引数で渡したクラスのオブジェクトを全て得ることができます。

たとえば全Stringオブジェクトを取得するには引数にStringを渡します。

```ruby
ObjectSpace.each_object(String)
```

ただし、たとえば次のクラスではこの方法はつかえません。

- Symbol
- Integer
- Float
- TrueClass
- FalseClass
- NilClass

全Symbolオブジェクトを取得する場合は代わりに `Symbol.all_symbols` がつかえます。

### クラス一覧を取得

ObjectSpace.each_objectの引数にClassオブジェクトを渡せば定義されている全クラスがわかります。

```ruby
ObjectSpace.each_object(Class).to_a.map(&:to_s).sort
```

クラス数を数えたところ次のようになりました。

- Ruby3.1.2 irb: 600個
- Ruby3.1.2 Rails7.0.3 rails c: 3200個

### ObjectSpace.count_objects

- https://docs.ruby-lang.org/ja/latest/class/ObjectSpace.html#M_COUNT_OBJECTS

全オブジェクトの種類と数を取得します。

```shell
ObjectSpace.count_objects
#=>
{:TOTAL=>59654,
 :FREE=>25,
 :T_OBJECT=>1160,
 :T_CLASS=>1075,
 :T_MODULE=>78,
 :T_FLOAT=>4,
 :T_STRING=>28897,
 :T_REGEXP=>326,
 :T_ARRAY=>6075,
 :T_HASH=>2524,
 :T_STRUCT=>67,
 :T_BIGNUM=>2,
 :T_FILE=>12,
 :T_DATA=>691,
 :T_MATCH=>970,
 :T_COMPLEX=>1,
 :T_SYMBOL=>77,
 :T_IMEMO=>17567,
 :T_ICLASS=>103}
```

### ObjectSpace.memsize_of

- https://docs.ruby-lang.org/ja/latest/method/ObjectSpace/m/memsize_of.html

オブジェクトごとのメモリ使用量を得ます。単位はバイトです。ただし、戻り値が正しくないこともあるので、ヒントとしてつかう必要があります。このメソッドを実行するには、 `require "objspace"` を実行してObjectSpaceを拡張する必要があります。

```ruby
require 'objspace'

ObjectSpace.memsize_of(10)               # => 0
ObjectSpace.memsize_of("1234567890" * 2) #=> 40
ObjectSpace.memsize_of("1234567890" * 3) #=> 71
```

### ObjectSpace#finalizer

- https://docs.ruby-lang.org/ja/latest/method/ObjectSpace/m/define_finalizer.html

オブジェクト解放時に実行される処理を登録します。Procオブジェクトやブロックをつかって解放時の処理を登録します。

```ruby
class Foo
end

Foo.new

finalize_proc = Proc.new { puts "finalize!" }
ObjectSpace.define_finalizer(Foo, finalize_proc)

# ObjectSpace.define_finalizer(Foo){ puts "finalize!" }
# Procの代わりにブロックを渡すこともできます
```

### ObjectSpace#trace_object_allocations_start

- https://docs.ruby-lang.org/ja/latest/method/ObjectSpace/m/trace_object_allocations.html
- https://docs.ruby-lang.org/ja/latest/library/objspace.html

オブジェクトが生成されたsource_locationを取得します。このメソッドを実行するには、 `require "objspace"` を実行してObjectSpaceを拡張する必要があります。

- example.rb
```ruby
require "objspace"
ObjectSpace.trace_object_allocations_start

def foo
  x = baz
  bar x
end

def bar(x)
  p ObjectSpace.allocation_sourcefile(x) => ObjectSpace.allocation_sourceline(x)
end

def baz
  Object.new # 14行目
end

foo
```
```shell
{"example.rb"=>14}
```

ObjectSpace.trace_object_allocations_startを実行したあとで検出が始まりますが、それでは間に合わないこともあります。そのときは次のように実行前に挿入することもできます。

- require_objectspace.rb

```ruby
require "objspace"
ObjectSpace.trace_object_allocations_start
```

```shell
$ RUBYOPT="-I. -rrequire_objectspace.rb" rake some_tasks
```

-Iオプションはファイルをロードするパスを追加するオプションです。`-I.`でカレントディレクトリを追加します。-rオプションは実行前にrequireを行うオプションです。RUBYOPT環境変数をつかうと、rubyコマンドを直接実行しないコマンドへオプションを指定することができます。

## TracePoint

Rubyの組込ライブラリであるTracePointをつかうとさまざまなイベントをフックしてあらかじめ指定したコードを実行できます。フック可能なイベントはメソッド呼び出し、例外発生、式実行などいくつか用意されています。`TracePoint.trace(イベント種別指定) do end` と書くと、以降の実行コードからイベントの監視を始めます。

TracePointで監視を始めると実行速度はかなり遅くなります。速度低下を減らすためには、たとえば `TracePoint#enable(target_line:)` をつかって指定した行のみで TracePoint を有効にすることで速度低下範囲を狭めることができます。debug gemの[traceコマンド](https://github.com/ruby/debug#trace)では監視機能を便利につかえるように提供しているので、debug gemをつかってかんたんに速度低下を少なく監視できるか調べてみるのも良い方法です。

### 監視できるイベント

たとえば次のようなイベントを監視できます。

- :line 式の評価
- :call Rubyで書かれたメソッドの実行
- :c_call Cで書かれたメソッドの実行
  - Rubyが提供するライブラリやGemの中にはCで書かれたメソッドもあります
- :b_call ブロック実行
- :raise 例外発生

- 全てのイベントはこのページで見れます
    - https://docs.ruby-lang.org/ja/latest/method/TracePoint/s/new.html

### 式実行時にブロック実行

次のコードは、式実行時に指定したブロックを実行するTracePointをつかったサンプルコードです。


```ruby
class Foo
  def bar
    "bar method" # 3行目
  end
end

TracePoint.trace(:line) do |tp| # 引数に:lineを渡すと式実行時にブロック実行
  # ブロック引数にはTracePontオブジェクトが渡される
  puts "[TP:#{tp.event}] #{tp.path}:#{tp.lineno}"
end
Foo.new.bar # 10行目

#=> [TP:line] example.rb:10
#=> [TP:line] example.rb:3
```

`TracePoint.trace`メソッドに引数で監視するイベントを渡します。ここでは `:line` を渡して式実行を監視するので、全ての実行行で渡したブロックが実行されます。

ブロック引数にはTracePointオブジェクトが渡されます。TracePointオブジェクトからは、たとえば次のような情報が取れます。取得できる情報は、イベントごとに異なります。

```ruby
TracePoint.trace(:line) do |tp|
  p tp.event # イベント名
  p tp.path # 実行されたソースコードのファイルパス 
  p tp.lineno # 実行された行番号
  p tp.method_id # 実行されたメソッド名
  p tp.defined_class # 実行されたメソッドが定義されたクラス
end
```

また、`tp.binding` で実行された場所でのBindingオブジェクトが取得できるので、たとえば `tp.binding.local_variables` で実行された行のスコープでのローカル変数一覧を取得できます。

### Railsアプリへ組み込み

RailsアプリでTracePointをつかうときは、たとえばconfig/initializers以下に次のコードを配置すればOKです。

```ruby
TracePoint.trace(:raise) do |tp|
  puts "[Event:#{tp.event}] #{tp.path}:#{tp.lineno} #{tp.method_id} #{tp.defined_class}"
end
```

rails sなどを起動してアクセスし、標準出力を観察してみてください。traceメソッドに`:raise`を渡しているので、例外発生時にログが出力されます。たとえば`:line`を渡すと、たくさん表示されて実行速度がすごく遅くなるので注意です。


### 全てのイベントを観察

traceメソッドにイベントを渡さないと、全てのイベントを監視します。ブロックが呼ばれる回数は増えるので、実行速度はかなり遅くなります。

```ruby
trace = TracePoint.trace do |tp|
  puts "[Event:#{tp.event}] #{tp.path}:#{tp.lineno} #{tp.method_id} #{tp.defined_class}"
end

class Foo
  def bar
    "bar method"
  end
end

Foo.new.bar
```

```
[Event:line] example.rb:5
[Event:c_call] example.rb:5 inherited Class
[Event:c_return] example.rb:5 inherited Class
[Event:class] example.rb:5
[Event:line] example.rb:6
[Event:c_call] example.rb:6 method_added Module
[Event:c_return] example.rb:6 method_added Module
[Event:end] example.rb:9
[Event:line] example.rb:11
[Event:c_call] example.rb:11 new Class
[Event:c_call] example.rb:11 initialize BasicObject
[Event:c_return] example.rb:11 initialize BasicObject
[Event:c_return] example.rb:11 new Class
[Event:call] example.rb:6 bar Foo
[Event:line] example.rb:7 bar Foo
[Event:return] example.rb:8 bar Foo
```

## 参考資料

- I am a puts debuggerer
  - https://tenderlovemaking.com/2016/02/05/i-am-a-puts-debuggerer.html
  - このページで紹介したObjectSpaceの便利なつかい方のほか、Rubyのさまざまなデバッグ方法を紹介しているページです
