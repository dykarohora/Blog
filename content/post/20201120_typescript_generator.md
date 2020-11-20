---
title: "TypeScriptとジェネレータ"
slug: "20200627_typescript-collection"
author: "d_yama"
date: 2020-11-20T17:10:00+09:00
draft: false
categories: ["TypeScript"]
tags: ["TypeScript", "Generator"]
---

# はじめに

ちょっとジェネレータを使いたいケースが出てきたので仕様を確認しようと思ったら、昔書いたテキストを発掘したのでメモも兼ねて公開。
C#をよく使ってた頃に書いたものなので、それと照らし合わせながらの内容になっています。

## ジェネレータとは
C#でいうところのイテレータ構文。ジェネレータ関数と呼ばれるものから返されるオブジェクトで、このオブジェクトはiterableなので`for...of`を使って列挙することが可能。

## ジェネレータ関数
ジェネレータ関数とは`function*`宣言（アスタリスクがついていることが大事）によって定義された関数で、ジェネレータオブジェクトを生成してそれを返す関数。返されたジェネレータオブジェクトの`next`メソッドをコールすると、ジェネレータ関数内の`yield`が出現するところまで処理が進みポーズ状態となる。`yield`式に値が設定されている場合は、`next`メソッドの戻り値に指定された値が含まれたオブジェクトが返される。

```typescript
function* generator() {
    console.log("one")
    yield 1
    console.log("two")
    yield 2
    console.log("three")
    yield 3
}

const g = generator()

console.log(g.next())
// [標準出力]
// one
// { value: 1, done: false }

console.log(g.next())
// [標準出力]
// two
// { value: 2, done: false }

console.log(g.next())
// [標準出力]
// three
// { value: 3, done: false }

console.log(g.next())
// [標準出力]
// { value: undefined, done: true}
```

C#のイテレータ構文と同様、`next`がコールされたあとジェネレータ関数の処理がどこまで進んでいるかの状態は内部に保存される。もう一度`next`をコールすると、前回の`yield`式の続きから処理が継続される。C#の場合はイテレータ構文が最後まで達したあとでも`Current`プロパティから最後の`yield`式の値を取得できたが、TypeScriptでは`undefined`となる。

## yield*式
複数のジェネレータ関数を組み合わせることもできる。

```typescript
function* anothereGenerator(i: number) {
    yield i + 1
    yield i + 2
    yield i + 3
}

function* generator(i: number) {
    yield i
    yield* anothereGenerator(i)
    yield i + 10
}

const g = generator(10)

console.log(g.next().value) // 10
console.log(g.next().value) // 11
console.log(g.next().value) // 12
console.log(g.next().value) // 13
console.log(g.next().value) // 20
console.log(g.next().value) // undefined
```

ジェネレータ関数の中で`yield*`式にて別のジェネレータ関数を指定することによって、自身の中に別のジェネレータオブジェクトを展開できる。

## ジェネレータ関数内でのreturn

C#でいうところの`yield break`。ジェネレータ関数内で`return`すると、そこでiterateは停止する。

```typescript
function* generator() {
    console.log("one")
    yield 1
    console.log("two")
    yield 2
    return
    console.log("three")
    yield 3
}

const g = generator()

console.log(g.next())
// [標準出力]
// one
// { value: 1, done: false }

console.log(g.next())
// [標準出力]
// two
// { value: 2, done: false }

console.log(g.next())
// [標準出力]
// { value: undefined, done: true}

console.log(g.next())
// [標準出力]
// { value: undefined, done: true}
```

`return`に値を指定すれば、iterateが停止したときにその値が返ってくる。

```typescript
function* generator() {
    console.log("one")
    yield 1
    console.log("two")
    yield 2
    return 'end'
    console.log("three")
    yield 3
}

const g = generator()

console.log(g.next())
// [標準出力]
// one
// { value: 1, done: false }

console.log(g.next())
// [標準出力]
// two
// { value: 2, done: false }

console.log(g.next())
// [標準出力]
// { value: 'end', done: true}

console.log(g.next())
// [標準出力]
// { value: undefined, done: true}
```

## ジェネレータに値を渡す

ジェネレータは`yield`を使って呼び出し元に値を渡すことができるが、反対に`next`メソッドに値を渡すことによって呼び出し元からジェネレータに値を渡すこともできる。

```typescript
function* sampleGenerator() {
    const num1 = yield 1
    console.log(`inside generator: ${num1}`)
    const num2 = yield 2
    console.log(`inside generator: ${num2}`)
    const num3 = yield 3
    console.log(`inside generator: ${num3}`)
}

const g = sampleGenerator()

console.log(`call next: ${g.next(10).value}`)  // ①
console.log(`call next: ${g.next(20).value}`)  // ②
console.log(`call next: ${g.next(30).value}`)  // ③
console.log(`call next: ${g.next(40).value}`)  // ④
```

実行結果は以下の通りとなる。

```
call next: 1
inside generator: 20
call next: 2
inside generator: 30
call next: 3
inside generator: 40
call next: undefined
```

実装と結果を照らし合わせていけばわかるが、まず①にて最初のyiledに到達するが、ここでは`num1`に値はセットされない。`num1`には②での`next`に渡した値がセットされる。そのような形で`yield`式の戻り値にはyieldに到達したあとの次の`next`呼び出し時に渡された値が渡ってくる。

## 反復処理
TypeScriptには繰り返し構文として`for..of`が用意されている（C#でいうところの`for-each`）。オブジェクトが`for..of`の対象として利用できる条件として`iterable`プロトコルに準拠する、といったものがある。これはC#でいうところの`IEnumerable`インタフェースと同じようなものだが、JavaScriptにはインタフェースがないため、プロトコルといった規約が決められている。

`iterable`プロトコルの内容は、`[Symbol.iterator]`というプロパティをもち、このプロパティの値が`iterator`プロトコルに順子するオブジェクトを返す関数である、というものである（これもC#と同じようなかんじ）

お察しのとおり`iterator`プロトコルも`IEnumerator`インタフェースと同じようなもので、bool型の`done`プロパティと任意の型の`value`プロパティを持つオブジェクトを返す`next`メソッドを実装しているという条件になる。

ジェネレータオブジェクトはこの`iteratrable`プロトコルと`iterator`プロトコルに準拠するので、`for..of`を使って反復処理ができる。

```typescript
interface Generator<T = unknown, TReturn = any, TNext = unknown> extends Iterator<T, TReturn, TNext> {
    // NOTE: 'next' is defined using a tuple to ensure we report the correct assignability errors in all places.
    next(...args: [] | [TNext]): IteratorResult<T, TReturn>;
    return(value: TReturn): IteratorResult<T, TReturn>;
    throw(e: any): IteratorResult<T, TReturn>;
    [Symbol.iterator](): Generator<T, TReturn, TNext>;
}
```

```typescript
function* generator():Generator<number, void, unknown> {
    yield 1
    yield 2
    yield 3
}

const gen = generator()

for(const v of gen) {
    console.log(v)
}

// [標準出力]
// 1
// 2
// 3
```

## Generatorの型パラメータ
見ての通り、ジェネレータオブジェクトには3つの型パラメータが定義されている。
```typescript
interface Generator<T = unknown, TReturn = any, TNext = unknown> extends Iterator<T, TReturn, TNext>
```

それぞれ何を表しているかというと

| Column 1 | Column 2 |
| -------- | -------- |
| T     | `yield`式の値の型   |
| TReturn | `yield return`式の値の型 | 
| TNext | `next`メソッドの引数の型 |

## ジェネレータ関数から配列を作る
`Array.from`メソッドは引数でジェネレータオブジェクトを受け取って配列を作ることができる。

```typescript
function* generator() {
    yield 1
    yield 2
    yield 3
    yield 4
    yield 5
}

const array = Array.from(generator())
console.log(array) // [ 1, 2, 3, 4, 5 ]
```