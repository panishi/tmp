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
Item 17: `new` で生成したオブジェクトをスパートポインタに渡すのは，独立したステートメントで行うようにしよう

2021/x/x

---
### はじめに
- 整数値を戻す `priority` という関数と，動的に確保した `Widget` と `priority` の戻り値を引数に取る関数 `processWidget` を考える．
```cpp
int priority();
void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);
```
- Item 13に従って，第1引数はスマートポインタで受け取るようにしている．

---
### 生ポインタで呼び出す方法
- 以下のように呼び出すことは出来ない．
```cpp
processWidget(new Widget, priority());
```
- `tr1::shared_ptr` の「ポインタを受け取るコンストラクタ」は `explicit` 宣言されており，暗黙の型変換は出来ない．
- 次のように呼び出せばよい．
```cpp
processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());
```

---
### リソース漏れの危険性1
- 上のコードでは，リソース管理クラスを使っているにも関わらず，リソース漏れの危険性がある！
- `processWidget` の呼び出しの前に，コンパイラは次の3つの処理のバイナリコードを生成する．
    - `priority` の呼び出し
    - 「`new Widget`」の実行
    - `tr1::shared_ptr` のコンストラクタの呼び出し

---
### リソース漏れの危険性2
- C++では，関数の引数評価の順序は定まっていない（JavaやC#では決められている）．
- 仮に以下の順序で呼ばれたとする．
    - 「`new Widget`」の実行
    - `priority` の呼び出し
    - `tr1::shared_ptr` のコンストラクタの呼び出し
- `priority` が例外を投げた場合，`new Widget` で生成されたポインタは `tr1::shared_ptr` に格納されず，リソース漏れが発生する．

---
### 問題の回避法
- `Widget` オブジェクトを生成し，そのポインタをスマートポインタに引き渡すのを，独立したステートメントで行えばよい．
```cpp
std::tr1::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
```
- ステートメントを分けることで，実行される順序がきちんと定まり，`tr1::shared_ptr` オブジェクトを生成している最中で `priority` が呼び出されることがなくなる．

---
### おわりに
- C++11以降では別な解決法がある．
```cpp
// リソース漏れの恐れはない
processWidget(std::make_shared<Widget>(), priority());
```
- Effective Modern C++のItem 21を参照．

---
### Things to Remember
- `new` で生成したオブジェクトをスマートポインタに渡すのは，独立したステートメントで行うようにしよう．
そうしないと，例外が投げられたときに，リソース漏れが起こるかもしれない．
