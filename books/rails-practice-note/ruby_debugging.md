---
title: "[Ruby基礎] デバッグに便利な道具"
---

# デバッグに便利な道具

## binding.irb

`binding.irb` を書いてプログラムを動作させると、その行が実行されたときにirbが起動し、コード実行を一時停止してデバッグすることができます。任意のRubyの式を実行できます。`exit`を実行すると、コード実行が再開されます。再開ではなく、プログラムを終了させるときは`exit!`を実行します。

binding.irbはgemをインストールせずに実行でき、またrequireを書く必要もありません。どんな環境でもつかえるデバッグの道具としてたいへん便利です。

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



