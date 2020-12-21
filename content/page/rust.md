---
title: "Rustメモ"
date: 2020-12-18T09:18:18-07:00
draft: true
---

# isize/usize

アーキテクチャによってビットサイズが変わる数値型。32bit環境なら32bit、64bit環境なら64bitとなる。配列やベクタの要素にアクセスするときのインデックスによく使われる。

# モジュールシステム

## モジュールを作る
Rustでは1つのファイルがモジュールとみなされる。

```rust
// module_a.rs
pub struct ModuleA {
  num: u32,
}
```

## モジュールを使う
main.rs(クレートならlib.rs)に使用するモジュールを宣言する

```rust
mod module_a
```

あるモジュールから別のモジュールを使う場合は
```rust
use crate::modeul_a::ModuleA
```