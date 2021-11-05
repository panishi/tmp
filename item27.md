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
Item 27: キャストは最小限にしよう

2021/11/5

---
### はじめに
- C++の規則は型によるエラーをなくすように作られている．
- 理論上は，プログラムがコンパイルできたなら，どのオブジェクトに対しても危険な操作や間違った操作はされていないということになる．
- これはとても価値のある保証だ．

---
### キャストが「型システム」をだめにする
- キャストはありとあらゆるトラブルの原因になる．
- C，Java，C#では，キャストはより必要性があり，より危険性が低い．
- C++は，CでもJavaでもC#でもないので，細心の注意を払ってキャストを扱わなければならない．

---
### 古いスタイルのキャスト
- Cスタイルのキャストは以下の通り．
```cpp
(T)式  // 式を型Tにキャスト
```
- 関数スタイルのキャストは以下の通り．
```cpp
T(式)  // 式を型Tにキャスト
```
- 括弧をどこに置くかの違いしかない．
- 本書では，これらを古いスタイルのキャストと呼ぶ．

---
### C++スタイルの4つのキャスト
- C++には，4つの新しいキャストの形式がある．
    - `const_cast<T>(式)`
    - `dynamic_cast<T>(式)`
    - `reinterpret_cast<T>(式)`
    - `static_cast<T>(式)`
- それぞれの目的は以下の通り．

---
### `const_cast`
- オブジェクトの `const` 性を取り除くために使われる．
- C++スタイルのキャストで `const` を取り除けるのは，このキャストのみ．

---
### `dynamic_cast`
- 「安全なダウンキャスト」に使われる．
- あるオブジェクトが，「特定の型の派生クラスのオブジェクト」であるかどうかを調べるもの．
- C++スタイルのキャストで，古いスタイルのキャストが実現できないものは，これだけ．
- このキャストだけは，実行時に大きなコストがかかる可能性がある（後述する）．

---
### `reinterpret_cast`
- 実装依存の（移植性のない）低レベルなキャストを行う．
- ポインタを `int` にキャストする等．
- このキャストは，低レベルなコード以外ではあまり使うべきではない．
- 本書でも1箇所でしか使っていない（Item 50を参照）．

---
### `static_cast`
- 暗黙の（非明示的な）型変換を強制するときに使われる．
- `void*` を特定のポインタ型にキャストするときや，基底クラスのポインタを派生クラスのポインタにキャストするときにも用いられる．
- `const` オブジェクトを非 `const` オブジェクトにキャストすることはできず，それが可能なのは `const_cast` のみ．

---
### 新しいスタイルのキャストのメリット
- 人間（目grep）にとっても，grepのような検索ツールにとっても，コード中で見つけやすく，型システムをコードのどこで壊しているか調べるのが簡単になる．
- それぞれ目的が限定されているので，コンパイラが間違いを見つけやすくなっている．

---
### 古いスタイルのキャストを使う場面
- 著者が古いスタイルのキャストを使うのは，関数に渡すオブジェクトを `explicit` なコンストラクトで生成・初期化するときのみ．
```cpp
class Widget {
public:
    explicit Widget(int size);
    …
};

void doSomeWork(const Widget& w);
doSomeWork(Widget(15));
doSomeWork(static_cast<Widget>(15));
```

---
### 「実行時の余分なコード」の生成
- キャストは，コンパイラに「型の変更」を告げるだけのものではない．
- 型の変換は，「実行時の余分なコード」を生成することになる．
```cpp
int x, y;
…
double d = static_cast<double>(x) / y;
```
- 大抵のコンパイラでは `int` と `double` の扱いが大きく異なるため，上記のキャストはほぼ間違いなく，余分なコードを生成することになる．

---
### アップキャストの例
```cpp
class Base { … };
class Derived : public Base { … };
Derived d;
Base* pb = &d;  // 暗黙的（非明示的）にDerived*をBase*に変換
```
- 最後の行において，型変換によってアドレスが変換前後で異なることがあり得る．
- その場合，`Derived*` の値から `Base*` の値への変換は実行時に行われる．

---
### オブジェクトのアドレスについて1
- C++では，ひとつのオブジェクトが複数のアドレスを持つこと **も** あり得る．
- これはCやJavaやC#では起こらない．
- C++においては，オブジェクトがメモリ上でどのように配置されているかを，勝手に仮定してはいけない．
- オブジェクトのアドレスを `char*` にキャストし，ポインタ演算を行えば，結果は恐らく未定義となってしまう．

---
### オブジェクトのアドレスについて2
- オブジェクトがメモリ上にどのように配置されるかは，コンパイラによって異なる．
- ある環境で「オブジェクトの配置をうまく使って有効だったキャスト」が，別の環境でも役に立つ保証はないということ．
- 世の中には，つらい代償を払ってこのことを学んだ悲しいプログラマがたくさんいることを覚えておこう．

---
### アプリケーションフレームワークの例1
- 一般的なウィンドウを表すクラス `Window` と，その派生クラス `SpecialWindow` があり，`Window` には `onResize` という仮想関数が定義されているとする．
- このアプリケーションフレームワークでは，`Window` の派生クラスの `onResize` では，はじめに `Window::onResize` を呼び出すことになっているとする．
- 以下の `SpecialWindow` の実装は一見正しそうだが，実はそうではない．

---
### アプリケーションフレームワークの例2
```cpp
class Window {
public:
    virtual void onResize() { … }
    …
};

class SpecialWindow : public Window {
public:
    virtual void onResize() {
        static_cast<Window>(*this).onResize();
        …
    }
};
```

---
### `Window` へのキャストの問題点
- 予想外な動作として，キャストは「`*this` の基底クラス部分」のコピーとして，新しい一時オブジェクトを作成してしまう．
- 上のコードは，現在のオブジェクトに対して `Window::onResize` を呼び出すことにならない．
- もし，`Window::onResize` が現在のオブジェクトを変更するものなら，現在のオブジェクトは必要な変更を受けず，コピーされた一時オブジェクトが変更を受けることになる．

---
### `Window` へのキャストを避ける方法
- コンパイラに `*this` を基底クラスだと思わせる必要などなく，単に基底クラスの `onResize` を現在のオブジェクトに対して呼び出せばよい．
```cpp
class SpecialWindow : public Window {
public:
    virtual void onResize() {
        Window::onResize();
        …
    }
};
```

---
### `dynamic_cast` の問題点
- 多くの実装で `dynamic_cast` は非常に遅くなる．
- よくある実装の1つでは，クラス名を文字列として比較する．
- より深い継承や多重継承があれば，このコストは更に増える．
- キャストを使うコードは疑ってかかるべきだし，効率が重要な箇所で `dynamic_cast` を使うコードは特によく考えるべきだ．
- `dynamic_cast` を避けるにはｍ，一般に以下の2つの方法がある．

---
### 型安全なコンテナを使う方法1
- はじめから「派生クラスのポインタ（特にスマートポインタ，Item 13を参照）のみを格納するコンテナ」を用意し，使う方法がある．
- インターフェースについて悩まずに，派生クラスの操作ができるようになる．
- 例えば，`SpecialWindow` のみが `blink` というメンバ関数を持っていたとする．

---
### 型安全なコンテナを使う方法2
```cpp
class Window { … };

class SpecialWindow : public Window {
public:
    void blink();
    …
};

typedef std::vector<std::shared_ptr<Window>> VPW;
VPW winPtrs;
…
```

---
### 型安全なコンテナを使う方法3
```cpp
// dynamic_castを使う好ましくないコード
for (VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); ++iter) {
    if (SpecialWindow* psw = dynamic_cast<SpecialWindow*>(iter->get())) {
        psw->blink();
    }
}
```

---
### 型安全なコンテナを使う方法4
```cpp
// dynamic_castを使わないコード
typedef std::vector<std::shared_ptr<SpecialWindow>> VPSW;
VPSW winPtrs;
…
for (VPSW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); ++iter) {
    (*iter)->blink();
}
```
- この方法では，`Window` の全ての派生クラスのポインタを同一のコンテナに格納することはできない．
- 1つひとつの派生クラスに対して，それぞれに対応する型安全なコンテナが必要になるだろう．

---
### 仮想関数を継承の仮想の一番上に置く方法1
- 派生クラスで定義する全ての関数を仮想関数として基底クラスに持たせ，その関数を使って派生クラスを操作するというもの．
- `blink` というメンバ関数が実際に意味があるのは `SpecialWindow` のみであっても，`Window` に「何もしない仮想関数」として `blink` を定義する．

---
### 仮想関数を継承の仮想の一番上に置く方法2
```cpp
class Window {
public:
    virtual void blink() {};  // よりよい方法については，Item 34を参照．
    …
};

class SpecialWindow : public Window {
public:
    virtual void blink() { … };
    …
};

typedef std::vector<std::shared_ptr<Window>> VPW;
VPW winPtrs;
…
```

---
### 仮想関数を継承の仮想の一番上に置く方法3
```cpp
for (VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); ++iter) {
    (*iter)->blink();
}
```
- 型安全なコンテナを使う方法も，仮想関数を継承の仮想の一番上に置く方法も，どちらの方法もいつでもできる訳ではないが，多くの場合 `dynamic_cast` の替わりになる．

---
### 絶対アカン設計1
- 以下のように `dynamic_cast` を連続して使うようなデザインは絶対に避けるべき．
```cpp
for (VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); ++iter) {
    if (SpecialWindow1* psw1 = dynamic_cast<SpecialWindow1*>(iter->get())) {
        …
    } else if (SpecialWindow2* psw2 = dynamic_cast<SpecialWindow2*>(iter->get())) {
        …
    } else if (SpecialWindow3* psw3 = dynamic_cast<SpecialWindow3*>(iter->get())) {
        …
    }
}
```

---
### 絶対アカン設計2
- このようなコードは大きくて効率が悪く，しかも脆弱だ．
- `Window` クラスの継承の階層が変更されると，更新が必要になるか，少なくともチェックが必要となってしまう（例えば，`Window` クラスの派生クラスが1つ追加されたら，おそらく `dynamic_cast` を連続で使っている条件分岐の項目を1つ増やす必要があるだろう）．

---
### おわりに
- 良いC++プログラムでは，非常にまれな場合にしか，キャストは使われない．
- 一方で，キャストを完全に取り去るというのも，現実的ではない．
- キャストは，疑わしいものとして，なるべく独立した部分に書くべき．
- 例えば，キャストを関数の内部に置いて，関数の呼び出し側からは汚い（キャストを用いた）コードが見えないようにするのがよいだろう．

---
### Things to Remember
- 可能な限りキャストを避けよう．特に，効率が重要な場合，`dynamic_cast` を避けよう．もし，設計上キャストが必要と感じたら，キャストを使わない代替案を考えてみよう．
- キャストが避けられない場合，それは関数の中に隠蔽してしまおう．そうすれば，その関数のクライアントは，自分のコードの中でキャストを使わなくて済むようになる．
- 古いCスタイルのものよりC++スタイルのキャストを使おう．C++スタイルのキャストは，見やすく，何をするかについても，よりはっきりしているから．
