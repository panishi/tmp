# Item 23：`std::move` と `std::forward` を理解する
## はじめに
`std::move` と `std::forward` については，こう動作すると解説を進めるよりも，こう動作しないと解説を進めた方が良いだろう．
- `std::move` は何もムーブしない．
- `std::forward` は何も転送しない．

実行コードは1バイトたりとも生成されない．

`std::move` と `std::forward` はキャストを実行する関数（テンプレート）に過ぎない．
- `std::move` は実引数を無条件に右辺値へキャストする．
- `std::forward` は特定の条件が満たされた場合にのみ同様にキャストする．

## `std::move` の実装例
C++11での `std::move` のサンプル実装を挙げる．
```cpp
template <typename T>  // std 名前空間
typename remove_reference<T>::type&&
move(T&& param)
{
    using ReturnType = typename remove_reference<T>::type&&;  // エイリアス宣言，項目9を参照
    return static_cast<ReturnType>(param);
}
```

`std::move` はオブジェクトの参照（厳密に言えばユニヴァーサル参照．Item24を参照）を受け取り，同じオブジェクトを表す参照を返す．

戻り型にある「`&&`」は，この関数が右辺値参照を返すことを表す．
Item28でも述べるように，型 `T` が左辺値参照の場合，`T&&` は左辺値参照になる．
この不一致を解消するために，`T` に型特性 `std::remove_reference` を使用している（Item9を参照）．
これにより，参照ではない型に「`&&`」を適用することを保証でき，`std::move` は真に右辺値参照を返すことも保証できる．
`std::move` はこのように実引数を右辺値へキャストしているだけだ．

なお，C++14ではより簡潔に実装できる．
これは，関数の戻り型を推論する機能（Item3を参照）と，エイリアステンプレート `std::remove_reference_t`（Item9を参照）のおかげだ．
```cpp
template <typename T>  // C++14，変わらずstd名前空間
decltype(auto) move(T&& param)
{
    using ReturnType = remove_reference_t<T>&&;
    return static_cast<ReturnType>(param);
}
```

## なぜ `std::move` という名前なのか
`std::move` の処理内容は実引数を右辺値へキャストするのみなので，`rvalue_cast` のようなより実態に即した名前が提案されたこともあった．

右辺値はムーブ対象の候補となるため，オブジェクトに `std::move` を使用すれば，コンパイラに対してこのオブジェクトがムーブ元になれると通知できる．
ムーブ元になれるオブジェクトの指定が容易になりるため，`std::move` と名付けられている．

## `const` とムーブ
通常は，右辺値はムーブ対象の候補となるだけにすぎない．
注釈 (annotation) を表現するクラスを例に考える．
このクラスのコンストラクタは注釈文の `std::string` 仮引数をとり，メンバ変数へコピーする．
```cpp
class Annotation {
public:
explicit Annotation(std::string text);  // Item41に従い，コピーする仮引数は値渡しする
    …
};
```

`Annotation` のコンストラクタは `text` の値を変更することはない．
ここで，「可能な場面では常に `const` を使用せよ」という昔ながらの伝統に従い，`text` を `const` と宣言してみる．
```cpp
class Annotation {
public:
explicit Annotation(const std::string text);
    …
};
```

`text` をメンバ変数へコピーする際のコストを削減するため，Item41に従い `std::move` により右辺値の `text` を得る．
```cpp
class Annotation {
public:
explicit Annotation(const std::string text);
    : text_(std::move(text))  // textをtext_へ「ムーブ」，見た目通りの処理内容ではない！
    { … }
    …
private:
    std::string text_;
};
```

上例のコードはコンパイル，リンクも実行もでき，`text` の内容を `text_` に代入もして思い通りなようにも見える．
思い描いた処理内容と異なるのは，**`text` が `text_` へムーブされるのではなくコピーされる** 点だ．
`text` は `std::move` により右辺値へキャストされるが，`const` 宣言されているため，キャスト前の `text` は左辺値の `const std::string` であり，キャスト後に右辺値の `const std::string` になる．
キャストしても `const` 性は変化しない．

コンパイラが `std::string` のコピー／ムーブコンストラクタのどちらを呼び出すのか考える．
```cpp
class string {  // std::stringの実際はstd::basic_string<char>のtypedef
public:
    …
    string(const string& rhs);  // コピーコンストラクタ
    string(string&& rhs);       // ムーブコンストラクタ
    …
};
```

`Annotation` コンストラクタのメンバ初期化リストでは，`std::move(text)` の結果は `const std::string` 型の右辺値としている．
`std::string` のムーブコンストラクタは `const` でない `std::string` の右辺値参照をとるため，`const std::string` 型の右辺値をムーブコンストラクタへ渡せない．
一方，`const` 左辺値参照の `const` 右辺値へのバインドは認められているため，コピーコンストラクタへは `const` 右辺値を渡せる．
そのため，上例のメンバ初期化では `text` を右辺値へキャストしているにも関わらず，`std::string` のコピーコンストラクタが実行される！
オブジェクトの値をムーブすると通常はオブジェクトを変更するため，実引数を変更する関数（ムーブコンストラクタなど）へ `const` オブジェクトを渡すことはできない．

上例からは2つのことを学べる．
- オブジェクトをムーブ元として使用可能にするには `const` と宣言してはいけない．
`const` オブジェクトに対するムーブ要求はコピー演算に変換され，警告などは何も出力されない．
- `std::move` は実際には何もムーブしないだけでなく，オブジェクトがムーブ可能であることすら保証しない．
`std::move` の効果として保証されるのは，結果が右辺値であることだけだ，

## `std::forward` について
`std::forward` についても `std::move` に似たことが言えるが，`std::move` が実引数を無条件に右辺値へキャストするのに対し，`std::forward` はある特定の条件が満たされた場合にのみキャストする．
キャストするのはどんな場合かを理解するため，`std::forward` の典型的な使用例を述べる．
最も多く使用される場面は，ユニヴァーサル参照仮引数をとる関数テンプレートであり，これは仮引数を他の関数へ渡す．
```cpp
void process(const Widget& lvalArg);  // 左辺値を処理
void process(Widget&& rvalArg);       // 右辺値を処理

template <typename T>  // processへ仮引数を渡すテンプレート
void logAndProcess(T&& param)
{
    auto now = std::chrono::system_clock::now();  // 現在時刻を得る
    makeLogEntry("Calling 'process'", now);
    process(std::forward<T>(param));
}
```

左辺値を渡した場合，右辺値を渡した場合の `logAndProcess` の呼び出しを考える．
```cpp
Widget w;

logAndProcess(w);             // 左辺値を渡す
logAndProcess(std::move(w));  // 右辺値を渡す
```

`logAndProcess` は受け取った仮引数 `param` を関数 `process` へ渡す．
`process` は左辺値をとるものと右辺値をとるものにオーバロードされている．
左辺値を渡して `logAndProcess` を呼び出した場合は左辺値をとる `process` のオーバロードが実行され，右辺値を渡して `logAndProcess` を呼び出せば右辺値をとる `process` のオーバロードが実行されると期待する．

しかし，すべての関数仮引数は左辺値だ．
そのため，`logAndProcess` 内の `process` 呼び出しはすべて，左辺値をとる `process` のオーバロードに解決されてしまう．
このオーバロード解決を防ぐには，`logAndProcess` へ渡した実引数が右辺値の場合にのみ，`param` を右辺値へキャストする仕組みが必要となる．
この動作こそ `std::forward` であり，`std::forward` が条件付きキャストである所以だ．

「実引数が右辺値により初期化された」という情報は，`logAndProcess` のテンプレート仮引数 `T` に組み込まれている．
`std::forward` へはこの仮引数が渡され，組み込まれた情報が使用されている（Item28を参照）．

## `std::move` と `std::forward` の違い
これらの違いは，`std::move` は常にキャストし，`std::forward` はある条件が満たされた場合にのみキャストするという点のみだ．
`std::move` は使用しないと決め，すべての場面で `std::forward` のみを使用すれば良いのかという意見もあるだろう．
純粋に技術的な観点に立てば，答えはyesだ．

`std::move` の魅力は利便性，誤用の恐れを削減できる点．および分かりやすさにある．
ムーブコンストラクタが何回呼び出されたかを数えるクラスを考えてみよう．
ムーブコンストラクト時にインクリメントする `static` なカウンタがあればよい．
`static` ではないメンバ変数は `std::string` しかないとすると，以下のように記述できる．
```cpp
class Widget {
public:
    Widget(Widget&& rhs)
        : s_(std::move(rhs.s_))
    { ++moveCtorCalls_; }
    …
private:
    static std::size_t moveCtorCalls_;
    std::string s_;
};
```

`std::forward` を用いて同じ動作を実装すると，次のようになる．
```cpp
class Widget {
public:
    Widget(Widget&& rhs)  // 一般的でなく望ましくもない実装
        : s_(std::forward<std::string>(rhs.s_))
    { ++moveCtorCalls_; }
    …
};
```

注目すべき点は2つある．
- `std::move` には実引数 (`rhs.s_`) を1つ与えるだけで済むが，`std::forward` では実引数に加えテンプレートの型実引数 (`std::string`) も与えなければならない．
- 渡す実引数が右辺値であるという情報を組み込む際の慣例として，`std::forward` へ渡すのは非参照型とする（Item28を参照）．

この2点を合わせると，`std::move` の方が `std::forward` よりも入力タイプ量が少なく済み，実引数が右辺値であることを示す情報を内包した型実引数を与える際の問題を未然に防げることが分かる．
また，誤った型を渡してしまうミスが発生する可能性を削減する効果もある．
例えば `std::string&` を渡すと，メンバ変数 `s_` はムーブコンストラクトされずコピーコンストラクトされてしまう．

さらに重要なことは，`std::move` は右辺値へ無条件にキャストする意図を表現するのに対し，`std::forward` では右辺値の参照の場合にのみキャストするという意図の表現になる点だ．
`std::move` の場合はムーブ動作となるのが通例だが，`std::forward` では元々の右辺値／左辺値の性質を維持したまま他の関数へオブジェクトを渡す転送 (forward) する．
この動作の差異は大きいため，区別するためにも別関数となっている．

## Things to remember
- `std::move` は右辺値への無条件キャストを実行するのみであり，自身では何もムーブしない．
- `std::forward` は実引数が右辺値にバインドされている場合に限り，その実引数を右辺値へキャストする．
- `std::move` も `std::forward` もプログラム実行時には何も実行しない．
