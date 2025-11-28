---
title: "[ActiveJob] Solid Queue"
---

# Solid Queue

## バックエンドにSolid Queueを設定

Solid Queueはデータベースをキューとして使うActive Job用バックエンドで、Rails8.0で追加されました。データベースとしてMySQL、PostgreSQL、SQLiteなどが利用可能です。FOR UPDATE SKIP LOCKED 句を使える場合はこれを利用してジョブをポーリングする時のブロックやロック待ちを回避します。

ここではdevelopment環境のActive Jobバックエンド設定をSolid Queueへ変更します。config/environments/development.rbへ、production環境と同様に次の設定を追加します。

config/environments/development.rb
```ruby
config.active_job.queue_adapter = :solid_queue
config.solid_queue.connects_to = { database: { writing: :queue } }
```

この設定でバックエンドとしてSolid Queueが使われ、ジョブキューとしてqueueデータベースへ接続します。ここでは全てのジョブでSolid Queueをつかう設定にしていますが、ジョブクラスごとにqueue_adapterメソッドをつかってバックエンドを切り替えることもでき、Solid Queueを段階的に導入するときに便利です。Solid Queueを段階的に導入するときは、デフォルトのバックエンドを `config.active_job.queue_adapter` で設定しておいて、バックエンド設定を変更するジョブで次のように`queue_adapter`メソッドを使って指定します。

app/jobs/something_job.rb
```ruby
class SomethingJob < ApplicationJob
  self.queue_adapter = :solid_queue
(略)
```

次に、config/database.ymlへdevelopment環境用のデータベース設定として`queue`キーを追加して書き換えます。今までの設定はprimaryキーとして設定します。キー名`queue`は、先ほど`config.solid_queue.connects_to = { database: { writing: :queue } }`設定に書いた名前`:queue`と同じにします。

config/database.yml
```ruby
development:
  primary:
    <<: *default
    database: storage/development.sqlite3
  queue:
    <<: *default
    database: storage/development_queue.sqlite3
    migrations_paths: db/queue_migrate
```

設定後、queueデータベースのマイグレーションを実行して必要なテーブル群を作成します。

```sh
$ bin/rails db:migrate:queue
```

queueデータベースのスキーマファイルはdb/queue_schema.rbとなります。スキーマファイルを見ると、solid_queue_ready_executions、solid_queue_scheduled_executionsなどのテーブルが作られたことがわかります。また、そのほかのSolid Queue設定ファイルとしてconfig/queue.yml、config/recurring.yml が置かれています。

動作確認のためにAsyncLogJobクラスとAsyncLogモデルを作っておきます。

app/jobs/async_log_job.rb
```ruby
class AsyncLogJob < ApplicationJob
  queue_as :default

  def perform(message: "hello")
    AsyncLog.create!(message: message)
  end
end
```

rails consoleを起動してジョブをキューへ加えてみましょう。この状態ではキューへ追加はされますが、まだジョブ実行はされません。

```ruby
active-job-example(dev)> AsyncLogJob.perform_later(message: "from Solid Queue")
略 Enqueued AsyncLogJob (Job ID: 略) to SolidQueue(default) with arguments: {:message=>"from Solid Queue"} 略
```

ジョブをキューへ追加できたら、次のコマンドでバックエンド処理を行うプロセスを起動してジョブを実行させます。

```sh
% bin/jobs start
```

ジョブが実行されて `AsyncLog.create!(message: message)` が行われたことを確認してみましょう。`AsyncLog.last` を見るとmessageが"from Solid Queue"なレコードが登録されている事が確認できます。

```sh
active-job-example(dev)> AsyncLog.last
=> #<AsyncLog id: 1, message: "from Solid Queue", 略>
```

　デフォルトではログは log/development.log へ記録され、bin/jobs を実行しているコンソールには何も表示されません。Solid Queueのロガーを次のように設定すると、ログがコンソールへ表示されます。

config/environments/development.rb
```ruby
config.solid_queue.logger = ActiveSupport::Logger.new(STDOUT)
```

## Mission Control - Jobs

Solid Queueのキューの状態を確認するWeb UIはmission_control-jobs Gem( https://github.com/rails/mission_control-jobs )として提供されています。ドキュメントを参考にしてセットアップしてください。また、production環境で利用する時は、適切なアクセス制限をかけてください。

Mission Controlをセットアップする手順は次の通りです。 `gem "mission_control-jobs"` をGemfileに追加してbundle installを実行します。routesへ以下を追記してマウントします。

config/routes.rb
```ruby
mount MissionControl::Jobs::Engine, at: "/jobs"
```

デフォルトでBASIC認証がかかるようになっているので、BASIC認証用のuser, password情報をcredentialsへ設定します。`bin/rails credentials:edit` を実行して、次の情報を追記します。

```ruby
mission_control:
  http_basic_auth_user: dev
  http_basic_auth_password: secret
```

rails serverを起動して、http://localhost:3000/jobs へアクセスするとキューの現在状況や完了したジョブの情報を見ることができます。

![Mission Control スクリーンショット](/images/rails_practice_note/solid_queue/mission_controll_ss_asynclogjob.png)

## ジョブの定期実行

Solid Queueはcronのようなジョブを定期実行する仕組みを提供しています。設定ファイルは`config/recurring.yml`です。デフォルトでは完了ジョブを消すジョブが12分ごとに繰り返し実行されるタスクが登録されています。

config/recurring.yml
```ruby
production:
  clear_solid_queue_finished_jobs:
    command: "SolidQueue::Job.clear_finished_in_batches(sleep_between_batches: 0.3)"
    schedule: every hour at minute 12
```

スケジューラは`solid_queue_recurring`キューへジョブを入れます。各タスクはジョブのクラスまたはコマンドを指定します。既に書かれている先ほどのコードは、`SolidQueue::Job.clear_finished_in_batches(sleep_between_batches: 0.3)` コマンドを12分ごとに繰り返し実行する設定になっています。各タスクのschedule設定はfugit Gemの文法で書くことができ、fugitがcronとして受けつける全ての構文を書けます。

次のコード例はdevelopment環境で1秒ごとに `AsyncLogJob.perform_later(message: "from scheduled job")` を実行する設定です。

config/recurring.yml
```ruby
development:
  a_periodic_job:
    class: AsyncLogJob
    args: [{ message: "from scheduled job" }]
    schedule: every second
```

複数マシンでアプリケーションサーバを起動しているようなケースでは同じ設定で複数のスケジューラが実行されますが、ジョブが重複してキューに入らないような仕組みが用意されています。solid_queue_recurring_executions テーブルのレコードが、ジョブがキューに入れられるのと同じトランザクションで作成されます。このテーブルは`task_key`（タスク識別キー）と`run_at`（実行時刻）にユニーク制約インデックスを持っているので、同時刻の同一タスクについて1つのジョブしか作成されません。この仕組みは preserve_finished_jobs設定がtrue (デフォルト) のときに動作します。

スケジュール設定ファイルは1つだけなので、複数のスケジューラが同じスケジュールを使うことはできますが、複数のスケジューラで異なる設定を使うことはできません。

ジョブの定期実行についてより詳しくはSolid QueueのREADMEに説明されています。

## 参考資料

- Solid Queue https://github.com/rails/solid_queue
- Railsガイド Active Jobの基礎 https://railsguides.jp/active_job_basics.html
