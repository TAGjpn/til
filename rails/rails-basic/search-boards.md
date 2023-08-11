# ransackを使った検索機能

# ブックマーク一覧ページで検索フォームに入力されたデータの処理の流れ

## 1.検索フォームのパーシャルの表示方法
インスタンス変数`@q`をローカル変数`q`に渡す。  
ルーティングヘルパーメソッド`bookmarks_boards_path`をローカル変数`url`に渡す。

### なぜurlオプションを指定するのか？
urlオプションで明示的に指定しないと、Boardモデル全体に対して検索をかけてしまう  
（明確な理由が見つからなかったが、ルーティングをネストさせているのが関係していそう）
```
 <%= render 'search_form', q: @q, url: bookmarks_boards_path %>
```
## 1.検索フォーム
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
