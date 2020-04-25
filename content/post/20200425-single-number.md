---
title: "LeetCode - Single Number"
date: 2020-04-25T09:19:17+09:00
slug: "20200425-single-number"
author: "d_yama"
draft: false
categories: ["algorithm"]
tags: ["TypeScript", "algorithm"]
---

# LeetCodeをはじめてみた

[牛尾さんのブログ](https://simplearchitect.hatenablog.com/entry/2020/04/23/075627)の中に[LeetCode](https://leetcode.com/)の話題があったので調べてみたら、面白そうだったのでやってみることにしました。

LeetCodeは日本でいうところの[AtCoder](https://atcoder.jp/)のようなサービスみたいです。コーディングテストでの頻出問題も多く紹介されており、面接対策としてはメジャーみたいです。LeetCodeでは4月の間、1日1題のマラソンみたいなイベントが行われているのでそれに参加してみることにしました。

# Single Number

第一問の内容は次の通り。

> 入力として空ではない整数の配列が与えられる。この配列はある1つの整数を除いて全ての要素が2回現れる。その1つの整数を見つける関数を書け。

## 一番最初に思いついた実装

```typescript
function singleNumber(nums: number[]) {
    for (const num of nums) {
        const leftIdx = nums.indexOf(num)
        const rightIdx = nums.lastIndexOf(num)
        if(leftIdx === rightIdx) {
            return num
        }
    }
}
```

各要素に対してインデックスを先頭と末尾からそれぞれ調べてそれが一致した要素が求めるもの、という実装ですね。しかしこちら見て分かる通り、時間計算量がO(n^2)となっている、遅い。

## 次に考えた実装

とりあえず計算量をO(n)に持っていきたい。そこで次に考えたのは、配列を操作して要素の出現回数を記録する、というもの。

```typescript
function singleNumber(nums: number[]) {
    const map = new Map<number, number>()

    for (const num of nums) {
        const count = map.get(num)
        if (count === undefined) {
            map.set(num, 1)
        } else {
            map.set(num, count + 1)
        }
    }

    for (const [key, count] of map) {
        if(count === 1) {
            return key
        }
    }
}
```

`Map<number, number>`を使って実装しました。これで時間計算量はO(n)となりました。実測してもだいぶ早くなった。

あ、でも入力の前提条件として、重複する要素の出現回数は2回と決まっているので、`Map`のvalueはbooleaでもいいかもしれませんね。

## もっと短く早く書けないのか？

もっと短く各方法はないのかなと思い解説を読んでみるとXORを使った方法が紹介されていました。

> a *XOR* 0 = a  
> a *XOR* a = 0

となるので

> a *XOR* b *XOR* a = a *XOR* a *XOR* b = 0 *XOR* b = b

すなわち配列の要素のすべてのXORを取れば、1回だけあらわれる要素が最後に残るということですね。
TypeScriptなら`Array.prototype.reduce`で実装できます。

```typescript
function singleNumber(nums: number[]) {
    return nums.reduce((acc, num) => acc ^ num)
}
```

あー、なるほど、って感じですね。重複している要素の重複回数が偶数でなくてはいけないという制約はありますが。