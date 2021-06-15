---
title: "PromiseベースのTable Storage SDK"
slug: "20210615_table-storage"
author: "d_yama"
date: 2021-06-15T21:59:00+09:00
draft: false
categories: ["Azure"]
tags: ["Table Storage"]
---

# Node SDK v12
[azure-sdk-for-js](https://github.com/Azure/azure-sdk-for-js/tree/master/sdk/tables/data-tables)のモジュールとしてAzure Table Storageを操作するためのSDKとして公開されています。このモジュールはNode.js、ブラウザの両方で動作します。

```bash
yarn add @azure/data-tables
```

## テーブル上のエンティティの操作
### 操作クライアントの作成

テーブル上のエンティティを操作するには`TableClient`オブジェクトが必要になります。接続文字列から作成する方法と、Storageアカウント名とキーから作成する方法がある。

```typescript
const connectionString = ''
const tableName = ''

const tableClient = TableClient.fromConnectionString(connectionString, tableName)
```

```typescript
const account = ''
const accountKey = ''
const tableName = ''

const credential = new AzureNamedKeyCredential(account, accountKey)
const tableClient = new TableClient(
  `https://${account}.table.core.windows.net`,
  tableName,
  credential
)
```

### エンティティの取得
`TableClient`クラスの`getEntity`メソッドから取得できる。`getEntity`のシグネチャは以下のようになっている。

```typescript
getEntity<T extends object = Record<string, unknown>>(
	partitionKey: string,
	rowKey: string,
	options?: GetTableEntityOptions
): Promise<GetTableEntityResponse<TableEntityResult<T>>>;
```

TableStorageのレコードは`PartitionKey`と`RowKey`が複合主キーとなるため、その2つを指定すれば一意なエンティティを取得できる。

型パラメータに`PartitionKey`と`RowKey`を除いたエンティティの型情報を渡すことによって、クエリの結果が片付けされて扱うことができる。

```typescript
export declare type TableEntityResult<T> = T & {
	etag: string;
    partitionKey?: string;
    rowKey?: string;
    timestamp?: string;
};
```

### クエリ
なにかしらの条件にもとづいてエンティティを取得する場合は`TableClient`クラスの`listEntities`メソッドを使用する。`listEntities`のシグネチャは以下のようになっている。

```typescript
listEntities<T extends object = Record<string, unknown>>(
	options?: ListTableEntitiesOptions
): PagedAsyncIterableIterator<TableEntityResult<T>, TableEntityResult<T>[]>;
```

引数に渡す`ListTableEntitiesOptions`型は以下のように定義されている。

```typescript
export declare type ListTableEntitiesOptions = OperationOptions & {
    queryOptions?: TableEntityQueryOptions;
    disableTypeConversion?: boolean;
};
```

`TableEntityQueryOptions`にてWHERE句に相当する条件とSLELECT句に相当する条件を指定する。

```typescript
export declare interface TableEntityQueryOptions {
    filter?: string;
    select?: string[];
}
```

`select`については必要なカラム名だけを指定すればよいのですが、`filter`については[OData filter](https://docs.microsoft.com/ja-jp/graph/query-parameters#filter-parameter)形式で記述する必要があります。`odata`関数という[[Tagged Template Literals]]を利用した関数が用意されているため、そちらを使ってクエリを記述するのがオススメです。

```typescript
const pk1 = 'pk-A'
const pk2 = 'pk-B'
const query = odata`PartitionKey eq ${pk1} or PartitionKey eq ${pk2}`
// queryは「PartitionKey eq 'pk-A' or PartitionKey eq 'pk-B'」
```

なお、もう一つのオプションである`disableTypeConversion`プロパティについてですが、こちらを`true`に設定すると、プロパティはJavaScript組み見込み型に型変換されず、文字列形式の値と型タイプのRecordの状態でオブジェクトが返されます。
例えば、あるカラムがInt32の値であった場合、`{value: "123", type: Int32"}`となります。このオプションはデフォルトでは`false`となっています。

戻り値は`PagedAsyncIterableIterator`型であり、これはAsyncIterableなオブジェクトである。したがって`for await`を使うことによって、取得できたエンティティを列挙することができる。

```typescript
export interface PagedAsyncIterableIterator<T, PageT = T[], PageSettingsT = PageSettings> {
  next(): Promise<IteratorResult<T, T>>;
  [Symbol.asyncIterator](): PagedAsyncIterableIterator<T, PageT, PageSettingsT>;
  byPage: (settings?: PageSettingsT) => AsyncIterableIterator<PageT>;
}
```

また、このAsyncIterableなオブジェクトは遅延評価がなされるため、実際に値の取り出しが評価されるまでTableStorageへのHTTPリクエストは送信されない。
TableStorageはエンティティのリストを取得する場合、1回のHTTPリクエストで最大1000件までしか取得できない制約がある。1000件を超える場合はページングされるため、レスポンスヘッダを確認して再度HTTPリクエストを送信して後続のエンティティを取得する必要がある。このSDKでは1000件を超えるエンティティを取得しようとしたとき、AsyncIteratorから1001件目のエンティティを取り出そうとしたタイミングでHTTPリクエストが送信されるよう設計されている。
旧来の[azure-storage](https://www.npmjs.com/package/azure-storage)の場合は、Continuation Tokenを使って自分で後続ページを取得する必要があったが、本SDKでは`listEntities`を使うことによってページングについては透過的に扱うことができる。

また、前述のように1件ずつIteratorResultを得る以外にも`PagedAsyncIterableIterator`オブジェクトの`byPage`メソッドを使えば、ページ単位でのIteratorResultを取得することもできる。

### 挿入
`TableClient`クラスの`createEntity`メソッドを使うことによってエンティティの挿入ができる。

```typescript
createEntity<T extends object>(
	entity: TableEntity<T>,
	options?: OperationOptions
): Promise<CreateTableEntityResponse>;
```

挿入するエンティティは`TableEntity<T>`型となっており、その定義は以下のようになっている。

```typescript
export declare type TableEntity<T extends object = Record<string, unknown>> = T & {
    partitionKey: string;
    rowKey: string;
};
```

旧来の[azure-storage](https://www.npmjs.com/package/azure-storage)では、ヘルパーメソッドを使ってエンティティのプロパティをそれぞれ専用の型に変換する必要があったが、本SDKでは組み込み型をそのまま使うことができる。数値型だけ注意する必要があって、Int32として扱う場合は`number`型として、Int64として扱う場合は`BigInt`型として数値を設定する必要がある。

なお、`createEntity`メソッドでは、エンティティに設定したpartitionKeyとrowKeyの組が既にテーブルに存在する場合は例外がスローされる。

### 更新
`TableClient`クラスの`updateEntity`メソッドを使うことによってエンティティの更新ができる。

```typescript
updateEntity<T extends object>(
	entity: TableEntity<T>,
	mode?: UpdateMode,
	options?: UpdateTableEntityOptions
): Promise<UpdateEntityResponse>;
```

使い方は`createEntity`メソッドとほぼ同様ではあるが、特徴的なのは第二引数の`UpdateMode`である。TableStorageはエンティティの更新方法として`Merge`と`Replace`の二つを用意している。`Merge`の場合は引数に与えられたエンティティ内に存在するプロパティだけが更新される。引数に与えられたエンティティ内に存在しないプロパティについては、更新前の値そのままとなる。一方`Replace`はその名の通り置き換えとなる。引数に与えられたエンティティ内に存在しないプロパティについては、置き換え後はそのカラムは**null**として記録される。

`updateEntity`の類似メソッドとして`upsertEntity`が存在するが、こちらは該当するエンティティが存在しない場合は挿入操作を行うメソッドとなる。（`updateEntity`メソッドは該当するエンティティが存在しない場合は例外をスローする）

### 削除
`TableClient`クラスの`deleteEntity`メソッドを使う。主キーであるPartitionKeyとRowKeyを指定して使用する。該当するエンティティが存在しない場合は例外をスローする。

```typescript
deleteEntity(
	partitionKey: string,
	rowKey: string,
	options?: DeleteTableEntityOptions
): Promise<DeleteTableEntityResponse>;

```

### エンティティグループトランザクション

Table Storageには、同一パーティションにあるエンティティに対してアトミックなトランザクションを実現する[エンティティグループトランザクション](https://docs.microsoft.com/ja-jp/azure/storage/tables/table-storage-design#entity-group-transactions)という機能があります。同一パーティションに存在するエンティティに限る、1回のトランザクションで操作できるエンティティの数は100までといった制限はありますが、APIのコール数1回で処理が完了するため何度もAPIを叩いて複数件のエンティティを処理するよりもはるかに早いです。またアトミック性も保証されているため、要件にマッチするなら積極的に使っていきたい機能です。[azure-storage](https://www.npmjs.com/package/azure-storage)でも利用できた機能ですが、本SDKでも`TableClient`クラスの`submitTransaction`メソッド経由で使うことができます。

```typescript
submitTransaction(actions: TransactionAction[]): Promise<TableTransactionResponse>;
```

引数に渡す`TransactionAction[]`ですが、どんなオブジェクトになるかは例を見るのが早いです。

```typescript
const actions: TransactionAction[] = [
   ["create", {partitionKey: "p1", rowKey: "1", data: "test1"}],
   ["delete", {partitionKey: "p1", rowKey: "2"}],
   ["update", {partitionKey: "p1", rowKey: "3", data: "newTest"}, "Merge"]
```

アクションとして含ませることができるのは、`create`、`delete`、`update`、`upsert`の4つです。`update`と`upsert`については`Merge`か`Replace`かも指定することができます。

