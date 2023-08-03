# 中間モデルを作成する
`User`モデルと`Board`モデルの中間モデル`Bookmark`モデルを作成する
```
$ rails g model Bookmark user:references board:references
```
UserモデルとBoardモデルと外部キーを持つカラムを追加する
## マイグレーションファイルの記述
```
class CreateBookmarks < ActiveRecord::Migration[7.0]
  def change
    create_table :bookmarks do |t|
      t.references :user, null: false, foreign_key: true
      t.references :board, null: false, foreign_key: true

      t.timestamps
    end
    add_index :bookmarks, [:user_id, :board_id], unique: :true
  end
end
```
Bookmarksテーブルに`user_id`と`board_id`の複合インデックスを作成する。

```
 add_index :bookmarks, [:user_id, :board_id], unique: :true
```
### なぜインデックスを作成するのか？
カラムに一意性規約のバリデーションを設定する場合はインデックスを作成する。  
→値が一意か調べるにはカラムの値を検索する必要があるため、インデックスを作成しておくことでパフォーマンスが向上するから。
## アソシエーションとバリデーションの設定
```
#/models/bookmark.rb

class Bookmark < ApplicationRecord
  belongs_to :user
  belongs_to :board
  validates :user_id, uniqueness: { scope: :board_id } #1
end
```
user_idとboard_idの組み合わせが一意であるようにするための記述。#1  
つまり、「このuser_idはこのboard_idのスコープ内で一意である」という指定。  
[参考ページ(特定スコープ内でuniquenessバリデーションをかける)](https://techracho.bpsinc.jp/hachi8833/2021_07_27/109827)
```
#/models/user.rb

  has_many :bookmarks, dependent: :destroy
  has_many :bookmark_boards, through: :bookmarks, source: :board
```
「このモデルがBookmarkモデルを通じて複数のBoardモデルを持っている」という関係性を作る。  
`bookmark_boards`を使うことでUserモデルに関連付けされたBoardモデルにBookmarkモデルを介してアクセルできるようになる。  
→ユーザーがブックマークした掲示板のコレクションを取得できるようになる。

### なぜthrough:複数形, source:単数形 なのか？
それはRailsのアソシエーションの命名規則によるものだダ。  
through: :bookmarksの部分は、「このモデルがどのモデルを経由してbookmark_boardsと関連付けられているのか」を指定しているダ。この場合、モデル名は複数形で書くのが慣習だダ。  
一方、source: :boardの部分は、「実際にbookmark_boardsとして参照されるオブジェクトは何か」を指定しているダ。この場合、そのオブジェクトのモデル名を単数形で書くのが慣習だダ。  
だから、through:のあとは複数形、source:のあとは単数形になっているダナ。  

[参考ページ](https://web-camp.io/magazine/archives/17680)

### ここで定義した`bookmark_boards`はどのように使うのか
Userモデル内で定義した`bookmark_boards`はBookmarkモデルを介して関連付けされたBoardモデルのコレクションを取得することができる。  
つまりUserオブジェクトに対してのみ使うことができる。

例）以下のboardsコントローラ内のbookmarksアクションではログイン中のユーザーがブックマークした一覧を取得できる。
```
#boards_controller.rb

  def bookmarks
    @bookmark_boards = current_user.bookmark_boards.includes(:user).order(created_at: :desc)
  end
```
```
#/models/board.rb

  has_many :bookmarks, dependent: :destroy
```
# ブックマーク一覧ページを作成する

## collectionルーティングを使いBookmark一覧を表示させるには？
ネストしたリソースを作成することで、`boards/bookmarks`というURLへアクセスできるようにルーティングを定義する。 
`boards/bookmarks`に対応するビューファイルとBoardsコントローラへの`bookmarks`アクションの作成は別途必要となる。
#### ルーティング
```
resources :boards do
  resources :comments, only: %i[create update destroy], shallow: true
  collection do
    get :bookmarks
  end
end
resources :bookmarks, only: %i[create destroy]
```
### collectionルーティングを使う意味は？
特定のリソースIDが不要なアクションをリソースのネストなしに定義できる。これにより、シンプルなURLとスッキリしたコントローラアクションを実現できる。  

通常のルーティングの場合は、`boards/user_id/bookmarks`となる。  
コレクションルーティングを定義することで`boards/bookmarks`でログイン中のユーザーのブックマーク一覧を表示することができる。
#### bookmarksアクション
ログイン中のユーザーのブックマーク一覧を`@bookmark_boards`に格納する。
```
#boards_controller.rb

  def bookmarks
    @bookmark_boards = current_user.bookmark_boards.includes(:user).order(created_at: :desc)
  end
```
#### Viewファイル
bookmarksアクション内で定義した`@bookmark_boards`に格納されたログイン中のユーザーのブックマーク一覧を表示する
```
#/boards/bookmarks.html.erb

      <% if @bookmark_boards.present? %>
        <%= render @bookmark_boards %>
      <% else %>
        <p><%= t('.no_result') %></p>
      <% end %>
```
# ブックマーク機能を作成する
ブックマーク機能は、以下の順序で処理が行われる様にする。（以下の内容は未ブックマークの場合）
1. ログイン中のユーザーがブックマークしていない投稿にブックマークボタンのパーシャル(bookmark)を表示する
2. 表示されたブックマークボタンを押すとbookmarksコントローラのcreateアクションを実行する
3. bookmarksコントローラのcreateアクション内で`bookmark(board)`メソッドを実行してUserオブジェクトに関連付けをする
4. Userモデル内で定義した`bookmark(board)`メソッドが実行されアクティブレコードが中間テーブルにブックマークした投稿を追加する
5. ブックマークに追加された投稿のボタンの表示をブックマーク済みのパーシャル(unbookmark)に切り替える

## Userモデル内にメソッドを定義する
これらを実装するためには、以下の３つの機能が必要となる。
- ブックマークを追加する機能
- ブックマークを解除する機能
- ブックマークをしているか判定する機能

### ブックマークを追加する機能
Userオブジェクトに対して`bookmark(board)`が呼び出されると、そのUserオブジェクトのブックマーク一覧(`bookmark_boards`)に引数で渡された`board`を追加する。　　
`board`に格納された投稿idの取得は、bookmarksコントローラ内で定義する。

`bookmark_boards << board`は引数で渡された`board`を`bookmark_boards`コレクションに追加するための記述。
```
  def bookmark(board)
    bookmark_boards << board
  end
```
### ブックマークを解除する機能
引数で渡された`board`をそのユーザーの`bookmark_boards`コレクションから削除する。

destroyメソッドの引数に`(board)`を指定することで、`bookmark_boards`コレクションから該当する`board`を探し削除する。
```
  def unbookmark(board)
    bookmark_boards.destroy(board)
  end
```

### ブックマークをしているか判定する機能
include?メソッドを使うことで引数に渡された`(board)`が`bookmark_boards`に含まれているか判定する。  
含まれている場合は`true`を返し、含まれていない場合は`false`を返す。
```
  def bookmark?(board)
    bookmark_boards.include?(board)
  end
```
