# 中間モデルを作成する
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
→値が一意か調べるにはカラムの値を検索する必要があるため、インデックスを作成しておくことでパフォーマンスが向上するため。
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
### なぜthrough:複数形, source:単数形 なのか？
それはRailsのアソシエーションの命名規則によるものだダ。  
through: :bookmarksの部分は、「このモデルがどのモデルを経由してbookmark_boardsと関連付けられているのか」を指定しているダ。この場合、モデル名は複数形で書くのが慣習だダ。  
一方、source: :boardの部分は、「実際にbookmark_boardsとして参照されるオブジェクトは何か」を指定しているダ。この場合、そのオブジェクトのモデル名を単数形で書くのが慣習だダ。  
だから、through:のあとは複数形、source:のあとは単数形になっているダナ。  

[参考ページ](https://web-camp.io/magazine/archives/17680)
```
#/models/board.rb

  has_many :bookmarks, dependent: :destroy
```

1. **Bookmark モデルとテーブルを作成する**  
まずは、Bookmarkという名前のモデルとテーブルを作成します。カラムとしては`user_id`と`board_id`が必要です。両方とも`integer`型で、それぞれユーザーと掲示板を参照します。

2. **バリデーションの設定**  
`Bookmark` モデルには特定の掲示板に対してユーザーが一つのみブックマークを作成出来るようにバリデーションを設定します。これは、`user_id` と `board_id` の組み合わせが一意であることを保証します。

3. **ブックマーク機能の実装**  
ブックマークの登録と解除のためのアクションを作成します。ユーザーがブックマークボタンを押したとき、ブックマークがまだ存在しなければ新しくブックマークを作成し、存在していればそのブックマークを削除します。

4. **ブックマークボタンの作成**  
ブックマークボタンを作成し、各掲示板の表示に追加します。ボタンのIDは一意になるように設定します。

5. **ブックマーク一覧画面の作成**  
ブックマークした掲示板一覧を表示する画面を作成します。この画面は、ユーザーがブックマークした掲示板の情報を取得して表示します。

6. **ブックマーク一覧へのリンクの作成**  
ヘッダーにブックマーク一覧画面へのリンクを作成します。リンクのパスは`bookmarks_boards_path`とします。

7. **フラッシュメッセージの表示**  
ブックマークの追加や解除の操作後には、対応するフラッシュメッセージを表示します。

8. **条件に応じたブックマークボタンの表示切替**  
自分が作成した掲示板にはブックマークボタンが表示されないようにします。また、ブックマークの追加や解除の操作後は掲示板一覧画面にリダイレクトされるようにします。

9. **ブックマーク一覧画面のメッセージ表示**  
ブックマーク一覧画面でブックマークした掲示板が存在しないときは、「ブックマーク中の掲示板がありません」という文言を表示します。

10. **パーシャルの利用**  
ブックマークボタンの表示は、ブックマークしている状況によって変わるため、パーシャルを利用してコードを整理します。
