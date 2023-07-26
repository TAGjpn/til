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

`shallow: true`：ネストされたルーティングを浅くするためのルーティングの設定オプション。  
子リソースの一部のアクション（通常は　show, edit, update, destroy)については、URLに親リソースのIDを含めない様にする。  
このオプションの利点は、URLがシンプルになることと、親リソースのIDが不要なアクションではそのIDを取得する必要がなくなること。
