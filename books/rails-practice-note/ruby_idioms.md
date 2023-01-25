---
title: "[Ruby基礎] Rubyコードの慣用句"
---

# Rubyコードの慣用句

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

また、パターンマッチと呼ばれるcase in構文もあります。

```ruby
case [1, 2, 3]
in [1, 2, a]
  puts a
else
  puts "others"
end
#=> 3
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

`String.new` のほかには、単項+をつかって `+"abc"`と書く方法や、Object#dupメソッドがfrozenではないオブジェクトを複製して返すことをつかって `"abc".dup` と書く方法もあります。

frozenなオブジェクトにしておくと、実行時の高速化で有利になったり、Ractorの共有可能オブジェクトとしてつかえるなどのメリットを得ることができるため、可能ならば最初からfrozen_string_literal行を書いて置くのが良いでしょう。既存のファイルにfrozen_string_literal行を足すときは、ソースコードに書かれた各Stringオブジェクトに対して変更を加えていないことを確認する必要があります。

また、このようにファイル先頭に書いて動作を指示する特別なコメント行のことをマジックコメントと呼びます。他には `# coding: utf-8` とファイルの文字コードエンコーディングを指定するマジックコメントもあります。ただし、codingマジックコメントのデフォルトはUTF-8なので、UTF-8のファイルをつかっているときには省略可能です。
