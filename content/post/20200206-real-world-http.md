---
title: "20200206 Real World Http"
date: 2020-02-06T09:36:30+09:00
draft: true
---

# Ch.1 HTTP/1.0のシンタックス：基本となる4つの要素

以下の4つが基本要素

* メソッドとパス
* ヘッダー
* ボディ
* ステータスコード

## メソッド

サーバにリソースの操作種別を伝えるための指示。

|メソッド名|説明|
|--------|----|
|GET|ヘッダーとコンテンツの要求|
|HEAD|ヘッダの身を要求|
|POST|新しいドキュメントの投稿|
|PUT|既存のドキュメントの更新|
|DELETE|ドキュメントの削除|

## パス

HTTPではURL(ほぼ=URI)で操作対象のリソースを指定する。  

スキーマ://ホスト名/パス

これが基本形

## ヘッダ

ヘッダとは本文以外の様々なメタ情報を付与するための仕組み。  
「フィールド名: 値」という形式で、リクエストとレスポンスそれぞれに付与される。  
ヘッダーはHTTP/1.0にて追加されるようになった。

### リクエストヘッダの一例

|ヘッダ名|説明|
|--------|----|
|User-Agent|HTTPリクエストの送信元であるアプリケーションの名称|
|Referer|HTTPリクエストの送信時にクライアントが見ていたページのURL|
|Authorization|認証情報を付与する領域|

### レスポンスヘッダの一例

|ヘッダ名|説明|
|--------|----|
|Content-Type|レスポンスデータのMIMEタイプ|
|Content-Length|レスポンスボディのサイズ|
|Content-Encoding|レスポンスボディが何らかの形式でエンコードされている場合、そのエンコード形式を示す|
|Date|ドキュメントの日時|

## ボディ

ヘッダから1行空行を挟んでそれ以降をボディという。  
レスポンスボディは返ってきたコンテンツそのもの。  
リクエストボディはサーバーへ渡す情報、処理実行時のパラメータとなる。  

## ステータスコード

レスポンスに付与される三桁の数字。

|ステータスコード|説明|
|--------|----|
|1xx|処理中であることを示す。特殊な用途で使われる。|
|2xx|リクエストで指定された処理の成功を示す。|
|3xx|サーバからのクライアントへの命令。キャッシュやリダイレクトの利用を指示する。|
|4xx|クライアントからのリクエストに不備があることを示す。|
|5xx|サーバにてエラーが発生したことを示す。|

# Ch.2 HTTP/1.0のセマンティクス：ブラウザの基本機能の裏側

## フォーム

フォームがサポートするメソッドはGETとPOST。  
GETを使った場合はクエリストリングに、POSTを使った場合はボディに送信データが付与される。  
curlを使う場合は-dオプションを使えば良い。  

```bash
curl --http1.0 -d title="Hello World" http://localhost:18888
```

ただし、-dオプションではURLエンコードがなされないので、ブラウザの動作をシミュレートするならば--data-urlencodeを使うとよい。

## マルチパートリクエスト

フォームでファイルを送信するときはContent-Type: multipart/form-dataを使用する。  
curlを使う場合は-Fオブションを使う

```bash
curl --http1.0 -F file=@test.txt http://localhost:18888
```

そのほかのパラメータを付与したい場合は、-dオプションと同じようにパラメータの数だけ-Fオブションを追加していけばよい。  
x-www-form-urlencodedの場合は&がパラメータの区切り文字であったが、マルチパートの場合はContent-Typeヘッダに指定されたboundaryが区切りとなる。

## コンテントネゴシエーション

サーバーとクライアント間で期待したコンテンツをやりとりするため、リクエストとレスポンスのヘッダを使って調整を行う。

* ファイルの種類の決定  
リクエストヘッダにAcceptを指定することで、クライアントが期待するコンテンツタイプを指定することができる。

* 表示言語の設定
リクエストヘッダにAccept-Languageを指定することで言語を指定できる。レスポンスではヘッダの仕様としてContent-Languageがあるが、htmlタグの中で言語を指定することがほとんどである。

* 文字セットの指定
ほとんどのブラウザにて全文字セットのエンコーダを保有しているのでリクエストで指定することはほとんどない。レスポンスではContent-Typeヘッダに文字セットの情報を載せている。

* 圧縮
リクエストではAccept-Encodingで圧縮形式を指定し、サーバ側で対応されていればコンテンツが圧縮されてレスポンスが返ってくる。レスポンスのContent-Encodingヘッダに圧縮形式が示されている。なお、レスポンスのContent-Lengthヘッダの値は、圧縮後のサイズになる。

## クッキー

省略

## 認証とセッション

省略

## プロキシ

省略

## キャッシュ

### 更新日時によるキャッシュ

レスポンスヘッダにLast-Modifiedがある場合、ブラウザはURLと日時とコンテンツをキャッシュする。  
その後、キャッシュ済みのURLを再度読み込むときは、リクエストヘッダにIf-Modified-SinceヘッダにLast-Modifiedで指定された日時をそのまま入れて送信する。  
サーバはその日時をチェックしてコンテンツが変更されていれば200 OKと共にコンテンツを返す。変更がない場合は304 Not Modifiedを返してレスポンスにコンテンツを含まない形で返信する。  

### Expiresヘッダ

レスポンスヘッダにExpiresヘッダがある場合、ブラウザはヘッダに記載されている日時が過ぎるまではサーバにリクエストを送ることなく、内部にキャッシュしたデータを強制的に使うようになる。  
未来永劫変更しない予定であっても、Expiresには1年後くらいまでを限度に設定するよう推奨されている。  

### E-Tag

Last-Modifiedと似ているが、レスポンスヘッダにE-Tagを含めてブラウザにキャッシュさせ、以降のリクエストヘッダにIf-None-MatchにE-Tagの値をつけて変更の有無をチェックする方法もある。  
Last-Modifiedは日時が値として設定されるが、E-Tagについてはレスポンスのハッシュなどサーバ側に裁量が委ねられている。  

### Cache-Control

柔軟なキャッシュ制御をサーバからクライアントに指示する仕組み。Expiresヘッダよりも優先される。

* public
同一コンピュータを使う複数ユーザ間でキャッシュを再利用できる。

* private
同一コンピュータを使う別のユーザ間でキャッシュを再利用することを禁止する。

* max-age=n
キャッシュの鮮度を秒で指定する。Expiresヘッダの秒指定バージョンみたいなもの。

* no-cache
キャッシュが有効かどうか、毎回サーバに問い合わせを行う。max-age=0と同等。

* no-store
キャッシュを許可しない