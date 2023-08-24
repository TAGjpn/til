# Sorceryのパスワードリセット機能の処理の流れ

## 手順
1. reset_passwordモジュールの導入
2. Userモデルにパスワードリセットのためのカラムを追加する
3. Userモデルにtokenの一意性バリデーションを追加する
4. PasswordResetsControllerを作成し、各アクションを作成する
5. パスワードリセット用のルーティングを追加する
6. `default_url_options`を設定する
7. メーラー`user_mailer.rb`を設定し、Tokenを含むメールを送信できるようにする
8. パスワードリセット申請(`new.html.erb`)とパスワード変更(`edit.html.erb`)のビューを作成する
9. 開発環境でメールの確認ができるように`letter opener`を設定する

## パスワードリセット機能の処理の流れ
1. ユーザーがパスワードリセットを申請する。
2. システムが一意のreset_password_tokenを生成し、そのユーザーのレコードに保存する。
3. システムがそのトークンを含むリンクをユーザーのメールアドレスに送信する。
4. ユーザーがメールに記載されているリンクをクリックする。
5. リンクのURLにはreset_password_tokenが含まれているため、システムはこのトークンを使用してデータベース内で対応するユーザーを検索する。
6. システムがそのユーザーのレコードを速やかに見つけ、パスワードリセットページをユーザーに表示する。

## 1.reset_passwordモジュールの導入
reset_passwordモジュールの導入は、[公式wiki](https://github.com/Sorcery/sorcery/wiki/Reset-password)を参考  
```
rails g sorcery:install reset_password --only-submodules
```
## 2&3.Userモデルにカラムを追加し、一意性の制約を追加する
reset_passwordモジュールをインストールすることで追加されたマイグレーションファイルにindexの追加と一意性の制約を追加する。
 - tokenを元にユーザーを検索する速度が向上する
 - tokenの重複を回避する

### tokenとは何か？
今回のパスワードリセット機能における`token`とは、ランダムに生成された文字列のこと。  
パスワードリセット申請が行われると`token`を生成し、入力されたメールアドレス宛に送信する。  
パスワードリセットページ`edit.html.erb`にアクセスするには、`token`が必要。  
そうすることで、メールアドレス所有者である本人だけがパスワードリセットを実行できるようになる。

```
class SorceryResetPassword < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :reset_password_token, :string, default: nil
    add_column :users, :reset_password_token_expires_at, :datetime, default: nil
    add_column :users, :reset_password_email_sent_at, :datetime, default: nil
    add_column :users, :access_count_to_reset_password_page, :integer, default: 0

    add_index :users, :reset_password_token, unique: true
```
## 4. PasswordResetsControllerを作成
公式wikiを元に、コントローラを作成。
```
class PasswordResetsController < ApplicationController
  skip_before_action :require_login
  def new; end

  def create
    @user = User.find_by(email: params[:email])
    @user&.deliver_reset_password_instructions!
    redirect_to login_path, success: t('.success')
  end

  def edit
    @token = params[:id]
    @user = User.load_from_reset_password_token(@token)
    not_authenticated if @user.blank?
  end

  def update
    @token = params[:id]
    @user = User.load_from_reset_password_token(params[:id])
    return not_authenticated if @user.blank?

    @user.password_confirmation = params[:user][:password_confirmation]
    if @user.change_password(params[:user][:password])
      redirect_to login_path, success: t('.success')
    else
      flash.now[:danger] = t('.fail')
      render :edit, status: :unprocessable_entity
    end
  end
end
```
### newアクション
パスワードリセット申請ページ`new.html.erb`をレンダリングする
```
  def new; end
```
### createアクション
1. フォームに入力されたメールアドレスからユーザーを特定し`@user`に格納する。  
2. `@user`が`nil`でない場合に`deliver_reset_password_instructions!`が実行される。
3. ログインページにリダイレクトし、フラッシュメッセージを表示する。

#### deliver_reset_password_instructions!では何が行われる？
1. ランダムなトークンを生成し、URLに埋め込む
3. 生成したトークンをユーザーの`reset_password_toke`カラムに保存
4. トークンを含むURLを記載したメールをユーザーに送信
```
  def create
    @user = User.find_by(email: params[:email])
    @user&.deliver_reset_password_instructions!
    redirect_to login_path, success: t('.success')
  end
```
### editアクション
ユーザーが受信したメールに記載されたURLへアクセスするとeditアクションが実行される  
`http://localhost:3000/password_resets/トークンの値/edit`
1. `params[:id]`でURLに埋め込まれたトークンの値を取得し`@token`に格納
2. `@token`の値を元にユーザーを検索し、該当するユーザーのオブジェクトを`@user`に格納する。
3. もし`@user`が`nil`の場合は、定義済みの`not_authenticated`メソッドを実行する
```
  def edit
    @token = params[:id]
    @user = User.load_from_reset_password_token(@token)
    not_authenticated if @user.blank?
  end
```
### updateアクション
1.  editアクションと同じ処理をまず行う
2.  フォームから送信された値をuserオブジェクトの`password_confirmation`に格納する
3.  `change_password`メソッドにより、新しいパスワードを設定して`password_confirmation`に格納された値と一致するかチェック
4.  一致する場合はログインページにリダイレクトし、成功メッセージを表示
5.  一致しない場合は、再度editアクションをレンダリングし、失敗メッセージを表示
```
  def update
    @token = params[:id]
    @user = User.load_from_reset_password_token(params[:id])
    return not_authenticated if @user.blank?

    @user.password_confirmation = params[:user][:password_confirmation]
    if @user.change_password(params[:user][:password])
      redirect_to login_path, success: t('.success')
    else
      flash.now[:danger] = t('.fail')
      render :edit, status: :unprocessable_entity
    end
  end
```
