---
marp: true
theme: "night"
transition: "slide"
slideNumber: true
title: "Chapter02"
---
<!-- theme: gaia -->
<!-- size: 16:9 -->
<!-- page_number: true -->
<!-- paginate: true -->

# プログラミングRust
9章　構造体

2020/11/xx

---
## 9.1　名前付きフィールド型構造体
- 名前付きフィールド型構造体の定義は次のように行う．
```rust
/// 8ビットグレースケールピクセルの長方形
struct GrayscaleMap {
    pixels: Vec<u8>,
    size: (usize, usize)
}
```

- 型名はキャメルケース，フィールド名やメソッド名はスネークケースが慣習．

---
#### 構造体式1
- 型の値を生成するには次のような **構造体式** を用いる．
```rust
let width = 1024;
let height = 576;
let image = GrayscaleMap {
    pixels: vec![0; width * height],
    size: (width, height)
};
```

- 構造体式は，型名で始まり，個々のフィールドの名前と値を並べたものを中括弧で囲む．

---
#### 構造体式2
- 省略形として，フィールド名と同じ変数や引数を用いてフィールドを指定する方法もある．
```rust
fn new_map(size: (usize, usize), pixels: Vec<u8>) -> GrayscaleMap {
    assert_eq!(pixels.len(), size.0 * size.1);
    GrayscaleMap { pixels, size }
}
```

- 構造体式 `GrayscaleMap { pixels, size }` は，`GrayscaleMap { pixels: pixels, size:
size }` の省略形．
- 一部のフィールドだけを省略形で記述することも可能．

---
#### フィールドへのアクセス
- おなじみの `.` 演算子を使う．
```rust
assert_eq!(image.size, (1024, 576));
assert_eq!(image.pixels.len(), 1024 * 576);
```

---
#### `pub` について1
- 構造体はデフォルトでプライベートであり，宣言されたモジュール内からしか見えない．
- 構造体をモジュールの外部からも見えるようにするには，定義の前に `pub` を付ける．
フィールドも同様．
```rust
/// 8ビットグレースケールピクセルの長方形
pub struct GrayscaleMap {
    pub pixels: Vec<u8>,
    pub size: (usize, usize)
}
```

---
#### `pub` について2
- 構造体を `pub` にして，フィールドをプライベートにすることもできる．
```rust
/// 8ビットグレースケールピクセルの長方形
pub struct GrayscaleMap {
    pixels: Vec<u8>,
    size: (usize, usize)
}
```

---
#### `pub` について3
- 構造体の値を作るには，構造体のすべてのフィールドが見えていなければならない，
- このため，`String` や `Vec` を構造体式では作れない．
- これらはすべてのフィールドがプライベートになっているため，これらの型を作るには，`Vec::new()` のようなパブリックメソッドを呼ぶ必要がある．

---
#### 他の構造体からのフィールド指定1
- 名前付きフィールド型構造体の値を作る際に，省略したフィールドを同じ型の他の構造体を使って指定することもできる．
- 構造体式で，名前付きフィールドの後ろに `..` と書くと，指定しなかったすべてのフィールドの値が，指定した同じ構造体型の値から取られる．

---
#### 他の構造体からのフィールド指定2
- ゲームに出て来るモンスターを表す構造体を考える．
```rust
struct Broom {
    name: String,
    height: u32,
    health: u32,
    position: (f32, f32, f32),
    intent: BroomIntent
}

/// Broomができる2つの活動
#[derive(Copy, Clone)]
enum BroomIntent { FetchWater, DumpWater }
```

---
#### 他の構造体からのフィールド指定3
- ゲームに出て来るモンスターを表す構造体を考える．
```rust
struct Broom {
    name: String,
    height: u32,
    health: u32,
    position: (f32, f32, f32),
    intent: BroomIntent
}

/// Broomができる2つの活動
#[derive(Copy, Clone)]
enum BroomIntent { FetchWater, DumpWater }
```

---
#### 余談
- この元ネタは「魔法使いの弟子 (The Sorcerer’s Apprentice)」だ．
- 新米の魔法使いがほうき (broom) に魔法をかけて自分の仕事を代わりにやらせるが，仕事が終わっても止めるさせる方法がわからない．
ほうきを斧で半分に切っても，ほうきが半分の長さの2本になるだけで，元のほうきと同じ盲目的な熱意で仕事を続ける．

---
#### 他の構造体からのフィールド指定4
```rust
// 値としてBroomを受け取り，所有権を得る．
fn chop(b: Broom) -> (Broom, Broom) {
// broom1の大半をbから作り高さだけを半分にする．
// String`はCopyではないので，broom1はbの名前の所有権を得る．
let mut broom1 = Broom { height: b.height / 2, .. b };

// broom2の大半をbroom1から作る.
// StringはCopyではないので，nameを明示的にクローンする．
let mut broom2 = Broom { name: broom1.name.clone(), .. broom1 };

// それぞれに別の名前を与える．
broom1.name.push_str(" I");
broom2.name.push_str(" II");
(broom1, broom2)
}
```

---
#### 他の構造体からのフィールド指定5
- ほうきを作って，半分に切り，何が起こるか見てみよう．
```rust
let hokey = Broom {
    name: "Hokey".to_string(),
    height: 60,
    health: 100,
    position: (100.0, 200.0, 0.0),
    intent: BroomIntent::FetchWater
};

let (hokey1, hokey2) = chop(hokey);
assert_eq!(hokey1.name, "Hokey I");
assert_eq!(hokey1.health, 100);
assert_eq!(hokey2.name, "Hokey II");
assert_eq!(hokey2.health, 100);
```

---
## 9.2　タプル型構造体
- 2つ目の種類の構造体は，タプルに似ているので **タプル型構造体** と呼ばれる．
```rust
struct Bounds(usize, usize);
```

- この型の値は，タプルを作るのと同じように作ることができるが，違いは構造体名を与えなければならないということだけだ．
```rust
let image_bounds = Bounds(1024, 768);
```

---
#### 要素とアクセス
- タプル型構造体に格納できる値を，タプルに格納する値と同様に **要素** (element) と呼ぶ．
- タプルの要素と同じようにアクセスできる．
```rust
assert_eq!(image_bounds.0 * image_bounds.1, 786432);
```

- `Bounds(1024, 768)` は関数呼び出しのように見えるだろうが，実際に型を定義すると関数が自動的に定義されている．
```rust
fn Bounds(elem0: usize, elem1: usize) -> Bounds { ... }
```

---
#### 名前付きフィールド型構造体とタプル型構造体の違い
- これらはとても良く似ている．
どちらを使うかは，読みやすさ，曖昧さ，簡潔さの問題．
- 値の要素にアクセスするのに `.` 演算子を使うことが多いのならば，名前でフィールドを指定するようにした方が，コードの読者に対してより多くの情報を与えることができ，打ち間違いに対しても頑健になるだろう．
- パターンマッチングで構成要素にアクセスすることが多いなら，タプル型構造体でも十分うまくいくだろう．

---
#### 新しい型
- タプル型構造体は **新しい型** を作るのに便利．
- 新しい型とは，構成要素が1つだけの構造体で，より厳密な型チェックを行うために定義する．
- 例えば，ASCII文字しか入っていないテキストを扱うのに，新しい型を次のように定義する．
```rust
struct Ascii(Vec<u8>);
```

- 新しい型を使うと，ASCII文字列を入力とする関数に間違って他のバイトバッファを渡してしまうようなミスをコンパイラが検出できる．

---
## 9.3　ユニット型構造体
- 3つ目の種類の構造体は，少しわかりにくい．
まったく要素を宣言しない構造体だ．
```RUST
struct Onesuch;
```

- このような型の値は，ユニット型 `()` の場合と同じようにメモリを消費しない．
- しかし，論理的にはこの空の構造体も，他の型と同様に値を持つ型であり，1つの状態しか取り得ない型になっている．
```rust
let o = Onesuch;
```

---
#### 範囲演算子 `..` との関連
- 実は，既に範囲演算子 `..` の説明のところで，ユニット型構造体は使われていた．
- `3..5` は，構造体値 `Range { start: 3, end: 5 }` の短縮形だが，両端の値を省略した式 `..` は，ユニット型構造体 `RangeFull` 値の短縮形．
- ユニット型構造体は，トレイトを扱う際にも役立つ（11章を参照）．

---
## 9.4　構造体のレイアウト
- メモリ上では，名前付きフィールド型構造体とタプル型構造体は同じ．
- 一連の値が，特定の方法でメモリ上に配置される．
```rust
struct GrayscaleMap {
    pixels: Vec<u8>,
    size: (usize, usize)
}
```

- `GrayscaleMap` の値は，図9-1に示すような形でメモリ上に置かれる．

---
#### C/C++との違い
- Rustは構造体のフィールドや要素の配置順について何も規定していない．
- 図9-1に示したのは可能な配置の1つにすぎないが，フィールドの値が構造体のメモリブロックに直接置かれることは保証されている．
- JavaScript，Python，Javaでは `pixels` や `size` はそれぞれヒープ上に取られたメモリブロック上に置かれ，`GrayscaleMap` のフィールドはそれを参照する．

---
#### Rustの場合
- Rustでは，`pixels` も `size` も直接 `GrayscaleMap` の値に埋め込まれる．
ベクタ `pixels` に所有されるヒープ上に取られたバッファだけが，独自のブロックに置かれる．
- `#[repr(C)]` 属性を付けることで，C/C++と互換性のある方法で，構造体をメモリ上に配置することができる（21章を参照）．

---
## 9.5　`impl` によるメソッド定義
- 任意の独自に定義した構造体に対して，メソッドを定義することができる．
- メソッドは，C++やJavaのように構造体定義の中に書くのではなく，別の `impl` ブロックに書く．
```rust
/// 文字の先入れ先出しキュー
pub struct Queue {
    older: Vec<char>,   // 古い要素たち，最も古いものが最後．
    younger: Vec<char>  // 新しい要素たち，最も新しいものが最後．
}
```

---
#### `impl` を使ったキューのメソッド実装
```rust
impl Queue {
    /// 文字をキューの末尾にプッシュ.
    pub fn push(&mut self, c: char) {
        self.younger.push(c);
    }

    /// キューの先端から文字をポップする．
    /// ポップする文字があれば，Some(c)を返し，空ならNoneを返す．
    pub fn pop(&mut self) -> Option<char> {
        if self.older.is_empty() {
            if self.younger.is_empty() {
                return None;
            }

            // youngerの要素をolderに移し，約束の順番に入れ替える．
            use std::mem::swap;
            swap(&mut self.older, &mut self.younger);
            self.older.reverse();
        }
        
        // ここに来たら，olderには何かが入っているはず．
        // VecのpopメソッドはOptionを返すので，そのまま返す．
        self.older.pop()
    }
}
```

---
#### `impl` ブロックについて
- `impl` ブロックは `fn` による関数定義の集合．
定義された関数が，ブロックの冒頭で指定された構造体型のメソッドとなる．
- メソッドは，特定の型に関連付けられているので，**関連付けられた関数** (associated function) とも呼ばれる．
- 「関連付けられた関数」と対をなしているのは，`impl` ブロックの外で定義される **自由関数** (free function) だ．

---
#### `self` について
- Rustは，呼び出される対象となる値を，最初の引数としてメソッドに与える．
この引数は，特別な名前 `self` でなければならない．
- `self` の型は明らか（`impl` ブロックの冒頭に書かれたものか，それへの参照）なため，型定義は省略することができる．
- `self: Queue`，`self: &Queue`，`self: &mut Queue` と書くのではなく，`self`，`&self`，`&mut self` と省略形で書くことができるということ．
- C++やJavaと違ってこの `self` は省略不可．
Pythonの `self`，JavaScriptの `this` と似ている．

---
####



---
#### おわりに
- zzz
