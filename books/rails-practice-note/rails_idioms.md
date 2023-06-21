---
title: "[Rails基礎] Railsコードの慣用句"
---

# Railsコードの慣用句

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

これでも良いのですが、RubyのRangeオブジェクトをつかうとプレースホルダをつかわず、キーワード引数のスタイルで読みやすく書くことができます。

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
