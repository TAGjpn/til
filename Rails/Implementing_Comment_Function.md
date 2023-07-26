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

