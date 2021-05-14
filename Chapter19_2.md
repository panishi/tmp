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

<style>
img[alt~="center"] {
  display: block;
  margin: 0 auto;
}
</style>

# プログラミングRust
19.2　チャネル

2021/5/14

---
## 19.2　チャネル
- チャネルは，あるスレッドから別のスレッドに値を送信する一方通行のパイプ．
- スレッド安全なキューと考えてもよい．

---
#### チャネルの使い方1
- チャネルはUnixのパイプに似ている．通常，両端の所有者は別のスレッドとなる．
![center height:450](img/Figure19-5.png)

---
#### チャネルの使い方2
- チャネルはRustの値を送信する．
`sender.send(item)`: 値が1つチャネルに置かれる．
`receiver.recv()`: 値を1つ取り除く．値の所有権は送信スレッドから受信スレッドに移る．
- チャネルが空なら，`receiver.recv()` は値が送信されるまでブロック
する．

---
#### チャネルは実は新しい技術ではない
- Erlangは，隔離されたプロセスとメッセージ通信を30年前から使っている．
- Unixのパイプはほとんど50年前からある．
- パイプは柔軟な組み合わせを可能にするためのもので，並列性のためにも役立つ．

---
#### Unixパイプラインの例
![center height:550](img/Figure19-6.png)

---
#### Rustのチャネルの効率性
- Rustのチャネルは値の送信がコピーではなく移動で行われるため，Unixのパイプよりも高速．
- 値の移動は．数メガバイトのデータを持つ構造体の場合であっても，高速に実行できる．

---
### 19.2.1　値の送信
- 転置インデックスを作る並列プログラムをチャネルを使って構築する．
    - 検索エンジンを構築する際の主要な要素の1つ．
    - どの単語がどの文書に現れたかを管理するデータベース．
- プログラム全体は，https://github.com/ProgrammingRust/fingertips にある．長くはなく，全部で千行ぐらいだ．

---
#### インデックスを作るパイプライン
![center height:550](img/Figure19-7.png)

---
#### 個々のスレッドのタスク
- 最初のスレッドは，単にディスクからメモリにソースドキュメントを1つずつ読み込むだけ（これをスレッドで実行しているのは，コードを可能な限り単純に書くためにブロッキングAPIの `File::open` と `read_to_string` を使っているため．ディスクが動いている間CPUを遊ばせておきたくない）．
- このステージの出力は，1つのドキュメントを表す長い `String`．
- このスレッドは，`String` のチャネルで次のスレッドにつながれている．

---
#### ファイルから読み出すスレッドの起動1
- まずファイルから読み出すスレッドを起動するところから始まる．
- `documents` はファイル名のベクタ `Vec<PathBuf>`． だ．
```rust
use std::fs::File;
use std::io::prelude::*;  // Read::read_to_stringのため
use std::thread::spawn;
use std::sync::mpsc::channel;

let (sender, receiver) = channel();
```

---
#### ファイルから読み出すスレッドの起動2
```rust
let handle = spawn(move || {
    for filename in documents {
        let mut f = File::open(filename)?;
        let mut text = String::new();
        f.read_to_string(&mut text)?;

        if sender.send(text).is_err() {
            break;
        }
    }
    Ok(())
});
```

---
#### チャネルの作成
```rust
let (sender, receiver) = channel();
```
- `channel` 関数は，センダとレシーバのペアを返す．
- 内部のキューデータ構造は標準ライブラリによって隠蔽されていて，実装の詳細は見えない．
- チャネルはファイルのテキストを送信するために使う．
    - `sender` の型は `Sender<String>`．
    - `receiver` の型は `Receiver<String>`．
- 明示的に `channel::<String>()` のようにも書けるが，ここではRustの型推論に任せている．

---
#### スレッドの起動
```rust
let handle = spawn(move || {
```
- `std::thread::spawn` を用いてスレッドを起動する．
- `sender` の所有者は，`move` クロージャを通じて新しいスレッドに移る（`receiver` はそうならない）．

---
#### ディスクからの読み出しとチャネルへの送信
- 単にディスクから読み出しているだけ．
```rust
    for filename in documents {
        let mut f = File::open(filename)?;
        let mut text = String::new();
        f.read_to_string(&mut text)?;
```
- 読み出しに成功したら，テキストをチャネルに送信する．
```rust
        if sender.send(text).is_err() {
            break;
        }
    }
```

---
#### 移動のコスト
- `sender.send(text)` は，値 `text` をチャネルに移動する．
- この値は最終的には，値を受け取る主体に移動される．
- この移動操作には3ワード（`String` のサイズ）分のコピーしかかからない．
- 対応する `receiver.recv()` も3ワードのコピーで行われ，低コストである．

---
#### `send` と `recv` が失敗する場合
`send` メソッドと `recv` メソッドは `Result` を返すが，これらが失敗するのはチャネルの相手がドロップされた場合のみ．
- `Receiver` がドロップされると，`send` は失敗する（書き込んだ値がいつまでもチャネルの中に残ることになるため）．
- `recv` は，チャネルの中に値がなく `Sender` がドロップされている場合に失敗する（`recv` が永久に待ち続けることになるため）．
- チャネルの一端をドロップすることは，接続をクローズするためによく行われる．

---
#### `sender.send(text)` が失敗する場合
- 失敗が起きるのは，受信スレッドが先に終了してしまった場合だけ．
- 意図的であってもエラーであっても，このようなことが起こった場合には，このファイル読み出しスレッドも黙って終了すればよい．

---
#### 戻りは `Result`
- このようなことが起こった場合，もしくはスレッドがすべてのドキュメントを読み終わった場合には，`Ok(())` を返す．
```rust
    Ok(())
});
```
- スレッドは，I/Oエラーにあたってしまった場合は即座に終了する．
- エラーはそのスレッドの `JoinHandle` に格納される．

---
#### エラー処理の方法
- エラーが起こったら `println!` で出力するだけで，次のファイルに進んでもいい．
- データに使うのと同じチャネルを使ってエラーを送ることもできる．この場合，チャネルには `Result` を送ることになる．
- 上例のアプローチは，軽量だが人任せにしない方法だ．`?` 演算子を使うと余分な周辺コードは必要ないし，`try/catch` もいらない．それでいてエラーを無視しているわけでもない．

---
#### `start_file_reader_thread`
```rust
fn start_file_reader_thread(documents: Vec<PathBuf>)
    -> (Receiver<String>, JoinHandle<io::Result<()>>)
{
    let (sender, receiver) = channel();

    let handle = spawn(move || {
        ...
    });

    (receiver, handle)
}
```
- この関数は新しいスレッドを作ったらすぐリターンする．パイプラインの各ステージに対してこのような関数を書いていく．

---
### 19.2.2　値の受信
- 値を送るループを実行するスレッドはできたので，これで `receiver.recv()` をループで呼び出す2つ目のスレッドを起動することができる．
```rust
while let Ok(text) = receiver.recv() {
    do_something_with(text);
}
```

---
#### ループの別な書き方
- `Receiver` はイテレート可能なので，下のように書くこともできる．
```rust
for text in receiver {
    do_something_with(text);
}
```
- 2つのループのどちらを書いても挙動は同じ．

---
#### ループの挙動
- 受信スレッドの制御がループの冒頭に移ったときに，たまたまチャネルが空なら，他のスレッドが値を送るのを待つ．
- チャネルが空で，`Sender` がドロップされた場合（ファイル読み出しスレッドが終了した場合），このループは正常に終了する．
- ファイル読み出しスレッドは，変数 `sender` を所有するクロージャを実行している．クロージャが終了すると，`sender` がドロップされる．

---
#### `start_file_indexing_thread` 1
```rust
fn start_file_indexing_thread(texts: Receiver<String>)
    -> (Receiver<InMemoryIndex>, JoinHandle<()>)
{
    let (sender, receiver) = channel();

    let handle = spawn(move || {
        for (doc_id, text) in texts.into_iter().enumerate() {
            let index = InMemoryIndex::from_single_document(doc_id, text);
            if sender.send(index).is_err() {
                break;
            }
        }
    });

    (receiver, handle)
}
```

---
#### `start_file_indexing_thread` 2
- 1つのチャネル (`texts`) から `String` 値を受け取り，別のチャネル (`sender` / `receiver`) に `InMemoryIndex` 値を書き出すスレッドを起動する．
- このスレッドの仕事は，第1ステージでロードされたファイルのテキストを受け取り．それぞれのドキュメントをメモリ上の．1ファイル用の小さな転置インデックスに変換することだ．
- このスレッドのメインループは単純で，ドキュメントのインデックスを作るのは 関数 `make_single_file_index` に任せている．
- このステージではI/Oを行わないので，`io::Error` を処理する必要はない．したがって，`io::Result<()>` ではなく `()` を返す．

---
### 19.2.3　パイプラインの実行
- 残り3つのステージは設計的には類似している．それぞれが，直前のステージで作られた `Receiver` を消費する．
- ここではコードは示さず．3つの関数のシグネチャだけを示す．ソースコードはオンラインにある．

---
#### 第3ステージ
- まずインデックスを，手に負えなくなるまでメモリ上でマージする．
```rust
fn start_in_memory_merge_thread(file_indexes: Receiver<InMemoryIndex>)
    -> (Receiver<InMemoryIndex>, JoinHandle<()>)
```

---
#### 第4ステージ
- それをディスクに書き出す．
```rust
fn start_index_writer_thread(big_indexes: Receiver<InMemoryIndex>,
output_dir: &Path)
    -> (Receiver<PathBuf>, JoinHandle<io::Result<()>>)
```

---
#### 第5ステージ
最後に，大きなファイルがいくつかできていたら，ファイルベースのマージアルゴリズムを使ってマージする．
```rust
fn merge_index_files(files: Receiver<PathBuf>, output_dir: &Path)
    -> io::Result<()>
```
- 最後のステージはパイプラインの末尾なので，`Receiver` を返さない．
- このステージはわざわざ新しいスレッドを作らず，呼び出しスレッドで実行するため，`JoinHandle` を返すこともない．

---
#### `run_pipeline` 1
```rust
fn run_pipeline(documents: Vec<PathBuf>, output_dir: PathBuf)
    -> io::Result<()>
{
    // パイプラインの5つのステージをすべて起動する．
    let (texts, h1) = start_file_reader_thread(documents);
    let (pints, h2) = start_file_indexing_thread(texts);
    let (gallons, h3) = start_in_memory_merge_thread(pints);
    let (files, h4) = start_index_writer_thread(gallons, &output_dir);
    let result = merge_index_files(files, &output_dir);

    // スレッドが終了するのを待つ．エラーは保存する．
    let r1 = h1.join().unwrap();
    h2.join().unwrap();
    h3.join().unwrap();
    let r4 = h4.join().unwrap();
```

---
#### `run_pipeline` 2
```rust
    // エラーが起きた場合には，最初に起きたエラーを返す．
    // （ここでは，h2とh3は失敗しない．純粋にメモリ上のデータ処理なので．）
    r1?;
    r4?;
    result
}
```
- 以前示した例と同じように，`.join().unwrap()` を用いて明示的に子スレッドのパニックを伝播させている．
- それ以外に変わっている点は，`?` をすぐに使わず，すべてのスレッドが終了するまで `io::Result` 値を取っておいていることだ．

---
#### パイプラインのパフォーマンス1
- このパイプラインは，シングルスレッド版よりも40%速い．
- 半日仕事の成果としては悪くないが，マンデルブロプログラムの675% に比べると物足りなさを感じる．
- 明らかに，システムのI/O容量や，CPUコアが飽和したわけではない．

---
#### パイプラインのパフォーマンス2
- パイプラインは，工場の組立ラインのようなもので，性能は最も遅いステージのスループットに制約される．
- 作りたてのチューニングされていない組立ラインは，単品生産よりも遅いかもしれないが，組立ラインの利点はチューニングできることにある．
- 計測してみると，2つ目のステージがボトルネックになっている．
- `.to_lowercase()` と `.is_alphanumeric()` はUnicodeのテーブルを参照するので時間がかかる．これ以降のステージは，ほとんどの時間を`Receiver::recv` で入力を待ってスリープしている．

---
#### パイプラインのパフォーマンス3
- これは性能を向上する余地があることを示している．ボトルネックに対応すれば，並列度は向上する．
- プログラムは独立したコンポーネントで構成されているので，このボトルネックに対応する方法を考えるのは簡単．第2ステージを手で最適化することもできるし，仕事を2つ以上のステージに分割してもいい．複数のファイルインデックススレッドを実行してもいいだろう．

---
### 19.2.4　チャネルの機能と性能
- `std::sync::mpsc` の `mpsc` 部分は，**multi-producer, single-consumer**（**複数の生産者，単一の消費者**）を意味する．
- Rustのチャネルが提供する種類の通信を表現したもの．

---
#### 多数のクライアントスレッドからのリクエストを1つのスレッドで処理する場合
- サンプルプログラムのチャネルは，1つの送信者から1つの受信者に値を運ぶだけだった（最も一般的な使い方）．
- Rustのチャネルは複数の送信者もサポートしている．
![center height:350](img/Figure19-8.png)

---
#### `Sender<T>` と `Receiver<T>`
- `Sender<T>` は `Clone` トレイトを実装している．
    - 複数の `Sender` を持つチャネルを得るには，普通のチャネルを作ってから必要なだけ `Sender` をクローンすればよい．
    - それぞれの `Sender` を別のスレッドに移動することもできる．
- `Receiver<T>` はクローンできない．
    - 1つのチャネルから複数のスレッドで値を受け取りたければ，`Mutex` が必要になる（本章の後ろの方で示す）．

---
#### Rustによる最適化1
- Rustのチャネルは，注意深く最適化されている．
- 最初にチャネルを作る際には，特殊な「1回限りの」キュー実装が用いられる．オブジェクトを1つだけ送る場合に，オーバーヘッドが最小になるように設計されている．
- 2つ目の値を送ると，別のキュー実装に切り替える．

---
#### Rustによる最適化2
- `Sender` をクローンすると，Rustはまた別の実装に切り替える．
- この実装は、複数のスレッドが同時に値を送っても安全なように設計されている．
- 3つの実装の中では最も遅いが，ロックフリーキューなので，値を送信するのも受信するのも高速．それぞれに対して，最大でも数回のアトミック操作，ヒープ確保，値の移動のコストしかかからない．
- システムコールが必要になるのは，キューが空で受信スレッドがスリープする場合だけであり，そのような場合は，どのみちチャネルを通過する通信速度が問題になることはない．

---
#### 性能低下の可能性1
- これら最適化がすべて機能しても，チャネルの性能を低下させてしまうアプリケーションのバグが考えられる．
- 受信スレッドが処理できるよりも速く値を送信してしまうと，チャネル内にバックログがどんどん溜まっていく．
- 例えば，ファイル読み込みスレッド（第1ステージ）は，ファイルインデックススレッド（第2ステージ）がインデックスするよりもはるかに速くファイルを読み込むことができる．結果，数百メガバイトの生データがディスクから読み込まれキューに押し込められることになる．

---
#### 性能低下の可能性2
- この種のよくない挙動は，メモリを消費し，局所性を下げる．
- さらに悪いことに，送信スレッドが動き続け，CPUなどのシステム資源を使い尽くしてさらに値を送ろうとするため，受信側が必要な資源を得られなくなってしまうこともある．

---
#### 同期チャネル1
- ここでも，RustはUnixのパイプからアイディアを得ている．
- Unixは，速すぎる送信者を強制的に遅くするため，バックプレッシャによる仕掛けを用いている．
    - 個々のパイプには固定されたサイズがあり，一時的にフルになってしまったパイプに書き込もうとすると，そのプロセスはパイプに空きができるまでシステムによってブロックされる．
- Rustの場合，この機構は **同期チャネル** と呼ばれる．
```rust
use std::sync::mpsc::sync_channel;

let (sender, receiver) = sync_channel(1000);
```

---
#### 同期チャネル2
- 同期チャネルは通常のチャネルとほぼ同じだが，作成時に保持できる値の数を指定できる．
- 同期チャネルでは，`sender.send(value)` がブロックする可能性のある操作となる．
- `start_file_reader_thread` の `channel` を，32個の値を上限とする `sync_channel` に変更したところ，ベンチマークデータセット対比で，スループットを減らすことなくメモリ消費量の2/3を削減することができた．

---
### 19.2.5　スレッド安全性： `Send` と `Sync`
- これまで，すべての値をスレッド間で自由に移動・共有できるかのように書いてきた．
- これはほとんど正しいが，Rustの完全なスレッド安全性を実現しているのは，実は2つの組み込みトレイト `std::marker::Send` と `std::marker::Sync` だ．
    - `Send` を実装する型は他のスレッドに値で渡しても安全．スレッド間で移動もできる．
    - `Sync` を実装する型は他のスレッドに非 `mut` 参照で渡しても安全．スレッド間で共有ができる．

---
#### `process_files_in_parallel` の例
- 「19.1.1　`spawn` と `join`」で示した `process_files_in_parallel` では，クロージャを使って `Vec<String>` を親スレッドから個々の子スレッドに渡していた．
- ベクタとそれに含まれる文字列は，親スレッドで確保され，子スレッドで解放される．
- `Vec<String>` が `Send` を実装しているということは．この操作が安全だとAPIが保証していることを意味する．
- `Vec` や `String` が内部的に用いているメモリアロケータはスレッド安全だ．

---
#### ほとんどの型は `Send` であり `Sync` でもある1
![center height:550](img/Figure19-9.png)

---
#### ほとんどの型は `Send` であり `Sync` でもある2
- 独自の構造体や列挙型にこれらのトレイトを実装するには，`#[derive]` を使う必要すらない．
    - フィールドがすべて `Send` なら `Send` になる．
    - フィールドがすべて `Sync` なら `Sync` になる．

---
#### `Send` でも `Sync` でもない型
- ほとんどすべてスレッド安全でない方法で可変性を使う型．
- 参照カウントスマートポインタの `std::rc::Rc<T>` を考える．
![center height:450](img/Figure19-10.png)

---
#### `Rc<String>` の例1
- `Rc<String>` をスレッド間で共有できたとすると，図19-10に示したように，2つのスレッドが同時に共有参照カウントをインクリメントしようとするとデータ競合が発生し，参照カウントは正確でなくなってしまう．
- これは，メモリ解放後の使用や，二重解放などの未定義動作を引き起こす

---
#### `Rc<String>` の例2
- もちろん，Rustはこのような事態を防いでくれる．
```rust
use std::thread::spawn;
use std::rc::Rc;

fn main() {
    let rc1 = Rc::new("hello threads".to_string());
    let rc2 = rc1.clone();
    spawn(move || {  // エラー
        rc2.clone();
    });
    rc1.clone();
}
```

---
#### `Rc<String>` の例3
- エラーメッセージは以下の通り．
```
error[E0277]: the trait bound `Rc<String>: std::marker::Send` is not satisfied
              in `[closure@...]`
  --> concurrency_send_rc.rs:10:5
   |
10 |     spawn(move || { // error
   |     ^^^^^ within `[closure@...]`, the trait `std::marker::Send` is not
   |           implemented for `Rc<String>`
   |
   = note: `Rc<String>` cannot be sent between threads safely
   = note: required because it appears within the type `[closure@...]`
   = note: required by `std::thread::spawn`
```

---
#### スレッド安全性のための制約
- Rustがスレッド安全性を保証するために，`Send` や `Sync` が役立っていることがわかる．
- これらのトレイトは，データをスレッド間の境界を超えて転送する `spawn` などの関数の型シグネチャに制約として現れる．
- スレッドをspawn する場合，引数として渡すクロージャは `Send` でなければならない．つまり，このクロージャが含むすべての値が `Send` でなければならない．
- チャネルを通じて他のスレッドに値を送りたいなら，その値も `Send` でなければならない．

---
### 19.2.6　ほとんどすべてのイテレータをつなげられるチャネル
- 前節で書いた転置インデックスビルダはパイプラインとして書かれていて，手動でチャネルを設定することでスレッドを立ち上げていた．
- 「15 章　イテレータ」で作ったイテレータパイプラインは，わずか数行のコードにより多くのことを押し込んでいるように見える．スレッドのパイプラインでも同じようなことができないだろうか？

---
#### イテレータのパイプラインとスレッドのパイプラインの統合
```rust
documents.into_iter()
    .map(read_whole_file)
    .errors_to(error_sender)  // エラーをフィルタで取り除く
    .off_thread()             // 上の仕事のためにスレッドを起動
    .map(make_single_file_index)
    .off_thread()             // 第2ステージのためにスレッドを起動
...
```

---
#### `OffThreadExt` 1
- トレイトを使うと標準ライブラリ型に対してもメソッドを追加することができ，上記を実現することができる．
```rust
use std::sync::mpsc;

pub trait OffThreadExt: Iterator {
    /// このイテレータをオフスレッドイテレータに変換する．
    /// next()の呼び出しは別のワーカスレッドで行われるので，
    /// イテレータとループのボディが並列に実行される．
    fn off_thread(self) -> mpsc::IntoIter<Self::Item>;
}
```

---
#### `OffThreadExt` 2
- このトレイトをイテレータ型に対して実装する．
- `mpsc::Receiver` がもともとイテレート可能型なのでこれは楽．
```rust
use std::thread::spawn;

impl<T> OffThreadExt for T
    where T: Iterator + Send + 'static,
          T::Item: Send + 'static
{
    fn off_thread(self) -> mpsc::IntoIter<Self::Item> {
        // ワーカスレッドからアイテムを転送するためのチャネルを作る．
        let (sender, receiver) = mpsc::sync_channel(1024);
```

---
#### `OffThreadExt` 3
```rust
        // このイテレータを新しいワーカスレッドに移動し，そこで実行する．
        spawn(move || {
            for item in self {
                if sender.send(item).is_err() {
                    break;
                }
            }
        });
        // そのチャネルから値を引き出すイテレータを返す．
        receiver.into_iter()
    }
}
```

---
#### `Where` 節について1
- 「11.5　制約のリバースエンジニアリング」で述べたような過程を経て決定．
- はじめは，すべてのイテレータに対して動作するよう，次にように書きたかった．
```rust
impl<T: Iterator> OffThreadExt for T
```

---
#### `Where` 節について2
- `spawn` を使って型 `T` のイテレータを新しいスレッドに移動しているので，`T:
Iterator + Send + 'static` と指定しなければならなかった．
- アイテムをチャネル経由で送信するので，`T::Item: Send + 'static` としなければならなかった．
- Rustは，ほとんどすべてのイテレータに並列の力を与えるツールを自由に加えることができるが，そのためにはそれを安全に使えるようにするための制約を理解し，記述しなければならないということ．

---
### 19.2.7　パイプラインを超えて
- パイプラインは有用で，チャネルの用途として自明なので，パイプラインを例として用いた．
- しかし，チャネルはパイプライン以外にも使うことができる．
- チャネルを使って，任意の非同期サービスを同じプロセス上の他のスレッド群に提供することができる．

---
#### ログを書き出す専用のスレッドの例
- クライアントとなるスレッドは，ログを書き出すスレッドにチャネル経由でログメッセージを送る．
- チャネルの `Sender` はクローンできるので，1つのログスレッドに対してログメッセージを送る `Sender` を，複数のクライアントスレッドに与えることができる．

---
#### 専用のスレッドでの実行の利点
- ログスレッドはログ・ファイルを必要に応じてローテートするが，この際に他のスレッドと面倒な協調作業をする必要はない（他のスレッドはブロックしないため）．
- メッセージは少しの間だけ無害にチャネルに蓄積され，ログスレッドが仕事に戻るのを待つ．

---
#### 返答を必要とするリクエストを他のスレッドに送る場合
- 最初のリクエストに `Sender` を含んだ構造体かタプルを用いて，ある種の返信用封筒として機能させる．
- このリクエストを受信したスレッドは，この `Sender` を用いて返答を返す．この相互通信は同期している必要はない．
- 最初のスレッドはブロックして返事を待つこともできるし，`.try_recv()` メソッドを用いて返事が来ているかどうか調べることもできる．

---
#### 19.3に向けて
- ここまでで，高並列な計算に用いるフォーク・ジョインと，緩やかに構成要素を結合するチャネルを示した．これらを用いれば幅広いアプリケーションを十分に記述できる，
- しかし，まだ終わりではない．

![center height:300](img/ToBeContinued....jpg)
