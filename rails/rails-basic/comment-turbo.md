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
creatアクションでは以下の処理が行われる。  
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
  エラー発生時の処理
<% else %>
  エラーが無い時の処理
<% end %>
```
### エラー発生時の処理(true)
新たにコメントフォームをレンダリングし、非同期で置き換える。  
そうすることで、入力内容はリセットせずにバリデーションエラーメッセージを表示させることができる。

ページをリロードするとバリデーションエラーメッセージが消えるのは、`@comment`に格納されたデータはリクエストが終了すると破棄されるため。
```
<%= turbo_stream.replace 'comment-form' do %>
  <%= render 'comments/form', comment: @comment, board: @comment.board %>
<% end %>
```
### エラーが無い時の処理(false)

```
<%= turbo_stream.prepend 'table-comment' do %>
  <%= render 'comments/comment', comment: @comment %>
<% end %>
<%= turbo_stream.replace 'comment-form' do %>
  <%= render 'comments/form', comment: Comment.new, board: @comment.board %>
<% end %>
```
