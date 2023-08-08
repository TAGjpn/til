# Turbo Streamを使用したブックマークボタンのajax化
## ajax化とは？
Ajax:Asynchronous JavaScript and XMLの略で、ページの一部を非同期で更新するための技術。  
Turboを使用してajax化することで、ページのリロードを行わずにページの一部分だけ更新をすることが可能。  
今回は、ボタンの置き換えをしたいので、`turbo_stream.replace`を使う。

## 「非同期」とはどういうことなのか？
同期的な処理では、１つのタスクが完了するまで次に進むことができない。
非同期的な処理では、１つのタスクが完了するのを待たずに次のタスクを進めることができる。

非同期的な処理をブックマークボタンで例えると、ブックマークボタンを押してブックマーク済みに切り替わるのを待たずに、次のボタンを押したりスクロールをしたりすることができる。

# gemの追加
Rails7には元からHotwireによりTurboが組み込まれているため、バージョンを指定する必要がなければ不要。
```
gem 'turbo-rails', "1.1.1"
```
```
$ bundle install
```

# ブックマークボタンが押されてからブックマーク済みボタンが表示されるまでの流れ
## 1.ユーザーがブックマークボタンを押すことでPOSTリクエスト
ボタンが押されると`bookmarks_path(board_id: board.id)`に対してPOSTリクエストが送信される。
`data: { turbo_method: :post }`と指定することで、POSTリクエストがTurboによって非同期で行われる。 
```
#_bookmark.html.erb

<%= link_to bookmarks_path(board_id: board.id), id: "bookmark-button-for-board-#{board.id}", data: { turbo_method: :post } do %>
  <i class="bi bi-star"></i>
<% end %>
```
### なぜこの記述でcreateアクションが呼び出されるのか？
POSTリクエストをサーバーに送ることによって、`routes.rb`に記述したcreateアクションが呼び出される。  
これは、RailsのRESTと呼ばれる設計原則に基づき、デフォルトでPOSTリクエストがcreateアクションにマッピングされているため。
```
resources :bookmarks, only: %i[create destroy]
```
[Railsのルーティング-Railsガイド](https://railsguides.jp/routing.html)

## 2.createアクションが実行される
createアクション内では以下の処理が行われる  
1. `params[:board_id]`で`bookmarks_path(board_id: board.id)`により送信された`board_id`を受け取って一致する掲示板を探し出してデータを`@board`に格納する
2. ログイン中のユーザーのブックマークコレクションに`@board`を追加する
```
  def create
    @board = Board.find(params[:board_id])
    current_user.bookmark(@board)
  end
```
処理が終わると`create.turbo_stream.erb`がレンダリングされる。  
ビューに渡すためにインスタンス変数(`@board`)にする必要がある。

### なぜ`create.turbo_stream.erb`がレンダリングされるのか？
Railsの規約に基づくデフォルトの動作で、`render`や`redirect`の指定がない場合はアクション名と同じ名前がついたビューファイルが読み込まれる。  
今回は、Turbo Streamの非同期リクエストに対するレスポンスのため、`create.turbo_stream.erb`が呼び出される。

- 通常のHTMLレスポンス：`アクション名.html.erb`  
- Turbo Streamのレスポンス:`アクション名.turbo_stream.erb`

[レイアウトとレンダリング-Railsガイド](https://railsguides.jp/layouts_and_rendering.html)

## 3.Turbo Streamによるブックマークボタンの置き換え
`turbo_stream.replace`により`"bookmark-button-for-board-#{@board.id}"`というIDを持つパーツ(HTMLエレメント)を`do`と`end`の間に書かれたブロックに置き換えることができる。 

`<%= render 'boards/unbookmark', board: @board %>`では、ブックマーク済みボタンのパーシャル(`_unbookmark.html.erb`)がレンダリングされる。  

また、renderメソッドでパーシャルを呼び出すときは、パーシャル内で使う変数を明示的に渡す必要がある。  
`board: @board`で、インスタンス変数`@board`の値をローカル変数`board`としてパーシャルに渡す。

その結果、createアクション実行後にブックマークボタンがブックマーク済みボタンに非同期で置き換えられる。
```
#create.turbo_stream.erb

<%= turbo_stream.replace "bookmark-button-for-board-#{@board.id}" do %>
  <%= render 'boards/unbookmark', board: @board %>
<% end %>
```
`@board`の値を`board`として渡さないと、パーシャル`_unbookmark.html.erb`内で`board`変数を使った処理ができず、ブックマーク解除ボタンが正しく表示されない。
