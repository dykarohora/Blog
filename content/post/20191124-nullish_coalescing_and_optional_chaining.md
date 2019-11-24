---
title: "Nullish CoalescingとOptional Chaining"
slug: "20191124-nullish_coalescing_and_optional_chaining"
author: "d_yama"
date: 2019-11-24T11:58:18+09:00
draft: false
categories: ["TypeScript"]
---

# とは？
TypeScript 3.7で実装された、null/undefinedまわりの記述をいいかんじにできる機能

## Nullish Coalescing
要は厳密にnullとundefinedを判定してくれる。

これまで変数やプロパティが`null` or `undefined`であった場合、フォールバックを用意するには次のように書く必要があった。

```typescript
function fallback(): string {
    return "hoge"
}

const fuga = undefined

// 条件分岐
let word1: string
if(fuga !== null && fuga !== undefined) {
    word1 = fuga
} else {
    word1 = fallback()
}
console.log(word1)      // hoge

// 三項演算子を使う
const word2 = (fuga !== null && fuga !== undefined) ? fuga : fallback()
console.log(word2)      // hoge
```

新しく追加された`??`演算子を使うことによってシンプルに書けるようになる。

```typescript
function fallback(): string {
    return "hoge"
}

const fuga = undefined

// Nullish Coalescing
const word3 = fuga ?? fallbac()
console.log(word3)      // hoge
```

そのほか有用な点として、これまで`null` or `undefined`をワンライナーで判定しようとした場合、`||`演算子を使うこともできた。これは`null`や`undefined`をbool型にするとfalseになるのを利用したもの。

```typescript
console.log(undefined || 'undefinedです')     // undefinedです
console.log(null || 'nullです')               // nullです

const word = 'ワードです'
console.log(word || 'null or undefined?')     // ワードです
```

しかし、TypeScriptでは`0`や`空文字`なんかもfalseになってしまうため、判定に使う変数にどんな値が入る可能性があるのか、プログラマ側で慎重に考える必要があった。

```typescript
const num = 0
console.log(num || 'number型の値が入っていません') // number型の値が入っていません
// → 0でも表示する必要がある、みたいな要件だったら詰み

const word = ''
console.log(word || 'string型の値が入っていません') // string型の値が入っていません
```

Nullish Coalescingは厳密に`null`か`undefined`かを判定してくれるので、そのあたりに気をつかう必要がなくなる。

```typescript
const num = 0
console.log(num ?? 'number型の値が入っていません') // 0

const word = ''
console.log(word ?? 'string型の値が入っていません') // (空文字)
```

## Optional Chaining
`.`(カンマ)を使ってオブジェクトのプロパティにアクセスするとき、プロパティが`null` or `undefined`であった場合、直ちに処理を停止してくれる機能。

TypeScriptでは強制アンラップみたいな挙動をする非nullアサーション演算子(`!`)が存在するが、これはnull許容型のプロパティにカジュアルにアクセスしたいときに便利である。しかし実体が`null`や`undefined`であった場合は実行時エラーを発生させてしまう可能性もある。

```typescript
interface Girl {
    name: string
    device?: Device  // undefined許容
}

interface Device {
    modelNumber: string
    lotNumber: number
}

const nanoha: Girl = {
    name: 'Nanoha Takamachi',
    device: {
        modelNumber: 'Raising Heart',
        lotNumber: 0
    }
}

const arisa: EarthGirl = {
    name: 'Arisa Bunnings'
}

console.log(nanoha.device!.modelNumber) // Raising Heart

// deviceプロパティはundefinedなのに、modelNumberプロパティにアクセスしようとするので実行時エラーが発生する
console.log(arisa.device!.modelNumber)
```

これまではこれを避けようとするとif文などでチェックする必要があるが、Optional Chainingでは新しく追加された`?`演算子を使うことによってシンプルに書ける。

```typescript
interface EarthGirl {
    name: string
    device?: Device
}

interface Device {
    modelNumber: string
    lotNumber: number
}

const nanoha: EarthGirl = {
    name: 'Nanoha Takamachi',
    device: {
        modelNumber: 'Raising Heart',
        lotNumber: 0
    }
}

const arisa: EarthGirl = {
    name: 'Arisa Bunnings'
}

console.log(nanoha.device?.modelNumber) // Raising Heart
console.log(arisa.device?.modelNumber)  // undefined
// → deviceがundefinedであった場合、後続のプロパティにはアクセスしない。式の値としてはundefinedが返ってくる。
```

# 参考
[TypeScript3.7の目玉機能を使ってみた - MAMANのITブログ](https://blog.mamansoft.net/2019/10/16/use-typescript3.7-great/)