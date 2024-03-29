---
marp: true
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

# Effective C++
Item 16: 対応する `new` と `delete` は同じ形のものを使おう

2021/x/x

---
### はじめに
- 以下のコードは誤りで，動作は未定義．
```cpp
std::string* stringArray = new std::string[100];
…
delete stringArray;
```
- 少なくとも，配列 `stringArray` が持つ100個のオブジェクトのうち99個は正しく破棄されず，それらのデストラクタはおそらく呼び出されない．

---
### `new` と `delete` で起こること
- `new` を使うと次の2つのことが起きる．
    - `new` 演算子（Item 49, 51を参照）によりメモリが確保される．
    - 確保されたメモリに対して，（1つか，それ以上の）コンストラクタが呼び出される．
- `delete` を使うと次の2つのことが起きる．
    - 確保されたメモリに対して，（1つか，それ以上の）デストラクタが呼び出される．
    - `delete` 演算子（Item 51を参照）によりメモリが解放される．
- `delete` にとっては，破棄するメモリ内のオブジェクト数が重要．

---
### メモリ上の構成1
- `delete` を適用するポインタが指しているのは，単独のオブジェクトか，オブジェクトの配列のいずれか．
- 単独のオブジェクトとオブジェクトの配列では，メモリ上の構成が異なる．
- よくある実装では，「配列」はすぐ近くに「要素数」の情報も置くようになっているため，`delete` はこの情報を使って，正しい数のデストラクタを呼び出すことが出来る．

---
### メモリ上の構成2
- 単独のオブジェクトの場合
```
┌─────
│ object │
└─────
```
- 配列
```
┌──────────────────────────────
│   n   │ object │ object │ object │   …   │ 
└──────────────────────────────
```

---
### ポインタに `delete` を適用する場合1
- 対象のポインタが配列を指していて，その「要素数の情報（上の図のn）」がどこに置いてあるという事実は，プログラマが書かなければ `delete` には分からない．
- `delete` に `[]` を使うことで，`delete` は自分が破棄するものが配列であると分かる．
- `[]` を使わなければ，`delete` は自分が削除するものは単独のオブジェクトであると判断する．

---
### ポインタに `delete` を適用する場合2
```cpp
std::string* stringPtr1 = new std::string;
std::string* stringPtr2 = new std::string[100];
…
delete stringPtr1;    // 単独のオブジェクトを破棄
delete[] stringPtr2;  // オブジェクトの配列を破棄
```

---
### `stringPtr1` に `delete[]` を適用した場合
- 結果は未定義．
- メモリ上の構成が上の図のようになるコンパイラなら，`delete` は単独オブジェクトの前にあるデータを読み出し，それを配列の要素数と解釈して，不必要な数のデストラクタを呼び出すことになる．
- 関係の無い（おそらく型も違う）オブジェクトまで破棄しようとする．

---
### `stringPtr2` に `delete` を適用した場合
- これも結果は未定義．
- 呼び出されるデストラクタは1つだけ．
- クラスのオブジェクトではなく，`int` のような組み込み型の配列であっても，結果は未定義となる．

---
### 以上の考察から導かれるガイドライン
- `new` でオブジェクトを生成するときに `[]` を使ったなら，対応する `delete` でも `[]` を使う．
- `new` でオブジェクトを生成するときに `[]` を使わなかったら，対応する `delete` でも `[]` を使わないようにする．
- このルールは特に，「ポインタをデータメンバに持ち，そのポインタが指すオブジェクトのメモリを動的に確保するクラスで，コンストラクタが複数ある」場合に，留意する必要がある．
- ポインタを初期化する全てのコンストラクトで同じ形式の `new` を使わなければならない．

---
### `typeef` における注意点
- `typedef` で定義したオブジェクトを `new` で生成したら，`delete` に `[]` を付けるべきかどうか，明示しておかなければならない．
```cpp
typedef std::string AddressLines[4];
std::string* pal = new AddressLines;  // new std::string[4] と同じ意味

delete pal;                           // 未定義！
delete[] pal;                         // 問題なし
```
- このような混乱を避けるため，配列を `typedef` することは控えた方がよいだろう．

---
### おわりに
- C++の標準ライブラリ（Item 54を参照）には，`string` や `vector` などがあり，これらを使えば配列を動的に生成する必要がほとんどなくなる．
- 上の例では，`AddressLines` は `vector<string>` でよい．
- （話者注）C++11ならば `array<string, 4>` の方がよいだろう．

---
### Things to Remember
- オブジェクトを `new` で生成するときに `[]` を使ったなら，対応する `delete` でも `[]` を使おう．
逆に，オブジェクトを `new` で生成するときに `[]` を使っていないなら，対応する `delete` でも `[]` を使わないように．
