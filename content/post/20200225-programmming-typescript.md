---
title: "20200210 Node Design Pattern"
date: 2020-02-07T17:55:37+09:00
draft: true
---


# Programming TypeScriptの読書メモ

## Ch.1 Introduction

TypeScriptはJavaScriptにはなかった型安全な世界を提供する。JavaScriptは弱い動的型付け言語であるので、コードの不整合がとても把握しづらい。
例えば以下のコードは不整合があるにもかかわらず、JavaScriptではエラーも発生せず実行できてしまう。この挙動がバグの原因だったりすると、原因特定がとても難しい。

```javascript
const invalid1 = 3 + []
console.log(invalid1)           // 3
console.log(typeof invalid1)    // string ←型がnumberから変化している

let obj = {}
console.log(obj.foo)            // undefined ←エラーが発生するわけでもない

function a(b) {
    return b/2
}

console.log(a("z"))          // Nan ←実行できてしまう
```

TypeScriptでは上記のコードの不整合は静的解析によってすべて事前に把握することができる。

## Ch.2 A10_000 Foot View

JavaScriptの弱い動的な型付けの挙動の説明と、TypeScriptによる静的な型解析の説明、およびそれらのメリット/デメリットを説明してくれる。