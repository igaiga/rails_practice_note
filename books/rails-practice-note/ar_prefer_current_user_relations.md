---
title: "[ActiveRecord] Model.findよりもcurrent_user.relation.findをつかう"
---

# Model.findよりもcurrent_user.relationをつかう

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

良い書き方であれば、自分(ログインユーザー)の所有する本に限定して検索できるので、表示してはいけない本を代入するタイミングをなくせます。また、current_userから辿るコードを主流派にしておけば、コードレビュー時に権限の観点でチェックすることがかんたんになります。権限を越えて取得できてしまう問題は大きな事故につながることが多いため、より安全なコードを書いておく価値があります。
