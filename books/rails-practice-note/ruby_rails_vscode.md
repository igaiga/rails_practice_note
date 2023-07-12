---
title: "[Ruby基礎][Rails基礎] Visual Studio CodeでRailsアプリを開発"
---

# Visual Studio CodeでRailsアプリを開発

Visual Studio Code(以降VSCode)での開発を便利にするノウハウを紹介していきます。

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

## debug gem

debug gemとVSCodeのデバッグ機能を組み合わせてつかう方法です。debug gemについて詳しくは [debug gem](https://zenn.dev/igaiga/books/rails-practice-note/viewer/ruby_debug_gem) のページに書いています。

事前準備として、VSCodeへ[VSCode rdbg Ruby Debugger拡張](https://marketplace.visualstudio.com/items?itemName=KoichiSasada.vscode-rdbg)をインストールしておきます。

また、ターミナルからVSCodeを起動するcodeコマンドがないときは、前の節を参考に [VSCode CLI](https://code.visualstudio.com/docs/editor/command-line) をインストールしておきます。

### VSCode上でRubyコード、Railsコードをデバッグ

VSCode上でブレークポイントを設定し、メニューから 実行 - デバッグの開始 を実行したときにdebug gemをつかってデバッグする方法です。プログラム実行がブレークポイントを設定したコードへ到達すると、debug gemの機能で一時停止し、VSCodeのパネルからステップ実行や再開を行うことができます。

「デバッグの開始」時の動作を設定するファイルは `.vscode/launch.json` です。ファイルがなければ自動作成されます。たとえば、rails serverを起動するためには.vscode/launch.jsonを次のように編集します。各設定の意味は後述します。

- .vscode/launch.json

```.vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Rails",
      "type": "rdbg",
      "request": "launch",
      "cwd": "${workspaceRoot}",
      "script": "bin/rails server",
      "args": [],
      "askParameters": true,
      "useBundler": true,
    }
  ]
}
```

VSCodeのメニューから 実行 - デバッグの開始 を実行すると、ダイアログでこれから実行するコマンドが表示されるので確認して実行します。

![](/images/rails_practice_note/vscode/vscode_run_command.png)

VSCodeに表示しているソースコードの行番号の左側をクリックするとブレークポイントが設定できます。ブラウザからRailsアプリへアクセスして、ブレークポイントまで処理が進むと一時停止します。メニューから 表示 - デバッグコンソール を実行すると、下部にパネルが表示され、debug gem コマンドを実行できます。

![](/images/rails_practice_note/vscode/vscode_debug_console.png)

実際に実行されていてるコマンドはたとえば次のようになっています。

```
rdbg --command --open --stop-at-load --sock-path=/var/folders/tv/b9rpcj011w110tjd0kv0_8h80000gn/T/ruby-debug-sock-501/ruby-debug-igaiga-18173 -- bundle exec ruby bin/rails server
```

launch.jsonの各設定の意味は次の通りです。

- `"name": "Debug Rails"` この構成の名前
- `"type": "rdbg"` この構成の種類(つかう拡張)
- `"request": "launch"` 「デバッグの開始」時の設定
- `"cwd": "${workspaceRoot}"` カレントディレクトリ設定
- `"script": "bin/rails server"` 実行するコマンド
- `"args": []` コマンドに渡すオプション
- `"askParameters": true` 実行前にダイアログで実行コマンドを確認
- `"useBundler": true` bundle exec を付与

また、環境変数を指定したいときは、.vscode/launch.jsonのconfigurationsに次のように追加します。

```.vscode/launch.json
"configurations": [
  {
    #...
    "env": {
      "WEB_CONCURRENCY": 0
    }
  }
]
```

サンプルコード: https://github.com/igaiga/rails704_ruby312_docker

### ターミナルで実行中のRubyプログラムからVSCodeへ接続してデバッグ

ターミナル上でRubyプログラムを実行しているとき、debug gemはターミナルの標準入出力をつかってデバッグコンソールを起動しますが、そこからVSCodeへ接続してデバッグすることもできます。

binding.breakで起動しているデバッグコンソールで`open vscode`コマンドを実行すると、VSCodeが起動して接続し、デバッグすることができます。`open chrome` のときと同様です。

Chromeデベロッパーツールとの接続はTCP接続(URLで接続先を指定)でしたが、VSCodeとの接続はUNIX domain socket(ファイルパスで接続先を指定)で接続されます。

## VSCode上でdocker-compose.ymlをつかってDockerコンテナを起動

VSCode上でDockerコンテナを起動して開発する方法です。事前に [Dev Containers拡張](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) をインストールしておきます。

VSCodeの左下アイコンをクリックして"Open Folder in Container (Remote-Containers)" コマンドを実行しますが、そのときの動作を設定するファイル `.devcontainer/devcontainer.json` を次の内容で置いておきます。

- .devcontainer/devcontainer.json

```.devcontainer/devcontainer.json
// For format details, see https://aka.ms/devcontainer.json. For config options, see the README at:
// https://github.com/microsoft/vscode-dev-containers/tree/v0.245.2/containers/ruby
{
  "name": "devcontainer settings for cimg/ruby",
  "dockerComposeFile": ["../docker-compose.yml"],
  "service": "app",
  "runServices": ["app"],
  "workspaceFolder": "/home/circleci/project",
  "remoteUser": "circleci",
  "customizations": {
    "vscode": {
      "extensions": [
      ],
      "settings": {
      }
    }
  }
}
```

name, service, runServices, workspaceFolder, remoteUser などはdocker-compose.ymlの設定に合わせて書いておきます。

docker-compose.ymlもRailsルートフォルダへ置いておきます。

- docker-compose.yml

```docker-compose.yml
version: '3'

services:
  app:
    image: cimg/ruby:3.1.2
    ports:
      - "3000:3000"
    volumes:
      - .:/home/circleci/project
    command: sleep infinity
```

VSCodeの左下アイコンをクリックして"Open Folder in Container (Remote-Containers)" コマンドを実行して、 `.devcontainer/devcontainer.json` と `docker-compose.yml` を配置したRailsアプリのフォルダを選択します。

VSCodeメニューの 表示 - ターミナル を実行すると、下部にパネルが表示され、コンテナ内で起動したターミナルとして利用できます。`docker compose exec app /bin/bash` コマンドを実行したときのターミナル相当の動作です。

rails serverを起動するときは `-b 0.0.0.0` オプションを追加してください。ホスト上のブラウザからコンテナ上で起動するrails serverへ接続するための設定です。

## VSCode上でDockerコンテナを起動してRailsアプリをデバッグ

前述の「VSCode上でdocker-compose.ymlをつかってDockerコンテナを起動」節の環境でデバッグ機能をつかうときの設定方法です。

ソースコードを開いてVSCodeからブレークポイントを設定します。

実行 - デバッグの開始 を選んでデバッガとrails serverを起動しますが、ホストからコンテナ上で起動するrails serverへ接続するために `-b 0.0.0.0` オプションを追加したいので、前述「Rubyコード、Railsコードをデバッグ」で書いたlaunch.jsonの`args` 設定を `"args": ["-b 0.0.0.0"]` へ変更します。launch.json全体は次のようになります。

- .vscode/launch.json

```.vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Rails",
      "type": "rdbg",
      "request": "launch",
      "cwd": "${workspaceRoot}",
      "script": "bin/rails server",
      "args": ["-b 0.0.0.0"],
      "askParameters": true,
      "useBundler": true,
    }
  ]
}
```

実行されるコマンド例は `rdbg --command --open --stop-at-load --sock-path=/tmp/ruby-debug-sock-3434/ruby-debug-ruby-debug-727 -- bundle exec ruby bin/rails server -b 0.0.0.0` となっていて、UNIX domain socketをつかってdebug gemとVSCodeが接続しています。また、VSCodeから事前に必要な作業が提案されるので、必要に応じてrdbg拡張のコンテナへのインストールやbundle installなどの作業を実行します。

ブラウザから http://localhost:3000 以下へアクセスして、ブレークポイントを設定した場所まで処理が進むと一時停止します。

サンプルコード: https://github.com/igaiga/rails704_ruby312_docker

## RuboCop

RuboCopに標準で入っているLSP機能と、それをつかうVSCode拡張 vscode-rubocop をインストールすることで、VSCode上で書いているコードをリアルタイムにRuboCopで検査することができます。

- VSCode拡張 vscode-rubocop https://marketplace.visualstudio.com/items?itemName=rubocop.vscode-rubocop
- RuboCop Docs LSP: https://docs.rubocop.org/rubocop/usage/lsp.html

セットアップはVSCode拡張 vscode-rubocop をVSCodeへインストールして、有効にするだけです。rubocopコマンドが実行可能になっていれば、自動でRuboCopを実行して結果を表示します。手動でLSPを起動するコマンドを実行する必要はありません。

![](/images/rails_practice_note/vscode/vscode_rubocop1.png)

## 参考資料

- Ruby in Visual Studio Code
  - https://code.visualstudio.com/docs/languages/ruby
  - VSCodeへRubyLSP, VSCode rdbg Ruby Debugger の各拡張をセットアップするドキュメント
