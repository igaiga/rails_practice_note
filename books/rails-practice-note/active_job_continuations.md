---
title: "[ActiveJob] Continuations"
---

# Active Job Continuations

Rails8.1で追加されたActive Job Continuations機能により、ジョブの中断と再開が可能になりました。ジョブを複数のステップで定義しておき、ジョブが中断されたときはどのステップが完了したかの進捗を保存し、再開時に続きから実行することができます。

## ステップ機能

次のコード例でステップの定義方法と、それらがどのように実行されるかを確認してみます。このコードではまだ中断と再開は行われず、正常に完了します。

app/jobs/something_job.rb Continuationsのステップ定義例
```ruby
class SomethingJob < ApplicationJob
  include ActiveJob::Continuable

  def perform
    # ステップに属さないコードは初回も再開時も常に実行される
    puts "=== Start/Resume Job"

    # ステップをブロックで定義
    step :first do |step|
      # ブロック引数を書くとActiveJob::Continuation::Stepオブジェクトが渡ってくる
      puts "=== 1st step"
    end

    puts "=== Middle point"

    # メソッドを呼び出すステップを定義
    step :second

    puts "=== Finish Job"
  end

  private

  def second(step)
    # 引数を書くとActiveJob::Continuation::Stepオブジェクトが渡ってくる
    puts "=== 2nd step"
  end
end
```

Continuationsのステップ定義例の実行結果
```
=== Start/Resume Job
=== 1st step
=== Middle point
=== 2nd step
=== Finish Job
```

Continuations機能を使うときは、ActiveJob::Continuableモジュールをincludeします。Continuationsの進捗を記録するための道具として、ステップとカーソルが用意されています。performメソッド中にstepメソッドをつかって各ステップを定義します。ステップおよびステップに属さないコードのどちらも書いた順に実行されます。ステップに属さないコードは、初回も再開時も毎回実行されます。

stepメソッドにはブロックまたはメソッド名を渡すことができます。ブロックを渡すときは、ブロック引数を書くとStepオブジェクト(ActiveJob::Continuation::Step)を受け取ることができます。stepメソッドへシンボルを渡すと、シンボル名のメソッドを実行します。実行されるメソッドに引数を1つ書いておくと、Stepオブジェクトを受け取ることができます。

次は、中断して再開するコードを実行してみましょう。2番目のステップであるsecondメソッドを次のように書き換えてジョブ実行を中断させます。`step.resumed?` は再開実行されているかどうかを判断するメソッドで、これを使って初回のみジョブを中断させる例外を投げるコードを書いています。

app/jobs/something_job.rb 2ndステップで中断させる例
```ruby
def second(step)
  unless step.resumed? # resumed?: 再開実行かどうか
    raise ActiveJob::Continuation::Interrupt.new
  end
  puts "=== 2nd step"
end
```

2ndステップで中断させる例の実行結果
```
=== Start/Resume Job
=== 1st step
=== Middle point
(略) # 5秒待ってからジョブが再開する
=== Start/Resume Job
=== Middle point
=== 2nd step
=== Finish Job
```

`ActiveJob::Continuation::Interrupt` 例外を投げると中断され、進捗情報を記録したジョブがキューに保存されます。デフォルト設定では5秒待つとジョブが再開されます。再開時のジョブ実行では、既に完了したステップ(ここでは1st step)は実行されません。

## カーソル機能

ステップの中でさらに詳細に進捗を記録したいときはカーソル機能を利用できます。

カーソル機能
```ruby
step :first do |step|
  p step.cursor #=> nil
  step.set! (step.cursor || 0) + 1
  p step.cursor #=> 1
  step.advance!
  p step.cursor #=> 2
(略)
```

Step#cursorメソッドで現在のカーソル値を取得し、Step#set!メソッドでカーソルに値を代入できます。カーソル値はsuccメソッドを持つオブジェクトにします。Step#advance!メソッドはカーソル値にsuccメソッドを呼び出してカーソル値を進めます。初回実行時のカーソル初期値はnilですが、ステップ定義に `start:` 引数を書くことで初期値を設定できます。

カーソルの初期値を設定
```ruby
step :first, start: 1 do |step|
  p step.cursor #=> 1
```

再開時のカーソル値のデフォルトは、前回実行時に中断したステップの最後の値が入っています。次のコードで確認してみましょう。

初回時と再開時のカーソル値

```ruby
step :first do |step|
  p "=== Start step"
  p "=== step.cursor #=> #{step.cursor}"
  step.set! (step.cursor || 0) + 1
  p "=== step.cursor #=> #{step.cursor}"
  raise ActiveJob::Continuation::Interrupt.new unless step.resumed?
end
```

実行結果
```
"=== Start step" # 初回
"=== step.cursor #=> " # デフォルトはnil
"=== step.cursor #=> 1"
"=== Start step" # 再開
"=== step.cursor #=> 1" # 再開時カーソル値は前回終了時の値
"=== step.cursor #=> 2"
```

カーソル機能をつかうと、配列の各要素で処理を行うようなジョブで、再開時に前回の続きから作業させることができます。

カーソル機能を使って配列の各要素を処理する
```ruby
step :first, start: 0 do |step| # カーソル初期値に0を設定
  items[step.cursor..].each do |item|
    # 再開時は、step.cursorに前回終了時の値が入っている
    item.do_something
    step.advance!
  end
end
```

`items[step.cursor..]` は初回はstep.cursorが0なのでitems配列の先頭から処理をします。再開時は`step.cursor`に前回終了時の値が入っているので、items配列の未作業の要素から処理を再開します。

## Continuationsの設定

ジョブクラスにContinuationsに関する設定を書くことができます。最大再開回数や再開までの時間を設定するときは、ジョブクラスに次のように書きます。

Continuationsに関する設定
```ruby
class SomethingJob < ApplicationJob
  self.max_resumptions = 3  # 最大再開回数。デフォルトはnil(無制限に再開)。
  self.resume_options = { wait: 2.seconds, queue: :resumed }
  # wait: 再開までの時間。デフォルトは5.seconds。
  # queue: 中断時に入れるキュー。デフォルトは初回と同じ。
```

max_resumptionsは最大再開回数を設定します。デフォルトはnilで、無制限に再開可能です。resume_optionsは再開までの時間（デフォルトは5秒）や再開時ジョブを入れるキュー（デフォルトは初回と同じキュー）などを設定できます。設定の詳細な説明はActive Job Continuation( https://api.rubyonrails.org/classes/ActiveJob/Continuation.html )およびretry_jobメソッド( https://api.rubyonrails.org/classes/ActiveJob/Exceptions.html#method-i-retry_job )のAPIページに書いてあります。

## 中断方法とチェックポイント

前述のサンプルコードではActiveJob::Continuation::Interrupt例外を明示的に投げるコードを書きましたが、実際はジョブがこの例外を投げます。チェックポイントにて、ジョブが `queue_adapter.stopping?` メソッドを呼び出して中断するか判断します。結果がtrueのとき、ジョブは ActiveJob::Continuation::Interrupt 例外を発生させて中断作業をします。キューアダプタに停止マークが付いた瞬間（`queue_adapter.stopping?` がtrueになったとき）にジョブが中断するのではなく、チェックポイントで `queue_adapter.stopping?` の結果を判断されるまで、またはプロセスが停止するまで、ジョブは実行を続けます。

チェックポイントは各ステップ開始前（初回ステップを除く）に置かれるほか、ステップ内では`set!`メソッド、`advance!`メソッドなどが呼び出されたときにも置かれます。また、`checkpoint!`メソッドで明示的にチェックポイントを置くこともできます。

中断作業によって、進捗情報はジョブデータ内のcontinuationキーにシリアライズされジョブとしてキューに戻ります。進捗情報には完了したステップのリストと、進行中のステップがある場合はそのステップとカーソルの情報が記録されています。進捗情報など、シリアライズされたデータはミッションコントロールを使うとジョブの情報から見ることができます。

![Mission Control スクリーンショット](/images/rails_practice_note/active_job_continuations/mission_controll_ss_somethingjob.png)

通常、ジョブが例外などを処理してエラー終了したときはFailed Jobとしてキューに戻されますが、進捗情報があるのに記録されず失われるのは不便です。これを防ぐために、ジョブがステップ完了するか、カーソルを進めたあとでエラーが発生したときは、再開させるジョブとして進捗情報を記録してキューに保存します。

また、ステップ定義時に `step :some_step, isolated: true` ようにisolatedオプションを追加しておくと、そのステップの開始前に必ず進捗情報を記録してキューに保存してからジョブが再開されます。isolatedオプションはチェックポイント間隔を短く設定できないような長いジョブのときに有用です。

## 参考資料

- Railsガイド Active Jobの基礎 https://railsguides.jp/active_job_basics.html
- RailsAPI Active Job Continuation https://api.rubyonrails.org/classes/ActiveJob/Continuation.html
- RailsAPI Active Job Continuation Step https://api.rubyonrails.org/classes/ActiveJob/Continuation/Step.html
