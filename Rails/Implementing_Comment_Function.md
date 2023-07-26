# コメント機能の実装について

## このページで解説する内容について  
RUNTEQカリキュラムのRails基礎11で学んだ内容を復習としてまとめたものです。  
主な内容としては、掲示板詳細ページにコメント機能を実装する手順についてです。

# コメントモデルの作成
## 1.UserモデルとBoardモデルを関連付けしてCommentモデルを作成する
```
#bodyという名前のtext型のカラムを持つCommentモデルを作成するためのコマンド

$ rails g model comment body:text user:references board:references
```
`user:references`：CommentモデルがUserモデルに対して外部キーを持つ様に指定できる。  
この指定により、UserモデルとCommentモデルの間で１つのユーザーが複数のコメントを持つ「一対多」の関係が作られる。  
モデル作成時に`references`を使用することで、Commentモデルに`belongs_to`の記述も行われる。

## 2.マイグレーションファイル
```
class CreateComments < ActiveRecord::Migration[7.0]
  def change
    create_table :comments do |t|
      t.references :user, null: false, foreign_key: true
      t.references :board, null: false, foreign_key: true
      t.text :body

      t.timestamps
    end
  end
end
```
モデル作成時に`references`を使用したことにより`foreign_key: true`が自動で設定される。

## 3.バリデーションの追加
Commentモデルにコメント用のバリデーションを追加する
```
class Comment < ApplicationRecord
  belongs_to :user
  belongs_to :board

  validates :body, presence: true, length: { maximum: 65_535 }
end
```
`presence: true`：bodyがnilや空の場合に無効にするためのバリデーション　  
`length: { maximum: 65_535}`：bodyが最大65,535文字までになるよう制限をするためのバリデーション

## 4.マイグレーションの実行
```
$ rails db:migrate
```

## 5.BoardモデルとUserモデルとの関連付け
BoardモデルとUserモデルに対して、１対多の関係であることをモデルに記述。
```
#app/models/board.rb と app/models/user.rb

has_many :comments, dependent: :destroy
```
`dependent: :destroy`：親モデル(UserまたはBoard)が削除されたら、該当する子モデル(Comment)も自動で削除をするための記述。

# コントローラの生成とルーティングの設定
## Commentsコントローラを生成する
```
$ rails g controller comments
```

## boardsにネストさせてcommentsのルーティングを設定する
掲示板詳細ページの中のコメントのように、１つのリソースが別のリソースに属しているときは、ルーティングをネストさせる。
```
#routes.rb

resources :boards, only: %i[index new create show] do
  resources :comments, only: %i[create edit destroy], shallow: true
end
```
### ルーティングをネストさせる理由
- リソースの関係性  
  ルーティングをネストさせることで、URLから直接リソースの関係性がわかりやすくなる。  
  例）`　/boards/1/comments/2`のようにidが２のコメントがidが１の掲示板に所属していると直感的にわかる
- 親リソースIDによるフィルタリング  
  コントローラのアクション内で親リソースのidを利用できる様になる。  
  特定の掲示板に対するコメントだけを取得したり、新規コメントをその掲示板に紐づけるといった操作が簡単になる。
-セキュリティ強化  
  ユーザーがその掲示板の所有者かどうか判定して、その掲示板に対するコメントの作成を許可したり制限したりするアクセス制御を行うことができるようになる。

### shallow: trueとは
ネストされたルーティングを浅くするためのルーティングの設定オプション。  
子リソースの一部のアクション（通常は　show, edit, update, destroy)については、URLに親リソースのIDを含めない様にする。  
このオプションの利点は、URLがシンプルになることと、親リソースのIDが不要なアクションではそのIDを取得する必要がなくなること。

# コントローラの設定
## Commentsコントローラ
```
class CommentsController < ApplicationController
  #コメントを作成するためのアクション
  def create
    comment = current_user.comments.build(comment_params)
    if comment.save
      redirect_to board_path(comment.board), success: t('defaults.flash_message.created', item: Comment.model_name.human)
    else
      redirect_to board_path(comment.board), danger: t('defaults.flash_message.not_created', item: Comment.model_name.human)
    end
  end

  private
  
  #パラメータに対して許可したものだけを受け付ける
  def comment_params
    params.require(:comment).permit(:body).merge(board_id: params[:board_id])
  end
end
```
コントローラ内の記述を１つずつ分解する   

```
 comment = current_user.comments.build(comment_params)
```
ログイン中のユーザー(`current_user`)のcommentsコレクションに対して、`comment_params`のパラメータを用いて新たにコメントを作成→変数`comment`に代入する
```
    if comment.save
      redirect_to board_path(comment.board), success: t('defaults.flash_message.created', item: Comment.model_name.human)
    else
      redirect_to board_path(comment.board), danger: t('defaults.flash_message.not_created', item: Comment.model_name.human)
    end
```
コメントが正常に保存された場合は、掲示板詳細ページにリダイレクトし、successメッセージを表示する  
コメントの保存に失敗した場合は、掲示板詳細ページにリダイレクトし、dangerメッセージを表示する
```
  private

  def comment_params
    params.require(:comment).permit(:body).merge(board_id: params[:board_id])
  end
```
パラメータをフィルタリングするためのメソッドを作成する。  
慣習として、モデル名に`_params`をつけることが一般的。

`.merge(board_id: params[:board_id]`：リクエストから受け取ったパラメータの中から`board_id`を取り出し、それを`comment_params`に追加して新たなパラメータを作成するための記述。  
ここで取り出される`board_id`は、どの掲示板に対するコメントなのかを示す情報で、そのコメントがどの掲示板に所属するのかを関連付けるために役立つ。

## Boardsコントローラにコメントを表示・投稿するためのshowアクションを追加

```
#app/controller/boards_controller.rb

  def show
    @board = Board.find(params[:id])
    @comment = Comment.new
    @comments = @board.comments.includes(:user).order(created_at: :desc)
  end
```
```
    @board = Board.find(params[:id])
```
`params[:id]`で受け取ったIDをもつBoardをデータベースから見つけて、その結果をインスタンス変数@boardに格納する。  
`@board`とすることでインスタンス変数になり、同じコントローラ内の他のアクションや`show.html.erb`内で受け取ったデータをつかうことが可能になる。

```
    @comment = Comment.new
```
新規のコメントのインスタンスを作成する。主に、フォームで新規コメントを作成する際に使用する。

```
    @comments = @board.comments.includes(:user).order(created_at: :desc)
```
@boardで取得した掲示板に関連付いているコメント(`@board.comments`)を取得する。  
N+1問題を防ぐために`includes(:user)`を使って各コメントに紐づくユーザー情報を１度のクエリで読み込ませる。

# Viewファイルの記述
## 掲示板詳細ページ
```
#app/views/boards/show.html.erb

<div class="container pt-5">
  <div class="row mb-3">
    <div class="col-lg-8 offset-lg-2">
      <h1><%= t('.title') %></h1>
      <!-- 掲示板内容 -->
      <article class="card">
        <div class="card-body">
          <div class='row'>
            <div class='col-md-3'>
              <%= image_tag @board.board_image_url, class: 'card-img-top img-fluid', width: '300', height: '200' %>
            </div>
            <div class='col-md-9 d-flex flex-column'>
              <div class="d-flex justify-content-between align-items-center">
                <h3 style='display: inline;'><%= @board.title %></h3>
                <ul class="list-inline justify-content-center" style="float: right;">
                  <li class="list-inline-item">
                    <a href="#" class='edit-comment-link'>
                      <i class='bi bi-pencil-fill'></i>
                    </a>
                  </li>
                  <li class="list-inline-item">
                    <a href="#" class='delete-comment-link'>
                      <i class="bi bi-trash-fill"></i>
                    </a>
                  </li>
                </ul>
              </div>
              <ul class="list-inline">
                <li class="list-inline-item"><%= @board.user.decorate.full_name %></li>
                <li class="list-inline-item"><%= l @board.created_at, format: :long %></li>
              </ul>
            </div>
          </div>
          <p><%= simple_format(@board.body) %></p>
        </div>
      </article>
    </div>
  </div>

  <!-- コメントフォーム -->
  <%= render 'comments/form', board: @board, comment: @comment %>

  <!-- コメントエリア -->
  <div class="row">
    <div class="col-lg-8 offset-lg-2">
      <table class="table">
        <tbody id="table-comment">
          <%= render @comments %>
        </tbody>
      </table>
    </div>
  </div>
</div>
```
`boards_controller.rb`のshowアクション内のインスタンス変数@boardからデータを取得して`@board.title`や`@board.body`を表示させることができる

### simple_formatヘルパーとは？
` <%= simple_format(@board.body) %>`のように記述することで、テキストをHTMLに変換する。  
simple_formatを使用することで、元のテキストの改行や連続する行に対して適切なHTMLタグで囲ってくれる。（`<br>`や`<p>`など）

### パーシャルのレンダリングとローカル変数へのデータの受け渡し
```
#コメントフォームのレンダリングとローカル変数へのデータを受け渡すためのコード

  <%= render 'comments/form', board: @board, comment: @comment %>
```
`render 'comments/form` ：commentsディレクトリ下の`_form.html.erb`をレンダリングする  

`board: @board, comment: @comment`：ローカル変数(board, comment)にshowアクションのインスタンス変数(@board, @comment)のデータを渡して、部分テンプレート内(`_form.html.erb`)でも同じ値を使える様にする

```
#コメントを順番に表示

 <%= render @comments %>
```
boradsコントローラのshowアクション内で記述した`  @comments = @board.comments.includes(:user).order(created_at: :desc)`から`@comments`に格納されたCommentモデルのオブジェクトごとにパーシャル`_comment.html.erb`を１つずつ順番にレンダリングする。  
`<%= render partial: 'comment', collection: @comments %>`を略した記述。

パーシャルには自動的に`comment`というローカル変数が渡されて、それぞれのレンダリングごとに現在のコメントがセットされる。  
ここで設定されるローカル変数の名称はインスタンス変数名の単数系になる。これはRailsの採用している機能にようもの。
また、render @commentsで表示させたいパーシャルのファイル名も単数系にする必要がある。

#パーシャルの記述
##コメントフォームの部分テンプレート
```
#app/views/comments/_form.html.erb

<div class="row mb-3" id="comment-form">
  <div class="col-lg-8 offset-lg-2">
    <%= form_with model: comment, url: board_comments_path(board) do |form| %>
      <%= form.label :body %>
      <%= form.text_area :body, class: 'form-control mb-3', row: '4', placeholder: Comment.human_attribute_name(:body) %>
      <%= form.submit t('defaults.post'), class: 'btn btn-primary' %>
    <% end %>
  </div>
</div>
```
 ` <%= form_with model: comment, url: board_comments_path(board) do |form| %>`：  
Commentモデルに関連するフォームを作成し、送信先URLを`board_comments_path(board)`に設定するための記述。

`board_comments_path(board)`：フォームに入力されたデータを適切な場所に保存するために`/boards/:board_id/comments/`という形式のURLを生成する。  
引数として`(board)`を渡すことで、`:board_id`の部分が`board`オブジェクトのIDに置き換えられる。

## 掲示板上のコメントを表示するための部分テンプレート
```
<tr id="comment-<%= comment.id %>">
  <td style="width: 60px">
    <%= image_tag('sample.jpg', class: 'rounded-circle', width: 50, height: 50) %>
  </td>
  <td>
    <h3 class="small"><%= comment.user.decorate.full_name %></h3>
    <p><%= simple_format(comment.body) %></p>
  </td>

  <% if current_user.own?(comment) %>
  <td class="action">
    <ul class="list-inline justify-content-center" style="float: right;">
      <li class="list-inline-item">
        <%= link_to '#', class: 'edit-comment-link' do %>
          <i class='bi bi-pencil-fill'></i>
        <% end %>
      </li>
      <li class="list-inline-item">
        <%= link_to '#', class: 'delete-comment-link' do %>
          <i class="bi bi-trash-fill"></i>
        <% end %>
      </li>
    </ul>
  </td>
  <% end %>
</tr>
```
`<% if current_user.own?(comment) %>`：条件分岐で、編集・削除ボタンを投稿したユーザーのみに表示するための記述
`current_user.own?(comment)`：ログイン中のユーザーがそのコメントの所有者か確認する。

### own?メソッドの定義
```
#app/models/user.rb

  def own?(object)
    id == object&.user_id
  end
```
ユーザーがオブジェクトの所有者か判定するためのメソッド
Userモデルに記述することで、アプリケーションのいろんな場所で再利用ができるのと、修正する場合もUserモデルのみで済むため保守性の向上にもなる。

`id == object&.user_id`：現在のユーザー(`self.id`)が特定のオブジェクト(今回はcomment)の所有者(`user_id`)であるか判定

セーフナビゲーション演算子`$.`：objectが`nil`の場合そのまま`nil`を返しNoMethodErrorを防ぐための記述。  
そうすることで、オブジェクトが存在しない場合falseを返し、エラーを起こさずにコードブロックをスキップできる。
