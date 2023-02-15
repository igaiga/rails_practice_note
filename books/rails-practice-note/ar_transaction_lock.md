---
title: "[ActiveRecord] トランザクションとロック"
---

# [ActiveRecord] トランザクションとロック

## トランザクション

DBへの複数の変更処理をアトミックにしたいときがあります。ここでの「アトミックにする」とは、「全て成功する」または「全て失敗する」とすることです。「一部だけ成功した状態を許さない」とも言えます。

アトミックにしたい処理は、たとえばXさんからYさんへお金を受け渡すようなケースです。「Xさんからお金を減じるDB処理」と「Yさんのお金を増やすDB処理」を行いますが、全て成功させるか、全て失敗させた場合は合計金額が釣り合います。全て失敗したケースでも、あとでもう1回同じ処理を実行することでリトライできます。もしも「Xさんからお金を減じるDB処理」だけが記録された「一部だけ成功した状態」をつくってしまうと、合計金額が釣り合わない不整合な状態になることに加えて、どこまで成功してどのような状態になったの把握が難しいので、リトライの処理も複雑で面倒になります。

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

## トランザクションisolation(分離)レベル

トランザクション中で更新したレコードを、他の処理が読みだそうとしたときの動作は、DBのisolationレベル設定によって異なります。isolationレベルはたとえば以下のコードで設定できます

```ruby
ActiveRecord::Base.transaction(isolation: :serializable) do
end
```

前述のサンプルコードはisolationレベルを最も強く分離するSERIALIZABLEレベルで設定しています。最も安全にデータを操作できますが、そのかわり、他のDBコネクションによる処理によって待たされるケースも最も多いです。

isolationレベルのデフォルトはMySQL(InnoDB)はREPEATABLE READ、PostgreSQLはREAD COMMITTEDです。isolationレベルの動作詳細はDBごとに差異もあります。

MySQL: https://dev.mysql.com/doc/refman/8.0/ja/set-transaction.html
PostgreSQL: https://www.postgresql.jp/document/14/html/transaction-iso.html

## 参考資料

- Rails APIリファレンス "Active Record Transactions": https://api.rubyonrails.org/classes/ActiveRecord/Transactions/ClassMethods.html
- Rails APIリファレンス "Active Record Transactions" 日本語訳: https://techracho.bpsinc.jp/hachi8833/2022_12_09/101160
