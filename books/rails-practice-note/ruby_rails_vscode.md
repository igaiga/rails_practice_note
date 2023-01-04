---
title: "[Ruby基礎][Rails基礎] Visual Studio CodeでRailsアプリを開発"
---

# Visual Studio CodeでRailsアプリを開発

Visual Studio Code(以降VSCode)での開発を便利にするノウハウを紹介していきます。

## debug gem

次のページにデバッガのつかい方を書いています。

[debug gem - 実行中のRubyコード、RailsコードをVSCodeを起動してデバッグ](https://zenn.dev/igaiga/books/rails-practice-note/viewer/ruby_debug_gem#%E5%AE%9F%E8%A1%8C%E4%B8%AD%E3%81%AEruby%E3%82%B3%E3%83%BC%E3%83%89%E3%80%81rails%E3%82%B3%E3%83%BC%E3%83%89%E3%82%92vscode%E3%82%92%E8%B5%B7%E5%8B%95%E3%81%97%E3%81%A6%E3%83%87%E3%83%90%E3%83%83%E3%82%B0)

## Docker

次のページにDockerとdocker-composeのつかい方を書いています。

[debug gem - VSCode上でdocker-compose.ymlをつかってDockerを起動してRailsアプリをデバッグ](https://zenn.dev/igaiga/books/rails-practice-note/viewer/ruby_debug_gem#vscode%E4%B8%8A%E3%81%A7docker-compose.yml%E3%82%92%E3%81%A4%E3%81%8B%E3%81%A3%E3%81%A6docker%E3%82%92%E8%B5%B7%E5%8B%95%E3%81%97%E3%81%A6rails%E3%82%A2%E3%83%97%E3%83%AA%E3%82%92%E3%83%87%E3%83%90%E3%83%83%E3%82%B0)

## RuboCop

vscode-ruby-light拡張をつかうと、書いているコードをリアルタイムにRuboCopで検査することができます。また、オートコレクトもワンクリックで行うことができます。

- [vscode-ruby-light](https://marketplace.visualstudio.com/items?itemName=r7kamura.vscode-ruby-light)

vscode-ruby-light拡張を有効にして、.rubocop.ymlがあり、rubocopコマンドが実行可能になっていれば、自動でRuboCopを実行して結果を表示します。

![](/images/rails_practice_note/vscode/vscode_rubocop1.png)

左側のアイコンをクリックすると、オートコレクトを実行することもできます。

![](/images/rails_practice_note/vscode/vscode_rubocop2.png)
![](/images/rails_practice_note/vscode/vscode_rubocop3.png)

## codeコマンド

[VSCode CLI](https://code.visualstudio.com/docs/editor/command-line) をインストールすると、ターミナルでcodeコマンドがつかえるようになります。codeコマンドをつかうとVSCode組み込みではないターミナルからでもVSCodeを起動することができます。また、VSCode組み込みのターミナルをつかっているときはCommandキー(macOS)やCtrlキー(Windows)を押してパスをクリックすることでそのファイルを開くことができます。

- `code .`
  - 現在のフォルダを開いてVSCodeを起動します
  - Railsアプリを開くときにRailsルートで実行するとRailsのコード群を選択しやすいです
- `code path`
  - 指定されたパスのファイルを開いてVSCodeを起動します
  - 例: `code work/sample.rb`
- `code -g path:line`
  - 指定されたパスのファイルの指定行(line)を開いてVSCodeを起動します
  - 例: `code -g work/sample.rb:5`

