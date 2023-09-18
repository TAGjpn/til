# system spec

## system specは何のために必要なのか？
ユーザーが実際にアプリケーションを使用する際のシナリオを想定してテストケースを作成する。  
例えば、リンクやボタンをクリックした際のページ遷移や、フォームの動作、JavaScriptの動作、アクセス制限などをテストする。  
これらの操作をテストすることで、コードの変更や新しい機能の追加により生じたバグを早期に検出することが可能。

## system specを書く手順
### 1.環境のセットアップをする
Gemのインストールをする。
```
group :test do
  gem 'capybara'
  gem 'selenium-webdriver'
  gem "webdrivers"
end
```
#### capybaraとは？
Webアプリケーションの統合テストツール。  
ユーザーが実際にブラウザで操作する様な動作をシミュレートし、ページの内容やJavaScriptの動作をテストすることができる。  
ウェブドライバと組み合わせてブラウザの操作を実現する。

#### selenium-webdriverとは？
ウェブブラウザを操作するためのツール。  
Capybaraと組み合わせることで、実際のブラウザ上でテストを行うことができる。

#### webdriversとは？
各ブラウザ用のWebDriverを自動で管理し、ダウンロードして更新するためのツール。  
`selenium-webdriver`gemを使用する際、特定のブラウザ(ChromeやFirefox)を操作するためには専用のドライバが必要となる。  
`webdrivers`gemがドライバを自動でダウンロードし、最新バージョンを保持してくれる。

### 2. formatの設定をする
`.rspec`ファイルに下記のコードを記述する。
```
--format documentation
```
これにより、デフォルトの成功したテスト`.`と失敗したテスト`F`だけでなく、「ドキュメンテーション形式」で出力してくれるようになる。

### 3.webdriverの設定
Dockerを使用してSystem specを実行するための設定を`spec/rails_helper.rb`に記述する。
```
Dir[Rails.root.join('spec', 'support', '**', '*.rb')].sort.each { |f| require f }
```
`spec/support`ディレクトリ下の｀.rb`ファイルをすべて読み込むための設定。  
通常supportディレクトリには、テストで使うヘルパーメソッドや共通の設定をまとめて配置することが多い。

```
config.before(:each, type: :system) do ... end
```
`type: ;system`と指定されたテストが実行されると、このブロック内に記述した設定が毎回適用される。

```
driven_by :remote_chrome
```
System specで使用するブラウザドライバとして、`:remote_chrome`を指定している。  
これにより、Docker内で動作するChromeブラウザを使ってテストを実行することが可能になる。

```
Capybara.server_host = IPSocket.getaddress(Socket.gethostname)
```
Capybaraで起動するテスト用のサーバーのホストw現在のホスト名の　IPアドレスに設定する。

```
Capybara.server_port = 4444
```
Capybaraで起動するテスト用のサーバーのポートを4444に設定する。

```
Capybara.app_host = "http://#{Capybara.server_host}:#{Capybara.server_port}"
```
テストを実行するアプリケーションのホスト情報を設定する。  
ブラウザがテストを実行する際の起点となるURLとなる。

```
Capybara.ignore_hidden_elements = false
```
Capybaraはデフォルトのままでは、ページ上の非表示要素を無視する。  
この設定をすることで、非表示要素も操作の対象となる。

```
#spec/support/capybara.rb

Capybara.register_driver :remote_chrome do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument('no-sandbox')
  options.add_argument('headless')
  options.add_argument('disable-gpu')
  options.add_argument('window-size=1680,1050')
  Capybara::Selenium::Driver.new(app, browser: :remote, url: ENV['SELENIUM_DRIVER_URL'], capabilities: options)
end
```
Capybaraの設定を書く。

### 4.テストケースを作成する
`spec/system/tasks_spec.rb`を作成

## テストケースの書き方
```
require 'rails_helper'

RSpec.describe 'Tasks', type: :system do
  let(:user) { create(:user) }
  let(:task) { create(:task) }

  describe 'ログイン前' do
    describe 'ページ遷移確認’ do
      context 'タスクの新規登録ページにアクセス' do
        it '新規登録ページへのアクセスが失敗する' do
          visit new_task_path
          expect(page).to have_content('Login required')
          expect(current_path).to eq login_path
        end
      end
    end
  end
end
```
### Railsのテスト用のヘルパーファイルを読み込む
```
require 'rails_helper'
```
テストで必要なライブラリや設定、ヘルパーメソッドなどを利用することができる。

### System Specを定義する
```
RSpec.describe 'Tasks', type: :system do
...
end
```
`describe`ブロックを用いて、`Tasks`という名前のSystem Specを定義する。  
`type: :system`は、このテストがSystem Specであることを明示する。

### FactoryBotを使ってオブジェクトを定義する
```
  let(:user) { create(:user) }
  let(:task) { create(:task) }
```
letメソッドを使ってテスト内で使用するオブジェクトを遅延評価で定義する。  
遅延評価とは、その変数が実際に呼び出されるまで評価　（実行）されない性質をもつ定義の仕方のこと。  
これにより、必要な時だけ変数の値を生成・評価できるため、テストのパフォーマンスを保持しつつ、テストのセットアップを柔軟に行うことができる。

#### letの主な特徴
1. 遅延評価  
   letで定義された変数は、その変数が最初に参照される時点でブロック内のコードを実行する。
   そのため、不要な初期化処理を避けることができる。
2. メモ化  
   一度評価されると、同じテストの中ではその結果がキャッシュされる。
3. 可読性の向上  
   letを使用することで、テストのセットアップを一箇所に集約できるためテストコードの可読性を向上させることができる。  
   

