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
