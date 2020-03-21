---
title: "es6-promiseのコードを読んでみた"
date: 2020-03-20T12:23:02+09:00
slug: "20200320_es6-promise-reading"
author: "d_yama"
draft: false
categories: ["JavaScript"]
tags: ["JavaScript", "Node.js", "Promise"]
---

# Node.jsでPromiseは欠かせない

Node.js（というかJavaScript）で非同期処理を書くにあたって、`Promise`や`async/await`は欠かせないものになっていると思っています。しかし、`Promise`の使い方はわかっていたとしても、その裏側でどのように動作しているのかはあまりよくわかっていませんでした。正しい使い方さえわかっていれば、それはそれでいいと思いますが、気になったら調べずにはいられないので実装を追ってみることにしました。

`Promise`はNode.jsにも組み込まれていますし、[Q](https://github.com/kriskowal/q)や[Bluebird](https://github.com/petkaantonov/bluebird)といった実装が存在しますが、今回はES6のPromiseと互換性のある[es6-promiseの実装](https://github.com/stefanpenner/es6-promise)を調べてみました。

なお、Node.jsのイベントループと[JavaScript Promiseの本](https://azu.github.io/promises-book/)の内容を理解している前提の記事となります。

## サンプルとするPromiseを使った非同期処理

本記事では次のコードが動作させることを目標にes6-promiseの実装を追ってみます。

```javascript
const timer = (sec) => {
    return new Promise((resolve, reject) => {
        if (sec >= 5) {
            return reject(`Can't wait for more than 5sec. [arg]: ${sec}`)
        }

        setTimeout(() => {
            return resolve(sec)
        }, sec * 1000)
    })
}

timer(3).then(time => console.log(`${time}sec elapsed.`), error => console.error(error))
timer(7).then(time => console.log(`${time}sec elapsed.`), error => console.error(error))
console.log('Timer setting completed.')

// [実行結果]
// Timer setting completed.
// Can't wait for more than 5sec. [arg]: 7
// 3sec elapsed.
```

シンプルなタイマー関数です。引数に指定した秒数だけ待機し、完了後に待機した秒数を返します。ただし、5秒以上を引数に指定した場合は即座に失敗します。この戻り値のPromiseオブジェクトに対し、`then`で成功時とエラー時のコールバックをそれぞれ登録しています。このコードを`Node.js(v12.16.1)`で実行すると、コメントのような結果となります。

## new Promise(fn)

Promiseを使うときにはコンストラクタ関数に非同期処理を実行する関数を渡します。Promsieのコンストラクタ関数のコードは次の通り。

```javascript
class Promise {
    constructor(resolver) {
        // これってなんなんだろうね？
        this[PROMISE_ID] = nextId()

        // 状態とPromise内で実行され処理の結果を保持するプロパティを初期化する
        this._result = this._state = undefined
        this._subscribers = []

        if(resolver !== noop) {
            initializePromise(this, resolver)
        }
    }
}

function noop() {}

const PENDING = void 0
const FULFILLED = 1
const REJECTED = 2
```

Promiseはそのライフサイクルの中で3つの状態を持ち、`PENDING`(未定)から始まり、非同期処理の結果に応じて`FULFILLED`(成功)か`REJECTED`(失敗)のどちらかに状態が遷移します。Promiseのコンストラクタでは、その状態を`PENDING`に初期化します。また、非同期処理の結果を保持するプロパティなども初期化します。`_subscribers`は`then`で渡された関数のキャッシュとして利用されます（`then`の中身をみるときに詳しくみます）。

プロパティの初期化が完了したあと、`intializePromise`をコールします。事前に`noop`と`resolver`を比較していますが、こちらは`then`が返すPromiseオブジェクトを生成するときに関係するものなので、ここでは気にしなくて大丈夫です。

```javascript
/**
 * @param promise Promiseオブジェクト
 * @param resolver Promiseコンストラクタ関数に渡された非同期処理
 */
function initializePromise(promise, resolver) {
    try {
        resolver(
          function resolvePromise(value) {
              resolve(promise, value)
          },
          function rejectPromise(reason) {
              reject(promise, reason)
          }
        )
    } catch (e) {
        reject(promise, e)
    }
}

function resolve(promise, value) {
    //...
}

function reject(promise, value) {
    //...
}
```

`initializePromise`にはコンストラクタで生成されたPromiseオブジェクトとコンストラクタに渡された非同期処理が引数として渡されます。今回の例だと`(resolve, reject) => ...`の関数が渡されます。

```javascript
const timer = (sec) => {
    return new Promise(
        // 以下の関数がinitalizePromiseに渡される
        (resolve, reject) => {
            if (sec >= 5) {
                return reject(`Can't wait for more than 5sec. [arg]: ${sec}`)
            }

            setTimeout(() => {
                return resolve(sec)
            }, sec * 1000)
        })
}
```

`intializePromise`では引数に渡された非同期処理が即実行されますが、実行時に二つの関数`resolvePromise`と`rejectPromise`が引数に渡されます。これらがPromiseを使うときの説明でよく言われる「処理結果が正常なら`resolve(結果の値)`を、失敗の場合は`reject(エラー)`と書きましょう」で出てくる`resolve`と`reject`の実態となります。

この`resolve`と`reject`の詳細を見る前に`then`の実装がどうなっているか見てみます。

## then

```javascript
then(onFulfillment, onRejection) {
    const parent = this
    const child = new this.constructor(noop)

    // 呼び出し元のPromiseオブジェクトの状態を取得する
    const {_state} = parent

    if (_state) {
        // FULFILLEDかREJECTEDである場合
        // ...
    } else {
        // PENDINGである場合
        subscribe(parent, child, onFulfillment, onRejection)
    }

    return child
}

function noop() {}
```

`then`の中で新しくPromiseオブジェクトを生成しています。`then`の呼び出し元Promiseオブジェクトと、この新しく生成した`Promise`オブジェクトを親子に見立ててPromiseチェーンを構成します。子Promiseオブジェクトを生成する時にはコンストラクタに空っぽの関数(`noop`)を渡します。Promiseのコンストラクタを見返してみると、コンストラクタの引数に渡された関数が`noop`であった場合は`initializePromise`を実行しないようになっています。子Promiseオブジェクトも初期状態は`PENDING`ですが、親Promiseでの処理が完了したあとに呼ばれることとなる`onFulfillment`か`onRejection`（`then`に渡すコールバック関数）にてその状態は変化します。なので子Promiseには親Promiseと違ってコンストラクタには空っぽの関数を渡しています。

子Promiseオブジェクトを生成したあとは、親のPromiseオブジェクトの状態によって処理が分岐します。今回の冒頭のサンプルの場合だと、非同期処理として`setTimeout`を利用しており`then`をコールした時点では`PENDING`である（ことがほとんど）なので、まずは`subscribe`がどうなっているのかを見てみます。

```javascript
function subscribe(parent, child, onFulfillment, onRejection) {
    const {_subscribers} = parent
    let {length} = _subscribers

    parent._onerror = null

    _subscribers[length] = child
    _subscribers[length + FULFILLED] = onFulfillment
    _subscribers[length + REJECTED] = onRejection

    if(length === 0 && parent._state) {
        // 呼び出し元のPromiseオブジェクトの状態がFULFILLEDかREJECTEDならコールバックを実行する
        // asap(publish, parent)
    }
}

const PENDING = void 0
const FULFILLED = 1
const REJECTED = 2
```

まんま`オブザーバーパターン`ですね。親Promiseオブジェクトが持つ配列に、子Promiseオブジェクトと、`then`に渡す親Promiseが成功/失敗したときのコールバック関数をそれぞれキャッシュします。のちほど親Promiseの状態が`FULFILLED`か`REJECTED`に遷移したときにキャッシュから関数が取り出され実行されます。

`then`に渡した関数がPromiseコンストラクタに渡した非同期処理が完了したタイミングで実行される仕組みがなんとなく見えてきたので、`initializePromise`に戻って`resolve`の実態が何なのかをみてます。

## resolve

`initailizaPromise`を再掲。

```javascript
/**
 * @param promise Promiseオブジェクト
 * @param resolver Promiseコンストラクタ関数に渡された非同期処理
 */
function initializePromise(promise, resolver) {
    try {
        resolver(
          function resolvePromise(value) {
              resolve(promise, value)
          },
          function rejectPromise(reason) {
              reject(promise, reason)
          }
        )
    } catch (e) {
        reject(promise, e)
    }
}

function resolve(promise, value) {
    //...
}

function reject(promise, value) {
    //...
}
```

タイマーのサンプルも並べてみる。

```javascript
const timer = (sec) => {
    return new Promise(
        // 以下の関数がinitalizePromiseに渡される
        (resolve, reject) => {
            if (sec >= 5) {
                return reject(`Can't wait for more than 5sec. [arg]: ${sec}`)
            }

            setTimeout(() => {
                return resolve(sec)
            }, sec * 1000)
        })
}
```

コンストラクタにて`initializePromise`がコールされ、その中でコンストラクタの引数に渡した非同期処理、すなわちタイマー関数がすぐさま実行されます。タイマー関数では`setTimeout`のコールバックがイベントループ内のキューに登録されます。そして指定した時間が経過するとコールバックが実行され`resolve(sec)`がコールされます。これは`initializePromise`が`resolver`に渡している`resolvePromise(value)`が実態で、その中で`resolve(promise, value)`がコールされます。この例だと`value`に入るのは`resolve(sec)`の`sec`(待機した秒数)になります。ということで(Promiseで定義されている)`resolve`の内容を見てみます。

```javascript
function resolve(promise, value) {
    if (promise === value) {
        // 処理結果がPromiseオブジェクトと同値の場合
        // 今回のサンプルではここにはこないので一旦スルー
    } else if (objectOrFunction(value)) {
        // 結果の値がオブジェクトか関数であった場合
        // 結果の値が別のPromiseオブジェクトの場合は、
        // そのPromiseオブジェクトの完了を待って結果を後続につなげていくので、
        // 別途仕組みが必要
    } else {
        // 処理の結果がプリミティブ型であった場合
        fulfill(promise, value)
    }
}

function fulfill(promise, value) {
    if (promise._state !== PENDING) return
    // Promiseオブジェクトのプロパティに処理の結果を登録し、状態をFULFILLEDに遷移させる
    promise._result = value
    promise._state = FULFILLED

    if (promise._subscribers.length !== 0) {
        // Node.jsではprocess.nextTickを使用するが、ブラウザやWeb Workerでは変わってくる。
        process.nextTick(() => publish(promise))
    }
}
```

`resolve`を通して`fulfill`がコールされ、その中で`process.nextTick`がコールされています。`publish`にてPromiseオブジェクトが保持しているコールバックのキャッシュ(`_subscribers`プロパティ)を使ってコールバックを実行するのですが、ここで`process.nextTick`を通してコールバックを呼び出しているので、コールバックも非同期で実行されることが保証されています。

ここはes6-promiseの実装からかなり端折っています。`process.nextTick`はNode.jsの機能であるためブラウザにそのようなAPIは存在しません。ライブラリでは実行環境を判定して、どのような仕組みで非同期的にコールバックを実行するか決めています。ブラウザでの動作がどうなるかは[es6-promiseの実装](https://github.com/stefanpenner/es6-promise)を見てください。

続いて`publish`です。

```javascript
function publish(promise) {
    const subscribers = promise._subscribers
    const settled = promise._state

    // thenでなにも登録されていない
    if (subscribers.length === 0) return

    let child, callback, detail = promise._result

    // subscribersを走査
    for (let i = 0; i < subscribers.length; i += 3) {
        child = subscribers[i]
        // FULFILLED = 1, REJECTED = 2 なので、以下のようにして成功/失敗時のコールバックを取り出せる
        callback = subscribers[i + settled]
        if (child) {
            invokeCallback(settled, child, callback, detail)
        } else {
            callback(detail)
        }
    }

}
```

Promiseオブジェクト内のプロパティからコールバックと、そのコールバックを登録したときに生成した子Promiseを取り出して`invokeCallback`を実行しています。

```javascript
function invokeCallback(settled, promise, callback, detail) {
    let hasCallback = typeof callback === 'function'
    let value, error, succeeded = true

    // thenに登録されたコールバックを実行する
    if (hasCallback) {
        try {
            value = callback(detail)
        } catch (e) {
            succeeded = false
            error = e
        }

        if(promise === value) {
            //
            return
        }
    } else {
        value = detail
    }

    // コールバックが成功すれば、子Promiseは成功とみなしてresolveする
    // コールバックが失敗ならば、子Promiseは失敗とみなしてrejectする
    if(promise._state !== PENDING) {

    } else if(hasCallback && succeeded) {
        resolve(promise, value)
    } else if(succeeded === false) {
        reject(promise, error)
    } else if(settled === FULFILLED) {
        fulfill(promise, value)
    } else if(settled === REJECTED) {
        reject(promise, value)
    }
}
```

コールバックを実行し、その結果に応じて子Promiseの状態を決定します。コールバックが成功していれば`resolve`をコールして子Promiseの状態を`FULFILLED`に、失敗していれば`reject`をコールして子Promiseの状態を`REJECTED`に遷移させています。`resolve`をコールすると親Promiseのときと同じようなプロセスを経て子Promiseに対する`then`で登録されたコールバックが実行されていくためPromiseチェーンが成立します。

ここまでで以下のコードは動くようになりました。

```javascript
timer(3).then(time => console.log(`${time}sec elapsed.`), error => console.error(error))
```

しかしPromiseが失敗したときのコールバックはまだ動きません。そのために次は`reject`を見ていきます。

## reject

```javascript
function reject(promise, reason) {
    if (promise._state !== PENDING) return

    promise._state = REJECTED
    promise._result = reason

    if (promise._subscribers.length !== 0) {
        process.nextTick(() => publish(promise))
    }
}
```

状態を`REJECTED`に遷移させているだけで、やっていることは`resolve`と変わりありません。その後`publish`→`invokeCallback`と続いていくのでPromiseチェーンが成立するのも変わりありません。

これでサンプル内の以下のコードも動作するはず！

```javascript
timer(7).then(time => console.log(`${time}sec elapsed.`), error => console.error(error))
```

と思いきやエラーがコンソール上に出力されません。これは意図した結果と異なります。どうしてこうなるのか、もう一度タイマー関数の実装を見てみます。

```javascript
const timer = (sec) => {
    return new Promise((resolve, reject) => {
        if (sec >= 5) {
            return reject(`Can't wait for more than 5sec. [arg]: ${sec}`)
        }

        setTimeout(() => {
            return resolve(sec)
        }, sec * 1000)
    })
}
```

Promiseに渡している関数ですが、引数がエラーに該当する場合は同期的に`reject`をコールしています。また、これまで見てきたとおり、Promiseのコンストラクタに渡す関数は即実行されます。すなわち、`then`でコールバックを登録する前にPromiseの状態が`REJECTED`に遷移してしまっているため、Promiseオブジェクトが保持するコールバックのキャッシュが`reject`コール時には空であるため状態が変化しても後続の処理が実行されません。これを解決するためには、Promiseオブジェクトの状態が変化した後に`then`でコールバックを登録したとしてもそのコールバックが動作するようにしてあげる必要があります。

## then、再び

その仕組みは`then`の中にあります。

```javascript
then(onFulfillment, onRejection) {
    const parent = this
    const child = new this.constructor(noop)

    const {_state} = parent

    if (_state) {
        // Promiseオブジェクトの状態がすでにFULFILLEDかREJECTEDに遷移している場合
        // thenの引数とPromiseオブジェクトの状態から、成功/失敗どちらのコールバックを実行するか決める。
        const callback = arguments[_state - 1]
        process.nextTick(() => invokeCallback(_state, child, callback, parent._result))
    } else {
        subscribe(parent, child, onFulfillment, onRejection)
    }

    return child
}

function noop() {}
```

最初はさらっとスルーしていましたが、`then`では呼び出し元のPromiseオブジェクトの状態によって処理が分岐します。状態がすでに確定している場合はコールバックを`process.nextTick`を介して実行しています（常に非同期で動作することを保証している）。

これで今回のサンプルのコードはすべて動作するようになりました！

## まとめ
Promiseの仕組みってどうなっているんだろう？と気になったので、[es6-promiseの実装](https://github.com/stefanpenner/es6-promise)を追ってみました。今回まとめたものは基本の基本だけなので、`catch`や`Promise.resolve`、`Promise.all`、`Promise.race`などの実装がどうなっているのか見てみるのもいいんじゃないかなと思います。

## 参考

* [es6-promiseの実装](https://github.com/stefanpenner/es6-promise)
* [JavaScript Promiseの本](https://azu.github.io/promises-book/)
* [Node.jsでのイベントループの仕組みとタイマーについて](https://blog.hiroppy.me/entry/nodejs-event-loop)