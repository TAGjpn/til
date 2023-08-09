# Turbo Streamでコメントの投稿・削除機能をajax化

# コメント投稿後に非同期で表示されるまでの流れ
## 1.createアクションが実行される
コメントフォームからPOSTリクエストが送信されると、createアクションが呼び出される。
```
  def create
    @comment = current_user.comments.build(comment_params)
    @comment.save
  end
```
createアクションでは以下の処理が行われる。  
- `comment_params`で取得したデータを元に、新たにコメントオブジェクトを生成する
- そのオブジェクトをログイン中のユーザーのコメントコレクションに追加し、`@comment`に格納する
- `@comment`をデータベースに保存する
- `render`や`redirect_to`の指定がないので、`create.turbo_stream.erb`がレンダリングされる

## 2.`create.turbo_stream.erb`でエラーによる条件分岐をする
createアクションで`@comment.save`をした際に、エラーが発生するとそのエラーメッセージも一緒に`@comment`に格納される。  
`if @comment.errors.present?`では、`@comment`の中のエラーの有無で条件分岐をする

エラーが発生している場合は`true`の処理、発生していない場合は`false`の処理が実行される。
```
#create.turbo_stream.erb

<% if @comment.errors.present? %>
  #エラー発生時の処理
<% else %>
  #エラーが無い時の処理
<% end %>
```
### エラー発生時の処理(true)
新たにコメントフォームをレンダリングし、非同期で置き換える。  
そうすることで、入力内容はリセットせずにバリデーションエラーメッセージを表示させることができる。

ページをリロードするとバリデーションエラーメッセージが消えるのは、新しいリクエストが発生すると、前回のリクエスト時にインスタンス変数`@comment`に格納された情報はクリアされるため。
```
<%= turbo_stream.replace 'comment-form' do %>
  <%= render 'comments/form', comment: @comment, board: @comment.board %>
<% end %>
```
### エラーが無い時の処理(false)
#### 投稿したコメントをコメント一覧の一番上に表示する
`turbo_stream.prepend`で`table-comment`の先頭に非同期的に追加することができる。  
インスタンス変数`@comment`からローカル変数`comment`へ値を渡しパーシャルでコメント内容を表示する。

#### コメントフォームを初期化する
`turbo_stream.replace`で`comment-form`を非同期的に置き換える。  
`Comment.new`でコメントの内容を初期化し、`board: @comment.board`で掲示板の情報をパーシャルへ渡し、フォームをレンダリングする。

`@comment.board`を渡さないと非同期通信でフォームを再表示した後に、コメントを再投稿してもどの掲示板に対するコメントか特定できなくなる。
```
<%= turbo_stream.prepend 'table-comment' do %>
  <%= render 'comments/comment', comment: @comment %>
<% end %>
<%= turbo_stream.replace 'comment-form' do %>
  <%= render 'comments/form', comment: Comment.new, board: @comment.board %>
<% end %>
```
