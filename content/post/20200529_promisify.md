---
title: "Azure StorageのNode.js用SDKのメソッドをPromise化する"
date: 2020-05-30T11:35:10+09:00
slug: "20200529-promisify"
author: "d_yama"
draft: false
categories: ["Azure"]
tags: ["TypeScript", "Azure", "Azure Blob Storage", "Azure Table Storage"]
---

# いまどきコールバックはつらい

[Promiseは必要](https://youtu.be/M3BM9TB-8yA)

Azure StorageをNode.jsから使うときには[こちらのazure-storageというnpmパッケージ](https://www.npmjs.com/package/azure-storage)を利用することが多いのですが、APIを実行するメソッドはコールバックスタイルのものしか提供されていません。要はPromiseを使えない、async/awaitも使えない状態です。（[後発のSDK](https://github.com/Azure/azure-sdk-for-js/tree/master/sdk/storage)ではPromiseスタイルのメソッドが提供されていますが、いまだTable Storageはサポートされていないのが悩ましい…。）

そこで、Node.jsが提供する[util.promisify](https://nodejs.org/dist/latest-v12.x/docs/api/util.html#util_util_promisify_original)という、コールバックスタイルの関数をPromiseを返す関数に変換してくれるユーティリティ関数を使って、azure-storageの各メソッドをPromiseをリターンする形に変換してそれを使うようにしています。

なお、言語はTypeScriptでエディタは[WebStorm](https://www.jetbrains.com/webstorm/)での話となります。

# まずは使ってみる

基本、`promisify`の引数に対象の関数オブジェクトを渡してあげるだけでよいのですが、BlobServiceやTableServiceのメソッドを変換するときは注意が必要で、`promisify`から返ってきた関数に対しbindでthisの参照先を元のオブジェクトに指定しなければいけません。

```typescript
const storageAccountName = 'xxx'
const storageAccountKey = 'xxxxx'

const blobService: BlobService = 
    azureStorage.createBlobService(storageAccountName, storageAccountKey)
// 変換で得られた関数にbindを使ってthisの参照先を明示的に指定する
const createContainerFunc = promisify(blobService.createContainer).bind(blobService)
```

BlobServiceやTableServiceのメソッドは内部でthisを使用しています。また、[promisifyの実装](https://github.com/nodejs/node/blob/master/lib/internal/util.js#L277)を見てみると、引数で渡された関数オブジェクトをPromiseでラップしたようなものを返しています。すなわち`promisify`にメソッドを渡して得た関数はBlobServiceやTableServiceのメソッドではなくなってしまうため、bindでthisの参照先を指定してあげないと実行時にエラーを吐いてしまいます。

変換後はPromiseライクに任意のタイミングでawaitしたりすることが可能です。try-catchを使ってエラーレスポンスを補足することも可能です。

```typescript
try {
    const task = createContainerFunc('new-container')
    // 何か他の仕事を進めたり
    await task
} catch(error) {
    // 指定したコンテナは既に存在したときとか
}
```

# オーバーロードメソッドもPromise化したい

TypeScriptではメソッドのオーバーロードが可能です。前節で例に使ったBlobコンテナを作成するcreateContainerメソッドにもオーバーロードがあります。

```typescript
// オプションなし
createContainer(
    container: string, 
    callback: ErrorOrResult<BlobService.ContainerResult>
): void;

// オプションあり
createContainer(
    container: string, 
    options: BlobService.CreateContainerOptions, 
    callback: ErrorOrResult<BlobService.ContainerResult>
): void;
```

その一方、前節でPromiseを返す形に変換したcreateContainerのType Infoを見てみると引数が一つしかありません。
![extension](/image/20200529_ws001.png)

このメソッドの第二引数にオブジェクトを渡そうとするとコンパイラの静的チェックにてエラーが出力されます。
![extension](/image/20200529_ws002.png)

このようなときは、ジェネリクスな`promisify`を使うことによって解決できます。`promisify`の型定義を見てみると次のようになっています。(一部だけ抜粋)

```typescript
function promisify<T1, TResult>
    (fn: (arg1: T1, callback: (err: any, result: TResult) => void) => void): (arg1: T1) => Promise<TResult>;
function promisify<T1>
    (fn: (arg1: T1, callback: (err?: any) => void) => void): (arg1: T1) => Promise<void>;

function promisify<T1, T2, TResult>
    (fn: (arg1: T1, arg2: T2, callback: (err: any, result: TResult) => void) => void): (arg1: T1, arg2: T2) => Promise<TResult>;
function promisify<T1, T2>
    (fn: (arg1: T1, arg2: T2, callback: (err?: any) => void) => void): (arg1: T1, arg2: T2) => Promise<void>;

    // (中略)

function promisify(fn: Function): Function;
```

定義を見て分かるとおり、型パラメータに入力となる関数の引数の型と戻り値の型を指定することができます。オーバーロードのシグネチャに合わせて型パラメータを指定してあげることによってpromiseスタイルのオーバーロード関数を得ることができます。

```typescript
const createContainerFunc =
    promisify<string, BlobService.CreateContainerOptions, BlobService.ContainerResult>(blobService.createContainer)
        .bind(blobService)

const option: BlobService.CreateContainerOptions = {/** metadataとか指定する **/}
await createContainerFunc('new-container', option)
```

# 仮引数名をもっと表現力豊かにしたい
これはちょっとした小ネタですが、`promisefy`で変換した関数のType Infoを見てみると、仮引数名が<i>arg1, arg2...</i>と機械的なものとして出てきます。型さえハッキリしていれば困ることはあまりないのですが、それでもちゃんと意味が分かる仮引数名であったほうがストレスは少ないです。

![extension](/image/20200529_ws003.png)

そんなときは変換した関数をキャッシュする変数に型アノテーションをつけることによって解決できます。

![extension](/image/20200529_ws004.png)

# Table Storageへのクエリ

Table Storageからクエリを使ってレコードを取得するメソッドとして、queryEntitiesメソッドがあります。これを`promisify`に通してみると、

![extension](/image/20200529_ws005.png)

このように戻り値から取得できるレコードの型の推論結果はunknownとなっています。

もともとqueryEntitiesメソッドはジェネリクスであり、型パラメータにレコードの型を指定すると、戻り値内にあるレコードにも型付けしてくれます。

```typescript
queryEntities<T>(
    table: string, 
    tableQuery: TableQuery, 
    currentToken: TableService.TableContinuationToken, 
    options: TableService.TableEntityRequestOptions, 
    callback: ErrorOrResult<TableService.QueryEntitiesResult<T>>
): void;
```

となると、Promiseスタイルに変換したときもレコードに対してunknown型ではなく意図した型付けをしたくなります。こちらも`promisify`の型パラメータによって型情報を持たせることができます。

```typescript
interface PeopleEntity {
    PartitionKey: EntityProperty<string>
    RowKey: EntityProperty<string>
    Name: EntityProperty<string>
    Age: EntityProperty<number>
}

const queryPeopleEntitiesFunc = promisify<
    string,
    TableQuery,
    TableService.TableContinuationToken,
    // 戻り値の型がジェネリクスなので、ここにレコードの型を指定してあげる
    TableService.QueryEntitiesResult<PeopleEntity>  
>(tableService.queryEntities).bind(tableService)
```

これによってpromiseスタイルに変換した関数の戻り値を型付けすることができます。

# まとめ

ジェネリクス版promisifyを使ってazure-storageのメソッドをタイプセーフな形でPromiseスタイルに変換する方法をまとめました。

# 参考

[Promise 対応していないコールバック形式のライブラリーを Promise にしたい - かずきのBlog@hatena](https://blog.okazuki.jp/entry/2019/09/11/202906)