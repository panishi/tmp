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
#### 変更可能参照の場合
- `push` メソッドも `pop` メソッドも，`&mut self` を引数に取る
- メソッドを呼び出すときに，自分で変更可能参照を借用する必要はない．
通常のメソッド呼び出し構文が暗黙によろしくやってくれる．
```rust
let mut q = Queue { older: Vec::new(), younger: Vec::new() };
q.push('0');  // (&mut q).push('0')と書かなくてよい
q.push('1');
assert_eq!(q.pop(), Some('0'));
q.push('∞');
assert_eq!(q.pop(), Some('1'));
assert_eq!(q.pop(), Some('∞'));
assert_eq!(q.pop(), None);
```

---
#### 共用参照の場合
- `self` を変更しないメソッドであれば，共有参照を取るように定義すればよい．
```rust
impl Queue {
    pub fn is_empty(&self) -> bool {
        self.older.is_empty() && self.younger.is_empty()
    }
}

assert!(q.is_empty());
q.push('☉');
assert!(!q.is_empty());
```

---
#### 値の場合1
- メソッドが `self` の所有権を取得したいのであれば，`self` を値で受けることもできる．
```rust
impl Queue {
    pub fn split(self) -> (Vec<char>, Vec<char>) {
        (self.older, self.younger)
    }
}
```

---
#### 値の場合2
```rust
let mut q = Queue { older: Vec::new(), younger: Vec::new() };

q.push('P');
q.push('D');
assert_eq!(q.pop(), Some('P'));
q.push('X');

let (older, younger) = q.split();
// qは未定義状態になった
assert_eq!(older, vec!['D']);
assert_eq!(younger, vec!['X']);
```

---
#### スタティックメソッド
- `self` を引数として取らないメソッドは，構造体の特定の値にではなく，構造体そのものに関連付けられた関数となる．
- C++やJavaと同様，Rustではこのようなメソッドを **スタティックメソッド** (static method) と呼ぶ．

---
#### スタティックメソッドと `new`
```rust
impl Queue {
    pub fn new() -> Queue {
        Queue { older: Vec::new(), younger: Vec::new() }
    }
}

let mut q = Queue::new();
q.push('*');
...
```

- コンストラクタ関数を `new` とするのは，Rustの慣習．
- ただし，`new` はキーワードではない．
`Vec::with_capacity` 等もある．

---
#### 複数の `impl` ブロック
- 1つの型に対して，複数の `impl` ブロックを書くこともできるが，すべての `impl` ブロックはその型を定義したのと同じクレートに入っていなければならない．
- しかし，Rustでは独自のメソッドを他の型に付けることもできる（11章　トレイトとジェネリクス」を参照）．

---
#### `impl` ブロックのメリット
- 型のデータメンバを見つけるのが簡単．
Rustでは1箇所にデータメンバがまとまっている．
- 名前付き構造体の構文にメソッド定義を入れる方法は想像できるが，タプル型やユニット型の構造体にメソッドをきれいに入れるのは難しい．
メソッドを `impl` ブロックに入れることで，3つの構文を1つにすることができる．
- 同じ `impl` 構文が，トレイトを実装する際にも使える．
11章で詳しく説明する．

---
## 9.6　ジェネリック構造体
- Rustの構造体は **ジェネリック** (generic) にすることができる．
- 構造体の定義がテンプレートになっていて，任意の型をプラグインできることを意味する．
```rust
pub struct Queue<T> {
    older: Vec<T>,
    younger: Vec<T>
}
```

---
#### ジェネリック構造体
- `Queue<T>` の `<T>` は，「任意の要素型 `T` に対して」と読む．
つまりこの定義は，「任意の要素型 `T` に対して，`Queue<T>` は2つの `Vec<T>` 型フィールドを持つ」と読む．
- `Queue<char>` では `T` は `char` なため，最初に示した `char` に特化した構造体とまったく同じになる．

---
#### ジェネリック構造体に対する `impl` ブロック
```rust
impl<T> Queue<T> {
    pub fn new() -> Queue<T> {
        Queue { older: Vec::new(), younger: Vec::new() }
    }

    pub fn push(&mut self, t: T) {
        self.younger.push(t);
    }

    pub fn is_empty(&self) -> bool {
        self.older.is_empty() && self.younger.is_empty()
    }
    ...
}
```

---
#### `Self`
- 仮引数 `self` には型を省略する書き方を使っている．
`Queue<T>` をあちこちに書くのは，面倒だし読みにくいためだ．
- ジェネリックであるなしに関わらず，すべての `impl` ブロックで特別な型パラメータ `Self` が，メソッドを追加する対象の型として定義される（キャメルケースになっていることに注意）．
```rust
pub fn new() -> Self {
    Queue { older: Vec::new(), younger: Vec::new() }
}
```

---
#### ジェネリクスと構造体式
- `new` のボディ部では，型パラメータを構造体式に使っていない．
`Queue { ... }` と書くだけで十分で，Rustの型推論があとはよろしくやってくれる．
- 関数のシグネチャと型宣言では常に型パラメータを与えなければならない．
コンパイラはこれらの部分を推論せず，これらの部分で明示的に指定された型を使って，関数ボディ部の型を推論する．

---
#### ターボフィッシュ
- スタティックメソッド呼び出しの際は，型パラメータをターボフィッシュ `::<> ` を用いて明示的に与えることができる．
```rust
let mut q = Queue::<char>::new();
```

実際には，Rustに推論させることが多い．
```rust
let mut q = Queue::new();
let mut r = Queue::new();
q.push("CAD");   // 明らかにQueue<&'static str>
r.push(0.74);    // 明らかにQueue<f64>
q.push("BTC");   // Bitcoins per USD, 2017-5
r.push(2737.7);  // Rustは根拠なき熱狂を検出し損ねた
```

---
## 9.7　生存期間パラメータを持つ構造体
- 構造体型が参照を含むのであれば，参照の生存期間を指定する必要がある．
- スライスの最大値と最小値の要素への参照を保持する構造体を考えてる．
```rust
struct Extrema<'elt> {
greatest: &'elt i32,
least: &'elt i32
}
```

---
#### `struct Extrema<'elt> `
- `struct Queue<T>` は，「任意の型 `T` に対して，その型を保持する構造体 `Queue<T>` を作ることができる」という意味．
- 同様に，`struct Extrema<'elt>` は，「任意の生存期間 `'elt` に対して，生存期間が `'elt` の参照を保持する `Extrema<'elt>` を作ることができる」という意味になる．

---
#### `find_extrema<'s>` 1
- 次の関数は，スライスをスキャンして要素を参照する `Extrema` 値を返す．
```rust
fn find_extrema<'s>(slice: &'s [i32]) -> Extrema<'s> {
    let mut greatest = &slice[0];
    let mut least = &slice[0];
    for i in 1..slice.len() {
        if slice[i] < *least { least = &slice[i]; }
        if slice[i] > *greatest { greatest = &slice[i]; }
    }
    Extrema { greatest, least }
}
```

---
#### `find_extrema<'s>` 2
- `find_extrema` は生存期間 `'s` を持つ `slice` の要素を借用しているので，返却する `Extrema` 構造体も参照の生存期間として `'s` を使わなければならない．
- Rustコンパイラは常に関数呼び出し時に生存期間パラメータを推論するので，`find_extrema` 呼び出しの際に明示的に書く必要はない．
```rust
let a = [0, -3, 0, 15, 48];
let e = find_extrema(&a);
assert_eq!(*e.least, -3);
assert_eq!(*e.greatest, 48);
```

---
#### `find_extrema<'s>` 3
- 返り値の型と引数が同じ生存期間を使うことが一般的なので，明らかな候補が1つしかない場合には，返り値の生存期間を省略できる．
- `find_extrema` のシグネチャを次のように書いても意味は変わらない．
```rust
fn find_extrema(slice: &[i32]) -> Extrema {
...
}
```

- `Extrema<'static>` を意図することもあるかもしれないが一般的とは言えないので，Rustは一般的な場合に対して省略形を提供する。

---
## 9.8　一般的なトレイトの自動実装
- Rustでは，標準的なトレイトとして `Copy`，`Clone`，`Debug`，`PartialEq` 等が提供されている．
- 上のような標準的なトレイトに対しては，動作をカスタマイズしたいのでない限り，自分で実装する必要はない．
- `#[derive]` 属性を付けるだけでいい．
```rust
#[derive(Copy, Clone, Debug, PartialEq)]
struct Point {
    x: f64,
    y: f64
}
```

---
#### トレイトの自動実装
- これらのトレイトは，すべてのフィールドがこれらのトレイトを実装しているならば自動的に実装できる．
- `Point` に対して `PartialEq` を自動的に実装できるのは，2つのフィールドがいずれも `f64` で，`f64` が `PartialEq` を実装しているためだ．
- 点には自明な順序が存在しないため，`PartialOrd` はあえて実装していない．
コピー可能，クローン可能などの性質は，構造体のパブリックAPIの一部となってしまうので，慎重に選択しなければならない．


---
## 9.9　内部可変性
- 可変性は多すぎると面倒なことになるが少しはないと困る．
- クモ型ロボットの制御システムを考える．
中心になる構造体 `SpiderRobot` には，設定とI/Oハンドルが格納される．
この構造体はロボットが起動する際に設定され，値は変わらない．
```rust
pub struct SpiderRobot {
    species: String,
    web_enabled: bool,
    leg_devices: [fd::FileDesc; 8],
    ...
}
```

---
#### クモ型ロボットの制御システム
- ロボット内の主要サブシステムは，それぞれ別の構造体で制御されるが，これらの構造体はすべて `SpiderRobot` を参照している．
- `Rc` は参照カウント (reference count) ．
`Rc` に収められている値は常に共有可能で，したがって常に不変．
```rust
use std::rc::Rc;
pub struct SpiderSenses {
    robot: Rc<SpiderRobot>,  // 設定とI/O へのポインタ
    eyes: [Camera; 32],
    motion: Accelerometer,
    ...
}
```

---
#### ログ機能
- `SpiderRobot` 構造体に標準の `File` タイプを使って，簡単なログ機能を付けてみる．
- ここで問題が起きる．
書き出すためのすべてのメソッドは `mut` 参照を要求するため，`File` 型は `mut` でなければならない．

---
#### 内部可変性
- このような状況はよく生じる．
欲しいのは，「ほんの少しだけ可変データ (`File`) を含んだ，ほとんどの部分が不変な値（`SpiderRobot` 構造体）だ．
これを **内部可変性** (interior mutability) という．
- Rustにはこれを実現する方法がいくつかある．
ここでは，`Cell<T>` と `RefCell<T>` を説明する．
これらはいずれも `std::cell` モジュールに含まれる．

---
#### `Cell<T>` 構造体1
- `Cell<T>` 構造体は，型 `T` のプライベートな値を1つだけ持つ．
`Cell` 型が特別なのは，`Cell` そのものに対する `mut` な参照を持っていなくても，そのフィールドを見たりセットしたりできる点だけだ．
- `Cell::new(value)`：新しい `Cell` を作り，与えられた値 `value` を中に移動する．
- `cell.get()`：`cell` の中の値のコピーを返す．

---
#### `Cell<T>` 構造体2
- `cell.set(value)`：与えられた値を `cell` の中にセットし，以前の値をドロップする．
このメソッドは，`self` を `mut` でない参照として受け取る．
- もちろんこれは，`set` という名前のメソッドとしては普通ではないが，まさにそれこそが `Cell` の存在理由となっている．
これは変更不可のルールを安全に曲げる方法で，それ以上でもそれ以下でもない．

---
#### `Cell<T>` 構造体3
`Cell` は，`SpiderRobot` に単純なカウンタを追加するのにも使える．
```rust
use std::cell::Cell;

pub struct SpiderRobot {
    ...
    hardware_error_count: Cell<u32>,
    ...
}
```

---
#### `Cell<T>` 構造体4
`SpiderRobot` の `mut` でないメソッドからでもこの `u32` に対して `.get()` メソッドや `.set()` メソッドでアクセスできる．
```rust
impl SpiderRobot {
    /// エラーカウンタを1増やす
    pub fn add_hardware_error(&self) {
        let n = self.hardware_error_count.get();
        self.hardware_error_count.set(n + 1);
    }

    /// ハードウェアエラーが報告されていたら真を返す
        pub fn has_hardware_errors(&self) -> bool {
        self.hardware_error_count.get() > 0
    }
}
```

---
#### `RefCell<T>` 構造体1
`Cell` は，共有されている変数に対して `mut` メソッドを呼ばせないため，これだけではログの問題は解決できない．
- `.get()` はセルに収められた値のコピーを返すので，`Copy` トレイトを実装した `T` にしか使えないが，ログには可変な `File` が必要で `File` はコピーできない．
- この場合にふさわしい道具は，`RefCell` だ．
`RefCell<T>` は `Cell<T>` と同様，型 `T` の値を1つだけ持つジェネリック型．
- `Cell` と違って，`RefCell` は `T` 値への参照の借用をサポートしている．

---
#### `RefCell<T>` 構造体2
- `RefCell::new(value)`：新しい `RefCell` を作り，値 `value` をその中に移動する．
- `ref_cell.borrow()`：`ref_cell` に収められた値への共有参照を，`Ref<T>` として返す．
このメソッドは，値が既に可変で借用されていた場合にはパニックを起こす．
- `ref_cell.borrow_mut()`：`ref_cell` に収められた値への可変参照を，`RefMut<T>` として返す．
このメソッドは，値が既に借用されていた場合にはパニックを起こす．

---
#### `RefCell<T>` 構造体3
- 2つの `borrow` メソッドは，「`mut` 参照は排他的な参照でなければならない」というRustのルールを破るとパニックを起こす．
```rust
let ref_cell: RefCell<String> = RefCell::new("hello".to_string());

let r = ref_cell.borrow();  // OK, Ref<String>を返す
let count = r.len();        // OK, "hello".len()を返す
assert_eq!(count, 5);

let mut w = ref_cell.borrow_mut();  // パニック：既に借用されている
w.push_str(" world");
```

---
#### `RefCell<T>` 構造体4
- パニックを避けるには，2つの借用を別のブロックに置けばよい．
そうすれば，`w` を借用しようとする前に，`r` はドロップされる．
- 通常の参照の場合，変数から参照を借用しようとすると，Rustが **コンパイル時** に参照が安全に使われていることを保証するためのチェックを行う．
- `RefCell` の場合は，同じルールを実行時チェックで実現するため，ルールに違反すればパニックが起きる．

---
#### `RefCell` を用いたログの実現1
```rust
pub struct SpiderRobot {
    ...
    log_file: RefCell<File>,
    ...
}

impl SpiderRobot {
    /// ログファイルに1行書き出す
    pub fn log(&self, message: &str) {
        let mut file = self.log_file.borrow_mut();
        writeln!(file, "{}", message).unwrap();
    }
}
```

---
#### `RefCell` を用いたログの実現2
- 変数 `file` の型は `RefMut<File>` になっている．
- この変数は，`File` の可変参照と同じようにして使うことができる．

---
#### セルの問題点
- セルとそれを含むすべての型は，スレッド安全ではないため，`Rust` では複数のスレッドからセルにアクセスすることを許さない．
- スレッド安全な内部可変性については19章で詳しく述べる．

---
#### おわりに
- 構造体は，名前付きフィールド型であれ，タプル型であれ，値を集めたものだ．
- `SpiderSenses` 構造体であれば，`Rc` 構造体への共有参照「と」，目「と」，加速度計「と」，というようにさまざまな値を持つ．
- 構造体の本質は，この「と」という言葉にある．
- 「と」ではなく「か」をベースにした型も便利なので，Rustでは多用されている．
これが次の章の主題だ．
