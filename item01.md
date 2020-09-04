# Item1：テンプレートの型推論を理解する
## はじめに
良いニュースと悪いニュースをお知らせしよう．

### 良いニュース
テンプレートの型推論が，現代のC++の最も面白い機能である `auto` の基礎になっている．
C++98のテンプレートの型推論に満足していれば，C++11の `auto` の型推論についても満足いくだろう．

### 悪いニュース
テンプレートの型推論規則を `auto` に適用すると，テンプレートに比べ分かりにくい結果になることがある．
このため，`auto` の基礎となるテンプレートの型推論を隅々まで把握しておくことが重要だ．
本項目では押さえておくべき重要な点を解説する．

## 関数テンプレートの疑似コード
以下では，下記にある関数テンプレートの疑似コードを考える．
```cpp
template <typename T>
void f(ParamType param);

f(expr);  // exprからTとParamTypeを推論
```

上記コードのコンパイル時に，コンパイラは `expr` から2つの型（`T` と `ParamType`）を推論する．
この2つは一致しないことが多く，`ParamType` は多くの場合修飾されている（`const` 修飾子や参照修飾子が付加されている）．
以下の例を考えよう．
```cpp
template <typename T>
void f(const T& param)

int x = 0;
f(x);
```

上例では `T` は `int` と推論されるが ，`ParamType` は `const int&` と推論される．

`T` に推論したのと同じ型が関数の実引数の型にも推論される（`expr` の型が `T` となる）と期待するのはごく自然なことだろう．
上例でも `x` は `int` であり，`T` は `int` と推論されるが，常にこのような結果になる訳ではない．
`T` に推論する型は `expr` の型だけから決定される訳ではなく，`ParamType` からも影響を受ける．
次に挙げる3通りのパターンがある．
- ケース1：`ParamType` が参照もしくはポインタだが，ユニヴァーサル参照ではない（ユニヴァーサル参照についてはItem24で述べるが，ここでは左辺値参照とも右辺値参照とも違うものが存在するとだけ認識しておけばよい）．
- ケース2：`ParamType` がユニヴァーサル参照である．
- ケース3：`ParamType` がポインタでも参照でもない．

以下では，各ケースについて解説する．

## ケース1 : `ParamType` が参照もしくはポインタだが，ユニヴァーサル参照ではない
この場合は最も単純だ．
1. `expr` が参照型ならば，参照性（参照動作部分）を無視する．
1. `expr` の型を `ParamType` とパターンマッチングし， `T` を決定する．

以下の例を考える．
```cpp
template <typename T>
void f(T& param);   // paramは参照

int x = 27;         // xはint
const int cx = x;   // cxはconst int
const int& rx = x;  // rxはconst intとしてのxの参照
```

`param` と `T` に推論される型は次のようになる．
```cpp
f(x);   // Tはint，paramの型はint&
f(cx);  // Tはconst int，paramの型はconst int&
f(rx);  // Tはconst int，paramの型はconst int&
```

2番目と3番目の呼び出しに注目しよう．
`cx` と `rx` には `const` を指定していて，`T` は `const int` と推論されている．
その結果，仮引数の型は `const int&` となる．
これにより，参照仮引数に `const` オブジェクトを渡せば，オブジェクトが変更されない（`const` 参照仮引数となる）．
`T&` を仮引数にとるテンプレートへ `const` オブジェクトを渡しても安全であり，またオブジェクトの `const` 性が `T` に推論される型の一部となる．

3番目の呼び出しでは，`rx` が参照型であるにも関わらず，`T` は参照型と推論されない点に注目しよう．
これは，型の推論では `rx` が備える参照性が無視されるためだ．

`param` の型を `T&` から `const T&` へ変更しても，それほど大きな違いはない．
`cx` と `rx` の `const` 性は維持されるが，`param` は `const` 参照であると想定しているため，`T` の一部として `const` を推論する必要がなくなる．
```cpp
template <typename T>
void f(const T& param);  // paramはconst参照

// 先の例と変わらず
int x = 27;
const int cx = x;
const int& rx = x;

f(x);   // Tはint，paramの型はconst int&
f(cx);  // Tはint，paramの型はconst int&
f(rx);  // Tはint，paramの型はconst int&
```

先の例と同様に，型推論では `rx` の参照性は無視される．

`param` が参照ではなくポインタ（または `const` を指すポインタ）だったとしても，同様に推論される．
```cpp
template <typename T>
void f(T* param);  // paramはpointer

int x = 27;          // 先の例と変わらず
const int* px = &x;  // pxはconst intとしてのxを指す

f(&x);  // Tはint，paramの型はint*
f(px);  // Tはconst int，paramの型はconst int*
```

## ケース2：`ParamType` がユニヴァーサル参照である
ユニヴァーサル参照を仮引数にとるテンプレートの場合はずっと分かりにくくなる．
仮引数は右辺値参照のように (`T&&`) 宣言されるが，左辺値の実引数が渡された場合の動作が変化する．
詳細はItem24で解説し，ここでは概要を簡単に述べる．
- `expr` が左辺値ならば，`T` も `ParamType` も左辺値参照と推論される．
これは2つの意味で特殊である．
    - テンプレートの型推論で，`T` を参照として推論するのはこの場合だけ．
    - `ParamType` の宣言には右辺値参照という形態をとりながら，推論される型は左辺値参照となる．
- `expr` が右辺値の場合は，「通常の」規則が適用される（ケース1）．

```cpp
template <typename T>
void f(T&& param);  // paramはユニヴァーサル参照

// 先の例と変わらず
int x = 27;
const int cx = x;
const int& rx = x;

f(x);   // xは左辺値，よってTはint&，paramの型もint&
f(cx);  // cxは左辺値，よってTはconst int&，paramの型もconst int&
f(rx);  // rxは左辺値，よってTはconst int&，paramの型もconst int&
f(27);  // 27は右辺値，よってTはint，ゆえにparamの型はint&&
```

なぜこのように動作するかについては，Item24で述べる．
重要なのは，「ユニヴァーサル参照の仮引数に対する型推論規則は，左辺値参照や右辺値参照の仮引数の場合とは異なる」という点だ．
特に，型推論が左辺値実引数と右辺値実引数を区別する点は重要であり，ユニヴァーサル参照に限った特殊な規則だ．

## ケース3 : `ParamType` がポインタでも参照でもない
この場合，値渡しとなる．
```cpp
template <typename T>
void f(T param);  // paramは値渡しされる
```

`param` は渡したもののコピー，すなわち全く別のオブジェクトとなる．
この点は，`expr` から `T` を推論する動作に大きく影響する．
1. これまでと同様に，`expr` が参照型ならば，参照性（参照動作部分）を無視する．
1. 参照性を無視した `expr` が `const` であればこれも無視する．
`volatile` であれば，同様に無視する（`volatile` についてはItem40参照）．

実際には次のようになる．
```cpp
// 先の例と同様
int x = 27;
const int cx = x;
const int& rx = x;

f(x);   // Tとparamの型はいずれもint
f(cx);  // Tとparamの型はいずれもやはりint
f(rx);  // Tとparamの型はいずれも変わらずint
```

`cx` および `rx` の値が `const` の場合でも，`param` は `const` とならない点に注意しよう．
`param` は `cx` や `rx` のコピーなため，`cx` と `rx` が変更不可である点は `param` には影響しない．
このため `expr` の `const` 性／`volatile` 性は，`param` の型を推論する際に無視される．

### ポインタ実引数
`const` （および `volatile`）が値渡しの場合にのみ無視される点は重要なため，よく覚えておこう．
これまで見てきたように，仮引数が `const` を指すポインタ／参照の場合は，`expr` の `const` 性は型を推論しても失われない．
では，`expr` が `const` オブジェクトを指す `const` なポインタであり，`expr` を `param` に値渡しした場合はどうなるだろうか．
```cpp
template <typename T>
void f(T param);  // paramは変わらず値渡しされる

const char* const ptr = "Fun with pointers";  // ptrはconstオブジェクトを指すconstなポインタ

f(ptr);  // const char* const型の実引数を渡す
```

アスタリスクの右にある `const` は `ptr` が `const` であることを意味する．
`ptr` を `f` に渡すと，`ptr` を構成する全ビットが `param` にコピーされるので，仮引数を値渡しする際の型推論規則により，`ptr` の `const` 性は無視され，`param` に推論される型は `const char*` となる．
型を推論しても，`ptr` が指すオブジェクトの `const` 性は維持されるが，`ptr` をコピーし新たなポインタ `param` を作成する時点で，`ptr` 自身の `const` 性は失われる．

### 配列実引数
あと少しだけ覚えておくことがある．
配列型とポインタ型は交換可能と言われることもあるが，両者は異なる型であるという点だ．
この目眩ましのような状態の元凶は，配列は多くの場面で，その先頭要素を指すポインタに **成り下がる** (decay) という動作だ．
```cpp
const char name[] = "J. P. Briggs";  // nameの型はconst char[13]

const char* ptrToName = name;  // 配列がポインタに成り下がる
```

`const char*` のポインタ `ptrToName` は `const char[13]` である `name` により初期化される．
`const char*` と `const char[13]` は同じ型ではないが，配列からポインタへ変換する規則により，コンパイル可能となる．

では、仮引数を値渡しするテンプレートに配列を渡すとどうだろうか？
```cpp
template<typename T>
void f(T param);  // 仮引数を値渡しするテンプレート

f(name);  // Tとparamにはどんな型が推論されるか？
```

まず，関数の仮引数として配列なぞはあり得ないという事実から確認しよう．
```cpp
void myFunc(int param[]);
```

配列として宣言してもポインタの宣言として扱われる．
つまり上例の `myFunc` は次のようにも宣言可能だ．
```cpp
void myFunc(int* param);  // 上例と同じ関数
```

仮引数の配列とポインタの等価性は，C++の土台であるC言語を根とし，成長した枝葉のようなもので，ここから配列とポインタは同じものであるという幻想が醸し出されている．
以下では，その幻想をぶち壊そう．

配列仮引数の宣言は，ポインタ仮引数として扱われるため，テンプレート関数へ値渡しされた配列の型はポインタ型と推論される．
つまり，テンプレート `f` を呼び出すと，その型仮引数 `T` は `const char*` と推論される．
```cpp
f(name);  // nameは配列だが，Tはconst char*と推論される
```

ここで変化球の登場だ．
関数は仮引数を真の配列とは宣言できないが，配列の参照としては宣言できる！
次のようにテンプレート `f` の実引数を参照に変更してみよう．
```cpp
template<typename T>
void f(T& param);  // 参照渡しの仮引数を持つテンプレート
```

そして配列を渡してみる．
```cpp
f(name);  // fへ配列を渡す
```

すると，`T` に推論される型は配列の型になる！
この型は配列の要素数も含んでおり，上例では `const char [13]` だ．
また，`f` の仮引数の型は，`const char (&)[13]` となる．
この点を押さえておくと，こんなことまで意識するごく一部の人々と議論する際に役立つこともあるだろう（？）．

面白いことに，配列の参照を宣言できるようになると，配列の要素数を推論するテンプレートを記述できる．
```cpp
// 配列の要素数をコンパイル時定数として返す
//（要素数のみを考慮するため，仮引数の配列に名前はない）
// constexprとnoexceptについては下記を参照
template <typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept
{
    return N;
}
```

Item15でも述べるが，`constexpr` と宣言することで，その戻り値をコンパイル時に使用できる．
これにより，例えば配列の宣言時に要素数を明示しなくとも，波括弧を用いた初期化子から要素数を算出できるようになる．
```cpp
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };  // keyValsの要素数は7
int mappedVals[arraySize(keyVals)];          // mappedValsも同じ
```

もちろん，現代のC++erならば，組み込み配列よりも `std::array` の方が当然お好みだろう．
```cpp
std::array<int, arraySize(keyVals)> mappedVals;  // mappedValsの要素数は7
```

`arraySize` は `noexcept` 宣言しているため，コンパイラにはより良いコードを生成する機会が生まれる．
Item14を参照のこと．

（話者注）C++17ならば標準ライブラリ `<iterator>` に `std::size` があるため，`araySize` を自作せずとも済む．
https://cpprefjp.github.io/reference/iterator/size.html


### 関数実引数
C++でポインタに成り下がるのは配列だけではなく，関数型も関数ポインタに成り下がる．
先に述べた配列の型推論に関することはすべて，関数の型推論および関数ポインタへの成り下がりにも適用される．
```cpp
void someFunc(int, double);  // someFuncは関数，型はvoid(int, double)

template <typename T>
void f1(T param);   // f1のparamは値渡し

template <typename T>
void f2(T& param);  // f2のparamは参照渡し

f1(someFunc);  // paramは関数を指すポインタと推論，型はvoid (*)(int, double)
f2(someFunc);  // paramは関数の参照と推論，型はvoid (&)(int, double)
```

現実に何らかの差異が生まれることはまずありえないが，配列がポインタに成り下がる点を覚えるならば，関数がポインタに成り下がることも知っておくとよいだろう．

## おわりに
ここまでで `auto` に関するテンプレートの型推論規則を学んだ．
初めに述べたように型推論規則はほとんどの場合は直観的に理解できる．
特別な注意が必要なのは，以下の2つ．
- ユニヴァーサル参照の型を推論する際の左辺値
- 配列と関数がポインタに成り下がる規則

コンパイラの胸ぐらをつかみ，「お前が推論する型を吐け！」と怒鳴りつけたくなることもある…かもしれない．
そんな時はItem4を読んで落ち着こう．

## Things to remember
- テンプレートの型推論時には，参照実引数は参照とは扱われない．
すなわち参照性は無視される．
- ユニヴァーサル参照仮引数の型を推論する際には，左辺値実引数を特別扱いする．
- 値渡しの仮引数の型を推論する際には，`const` および／または `volatile` 実引数は非 `const`，非 `volatile` と扱われる．
- 参照を初期化するものでなければ，配列または関数実引数はテンプレートの型推論時にポインタに成り下がる．
