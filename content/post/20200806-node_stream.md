---
title: "[Node.js] pipeを使うときに気をつけること"
slug: "20200806-node_stream"
author: "d_yama"
date: 2020-08-10T15:38:06+09:00
draft: false
categories: ["Node.js"]
tags: ["TypeScript", "Stream"]
---

# はじめに

[Node.jsのストリーム](https://nodejs.org/api/stream.html)は大容量のファイルやBlobをメモリ面で効率よく使いたいときにとても便利です。Streamは内部に固定長のバッファを持つことによって、バッファがあふれないようににデータの読み込み/書き込みを調整してくれます。

また、`pipe()`メソッドを使うことによってストリーム通しを繋げてパイプラインを簡単に作ることができます。簡単に作ることはできるのですが挙動を把握しておかないと、プロセスがくらっしゅしてしまったりメモリリークの原因となってしまうので注意が必要です。

今回は`pipe()`メソッドを使うときの注意点をtipsとしてまとめます。

# pipeをつかったときのストリームの挙動

サンプルとして、「サーバにHTTP接続すると、サーバ上にあるファイルをダウンロードできる」というものを用意してみました。HTTPサーバは`http`モジュールを使います。レスポンスである`ServerResponse`はwritableストリームなので、`fs`モジュールでファイルのreadableストリームを作り、これらを`pipe()`メソッドで繋げます。

```typescript
import http from 'http'
import fs from 'fs'

const filename = process.argv[2]

const server = http.createServer((req, res) => {
    const stream = fs.createReadStream(filename)
    stream.pipe(res)
})

server.listen(3000, () => console.log(`http://localhost:3000`))
```

正常系についてはこれだけでも十分に動作します。しかし異常系についてはプロセスがクラッシュしてしまいます。

というのも、`Stream`は`EventEmitter`を継承しています。そして`EventEmitter`において、`error`イベントに対するリスナーが登録されていない状態で`error`イベントが発生した場合、Node.jsはスタックトレースを出力してプロセスを終了させてしてしまいます。試しに[`readable.destryo()`](https://nodejs.org/api/stream.html#stream_readable_destroy_error)を使って、Readableストリームで`error`イベントを発生させてみます。

```typescript
const server = http.createServer((req, res) => {
    const stream = fs.createReadStream(filename)
    stream.pipe(res)
    stream.destroy(new Error('エラーを意図的に起こす'))
})

// サーバにHTTP接続すると、以下のスタックトレースを吐き出してプロセスが終了する
// Error: エラーを意図的に起こす
//     at Server.<anonymous> 
//     at Server.emit (events.js:210:5)
//     at parserOnIncoming (_http_server.js:745:12)
//     at HTTPParser.parserOnHeadersComplete (_http_common.js:115:17)
// [ERROR] 15:30:55 Error: エラーを意図的に起こす
```

プロセスが死にました。このままではサーバの役割を果たせないので、Readableストリームにエラーイベントに対するリスナーを追加します。

```typescript
const server = http.createServer((req, res) => {
    const stream = fs.createReadStream(filename)
    stream.on('error', (error) => {
        console.log(`${error}`)
        // some error handling...
    })
    stream.pipe(res)
    stream.destroy(new Error('エラーを意図的に起こす'))
})
```

これでReadableストリームでエラーが発生してもプロセスが落ちることはなくなりました。しかしまだ問題があります。ブラウザからこのサーバに接続すると、サーバ側ではエラーイベントをハンドリングはするのですが、ブラウザから見るとレスポンスがいつまでたっても返ってきません。

この事象が発生する原因は[ドキュメントに書かれています](https://nodejs.org/api/stream.html#stream_readable_pipe_destination_options)。`pipe()`を使ってストリームを繋げたとき、Redableストリーム側で読み込みが終わった、すなわち`end`イベントがemitされると接続されているwritableストリームの`end()`をオートでコールしてくれます。ただし、`error`イベントが発生した場合はその限りではなく、writableストリームは開きっぱなしなので手動で閉じてあげる必要があります。

```typescript
const server = http.createServer((req, res) => {
    const stream = fs.createReadStream(filename)
    stream.on('error', (error) => {
        console.log(`${error}`)
        res.statusCode = 500
        res.end('500 error')
    })
    stream.pipe(res)
    stream.destroy(new Error('エラーを意図的に起こす'))
})
```

これでエラー発生時に500エラーをブラウザに返してくれます。

ストリームはとても便利なものですが、エラーハンドリングを疎かにするとアプリのクラッシュに繋がるので注意しましょう。