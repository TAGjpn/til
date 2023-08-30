# 管理画面へのログインページを表示するまで

## 大まかな手順
1. AdminTEQのインストール
2. admin用のルーティングの追加
3. admin用のコントローラの作成
4. admin用のレイアウトファイルとビューファイルの作成
5. jsファイルとcssファイルの読み込み
6. titleをadminでも表示できるようにhelpersファイルのpage_titleメソッドの書き換え

## 1. AdminTEQのインストール
管理画面へのログイン画面や管理画面のテンプレートを導入するためにAdminTEQをインストール
```
yarn add "admin_teq@git+https://github.com/runteq/AdminTEQ.git#main"
```
これにより、`app/node_modules`が作成され、cssやjsの読み込みが可能になる。

## 2. admin用のルーティングの追加
コントローラやビューの階層をadminに限定するために`namespace`を使ってルーティングの定義をする・  
これにより、`app/controllers/admin/`内のコントローラを使用してビューを表示させることができる。

`/admin/login`というURLへのGETリクエストでログインページを表示するために以下の様に設定する。  
` get 'login', to: 'user_sessions#new', :as => :login`  
これにより、  
`Admin::UserSessionsController`の`new`アクションが呼び出される。

また、`:as => :login`で名前付きルーティングを作成することで、`login_path`や`login_url`というヘルパーメソッドがビューやコントローラ内で使用可能になる。
```
namespace :admin do
  root to: 'dashboards#index'
  resource :dashboard, only: %i[index]
  get 'login', to: 'user_sessions#new', :as => :login
  post 'login', to: 'user_sessions#create'
  delete 'logout', to: 'user_sessions#destroy', :as => :logout
end
 ```
## 3. admin用のコントローラの作成
管理画面用のコントローラは`application_controller`を継承する`admin/base_controller`を作成し、全ての管理画面用コントローラー(管理画面へのログイン機能も含む)はこの`base_controller` を継承する設計で進める必要がある。

そのため、以下の様にコントローラを継承させる。

`ApplicationController`を継承させて`Admin::Basecontroller`を作成。

```
#app/controllers/admin/base_controller.rb

class Admin::BaseController < ApplicationController

```
`Admin::BaseController`を継承させて`Admin::UserSessionsController`を作成。
```
#app/controllers/admin/user_sessions_controller.rb

class Admin::UserSessionsController < Admin::BaseController
```
### なぜベースコントローラを作成するのか？
`Admin::BaseController`を作るメリットはいくつかある。
- 管理者機能で使う共通の処理をまとめることができる
- 継承したコントローラでオーバーライドすることができる
- メンテナンス性の向上
- 一般ユーザーと管理者を明確に分けることができる

## 4. admin用のレイアウトファイルとビューファイルを作成
一般ユーザーとは別でレイアウトファイルを作成する。  
`app/views/admin/layouts/admin_login.html.erb`

管理者用のログイン画面のビューファイルを作成  
`app/views/admin/user_sessions/new.html.erb`
```
<% content_for(:title, t('.title')) %>
<main class="form-signin w-100 m-auto">
  <%= render 'shared/flash_message' %>
  <%= form_with url: admin_login_path, data: { turbo: false } do |f| %>
    <h1 class="h3 mb-3 fw-normal"><%= t('.title') %></h1>
    <div class="form-floating">
      <%= f.email_field :email, class: "form-control", placeholder: "name@example.com" %>
      <%= f.label :email%>
    </div>
    <div class="form-floating">
      <%= f.password_field :password, class: "form-control", placeholder: "Password" %>
      <%= f.label :password %>
    </div>
    <%= f.submit t('.login'), class: "w-100 btn btn-lg btn-primary" %>
  <% end %>
  <%= render 'admin/shared/footer'%>
</main>
```

## 5. jsファイルとcssファイルの読み込み
