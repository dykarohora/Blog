---
title: "Rustのライフタイム注釈について調べてみた"
slug: "20201230_rust_lifetime"
author: "d_yama"
date: 2020-12-30T09:45:00+09:00
draft: false
categories: ["Rust"]
tags: ["Rust"]
aliases: ["ライフタイム"]
---

# [前提] 借用規則

1. 不変・可変を問わず、参照のライフタイムは参照先の値のスコープよりも短くなくてはいけない
2. 値が共有されている間（不変の参照が有効な間）は値の変更を許さない。すなわちオブジェクトTに対して以下のいずれかの状態のみしか存在を許可しない
	1. 任意個の不変の参照&Tを持つ
	2. 単一の可変の参照&mut Tを持つ

この結果、
* 不変の参照のライフタイムが尽きていない状態で可変の参照は存在することができない
* その逆も然りで、可変の参照が存在する状態で不変の参照は存在することができない
* 不変・可変に関わらず、参照が存在する場合は値に直接アクセスしてその値を変更することはできない
	* 可変の参照が存在するならば、可変の参照を通じてしか変更ができないということ

例えば次のコードはコンパイルエラーとなる。
```rust
let mut c1 = Child(5);  
let rc = &c1;  
println!("{:?}", c1);  
c1.0 = 4;  				// <- assignment to borrowed `c1.0` occurs here
println!("{:?}", rc);
```

c1についての不変の参照が存在するからだ。

```rust
let mut c1 = Child(5);  
let rc = &mut c1;  
// println!("{:?}", c1);  
c1.0 = 4;  				// <- assignment to borrowed `c1.0` occurs here
println!("{:?}", rc);
```

可変の参照であったとしても、やはり変更はできない。変更のインタフェースは常に一つだけ。

## 借用チェッカー

```rust
{
    let r;                // ---------+-- 'a
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
    println!("r: {}", r); //          |
}                         // ---------+
```

* Rustの借用チェッカーでは、`r`には`'a`とラベリングされたライフタイムが、`x`には`'b`とラベリングされたライフタイムがあるとみなす。
* `r`は`x`の参照を得ているが、`x`のライフタイムが尽きた後にその値を使おうとしているので、ダングリング参照が発生するためRustはコンパイルエラーを吐き出す。

# 関数の引数と戻り値のライフタイム
次の関数はコンパイルが通らない。

```rust
fn longest(x: &str, y: &str) -> &str {  
 if x.len() > y.len() {  
 x  
    } else {  
 y  
    }  
}
```

コンパイラからのメッセージ

```
error[E0106]: missing lifetime specifier
  --> src/main.rs:28:33
   |
28 | fn longest(x: &str, y: &str) -> &str {
   |               ----     ----     ^ expected named lifetime parameter
   |
   = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
```

> この関数の戻り値の型は借用された値（参照）です。しかし、関数のシグネチャからは戻り値が`x`からの借用なのか`y`からの借用なのか読み取れません。

このようにライフタイム注釈をつけるとコンパイルは通る。
```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {  
 if x.len() > y.len() {  
 x  
    } else {  
 y  
    }  
}
```
2つの引数のライフタイムは同じで、戻り値のライフタイムも同じであるとコンパイラに教え込んでいるような感じ。

一方、以下のようにするとコンパイルは通らない。
```rust
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {  
 if x.len() > y.len() {  
 x  
    } else {  
 y  
    }  
}
```
第一引数と第二引数のライフタイムにそれぞれラベル付け（ライフタイム注釈）を行い戻り値のライフタイムは第一引数と同じ、とコンパイラに伝えているようなイメージ。

コンパイラからのメッセージは以下の通り
```
error[E0623]: lifetime mismatch
  --> src/main.rs:26:9
   |
22 | fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
   |                                   -------     -------
   |                                   |
   |                                   this parameter and the return type are declared with different lifetimes...
...
26 |         y
   |         ^ ...but data from `y` is returned here

```
第二引数と戻り値のライフタイムは異なっている。関数の中身でも第二引数（参照）を返すケースがあるのでライフタイムがミスマッチを起こしている、というもの。

これは以下のように書き直すとコンパイルが通るようになる。
```rust
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &'a str where 'b: 'a {  
 if x.len() > y.len() {  
 x  
    } else {  
 y  
    }  
}
```

`where`の後ろにライフタイム`'a`と`'b`の関係性を示しており、これは`'b`の方が`'a`よりもライフタイムが長いということを示している。

一方、`'a`の方が`'b`よりもライフタイムが長いと示すとコンパイルは通らなくなる。
```
error[E0623]: lifetime mismatch
  --> src/main.rs:26:9
   |
22 | fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &'a str where 'a: 'b {
   |                       -------     ------- these two types are declared with different lifetimes...
...
26 |         y
   |         ^ ...but data from `y` flows into `x` here

```
このことから、戻り値である参照のライフタイムは、引数で与えられた参照のライフタイム以下の期間でないといけないことがわかる。

ライフタイムを明示した関数longestの利用ケースとして、以下のような場合はコンパイルが通らない。

```rust
let strA = String::from("AB");  
let result;  
{  
 let strB = String::from("ABC");  
 result = longest(strA.as_str(), strB.as_str());  // borrowed value does not live long enough
}  
println!("{}", result)
```

longest関数は与えられた2つの文字列スライスのうち、長い方を返す関数である。上の場合ははstrBが帰ってきてresultにセットされるので、strBのライフタイムが尽きた、スコープの外でresultを使うことができないというのは理解できる。

じゃあstrAの方が長ければコンパイルは通るのかというと当たり前だがそんなわけない。

ここで関数シグネチャにつけたライフタイム注釈を改めて見てみると、
```rust
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &'a str where 'b: 'a {  
 if x.len() > y.len() {  
 x  
    } else {  
 y  
    }  
}
```
戻り値の参照のライフタイムは第一引数と同じ、もしくは第二引数以下である、とみなすことができる。コンパイルエラーが発生したコードでは、戻り値を保持する変数resultは第二引数のstrBよりもライフタイムが**長い**ので、関数シグネチャに付与したライフタイム注釈と矛盾を起こしている。だからコンパイルが通らない。

# ライフタイム省略規則
関数の引数と戻り値に参照が現れるとき、それらの関係を示すためにライフタイム注釈をつける必要があることは前述の通り。このライフタイム注釈は、コンパイラが推測できるときに限り省略することができる。コンパイラは以下の規則にしたがってライフタイムを推測する。

1. 参照型の引数はそれぞれ独自のライフタイムをもつ（＝それぞれ異なるライフタイム注釈をつけることがでこきる）
2. 引数の中で参照型が1つだけなら、その引数のライフタイムと戻り値（参照）のライフタイムと同一とみなす
3. 第一引数が&selfまたは&mut selfならば、戻り値（参照）のライフタイムはselfと同一とみなす


## 構造体定義のライフタイム注釈

構造体に参照型を持たせたいときがある、こんなかんじで。

```rust
#[derive(Debug)]  
struct Parent {  
 ref_chile: &Child  
}  
  
#[derive(Debug)]  
struct Child(usize);
```

しかしこれはコンパイルが通らない。

```
error[E0106]: missing lifetime specifier
  --> src/main.rs:10:16
   |
10 |     ref_chile: &Child
   |                ^ expected named lifetime parameter
   |
help: consider introducing a named lifetime parameter
   |
9  | struct Parent<'a> {
10 |     ref_chile: &'a Child
   |
```
参照型のフィールドにライフタイム注釈がないと言われている。
言われた通りにライフタイム注釈をつけるとコンパイルが通るようになる。

```rust
#[derive(Debug)]  
struct Parent<'a> {  
 ref_child: &'a Child  
}
```

これはParent構造体はフィールドにあるChildの不変の参照よりもライフタイムが短いということをコンパイラに伝えている。

このParent構造体は次のようにしてインスタンスを作れる。

```rust
let c1 = Child(5);  
let p1 = Parent { ref_child: &c1 };  
println!("{:?}", p1);
```

以下の場合はコンパイルエラーが発生する。

```rust
let p1;  
{  
 let c1 = Child(5);  
 p1 = Parent { ref_child: &c1 };  
}  
println!("{:?}", p1);
```

Childのライフタイムが尽きているのに、その参照を持つParentがそのライフタイム以上に存在をすることを許さないということですね。
```
error[E0597]: `c1` does not live long enough
  --> src/main.rs:26:34
   |
26 |         p1 = Parent { ref_child: &c1 };
   |                                  ^^^ borrowed value does not live long enough
27 |     }
   |     - `c1` dropped here while still borrowed
28 |     println!("{:?}", p1);
   |                      -- borrow later used here

```

構造体が参照型のフィールドを複数持つ場合もライフタイム注釈をつけて定義する。

```rust
#[derive(Debug)]  
struct Parent<'a> {  
 ref_child_a: &'a Child,  
 ref_child_b: &'a Child  
}
```

関数の節で見たときのように、それぞれの参照型のフィールドに別々のライフタイム注釈をつけ、そのライフタイム間の関係性も定義できる。

```rust
#[derive(Debug)]  
struct Parent<'a, 'b> where 'b: 'a {  
 ref_child_a: &'a Child,  
 ref_child_b: &'b Child,  
}
```

ただ、この定義のやり方はあまり意味がなさそう。構造体のインスタンスのライフタイムがフィールドの参照よりも短ければよいので、例えば以下のコードのように`'b`のライフタイムが`'a`より短くても問題なく実行できる。

```rust
    let p1;
    {
        let c1 = Child(5);
        {
            let c2 = Child(10);
            {
                p1 = Parent { ref_child_a: &c1, ref_child_b: &c2 };
                println!("{:?}", p1);
            }
            println!("{:?}", c1);
            println!("{:?}", c2);
        }
    }
```

# メソッド定義におけるライフタイム注釈

構造体が参照型のフィールドをもつ＝ライフタイム注釈がある場合は、メソッド定義を行うimplでもライフタイム注釈が必要となる。

```rust
#[derive(Debug)]  
struct Parent<'a> {  
	ref_child_a: &'a Child,  
}  
  
impl<'a> Parent<'a> {  
	pub fn get_child_age(&self) -> usize {  
		self.ref_child_a.0  
	}  
}
```

メソッドが（self以外の）参照型の引数を持つときには関数の時と同様にライフタイム注釈をつける必要がある。

```rust
#[derive(Debug)]  
struct Parent<'a> {  
	ref_child_a: &'a Child,  
}  
  
impl<'a> Parent<'a> {  
	pub fn get_child_age(&self) -> usize {  
		self.ref_child_a.0  
	}  
  
 	pub fn compare_child_age(&'a self, another_child: &'a Child) -> &'a Child {  
 		if self.ref_child_a.0 > another_child.0 {  
 			self.ref_child_a  
 		} else {  
 			another_child  
        }  
	}
}
```

なお、compare_child_ageメソッドのライフタイム注釈は省略することができない。省略した場合は、省略規則にしたがってコンパイラは次のようにライフタイムを推測する。

```rust
// 規則1.より各引数にそれぞれライフタイム注釈がつく
// 規則3.より戻り値のライフタイム注釈はselfと同じ'aとなる
// メソッドは実装上、'bのライフタイムをもつanother_childを返す可能性がある
// ライフタイム'aと'bの関係性が明示されていない、'bのライフタイムが'aより短い可能性があるため、コンパイラはそれを許さない
pub fn compare_child_age(&'a self, another_child: &'b Child) -> &'a Child {  
	if self.ref_child_a.0 > another_child.0 {  
		self.ref_child_a  
 	} else {  
		another_child  
	}  
}
```

冗長な書き方ではあるが、引数それぞれにライフタイム注釈をつけて`where`を使ってライフタイム注釈間の関係性を明示して定義することもできる。

```rust
impl<'a> Parent<'a> {  
	pub fn compare_child_age<'b>(&'a self, another_child: &'b Child) -> &'a Child where 'b: 'a {  
		if self.ref_child_a.0 > another_child.0 {  
			self.ref_child_a  
		} else {  
			another_child  
		}
	}
}
```

# 参考

[ライフタイムで参照を検証する - The Rust Programming Language 日本語版](https://doc.rust-jp.rs/book-ja/ch10-03-lifetime-syntax.html)
