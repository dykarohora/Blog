---
title: "Blob Storage間でデータを移動する with StreamAPI"
slug: "20190318_blobstorage"
author: "d_yama"
date: 2019-03-18T14:21:48+09:00
draft: false
categories: ["Azure"]
tags: ["Azure","Blob Storage","Node.js","TypeScript"]
---

# はじめに
最近はAzure Blob Storageを使ってデータを管理するようなアプリをNode.js + TypeScriptで作っていました。
そんな中で、コンテナ内のデータを別のコンテナにコピーする、という処理が必要になったので、そのとき色々考えていたことを記録しておきたいと思います。

## Blob Storageとは
[BLOB Storage](https://azure.microsoft.com/ja-jp/services/storage/blobs/)  

BLOB(バリナリラージオブジェクト)、すなわちバイナリデータをMicrosoft Azure上で管理できるクラウドストレージサービスです。

# 単純な実装
脳みそを空っぽ状態で書くとしたら、いったん対象のバイナリデータをメモリ上にロードして、別のコンテナに再度アップロードする方法があるかと追います。

Azure Node SDKを使ったBlob Storageの基本操作は[公式ドキュメント](https://docs.microsoft.com/javascript/api/overview/azure/storage?view=azure-node-latest)が参考になります。
```csharp
class BlobOperator {
    // BlobServiceの生成はよろしくやってください
    private blobService: BlobService

    public constructor(blobService: BlobService) {
        this.blobService = blobService
    }

    public copyBlob(srcContainer: string, dstContainer: string, blobId: string) {
        return new Promise<void>((resolve, reject) => {
            this.blobService.getBlobToText(srcContainer, blobId, (error, text, response) => {
                this.blobService.createBlockBlobFromText(
                    dstContainer,
                    blobId,
                    text,
                    (error, result, response) => {
                        return resolve()
                    }
                )
            })
        })
    }
}
```

これでも目的は達成できるのですが、節の頭にも書いた通り対象のバイナリデータをメモリにロードしているため、バイナリデータが大きければ大きいほどたくさんのメモリが消費されてしまいます。バイナリデータのサイズ次第ではアプリがクラッシュしてしまう、なんてこともありえます。

# Streamを使ってメモリ使用効率を改善する
この問題は、Node.jsのStream APIを使うことによって解決することができます。

## Stream APIとは
[Stream API](https://nodejs.org/api/stream.html)  

[Stream API入門](https://qiita.com/Mizunashi_Mana/items/872354cd7bf25090932f)  

Node.js上でデータをストリームとして取り扱うためのAPIです。  
ざっくり言うと、Node.jsによって生成されたStreamオブジェクトは内部にある一定の長さのBufferを持っているので、読み取ったデータはこのBufferに保存して別のStreamに書き込みが完了したらBufferをクリアして新しいデータを読み取っていく、という動きをします。  
ですので今回の場合ですと、コンテナからデータを少し読み取ったらどんどん別のコンテナにデータを書き込んでいき、書き込みが完了したデータはメモリ上からクリアして、新しいデータを読み取って順次処理していきます。アプリはこのStreamが持つ内部Buffer分だけのメモリがあればよいので、バイナリデータの大きさに比例してメモリ使用量が増えていくということはなくなります。  

## 修正したコード
Azure Node SDKにはコンテナ上のBlobをStreamとして取り扱うためのAPIが実装されています。
```csharp
class BlobOperator {
    // BlobServiceの生成はよろしくやってください
    private blobService: BlobService

    public constructor(blobService: BlobService) {
        this.blobService = blobService
    }

    public copyBlobWithStream(srcContainer: string, dstContainer: string, blobId: string): Promise<void> {
        return new Promise<void>((resolve, reject) => {
            this.blobService
                .createReadStream(srcContainer, blobId, () => {})
                .pipe(
                    this.blobService.createWriteStreamToBlockBlob(dstContainer, blobId, () => {
                        return resolve()
                    })
                )
        })
    }
```

デバッグ実行でStreamの中身を見てみたら、各StreamのBufferサイズは4MBでした。

## エラーハンドリング
前述のコードではエラー処理は記述しておりませんでした。しかし実際には読み取り元のコンテナが存在しない(Not Found)、書き込み先にすでにデータが存在する(Conflict)といったエラーが発生するかと思います。

そうするとエラー処理が必要となってきます。前述のコードで使用した関数にはコールバック関数を渡すことができるのでそこでエラー処理をしたいところですが、このコールバック関数はStreamのendイベント、もしくはflashイベントが発火されたあとに動作します。ですのでこのままだとコールバックでエラー処理をすることはできません。

ではどこでエラー処理をすればいいのかというと、Not Found等のエラーは関数実行時に例外がスローされるので`try-catch`を使うというのが一つの手です。もう一方の方法として、関数から生成したStreamのerrorイベントにリスナーを登録すると、エラー発生時にコールバック関数が動作するようになるので、空のリスナーを登録しておくというものです（もちろんリスナーの中でエラー処理してもいい）。

```csharp
class BlobOperator {
    // BlobServiceの生成はよろしくやってください
    private blobService: BlobService

    public constructor(blobService: BlobService) {
        this.blobService = blobService
    }

    public copyBlobWithErrorHandling(
        srcContainer: string,
        dstContainer: string,
        blobId: string
    ): Promise<void> {
        return new Promise<void>((resolve, reject) => {
            const srcStream = this.blobService.createReadStream(
                srcContainer,
                blobId,
                (error, result, response) => {
                    if (error) {
                        if (response.statusCode === 404) {
                            // コールバックの第三引数にAzureからのレスポンスオブジェクトが入っているので、
                            // ステータスコードに応じてエラー処理を分岐する、とかもできる
                        }
                        return reject(error)
                    }
                    return resolve()
                }
            )

            const dstStream = this.blobService.createWriteStreamToBlockBlob(
                dstContainer,
                blobId,
                (error, result, response) => {
                    if (error) {
                        // コピー元BlobのReadable Streamと同じような感じでエラー処理すればいいと思う
                        return reject(error)
                    }
                }
            )

            srcStream.pipe(dstStream)

            // Streamのerrorイベントにリスナーを登録しておくと、コールバックにエラーが渡される
            srcStream.on('error', () => {})
            dstStream.on('error', () => {})
        })
    }
}
```

自分はコールバック関数を渡す必要があるものは、できるだけその中でコンテキストが完結するようにしたいので上記の書き方をすることが多いです。ただ、そのためだけに空のイベントリスナーを登録するのはノイズ出し、初見の人にはリスナー登録部分のコードの意図が伝わらないのでベストの方法ではないな、と思っています。

# まとめ
Node.jsのStream APIを使ってAzure Blob Storage間でデータを転送する方法を紹介しました。Blob Storageではサイズが大きいデータを操作することが多いので、メモリ使用量は気をつけたいポイントです。