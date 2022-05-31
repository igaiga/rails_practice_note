---
title: "[Ruby基礎] ObjectSpace"
---

# ObjectSpace

Rubyの組込ライブラリであるObjectSpaceをつかうとオブジェクトに関するいろいろな情報を得たり、オブジェクトを探すことができます。ObjectSpaceにある各種メソッドについて説明します。

## ObjectSpace.each_object

- https://docs.ruby-lang.org/ja/latest/class/ObjectSpace.html#M_EACH_OBJECT

引数で渡したクラスのオブジェクトを全て得ることができます。

たとえば全Stringオブジェクトを取得するには引数にStringを渡します。

```ruby
ObjectSpace.each_object(String)
```

ただし、シングルトンなもの、たとえばSymbolではこの方法は使えません。全Symbolオブジェクトを取得する場合は代わりに `Symbol.all_symbols` がつかえます。

## クラス一覧を取得

ObjectSpace.each_objectの引数にClassオブジェクトを渡せば定義されている全クラスがわかります。

```ruby
ObjectSpace.each_object(Class).to_a.map(&:to_s).sort
```

クラス数を数えたところ次のようになりました。

- Ruby3.1.2 irb: 600個
- Ruby3.1.2 Rails7.0.3 rails c: 3200個

## ObjectSpace.count_objects

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

## ObjectSpace.memsize_of

- https://docs.ruby-lang.org/ja/latest/method/ObjectSpace/m/memsize_of.html

オブジェクトごとのメモリ使用量を得ます。単位はバイトです。ただし、戻り値が正しくないこともあるので、ヒントとしてつかう必要があります。このメソッドを実行するには、 `require "objspace"` を実行してObjectSpaceを拡張する必要があります。

```ruby
require 'objspace'

ObjectSpace.memsize_of(10)               # => 0
ObjectSpace.memsize_of("1234567890" * 2) #=> 40
ObjectSpace.memsize_of("1234567890" * 3) #=> 71
```

## ObjectSpace#finalizer

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

## ObjectSpace#trace_object_allocations_start

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

## 参考資料

- I am a puts debuggerer
    - http://tenderlovemaking.com/2016/02/05/i-am-a-puts-debuggerer.html
