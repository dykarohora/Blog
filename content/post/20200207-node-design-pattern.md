---
title: "20200207 Node Design Pattern"
date: 2020-02-07T17:55:37+09:00
draft: true
---

# Ch.1 Node.jsの世界へようこそ

## ノンブロッキングI/O

# Ch.2 Node.jsの基本パターン

## コールバックパターン

Node.jsでは非同期処理を実現するためにコールバック関数を必要とするAPIが多く存在する。  
APIにコールバックを渡すとイベントループ内のキューにコールバック関数が挿入され、イベント発生時にキューの中からコールバック関数が取りだされ実行される。  

### 関数定義において、同期と非同期を混在させるのは危険

```javascript
const fs = require('fs')
const cache = {}

function inconsistentRead(filename, callback) {
    if(cache[filename]) {
        // キャッシュにデータが存在する場合は同期的にコールバックが実行される
        callback(cache[filename])
    } else {
        // キャッシュにデータが存在しない場合は非同期的にファイルを読み取り、読み取り完了後にコールバックを実行する
        fs.readFile(filename, 'utf8', (err, data) => {
            cache[filename] = data
            callback(data)
        })
    }
}

function createFileReader(filename) {
    const listeners = []
    // ファイルの読み取りが完了したら、リスナーを走査してコールする
    inconsistentRead(filename, value => {
        listeners.forEach(listener => listener(value))
    })

    // クロージャとして保持しているリスナー配列にリスナーを追加する関数を返す
    return {
        onDataReady: listener => listeners.push(listener)
    }
}

// ファイルの読み取り、キャッシュが存在しないので非同期的に読み込まれる
const reader1 = createFileReader('tsconfig.json')
// 読み取り完了後のコールバックを定義
reader1.onDataReady(data => {
    // 最初のcreateFileReaderは非同期で読み取られるので、このコールバックは実行される
    console.log('First call data: ' + data)
    // 2回目のファイル読み取り
    // この時点でキャッシュにはデータが存在するので同期的に処理が進む
    // ただし、2回目のcreateFileReaderはリスナーが空なので、何も処理は発生しない
    const reader2 = createFileReader('tsconfig.json')
    // リスナーにコールバックを追加しても、この時点ではすでにファイル読み取りが完了しているので、このコールバックに処理が渡ることはない
    reader2.onDataReady(data => {
        console.log('Second call data: ' + data)
    })
})
```

非同期処理と同期処理が混在していると実行時の振る舞いが分かりにくくなるので避けるべき。  
解決策としては同期処理だけに寄せてしまうというのが考えられる。ここでいうところのファイル読み取りをfs.readFileSync関数に変更することが該当する。  もしくは非同期処理に寄せてしまうというのでも解決できる。キャッシュが存在する時のコールバック呼び出しをprocess.nextTickに渡せば、コールバックはnextTickキューに入っていくので、コードを全て実行したあとのイベントループ開始時に遅延実行させることができる。

### Node.jsのコールバックについてのお作法

* コールバックを渡す引数は関数の最後の引数とする
* コールバック関数の先頭の引数はエラーオブジェクトとする
* エラーはスローしないでコールバック関数にエラーオブジェクトを渡す

```javascript
const fs = require('fs')

function readJsonThrow(filename, callback) {
    fs.readFile(filename, 'utf8', (err, data) => {
        if (err) {
            return callback(err)
        }
        callback(null, JSON.parse(data))
    })

}

try {
    readJsonThrow('test.txt', (err, json) => {
        if (err) {
            console.log(err)
        } else {
            console.log(JSON.stringify(json))
        }
    })
} catch {
    // JSON.parseの失敗をキャッチできない
    // readJsonThrowのコールスタックとコールバック関数のコールスタックは異なっている
    console.log('catch')
}
```

上記のコードだと、JSON.parseでのエラーを補足することができない。補足するためには次のように修正する必要がある。

```javascript
const fs = require('fs')

function readJsonThrow(filename, callback) {
    fs.readFile(filename, 'utf8', (err, data) => {
        if (err) {
            return callback(err)
        }

        try {
            callback(null, JSON.parse(data))
        } catch (error) {
            return callback(error)
        }
    })

}

readJsonThrow('test.txt', (err, json) => {
    if (err) {
        console.log(err)
    } else {
        console.log(JSON.stringify(json))
    }
})
```

## モジュール定義のパターン

[モジュールシステムの基本は知っている前提](https://hololab.esa.io/posts/229)

### オブジェクトのエクスポート

exportsオブジェクトのプロパティに公開したい関数や変数を追加していく手法。

```javascript
// module
exports.info = (message) => {
    console.log(`info: ${message}`)
}

exports.verbose = (message) => {
    console.log(`verbose: ${message}`)
}

// client
const logger = require('./logger')
logger.info('information') // info: information
logger.verbose('verbose')  // verbose: verbose
```

exportsオブジェクトが名前空間のような役割をはたす。Nodeのコアモジュールはこの形式が多い。

ちなみにTypeScriptの場合は以下のコードが同等となる（TypeScript 3.7.5）。

```typescript
// module
export function info(message: string) {
    console.log(`info ${message}`)
}

export function verbose(message: string) {
    console.log(`verbose ${message}`)
}

// client
import * as logger from './logger'

logger.info('information')
logger.verbose('verbose')
```

### 関数のエクスポート

module.exportsプロパティに関数オブジェクトを指定する手法。

```javascript
// module
module.exports = (message) => {
    console.log(`info: ${message}`)
}

module.exports.verbose = (message) => {
    console.log(`verbose: ${message}`)
}

// client
const logger = require('./ch2-function_export')

logger('information')
logger.verbose('verbose')
```

モジュールの主要な機能と副となる機能を分離して公開できる。

TypeScriptの場合だと以下のような感じ。

```typescript
// module
export default function logger(message: string) {
    console.log(`information: ${message}`)
}

logger.verbose = function(message: string) {
    console.log(`verbose: ${message}`)
}

// client
import logger from './logger_function'

logger('information')
logger.verbose('verbose')
```

export defaultはクライアント側でモジュール側とは違う名前をつけることができてしまい、参照を辿るのが面倒になりがちなのであまり好きではない。

### コンストラクタのエクスポート

```javascript
// module
function Logger(name) {
    this.name = name
}

Logger.prototype.log = function (message) {
    console.log(`information: ${message}`)
}

Logger.prototype.verbose = function (message) {
    console.log(`verbose: ${message}`)
}

module.exports = Logger

// client
const Logger = require('./ch2-constructor_export')

const logger = new Logger('example')

logger.log('information')
logger.verbose('verbose')
```

module.exportsプロパティにコンストラクタ関数を指定すればprototypeに追加された関数もクライアント側で呼び出すことができる。クライアント側ではC#ライクに使える。  
[prototypeについてはこちら](https://hololab.esa.io/posts/363)

TypeScriptならもっとC#っぽっく書ける。

```typescript
// module
export class Logger{
    constructor(private name: string) {
    }

    public log(message: string) {
        console.log(`information: ${message}`)
    }

    public verbose(message: string) {
        console.log(`verbose: ${message}`)
    }
}

// client
import {Logger} from './logger_class'

const logger = new Logger('example-ts')

logger.log('information')
logger.verbose('verbose')
```

classはES2015からJavaScriptでも使える。

### インスタンスのエクスポート

コンストラクタ関数ではなく、コンストラクタ関数から生成したインスタンスをmodule.exportsに指定するといった手法も見受けられる。利点としては、Nodeでは一度ロードしたモジュールはキャッシュされるので、インスタンスを公開したモジュールはシングルトンのように活用することができる。しかし、モジュールの依存関係の都合上、同一モジュールの複数のバージョンがプロジェクトで利用される場合は、シングルトンが保証されるわけではないので注意が必要。個人的にはシングルトンとして利用するかどうかはクライアントに委ねるべきだと思うので、ライブラリとして作ったモジュールではインスタンスではなく、コンストラクタをエクスポートするのが良いと思う。

## オブザーバーパターン

オブザーバーパターンを実現する仕組みとしてNode.jsではコアモジュールにEventEmitterというクラスがある。

```javascript
const EventEmitter = require('events').EventEmitter
const fs = require('fs')

class FindPatter extends EventEmitter {
    constructor(regex) {
        super()
        this.regex = regex
        this.files = []
    }

    addFile(file) {
        this.files.push(file)
        return this
    }

    find() {
        this.files.forEach(file => {
            fs.readFile(file, 'utf8', (err, content) => {
                // 読み込みに失敗したらerrorイベントをemit
                if (err) return this.emit('error', err)

                // ファイル読み込みが完了したらfilereadイベントをemit
                this.emit('fileread', file)
                const match = content.match(this.regex)
                if (match) {
                    // matchの分だけfoundイベントをemit
                    match.forEach(elem => this.emit('found', file, elem))
                }
            })
        })
        return this
    }
}

const findPatterObj = new FindPatter(/hoge/g)

findPatterObj
  .addFile('test.txt')
  .find()
  // foundイベントのリスナー
  .on('found', (file, match) => console.log(`Matched "${match}" in file ${file}`))
  // errorイベントのリスナー
  .on('error', (err) => console.error(err))
```

上の例だとイベントのemitはファイル読み込み(fs.readFile)のコールバックから行われるので、イベントループにてemit→リスナーの実行は行われる。

### EventEmitterのAPIは同期的

イベント＝非同期みたいなイメージがあるけどそんなことない。イベントの仕組みと非同期制御はまったく別の話。

```javascript
const EventEmitter = require('events').EventEmitter

class SyncEventer extends EventEmitter {
    constructor() {
        super()
    }

    fireEvent(event) {
        this.emit(event)
    }
}

const eventEmitter = new SyncEventer()

eventEmitter.fireEvent('hoge')
eventEmitter.on('hoge', () => console.log('hoge emitted'))
```

例えば上のコードを実行してみると、標準出力には何も表示されない。これは`fireEvent('hoge')`を実行したタイミングでは`hoge`イベントのリスナーは何も登録されていないため。次行のリスナー登録時にすでにイベントはemit済みなので、`on`に渡した関数が実行されることはない。これを直すためにはリスナーの登録をイベントのemit前にすればよい。

```javascript
const EventEmitter = require('events').EventEmitter

class SyncEventer extends EventEmitter {
    constructor() {
        super()
    }

    fireEvent(event) {
        this.emit(event)
    }
}

const eventEmitter = new SyncEventer()

eventEmitter.on('hoge', () => console.log('hoge emitted'))
eventEmitter.fireEvent('hoge') // hoge emitted
```

もしくは`fireEvent`メソッドでのイベントの発火を`setImmediate`や`nextTick`で非同期でイベントループ上で行うようにしてもよい。

```javascript
const EventEmitter = require('events').EventEmitter

class AsyncEventer extends EventEmitter {
    constructor() {
        super()
    }

    fireEvent(event) {
        setImmediate(() => this.emit(event))
    }
}

const eventEmitter = new AsyncEventer()

eventEmitter.on('hoge', () => console.log('hoge emitted'))
eventEmitter.fireEvent('hoge') // hoge emitted
```

Node.jsのコンテキスト的にはこっちの方が合ってる気がする。

## EventEmitterを使うかコールバックパターンを使うか

それぞれ動作自体にはそれほど違いはない。ただ、特徴はそれぞれ異なっているので綺麗な設計を意識するのならば把握しておきたい

* コールバックパターンが向いている
  * コールバックの発生が1回だけ
  * 必要となるコールバックでの処理は1種類
  * コールバックに伝える非同期処理の結果が失敗か成功の2種類だけ

* EventEmitterが向いている
  * 同じイベントが複数回発生する
  * 1つのイベントに対して必要となるコールバックが動的に変わる
  * イベント（＝非同期処理中に発生したこと）の数が複数ある

# Ch.3 コールバックを用いた非同期パターン

以下の原則を守って実装する。

* if文にはelseを使わずに即時returnする。
* コールバックでは独立した関数として定義する。クロージャを避ける。
* 一つの関数にネストしたコールバックを記述せずに複数の関数に分割する。

## 非同期処理の逐次処理

複数の非同期処理を順番に実行する。基本的なアイデアは以下の通り。

```typescript
function task1(callback: Function) {
    console.log('task1 start')
    setTimeout(() => {
        console.log('task1 completed')
        task2(callback)
    }, 250)
}

function task2(callback: Function) {
    console.log('task2 start')
    setTimeout(() => {
        console.log('task2 completed')
        task3(callback)
    }, 500)
}

function task3(callback: Function) {
    console.log('task3 start')
    setTimeout(() => {
        console.log('task3 completed')
        callback()
    }, 1000)
}

task1(() => console.log('all tasks completed'))
```

決められた順番で非同期処理を実行したい場合は、処理の完了時に次の非同期処理を呼び出すようにすればよい。見て分かる通り、次の非同期処理を呼び出す時にクライアントから渡されたコールバックを引数に渡しているのでクロージャになっていない。  

これを少し一般化すると

```typescript
function iterateTasks(collection: Function[], finalCallback: Function) {
    function inner(index: number) {
        if (index === collection.length) {
            return finalCallback()
        }
        const task = collection[index]
        task(() => inner(index + 1))
    }

    inner(0)
}
```

第一引数に渡すのが非同期処理のリストで、リストの並び順に逐次処理されることを期待している。第二引数が逐次処理完了後のコールバック。内部関数で引数に渡されたリストをラップし、非同期処理を実行する。非同期処理終了後に内部関数を再帰で呼び出している。再帰の停止ポイントは非同期処理のリストが全て処理されたときとなる。

実行サンプルは以下の通り。

```typescript
function asyncOperation(callback: Function) {
    console.log('task start')
    setTimeout(() => callback(), 1000)
}

const tasks = [
    asyncOperation,
    asyncOperation,
    asyncOperation
]

iterateTasks(tasks, () => console.log('complete all tasks'))
```

## 非同期処理の並行処理

非同期処理の並行処理において、すべての処理が完了したことを待ち受けたいときは対象の非同期処理の数をキャッシュしておき、各非同期処理のコールバックにてカウンタをインクリメントし、かつカウンタが非同期処理の数に到達したかをチェックすればよい。

```typescript
function concurrent(collection: Function[], finalCallback: Function) {
    const taskNum = collection.length
    let completed = 0
    collection.forEach(task => task(() => {
        if (++completed === taskNum) {
            finalCallback()
        }
    }))
}

function asyncOperation(callback: Function) {
    console.log('task start')
    setTimeout(() => {
        console.log('task completed')
        callback()
    }, Math.random() * 5 * 1000)
}

const tasks = [
    asyncOperation,
    asyncOperation,
    asyncOperation
]

concurrent(tasks, () => console.log('all task completed'))
```

Node.jsはシングルスレッドで動作するため、スレッド同期についてのレースコンディションの問題は発生しにくい。しかし、ファイルI/OやネットワークI/Oが絡むと無駄な処理が発生したりクラッシュの原因となる可能性があるので注意したい。

## 同時実行数を制限した並行処理

並行処理を実行するときに同時実行数を際限なく増やすと、リソースを喰い潰してしまい過負荷状態になってしまう可能性がある。それを回避するアイデアとして、同時実行数を制限することが考えるが、以下のようにして実装できる。

* 最小は同時実行数の最大限までタスクを起動する
* タスクが完了するたびに制限いっぱいまで残りのタスクを起動する
* タスクが全て完了するまで繰り返す

```typescript
function oneSecTimer(callback: Function) {
    console.log('task start')
    setTimeout(() => callback(), 1000)
}

const taskList = [
    oneSecTimer,
    oneSecTimer,
    oneSecTimer
]


const concurrency = 2
let running = 0, completed = 0, index = 0
function next() {
    while(running < concurrency && index < taskList.length) {
        const task = taskList[index++]
        task(() => {
            completed++

            if(completed === taskList.length) {
                return finish()
            }

            running--
            next()
        })
        running++
    }
}

function finish() {
    console.log('complete all tasks')
}

next()
```

# Ch.4 ES2015以降の機能を使った非同期パターン

## Promise

基本的なことは[こちら](https://hololab.esa.io/posts/427)

### コールバックパターンの非同期処理をPromise化する

自分で実装すると次のようになる

```javascript
'use strict'

module.exports.promisify = function (callbackBasedApi) {
    return function promisified() {
        // APIはここから実行される。
        // APIの引数を配列に分解
        const args = [].slice.call(arguments)

        return new Promise((resolve, reject) => {
            // args配列にAPI実行後のコールバック関数をつっこむ
            args.push(function (err, result) {
                // APIがエラーを返してきた場合はreject
                if (err) {
                    return reject(err)
                }
                // コールバック関数の結果が二つ(errとresult)ならば、
                // fulfilledでresultを返す
                if (arguments.length <= 2) {
                    resolve(result)
                    // 3つ以上ある場合はerrを取り除いて残り全てをfulfilledで返す
                } else {
                    resolve([].slice.call(arguments, 1))
                }
            })
            // 元のAPIを実行、Promise
            callbackBasedApi.apply(null, args)
        })
    }
}
```

利用側は以下の通り

```javascript
const fs = require('fs')
const utilities = require('./utilities')
const readFile = utilities.promisify(fs.readFile)

readFile('./test.txt', 'utf8')
  .then(data => console.log(data))
  .catch(err => console.log(err))
```

また、Node.js v8より`util`パッケージに`promisify`関数が追加されたので、これを使っても同じことができる。

### Promiseでの逐次処理

`Promise`でラップした処理が完了すると、`then`メソッドに渡した関数が実行される。また`then`メソッドは`Promise`オブジェクトを返すので、`then`を複数つないでいくことができる。これをプロミスチェーンと呼ぶ。

```typescript
function timer(sec: number): Promise<void> {
    return new Promise((resolve) => {
        setTimeout(() => resolve(), sec * 1000)
    })
}

// 1秒,2秒,3秒のタイマー
const timerTasks = [
    timer.bind(null, 1),
    timer.bind(null, 2),
    timer.bind(null, 3)
]

let promise = Promise.resolve()
timerTasks.forEach(task => {
    promise = promise.then(() => {
        console.log('task complete')
        return task()
    })
})

promise.then(() => {
    console.log('all tasks competed')
})
```

各非同期処理を`then`でつないでいけば、そのプロミスチェーンはタスクリストは逐次処理で実行される。プロミスチェーンの頭に`Promise.resolve()`を用意しているが、こちらは生成が完了した段階でFulfilled状態のpromiseオブジェクトを返す静的メソッドである。先頭のタスクを取り出してチェーンをつないでもよいが、forループを避けるために使っている。また、reduceを使っても良い。

```typescript
timerTasks
  .reduce((prev, task) => prev.then(() => task()), Promise.resolve())
  .then(() => {
      console.log('all tasks completed')
  })
```

### Promiseでの並行処理

基本は`Promise.all`を使えばよい。`Promise.all`にPromiseのリストを渡すと、リスト内のPromiseオブジェクトが全てFulfilled状態になったときに、Fulfilled状態となるPromiseオブジェクトを返してくれる関数である。

```typescript
Promise.all(timerTasks)
  .then(() => console.log('all tasks completed'))
```

### Promiseでの同時実行制御

NodeのPromiseに非同期処理の同時実行数を制御する機能はないので自作するしかない。基本のアイデアはコールバックパターンのときのものと同様。

```typescript
export class PromiseQueue {

    private running: number
    private queue: Function[]

    constructor(private concurrency: number) {
        this.running = 0
        this.queue = []
    }

    public pushTask(task: Function) {
        this.queue.push(task)
        this.next()
    }

    public next() {
        while(this.running < this.concurrency && this.queue.length) {
            const task = this.queue.shift()
            task!().then(() => {
                this.running--
                this.next()
            })
            this.running++
        }
    }
}
```

異なるのはタスクの実行完了時の処理を`then`のなかで実行するだけ、というもの。使う側は以下の通りとなる。

```typescript
import {PromiseQueue} from './taskQueue'

function timer(sec: number): Promise<void> {
    return new Promise((resolve) => {
        setTimeout(() => {
            console.log(`${sec} sec passed`)
            resolve()
        }, sec * 1000)
    })
}

const timerTasks = [
    timer.bind(null, 1),
    timer.bind(null, 2),
    timer.bind(null, 3),
    timer.bind(null, 4),
    timer.bind(null, 5)
]

const queue = new PromiseQueue(3)

timerTasks.forEach(task => queue.pushTask(task))
```

## ジェネレータ

ジェネレータの基礎については[こちら](https://hololab.esa.io/posts/189)

ジェネレータを使って非同期処理を逐次実行する方法として、次のようなアイデアがある。

```typescript
function asyncFlow(generatorFunction: Function) {
    function callback(err: any) {
        if (err) {
            return generator.throw(err)
        }
        const results = [].slice.call(arguments, 1)
        generator.next(results.length > 1 ? results : results[0])
    }

    const generator = generatorFunction(callback)
    generator.next()
}
```

引数にジェネレータ生成関数を受け取るというもの。ジェネレータ生成関数内では、実行したい非同期処理を実行したい順番に書いていく。

```typescript
// 指定した秒数を待機するタイマー
// 待機完了後、待機した秒数を結果として返す
function timerWithCallback(sec: number, callback: Function) {
    setTimeout(() => {
        callback(null, sec)
    }, sec * 1000)
}

function* sequentialTimer (callback: any) {
    const num1 = yield timerWithCallback(1, callback)
    console.log(`${num1} sec passed`)
    const num2 = yield timerWithCallback(2, callback)
    console.log(`${num2} sec passed`)
    const num3 = yield timerWithCallback(3, callback)
    console.log(`${num3} sec passed`)
    console.log('all tasks completed')
}
```

非同期処理の実行タイミングで`yield`を使う（待機するようなイメージ）。`yield`の戻り値から非同期処理の結果を取得する。

```typescript
asyncFlow(sequentialTimer)

// 1 sec passed
// 2 sec passed
// 3 sec passed
// all tasks completed
```

`asyncFlow`の実装をみてみると、引数に渡された生成関数からジェネレータを取得している。また、生成関数の引数に渡している関数は非同期処理のコールバック関数で、こちらは非同期処理が成功した場合はジェネレータの`next`メソッドに結果を渡し、失敗した場合は`generator.throw`でエラーを投げるようになっている。よってジェネレータ内の`yield`の戻り値に非同期処理の結果が渡ってくるようになっている。

# Ch.5 ストリーム

Node.jsではストリーミングデータを処理するAPIが充実している。`stream`モジュールとして提供されており、`EventEmitter`を継承して実装されている。

## ストリームの型

コアモジュールで提供されているストリームは以下の4種類

* Readable  
読み込み可能なストリーム
* Writable  
書き込み可能なストリーム
* Duplex  
読み込みも書き込みも可能なストリーム(net.Socketとか)
* Transform  
流れてきたデータを変換するためのストリーム(zlibとか)

## バッファリング

WritableストリームとReadableストリームは内部にバッファを持っており、このバッファサイズはコンストラクタに`highWaterMark`オプションを使って指定できる。

## Readable

Readableストリームは`flowing`と`paused`の2つのモードがある。`flowing`ではデータの読み取りはリソースから自動で行われ`EventEmitter`によるイベントを通じてデータを取得できる。一方、`paused`は明示的に`read`メソッドを呼び出さないとデータの取得ができない。すべてのReadableストリームは`paused`モードであるが、以下のいずれかのアクションを起こすことで`flowing`モードに切り返すことができる

* `data`イベントのハンドラを追加する
* `resume`メソッドを呼び出す
* `pipe`メソッドでWritableストリームとつなげる

Readableストリームはイベントハンドラの追加や明示的な呼び出しによってデータを取り出さなければ内部的にデータは生成されない。

`data`イベントハンドラを追加せず、`resume`メソッドを呼び出すなどしてReadableストリームをflowingモードに切り替えると、読み取ったデータは消失してしまうので注意が必要である。

また、Readableストリームには3つの状態があり、`_readableState.flowing`プロパティから参照できる

* null  
イベントハンドラやパイプなど、読み込まれたデータを消費する方法が設定されていない。このときストリームはデータ生成を行わない
* false  
`pause`メソッドや`unpipe`メソッドが呼び出される、もしくはバックプレッシャーでWritableの`write`メソッドがfalseを返してきたときの状態。データの生成は行われるが、生成されたデータは内部のバッファに蓄積されていく。
* true  
`data`イベントのハンドラやパイプなどによってデータを消費する準備が整ったときにtrueとなる。このときストリームはデータを生成し、各イベントを発火する。

Readableストリームでは、`push`メソッドが呼び出されるとデータがバッファリングされる。そのデータは`read`メソッドが呼び出されるまで内部のキューに止まり続ける（Bufferのこと？）

### Readbleストリームを作ってみる

```javascript
import {Readable, ReadableOptions} from 'stream'
import Chance from 'chance'

const chance = new Chance()

/**
 * ランダムな文字列を生成するストリーム
 */
class RandomStream extends Readable {
    constructor(options:ReadableOptions) {
        super(options)
    }

    /**
     * flowingモードになったらランダム文字列を生成し流していく
     * 5%の確率で生成を終了する
     * @param size
     * @private
     */
    _read(size: number): void {
        const chunk = chance.string()
        this.push(chunk, 'utf8')
        if(chance.bool({likelihood:5})){
            // ストリームの終了を知らせる契機
            this.push(null)
        }
    }
}

const rs = new RandomStream({encoding: 'utf8'})
rs.on('data', chunk => {
    console.log(chunk)
})
```

Readableストリームを実装するには、`stream.Readable`を継承して作ると良い。そのとき`_read`が抽象メソッドとなっているので、こちらを自分で実装する必要がある。`_read`メソッドは`read`から呼ばれるメソッドで、`data`イベントのハンドラや`pipe`で繋いだあとから内部的にコールされるようになる。

1. Readableの`pipe`を使ってWritableとつなげる
2. Readableyの内部(`pipe`メソッドの中)では`resume`がコールされてデータの生成が始まる
3. `resume`のなかでnextTickキューに`read`をコールする関数が登録される
4. `read`の中では`_read`がコールされて、即`data`イベントをemitするか、内部のバッファにためられる

## Writable

Writableストリームはデータの書き込み先を抽象化したオブジェクトである。基本的に`write`メソッドでデータを書き込んでいき、終了時に`end`メソッドを呼び出す。

### バックプレッシャ

I/Oの都合等でデータの消費よりもデータの生成が早い場合、バッファがどんどん増大し、メモリリソースが枯渇する場合がある。`pipe`を使わずにWritableストリームに書き込む場合は現在のバッファ量に気を使わなくてはいけない。`write`メソッドの戻り値（boolean）から判断することができ、falseが返ってきた場合はWritableのバッファが満杯なので書き込みを中断し、バッファに空きができたときにemitされる`drain`イベントのハンドラに書き込みを再開してやるようにすればよい。

```typescript
import Chance from 'chance'
import * as http from 'http'

const chance = new Chance()

const timer = function (sec: number) {
    return new Promise<void>(resolve => {
        setTimeout(() => resolve(), sec * 1000)
    })
}

http.createServer((req, res) => {
    res.writeHead(200, {'Content-Type': 'text/plain'})

    async function generateMore() {
        while (chance.bool({likelihood: 80})) {
            await timer(1)
            const shouldContinue = res.write(
              chance.string({length: (16 * 1024) - 1})
            )

            // writeからfalseが返ってきた場合はWritableのバッファがいっぱいなので
            // writeを注視し、drainイベントを待ち受ける
            if (!shouldContinue) {
                console.log('BackPressure')
                return res.once('drain', generateMore)
            }
        }

        res.end(`\nThe end...\n`, () => console.log('all data was sent'))
    }

    generateMore()
}).listen(8080, () => console.log('http://localhost:8080'))
```

### Writableストリームを実装してみよう

Writalbeを独自で実装する場合は`stream.Writable`を継承し、`_write`抽象メソッドを実装すれば良い

`pipe`はReadableのメソッド。Readableから`data`イベントが流れてくると、Writableの`write`にchunkが渡されて実行されるようになっている。`write`メソッド内部ではバッファがいっぱいでないか確認して、問題なさそうだったら`_write`メソッドをコールして書き込み処理を実行する。

```typescript
import * as stream from 'stream'
import * as fs from 'fs'
import * as path from 'path'
import mkdirp from 'mkdirp'

class ToFileStream extends stream.Writable {
    constructor() {
        super({objectMode: true})
    }

    _write(chunk: any, encoding: string, callback: (error?: (Error | null)) => void): void {
        mkdirp(path.dirname(chunk.path), err => {
            if(err) {
                return callback(err)
            }
            fs.writeFile(chunk.path, chunk.content, callback)
        })
    }
}

const tfs = new ToFileStream()
tfs.write({path: 'file1.txt', content: 'hello'})
tfs.write({path: 'file2.txt', content: 'hello world'})
tfs.write({path: 'file3.txt', content: 'good bye'})
tfs.end(() => console.log('all files created'))
```

## Duplex

ReadableかつWritableなストリーム、`net.Socket`がDuplexストリーム。独自実装する場合は`_read`と`_write`の両方が必要になる

## Transform

Duplexストリームの特殊ケースで、データ変換を目的に設計されている。例えばDuplexストリームである`net.Socket`などでは受信したデータと送信したデータに依存関係がない。しかし`Trasform`では受信したデータを加工して今日中する必要がある。

### Transformストリームの実装

Transformストリームを実装するためには`_transform`メソッドと`_flush`メソッドを実装する必要がある。`_transform`は`_write`の実装方法と感覚が近いが、後続のストリームに流すためには`push`を使って内部バッファにためていく必要がある。


```typescript
import * as stream from 'stream'
import * as util from 'util'

class ReplaceStream extends stream.Transform {
    private tailPiece = ''
    constructor(private searchString: string, private replaceString: string) {
        super()
    }

    _transform(chunk: any, encoding: string, callback: (error?: (Error | null), data?: any) => void): void {
        const pieces = (this.tailPiece + chunk).split(this.searchString)

        const lastPiece = pieces[pieces.length - 1]
        const tailPieceLen = this.searchString.length - 1

        this.tailPiece = lastPiece.slice(-tailPieceLen)
        pieces[pieces.length-1] = lastPiece.slice(0,-tailPieceLen)
        this.push(pieces.join(this.replaceString))
        callback()
    }

    _flush(callback: (error?: (Error | null), data?: any) => void): void {
        this.push(this.tailPiece)
        callback()
    }
}

const rs = new ReplaceStream('World', 'Node.js')
rs.write('Hello W')
rs.write('orld\n')
rs.write('Hello W')
rs.write('orld')
rs.end()
rs.pipe(process.stdout)
```

## pipe

すでに何度か使っているが、Readableストリームが生成したデータをWritableストリームに送り出すための仕組み。Readableストリームのメソッドとして提供されており、引数に渡したWritableストリームを戻り値として返すので、DuplexストリームやTransformストリームを渡すとチェーンを構成できる。なお、WritableストリームはReadableストリームがendイベントをemitすると自動的に終了される。バックプレッシャーの制御も`pipe`を使うことによって移譲することができる。

注意点としては`error`イベントはそれぞれのストリームについてハンドラを設定する必要がある。

## オブジェクトモード

## ストリームを使った非同期処理の逐次実行

ファイル名の配列をReadableストリームに変換して、1つずつファイルを読み込み1つのファイルに書き出す。

```javascript
const fromArray = require('from2-array')
const through = require('through2')
const fs = require('fs')

function concatFiles(destination, files, callback) {
    const destStream = fs.createWriteStream(destination)
    // ファイル名のリストをReadableストリームに変換して、要素ずつ流すようにする
    fromArray.obj(files)
      .pipe(through.obj((file, enc, done) => {
          // 流れてきたファイル名でReadableストリームを作ってdestにpipeする
          // このときReadableStreamを逐次的に差し替えて流していくので、ReadableStreamが終了してもWritableは終了しないようにしておく
          const src = fs.createReadStream(file)
          src.pipe(destStream, {end: false})
          src.on('end', done)
      })).on('finish', () => {
        destStream.end()
        callback()
    })
}

const files = ['file1.txt', 'file2.txt', 'file3.txt']
concatFiles('output.txt', files, () => console.log('Files concatenated successfully'))
```

# Ch.6 オブジェクト指向デザインパターンのNode.jsへの適用

## ファクトリ

オブジェクトの生成を実装から分離することが目的

## 公開コンストラクタ

## プロキシ

プロキシは他のオブジェクト（サブジェクト）へのアクセスを制御するオブジェクトで、サブジェクトと同じインタフェースを持って実装される。用途としては、

* データの妥当性確認  
サブジェクトに入力を渡す前にプロキシが妥当性を確認する
* セキュリティ  
クライアントがサブジェクトを操作する権限があるかを検証する
* キャッシュ  
プロキシ内部にキャッシュ領域を持ち、サブジェクトにアクセスするまえに出力を再利用できないか確認する
* 遅延初期化  
サブジェクトのインスタンス化の負荷が大きい時に、プロキシをいったんインスタンス化し、サブジェクトは必要となる時まで初期化を遅らせる
* ロギング  
サブジェクトへのアクセスを記録する

### プロキシを作ってみる

```javascript
// オブジェクトのプロトタイプチェーンを使って合成
function createProxyV1(subject) {
    const proto = Object.getPrototypeOf(subject)

    function Proxy(subject) {
        this.subject = subject
    }

    Proxy.prototype = Object.create(proto)

    // プロキシにてインターセプト
    Proxy.prototype.hello = function() {
        return this.subject.hello() + ' world!'
    }

    // サブジェクトに移譲
    Proxy.prototype.goodbye = function() {
        return this.subject.goodbye.apply(this.subject, arguments)
    }

    return new Proxy(subject)
}

// オブジェクトリテラルで新しいオブジェクトを生成して返す
function createProxyV2(subject) {
    return {
        hello: () => (subject.hello() + ' world!'),
        goodbye: () => (subject.goodbye.apply(subject, arguments))
    }
}

// オブジェクトのメソッドを差し替えて返す
function createProxyV3(subject) {
    const originalHello = subject.hello
    subject.hello = () => (originalHello.call(this) + ' world!')
    return subject
}
```

### ES2015のプロキシ

ES2015では`Proxy`というオブジェクトが追加された。

```javascript
const proxy = new Proxy(target, handler)
```

`target`はプロキシが適用されるサブジェクトで、`handler`は

## デコレータ

## アダプタ

## ストラテジ

## ステート

## テンプレート

## ミドルウェア

## コマンド