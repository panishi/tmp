# Item 34：`std::bind` よりもラムダを優先する
## はじめに
`std::bind` よりもラムダを優先する最も大きな理由は，その高い可読性だ．
例えば，警告音を設定する次のような関数を考える．
```cpp
// 時刻のtypedef（文法についてはItem 9を参照）
using Time = std::chrono::steady_clock::time_point;
// "enum class"についてはItem 10を参照
enum class Sound { Beep, Siren, Whistle };
// 鳴らす長さのtypedef
using Duration = std::chrono::steady_clock::duration;
// 時刻tで音sをd時間鳴らす
void setAlarm(Time t, Sound s, Duration d);
```

## ラムダを使った場合
プログラムのある時点で，1時間後に30秒間警告音を鳴らすと設定してみよう．
ただし音の種類はまだ指定されていない．
`setAlarm` のインタフェースを改善し，音の種類のみを指定するラムダを次のように記述する．
```cpp
// setSoundLは設定から1時間後に30秒間鳴らす音を指定する関数オブジェクト
const auto setSoundL = [](Sound s)
{
    // 修飾しなくともstd::chrono コンポーネントを使用可能にする
    using namespace std::chrono;

    setAlarm(
        steady_clock::now() + hours(1),
        s,
        seconds(30));
};
```

ラムダの経験があまりなくとも，ラムダの仮引数 `s` が `setAlarm` に実引数として渡されていると分かるだろう．

C++14では，C++11のユーザ定義リテラルを基とし，秒，ミリ秒，時間などを表す標準接尾語も使用できる．
この接尾語は `std::literals` 名前空間で定義されている．
```cpp
const auto setSoundL = [](Sound s)
{
    using namespace std::chrono;
    using namespace std::literals;  // C++14 の接尾語

    setAlarm(
        steady_clock::now() + 1h,
        s,
        30s);
};
```

## `std::bind` を使った場合（誤ったver）
`std::bind` を用いた書き方を説明する．
誤りを含んだままだが，すぐ後で修正する．
```cpp
using namespace std::chrono;
using namespace std::literals;
using namespace std::placeholders;  // 「_1」の使用に必要

const auto setSoundB = std::bind(
    setAlarm,
    steady_clock::now() + 1h,
    _1,
    30s);
```

プレースホルダ「`_1`」は初心者には謎に見えるだろう．
これは `setSoundB` 呼び出しの先頭実引数を `setAlarm` の第2実引数に渡すことを表す．
この実引数の型は `std::bind` 呼び出しでは明示されていないため，`setSoundB` に渡される実引数を判断するには `setAlarm` の宣言を確認する必要がある．

ただし，上例には誤りがある．
ラムダの場合は，「`steady_clock::now() + 1h`」という式が `setAlarm` の実引数であるという点が明らかであり，`setAlarm` を呼び出す際に評価される．
しかし，`std::bind` 呼び出しでは「`steady_clock::now() + 1h`」を，`setAlarm` ではなく `std::bind` の実引数として渡している．
そのため，この式は `std::bind` 呼び出し時に評価され，式が表現する時刻はバインドオブジェクト内に保持される．
その結果，警告音は `setAlarm` 呼び出しから1時間後ではなく，**`std::bind` 呼び出しから1時間後に** 鳴ってしまう！

## `std::bind` を使った場合（正しいver）
この問題を解決するには，`setAlarm` が呼び出されるまで `std::bind` に式の評価を遅延させるよう通知する必要がある．
そのためには，`std::bind` 呼び出し内で `std::bind` をさらに呼び出せばよい．
```cpp
const auto setSoundB = std::bind(
    setAlarm,
    std::bind(
        std::plus<>(),
        std::bind(steady_clock::now),
        1h),
    _1,
    30s);
```

上例で記述してあるのは「`std::plus<>`」であり，「`std::plus<type>`」ではない．
C++14では標準演算子テンプレートのテンプレート型実引数は一般に省略可能なため，記述する必要がない．
C++11にはこの機能がないため，C++11次のようになる．
```cpp
Time addTimeAndDuration(Time t, Duration d)
{
    return t + d;
}

const auto setSoundB = std::bind(
    setAlarm,
    std::bind(
        addTimeAndDuration,
        std::bind(steady_clock::now),
        hours(1)),
    _1,
    30s);
```

C++14の汎用 `std::plus` を実質的に自作する手もある．
```cpp
struct genericAdder
{
    template<typename T1, typename T2>
    auto operator()(T1&& param1, T2&& param2)
        -> decltype(std::forward<T1>(param1) + std::forward<T2>(param2))
    {
        return std::forward<T1>(param1) + std::forward<T2>(param2);
    }
};

const auto setSoundB = std::bind(
    setAlarm,
    std::bind(
        genericAdder(),
        std::bind(steady_clock::now),
        hours(1)),
    _1,
    30s);
```

上例を見てもラムダの方が魅力的に思えないようであれば，目を検査した方が良いだろう（by 著者）．

## `setAlarm` をオーバロードした場合
`setAlarm` をオーバロードすると，新たな問題が発生する．
第4仮引数として警告音の音量をとるオーバロードを追加した場合を考える．
```cpp
enum class Volume { Normal, Loud, LoudPlusPlus };

void setAlarm(Time t, Sound s, Duration d, Volume v);
```

ラムダの場合は，実引数が3つのバージョンの `setAlarm` にオーバロード解決するため，変わらず動作する．

しかし，`std::bind` ではコンパイルすらできない．
コンパイラからすると，2つある `setAlarm` 関数のどちらを `std::bind` に渡すべきかを判断できない．
コンパイラにとって既知なのは関数名だけであり，これだけでは情報不足で曖昧だ．

これをコンパイル可能にするには，`setAlarm` を適切な関数ポインタ型へキャストする必要がある．
```cpp
using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);

auto setSoundB = std::bind(
    static_cast<SetAlarm3ParamType>(setAlarm),
    std::bind(
        std::plus<>(),
        std::bind(steady_clock::now),
        1h),
    _1,
    30s);
```

この変更により，ラムダと `std::bind` の相違点がもう1つ生まれる．
`setSoundL` の関数呼び出し演算子（ラムダのクロージャクラスの関数呼び出し演算子）内では，`setAlarm` 呼び出しはコンパイラが普段通りにインライン展開できる通常の関数呼び出しだ．
```cpp
setSoundL(Sound::Siren);  // setAlarm本体をここにインライン展開可能
```

しかし，`std::bind` に渡しているのは `setAlarm` を指す関数ポインタであり，`setSoundB` の関数呼び出し演算子（バインドオブジェクトの関数呼び出し演算子）内では，`setAlarm` 呼び出しは関数ポインタを介して実行される．
`setSoundB` 経由の `setAlarm` 呼び出しは，`setSoundL` 経由の場合と比べて全体がインライン展開されるなどまずあり得ない．
```cpp
setSoundB(Sound::Siren);  // ここでsetAlarm本体はまずインライン展開されない
```

このため，`std::bind` を用いるよりも，ラムダを用いた方が高速なコードを生成可能となっている．

## より複雑な処理を実行する場合
より複雑な処理を実行する場合は，ラムダの方がずっと有利になる．
例えば，実引数が最小値 (`lowVal`) と最大値 (`highVal`) の間にあるか否かを判定する，C++14のラムダを考えてみる．
```cpp
const auto betweenL = [lowVal, highVal](const auto& val)
{
    return lowVal <= val && val <= highVal;
};
```

`std::bind` でも同じ内容を表現でるが，「処理内容は維持し見た目を難解にする (job security through code obscurity)」の典型のようなコードになる．
```cpp
using namespace std::placeholders;

const auto betweenB = std::bind(
    std::logical_and<>(),
    std::bind(std::less_equal<>(), lowVal, _1),
    std::bind(std::less_equal<>(), _1, highVal));
```

C++11では比較する型を明示する必要があるため，次のようになる．
```cpp
const auto betweenB = std::bind(
    std::logical_and<bool>(),
    std::bind(std::less_equal<int>(), lowVal, _1),
    std::bind(std::less_equal<int>(), _1, highVal));
```

当然，C++11のラムダは `auto` 仮引数を受け取れないため，ここでも型を明示しなければならない．
```cpp
const auto betweenL = [lowVal, highVal](int val)
{
    return lowVal <= val && val <= highVal;
};
```

どちらにせよ，ラムダを用いる方が記述量を減らせるだけでなく，分かりやすく，かつ保守性にも優れると賛成して頂けるだろう．

## `std::bind` のややこしい点
`std::bind` の動作を把握し切れないものは，プレースホルダ以外にもある．
`Widget` の圧縮したコピーを作成する関数を考えてみる．
```cpp
enum class CompLevel { Low, Normal, High };  // 圧縮レベル

Widget compress(                             // wを圧縮したコピーを作成
    const Widget& w,
    CompLevel lev);
```

`Widget w` の圧縮レベルを指定する関数オブジェクトも作成する．
`std::bind` を用いると以下の通り．
```cpp
using namespace std::placeholders;

Widget w;
const auto compressRateB = std::bind(compress, w, _1);
```

上例を用いて `w` を `std::bind` へ渡すと，`std::bind` は後の `compress` 呼び出しに備えて `w` を内部に保持する．
この保持の仕方は，値だろうか，それとも参照だろうか？
`std::bind` 呼び出しから `compressRateB` 呼び出しまでの間に `w` が変更されるようであれば，参照ならば変更が反映されるが，値ならば反映されない．
この違いは大きな意味を持つ．

答えは，「値を保持する」だ．
この動作は `std::bind` 呼び出しからでは読み取れず，`std::bind` はこう動作すると覚えるしかない．
ただし，呼び出し側で `std::ref` を用いれば，参照として保持される実引数も実現できる．
```cpp
const auto compressRateB = std::bind(compress, std::ref(w), _1);
```

一方で，ラムダを用いれば，`w` の値・参照のどちらをキャプチャするのかを明示できる．
```cpp
const auto compressRateL = [w](CompLevel lev)
{
    return compress(w, lev);
};
```

ラムダは，どのように仮引数を渡すかも明示する．
自明だが，仮引数 `lev` は次のように値渡しだ．
```cpp
compressRateL(CompLevel::High);
```

`std::bind` により作成したオブジェクトを呼び出す場合は，実引数はどのように渡されるだろうか？
```cpp
compressRateB(CompLevel::High);
```

実際には，全実引数はバインドオブジェクトに参照渡しされる．
バインドオブジェクトの関数呼び出し演算子が完全転送するためだ．
ここでも，`std::bind` はこう動作すると丸暗記するしかない．

## C++11における `std::bind`
ラムダと比較すると，`std::bind` を用いたコードは可読性，表現力が乏しく．恐らくは効率でも劣る．
C++14で `std::bind` を用いる妥当性はないが，C++11では次の2つの制約が発生する場面では妥当と言える．

### ムーブキャプチャ (move capture)
C++11のラムダではムーブキャプチャを使用できないが，ラムダと `std::bind` を組み合わせればエミュレートできる（Item 32を参照）．

### 多態関数オブジェクト (polymorphic function object)
バインドオブジェクトの関数呼び出し演算子は内部で完全転送するため，任意の型の実引数を受け取れる（Item 30で述べた完全転送の制限は除く）．
この動作は，テンプレート化した関数呼び出し演算子を持つオブジェクトをバインドする場合に有用である．
例えば、次のクラスがあるとする．
```cpp
class PolyWidget
{
public:
    template<typename T>
    void operator()(const T& param) const;
    …
};
```

`std::bind` では `PolyWidget` を次のようにバインドできる．
```cpp
PolyWidget pw;
const auto boundPW = std::bind(pw, _1);
```

`boundPW` には異なる型の実引数を与え呼び出せる．
```cpp
boundPW(1930);       // PolyWidget::operator()にintを渡す
boundPW(nullptr);    // PolyWidget::operator()にnullptrを渡す
boundPW("Rosebud");  // PolyWidget::operator()に文字列リテラルを渡す
```

C++11のラムダでは上例を実現できないが，C++14では `auto` 仮引数を用いて容易に実現できる．
```cpp
const auto boundPW = [pw](const auto& param)
{
    pw(param);
};
```

もちろんこれは極端な例だが，C++14のラムダに対応したコンパイラの普及はすでに始まっているため，これは一時的なことだろう．

## おわりに
2005年に `bind` がC++に非公式に追加された時は，C++98からの大きな改善だった．
しかし，C++11でラムダ対応が追加されたため `std::bind` は時代遅れの代物と化し，C++14にいたっては使用する理由もなくなった．

## Things to remember
- `std::bind` よりもラムダの方が可読性，表現力に優れ，恐らく効率でも勝る．
- C++11 に限るが，`std::bind` はムーブキャプチャの実装や，テンプレート化した関数呼び出し演算子を持つオブジェクトをバインドする際に有用となる場合がある．
