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

