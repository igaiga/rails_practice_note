---
title: "[Rails基礎] Railsの知識"
---

# Railsの知識

## blank?, present?, empty?

`blank?` メソッドは、レシーバが次の値のときにtrueを返します。

`false`, `nil`, `""`, `" "`, `[]`, `{}`

falseとnil、空っぽいオブジェクトがtrueになります。

`present?` メソッドは `blank?` メソッドの真偽反転版です。

`blank?` は便利なので多用しがちですが、`""`, `[]`, `{}` かどうかを確かめるときは `empty?` メソッドをつかう方がコードが読みやすくなります。`false`, `nil` もレシーバにくるのだろうかと考える必要がなくなり、考える範囲を狭くすることができるからです。

## presence

`presence` メソッドは、レシーバが `present?` に対してtrueになるときはレシーバを返し、それ以外ではnilを返します。次のようなコードでつかわれます。

```ruby
a = 1
result = a.presence || 5
p result #=> 1

a = false
result = a.presence || 5
p result #=> 5
```

`object.presence` メソッドは `object.present? ? object : nil` と同じ意味になります。

## Time.zone.now, Time.current

Railsで現在時刻を取得するには `Time.zone.now` または `Time.current` をつかいます。どちらも同じ結果になります。これらは `ActiveSupport::TimeWithZone` クラスのインスタンスです。Railsアプリ設定のタイムゾーン(`Rails.application.config.time_zone`)を反映した日時情報を持ちます。同様に、文字列から日時を作成するときは `Time.zone.parse` メソッドをつかいます。

一方で、 `Time.now` (`Time`クラス)や `DateTime.now` (`DateTime`クラス)はRailsでは特別な理由がなければつかわないようにします。これらはOSのTZ設定を反映した日時情報になるため、Railsアプリ設定と異なるタイムゾーンを持つ場合があるからです。

同様の理由で、今日の日付を取るときは `Time.zone.today`のように、`TimeWithZone`オブジェクトを最初に作成してからtodayメソッドを呼ぶと、Railsアプリのタイムゾーン設定をつかって取得できます。 `Date.today` をつかうと、OSのTZ設定を反映した今日が取得されてしまうので、Railsアプリでつかうことはほぼないはずです。

## ActiveRecord whereメソッドの不等号判定にRangeオブジェクトをつかう

モデルオブジェクトで、次のようなSQL where句をつかって条件を追加したいケースを考えます。

```sql
WHERE age >= 13 and age <= 19
```

ActiveRecordのプレースホルダをつかって書くと次のようになります。

```ruby
where("age >= ?", 13).where("age <= ?", 19)
# or
where("age >= :min and page <= :max", min: 13, max: 19)
```

これでも良いのですが、RubyのRangeオブジェクトをつかうとプレースホルダをつかわず、キーワード引数のスタイルで読みやすく書くことができます。また、joinによって複数テーブルのカラムが対象になるときに、キーワード引数のスタイルで書いておくとActiveRecordが自動でテーブル名を指定するため、曖昧なケースを回避できるメリットもあります。

```ruby
where(age: 13..19)
```

`13..19` がRangeオブジェクトで「13以上、19以下（13 <= x <= 19）」を表します。

`..`は終端を含むRangeオブジェクトになります。終端を含まないRangeオブジェクトをつくるときは`...`をつかいます。`13...19`は「13以上、19未満（13 <= x < 19）」を表します。

`where("age >= ?", 13)` もRangeオブジェクトをつかって書くことができます。

```ruby
where(age: 13..)
```

Rangeオブジェクト`13..`は「13以上（13 <= x）」を表します。このようなRangeオブジェクトはendless Rangeオブジェクトとも呼ばれ、Ruby2.6で書けるようになりました。


`where("age <= ?", 19)` もRangeオブジェクトをつかって書くことができます。

```ruby
where(age: ..19)
```

Rangeオブジェクト`..19`は「19以下（x <= 19）」を表します。このようなRangeオブジェクトはbeginless Rangeオブジェクトとも呼ばれ、Ruby2.7で書けるようになりました。

endless Rangeオブジェクトとbeginless Rangeオブジェクトをつかうと不等号の4パターンのうち3パターンを書くことができますが、始端を含まないパターンだけは対応するRangeオブジェクトがないので書けません。プレースホルダをつかって書くか、または、始端を含む形に変えてRangeオブジェクトをつかって書いても良いでしょう。

```ruby
# age <= 19
where(age: ..19)

# age < 19
where(age: ...19)

# age >= 13
where(age: 13..)

# age > 13
where("age > ?", 13)
# または整数ならば次のようにも書ける
where(age: 14..)
```

整数だけでなく、TimeWithZone型のような日時型でもつかえます。たとえば、registered_atが先月の始めから終わりまでに入っているかどうかは次のコードで調べることができます。

```ruby
where(registered_at: Time.zone.now.last_month.beginning_of_month..Time.zone.now.last_month.end_of_month)
# where("registered_at >= ?", Time.zone.now.last_month.beginning_of_month).where("registered_at <= ?", Time.zone.now.last_month.end_of_month). と同じ
```

また、beginning_of_monthからend_of_monthまででつくられるRangeオブジェクトはall_monthメソッドでもつくることができます。

```ruby
Time.zone.now.last_month.all_month
#=>
# Thu, 01 Jun 2023 00:00:00.000000000 UTC +00:00
# ..
# Fri, 30 Jun 2023 23:59:59.999999999 UTC +00:00

# 以下と同じ

Time.zone.now.last_month.beginning_of_month..Time.zone.now.last_month.end_of_month
#=>
# Thu, 01 Jun 2023 00:00:00.000000000 UTC +00:00
# ..
# Fri, 30 Jun 2023 23:59:59.999999999 UTC +00:00
```

## 範囲内に含まれるかどうかをRangeオブジェクトをつかって判定する

変数ageが13以上19以下かどうかを調べる次のようなコードがあるとします。

```ruby
puts "teenage" if 13 <= age && age <= 19
```

Rangeオブジェクトと`include?`メソッドをつかうと次のように書くことができます。

```ruby
puts "teenage" if (13..19).include?(age)
```

include?メソッドがつかわれることにより「範囲内に含まれるかどうか」の意味をコードで表現することができます。

また、変数ageをレシーバにして`in?`メソッドで書くことで、ageを主語として表現することもできます。include?メソッドはRubyで定義されているメソッドで、in?メソッドはRailsで定義されているメソッドです。

```ruby
puts "teenage" if age.in?(13..19)
```

include?メソッドやin?メソッドは整数オブジェクトだけでなく、日時オブジェクトほかRangeオブジェクトをつくれるものに対してつかえます。

## シリアライズ、デシリアライズされるところへ入れるオブジェクトに注意しよう

ジョブキュー、セッション、キャッシュなど、オブジェクトを格納するとシリアライズ、デシリアライズされるところがあります。このような場所では、シリアライズ、デシリアライズ可能なオブジェクトを選ぶ必要があります。シリアライズ、デシリアライズを安全に行えないオブジェクトを入れると、格納時や取り出し時に問題が起こることがあります。特に、格納時と取り出し時のRubyやRailsのバージョンが異なるときにシリアライズ、デシリアライズ方法が変わったことで問題が起こることがあり、RubyやRailsをバージョンアップ作業をしたときにバグとして表出するので気づきづらく、注意が必要です。

多くのケースで安全にシリアライズ、デシリアライズ可能なオブジェクトはStringやInteger、ArrayやHashなどです。ArrayやHashは中に入っているオブジェクトもシリアライズ、デシリアライズ可能である必要があります。

モデルであるActiveRecordオブジェクトは、そのままではシリアライズ、デシリアライズ時に問題が起こることがあるので、to_global_idメソッドをつかってGlobalIDオブジェクトへ変換することで安全にシリアライズ、デシリアライズをすることができます。GlobalIDオブジェクトはモデルオブジェクトをアプリ内で一意に識別するURIを生成します。格納時、GlobalIDオブジェクトに対してシリアライズ処理を行うと、URI文字列（String）オブジェクトとして格納されます。このURI文字列はto_sメソッドで確認することができます。

```ruby
gid = Book.first.to_global_id
#=> #<GlobalID:0x0000000106fc3558 @uri=#<URI::GID gid://books-app/Book/1>>
gid.to_s
#=> "gid://books-app/Book/1"
```

GlobalIDオブジェクトおよびそのURI文字列からActiveRecordオブジェクトへ復元するときは`GlobalID::Locator.locate`メソッドをつかいます。このメソッドを呼び出したときにGlobalID URIから該当モデルとそのidを情報を取り出し、DBへselectクエリを実行してActiveRecordオブジェクトへと復元されます。

```ruby
gid_string = Book.first.to_global_id.to_s
#=> #<GlobalID:0x00000001085cfe00 @uri=#<URI::GID gid://books-app/Book/1>>

GlobalID::Locator.locate(gid_string)
# Book Load (0.2ms)  SELECT "books".* FROM "books" WHERE "books"."id" = ? LIMIT ?  [["id", 1], ["LIMIT", 1]]
#=> #<Book:0x00000001080ad170
# id: 1,
# title: "Ruby超入門",
# memo: "Rubyの入門書",
# created_at: Tue, 20 Jun 2023 06:45:44.419445000 UTC +00:00,
# updated_at: Tue, 20 Jun 2023 06:45:44.419445000 UTC +00:00>
```

まとめると、ActiveRecordオブジェクトを格納するときはto_global_idメソッドをつかってGlobalIDオブジェクトへ変換する、取り出すときは`GlobalID::Locator.locate`メソッドをつかってGlobalIDオブジェクトからActiveRecordオブジェクトへ復元する、という手順を踏みます。特定のケース、たとえばActiveJobのキューへジョブを入れるときなどは、モデルオブジェクトからGlobalIDオブジェクトへの自動変換、自動復元を行う機能が用意されています。

GlobalIDはGemになっていて、READMEページに詳細な解説があります。

- https://github.com/rails/globalid

## Model.findよりもcurrent_user.relationをつかう

ControllerでModel.findでDBからレコードを得るケースでは、状況によってはcurrent_user.relation.findで取得する方が好ましいです。

前提として、次のような状況だとします。

- current_userでログインユーザーを取得できる
- UserとBookが1対多の関係
- BookはUserにより所有されていて、User#owned_books scopeで自分の所有する本を取得できる
- Book#owned_by?(user) でuserが所有する本かどうか調べることができる
- 他のUserが所有する本を閲覧することはできない

BooksController#show では次のようにBookを取得できますが、より良い書き方があります。

良くないコード例

```ruby
@book = Book.find(params[:id])
raise ActiveRecord::RecordNotFound.new unless @book.owned_by?(current_user)
```

より良いコード例

```ruby
@book = current_user.owned_books.find(params[:id])
```

良い書き方であれば、自分(ログインユーザー)の所有する本に限定して検索できるので、表示してはいけない本を代入するタイミングをなくせます。また、current_userから辿るコードを主流派にしておけば、コードレビューのときに権限の観点でチェックすることがかんたんになります。権限を越えて取得できてしまう問題は大きな事故につながることが多いため、より安全なコードを書いておく価値があります。

## saveとsave!の組など、メソッドの末尾が!有無の組のつかいわけ

ActiveRecordにはsaveメソッドとsave!メソッドの組のように、メソッド名末尾!の有無で成功時および失敗時の動きが変わるメソッドがあります。saveメソッドは（たとえばバリデーションエラーなどで）失敗したときにfalseを返します。save!メソッドは失敗したときに例外を投げます。（別のケースとして破壊的変更を行う意図で末尾に!がつくメソッドもありますが、ここではそれらは考えません。）

saveとsave!の組のつかいわけは、**「ほぼ失敗しないときはsave!メソッドをつかい、失敗が想定されるときはsaveメソッドをつかう」** という判断がおすすめです。失敗が想定されるケースでよくあるのはバリデーションエラーです。また、「ほぼ失敗しない」は「99％失敗しない」くらいを想定しています。

saveメソッドとsave!メソッドの典型的なコードは次のようになります。

saveメソッド

```ruby
book = Book.new(title: "Ruby超入門")
if book.save # たとえばバリデーションエラーで失敗するかもしれない
  # 成功時の処理
else
  # 失敗時の処理
end
```

save!メソッド

```ruby
book = Book.new(title: "Ruby超入門")
book.save! # ほぼ失敗しない
# 成功時の処理
# 万が一失敗したときは、アプリ全体など汎用的な例外処理に任せる
```

少し視点を変えて、失敗時のコードを近くに書きたいかどうかで考えても良いでしょう。「失敗時のコードをここに書きたければsaveメソッド、失敗時のコードは書かずにもしも例外が発生したらアプリ全体向けなどの汎用的な例外処理に任せたければsave!メソッドをつかう」と考えることもできます。失敗したときに特別な処理を書きたければsaveメソッドをつかって戻り値を見ることになりますし、失敗がほぼ起きないケースでは失敗時のコードを特別に書く必要がないので例外を投げるsave!メソッドをつかうことになります。逆に、save!メソッドをつかうときに近くで例外をrescueして失敗時の処理を書くことは、コードが読みづらくなることが多いので避けた方が良いでしょう。

また、saveメソッドで戻り値を確認しないコードを書くと、失敗時に気づけないバグになることがあるので注意が必要です。saveメソッドをつかうならば戻り値を必ず確認しましょう。

saveメソッドの良くないコード例

```ruby
book = Book.new(title: "Ruby超入門")
book.save # 戻り値を見ていないので、失敗時も成功時の処理をしてしまう
# 成功時の処理
```

メソッドの末尾が!有無の組にはsaveとsave!メソッドのほか、createとcreate!メソッド、destroyとdestroy!メソッドなどもあります。これらのメソッドの組も同様に考えることができます。また、削除するときはほとんどのケースで失敗することを想定しないので、destroyメソッドではなくdestroy!メソッドをつかうのが良いでしょう。
