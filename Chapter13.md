---
marp: true
theme: "night"
transition: "slide"
slideNumber: true
---
<!-- theme: gaia -->
<!-- size: 16:9 -->
<!-- page_number: true -->
<!-- paginate: true -->

# プログラミングRust
13章　ユーティリティトレイト

2021/1/15

---
## 13.1　`Drop`
- 値の所有者がいなくなると，Rustは値を **ドロップ** (drop) する．
- 値をドロップするには，以下のような全てを解放する必要がある．
    - その値が所有する他の値．
    - ヒープ上のストレージ．
    - システムリソース．
- ドロップは以下のような状況で発生する．
    - 変数がスコープから出たとき．
    - 式の値が `;` 演算子で捨てられたとき．
    - ベクタの後ろの要素を捨てて短くしたとき．

---
#### 自動でドロップされる場合
```rust
struct Appellation {
    name: String,
    nicknames: Vec<String>
}
```

- `Appellation` は，ヒープストレージを所有する.
- Rustは，`Appellation` がドロップされる際にこれらのヒープを自動
的に解放してくれる．
- `std::ops::Drop` トレイトを実装することで，独自型の値がドロップされる際の動作をカスタマイズ可能．

---
#### `Drop` の実装
```rust
trait Drop {
    fn drop(&mut self);
}
```

- `Drop` の実装は，C++のデストラクタに似ている．
- ある値がドロップされ，その値が `std::ops::Drop` を実装していた場合，Rustはその値の持つフィールドや要素をドロップする前に，その値の `drop` メソッドを呼び出す．
- `drop` メソッドは暗黙にしか呼び出せず，明示的に呼び出そうとするとエラーになる．

---
#### `Appellation` 型の `Drop` の実装1
- `Drop::drop` が呼び出されるのは，フィールドや要素がドロップされる前．
- `Appellation` 型の `Drop` の実装では，フィールドの値を利用することができる．
```rust
impl Drop for Appellation {
    fn drop(&mut self) {
        print!("Dropping {}", self.name);
        if !self.nicknames.is_empty() {
            print!(" (AKA {})", self.nicknames.join(", "));
        }
        println!("");
    }
}
```

---
#### `Appellation` 型の `Drop` の実装2
- 次のように実行してみる．
```rust
{
    let mut a = Appellation { name: "Zeus".to_string(),
                              nicknames: vec!["cloud collector".to_string(),
                                              "king of the gods".to_string()] };
    println!("before assignment");
    a = Appellation { name: "Hera".to_string(), nicknames: vec![] };
    println!("at end of block");
}
```

---
#### `Appellation` 型の `Drop` の実装3
- 2つ目の `Appellation` を `a` に代入すると，最初の `Appellation` がドロップされる．
- `a` のスコープを離れる際に，2つ目の `Appellation` もドロップされる．
- 実行結果は次のようになる．
```
before assignment
Dropping Zeus (AKA cloud collector, king of the gods)
at end of block
Dropping Hera
```

---
#### メモリの解放について
- `Vec` 型も `Drop` を実装しており，個々の要素をすべてドロップしてから専有していたバッファを解放する．
- `String` はテキストを保持するため内部に `Vec<u8>` を持っているので，独自に `Drop` を実装する必要はない．
- `Appellation` 値がドロップされると，`Vec` の `Drop` 実装が個々の文字列を解放し，ベクタの要素を解放する．
- `Appellation` の値そのものを保持するメモリに関しては，この値の所有者がメモリを解放する責任を負う．

---
#### 移動とドロップ1
- 変数の値が移動されていた場合，その変数はドロップされない．
- この原則は，制御フローによって変数から値が移動されているか移動されていないかわからない場合にも適用される．
```rust
let p;
{
    let q = Appellation { name: "Cardamine hirsuta".to_string(),
                          nicknames: vec!["shotweed".to_string(),
                                          "bittercress".to_string()] };
    if complicated_condition() {
        p = q;
    }
}
println!("Sproing! What was that?");
```

---
#### 移動とドロップ2
- `complicated_condition()` が `true` か `false` かによって，`p` と `q` のどちらかが `Appellation` を保持することになり，もう一方は未初期化状態になる．
- どちらになるかによって，値がドロップされるタイミングが `println!` の前になるか後になるかが決まる．
- 値はあちこちに移動されているかもしれないが，ドロップされるのは一度だけだ．

---
#### `FileDesc` の例1
- Rustが知らない資源を管理する型を定義しているのでもない限り，`std::ops::Drop` を実装する必要はない．
- Unixシステムでは，Rustの標準ライブラリは `FileDesc` を用いてOSのファイルディスクリプタを表現している．
```rust
struct FileDesc {
    fd: c_int,  // c_intはi32の別名
}
```

- `FileDesc` の `fd` フィールドは，ファイルディスクリプタの番号で，使い終わったらクローズするべきもの．

---
#### `FileDesc` の例2
- 標準ライブラリでは，`FileDesc` の `Drop` は下記のように実装されている．
```rust
impl Drop for FileDesc {
    fn drop(&mut self) {
        let _ = unsafe { libc::close(self.fd) };  // システムコールclose関数を表す
    }
}
```

- Rustのコードは `unsafe` ブロックの中でしかC関数を呼び出すことができないので，`unsafe` を使っている．

---
#### `Drop` と `Copy`
- ある型が `Drop` トレイトを実装しているならば，`Copy` トレイトを実装することはできない．
- ある型が `Copy` だということは，単純なバイト単位の複製で値のコピーが作れる．
- しかし，同じデータに対して `drop` メソッドを2回以上呼び出すことは，ほとんどの場合何か間違っている．

---
#### `drop` 関数
- 標準のプレリュードには，値をドロップするための関数 `drop` が定義されている．
```rust
fn drop<T>(_x: T) { }
```

- この関数は値で引数を受け取り，所有権を呼び出し元から引き継いだ上で何もしない．
- `_x` がスコープから出る際に，`_x` の持つ値がドロップされる．

---
## 13.2　`Sized`
- sized型は，その型の値のメモリ上でのサイズが常に同じになるような型．
- Rustのほとんどの型はsized．
    - `u64` は8バイト，`(f32, f32, f32)` 型のタプルは12バイト．
    - 列挙型もsized，どのヴァリアントが実際に使われているかに関わらず，常に最大のヴァリアントを保持できる空間を占有するため．
    - `Vec<T>` は，値そのものはバッファへのポインタと容量と長さだけなのでsized．

---
#### unsizedな型1
- Rustには数少ないがunsizedな型も存在し，これらの型の値のサイズは一定ではない．
- 文字列スライス型 `str` は，unsized．
    - 文字列リテラル `"diminutive"` や `"big"` はそれぞれ10バイトと3バイトを占める `str` スライスへの参照（図13-1を参照）．
- `[T]` のような配列スライス型もunsized．
    - `&[u8]` のような共有参照は任意のサイズの `[u8]` スライスを指すことができる．

---
#### unsizedな型2
- トレイトオブジェクトの参照先も，unsized型．
- トレイトオブジェクトはあるトレイトを実装した何らかの値へのポインタ(11章を参照）．
- `&std::io::Write` や `Box<std::io::Write>` は，`Write` トレイトを実装した何らかの値へのポインタ．
- `Write` を実装するかもしれない型の集合は閉じていないので，`Write` のサイズは定まらずunsizedになる．

---
#### unsized型の制約
- Rustでは，unsizedの値を変数に格納したり、引数として渡したりすることはできない．
- `&str` や `Box<Write>` のようにポインタを介してしか扱うことしかできない．
- ポインタはサイズが決まっているため．

---
#### トレイトオブジェクトとスライスへのポインタの対称性
- トレイトオブジェクトとスライスへのポインタのいずれも，型にはそれを使うための十分な情報がない．
    - `[u8]` をインデックスするには長さがわかっていなければならない．
    - `Box<Write>` のメソッドを呼び出すには，参照されている値に適した `Write` の実装がわかっていなければならない．
- いずれの場合も，ファットポインタが長さやvtableのポインタを与えることで，型に欠けている情報を補う．
- 静的には失われた情報を動的な情報で補っているということ．

---
#### `Sized` トレイト
- すべてのsizedな型は，`std::marker::Sized` トレイトを実装している．
    - メソッドも関連型もなく，Rustは適用できるすべての型に対して自動的にこのトレイトを実装する．
    - 独自に実装することはできない．
    - 型変数の制約にしか使えない．
- `T: Sized` という制約は，`T` がコンパイル時にサイズの決まる型であることを要求している．
- この種のトレイトは **マーカートレイト** (marker trait) と呼ばれる．

---
#### ほとんどの型変数は `Sized` 型に限定するべき
- 暗黙のデフォルトで `Sized` となっている．
- `struct S<T> { ... }` と書くと，Rustコンパイラは `struct S<T: Sized> { ... }` と書いたものとして解釈する．
- `T` を制約したくないのであれば，`struct S<T: ?Sized> { ... }` と明示的に書く必要がある．
- `?Sized` という構文はこの場合にだけ使え，「`Sized` でなくてもいい」ということを意味している．

---
#### `Sized` かも
- 制限されてはいるが，unsized型はRustの型システムがスムーズに機能することを助けている．
- 標準ライブラリのドキュメントを読んでいると，型変数の制約が `?Sized` になっているものがあるが，ほとんどの場合その値が参照されているだけで，通常の値だけでなくスライスやトレイトオブジェクトでもコードが動作するように付けられている．
- ある型変数が `?Sized` になっていることを，`Sized` かも (questionably sized) ということがある．

---
#### `RcBox` 1
- 構造体型の最後のフィールドだけは `unsized` でもよい．
- そのような構造体はそれ自身 `unsized` になる．
- 参照カウントポインタ `Rc<T>` は，プライベート型 `RcBox<T>` へのポインタとして実装されているが，この型は参照カウントと `T` の両方を格納する．
```rust
struct RcBox<T: ?Sized> {
    ref_count: usize,  // 参照を数えるカウンタ
    value: T,          // 参照の数を数えるT
                       // このフィールドへのポインタを参照解決する
}
```

---
#### `RcBox` 2
- `RcBox` をsized型に対して使うとその結果は `Sized` な構造体になり，unsized型に対して使うとunsizedな構造体になる．
    - `RcBox<std::fmt::Display>` に対して使った結果の `RcBox<Display>` はunsizedな構造体．
- `RcBox<Display>` の値を直接作ることはできず，まず `Display` を実装した値を `value` 型として持つ通常のsizedな `RcBox` を作る．
    - 例えば `RcBox<String>` など．
    - `RcBox` を指す参照 `&RcBox<String>` から，ファット参照 `&RcBox<Display>` を作る．

---
#### `RcBox` 3
```rust
let boxed_lunch: RcBox<String> = RcBox {
    ref_count: 1,
    value: "lunch".to_string()
};

use std::fmt::Display;
let boxed_displayable: &RcBox<Display> = &boxed_lunch;
```

- この変換は値が関数に渡される際に暗黙に行われ，`&RcBox<Display>` を期待している関数に `&RcBox<String>` を渡すことができる．

---
## 13.3　Clone
- `std::clone::Clone` トレイトは，自身のコピーを作ることができる型のトレイトだ．
```rust
trait Clone: Sized {
    fn clone(&self) -> Self;
    fn clone_from(&mut self, source: &Self) {
        *self = source.clone()
    }
}
```

---
#### `clone` メソッド
- `clone` メソッドは，`self` の独立したコピーを作りそれを返す．
- メソッドの返り値の型は `Self` になっており，関数は一般に `unsized` の値を返すことができないため，`Clone` トレイトは `Sized` トレイトを拡張している．
- 事実上 `Self` の型を `Sized` に制約している．

---
#### `clone` のコスト
- `Vec<String>` をコピーすると，ベクタだけではなく個々の `String` 型の要素もコピーされる．
- Rustが値のクローンを自動的に作らず，明示的にメソッドを呼ばせるのはこのため．
- `Rc<T>` や `Arc<T>` などの参照カウントポインタは例外で，これらの型をクローンしても参照カウントをインクリメントして新しいポインタを返すだけだ，

---
#### `clone_from` メソッド1
- `clone_from` メソッドは，`self` を書き換えて `source` のコピーにする．
- デフォルトの定義では，`source` をクローンしそれを `*self` に移動する．

---
#### `clone_from` メソッド2
- ただし，型によっては同じ効果をより高速に得られるものもある．
    - `String` の `s` と `t` について，`s = t.clone();` とすると，`t` をクローンし `s` の古い値をドロップしてから，クローンされた値を `s` に移動する．
    - これにはヒープ確保が1回とヒープ解放が1回必要．
    - 元の `s` が所有していたヒープバッファが，`t` の内容を収めるのに十分な容量ならば，ヒープの確保も解放も必要ない．
- ジェネリックコードでは，可能な限り `clone_from` を使って，このような最適化が実行されるようにするべき．

---
#### `Clone` の自動実装
- 独自の型に `Clone` を実装する場合，デフォルト動作（すべてのフィールドまたは要素に対して `clone` を行って，それらを使って新しい値を作る）でよく，デフォルトの `clone_from` に満足できるのならば，型定義の上に `#[derive(Clone)]` 属性を付けるだけで，自動的にRustが `Clone` トレイトを実装してくれる．

---
#### 標準ライブラリでの `Clone`
- 標準ライブラリに定義されている型のうち，コピーして意味のあるものほとんどすべてが `Clone` を実装している．
- 中にはコピーすることに意味のない型もある．
    - `std::sync::Mutex` は `Clone` を実装しない．
    - `std::fs::File` のような型はコピーできるが，OSに十分な資源がなけれ
- その代わり，`std::fs::File` は `try_clone` メソッドを実装している．
- このメソッドは `std::io::Result<File>` を返すので．失敗を報告することができる．

---
## 13.4　Copy
- ある型が `Copy` 型となるのは，`std::marker::Copy` マーカトレイトを実装している場合だ．
```rust
trait Copy: Clone { }
```

- これを実装するのは簡単だ．
```rust
impl Copy for MyType { }
```

---
#### `Copy` を実装できる型・できない型
- `Copy` マーカトレイトには言語上で特殊な意味を持つので，`Copy` を実装できる型は浅いバイト単位のコピーだけでコピーが可能な型のみに制限されている．
- 他の資源，例えばヒープ上のバッファやOSのハンドルを持つような型を `Copy` にすることはできない．
- `Drop` トレイトを実装している型は `Copy` にすることはできない．
- ある型が特別な後始末用のコードを必要とするならば，コピーする際にも何らかの特別な方法が必要なはずなので，`Copy` 型にはできない．

---
#### `Copy` の自動実装
- `#[derive(Copy)]` とすることで，`Copy` を自動実装することもできる．
    - 2つまとめて `#[derive(Copy, Clone)]` とすることが多い．
- ただし，型を `Copy` にするかどうかは慎重に考えた方がよい．
    - `Copy` にした方が使いやすいのは間違いないが，実装は強く制約される．
    - 暗黙のコピーが高価である場合もある（4.3を参照）．

---
## 13.5　`Deref` と `DerefMut`
- 独自実装した型に対して `std::ops::Deref` トレイトや `std::ops::DerefMut` トレイトを実装することで，その型に対する `*` や `.` などの参照解決演算子の動作を指定できる．
- `Box<T>` や `Rc<T>` などのポインタ型はこれらのトレイトを実装することで，Rust組み込みのポインタ型と同じように振る舞う．
- 代入先に使われる場合や，参照先に対する可変参照が要求される場合には，`DerefMut` (dereference mutably) トレイトが使われる．
- 読み出し参照で十分な場合には `Deref` トレイトが使われる．

---
#### `Deref` と `DerefMut` の実装1
- これらのトレイトは次のように定義されている．
```rust
trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}

trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

---
#### `Deref` と `DerefMut` の実装2
- `deref` メソッドや `deref_mut` メソッドは，`&Self` 参照を受け取り，`&Self::Target` 参照を返す．
- `Target` は `Self` が所有しているか参照している型を指す．- - - - `DerefMut` は `Deref` を拡張している．
    - 何かを参照解決して変更できるのであれば，当然共有参照の借用もできるはずなため．
- これらのメソッドは `&self` と同じ生存期間を持つ参照を返すので， `self` は返された参照が生きている間はずっと借用されたままになる．

---
#### 参照解決型変換1
- `deref` メソッドは，`&Self` 参照を受け取り `&Self::Target` 参照を返すので，Rustはこれを前者から後者への自動変換にも用いる．
- `deref` を呼び出すことで型の不整合が防げるのなら，Rustは自動的に `deref` を呼び出すということ．
- `DerefMut` を実装していれば，可変参照に対する同様の変換が行われる．
- これらは，**参照解決型変換** (deref coercions) と呼ばれている．

---
#### 参照解決型変換2
- `Rc<String>` の値 `r` に対して `String::find` を実行したければ，`(*r).find('?')` ではなく `r.find('?'){}` と書くだけでよい．
メソッド呼び出しが暗黙に `r` を借用し，`Rc<T>` が `Deref<Target=T>` を実装しているので `&Rc<String>` が `&String` に自動型変換される．
- `String` は `Deref<Target=str>` を実装しているため，`&String` から自動型変換で `&str` が得られるので，`str` のメソッドをすべて `String` で再実装する必要はない．
- バイトのベクタ `v` をバイトスライス `&[u8]` を受け取る関数に渡したい場合，引数を `&v` とするだけでよい．
これは，`Vec<T>` が `Deref<Target=[T]>` を実装しているため．

---
#### 参照解決型変換3
- Rustは，必要ならば参照解決型変換を連続して行う．
- 例えば，`split_at` を `Rc<String>` に対して直接実行でききるのは，`&Rc<String>` が `&String` に参照解決され，さらにこれが `&str` に参照解決され，`&str` が `split_at` メソッドを持っているためだ．

----
#### `Selector` の実装例1
- 次のような型を考える．
```rust
struct Selector<T> {
    /// `Selector`で利用できる要素を示す．
    elements: Vec<T>,
    /// `elements`内の「現在使用している」要素を示す．
    /// `Selector`は，現在使用している要素へのポインタとして機能する．
    current: usize
}
```

- `Selector` がドクコメントに書かれた通りに機能するようにするには，この型に対して `Deref` と `DerefMut` を実装しなければならない．

----
#### `Selector` の実装例2
```rust
use std::ops::{Deref, DerefMut};

impl<T> Deref for Selector<T> {
    type Target = T;
    fn deref(&self) -> &T {
        &self.elements[self.current]
    }
}

impl<T> DerefMut for Selector<T> {
    fn deref_mut(&mut self) -> &mut T {
        &mut self.elements[self.current]
    }
}
```

---
#### `Selector` の利用例
```rust
let mut s = Selector { elements: vec!['x', 'y', 'z'], current: 2 };

// `Selector`は`Deref`を実装しているので，
// `*`演算子を使って現在使用している要素を参照できる．
assert_eq!(*s, 'z');

// 'z'がアルファベットかどうかを，`char`の
// メソッドを`Selector`に対して直接使ってチェックする．
// 参照解決型変換が行われる．
assert!(s.is_alphabetic());

// `Selector`の参照先に代入することで，'z'を'w'に変える．
*s = 'w';
assert_eq!(s.elements, ['x', 'y', 'w']);
```

---
#### その他の注意1
- `Deref` トレイトと `DerefMut` トレイトは，`Box`，`Rc`，`Arc` などのスマートポインタ型を実装するために設計されている．
- また，参照で使う場合が多い型の「所有バージョン」を実装するのにも適している．
    - `Vec<T>` は `[T]` の，`String` は `str` の所有バージョン．
- C++では基底クラスのメソッドが自動的にサブクラスから利用できるが，これと同じ目的で `Target` 型のメソッドを自動的に使うためだけに `Deref` や `DerefMut` を実装してはいけない．
- 期待するように動作するとは限らないし，何かおかしなことが起こった場合には混乱のもととなる．

---
#### その他の注意2
- 参照解決型変換は，混乱の元ともなる．
- Rustは型の不整合を解決するためにこの機能を使うが，型変数の制約を満足させるためには用いない．
```rust
let s = Selector { elements: vec!["good", "bad", "ugly"],
                   current: 2 };

fn show_it(thing: &str) { println!("{}", thing); }
show_it(&s);
```

- Rustコンパイラは，引数の型が `&Selector<&str>`，仮引数の型が `&str` であることを見出し，`Deref<Target=str>` の実装を見つけ，この呼び出しを `show_it(s.deref())` と書き換える．

---
#### その他の注意3
- しかし，`show_it` をジェネリック関数に変更するとRustは急に非協力的になる．
```rust
use std::fmt::Display;
fn show_it_generic<T: Display>(thing: T) { println!("{}", thing); }
show_it_generic(&s);
```

```
error[E0277]: the trait bound `Selector<&str>: Display` is not satisfied
    |
542 |         show_it_generic(&s);
    |         ^^^^^^^^^^^^^^^ trait `Selector<&str>: Display` not satisfied
    |
```

---
#### その他の注意4
- 関数に渡しているのは `&Selector<&str>` で，関数の引数の型は `&T` となっているので，型パラメータ `T` は `Selector<&str>` でなければならない．
- さらにRustは，制約 `T: Display` が満たされているかをチェックするが，制約を満たすためには参照解決型変換が行われないので，このチェックが失敗する．
- この問題を回避するためには，`as` 演算子を使って明示的に型変換すればよい．
```rust
show_it_generic(&s as &str);
```
