# content_forを使いタイトルを動的に出力する
## このページで解説する内容について
RUNTEQカリキュラムのRails基礎12で学んだ内容を復習としてまとめたものです。  
主な内容としては、`content_for`を使ってタイトルを動的に出力する方法について解説します。

## content_forでタイトルを出力する流れ
1. `content_for`でタイトルを`:title`というキーに格納してメモリに保存する
2. `yield`を使って保存された値を取り出し`page_title`メソッドの引数として渡す
3. 渡された引数を元に`page_title`メソッド内で定義した三項演算子によりタイトルが生成されてビューに出力される

# ヘルパーメソッドを定義する
ページタイトルを生成するためのヘルパーメソッドを定義する
```
#app/helpers/application_helper.rb

module ApplicationHelper
  def page_title(title = '')
    base_title = 'RUNTEQ BOARD APP'
    title.empty? ? base_title : "#{title} | #{base_title}"
  end
end
```
`page_title`メソッドの引数`title`に対してデフォルト値を空文字にするために`''`を設定する。  
これにより、`page_title`が引数なしで呼び出された場合でもエラーにならずに三項演算子の処理が行われる。

## 三項演算子について
```
    title.empty? ? base_title : "#{title} | #{base_title}"
```
上記の記述は三項演算子を使い、条件を評価した結果に応じて２つの異なる結果を返すことができるようになっている。  
```
条件 ? 真の場合の値 : 偽の場合の値
```
つまりこの記述の場合は、  
`title`が空の場合、`base_title`を返す
`title`が空でない場合、`"#{title} | #{base_title}"`を返す

# content_forメソッドでタイトルをメソッドの引数に渡す
```
<% content_for(:title, t('.title')) %>
```
タイトルを動的に出力したいページにコードを記述することで、ビューファイルがレンダリングされた時に、`:title`というキーに`t('.title')`の値を格納する。  
`content_for`により`:title`に格納された値は、`yield`により取り出され、`page_title`メソッドの引数として渡される。

# ページタイトルの出力
```
    <title><%= page_title(yield(:title)) %></title>
```
`page_title`メソッドを使って生成したタイトルを出力する。  
`page_title`メソッドの引数として`yield(:title)`を渡すことで、`content_for`で格納された`:title`の値を取り出すことができる。

`:title`に取り出す値が格納されていない場合は、三項演算子(`empty?`)の結果がtrueの処理(今回の場合は`base_title`のみ出力)が行われる。

# content_forにより格納された値はどのように受け渡されるのか？
## 1.content_forで値を一時的に保存する
`content_for`を使用して値を指定すると、一時的にRailsによりメモリ上に保存される。  
これにより、ビューがレンダリングされる際に値を取り出して使うことができる。

## 2.yieldで保存された値を取り出す
`yield`を使うことで`content_for`で格納された値を取り出すことができる。  
同じリクエスト中であれば、複数回`yield`により取り出すことが可能。

## 3.保存した値が破棄される
各リクエストが処理され、レスポンスが完了すると、そのリクエストに関連した`content_for`により保存された値はメモリから破棄される。  
