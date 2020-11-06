# Item12：オーバライドする関数は `override` と宣言する
## はじめに
「オーバライド (override)」という言葉は「オーバロード（overload)」によく似ているため，初めにその違いを明確にしておく．
仮想関数のオーバライドとは，基底クラスのインタフェースを介して派生クラスの関数の実行を実現するものだ．
```cpp
class Base {
public:
    virtual void doWork();  // 基底クラスの仮想関数
…
};
class Derived : public Base {
public:
    virtual void doWork();  // Base::doWorkをオーバライド（この「virtual」は必須ではない）
…
};

std::unique_ptr<Base> upb
    = std::make_unique<Derived>();  // 派生クラスオブジェクトを指す，
                                    // 基底クラス型のポインタを作成
                                    // std::make_uniqueについてはItem21を参照
…

upb->doWork();  // 基底クラスのポインタ経由でdoWorkを呼び出す
                // 派生クラスの関数が実行される
```

## オーバーライドの要件と参照修飾子
オーバライドの要件は以下の通り．
- 基底クラスの関数は仮想関数でなければならない．
- 基底クラス，派生クラスの関数名は一致していなければならない（デストラクタは例外）．
- 基底クラス，派生クラスそれぞれの関数の仮引数型は一致していなければならない．
- 基底クラス，派生クラスそれぞれの関数の `const` 性は一致していなければならない．
- 基底クラス，派生クラスそれぞれの関数の戻り型および例外指定は互換でなければならない．

C++11ではもう1つ要件がある．
- 関数の **参照修飾子** (reference qualifier) も一致していなければならない．

メンバ関数の参照修飾子は，C++11のあまり宣伝されない機能の1つなため，今まで耳にしていなくとも驚くことはない．
メンバ関数の参照修飾子とは，その関数を左辺値のみ・右辺値のみに使用するよう制限を課すもの．
参照修飾子は仮想関数かどうかとは独立に指定可能だ．
```cpp
class Widget {
public:
…
    void doWork() &;   // このバージョンのdoWorkは
                       // *thisが左辺値の場合にのみ使用可能

    void doWork() &&;  // このバージョンのdoWork は
};                     // *thisが右辺値の場合にのみ使用可能
…

Widget makeWidget();  // factory 関数（右辺値を返す）
Widget w;             // 通常のオブジェクト（左辺値）
…

w.doWork();             // 左辺値用のWidget::doWorkを呼び出す
makeWidget().doWork();  // 右辺値用のWidget::doWorkを呼び出す
```

メンバ関数の参照修飾子については後で詳細に解説する．
基底クラスの仮想関数に参照修飾子を指定した場合，オーバライドする派生クラスの関数にも同じ参照修飾子が必要であるとは覚えておこう．
2つの参照修飾子が一致しなければ，派生クラスで宣言した関数はそのまま残りるが，基底クラスの関数をオーバライドしない．

## オーバーライドしないケース
オーバライドに関する誤りを含むコード（上記要件を満たさないコード）でも通常はコンパイルできる．
しかし，プログラマの意図とは異なる動作となる．
以下のコードは完全に正当かつ一見もっともらしく見えるが，仮想関数を何もオーバライドしていない．
```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};

class Derived : public Base {
public:
    virtual void mf1();
    virtual void mf2(unsigned int x);
    virtual void mf3() &&;
    void mf4() const;
};
```

オーバーライドされない理由は以下の通り．
- `mf1` は，`Base` が `const` と宣言している，`Derived` ではそうしていない．
- `mf2` は，`Base` では `int` をとるが，`Derived` では `unsigned int` である．
- `mf3` は，`Base` では左辺値修飾されているが，`Derived` では右辺値修飾されている．
- `mf4` は，`Base` が `virtual` と宣言していない．

「でも実際にはコンパイラは何も警告してこないし，気にすることはないっしょｗ」と思う読者もいるかもしれない．
しかし恐らく気にすることになるだろう．
（話者注：ポリモルフィックな動作をしなくなってしまうので，バグの温床となるだろう）．

## `override` キーワード
派生クラスでのオーバライドが正しく動作するよう宣言するのは重要でありながらも誤りやすいため，C++11では派生クラスの関数が基底クラスの関数をオーバライドすることを明示する方法が提供された．
`override` と宣言すればよい．
```cpp
class Derived : public Base {
public:
    virtual void mf1() override;
    virtual void mf2(unsigned int x) override;
    virtual void mf3() && override;
    virtual void mf4() const override;
};
```

上例は当然コンパイルできない．
このように記述すればコンパイラはオーバライドに関するエラーメッセージを期待通りまくしたててくれる．
このため，オーバライド関数はすべて `override` と宣言すべき．

`override` を用いた，コンパイル可能なコードは次の通り．
```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    virtual void mf4() const;
};

class Derived : public Base {
public:
    virtual void mf1() const override;
    virtual void mf2(int x) override;
    virtual void mf3() & override;
    void mf4() const override;  // 「virtual」を追加しても問題ないが，必須ではない
};
```

正しく動作させるため，上例では `Base::mf4` を `virtual` と宣言している．
オーバライドに関する誤りのほとんどは派生クラスで発生するが，基底クラスが原因になることもある．

オーバライドするすべての派生クラスで `override` と記述するポリシは，基底クラスの仮想関数のシグネチャの変更を検討する場合に，その影響を図る目安にもなる．
派生クラスですべてに `override` を用いておき，シグネチャを試しに変更し，システムをリコンパイルし，変更の影響範囲を確認すれば（どの派生クラスがコンパイルエラーになったか），そのシグネチャの変更が本当に値するかを否かを判断できる．
`override` を用いなければ，単体テストがカバーする範囲が広いことを願うしかない．
派生クラスで基底クラスの仮想関数をオーバライドしたつもりでも実際にはできていない場合があり，コンパイラの警告もないであろうためだ．

## コンテクスト依存予約語
これまでもC++には予約語 (keyword) があったが，C++11では **コンテクスト依存予約語** (contextual keyword) が2つ追加された．
`override` と `final` だ．
この2つの英単語は予約されてはいるが，予約語として意味をなすのはある決まったコンテクスト（場面，文脈）に限定される．
`override` が意味をなすのは，メンバ関数宣言の末尾に記述された場合のみに制限される．
```cpp
class Warning {       // C++98 で使用していた可能性がある古いクラス
 public:
    …
    void override();  // C++98 でもC++11 でも正当（意味も変わらない）
    …
};
```

## メンバ関数の参照修飾子について
実引数に左辺値のみをとる関数を記述する場合，非 `const` な左辺値参照仮引数とするのが通例だ．
また，実引数に右辺値のみをとる関数を記述する場合，右辺値参照仮引数とするのが通例だ．
```cpp
void doSomething(Widget& w);   // 左辺値のWidgetのみをとる
void doSomething(Widget&& w);  // 右辺値のWidgetのみをとる
```

メンバ関数の参照修飾子は，そのメンバ関数を実行するオブジェクト (`*this`) を同様に
区別するためのもの．
メンバ関数宣言の末尾に記述し，`*this` が `const` であることを表す `const` とよく似ている．

`Widget` クラスが `std::vector` のメンバ変数を持っており，コード利用者が直接アクセスできるようアクセサ関数を提供する場合を考える．
```cpp
class Widget {
public:
    using DataType = std::vector<double>;  // 「using」についてはItem9を参照
    …
    DataType& data() { return values; }
    …
private:
    DataType values;
};
```

上例はカプセル化が高度には図られておらず，まず陽の目を見ることがない設計だが，それは置いておいて，コード利用側がどうなるかを考えよう．
```cpp
Widget w;
…
auto vals1 = w.data();  // w.valuesをvals1へコピー
```

`Widget::data` の戻り型は左辺値参照 (`std::vector<double>&`) であり，左辺値参照は左辺値となるよう定義されているため，`vals1` を左辺値から初期化することになる．
そのためコメントにもあるよう，`vals1` は `w.values` からコピーコンストラクトされる．

ここで，`Widget` を作成する `factory` 関数があるとしよう．
```cpp
Widget makeWidget();
```

また，`makeWidget` が返す `Widget` 内の `std::vector` を用い，変数を初期化したいとする．
```cpp
auto vals2 = makeWidget().data();  // Widget内の値をvals2へコピー
```

上例でも `Widgets::data` は左辺値参照を返し，左辺値参照は同様に左辺値になるため，やはり新規オブジェクト (`vals2`) は `Widget` 内の `values` からコピーコンストラクトされる．
しかし，今回の `Widget` は `makeWidget` が返した一時オブジェクト（右辺値）のため，内部で実行される `std::vector` のコピーは無駄だ，
ここではムーブの方が望ましいが，`data` は左辺値参照を返すため，C++の規則ではコンパイラがコピーするコードを生成することとなっている．

必要なのは，右辺値 `Widget` の `data` 呼び出しでは，結果も右辺値となることを明示する方法だ，
参照修飾子を用いることで，左辺値，右辺値どちらのオーバロードも実現できる．
```cpp
class Widget {
public:
    using DataType = std::vector<double>;
…
    DataType& data() & { return values; }             // Widgetが左辺値ならば，左辺値を返す
    DataType data() && { return std::move(values); }  // Widgetが右辺値ならば，右辺値を返す
…
private:
    DataType values;
};
```

なお，メンバ関数に参照修飾子を付加した場合，そのオーバロードにも参照修飾子を付加しなければならない．
以下の例は誤ったコードだ．
```cpp
class Widget {
public:
…
    DataType& data() && { return std::move(values); }  // Widgetが右辺値ならば，右辺値を返す
    DataType data()　{ return values; }  // エラー！「data」の他のオーバロードが参照修飾されている
…
}; 
```

左辺値参照のオーバロードは左辺値参照を返し，右辺値参照のオーバロードは一時オブジェクトを返す．
このため，利用者コードは期待通りに動作するようになる．
```cpp
auto vals1 = w.data();             // Widget::dataの左辺値オーバロードを呼び出す
                                   // vals1をコピーコンストラクト
auto vals2 = makeWidget().data();  // Widget::dataの右辺値オーバロードを呼び出す
                                   // vals2をムーブコンストラクト
```

メンバ関数 `data` の右辺値参照オーバロードが右辺値を返すようにするには，右辺値参照を返す方が良い．
戻り値用に一時オブジェクトを作成する動作を避けられ，元々の `data` インタフェースの参照戻しとも一貫性を維持できる．
```cpp
class Widget {
public:
    using DataType = std::vector<double>;
…
    DataType& data() & { return values; }
    DataType&& data() && { return std::move(values); }
…
private:
    DataType values;
};
```

## おわりに
この動作が良いことは紛れもないが，ハッピーエンドの暖かな光に惑わされ，本項目の真のポイントを忘れてはいけない．
真のポイントは，「基底クラスの仮想関数を派生クラスでオーバライドする場合は，常に `override` を付加し宣言すること」だ．

## Things to remember
- オーバライド関数は `override` と宣言する．
- メンバ関数の参照修飾子を用いると，左辺値オブジェクト，右辺値オブジェクト (`*this`) を区別できる．
