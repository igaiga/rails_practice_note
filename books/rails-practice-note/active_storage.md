---
title: "[ActiveStorage] Active Storageの各種機能"
---

# Active Storageの各種機能

## バリアント画像の生成タイミング

---

※注

「バリアント画像の生成タイミング」節で書かれている内容はRails 8.2 で以下の変更が入る予定です。
https://github.com/rails/rails/commit/3d105fc346fbb3121bbcf6340f2b19104bf326f0
processオプションへ:lazily, :later, :immediatelyを渡すことで切り替え可能になる予定です。`preprocessed: true` オプションを非推奨とし、代わりに `process: :later` を推奨、Rails 9.0 で削除予定とのこと。

---

　アップロードした画像をそのまま表示するのではなく、リサイズしたものを使いたいことがよくあります。Active Storageにはサイズ違い画像であるバリアントを生成する機能があります。

　バリアント画像の生成タイミングとして、「バリアント画像表示時まで遅延させて生成」、「アップロード時に非同期ジョブで生成」、「アップロード時に同期処理で生成」の3種類から選ぶことができます。

　「バリアント画像表示時まで遅延させて生成」がデフォルトの動作で、何も指示しなければこれになります。ブラウザからバリアント画像URLへアクセスした時にその画像がまだ存在しなければ、元画像から指定されたフォーマットへ変換し、生成した画像を置いたURLへリダイレクトします。

　「アップロード時に非同期ジョブで生成」する設定にするには、variantメソッドへ`preprocessed: true`オプションを追加します。アップロード時の作業の中でバリアント画像生成ジョブがキューに入ります。

```ruby
has_one_attached :portrait do |attachable|
  attachable.variant :thumb, resize_to_limit: [100, 100], preprocessed: true
end
```

　「アップロード時に同期処理で生成」するにはprocessedメソッドを使います。`portrait.variant(:thumb).processed`のように書くことで、processedメソッドが呼び出されたときにバリアント画像が生成されます。生成処理が終わるまでレスポンスを返さないことになるので、変換処理時間が長い場合には不向きです。

　生成されたバリアント画像は設定されているストレージへ格納されて、2回目以降は変換処理をせずに生成済みの画像URLへリダイレクトします。生成済みかどうかをストレージへ確認しにいくのは時間がかかるため、生成済みバリアント画像情報をactive_storage_variant_recordsテーブルへ格納することで、既に生成されているかをDBアクセスで調べることができます。この機能はconfig.active_storage.track_variants設定で有効無効を設定できます。

## ファイル配信方法

　Active Storageはファイル配信方法として「リダイレクトモード」と「プロキシモード」の2種類をサポートしています。

　リダイレクトモードはデフォルトのファイル配信方法です。ファイル(Blob)に対して永続的なアプリケーションURLを生成します。このURLにアクセスすると、ファイル実体へアクセスする有効期限(デフォルトは5分間)のある署名付きURLへリダイレクトされます。リクエストごとに期限付きのURLを生成することで、仮にファイルのURLが公開されても被害を限定的に抑えることができます。

　別の配信方法としてプロキシモードを利用することもできます。アプリケーションサーバがリクエスト処理中にストレージサービスからファイルをダウンロードしてレスポンスとして返します。プロキシモードではファイルに対して永続的なアプリケーションURLが生成されます。

　CDNをつかうときはプロキシモードが適しています。Active Storageのプロキシコントローラは、デフォルトでCDNへレスポンスをキャッシュするように指示するHTTPヘッダーを設定するので、そのまま利用できます。生成されるURLとしてアプリのホストではなくCDNのホストを使うなど、設定について詳しくはRailsガイドの「ActiveStorageの概要」ページの「Active Storageの手前にCDNを配置する」の節を参照してください。（脚注: https://railsguides.jp/active_storage_overview.html ）

　アプリ全体をプロキシモードで動作させるには、たとえばconfig/initializers/active_storage.rbへ次のように設定を書きます。

```ruby
Rails.application.config.active_storage.resolve_model_to_route = :rails_storage_proxy
```

　また、特定ファイルのみをプロキシモードで配信させるときは、`rails_storage_proxy_path` URLヘルパーで生成したURLを使います。

```ruby
<%= image_tag rails_storage_proxy_path(@book.image) %>
```

## ダイレクトアップロード機能

　Active Storageはダイレクトアップロードにも対応しています。これにより、アプリケーションサーバを経由せずに、ファイルを直接クラウドストレージなどへアップロードできます。アプリケーションサーバを経由しないことで、アップロードにかかる時間を減らすことができ、さらにアプリケーションサーバへの負荷を減らすことができます。

　ダイレクトアップロード機能を利用するために、用意されているJavaScriptライブラリを使う設定を3カ所に書き加えます。以下の説明はImport mapsを使う想定で書いていますが、別のJavascriptライブラリを使うときの設定はRailsガイドなどを参照してください。

　config/importmap.rbに以下を追加します。

```ruby
pin "@rails/activestorage", to: "activestorage.esm.js"
```

　次にapp/javascript/application.jsへ以下を追加します。

```js
import * as ActiveStorage from "@rails/activestorage"
ActiveStorage.start()
```

　最後にform.file_fieldメソッドへ`direct_upload: true`オプションを追加します。


```erb
<%= form.file_field :portrait, direct_upload: true %>
```

　`direct_upload: true`を追加することで、フォームのサブミットボタンを押したタイミングでファイルをストレージへダイレクトアップロードするようになります。複数のファイルを同時にダイレクトアップロードしたい場合はさらに`multiple: true`オプションも追加します。

　ダイレクトアップロード機能が使われた場合には、リクエストのログ先頭に次のように表示されます。

```
Started POST "/rails/active_storage/direct_uploads" for ::1 at 2026-02-13 15:25:18 +0900
Processing by ActiveStorage::DirectUploadsController#create as JSON
```

　クラウドストレージへダイレクトアップロードするときは、ストレージサービスにCORSを設定して、自分のアプリからのクロスオリジンリクエストを許可してください。詳しくはRailsガイド ActiveStorageの概要 ページの「12.2 CORS（Cross-Origin Resource Sharing）を設定する」を参照してください。

　また、ダイレクトアップロード実行時に発火するJavaScript用のイベントが用意されています。これらのイベントを使うとアップロードの進捗状況を表示したりするなどユーザ体験を向上させることができます。詳しくはRailsガイド ActiveStorageの概要 ページの「12.3 ダイレクトアップロードのJavaScriptイベント」を参照してください。

## ダイレクトアップロード機能でバリデーションエラー時のファイル再送信を防ぐ

　次のような、名前必須のバリデーションを追加したケースを考えます。

```ruby
class User < ApplicationRecord
  validates :name, presence: true
  has_one_attached :portrait
end
```

　画像を添付したけれど名前を入力せずにフォームを送信した場合は、バリデーションエラーとなってUserモデルのレコードは保存されません。添付ファイルが属するモデルのレコードがバリデーションエラーで保存されなかったとき、一緒に添付されたアップロードファイルはどのような状態になるでしょうか。この結果はダイレクトアップロード機能が使われたかどうかにより異なります。

　ダイレクトアップロード機能を使わずにアップロードされたケースでは、フォームに添付したファイルは関連するレコードが保存に成功するまでストレージサービスへ送信されません。つまり、フォームを送信したモデルがバリデーションに失敗すると、そのときのファイルは失われ、再度アップロードする必要があります。

　ダイレクトアップロード機能を使ったケースでは、フォーム送信前に添付ファイルが送信されて保存されます。モデルがバリデーションに失敗して保存されなかったときも、アップロードされたファイルはActiveStorage::Blobモデルのレコードとして保存されています。hidden_fieldに保存済みファイルのsigned_idを埋めておけば、バリデーションエラーを解消してフォームを再送信するときに、再アップロードせずに添付ファイルをモデルと紐付けることができます。コード例は次のようになります。

```erb
<% if user.portrait.attached? %>
  <%= form.hidden_field :portrait, value: user.portrait.signed_id %>
  <p> ファイルはアップロード済みです（変更する場合は再選択してください）</p>
<% end %>
<%= form.file_field :portrait, direct_upload: true %>
```

　元々あった`<%= form.file_field :portrait, direct_upload: true %>`の前に、`user.portrait.attached?`のときにhidden_fieldで`user.portrait.signed_id`を埋め込むコードを追加します。後ろのfile_fieldでファイル未選択のときは、このsigned_idからActiveStorage::Blobモデルのレコードを探して保存するモデルと紐付けます。file_fieldでファイルが選択されたときは、添付ファイルが再送信されて保存され、新しく保存されたファイルがモデルと紐付けられます。

　必要に応じてモデルと紐付けされなかったファイルを掃除すると良いでしょう。`ActiveStorage::Blob.unattached`メソッドを使うとモデルと紐付けされていないファイルのレコード群を取得できます。1日以上前に保存された未紐付けファイル群を掃除するコード例は`ActiveStorage::Blob.unattached.where(created_at: ..1.day.ago).find_each(&:purge)`となります。

## Active Storageのコントローラクラス群と対応するURL

　Active Storageは以下のルーティングをデフォルトで生成します。

| HTTP Verb | Path | Controller#Action |
| :--- | :--- | :--- |
| GET | /rails/active_storage/blobs/redirect/:signed_id/*filename | ActiveStorage::Blobs::RedirectController#show |
| GET | /rails/active_storage/blobs/:signed_id/*filename | ActiveStorage::Blobs::RedirectController#show |
| GET | /rails/active_storage/blobs/proxy/:signed_id/*filename | ActiveStorage::Blobs::ProxyController#show |
| GET | /rails/active_storage/representations/redirect/:signed_blob_id/:variation_key/*filename | ActiveStorage::Representations::RedirectController#show |
| GET | /rails/active_storage/representations/:signed_blob_id/:variation_key/*filename | ActiveStorage::Representations::RedirectController#show |
| GET | /rails/active_storage/representations/proxy/:signed_blob_id/:variation_key/*filename | ActiveStorage::Representations::ProxyController#show |
| GET | /rails/active_storage/disk/:encoded_key/*filename | ActiveStorage::DiskController#show |
| PUT | /rails/active_storage/disk/:encoded_token | ActiveStorage::DiskController#update |
| POST | /rails/active_storage/direct_uploads | ActiveStorage::DirectUploads#create |

　各コントローラの仕事は次のようになっています。

- ActiveStorage::Blobs 配下のコントローラ : 原寸ファイルの表示など
- ActiveStorage::Representations 配下のコントローラ : バリアントファイルの表示など
- ActiveStorage::DiskController : ローカルディスク使用時のアップロードなど
- ActiveStorage::DirectUploadsController : ダイレクトアップロード機能など
- ActiveStorage::BaseController : ActiveStorageの各コントローラ共通の親クラス

　デフォルトのルーティングを生成させないようにするには、`Rails.application.config.active_storage.draw_routes = false`設定を追加します。

## ファイルへのアクセス制限

　アップロードしたファイルにアクセスできるユーザを制限したいケースを考えます。たとえば投稿された画像へのアクセスをログイン済みユーザーのみへ限定したいケースなどです。

　Active Storageは前の節で挙げた/rails/active_storage/から始まるURL群を生成します。これらのURLは全てデフォルトでpublicであり、URLを知っていれば誰でもアクセス可能です。また、ActiveStrorageの各コントローラはApplicationControllerを継承していないので、ApplicationControllerのbefore_actionに認証認可するコードが書かれていても実行されません。

　これらのpublicなURLで認証処理を行う1つの設計方法は、ActiveStorageの各コントローラ共通の親クラスであるActiveStorage::BaseControllerにbefore_actionで認証認可するコードを追加する方法です。ActiveStorageの全コントローラでbefore_actionを使って認証処理をかけておき、認証不要なURLがあればそのコントローラでskip_before_actionすることで処理を外すこともできます。

config/initializers/active_storage.rb

```ruby
Rails.configuration.to_prepare do
  ActiveStorage::BaseController.class_eval do
    before_action :authorize_active_storage_access
    def authorize_active_storage_access
      # 認証認可を行うコードをここに書く
    end
  end
```

　ActiveStorage::BaseControllerへbefore_actionで認証認可を行うメソッドを登録しています。ActiveStorage::BaseControllerへコードを追加するときに、Rails.configuration.to_prepareメソッドブロックで囲み、ActiveStorage::BaseControllerへclass_evalしています。config/initializers/ 以下のコードは起動時にしか実行されないので、起動時と再読み込み時のどちらでも動作させるためにこの書き方をしています。

　認証が不要な機能に対して処理を外すにはskip_before_actionを書きます。たとえばダイレクトアップロード機能の認証処理を外すには、次のコードのようにActiveStorage::DirectUploadsControllerにskip_before_actionを書きます。

config/initializers/active_storage.rb

```ruby
Rails.configuration.to_prepare do
  # 認証不要のコントローラではskip_before_actionする
  ActiveStorage::DirectUploadsController.class_eval do
    skip_before_action :authorize_active_storage_access
  end
end
```

　Railsアプリが作るURLはこれで認証処理を挟むことができますが、S3などRailsアプリ外のURLが露出するときには対応が必要です。たとえば、配信方法をプロキシモードにしてコントローラ内でアプリ外ストレージからファイルを取得し、Railsアプリが生成したURLをユーザーに提供する方法があります。

　より詳細に認証をしたいとき、たとえばページごとに認証を行いたいケースでは自分でURLおよびコントローラを設計する方法もあります。前の節で出てきた`Rails.application.config.active_storage.draw_routes = false`設定を追加してデフォルトルーティングを生成しないようにして、自分でルーティングおよびコントローラを設計する方法です。

Railsガイド ActiveStorageの概要 「6.3 認証済みコントローラ」 ページも参考になります。

## プレビュー表示

　Active Storageは動画とPDFについて組み込みのプレビュー機能を持っています。たとえば、動画ファイルは最初のフレームの画像を、PDFファイルは最初のページ画像を表示できます。

　動画のプレビュー機能にはActiveStorage::Previewer::VideoPreviewerが使われ、FFmpegに依存しています。PDFのPreviewerは2種類提供されていて、1つはPopplerを使うActiveStorage::Previewer::PopplerPDFPreviewer、もう1つはMuPDFを使うActiveStorage::Previewer::MuPDFPreviewerです。Popplerはpoppler Gem、MuPDFはmupdf Gemでインストール可能です。また、PDFの各PreviewerではAdobe Illustratorの.aiファイルもプレビュー可能です。

　次のコードは、添付ファイルに対してpreviewメソッドを呼び出してプレビュー画像を表示するサンプルコードです。動画をプレビューするには事前にffmpegをインストールしてください。たとえばmacOSでは`brew install ffmpeg`でインストールできます。PDFをプレビューするにはpoppler Gemまたはmupdf GemのどちらかをGemfileに追加してインストールしてください。

```ruby
<%= image_tag(user.portrait.preview(resize_to_limit: [100, 100])) %>
```

　previewメソッドの引数 `resize_to_limit: [100, 100]` の部分は、事前に定義したバリアント名(以前の節で出てきた`:thumb`など)を書くこともできます。

　previewメソッドは対応していないファイルフォーマットではエラーになります。かわりにrepresentationメソッドを使うことで、プレビュー可能なときはプレビューを表示、それ以外のときはバリアントを表示できます。プレビューもバリアントも表示できないときはActiveStorage::UnrepresentableError例外を投げます。representable?メソッドを使うと、representationメソッドを呼び出す前に表示可能かどうかをチェックできます。次のコードはrepresentable?メソッドで表示可能かを判断して、trueのときはrepresentationメソッドを呼び出して表示、falseのときはファイルをリンク形式でダウンロードさせます。

```ruby
<% if user.portrait.attached? %>
  <% if user.portrait.representable? %>
    <%= image_tag user.portrait.representation(resize_to_limit: [100, 100]) %>
  <% else %>
    <%= link_to "ダウンロード", rails_blob_path(user.portrait, disposition: "attachment") %>
  <% end %>
<% end %>
```

　別フォーマットにプレビュー機能を追加するにはカスタムプレビューアを追加します。詳細はAPIリファレンス「ActiveStorage::Preview」（脚注： https://api.rubyonrails.org/classes/ActiveStorage/Preview.html ）に書かれています。

## ActiveStorage::Analyzerによるデータ解析

　Active Storageではファイルをアップロードしたときに、Active Jobを使った非同期ジョブでアナライザーが実行されファイルの解析が行われます。Active Jobはrails newした状態で利用可能で、development環境ではRailsのプロセスで処理するAsyncアダプターを使ってActive Storageのアナライザーによる解析が実行されます。production環境ではデフォルトでSolid Queueを使う設定になっているので、`bin/jobs start`コマンドなどで非同期バックエンドプロセスを起動してActive Jobのジョブを実行させてください。

　アナライザーにより解析された情報はactive_storage_blobsテーブルのmetadataカラムにJSON形式で保存されます。取得するときはmetadataメソッドを使います。

```ruby
user.portrait.metadata
#=> {"identified" => true, "width" => 2168, "height" => 2070, "analyzed" => true}
```

　Active Storageでは画像、動画、音声用の組み込みアナライザーをActiveStorage::Analyzerネームスペース以下に持っています。Rails.application.config.active_storage.analyzersで利用されるアナライザー一覧を調べることができます。Active Storage は登録済みの各アナライザーに対して順にaccept?メソッドを呼び出し、trueを返す最初のアナライザーを使います。

```ruby
Rails.application.config.active_storage.analyzers
#=> [ActiveStorage::Analyzer::ImageAnalyzer::Vips,
     ActiveStorage::Analyzer::ImageAnalyzer::ImageMagick,
     ActiveStorage::Analyzer::VideoAnalyzer,
     ActiveStorage::Analyzer::AudioAnalyzer]
```

　アナライザーによる解析の結果、ファイル種別ごとに情報を得ます。画像は幅（width）、高さ（height）の情報。動画は幅（width）、高さ（height）、再生時間（duration）、アスペクト比（display_aspect_ratio）、動画の存在を表すvideo（boolean）、音声の存在を表すaudio（boolean）の情報。音声は再生時間（duration）、ビットレート（bit_rate）、サンプリングレート（sample_rate）の情報です。

## 参考資料

- Railsガイド Active Storageの概要 https://railsguides.jp/active_storage_overview.html

