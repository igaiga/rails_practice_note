---
title: "[ActiveRecord] selectとpluckの違い"
---

# [ActiveRecord] selectとpluckの違い

ActiveRecordのselectメソッドとpluckメソッドは次のように動きます。

- selectはActiveRecord::Relationを返し、to_aメソッドなどでSQLが発行され、ActiveRecordオブジェクトの配列が取れる
- pluckは呼び出すとすぐにSQL発行し、結果の値の配列が取れる
- `select(Arel.sql("... as foo"))`を実行すると、取得したActiveRecordオブジェクトにSQLの結果を取得するfooメソッド(attributes)がつくられる

どちらのメソッドもattributes(カラム)のシンボルを渡したり、SQLの一部を書くこともできます。

```ruby
irb> Book.pluck(:title)
#=> SELECT "books"."title" FROM "books"
=> ["Ruby"] # 結果の値が取れる

irb> Book.select(:title)
#=> SELECT "books"."title" FROM "books"
=> [#<Book:0x000000010a206628 id: nil, title: "Ruby">] # 結果を入れたActiveRecordオブジェクトが取れる
```

selectをつかう場合、取得できるのはActiveRecordオブジェクトなので、attributesにないものを取得しようとしても一見そのメソッドがないように見えます。SQLを書いてas some_nameを書くと、ActiveRecordがattributesとしてsome_nameメソッドをつくる機能があるので、これをつかって取得できます。

```ruby
irb> Book.pluck(Arel.sql("count(*)"))
#=> SELECT count(*) FROM "books"
=> [1]

irb> Book.select(Arel.sql("count(*) as count")).first
=> #<Book:0x0000000129e5b350 id: nil>

irb> Book.select(Arel.sql("count(*) as count")).first.count
#=> Book Load (0.5ms)  SELECT count(*) as count FROM "books" ORDER BY "books"."id" ASC LIMIT ?  [["LIMIT", 1]]
=> 1 # モデルに格納する場所がなさそうに見えるが、as countが実行されてcountメソッド(attributes)で取り出せる。

irb> Book.select(Arel.sql("count(*) as count")).first.attributes
=> {"count"=>1, "id"=>nil} # attributesとしてcountが追加されている

irb> Book.select(Arel.sql("count(*) as foo")).first.foo
=> 1 # as xxx の名前はなんでもOK
```

### サンプルコード

- https://github.com/igaiga/basics_of_active_record/tree/select_pluck
