---
title: "[Ruby基礎][Rails基礎] RubyとRailsでの調べ方"
---

# RubyとRailsでの調べ方

わからないメソッドや機能があったときに調べる公式サイトと、その使い方を説明します。

## Railsでの調べ方

### Railsガイド

- 日本語版: [https://railsguides.jp](https://railsguides.jp)
- 英語版: [https://guides.rubyonrails.org](https://guides.rubyonrails.org)

RailsガイドにはRailsの多くの機能についての説明が書かれています。RailsガイドはRailsの機能開発とセットで書かれているため、常に最新バージョンのRailsに対応した記事を読むことができます。たまに読み返すと、新機能が追記されていて「こんな書き方もできるのか」と勉強になります。

調べたい機能がわかっているときは辞書のようにつかうこともでき、また、読み物のように全体を一通り読むことでRailsアプリ開発につかえる道具を広く身につけることもできます。時間をかけてでも全体へ目を通す価値のある資料です。

### api.rubyonrails.org

- [https://api.rubyonrails.org](https://api.rubyonrails.org)

Railsのソースコード中にコメント行で書かれているドキュメントを、メソッド名やクラス名で検索して読むことができる、Railsのリファレンスマニュアルです。

また、各メソッドの説明からGitHubの該当ソースコードへのリンクも貼られているので、ソースコードを調べる入り口としても便利です。

詳しく説明が書かれているので、知らない機能について調べるときに最初に読む資料としてRailsガイドとあわせてお勧めです。

## Rubyでの調べ方

### Rubyリファレンスマニュアル

- [https://docs.ruby-lang.org/ja/latest/doc/index.html](https://docs.ruby-lang.org/ja/latest/doc/index.html)

RubyリファレンスマニュアルをつかうとRubyの各機能について調べることができます。通称は「るりま」です。

たとえばArrayクラスのメソッド一覧を調べるには、「組み込みライブラリ Builtin libraries - Array」とたどります。わからない機能を調べる辞書的な使い方のほかに、全体を一読するのもお勧めです。まずはよくつかうArray, Hash, String, Enumerableのページだけでも読んでおくとRubyを書く力が上がります。

また、Rubyリファレンスマニュアルの全文検索として[るりまサーチ](https://docs.ruby-lang.org/ja/search/)のサイトがあります。たとえば、メソッド名を入力して検索すると、そのメソッドを持つクラス一覧を表示して、リファレンスマニュアルへのリンクが表示されます。

Rubyリファレンスマニュアルの利用方法は[『ゼロからわかる Ruby超入門』](https://gihyo.jp/book/2018/978-4-297-10123-7) でも説明しています。
