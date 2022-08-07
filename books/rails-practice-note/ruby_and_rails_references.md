---
title: "[Ruby基礎][Rails基礎] RubyとRailsでの調べ方"
---

# RubyとRailsでの調べ方

わからない機能やメソッドがあったときに調べる公式サイトと、そのつかい方を説明します。

## Railsでの調べ方

### Railsガイド

- 日本語版: [https://railsguides.jp](https://railsguides.jp)
- 英語版: [https://guides.rubyonrails.org](https://guides.rubyonrails.org)

RailsガイドにはRailsの多くの機能についての説明が書かれています。私はRailsの機能のつかい方がわからないときに最初にあたるサイトとしてつかっています。調べ物でつかうのも良いですが、一度通読しておくと、どのような機能があるかを把握できるのでその後の開発を加速させることができます。Railsの機能開発とセットで書かれているため、最新バージョンのRailsに対応した記事を読むことができます。たまに読み返すと、新機能が追記されていて「こんな書き方もできるのか」と勉強になります。時間をかけてでも全体へ目を通す価値のある資料です。

### api.rubyonrails.org

- [https://api.rubyonrails.org](https://api.rubyonrails.org)

Railsのリファレンスページです。メソッド名やクラス名で検索することができます。Railsのコード中に書かれたドキュメントを、読みやすく検索しやすく提供しているページです。

また、各メソッドの説明からGitHubの該当ソースコードへのリンクも貼られているので、ソースコードを調べる入り口としても便利です。

詳しく説明が書かれているので、知らない機能について調べるときに最初に読む資料としてRailsガイドとあわせてお勧めです。

## Rubyでの調べ方

### Rubyリファレンスマニュアル

- [https://docs.ruby-lang.org/ja/latest/doc/index.html](https://docs.ruby-lang.org/ja/latest/doc/index.html)

RubyリファレンスマニュアルをつかうとRubyの各機能について調べることができます。通称は「るりま」です。

たとえばArrayクラスのメソッド一覧を調べるには、「組み込みライブラリ Builtin libraries - Array」とたどります。わからない機能を調べる辞書的なつかい方のほかに、全体を一読するのもお勧めです。まずはよくつかうArray, Hash, String, Enumerableのページだけでも読んでおくとRubyを書く力が上がります。

また、Rubyリファレンスマニュアルの全文検索として[るりまサーチ](https://docs.ruby-lang.org/ja/search/)のサイトがあります。たとえば、メソッド名を入力して検索すると、そのメソッドを持つクラス一覧を表示して、リファレンスマニュアルへのリンクが表示されます。

Rubyリファレンスマニュアルの利用方法は[『ゼロからわかる Ruby超入門』](https://gihyo.jp/book/2018/978-4-297-10123-7) でも説明しています。
