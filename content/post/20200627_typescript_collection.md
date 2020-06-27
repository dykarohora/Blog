---
title: "TypeScriptでイテレータを使ってみよう！"
slug: "20200627_typescript-collection"
author: "d_yama"
date: 2020-06-27T21:14:37+09:00
draft: false
categories: ["TypeScript"]
tags: ["TypeScript", "Iterator", "ファーストクラスコレクション"]
---

# 発端
先週書いているTypeScriptコードの中でコレクションクラスを作っていたんですよね。クラスの外からArrayを触れるような実装だったのでC#で言うところのIEnumerable的な振る舞いをさせたいなと思ったので何か手はないものかと思い調べてみたそのまとめです。

# イテレータとは
Wikipediaでは次のように書かれています。

> イテレータ（英語: Iterator）とは、プログラミング言語において配列やそれに類似するデータ構造の各要素に対する繰返し処理の抽象化である。実際のプログラミング言語では、オブジェクトまたは文法などとして現れる。反復するためのものの意味で反復子（はんぷくし）と訳される。繰返子（くりかえし）という一般的ではない訳語もある。
> イテレータ - Wikipedia
> https://ja.wikipedia.org/wiki/%E3%82%A4%E3%83%86%E3%83%AC%E3%83%BC%E3%82%BF

コレクションオブジェクトの要素を順番に取り出して処理をすること、みたいなかんじですかね。`forループ`や`for...of`構文と関連がありそうです。

MDNでも[反復処理プロトコル](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Iteration_protocols)というページに説明があります。

## 反復子プロトコル (iterator protocol)
JavaScriptでプロトコルってあまり聞き覚えがないですね。ある動作（イテレータの場合は反復処理）を実現するために必要となる仕様、みたいな、インタフェースに考え方が近いのですが型定義とはちょっと違ってたりします。プロトコルを満たしたオブジェクトを作るとランタイム側でよしなにいいかんじに扱ってくれるための仕組みです。

（ちなみにSwiftでは[プロトコル](https://docs.swift.org/swift-book/LanguageGuide/Protocols.html)という仕組みがあったりします）

JavaScriptでは以下の仕様を満たす`next`メソッドを持つオブジェクトは反復子プロトコルを実装している、すなわち**イテレータである**と言えます。
* 戻り値は`done`と`value`の2つのプロパティを持つ
* `done`プロパティは繰り返し処理が完了したかどうかを表すboolean値(trueならば終了している)
* `value`プロパティは任意の型の値

ですので次のようなオブジェクトもイテレータと言えます。（意味があるコードかはさておき）

```typescript
const iterator = {
    next: () => {
        return {value: 'str', done: false}
    }
}
```

## 反復可能プロトコル(iterable protocol)
オブジェクトが反復可能プロトコルの仕様を満たしていると、自分で作ったオブジェクトに対して`for...of`構文を使うことができます。（ランタイムが内部でよしなにしてくれるところ）

反復可能プロトコルの仕様は、オブジェクトが`[Symbol.iterator]`プロパティを持ち、そのプロパティが**イテレータを返す関数**であること、です。

さきほどのイテレータを流用してつくると、、

```typescript
const iterator = {
    next: () => {
        return {value: 'str', done: false}
    }
}

const iterable = {
    [Symbol.iterator]: () => iterator
}

// イテラブルなオブジェクトはfor...ofで繰り返し処理ができる
for(const str of iterable) {
    console.log(str)        // ただ無限ループになってしまうので停止条件は書かないと…
}
```

# TypeScriptでイテレータを作ってみる
もうちょっと使い道がありそうなものを作ってみます。前節ではオブジェクトベタ書きでしたが、ちゃんとクラスとしてデザインしてみます。

TypeScriptでは`IterableIterator`というインタフェースがあるのでそちらを使いたいと思います。

```typescript
interface Iterator<T, TReturn = any, TNext = undefined> {
    return?(value?: TReturn): IteratorResult<T, TReturn>;
    throw?(e?: any): IteratorResult<T, TReturn>;
}

interface Iterable<T> {
    [Symbol.iterator](): Iterator<T>;
}

interface IterableIterator<T> extends Iterator<T> {
    [Symbol.iterator](): IterableIterator<T>;
}
```

このインタフェースを実装したクラスは反復子プロトコルと反復可能プロトコルの両方を満たします。

```typescript
class MyCollection implements IterableIterator<number> {
    public constructor(private readonly array: number[]) {
    }

    [Symbol.iterator](): IterableIterator<number> {
        return this
    }

    private index = 0
    next(...args: [] | [undefined]): IteratorResult<number, any> {
        return this.index < this.array.length
          ? {value: this.array[this.index++], done: false}
          : {value: undefined, done: true}
    }
}

const collection = new MyCollection([1, 2, 3, 4, 5, 6])

for (const num of collection) {
    console.log(num)    // 1.. 2.. 3.. 4.. 5.. 6..
}
```

## こんなもの作って何がうれしいの？
自分で作ったクラスで`for...of`が使えるのはわかったけど、Arrayが持ってるsliceやfilterとか使えないし要素の追加や変更すらできやしない、不便すぎない？Arrayそのまま使えばよくね？

ごもっともですね。iterableオブジェクトはArrayと比べて出来ないことが多すぎです。ですがプログラミングでは**出来ないようにしておく**ことがとても効果的にはたらくことがあります。例えば先ほど作ったコレクションクラスは、オブジェクトを生成したあとは`Array`と違って要素の変更や追加はできません。このような状態を`イミュータブル(変更不可)`といいます。例えばメソッドの引数にイミュータブルなオブジェクトが渡されたとして、メソッドの中でそのオブジェクトの中身が変わらないことが保証されていると、変化を気にしなくてよくなるのでコードの見通しがよくなりデバッグもしやすくなるかと思います。

また、イテレータの話からは離れてしまいますが、自分で作ったコレクションに対して何か振る舞いを持たせたいときにも、**出来ることと出来ないこと**を明確にしておくと堅牢なコードを書くのに役立つことがあります。例えば買い物かごみたいなものを作って合計金額を算出したいとき、こんな実装が考えられます。

```typescript
interface Item {
    title: string
    price: number
}

class Cart {
    public constructor(private readonly cart: Item[] = []) {
    }

    public enter(item: Item) {
        this.cart.push(item)
    }

    public get itemList() {
        return this.cart
    }
}

const cart = new Cart()
cart.enter({title: "apple", price:100 })
cart.enter({title: "orange", price:100 })
cart.enter({title: "魔法少女リリカルなのは A's Blu-ray BOX", price: 26900})

const sum = cart.itemList.reduce((sum, item) => sum + item.price, 0)
```

Cartを使う側がgetter経由で内部の配列にアクセスして合計金額を計算する、というものですね。アクセスした配列は`Array`型なので、Cartを使う側が中身を書き換えることは自由にできます。

これを合計金額の算出ロジックをコレクションクラスに隠蔽するように書き換えてみると、

```typescript
class Cart {
    public constructor(private readonly stock: Item[] = []) {
    }

    public enter(item: Item) {
        this.stock.push(item)
    }

    public getPriceSummation(): number {
        let sum = 0
        for (const item of this.stock) {
            sum += item.price
        }
        return sum
    }
}
```

このように合計金額を計算するロジックを`Cart`の中に閉じ込めて、使う側が内部の配列に**アクセスすることができないので**、極端ではありますがカートの中身をめちゃくちゃにしてしまうような危ないコードが書かれてしまうのをある程度防ぐことができます。

イテレータの話というよりはオブジェクト指向プログラミングの話になってしまいましたが、オブジェクト指向プログラミングでコレクションクラスを設計するときに、**コレクションクラスを使う側が出来ることと出来ないことをハッキリさせる**といいことがあったりするので、そのときにはイテレータが役に立つ場面があるかもしれないよ、ということが言いたいです。

コレクションクラスの設計についてもっと詳しく知りたい方は「ファーストクラスコレクション」で調べてみるといいと思います。

# とはいえfilterとかmapくらいは使いたい
iterableオブジェクトは繰り返し処理しかできません。その他のコレクション操作についてはプロポーサルはあるようですが、まだ実装されている処理系はあまりないかと思います。

ならば必要なものだけ実装すればいいじゃない。

## filter
せっかくなので`filter`の戻り値でArrayを返すのではなく、iterableなオブジェクトを返すようにしてみました。

```typescript
class MyCollection implements IterableIterator<number> {
    // 追加
    filter(pred: (num: number) => boolean): IterableIterator<number> {
        let currentIndex = 0
        let arrayRef = this.array
        const obj: IterableIterator<number> = {
            [Symbol.iterator](): IterableIterator<number> {
                return obj
            }, next(args: any): IteratorResult<number, any> {
                while (currentIndex < arrayRef.length) {
                    if (pred(arrayRef[currentIndex])) {
                        return {value: arrayRef[currentIndex++]}
                    }
                    currentIndex++
                }
                return {value: undefined, done: true}
            }
        }
        return obj
    }
}

const collection = new MyCollection([1, 2, 3, 4, 5, 6])
for (const num of collection.filter(num => num % 2 == 0)) {
    console.log(num)    // 2.. 4.. 6..
}
```

## map

`map`についても`filter`と実装の方向性は変わりませんが、任意の方に変換できるようにジェネリックメソッドにしました。

```typescript
class MyCollection implements IterableIterator<number> {
    // 追加
    map<T>(func: (num: number) => T): IterableIterator<T> {
        let currentIndex = 0
        let arrayRef = this.array
        const obj: IterableIterator<T> = {
            [Symbol.iterator](): IterableIterator<T> {
                return obj
            }, next(args: any): IteratorResult<T, any> {
                if (currentIndex >= arrayRef.length) {
                    return {value: undefined, done: true}
                }
                return {value: func(arrayRef[currentIndex++]), done: false}
            }
        }
        return obj
    }
}

const collection = new MyCollection([1, 2, 3, 4, 5, 6])
for (const str of collection.map<string>(num => `twice: ${num * 2}`)) {
    console.log(str)
}
```

# まとめ
TypeScriptのイテレータとその使い道を紹介しました。Array禁止、すべてをファーストクラスコレクションにしろ、と言いたいわけではなく、プログラミングする上でコレクションを使うときにそのコレクションに制約を持たせたい、ドメインに固有の振る舞いを持たせて実装を隠蔽したい、そういったときの一つの選択肢としてこのような手段がありますよ、って思ってもらえればと思います。

# 参考
* [反復処理プロトコル - MDN web docs](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Iteration_protocols)
* [Iterators - TypeScript Deep Dive](https://basarat.gitbook.io/typescript/future-javascript/iterators)
    * [日本語版](https://typescript-jp.gitbook.io/deep-dive/future-javascript/iterators) 
