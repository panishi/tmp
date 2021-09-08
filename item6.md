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
Item 6: コンパイラが自動生成することを望まない関数は，使用を禁止しよう

2021/9/9

---
### はじめに
- 不動産が利用するソフトウェアにおいて，「売り家」を表すクラスを考える．
```cpp
class HomeForSale {…};
```

---
### 売り家はコピー出来ない（はず）
- どの不動産業者も「同じ物件は決してない」と言うことを考えると，`HomeForSale` はコピー出来ないはず．
- コピーしようとした場合にはコンパイルエラーとしたい．
```cpp
HomeForSale h1;
HomeForSale h2;

HomeForSale h3(h1);  // h1をコピーコンストラクトしようとしている

h1 = h2;             // h2によってコピー代入しようとしている
```

---
### コピーをコンパイル禁止にするのは難しい
- 普通，クラスに特定の関数を付けたくない場合，その関数を宣言しなければよい．
- しかしコピーコンストラクタとコピー代入演算子は，宣言せずに使用しようとするとコンパイラが自動生成してしまう（Item 5を参照）．

---
### `private` に宣言する
- コンパイラが生成する関数は，全て `public` なもの．
- コピーコンストラクタとコピー代入演算子を `private` に宣言すれば，明示的に宣言したためコンパイラが生成することはなく，クラス外部からの利用も出来ない．

---
### 宣言のみで定義を書かない
- クラス内部やフレンド関数では `private` 関数を呼び出せるため，上記ではまだ安全ではない．
- `private` に宣言するだけでなく，定義を書かなければ，うっかり呼び出された場合にリンカエラーとなる．
- これはよく知られたテクニックで，かつては標準ライブラリの `ios_base` や `basic_ios` においても使われていた．
https://cpprefjp.github.io/reference/ios/ios_base/op_constructor.html
https://cpprefjp.github.io/reference/ios/basic_ios/op_constructor.html


---
### `HomeForSale` への適用
```cpp
class HomeForSale {
public:
    …
private:
    …
    // 宣言のみ
    HomeForSale(const HomeForSale&);
    HomeForSale& operator=(const HomeForSale&);
};
```
- 実装を省略していて使われないため，仮引数名は省略している（書いてはいけない訳ではない）．

---
### リンカエラーではなくコンパイルエラーとする方法1
- コピーコンストラクタとコピー代入演算子を `private` 宣言した `Uncopyable` を作り，`HomeForSale` はそれを継承するようにする．
```cpp
class Uncopyable {
protected:
    // 生成と破棄は許可する
    Uncopyable() {}
    ~Uncopyable() {}

private:
    // しかし，コピー（代入を含む）は禁止する
    Uncopyable(const Uncopyable&);
    Uncopyable& operator=(const Uncopyable&);
};
```

---
### リンカエラーではなくコンパイルエラーとする方法2
```cpp
// このクラスはコピーコンストラクタやコピー代入演算子を宣言出来ない
class HomeForSale : private Uncopyable {
    …
};
```

---
### 実はやや高度な知識が使われている
- `private` 継承を用いている（Item 32，39を参照）．
- `Uncopyable` のデストラクタは仮想でなくてよい（Item 7を参照）．
ポリモルフィックな基底クラスとして振舞っていないため（Item 39も参照）．
- Boostライブラリ（Item 55を参照）には，`noncopyable` というクラスがいる（英語的には違和感があるが…）．

---
### おわりに
- 実はこのItemで学んだ内容はC++11以降では使わない．
- `delete` を使えば，リンカエラーではなくコンパイルエラーと出来る（元々やりたかったこと）．
```cpp
class HomeForSale {
public:
    …
    HomeForSale(const HomeForSale&) = delete;
    HomeForSale& operator=(const HomeForSale&) = delete;
};
```
- Effective Modern C++のItem 11を参照．

---
### Things to Remember
- コンパイラが自動でコードを生成することを禁止するために，「対応するメンバ関数を `private` に宣言し，定義を書かない」という方法がある．また，`Uncopyable` のようなクラスを使う方法もある．
