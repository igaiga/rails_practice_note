---
title: "[ActiveRecord] トランザクションとロック"
---

# [ActiveRecord] トランザクションとロック

## トランザクション

DBへの複数の変更処理をアトミックにしたいときがあります。ここでの「アトミックにする」とは、「全て成功する」または「全て失敗する」とすることです。「一部だけ成功した状態を許さない」とも言えます。

アトミックにしたい処理は、たとえばXさんからYさんへお金を受け渡すようなケースです。「Xさんからお金を減じるDB処理」と「Yさんのお金を増やすDB処理」を行いますが、全て成功させるか、全て失敗させた場合は合計金額が釣り合います。全て失敗したケースでも、あとでもう1回同じ処理を実行することでリトライできます。アトミックな処理にすることで、データ不整合な状態をつくらないこと、状態が全成功または全失敗のどちらかなので現状を把握しやすいこと、失敗時にリトライできること、がメリットとして得られます。

一方でアトミックではない処理では、「Xさんからお金を減じるDB処理」だけが記録された「一部だけ成功した状態」をつくってしまうことがあります。合計金額が釣り合わない不整合な状態になることに加えて、どこまで成功してどのような状態になったの把握が難しいので、リトライの処理も複雑で面倒になります。

DBへの複数の変更処理をアトミックにしたいときには、ActiveRecord::Base.transactionメソッドをつかうことができます。2つのモデルAliceとCarolに対してDBへ記録を行うメソッドを実行したとき、「両方とも成功する(記録する)」または「両方とも失敗する(記録しない)」ようにするには、次のようにActiveRecord::Base.transactionメソッドへ渡すブロック中に処理を書きます。

```ruby
ActiveRecord::Base.transaction do
  Alice.pay!(100) # Aliceの所持金から100円減らす
  Carol.receive!(100) # Carolの所持金を100円増やす
end
```

このとき、「Alice.pay!(100)」と「Carol.receive!(100)」は両方記録するか、両方とも記録されないか、のどちらかになります。仮に、Carol.receive!(100)で失敗して例外が投げられたときは、既に実行したAlice.pay!(100)によるDB変更は巻き戻されて記録されません。この巻き戻し処理はロールバックと呼ばれます。反対に、成功してDBへ記録されることをコミットと呼びます。ここでは2つの処理をブロック中に書いてアトミックにしていますが、より多くの処理を書くこともできます。

途中でロールバックさせるためには、例外を投げてtransactionメソッドに渡したブロックを抜ける必要があります。returnで抜けるとロールバックされずにコミットされるので注意してください。

また、Alice.transactionとモデルをレシーバにしてtransactionメソッドを書くこともできます。ただし、特定のモデルをレシーバにしても、トランザクションの範囲はモデル単位ではなくDBコネクション単位になるので、Alice.transactionと書いてもAlice以外のモデルも含めてトランザクションが有効になります。単一DBへ接続するアプリではActiveRecord::Base.transactionもModel.transactionも同じ動作になります。たとえば、複数DBへ接続するアプリでは、Model.transactionと書くメリットがあります。そのモデルがつかうDBコネクションに対してトランザクションが作成されるためです。

同一処理で複数のデータが一緒に保存されるケースなどデータの整合性を保ちたいときは、トランザクションをうまく活用してアトミックな処理にすることを検討してみてください。

トランザクションについて詳しくは次のページも参考になります。

- Rails APIリファレンス "Active Record Transactions": https://api.rubyonrails.org/classes/ActiveRecord/Transactions/ClassMethods.html
- Rails APIリファレンス "Active Record Transactions" 日本語訳: https://techracho.bpsinc.jp/hachi8833/2022_12_09/101160

## トランザクションisolation(分離)レベル

トランザクション中で更新したレコードを、他の処理が読みだそうとしたときの動作は、DBのisolation(分離)レベル設定によって異なります。isolationレベルはたとえば次のコードで設定できます

```ruby
ActiveRecord::Base.transaction(isolation: :serializable) do
end
```

このサンプルコードはisolationレベルを最も強く分離するSERIALIZABLEレベルで設定しています。最も安全にデータを操作できますが、そのかわり、他のDBコネクションによる処理によって待たされるケースも最も多いです。

isolationレベルのデフォルトはMySQL(InnoDB)ではREPEATABLE READ、PostgreSQLではREAD COMMITTEDです。isolationレベルの動作詳細はDBごとに差異もあります。

- MySQL: https://dev.mysql.com/doc/refman/8.0/ja/set-transaction.html
- PostgreSQL: https://www.postgresql.jp/document/14/html/transaction-iso.html

## ロック

ここでは悲観的(pessimistic)ロックである行ロックについて説明します。

行ロックはテーブルのレコードごとにロックを取ります。ロックがかかっている間は、他の処理がそのレコードのロックを取得できなくなります。

DBからレコードを取得して、その行で行ロック取得するときのコードは次のようになります。

```ruby
ActiveRecord::Base.transaction do
  User.lock.find(user_id) # DBからレコード取得＆そのレコードで行ロック取得
end # transactionブロックを抜けるとロック解放
```

```
TRANSACTION (0.2ms)  BEGIN
User Load (0.3ms)  SELECT `users`.* FROM `users` WHERE `users`.`id` = 1 LIMIT 1 FOR UPDATE
TRANSACTION (0.1ms)  COMMIT
```

ロックはトランザクション中で有効なので、ロックを取得するときは `ActiveRecord::Base.transaction do end` の中で `lock` メソッドをつかいます。トランザクションを抜けるとロックは解放されます。発行されるSQLは通常のSELECT文の末尾にFOR UPDATEがついた SELECT FOR UPDATE です。また、ここではfindメソッドをつかっていますが、たとえばfind_byなど、他の取得系メソッドでも同様につかえます。

既にDBから取得済みのモデルオブジェクトからレコードの行ロックを取得するときは `lock!` メソッドをつかいます。

```ruby
user = User.find(user_id)
ActiveRecord::Base.transaction do
  user.lock! # 既にDBから取得済みのモデルオブジェクトのレコードの行ロックを取得
  # 処理
end #transactionブロックを抜けるとロック解除
```

または `with_lock` メソッドにブロックを渡す方法もあります。この書き方ではトランザクション作成とロックの取得を同時にできます。さきほどのコードをwith_lockで書き直すと次のようになります。

```ruby
user = User.find(user_id)
user.with_lock do # トランザクション作成＆ロック取得
  # 処理
end
```

ロックをつかうと、ロックを取得した1つの実行体だけが該当の処理を実行するコードを書くことができます。Railsアプリでの実行体とは、DBコネクションを1つつかって処理を進めるRubyのプロセスやスレッドです。Railsのリクエストを処理するPumaのワーカーや、非同期ジョブを実行するワーカーなどが該当します。

ロックをつかうときには、デッドロックに注意する必要があります。デッドロックとは、複数の実行体が相手の持っているロックの解放を待っていて処理が進まなくなる状態です。Railsではタイムアウト設定がされているので、いずれかの実行体がタイムアウトしてロックを開放するまで、双方が停止してしまいます。デッドロックを防ぐ万能な方法を説明することは難しいですが、「ロックを取る作業の実行時間は短くする」「1つのトランザクション中で2つ以上のロックを取らない」「どうしても1つのトランザクション中で2つ以上のロックを取りたいときは、全てのコードで取る順番を同じにする」へ気をつかうだけでも効果は得られます。

ロックについて詳しくは、次のページも参考になります。

- Rails API https://api.rubyonrails.org/classes/ActiveRecord/Locking/Pessimistic.html
- Railsガイド "Active Record クエリインターフェイス - レコードを更新できないようロックする - 悲観的ロック（pessimistic）": https://railsguides.jp/active_record_querying.html
