# 掲示板の編集、削除機能の実装
## このページで解説する内容について
RUNTEQカリキュラムのRails基礎13で学んだ内容を復習としてまとめたものです。

# 掲示板の編集、削除機能の実装の流れ

# 自分が作成した掲示板のみ編集・削除ボタンを表示させる
## 編集・削除ボタンのパーシャルを作成する
同じコードを各ビューファイルに書くよりも、パーシャルを作成して呼び出すことでコードの再利用性とメンテナンス性を向上させることができる。  
パーシャルで受け取りたいローカル変数( `item`, `edit_path`, `delete_path`, `delete_data` )を設定する。
```
#/shared/_crud_menus.html.erb

<% if current_user.own?(item) %>
  <ul class="list-inline justify-content-center">
    <li class="list-inline-item">
      <%= link_to edit_path, id: "button-edit-#{item.id}" do %>
        <i class='bi bi-pencil-fill'></i>
      <% end %>
    </li>
    <li class="list-inline-item">
      <%= link_to delete_path, id: "button-delete-#{item.id}", data: delete_data do %>
        <i class="bi bi-trash-fill"></i>
      <% end %>
    </li>
  </ul>
<% end %>
```
`<% if current_user.own?(item) %>`：条件分岐で、編集・削除ボタンを投稿したユーザーのみに表示するための記述  
`current_user.own?(item)`：ログイン中のユーザーがそのコメントの所有者か確認する。

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

セーフナビゲーション演算子`&.`：objectが`nil`の場合そのまま`nil`を返しNoMethodErrorを防ぐための記述。  
そうすることで、オブジェクトが存在しない場合falseを返し、エラーを起こさずにコードブロックをスキップできる。  

## パーシャルにローカル変数を渡して呼び出す
`crud_menus`を呼び出す時は、パーシャル内で使いたいローカル変数をlocalsオプションで渡す。  
掲示板詳細ページの場合、ローカル変数として`item`, `edit_path`, `delete_path`, `delete_data`を渡す必要がある。  
`locals: {  }`は以下の様に省略可能。
```
#/boards/show.html.erb

<%= render 'shared/crud_menus',
  item: @board,
  edit_path: edit_board_path(@board),
  delete_path: board_path(@board),
  delete_data: { turbo_method: :delete, turbo_confirm: t('defaults.delete_confirm') }
%>
```

# 掲示板の更新・削除
## アクセス制限とインスタンス変数の設定
`edit`, `update`, `destroy`に対してアクセス制限とインスタンス変数の設定をするためメソッドを作成。  
各アクションに記述する感じでもOK。
```
  private

  def find_board
    @board = current_user.boards.find(params[:id])
  end
```
このメソッドでは、ログイン中のユーザーの掲示板かどうか判定している。  
具体的には、ログイン中のユーザー(`current_user`)の掲示板の中から、パラメータで渡された掲示板(`params[:id]`)を探す。  
もし見つからない場合は、エラーを発生させる。

この記述により、
- アクセス制限：`edit`, `update`, `destroy`アクションはログインユーザーが所有する掲示板に対してのみ実行できるようにする。
- インスタンス変数の設定：`@board`には、選択された掲示板の全データ（`title`、`body`など）が格納される。

`before_action`で各アクションが実行される前に`find_board`メソッドを呼び出す。
```
class BoardsController < ApplicationController
  before_action :find_board, only: %i[edit update destroy]
```
## 掲示板の更新機能
```
  def edit; end

  def update
    if @board.update(board_params)
      redirect_to @board, success: t('defaults.flash_message.updated', item: Board.model_name.human)
    else
      flash.now[:danger] = t('defaults.flash_message.not_updated', item: Board.model_name.human)
      render :edit, status: :unprocessable_entity
    end
  end
```
`board_params`メソッドにより送信されたパラメータから取り出した情報を使って、`@board`の更新を行う。  
更新に成功すれば`true`が実行される(詳細ページへリダイレクト)  
更新に失敗すれば`false`が実行される。(エラーメッセージを表示して編集画面を再表示)

### status: :unprocessable_entityをつける理由
Hotwireを使用している場合、status: :unprocessable_entityをつけることでバリデーションエラー等が発生した時にHTTPステータスコード422を返すことができる。  
このステータスコードが返されると、クライアント側のHotwireがエラーを検知し、フラッシュメッセージの表示等の適切なアクションを取るようになる。  
Hotwireを使わない場合は不要。

## 掲示板の削除機能
```
  def destroy
    @board.destroy!
    redirect_to boards_path, status: :see_other, success: t('defaults.flash_message.deleted', item: Board.model_name.human)
  end
```
