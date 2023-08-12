# ブックマーク一覧ページで「しまじろう」と検索したらどんな流れで処理されるのか？

Ransackを使った処理の流れは、以下の様になっている。
1. 検索をすると、キーワードと検索条件をクエリパラメータとして送信する
2. 受け取ったパラメータを元にransackがコレクションから検索を行う
3. 検索結果を元に再度ビューにレンダリングを行う

## 1.検索フォームのパーシャルの表示方法
インスタンス変数`@q`をローカル変数`q`に渡す。  
ルーティングヘルパーメソッド`bookmarks_boards_path`をローカル変数`url`に渡す。
```
 <%= render 'search_form', q: @q, url: bookmarks_boards_path %>
```
## 2.検索フォームからコントローラへ入力データを渡す
`q: @q, url: bookmarks_boards_path`として2つのローカル変数を受け取っている。  
これにより、以下の流れでコントローラへ入力されたキーワードと検索条件を渡すことができる。

1. ユーザーが「しまじろう」という文字列をフォームに入力し検索をする。
2.  `?q[:title_or_body_cont]=しまじろう`というURLのクエリパラメータとして送信する

### URLに付与されるクエリパラメータとは？
URLの後ろに?で始まるキーと値の組み合わせで、特定の情報や要求をサーバーに伝えるための手段。

今回の場合は、`:title_or_body_cont`という検索条件と`しまじろう`というキーワードを送信している。  
これにより、`title`または`body`のカラムに`しまじろう`というキーワードが含まれる掲示板のみを検索することができる。  

### URLオプションを指定する理由は？

urlオプションを指定することで、指定されたURL`bookmarks_boards_path`へRansackの検索クエリが送信される。  
そのURLに対応するコントローラ`boards`のアクション`bookmarks`で、検索処理が行われる。
   
```
<%= search_form_for q, url: url do |f| %>
  <div class="input-group mb-3">
    <%= f.search_field :title_or_body_cont, class: 'form-control', placeholder: t('defaults.search_word') %>
    <div class="input-group-append">
      <%= f.submit class: "btn btn-primary" %>
    </div>
  </div>
<% end %>
```
## 3.コントローラ内でRansackが検索を行いビューへ結果を返す
1. `current_user.bookmark_boards`で、現在のユーザーがブックマークした掲示板を取得する
2. その結果に対して `.ransack(params[:q])` として、Ransackの検索メソッドを適用する（`params[:q]`には「しまじろう」というキーワードと`:title_or_body_cont`が格納されている）
3. 検索結果が`@q`の中に格納される（実際にはRansackの検索クエリオブジェクト）
4. `@q.result`を使うことで検索結果を取得し、`@bookmark_boards`へ格納する
5. 再度`<%= render @bookmark_boards %>`が実行されることで、「しまじろう」が含まれたブックマーク済みの掲示板のみがレンダリングされる
```
  def bookmarks
    @q = current_user.bookmark_boards.ransack(params[:q])
    @bookmark_boards = @q.result.includes(:user).order(created_at: :desc).page(params[:page])
  end
```

### 初回アクセス時も正常に動作する理由(by ChatGPT)

1. **初回のアクセス時**:
   - ユーザーが`bookmarks`のページを開いた時、`bookmarks`アクションが実行されます。
   - `params[:q]`には何も入っていないため、`current_user.bookmark_boards.ransack(params[:q])`はユーザーがブックマークした掲示板全てを検索の対象とします。
   - `@bookmark_boards`にはその結果、すなわちユーザーがブックマークした掲示板全てが格納されます。
   - `bookmarks.html.erb`で`<%= render @bookmark_boards %>`としているので、ブックマークしている掲示板が全てレンダリングされる。

2. **Ransackを使っての検索時**:
   - ユーザーがキーワード（例えば「しまじろう」）を入力して検索を実行すると、検索のクエリが`params[:q]`として送信されます。
   - 再び`bookmarks`アクションが実行される際に、`params[:q]`には今度は検索キーワードが入っています。それが`current_user.bookmark_boards.ransack(params[:q])`に渡され、指定されたキーワードを含むブックマーク済みの掲示板を検索します。
   - `@bookmark_boards`には検索結果が格納されます。
   - 再び`bookmarks.html.erb`で`<%= render @bookmark_boards %>`が実行されるので、キーワードが含まれたブックマーク済みの掲示板のみがレンダリングされる。

このように、初回アクセス時と検索後の動作が変わるのは、`params[:q]`の内容が変わることでRansackによる検索の結果が変わるためです。
