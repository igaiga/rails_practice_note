---
title: "[Rails基礎] RuboCop カスタムCopのつくり方"
---

RuboCopに自作のカスタムCop（検査ルール）を追加実装する方法です。

## カスタムCop用 Gemを作成

rubocop-extension-generator gemをつかって、これからつくるカスタムCop gemをローカルに生成します。

- https://github.com/rubocop/rubocop-extension-generator

rubocop-extension-generatorコマンドにつづけて新しくつくるカスタムCop用gem名を指定します。rubocop-xxxという名前にするルールになっています。

- $ rubocop-extension-generator rubocop-igaiga

いくつか質問が出てくるので、答えながら設定を進めます。

生成後、rubocop-name.gemspecファイルを編集して、情報を埋めておきます。TODOの部分に入力していないと、bundle installコマンド実行時にエラーメッセージと共に入力を促されます。

## 新しいCopを生成

これからつくる新しいCopをどのカテゴリーに所属させるかを決めます。主なカテゴリー種類は以下の通りです。

- Layout cops: インデントやスペースの有無を検査します
- Lint cops: バグの可能性があるコードを指摘します
- Metrics cops: クラスの長さやメソッドの長さなど、コードの複雑さを測定します
- Naming cops: メソッド名や定数名など、 名前に関する検査をします
- Security cops: 脆弱性につながる可能性がある問題など、 セキュリティ観点での検査をします
- Style cops: コーディング規約に沿っているかを検査します

ここではStyleカテゴリのreplace_elsifという名前のCop定義ファイルを作成します。if elsifが書かれていたときに、case whenで書き換えられる旨を指摘するCopです。次のコマンドで新しいCop定義ファイルを生成します。

- $ `bundle exec rake new_cop\[Style/replace_elsif\]`

```
[create] lib/rubocop/cop/style/replace_elsif.rb
[create] spec/rubocop/cop/style/replace_elsif_spec.rb
[modify] lib/rubocop/cop/igaiga_cops.rb - `require_relative 'style/replace_elsif'` was injected.
[modify] A configuration for the cop is added into config/default.yml.
Do 4 steps:
  1. Modify the description of Style/ReplaceElsif in config/default.yml
  2. Implement your new cop in the generated file!
  3. Commit your new cop with a message such as
     e.g. "Add new `Style/ReplaceElsif` cop"
  4. Run `bundle exec rake changelog:new` to generate a changelog entry
     for your new cop.
```

表示された4ステップをそれぞれ対応していきます。

- 1. config/default.yml の Style/ReplaceElsif の情報を書きます
- 2. 生成されたファイルにCopを実装します
- 3. git commitします
- 4. Changelogへ追記

## config/default.ymlを編集

1番はconfig/default.ymlのStyle/ReplaceElsifの情報を埋めます。変更例は以下です。

```yaml
Style/ReplaceElsif:
  Description: "elsif can be replaced by case when"
  Enabled: true
  VersionAdded: "0.1.0"
```

Enabledにはデフォルト時の有効無効設定を書きます。VersionAddedには追加されるときの該当Gemのバージョンを書きます。

## テストコード実装

2番では生成されているファイルを編集してCopを実装していきます。

生成時にテストファイルも合わせて生成されているので、先に実装しておくとテスト実行しながらCopを実装できて便利です。ここではRSpecファイルを生成した例として spec/rubocop/cop/style/replace_elsif_spec.rb にテストコードを実装します。

```ruby
# frozen_string_literal: true

RSpec.describe RuboCop::Cop::Style::ReplaceElsif, :config do
  let(:config) { RuboCop::Config.new }

  it "Registers an offense when using `elsif`" do
    expect_offense(<<~RUBY)
      x = 1
      y = 2
      if x == 9
      elsif y == 2
      ^^^^^^^^^^^^ Style/ReplaceElsif: `elsif` can be replace with `case`[...]
      end
    RUBY
  end

  it "No offenses" do
    expect_no_offenses(<<~RUBY)
      x = 1
      y = 2
      case
      when x == 9
      when y == 2
      end
    RUBY
  end

  # it "auto corrected code is valid" do
  #   expect_offense(<<~RUBY)
  #     x = 1
  #     y = 2
  #     if x == 9
  #       puts "foo"
  #     elsif y == 2
  #     ^^^^^^^^^^^^ Style/ReplaceElsif: `elsif` can be replace with `case`[...]
  #       puts "bar"
  #     else
  #       puts "baz"
  #     end
  #   RUBY
  #
  #   corrected_code = <<~RUBY
  #     x = 1
  #     y = 2
  #     case
  #     when x == 9
  #       puts "foo"
  #     when y == 2
  #       puts "bar"
  #     else
  #       puts "baz"
  #     end
  #   RUBY
  #
  #   expect_correction(corrected_code)
  # end
end
```

テストコード中の `[...]` は任意の文字列を省略して書く記法です。

`bundle exec rspec spec/rubocop/cop/style/replace_elsif_spec.rb` を実行できることを確認しておきます。

## Copコード実装

lib/rubocop/cop/style/replace_elsif.rb にCopを実装します。実装例は次の通りです。

```ruby
# frozen_string_literal: true

module RuboCop
  module Cop
    module Style
      # @example
      #   # bad
      #   if x == 1
      #     do_something1
      #   elsif y == 2
      #     do_something2
      #   end
      #
      #   # good
      #   case
      #   when x == 1
      #     do_something1
      #   when y == 2
      #     do_something2
      #   end
      #
      #   # good
      #   case x
      #   when (1..20)
      #     do_something1
      #   when (21..)
      #     do_something2
      #   end
      #
      class ReplaceElsif < Base
        MSG = "`elsif` can be replace with `case`."

        def on_if(node)
          return unless node.elsif?

          add_offense(node)
        end
      end
    end
  end
end
```

`# @example` につづけて良い例、悪い例をコメントで書きます。MSGには該当したときに表示するメッセージを書きます。

ここでは、`on_if(node)` メソッド中で各ノードに対して検査を行い、警告に該当するときは `add_offense(node)` メソッドでその部分に警告がある旨を登録します。警告するだけでなく、自動置換に対応するよう実装することも可能なので、後述します。

`on_if(node)` メソッドはif式が書かれているときに呼び出されます。nodeにはRuboCopでつかわれているParser gemが静的解析したASTの該当部分であるRuboCop::AST::IfNodeオブジェクトが代入されています。このオブジェクトにmethodsメソッドで利用可能なメソッドを調べたところ、`elsif?` メソッドがありました。これでelsifがつかわれているノードかどうか調べることができそうです。elsifノードのときに `add_offense(node)` メソッドを呼び出すように実装しています。テストが通るようにCopを実装していきます。

node検査方法について詳しくはRuboCop公式ページに詳しく書かれています。`on_if` メソッド以外の対象ノードが出てきたときに呼ばれるメソッド群には `on_send`, `on_block` ほかが用意されています。また、`binding.irb`などで止めてmethodsメソッドなどをつかってオブジェクトに利用できるメソッドを尋ねるのも良い方法です。

RuboCop公式ページ "Development": https://docs.rubocop.org/rubocop/development.html

## lib/rubocop/cop/gem_name_cops.rb の require_relative を確認

lib/rubocop/cop/gem_name_cops.rb には、このGemがつかわれるときにrequireするファイル群を書いていきます。生成時に自動でCop名が追記されていることを確認すればOKです。

```ruby
# frozen_string_literal: true
require_relative 'style/replace_elsif'
```

実装ができたところでgit commitしておくと良いでしょう。また、CHANGELOG.mdも生成されているので、追加したCopを追記しておきます。

## rubocopコマンドで確認する

作成したCopをrubocopコマンドを実行して動かしてみます。

target.rbに検査対象コードを書いておきます。

```ruby
x = 1
y = 2
if x == 9
elsif y == 2
end
```

次のコマンドを実行します。`--require` オプションで、実装したCopファイルを指定します。`--only Style/ReplaceElsif` で実装中のCopのみ検査するように指示しています。 `--cache false` コマンドはRuboCopのキャッシュをつかわない設定です。RuboCopは `~/.cache/rubocop_cache/` フォルダ以下に過去の検査結果をキャッシュして、ファイルが未変更であるときなど、必要ないときには実行しないようになっています。

- $ bundle exec rubocop target.rb --require ./lib/rubocop/cop/style/replace_elsif.rb --only Style/ReplaceElsif --cache false

メッセージがいろいろと表示されますが、終盤で次のように指摘メッセージが意図通り表示されれば成功です。

```
...
Inspecting 1 file
C

Offenses:

target.rb:4:1: C: Style/ReplaceElsif: elsif can be replace with case.
elsif y == 2
^^^^^^^^^^^^

1 file inspected, 1 offense detected
```

## auto correct機能で修正可能にする

TODO: あとで書きます。

| メソッド |  |
| --- | --- |
| replace(node, content) | node を content で置換 |
| insert_before(node, content) | node の前に content を追加 |
| insert_after(node, content) | node の後に content を追加 |
| wrap(node, insert_before_content, insert_after_content) | nodeをinsert_before_contentとinsert_after_contentで挟みます |

## Gemを試験する

別の場所にたとえばrails newした新しいアプリを置いて、実装中のRubocop カスタムCop Gemをつかってみます。実装中のRubocop カスタムCop Gemには実装したファイルを忘れずにgit commitしておきます。

新しいアプリのGemfileへ以下を追記します。pathオプションには作成中のGemフォルダへの絶対パスを書きます。

```Gemfile
group :development do
  gem "rubocop", require: false
  gem "rubocop-igaiga", path: "/Users/igaiga/work/rubocop-igaiga", require: false
end
```

.rubocop.yml を次の内容で作成しておきます。

```.rubocop.yml
require:
  - rubocop-igaiga

AllCops:
  NewCops: enable
  SuggestExtensions: false
```

次のコマンドを実行して検査できればOKです。

- $ bundle exec rubocop --cache false --only Style/ReplaceElsif

# 参考資料

- RuboCop公式ドキュメント "Development"
    - https://docs.rubocop.org/rubocop/development.html
- RubocopのRSpecサンプルコード
    - https://github.com/rubocop/rubocop-rspec
- RuboCop gem cop置き場
    - https://github.com/rubocop/rubocop/tree/master/lib/rubocop/cop
- RuboCopコミッタであるkoicさんの講演資料 "Rubocopのしくみ"
    - https://speakerdeck.com/koic/rubocop-under-a-microscope
- Money Forward Developers Blog "Rubocop でカスタムルールを作る"
    - https://moneyforward.com/engineers_blog/2021/09/02/rubocop/
- 神速さんブログ "RuboCopの新しいルールを追加する方法（Custom Copの作り方）"
    - https://sinsoku.hatenablog.com/entry/2018/04/24/022911
