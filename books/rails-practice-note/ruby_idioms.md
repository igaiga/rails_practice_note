---
title: "[Ruby基礎] Rubyの知識"
---

# Rubyの知識

## !!

`!!foo` はオブジェクトを次のように変換する書き方です。

- nilまたはfalseのとき: false
- それ以外: true

たとえば次のようなコードでつかわれます。

```ruby
def something?
  foo = xxx # 何か代入
  !!foo
end
```

突然出てくると「これは何者だ？」となりますが、真偽反転の`!`が2回つかわれていると解釈すれば意味を取りやすいです。ただ、真偽反転を2回繰り返しても条件判定の結果には影響しないのに、なぜ `!!` をつかうのでしょうか？

`!!` は必ずtrueまたはfalseに変換するので、任意のオブジェクトやnilをメソッドから返したくないときによくつかわれます。たとえば、Rubyでは末尾が?で終わるメソッドはtrue/falseを返す慣習になっているので、オブジェクトをtrue/falseへ変換するときに!!が便利につかえることが多いです。

一方で、メソッド内でifで判定するだけなど、メソッドの戻り値としてつかわないときは `!!` をつけずにそのまま条件判定につかうことが多いです。

## ||=

`||=` は `変数 ||= 初期値` という形でよくつかわれます。次のサンプルコードで説明します。

```ruby
foo = 555
foo ||= 1
p foo #=> 555

foo = nil
foo ||= 1
p foo #=> 1
```

変数fooに何か(nilとfalseを除く)が代入されていたら、`foo ||= 1` は何もしません。変数fooにnilまたはfalseが代入されていたり、または何も代入されていないときは`foo ||= 1`は1を代入します。ざっくりと「変数に何か入っていたらそのまま、何も入っていなかったら`||=`の右に書かれたオブジェクトを初期値としてつかう」と考えると覚えやすいかもしれません。

解釈としては、`foo ||= 1` は `foo = foo || 1` と同じです。`||` は最初の項がnilまたはfalseのときだけ2項目を戻り値にして、それ以外のときは最初の項を戻り値にするので、前述の動作になります。似た書き方でよくつかわれるのは`foo += 1`で、これは`foo = foo + 1`と同じです。

ただし、true/falseを初期値につかうときは気をつけてください。事前にfalseが代入されているときに意図しない動作になっているかもしれません。

```ruby
foo = false
foo ||= true
p foo #=> true # もともと代入されていたfalseをつかいたかった
```

## foo(&:bar)

コード例で説明します。

```ruby
["a","b","c"].map(&:upcase)
#=> ["A", "B", "C"]
```

これは次のコードと同じ結果になります。

```ruby
["a","b","c"].map{|x| x.upcase}
#=> ["A", "B", "C"]
```

メソッドの引数に `&:method` と書くと、`{|x| x.method}`というブロックを渡すことになります。Arrayオブジェクトのmapメソッドやeachメソッドでよくつかわれる書き方です。

覚えて機械的に置き換えるのが楽ですが、もしもなぜこうなるかのカラクリを知りたい方は以降の段落を読んでください。次の節まで読み飛ばしても問題ないです。

`map(&:upcase)` の `&` は引数に渡されたProcオブジェクトを展開してブロックとして渡す記法です。詳しくはこちらのページに書いています。

[Rubyの引数でつかわれる記法](https://zenn.dev/igaiga/books/rails-practice-note/viewer/ruby_argument_literals)

&の後ろにシンボルが書かれているので、RubyはこれをSymbol#to_procメソッドを呼んでProcオブジェクトへと変換を試みます。`:upcase.to_proc` は `Proc.new{|x| x.upcase}` 相当のコードになります。

コード全体は`["a","b","c"].map(&Proc.new{|x| x.upcase})` となるので、引数の前についた&によってProcオブジェクトを展開してブロックとして渡し、 `["a","b","c"].map{|x| x.upcase}` 相当のコードになります。

## case whenで多様なN者択一のコードを書く

case when構文はN個の選択肢の中から1つを選んで実行する、N者択一を表現するコードです。よく知られているのは次のようなコードです。

```ruby
x = 2
case x
when 1
  puts 1
when 2
  puts 2
else
  puts "others"
end
#=> 2
```

caseの後ろに変数を置いて、代入されたオブジェクトをwhenでマッチさせて分岐します。

他にも、caseの後ろに変数を置かずにwhenで条件判定する書き方があります。

```ruby
x = 2
y = 2
case
when x == 1
  puts 1
when y == 2
  puts 2
else
  puts "others"
end
#=> 2
```

N個の分岐の中から最初にマッチしたものへ進むので、この書き方もN者択ーを表現しています。

これをつかえば、if elsif 構文をcase when構文で書き換えることができます。次のコードは同じ結果になりますが、N者択一を表現したいのであれば、case when構文をつかった方が直感的です。

```ruby
x = 2
y = 2
case
when x == 1
  puts 1
when y == 2
  puts 2
else
  puts "others"
end
#=> 2

# if x == 1
#   puts 1
# elsif y == 2
#   puts 2
# else
#   puts "others"
# end
```

case when構文は他にもさまざまな便利な書き方ができます。whenでの条件判定は===で行われます。

```ruby
x = 100
case x
when 1..20
  puts "1..20"
when (21..)
  puts "21.."
else
  puts "others"
end
#=> 21..
```

```ruby
x = "foo"
case x
when String
  puts "String"
when Integer
  puts "Integer"
end
#=> String
```

また、パターンマッチと呼ばれるcase in構文もあります。詳しくは後述します。

## 小数を正確に扱う

たとえば次のような小数を計算するコードは、結果が不正確になります。

```ruby
p 0.1+0.1+0.1 #=> 0.30000000000000004
```

これは、`0.1`がFloatオブジェクトとなり、Floatオブジェクトは正確な値を保持できないことがあることが原因です。

Floatオブジェクトの代わりに、RationalオブジェクトやBigDecimalオブジェクトをつかうことで正確に小数を扱うことができます。

数字の末尾にrをつけて書くとRationalオブジェクトをつくることができます。

BigDecimalオブジェクトをつかうときは `require "bigdecimal"` が必要です。

```ruby
p 0.1r+0.1r+0.1r #=> 3/10
p (0.1r+0.1r+0.1r).to_f #=> 0.3

require "bigdecimal"
p BigDecimal("0.1")+BigDecimal("0.1")+BigDecimal("0.1") #=>0.3e0
```

## &.

`&.` はレシーバのnilチェックをしてメソッド呼び出しをする演算子です。レシーバがnilのときはメソッド呼び出しをせずnilを返し、nil以外のレシーバではメソッド呼び出しをします。

```ruby
x = "abc"
p x&.upcase #=> "ABC"

x = nil
p x&.upcase #=> nil
```

`&.` は通称「ぼっち演算子」とも呼ばれます。人が1人でひざを抱えて座っているように見えるからです。

また、Railsに用意されている `try!` , `try` メソッドは `&.` とほぼ同等な処理をします。特に理由がなければ  `&.` をつかう方が自然です。正確には、`&.` は `try!` と同等で、レシーバがnil以外のとき、呼び出すメソッドが実装されていないときの動作が次のように違います。

```ruby
123&.upcase #=> NoMethodError例外
123.try!(:upcase) #=> NoMethodError例外
123.try(:upcase) #=> nil
```

## ブロックとローカル変数

ブロックの内外にあるローカル変数の仕様についてまとめます。

ブロック中で代入したローカル変数は、ブロックの中でだけつかうことができます。

```ruby
1.times do
  x = 1
  p x #=> 1
end
p x #=> NameError: undefined local variable or method 'x'
```

ブロックの前で代入したローカル変数は、ブロックの中でもつかうことができます。ブロックが終わった後もつかうことができます。ブロックの中でそのローカル変数を変更した場合、その変更はブロックを抜けた後も残ります。

```ruby
x = 1
1.times do
  p x #=> 1
  x = 2
  p x #=> 2
end
p x #=> 2
```

この仕様をつかうと、ブロックの中でつくられた結果を、ブロックが終わった後につかうことができます。

```ruby
result = []
[1,2,3].each do |i|
  result << i * 2
end
p result #=> [2, 4, 6]
```

## selfを省略できないケース

selfを省略できるとき、ほとんどのケースで省略することが一般的です。

```ruby
class Foo
  def foo
    "foo!"
  end
  def foo2
    # fooメソッドを呼び出すとき、 self.foo と書くよりも、selfを省略して foo と書く方が一般的
    foo
  end
end
puts Foo.new.foo2 #=> "foo!"
```

fooメソッドを呼び出すとき、 `self.foo` と書くよりも、selfを省略して `foo` と書く方が一般的です。

selfを省略できない注意が必要なケースは、ローカル変数への代入と見分けがつかなくなるときです。次のようなコードです。

```ruby
class Foo
  def foo=(arg)
    @foo = arg
  end
  def set_foo
    foo = 1 # ローカル変数fooへ代入
    self.foo = 2 # foo=メソッドの呼び出し
  end
end
```

`foo = 1` と書くとfoo=メソッドの呼び出しではなく、ローカル変数fooへの代入になります。foo=メソッドを呼び出したいときは`self.foo = 2` と書きます。このサンプルコードのfoo=メソッドのように自分で定義しているときは気づきやすいですが、Railsのモデルクラスや、Rubyのattr_writerメソッドのように直接書かれていないこともあるので注意です。

次のサンプルコードはRailsのモデルクラスでの例です。

```ruby
class User < ApplicationRecord
  # usersテーブルにカラムnameがあるとき
  def set_name
    name = "igaiga" # ローカル変数nameへ代入
    self.name = "igaiga" # name=メソッドの呼び出し(usersテーブルのカラムnameへ代入)
  end
end
```

## super

superはメソッド探索順で次の順序である同名メソッドを呼び出します。

```ruby
class A
  def foo(a, b)
    p "#{a}, #{b}"
  end
end

class B < A
  def foo(x, y)
    super # A#fooメソッドを引数x,yを渡して呼び出し
  end
end

B.new.foo(1,2) #=> "1, 2"
```

superが呼ばれると、メソッド探索順で次の順序である同名メソッド(以降、探索次順メソッド)が呼び出されます。つまり、Bクラスのfooメソッド中でsuperが呼ばれると、探索次順メソッドA#fooメソッドを呼び出します。

`super` と書くと、現在のメソッドへ渡ってきている引数を、自動的に探索次順メソッドへ渡します。`super(x)` のように引数を明示することもできます。引数を渡したくないときは `super()` と書きます。

また、superの戻り値は探索次順メソッドが返したオブジェクトとなります。

必ず親クラスのメソッドが探索次順メソッドとして選ばれるわけではなく、たとえば、includeされたモジュールに同名メソッドがあれば、それが探索次順メソッドになります。

その場所でsuperを呼んだときに、どのクラスのメソッドが呼ばれるかを調べるにはMethod#super_methodメソッドをつかって `method(__method__).super_method` と書けます。 `__method__` はメソッド名をシンボルで取得するメソッドで、 `method(__method__)` でそのメソッド自身のMethodオブジェクトを取得できます。

```ruby
class A
  def foo
  end
end

class B < A
  def foo
    p method(__method__).super_method
  end
end

B.new.foo #=> #<Method: A#foo() example.rb:2>
```

また、メソッド探索順の調べ方はたとえば `B.ancestors` のように、Module#ancestorsメソッドで調べることができます。

```ruby
class A
  def foo
  end
end

class B < A
  def foo
  end
end

p B.ancestors #=> [B, A, Object, Kernel, BasicObject]
```

superがよくつかわれるケースはinitializeメソッドです。継承したときにinitializeメソッド中にsuperを書かないと、親クラス群のinitializeメソッドが呼ばれないので、初期化が不十分になることがあります。initializeメソッドを実装するときは、原則としてsuperを呼び出すのが良いでしょう。（initializeメソッドを実装しないケースでは、探索次順のinitializeメソッドが呼ばれるので問題ないです。）

```ruby
class A
  def initialize(x, y)
    @x = x
    @y = y
  end
end

class B < A
  def initialize(x,y)
    # ... 自分のクラスの初期化処理(親クラスより前に実行)
    super
    # ... 自分のクラスの初期化処理(親クラスより後に実行)
  end
  def x
    @x
  end
end

p B.new(1,2).x #=> 1
```

## Hashで値が省略されたとき

Hashオブジェクトで値が省略されたときは、キー名のローカル変数またはメソッドを値としてつかいます。

```ruby
a = 1
b = 2
{a:, b:} #=> {a: 1, b: 2}
```

`{a:, b:}` は `{a: a, b: b}` と同じになります。

また、キーワード引数でも同様の書き方ができます。

```ruby
def foo(a:, b:)
  p a #=> 1
  p b #=> 2
end
a = 1
b = 2
foo(a:, b:)
# foo(a: a, b: b) と同じ
```

これらの文法はRuby3.1から導入されました。

## Hashのデフォルト値

Hashオブジェクトの「キーに対応する値が存在しない時のデフォルト値」を設定するための機能として、Hash.newメソッドとHash#default=メソッドへそれぞれ引数を渡す方法があります。デフォルト値として毎回同一のオブジェクトを返すため、Integerオブジェクトのような破壊的操作を行わないオブジェクトでは問題ないのですが、デフォルト値のオブジェクトに対して破壊的操作を行うメソッドをつかったときに意図しない動作になることがあります。

次のコードはデフォルト値としてIntegerオブジェクトをつかう例です。このコードは意図通り動いているかと思います。

```ruby
hash = {}
hash.default = 0 # hash = Hash.new(0) も同様の結果
p hash[:a] #=> 0
hash[:b] = hash[:b] + 10 # hash[:b]のデフォルト値は0
p hash[:b] #=> 10
hash[:c] = hash[:c] + 10 # hash[:c]のデフォルト値は0
p hash[:c] #=> 10
```

次のコードはデフォルト値として空の配列オブジェクトをつかう例です。「キーごとに新しい空配列をデフォルト値としてつかいたい」ケースでは、意図通りに動きません。

```ruby
hash = {}
hash.default = [] # hash = Hash.new([]) も同様の結果
p hash[:a] #=> []
hash[:b] << "b" # hash[:b]のデフォルト値は[]、<<で破壊的に変更される
p hash[:b] #=> ["b"]
hash[:c] << "c" # hash[:c]のデフォルト値は [] << "b" で破壊的変更された["b"]
p hash[:c] #=> ["b", "c"] # ["c"] にはならない
```

キーに対応する値が存在しない時のデフォルト値として、キーごとに新しいオブジェクトをつくりたいときはHash.newメソッドへブロックを渡すか、Hash#default_proc=メソッドへProcオブジェクトを渡します。デフォルト値がつかわれるときに、渡したブロックやProcオブジェクトが実行されてデフォルト値になります。

```ruby
hash = {}
hash.default_proc = Proc.new{|h, key| h[key] = [] }
# または次のコードでも同じ
# hash = Hash.new{|h, key| h[key] = [] }

p hash[:a] #=> []
hash[:b] << "b"
p hash[:b] #=> ["b"]
hash[:c] << "c"
p hash[:c] #=> ["c"]
```

## Numbered parameters

Numbered parametersは、ブロック引数(変数)を省略して書ける記法です。

```ruby
result = ["a", "b"].map do
  _1.upcase
end
p result #=> ["A", "B"]
```

このコードは次のコードと同様です。

```ruby
result = ["a", "b"].map do |x|
  x.upcase
end
p result #=> ["A", "B"]
```

ブロック引数を書かずに、 `_1` をその替わりとしてつかうことができます。

複数のブロック引数を取るメソッドでは、`_1`, `_2`, `_3` ... がつかえます。

```ruby
result = {"a" => "A", "b" => "B"}.map do
  { _1.upcase => _2.downcase }
end
p result #=> [{"A"=>"a"}, {"B"=>"b"}]
```

このコードは次のコードと同様です。

```ruby
result = {"a" => "A", "b" => "B"}.map do |key, value|
  { key.upcase => value.downcase }
end
p result #=> [{"A"=>"a"}, {"B"=>"b"}]
```

Numbered parametersは、Ruby2.7.0で導入された比較的新しい記法です。

また、Ruby3.4からは `_1` のようにつかえる `it` が導入されました。Numbered parametersを1つしかつかわないときに読みやすく書くことができます。

```ruby
result = ["a", "b"].map do
  it.upcase
end
p result #=> ["A", "B"]
```

## Frozen String Literal

コードの1行目に `# frozen_string_literal: true` と書くと、そのファイル中に書かれたStringリテラル全てをイミュータブル(変更不可)にします。各Stringオブジェクトにfreezeメソッドを呼んだことと同等になります。

```ruby
# frozen_string_literal: true

str = "abc"
str.frozen? #=> true
str.upcase! #=> can't modify frozen String: "abc" (FrozenError)
```

このファイル中で変更可能なStringオブジェクトを書きたいときは、たとえば `String.new` をつかいます。

```ruby
# frozen_string_literal: true

str = String.new("abc")
str.frozen? #=> false
str.upcase!
p str #=> "ABC"
```

`String.new` のほかには、単項+をつかって `+"abc"`と書く方法や、Object#dupメソッドがfrozenではないオブジェクトを複製して返すことをつかって `"abc".dup` と書く方法もあります。単項+でfreezeされていない（変更可能な）オブジェクト、単項-でfreezeされた（変更不可能な）オブジェクトがつくれることを覚える方法としては、「気温がマイナスならばfreeze、プラスならばfreezeではない」という[方法](https://x.com/yukihiro_matz/status/1734502349778067864)があります。

frozenなオブジェクトにしておくと、実行時の高速化で有利になったり、Ractorの共有可能オブジェクトとしてつかえるなどのメリットを得ることができるため、可能ならば最初からfrozen_string_literal行を書いて置くのが良いでしょう。既存のファイルにfrozen_string_literal行を足すときは、ソースコードに書かれた各Stringオブジェクトに対して変更を加えていないことを確認する必要があります。

また、このようにファイル先頭に書いて動作を指示する特別なコメント行のことをマジックコメントと呼びます。他には `# coding: utf-8` とファイルの文字コードエンコーディングを指定するマジックコメントもあります。ただし、codingマジックコメントのデフォルトはUTF-8なので、UTF-8のファイルをつかっているときには省略可能です。

## JSON.parseメソッド symbolize_namesオプション

JSON形式のデータをHashへ変換するJSON.parseメソッドがあります。このメソッドで変換すると結果のHashのキーは文字列になりますが、symbolize_namesオプションをtrueにして変換するとHashのキーをシンボルにすることができます。

```ruby
require "json"

json = {a: 1, b: 2}.to_json
#=> "{\"a\":1,\"b\":2}"

JSON.parse(json)
#=> {"a"=>1, "b"=>2}

JSON.parse(json, symbolize_names: true)
#=> {:a=>1, :b=>2}
```

https://docs.ruby-lang.org/ja/latest/method/JSON/m/parse.html

## パターンマッチ

パターンマッチはRuby2.7でexperimentalな機能として導入され、Ruby3.0で正式な機能になりました。1行パターンマッチである `=>` はRuby3.0でexperimentalな機能として導入され、Ruby3.1で正式な機能になりました。

パターンマッチ構文には、case/in式、in式、`=>`があります。

ここに書いた内容はパターンマッチのごく一部の機能で、ほかにも多くの機能を持っています。[「n月刊ラムダノート Vol.1, No.3(2019)」](https://www.lambdanote.com/products/nmonthly-vol-1-no-3-2019) には実装者である辻本さんによる詳しい説明が書かれていますので、知識を深めたいときにお勧めします。

### パターンマッチcase/in式

case/in式は次のようにつかいます。

```ruby
case [1,2,3]
in [x,2,3]
end
p x #=> 1
```

caseにつづけて書いた式（先ほどの例では `[1,2,3]` ）を実行して、実行結果とマッチするものをin節に書いたパターン群から探します。in節には変数入りのパターンを書くことができ、caseにつづけて書いた式の結果とパターンとを対応させ、それぞれの変数に代入します。

便利なケースは、深いハッシュや配列にマッチさせるときや、複数の変数に代入するときです。

```ruby
case { a: [1,2,3], b: { c: 10 }}
in { a: [x,2,3], b: { c: y }}
end
p x #=> 1
p y #=> 10
```

in節が複数あるときは、最初にマッチしたものが実行されます。

```ruby
case [1,2,3]
in [x,2,3]
in [x,y,3]
end
p x #=> 1
p y #=> nil
```

in節に該当するパターンがないときは、NoMatchingPatternError例外が発生します。ちなみに、似た文法であるcase when構文では該当する候補がなくても例外は発生しません。

```ruby
case [1,2,3]
in [1,2,3,4]
end
#=> NoMatchingPatternError
```

else節を書けばどのin節もマッチしないときに選ばれます。else節を書くとかならずどれかにマッチするので、例外は発生しなくなります。

```ruby
case [1,2,3]
in [1,2,3,4]
else
end
```

in節に式を書くと、マッチしたときに実行されます。

```ruby
case [1,2,3]
in [x,2,3]
  puts "[x,2,3]"
in [x,y,3]
  puts "[x,y,3]"
end #=> "[x,2,3]"
p x #=> 1
```

case/in式の戻り値は、マッチしたin節の式の実行結果になります。in節の式が書かれていないときはnilになります。

```ruby
result = case [1,2,3]
  in [x,2,3]
  end
p result #=> nil
```

```ruby
result = case [1,2,3]
  in [x,2,3]
    :foo
  end
p result #=> :foo
```

in節に既存の変数が指すオブジェクトをつかいたいときには、ピン演算子 `^` をつけて変数を書きます。ピン演算子をつけないと、書いた変数に代入するので違う意味になります。

```ruby
a = [1,2,3]
result = case [1,2,3]
  in ^a # in [1,2,3] と同じになる
    :foo
  end
p result #=> :foo
```

### 1行パターンマッチ in式, `=>`

（caseなしの）in式をつかうと、1行パターンマッチが書けます。

```ruby
[1,2,3] in [x,y,3]
p x #=> 1
p y #=> 2
```

in式ではマッチしなくても例外は発生しません。

```ruby
[1,2,3] in [x,y,5]
p x #=> nil
p y #=> nil
```

`=>` も1行パターンマッチの文法です。

```ruby
[1,2,3] => [a,b,3]
p a #=> 1
p b #=> 2
```

`=>` はcase/in式と同じく、マッチしないときにNoMatchingPatternError例外を発生させます。

```ruby
[1,2,3] => [x,2,3,4]
#=> NoMatchingPatternError
```

in式や`=>`は右側に書いた変数へ代入する、右代入としてもつかえます。

```ruby
1 in x
p x #=> 1
```

```ruby
1 => x
p x #=> 1
```

## RubyのライブラリとDefault gems、Bundled gems

Rubyのライブラリには Ruby をインストールした時点から使う事ができる組み込みライブラリと標準添付ライブラリ、そして主にユーザーがRubyのインストールとは別にインストールするGemの3種類があります。

組み込みライブラリはRuby本体に内蔵されていて、requireを書かなくてもつかうことができるクラス群です。例としてはArray、Hash、Stringなどがあります。標準添付ライブラリはRubyと一緒にインストールされていて、requireを書いてつかうクラス群です。例としてはJSON、YAML、OpenSSLなどがあります。

GemはRubyGems(gemコマンド)やBunlder(bundleコマンド)でインストールするライブラリで、BundlerでつかうときにはGemfileに追記して、必要に応じてrequireしてつかいます。[RubyGems.org](https://rubygems.org/)で利用できるGemの一覧を見ることができます。例としてはRedis gemやRack gemなどがあり、Railsも複数のGemの集合体です。

ここまで説明してきたように、組み込みライブラリと標準添付ライブラリはRubyと一緒にインストールされます。これらのライブラリにたとえば脆弱性を修正するようなリリースがあったときに、Rubyのバージョンを上げるよりも、そのライブラリだけをバージョンアップできた方が便利です。

組み込みライブラリと標準添付ライブラリの多くはDefault gemsという仕組みでも提供されていて、RubyGemsの仕組みをつかってライブラリ単体でバージョンアップすることができます。RubyGemsのgemコマンドや、Gemfileに追記してBunlderのbundleコマンドをつかうことで、Default gemsで提供されているライブラリは新しいバージョンをインストールして利用可能になります。Default gemsはRubyと一緒にインストールされるライブラリをGemをつかって更新する仕組みととらえることができます。また、Default gemsはアンインストールできません。Default gemsの例としてはTime、URI、YAMLなどがあります。

Bundled gemsは最初に説明した3種類の最後の Gem と同じですが、Rubyと一緒にインストールされるGemです。Rubyの開発と共にテストされていて、対象Rubyバージョンでのインストールおよび動作が確認されているメリットがあります。一緒にインストールされているだけでBundlerで特別扱いされることはなく、Bundled gemsをBundlerでつかうときには一般のGem(Default gemsでもなく、Bundled gemsでもないもの)と同様にGemfileへ追記が必要です。Rubyインストール時よりも新しいバージョンのBundled gemsが提供されていれば、BundlerやRubyGemsをつかって新しいバージョンをつかうことができます。Bundled gemsはアンインストールすることもできます。Bundled gemsの例としてはRake、TypeProfなどがあります。

- 参考資料
  - [徹底解説！ default gemsとbundled gemsのすべて](https://gihyo.jp/article/2024/01/ruby3.3-bundled-gems)

## deprecatedカテゴリのWarningを出力する

deprecated カテゴリのWarningはデフォルトで出力されません。（Ruby2.7.1から2.7.2へのバージョンアップでデフォルトでは出力されなくなる仕様変更がありました。）deprecated カテゴリのWarningを出力するときは次のようにオプションを指定します。RUBYOPT環境変数をつかうとrspecコマンドなどでRubyに対してオプションを指定できます。

```
$ ruby -W:deprecated foo.rb
$ RUBYOPT='-W:deprecated' bin/rspec spec/foo_spec.rb
```

また、 `-W:deprecated` の代わりに `-W` と書くとさらに広い範囲で全ての警告を表示します。
