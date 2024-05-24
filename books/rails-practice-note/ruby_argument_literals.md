---
title: "[Ruby基礎] Rubyの引数でつかわれる記法"
---

# Rubyの引数でつかわれる記法

Rubyの引数に `*`, `**`, `&`, `...` といった記法がつかわれることがあります。それぞれの記法がどのようにつかわれるかを説明していきます。

## 配列受け取りと配列展開渡し

引数を渡すとき、引数を受け取るときに `*` をつけることで配列を展開したり、配列として受け取ることができます。

引数を渡すときに、配列の前に `*` をつけて渡すと、配列の要素分の引数として渡すことができます。

```ruby
def foo(arg1, arg2)
  p arg1 #=> "a"
  p arg2 #=> "b"
end

foo(*["a", "b"])
```

引数を受け取るときに、引数の前に `*` をつけると、複数の引数を配列として受け取ることができます。

```ruby
def foo(*array)
  p array #=> ["a", "b"]
end

foo("a", "b")
```

この書き方で、メソッド呼び出し側の引数の個数を可変にすることも可能です。

```ruby
def foo(*array)
  p array
end

foo("a", "b") #=> ["a", "b"]
foo("a", "b", "c") #=> ["a", "b", "c"]
foo #=> []
```

通常の引数、キーワード引数と一緒につかうこともできます。書く順序によってはエラーになることに気をつけてください。

```ruby
def foo(arg1, *args, kwarg1:)
  p arg1 #=> "a"
  p args #=> ["b", "c"]
  p kwarg1 #=> "value"
end

foo("a", "b", "c", kwarg1: "value")
```

## ハッシュ受け取りとハッシュ展開渡し

キーワード引数を渡すとき、キーワード引数を受け取るときに `**` をつけることでハッシュを展開したり、ハッシュとして受け取ることができます。この記法はRuby2.7で導入されました。

引数を渡すときに、ハッシュの前に `**` をつけて渡すと、ハッシュの各要素をキーワード引数として渡すことができます。

```ruby
def foo(kwarg1:, kwarg2:)
  p kwarg1 #=> "a"
  p kwarg2 #=> "b"
end

foo(**{kwarg1: "a", kwarg2: "b"})
```

引数を受け取るときに、引数の前に `**` をつけると、複数のキーワード引数をハッシュとして受け取ることができます。

```ruby
def foo(**hash)
  p hash #=> {:a=>1, :b=>2}
end

foo(a: 1, b: 2)
```

この書き方で、メソッド呼び出し側の引数の個数を可変にすることも可能です。

```ruby
def foo(**hash)
  p hash
end

foo(a: 1, b: 2) #=> {:a=>1, :b=>2}
foo(a: 1, b: 2, c: 3) #=> {:a=>1, :b=>2, :c=>3}
foo #=> {}
```

通常の引数、キーワード引数と一緒につかうこともできます。書く順序によってはエラーになる点に気をつけてください。

```ruby
def foo(arg1, *array, kwarg1:, **hash)
  p arg1 #=> "a"
  p array #=> ["b", "c"]
  p kwarg1 #=> "value"
  p hash #=> {:kwarg2=>"x"}
end

foo("a", "b", "c", kwarg1: "value", kwarg2: "x")
```

## Proc受け取りとProc展開渡し

メソッドがブロックを受け取るときに、引数に `&` をつけることでProcオブジェクトとしてブロックを受け取ることができます。

```ruby
def foo(&block)
  p block #=> #<Proc:0x...>
  p block.call #=> 1
end

foo do
  1
end
```

メソッドを呼び出すとき、Procオブジェクトの前に `&` をつけて引数を渡すと、ブロックつきでメソッドを呼び出したことになります。

```ruby
def foo
  p yield #=> 1
end

block = Proc.new do
  1
end

foo(&block)
```

また、たとえばこのコード `["abc", "123"].map(&:reverse)` につかわれている `&:method_name` 記法、この&はこのProc展開渡しの意味です。後ろにつづくシンボルにSymbol#to_procメソッドが呼ばれ、結果として  `["abc", "123"].map{|x| x.reverse}` と同じ動作をします。

## 委譲記法

ここまでに出てきた記法をつかうと、任意の引数を受け取るメソッドを書くことができます。

```ruby
def foo(*array, **hash, &block)
  p array #=> ["a", "b"]
  p hash #=> {:c=>"x", :d=>"y"}
  p block #=> #<Proc:0x... >
end

foo("a", "b", c: "x", d: "y") do
  1
end
```

受け取った引数を別のメソッドへ渡すこともできます。

```ruby
def bar(*array, **hash, &block)
  p array #=> ["a", "b"]
  p hash #=> {:c=>"x", :d=>"y"}
  p block #=> #<Proc:0x... >
end

def foo(*array, **hash, &block)
  bar(*array, **hash, &block)
end

foo("a", "b", c: "x", d: "y") do
  1
end
```

先ほどのコードと同じ動作のコードを、委譲記法 `...` をつかって書くこともできます。`...` は全ての引数を受け取り、渡すことができます。この記法はRuby2.7で導入されました。

```ruby
def bar(...)
  p(...)
#=>
# "a"
# "b"
# {:c=>"x", :d=>"y"}

  p yield #=> 1
end

def foo(...)
  bar(...)
end

foo("a", "b", c: "x", d: "y") do
  1
end
```

`...` で受け取った引数を変更することはできません。ただし、引数の受け取り方を変えることで、先頭側の引数を調整することはできます。たとえば次のような書き方です。この記法はRuby2.7.3以降で利用可能です。

```ruby
def bar(...)
  p(...)
#=>
# "b"
# {:c=>"x", :d=>"y"}

  p yield #=> 1
end

def foo(arg1, ...) # 引数の先頭1つ目をarg1に入れ、残りは...で受け取る
  p arg1 #=> "a"
  bar(...) # 引数の先頭1つ目を抜いた残りを渡す
end

foo("a", "b", c: "x", d: "y") do
  1
end
```

委譲記法には、通常の引数のみを委譲する `*`、キーワード引数のみを委譲する `**`、ブロック引数のみを委譲する `&` もあります。前述の例との違いは、変数に名前をつけずに別のメソッドへ委譲することができることです。 `&` はRuby3.1で、`*`と`**`はRuby3.2で導入されました。これらの無名委譲記法は、受け取った引数を変更することはできません。

```ruby
def bar(a, b)
  [a, b]
end

def foo(*)
  bar(*)
end

p foo(1, 2) #=> [1, 2]
```

```ruby
def bar(a:, b:)
  [a, b]
end

def foo(**)
  bar(**)
end

p foo(a: 1, b: 2) #=> [1, 2]
```

```ruby
def bar
  yield
end

def foo(&)
  bar(&)
end

p foo{ 1 } #=> 1
```

## 参考資料

Ruby3.0とRuby2.7でキーワード引数に関する非互換な仕様整理が行われています。書き換え方法は公式ページが参考になります。

[Separation of positional and keyword arguments in Ruby 3.0](https://www.ruby-lang.org/en/news/2019/12/12/separation-of-positional-and-keyword-arguments-in-ruby-3-0/)

また、キーワード引数の仕様変更とは直接関係ないですが、Ruby2.7.1から2.7.2へのバージョンアップでdeprecated カテゴリのWarningはデフォルトで出力されなくなる変更が入っています。Warningを出力するときのオプション指定方法は以下の通りです。詳しくは[「Rubyの知識」](./ruby_idioms)のページの「deprecatedカテゴリのWarningを出力する」に書いてあります。

```
$ ruby -W:deprecated foo.rb
$ RUBYOPT='-W:deprecated' bin/rspec spec/foo_spec.rb
```
