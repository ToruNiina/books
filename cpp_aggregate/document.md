---
title: C++ 集成体
author: onihusube
date: 2020/12/26
geometry:
  width: 188mm
  height: 263mm
coverimage: cover.jpg
backcoverimage: backcover.jpg
okuduke:
  revision: 第2版
  #printing: 日光企画
---
\clearpage

# はじめに

　この本はC++20をベースとして書かれています。基本的なところはC++17以前から変わってはいませんが、一部の仕様はC++のバージョン毎に異なったりしています。もしかしたらお手元のコンパイラではエラーになったり挙動が異なったりするかもしれません。

# 集成体（*Aggregate*）とは

　集成体（*Aggregate*）とはいくつかの条件を満たした構造体（クラス）および共用体の事です（C形式の配列も集成体ですがここでは構造体だけに着目します）。とはいえ特別なものではなく、C++におけるC言語の構造体相当の構造体の事をそう呼びます。最も有名な集成体は`std::array`でしょう。`std::array`は集成体となるように実装されます。

```cpp
// 集成体の一例
struct aggregate {
  int n;
  double m;
  std::string str;
};

// 集成体初期化
aggregate agg = {20, 3.14, "Aggregate"};
std::array<int 5> arr = {1, 2, 3, 4, 5};
```

　このように、集成体では集成体初期化によって仮想的なコンストラクタが提供されるため、メンバを初期化するためにコンストラクタを書く必要が無くなります。

　集成体は、単にいくつかのデータをまとめたタプルの様な型やすべてのメンバがパブリックであるようなクラスに対して活用することができ、余計なコンストラクタやセッター/ゲッターといったボイラープレートコードを削減することができます。

\clearpage

# 集成体の要件

　ある構造体（クラス）が集成体となるためには次の条件を満たしていなければなりません。

1. ユーザー宣言コンストラクタおよび継承コンストラクタを持たない
2. `public`ではない（非`static`）メンバ変数を持たない
3. 仮想関数を持たない
4. `public`以外の継承をしていない

　これらの規則を守ってさえいれば他の事は何をしても大丈夫です。また、この規則を守らなくても別にコンパイルエラーになったりせず、普通のクラスと同等の扱いになるだけです。

## ユーザー宣言コンストラクタおよび継承コンストラクタを持たない

　これは言い換えると、あらゆるコンストラクタを宣言していないということです。コピー/ムーブコンストラクタ、`= default`や`= delete`なコンストラクタも含めて、あらゆるコンストラクタを書いてはいけません。

```cpp
// これらはどれも集成体ではない

struct A1 {
  int n;

  A(int m) : n(m) {}
};

struct A2 {
  int n;

  A() = default;
};

struct A3 {
  int n;

  A(A&&) = default;
};
```

　このようなコンストラクタ宣言が一つでもあるとその構造体は集成体ではなくなります。

　もう一つの継承コンストラクタというのは、継承しているときに基底クラスのコンストラクタを有効化する`using`のことで、これによって基底クラスのコンストラクが利用可能になりますが、コンストラクタが宣言されているのと等価なので集成体ではなくなります。

```cpp
struct base {
  int n;

  base() : n(0) {}

  base(int m) : n(m) {}
};

// A1は集成体ではない
struct A1 : base {
  using base::base; // コンストラクタを継承する
};

// A2は集成体
struct A2 : base {};
```

　この規定があることからわかるように、集成体になるにあたっては公開継承している分には問題ないですし、基底クラスが集成体でなくてもOKです。

## `public`ではないメンバ変数を持たない

　これはそのままの意味です。`private`とか`protected`でメンバ変数を宣言してはいけません。ただし、この場合のメンバ変数とは非静的なメンバ変数であって、`static`メンバ変数がどこにあろうと集成体となることを妨げません。

```cpp
// これは集成体ではない
class A1 {
  int n;  // classのデフォルトアクセス指定はprivate
};

// これも集成体ではない
struct A2 {
  int n;

protected:
  int m;
};

// これは集成体
struct A3 {
  int n;

private:
  static constexpr int M = 0;
};
```

　メンバのアクセス指定の制限があるのは非静的メンバ変数だけなので、静的メンバ変数同様にメンバ関数のアクセス指定も自由に書くことができます。

## 仮想関数を持たない

　仮想関数を持つということは構造体のレイアウトに仮想関数テーブルなどの動的ポリモルフィズムのための不可視のメンバが追加されることになるため、集成体ではなくなります。また、継承によって間接的に持つこともできません。

```cpp
struct interface {
  virtual int f() = 0;
};

// 集成体ではない
struct A1 {
  int n;

  virtual int g() {
    return 1;
  }
};

// 集成体ではない
struct A2 : interface {
  int n;
};
```

　インターフェースクラスを継承するような場合は残念ながら集成体を活用することはできません。

## `public`以外の継承をしていない

　継承すること自体は問題ないのですが、集成体では`public`継承だけが許可されます。

```cpp
// 集成体ではない
struct base {
  int n;

  base() = default;
};

// 集成体ではない
struct A1 : private base {
  double d;
};

// 集成体ではない
struct A2 : protected base {
  double d;
};

// これは集成体
struct A3 : public base {
  double d;
};
```

　この例からもわかるように、継承する場合は基底クラスが集成体でなくてもOKです。

\clearpage

# 集成体の特性

　ここからは、構造体を集成体にすることによって得られるいくつかの性質を見ていきます。

## 構造化束縛への適合

　集成体にすることによって構造化束縛で自然に分解できるようになります。本来構造化束縛に適合するためには`std::tuple`と同様のタプルインターフェースを実装しなければなりませんが、集成体は言語サポートによって何もしなくても構造化束縛できます。

```cpp
struct point3d {
  double X, Y, Z;
};

point3d f() {
  return {1.0, 2.0, 3.0};
}

int main() {
  auto [x, y, z] = f();
  // x == 1.0, y == 2.0, z == 3.0
}
```

## ボイラープレートコードの削減

　集成体初期化が可能になることによってコンストラクタを書かなくて良くなり、すべてのメンバは必然的に`public`なのでゲッター/セッターのようなものも必要なくなります。

```cpp
struct aggregate {
  int n;
  double m = 2.72;
  std::string str;
};

// このクラスを普通に書くと例えば次のようになる

class not_aggregate {
  int n;
  double m = 2.72;
  std::string str;

public:

  not_aggregate() = default;

  not_aggregate(int an, double am, std::string astr)
    : n(an)
    , m(am)
    , str(std::move(astr))
  {}

  int get_n() const {
    return n;
  }

  void set_n(int an) const {
    n = an;
  }

  // 以下略
};
```

　あえて書いてみるとそれなりの量のコードを削減することができるのが分かるでしょう。

　また、構造化束縛をする場合はタプルインターフェースを備える必要がありますが、これもそこそこの量のボイラープレートが必要になります。先程説明したように、集成体は構造化束縛にそのまま適合できるため、ここでもそれなりの量のボイラープレートを削減できます。

## 必然的な*trivial*性

　集成体はコンストラクタを宣言できず、それによって代入演算子を書くこともほぼなく、デストラクタをあえて書くことも稀です。そのため、多くの場合必然的に*trivial*なクラスとなることができます。*trivial*なクラスとなることで、そうでないクラスと比べて関数に渡すときに有利になることがあります。

```cpp
struct agg_tuple {
  int a, b;
};

void func_tuple(std::tuple<int, int>);
void func_struct(agg_tuple);

int main() {
  // この2つの関数呼び出しは同じコードを生成してほしい
  func_tuple({1, 2});
  func_struct({1, 2});
}
```

　例えばこのようなコードをコンパイルすると、集成体である`agg_tuple`を渡す関数呼び出しは`int`型の値2つをレジスタに配置して引数として渡すコードが生成されますが、`std::tuple`を渡す関数呼び出しでは`int`型の値2つをスタックに配置してそのポインタを引数として渡すコードが生成されることがあります。これがまさに、`std::tuple`が*trivial*ではないために起こることです（このことはライブラリの実装によって変化します）。例えば、次の様なコードが生成されます。

```asm
main:
  push    rax
  # func_tuple()の呼び出し
  movabs  rax, 4294967298
  mov     qword ptr [rsp], rax
  mov     rdi, rsp
  call    func_tuple(std::tuple<int, int>)
  # func_struct()の呼び出し
  movabs  rdi, 8589934593
  call    func_struct(agg_tuple)
  # main()からのreturn
  xor     eax, eax
  pop     rcx
  ret
```

　このことの原因は、ABIによって「非*trivial*な型のオブジェクトを関数に渡す際は、一時オブジェクトを作成しその参照を渡す」というように定められているためです。ここでのABIとは非Windowsのx86-64環境で使われるItanium C++ ABIを指しますが、x86-64環境ならば他のABIでも同様の要求があるようです。そして、それらABIの要求する非*trivial*な型とは、次のいずれかに該当するものです。

- 非*trivial*なコピーコンストラクタを持つ
- 非*trivial*なムーブコンストラクタを持つ
- 非*trivial*なデストラクタを持つ
- 全てのコピー/ムーブコンストラクタが`delete`されている

　`std::tuple`は言語サポートが無く純粋にC++のコードとして実装されるため、その実装は可変長テンプレートと継承を駆使した非常に複雑なものになります。その結果、実装によっては上記のいずれかに抵触し*trivial*性が失われることがあります。一方、`std::pair`はおそらくどう実装しても要素型が*trivial*であれば`std::pair`自身も*trivial*となるので、この様な問題は起きないはずです。

　集成体では`std::pair`と同様に、要素型と基底クラスが*trivial*であれば必然的に*trivial*となります。この様な複雑な事情を知らずとも、この様なオーバーヘッドを自然に回避することができます。

　ちなみに、同様の問題は`std::optional`や`std::variant`でも起こりえることですが、こちらはその要素型が（全て）*trivial*であれば自身も*trivial*となるように規格によって規定されているのでこの様な問題は起こりません。`std::tuple`にはそのような規定は無く、実装によってはこの様な事が起こってしまいます。

\clearpage

## 集成体初期化（*Aggregate Initialization*）

　集成体初期化は集成体にすることによって得られる最も大きな特徴でしょう。`{}`（波かっこ）を用いた初期化構文によって初期化できるようになり、通常の`()`（丸かっこ）による初期化とは少し違った振る舞いをします。これは集成体にすることによって仮想的なコンストラクタが自動生成されていると見ることができます。

```cpp
struct aggregate {
  int n;
  double m = 2.72;  // デフォルトメンバ初期化
  std::string str;
};

// 集成体初期化
aggregate agg1 = {20, 3.14, "Aggregate"};
aggregate agg2 = {20, 3.14}; // strはデフォルトコンストラクタで初期化
aggregate agg3 = {20};       // mは2.72で初期化される
aggregate agg4 = {};         // nはゼロ初期化（0相当の値で初期化）される
```

　集成体初期化は基本的にはこのように波かっこによって初期化します。初期化子の数が集成体のメンバの数を超えているとエラーになりますが、足りない場合はデフォルトメンバ初期化によって初期化され、それが無い場合は再帰的に`{}`で初期化されることになります。

　また、明示的に空のリスト（`{}`）によって初期化することもでき、その場合の初期化はそれぞれのメンバを`{}`で初期化したようになります（デフォルトメンバ初期化は無視されます）。ただし、`{}`さえも省略して初期化子をなしにすることはできません。

```cpp
aggregate agg1 = {20, 3.14, {}}; // strはデフォルトコンストラクタで初期化
aggregate agg2 = {20, {}, {}};   // mはゼロ初期化（0.0）される
aggregate agg3 = {{}, {}, {}};   // nはゼロ初期化（0）される
aggregate agg4 = { , , };        // コンパイルエラー、{}が必要
```

　このような初期化は、デフォルトコンストラクタを持つクラス型の場合はデフォルトコンストラクタを呼び出すことに対応しています。

　このことと同じように、集成体に含まれるクラス型のコンストラクタを`{}`によって呼び出すことができます。

```cpp
// strは"string"の先頭3文字で初期化
aggregate agg1 = {20, 3.14, {"string", 3}};

// strは"string"と渡されたアロケータで初期化
aggregate agg2 = {20, 3.14, {"string", oreore_alloc{}}};
```

　この時、`{}`で初期化されているメンバが集成体であれば集成体初期化によって初期化されます。ただ残念ながら、このときに`()`でコンストラクタを呼び出すことはできません。それによって、`std::string`の特定のコンストラクタが呼び出せないなどの影響があります。

　集成体が継承をしているときは、集成体初期化の最初で基底クラスに対する初期化をを行います。もし意図せずに省略された場合はその分初期化子の対象がずれることになり、思わぬバグを生むかもしれません。

```cpp
// 集成体ではない
struct base {
  int n;

  base() = default;
  base(int m) : n(m) {}
};

struct aggregate2 : base {
  int n;
  double m = 2.72;
  std::string str;
};

// 基底クラスはデフォルトコンストラクタで初期化
aggregate2 agg1 = {{}, 10, 3.14, "str"};

// 基底クラスはbase::base(int)のコンストラクタで初期化
aggregate2 agg2 = {{20}, 10, 3.14, "str"};
aggregate2 agg3 = {20, 10, 3.14, "str"};

// エラー（初期化対象がずれており、この例では2番目と3番目で変換エラーが出る）
aggregate2 agg4 = {10, 3.14, "str"};
```

　この3番目の例のように、集成体初期化ではネストしている型の初期化に必要な`{}`を省略することができます。

```cpp
struct A {
  int arr[5];
};

struct aggregate3 : base {
  A array;
  int n;
  std::string str;
};

aggregate3 agg1 = {{10}, {{0, 1, 2, 3, 4}}, 1, "string"}; // フル{}
aggregate3 agg2 = {{10}, { 0, 1, 2, 3, 4 }, 1, "string"}; // 1段省略
aggregate3 agg3 = { 10 ,   0, 1, 2, 3, 4  , 1, "string"}; // 全省略
aggregate3 agg4 = { {} , {}               , 1, "string"}; // {}による初期化
```

　ただし、そのようなクラス型の引数が多かったりすると境界が曖昧になって読みづらくなるので`{}`は入れておいた方がいいでしょう。clangはこの場合の3番目の例の`A`の初期化（2番目の初期化子）に対してそのような警告を発します。

　ここまで集成体初期化の例にはすべて`= {}`の形の初期化構文を使用してきましたが、`=`をなくして変数名に直接続ける形の初期化も可能です。これらはどちらも集成体初期化として扱われます。

```cpp
aggregate  agg1{20, 3.14, {}};
aggregate2 agg2{{}, 10, 3.14, "str"};
aggregate3 agg3{{10}, { 0, 1, 2, 3, 4 }, 20, "string"};
```

　これと比較すると、`= {}`の構文は右辺で一時オブジェクトを生成して左辺にムーブのようにも見えるかもしれませんが、集成体初期化の場合はどちらの構文でも同じように直接左辺の変数とそのメンバを初期化します。

　実際のところ、これらのような集成体初期化を問題を起こさずに行えるような型のことを集成体と呼んでいます。先程の集成体の要件はすべてこのような集成体初期化を妨げないために設けられています。そして、集成体初期化はC言語の構造体にとっては普通の初期化方法であり、その意味で集成体とはCの構造体と同じようなC++の構造体と言えます。つまるところ、集成体とはかなり普通の構造体の事です。

### 縮小変換の禁止

　`{}`による集成体初期化ではその初期化に際して縮小変換（*Narrowing Conversion*）が禁止されています。これは`()`による初期化と異なっています。

　縮小変換とは変換先の型が変換元の型の表現を全て受け止めきれないような変換のことで、縮小変換前後で情報の欠落が発生します。C++はCからこれを受け継ぎ、特に警告やエラーになることもなく暗黙変換の一環として各所で実行されます。特に浮動小数点数型の縮小変換では精度低下を伴うので静かなバグを埋め込むことが多発しがちです。

　縮小変換とは以下の様な変換です。

- `double -> float`のような精度が落ちる浮動小数点数型の変換
- 浮動小数点数型と整数型の間の双方向の変換
- 整数型と整数型の間で、表現の欠落が発生する変換
    - ビット幅が小さくなる変換 : `int -> char`など
    - 符号付から符号なしへの変換 : `int -> std::size_t`など
    - 符号なしから符号付への変換で正の表現が欠落しうる変換 : `std::size_t -> short`など
- ポインタ型から`bool`への変換（C++20以降）

　集成体初期化においてはこのような変換が起こる場合はコンパイルエラーとなります。

```cpp
struct narrow {
  int n;
  float f;
};

int main() {
  unsigned int un = 10;
  long double ld = 1.0;

  narrow n1 = {20, 0.1f}; // OK
  narrow n2 = {un, 0.1f}; // unsigned -> intの縮小変換、コンパイルエラ－
  narrow n3 = {20, ld};   // long double -> floatの縮小変換、エラー
  narrow n4 = {un, ld};   // 両方縮小変換、コンパイルエラー
}
```

　ただし、定数式中の`{}`集成体初期化で縮小変換が発生する時、その変換に伴って情報の欠落が発生しないことが確認できる場合は一部の縮小変換が許可されます。

```cpp
struct A {
  unsigned int n;
};

int n = 10;
A a1 = {n};   // コンパイルエラー
A a2 = {10};  // OK
A a3 = {-1};  // コンパイルエラー
```

　このように、縮小変換が禁止されていることによってより安全な初期化が出来るようになっています。

　C++（コミュニティ、標準化委員会）の一つの見解（そして後悔）として、C言語から暗黙変換を受けつくべきではなかったというのがあります。この思想はC++にとどまらないようで、多くの後継の言語は暗黙変換を型システムから排除しています。この集成体初期化がそうであるように、C++11以降に追加される機能の多くはその思想に基づいて縮小変換を含めた暗黙変換を嫌う傾向にあります。ただ、暗黙変換は既にC++コードの多くの場所で定着しているため禁止されていることはユーザーから見ると不便であり、また意味不明なコンパイルエラーが発生することにもなります。そのため、単純に暗黙変換を排除することはできず、これからも付き合っていく必要がありそうです・・・

### 初期化子評価順序の規定

　集成体初期化の`()`と異なるもう一つの点は、`{}`内の初期化式の評価順序が規定されていることです。

```cpp
struct aggregate {
  int a, b, c, d;
};

struct not_aggregate {
  int a, b, c, d;

  not_aggregate(int n1, int n2, int n3, int n4)
    : a(n1), b(n2), c(n3), d(n4)
  {}
};

int main() {
  int i = 0;

  aggregate agg = {++i, ++i, ++i, ++i}; // 評価順序は左から右と規定される
  // agg = {1, 2, 3, 4}

  i = 0;

  not_aggregate na(++i, ++i, ++i, ++i); // 評価順序は未規定
  // na = {4, 4, 4, 4} GCC 10.1
  // na = {1, 2, 3, 4} Clang 10.0.0
  // na = {4, 3, 2, 1} MSVC 2019 16.7 Preview 6.0
}
```

　初期化式とはこの様に初期化子として指定されている式の事を言います。その評価順序は、同じ変数に対して変更を加えるような式が一つの初期化構文内に複数あるとき問題になります。`()`だとそれぞれの式の実行順序が決まっていませんが、`{}`の場合はその初期化子の順番通りに左から右へ実行しつつ初期化すると明確に決まっています。

### 指示付初期化（*Designated Initialization*）

　指示付初期化とは、集成体初期化時にその集成体のメンバ名を指定する形で初期化する構文の事です。C言語ではC99から可能でしたが、C++にはC++20から制限付きで導入されました。

```cpp
struct point3d {
  double X, Y, Z;
};

int main() {
  // この二つの初期化構文は同じ意味
  point3d vec1 = { .X = 2.0, .Y = 3.0, .Z = 1.0};
  point3d vec2 = { 2.0, 3.0, 1.0};
}
```

　このような名前指定（`.X`や`.Y`）の事を指示子と呼びます。指示子を指定できること以外は普通の集成体初期化と同じ効果（初期化式の評価準や縮小変換の禁止など）を持ちます。

　ただしいくつか制約があり、ほとんど普通の集成体初期化でメンバ名を指定できるようになったくらいのことしかできません。

1. 指示子の順番はメンバの宣言順
2. 指示子を書くならば全てに指示子による初期化が必要
      - 指示子のあるなしが混在できない
3. 指示子はネストできない

```cpp
struct nest {
  int n;
  point3d p;
  int m;
};

int main() {
  // 1. 指示子の順番はメンバの順番通りでなければならない
  point3d v1 = { .X = 1.0, .Z = 2.0, .Y = 1.0}; // エラー
  point3d v2 = { .X = 1.0, .Y = 1.0, .Z = 2.0}; // OK

  // 2. 指示子はあるかないかのどちらか
  point3d v3 = { 1.0, .Y = 2.0, 3.0};       // エラー
  point3d v4 = { .X = 1.0, .Y = 2.0, 3.0};  // エラー

  // 3. ネストした集成体に対しては{}をネストさせる
  nest n1{ .n = 1, .p.X = 1.0, .p.Y = 1.0, .p.Z = 1.0, .m = 2};    // エラー
  nest n2{ .n = 1, .p = { .X = 1.0, .Y = 1.0, .Z = 1.0 }, .m = 2}; // OK
}
```

　この3番目の例のように指示子による初期化では`{}`初期化を使用できます。また、順番が入れ替わらない限りは指示子を省略することもでき、その場合はデフォルトメンバ初期化があればそれによって、無ければ`{}`を指定したかの様に初期化されます。

```cpp
int main() {
  // {}を使った初期化ができる
  point3d v1 = { .X = {1.0}, .Y{1.0}, .Z{} };

  // 省略した要素は{}（この場合は0.0）で初期化される
  point3d v2 = { .X = 1.0, .Y = 1.0 };
  point3d v3 = { .X = 1.0 };

  // 順番通りであれば要素の初期化子は省略できる
  point3d v4 = { .X = 1.0, .Z = 1.0 };
  point3d v5 = { .Y = 1.0 };
}
```

　このような指示付初期化は初期化するメンバ名を明確にすることによって初期化指定の間違いを発見しやすくする効果があり（間違えるとコンパイルエラーになる）、大量のメンバを持つ構造体の初期化を読みやすくできたりします。他にも、共用体の初期化時に初期化するメンバを指定して初期化できたり、応用すると簡単な名前付き引数を実現できたりします。

```cpp
union U {
  int n;
  double d;
};

int main() {
  U u1 = {10};          // nを10で初期化
  U u2 = { .n = 10 };   // 同上
  U u3 = { .d = 3.14 }; // dを3.14で初期化

  U u4 = {3.14};                // エラー、縮小変換
  U u5 = { .n = 1, .d = 2.72 }; // エラー、初期化子が多い
}
```

### 丸かっこによる集成体初期化

　ここまでのように集成体初期化は`{}`によって行われるものでしたが、C++20より`()`でも行えるようになります。ただし、変数名に直接続く形しか許可されず、間に`=`が入る形は許可されません。

```cpp
struct aggregate {
  int n;
  double m = 2.72;
  std::string str;
};

int main() {
  aggregate a1{10, 2.72, "braces"};
  aggregate a2(10, 2.72, "parentheses");

  // これはだめ
  aggregate a3 = (10, 2.72, "parentheses");
}
```

　ただし、いくつか`{}`による集成体初期化と異なるところがあります。

1. 縮小変換が許可される
2. ネストする波かっこが省略できない
3. ネストする場合に丸かっこを使えない
4. 指示付初期化できない
5. 空の`()`を使えない

```cpp
struct narrow {
  int n;
  float f;
};

struct wrap {
  narrow n;
  int m;
};


int main() {
  unsigned int n = 10;
  long double d = 1.0;

  // 1. 縮小変換の許可
  narrow n1{n, d}; // 2つともエラー
  narrow n2(n, d); // OK

  // 2. 波かっこ省略できない
  wrap w1{10, 1.0f, 20};    // OK
  wrap w2(10, 1.0f, 20);    // エラー
  wrap w3({10, 1.0f}, 20);  // OK

  // 3. ネストして丸かっこを使えない
  wrap w4((10, 1.0f), 20);  // エラー

  // 4. 指示付初期化できない
  narrow n3(.n = 10, .f = 1.0f);        // エラー
  wrap w5({ .n = 10, .f = 1.0f}, 20);   // OK

  // 5. 空にすると関数宣言とあいまいになる
  narrow n4{};  // OK、変数宣言
  narrow n5();  // エラーにはならないが関数宣言として処理されている
}
```

　これらの制約の大部分は`()`による集成体初期化がコンストラクタ呼び出しの意味論を踏襲するように設計されていることから来ています。

　このような`()`による集成体初期化は集成体をより通常のクラスに近づけるもので、主に、標準コンテナなどが備える`emplace`関数で集成体を初期化できなかった問題を解決するものです。

```cpp
int main() {
  std::vector<aggregate> v{};

  v.emplace_back(10, 3.14, "emplace");            // C++17まではエラー
  v.emplace_back(aggregate{10, 3.14, "emplace"}); // OK、ムーブ構築
}
```

　`emplace`系関数はコンストラクタの引数を受け取ってその内部でコンストラクタを呼び出して構築を行うもので、（コンテナ等に対する）直接構築などとも呼ばれます。その実装は共通して、内部で構築する領域へ`placement new`することで構築します。

```cpp
// 単純なemplace関数の実装例
template<typename... Args>
T& emplace(Args&&... args) {
  // 構築したい領域をstorageとする
  // ここで()を使っているために集成体初期化が行われない
  T* p = new(&storage) T(std::forward<Args>(args)...);

  // こうすると集成体初期化は行われるが、多くのクラス型で問題が起こる
  T* p = new(&storage) T{std::forward<Args>(args)...};

  return *p;
}
```

　ここでの`new`による構築時に`T()`としていることによって、`T`が集成体である場合に集成体初期化が行われません（コピー/ムーブコンストラクタは呼ばれる）。そこを`{}`に変えてしまえば集成体初期化は行われますが、`{}`と`()`の性質の違いから後方互換性を破壊してしまいます。`()`による集成体初期化はこのような場合にわざわざ集成体にコンストラクタを書いたり、ムーブ構築に切り替えたりしなくてもよくするために導入されました。

　このようなものにはほかにも、`std::make_shared()`や`std::make_unique()`、`std::make_from_tuple()`などがあります。

　実際のところ、`()`による集成体初期化が必要となるのはこのようなライブラリの深部においてだけであり、普通に使う分には常に`{}`を使うべきです。

### 集成体初期化と一様初期化（*Uniform Initialization*）

　C++11から一様初期化構文が導入され、非集成体のクラスの変数でも`{}`によってコンストラクを呼び出して初期化できるようになりました。この一様初期化においても縮小変換の禁止や初期化式評価順序の規定などの集成体初期化の性質を利用することができます。ただ、指示付初期化は行えません。

　一様初期化は配列や集成体に対して可能だった`{}`による初期化をクラス型に対して拡張することで、初期化構文を統一することを目指したものでした。しかし、縮小変換が禁止されていることによって`{}`を使うと謎のエラーが発生したり、`std::initializer_list`をとるコンストラクタを持つ型に対する`{}`の初期化は最優先で`std::initializer_list`をとるコンストラクタを呼び出してしまうなどの絶妙に使いづらくわかりづらいところがあり、実際には`{}`による初期化構文の統一は果たせてはいません。

　`std::initializer_list`の扱いを除けば、集成体とクラス型の間で一貫した初期化を行うことができ、初期化式の評価順序の保証や縮小変換禁止によってより安全な初期化を行うことができます。また、空のコンストラクタを明示的に呼ぶときに関数宣言と曖昧になることもありません。そのため、C++ Core Guidlineでは殆ど常に`{}`によって初期化を行うことを推奨しています。

　なお、一様初期化構文において問題となることは集成体初期化では問題にならないので集成体初期化で`{}`を嫌う理由はありません。

\clearpage

# 特殊メンバ関数

　集成体ではコンストラクタ宣言を行えないので、クラスの特殊メンバ関数はコンパイラによって自動的に定義され使用可能になります。

## デフォルトコンストラクタ

　空のリスト（`{}`）が実質的にデフォルトコンストラクタと同じ働きをします。空のリストで初期化すると、集成体のメンバのうちデフォルトメンバ初期化されているものはそれによって初期化され、基底クラスおよび残りのメンバは再帰的に空のリスト（`{}`）で初期化されます（すなわち、集成体は集成体初期化され、デフォルトコンストラクタを持つクラス型はデフォルトコンストラクタが呼ばれ、そうでないものは値初期されます）。

　従って、メンバの中にデフォルトメンバ初期化されておらずデフォルトコンストラクタを持たないような型があると、空のリストによる初期化はコンパイルエラーとなります。

　


```cpp
struct not_def_init {
  int n = 0;
  not_def_init() = delete;
  not_def_init(int an) : n(an) {}
};

struct A1 {
  int n = 10;
  double d;
};

struct A2 {
  int n;
  not_def_init ni;  // デフォルトコンストラクタを持たない
};

int main() {
  A1 a1 = {}; // n == 10, d == 0.0
  A2 a2 = {}; // エラー
}
```

## コピー/ムーブコンストラクタ

　コピー/ムーブコンストラクタは通常のクラス型で一切宣言しない時と同様に利用可能になります。すなわち、メンバ毎にその型のコピー/ムーブコンストラクタを呼び出していく実装がなされます。従って、メンバの中にムーブコンストラクタを持たない型がある場合はその集成体のムーブコンストラクタは暗黙に`delete`され、コピーコンストラクタを持たない型がある場合は同様にコピーコンストラクタは暗黙に`delete`されます。

　C++17までは`default/delete`宣言によってある程度これを制御できたのですが、現在はあらゆるコンストラクタの宣言が禁止されているため明示的に制御する方法はありません。

```cpp
struct not_move {
  not_move() = default;
  not_move(const not_move&) = delete;
};

struct A1 {
  std::unique_ptr<int> up;  // コピー不可
};

struct A2 {
  not_move nc;  // コピー・ムーブ不可
};

int main() {
  A1 a1 = {};
  A2 a2 = {};

  A1 c1 = {a1};             // エラー
  A1 m1 = {std::move(a1)};  // OK

  A2 c2 = {a2};             // エラー
  A2 m2 = {std::move(a2)};  // エラー
}
```

## コピー/ムーブ代入演算子

　コピー/ムーブ代入演算子はコピー/ムーブコンストラクタと同様に利用可能になります。こちらは宣言してあっても集成体となるかどうかに一切影響を与えないので、`default/delete`宣言を明示的に行うことができ、自前で実装することもできます。

（サンプルコードは先程のコピー/ムーブコンストラクタのものとほぼ同じなので省略します）

\clearpage

# 集成体と参照型メンバ

　集成体のメンバ変数に制約はなく、どんな型でもメンバとなることができます。`const/volatile`、ポインタ型、そして参照型もです。集成体で参照型をメンバとする場合、非集成体と比較してメリットが1つあります。

　まず、参照メンバをもつ普通のクラス型を書いてみます。

```cpp
class cont_ref {
  int& m_ref;

public:
  cont_ref(int n) // ??
   : m_ref(n)
  {}

  operator int() const {
    return m_ref;
  }
};

int main() {
  int n = 10;

  cont_ref cr{n};

  n = 20;

  std::cout << int(cr); // Undefined Behavior・・・
}
```

　はい、普通です。一箇所致命的なミスをしている点を除けば。

　コンストラクタ引数が参照型になっていません。こうなっていると、コンストラクタ引数のローカル変数でクラスメンバの参照を初期化することになるので、コンストラクタ呼び出しの終了後にメンバの参照はダングリング参照となりそのアクセスは未定義動作となります。たった一文字忘れただけなのに・・・。もちろん、局所的に見ればこれは何ら問題の無いコードなのでコンパイルエラーにはなりません（clangだけは一応警告してくれるようです）。

　上記クラスのコンストラクタは正しくは次のように書かなければなりません。

```cpp
class cont_ref {
  int& m_ref;

public:
  cont_ref(int& n)  // &の1文字が重要！
   : m_ref(n)
  {}
};
```

　これによって意図通りにコンストラクタに渡された別の変数の参照をメンバとして保持できるようになります。ではこれを集成体で書くとどうなるでしょうか？

```cpp
struct cont_ref {
  int& m_ref;
};

int main() {
  int n = 10;

  cont_ref cr = {n};

  n = 20;

  std::cout << cr.m_ref; // OK、20
}
```

　コンストラクタを書く必要がないのでとてもすっきりします。そして、間違えようがありません。この様に、参照型メンバを扱う際は集成体にすると省コードかつ安全に書くことができます。

　ちなみに、参照型メンバを持つ時はデフォルト構築とあらゆる代入操作は出来ません。コピーとムーブコンストラクタは他のメンバが利用可能であれば利用可能となります。ただ、この振る舞いはクラス型の場合もほぼ同様です。

```cpp
int main() {
  cont_ref cr1{}; // エラー、参照型メンバは必ず初期化されなければならない

  int n = 10;

  cont_ref cr2{n};
  cont_ref cr3{cr2};  // OK、コピー（ムーブ）コンストラクタは利用可能

  cr3 = cr2;  // エラー、コピー（ムーブ）代入演算子は利用不可
}
```

　このことはクラス型で参照型メンバを持つ時にコンストラクや代入演算子を実装することを考えてみると分かりやすいです。初期化後の参照型変数に対するあらゆる操作は参照先の変数に対する操作になるため、参照そのものをコピーしたりできずコピー/ムーブ代入演算子を実装することが出来ません。一方、コンストラクタの場合はコンストラクタ初期化子リストでメンバ参照の初期化が行えるため、コピー/ムーブコンストラクタは実装することができます。

## 一時オブジェクトの寿命延長

　集成体で参照型メンバを持ち集成体初期化を行う時、初期化子の一時オブジェクトの寿命を延長させることができます。とはいっても、参照型とはこの場合`const`左辺値参照と右辺値参照メンバが対象です。

```cpp
struct lifetime_extender {
  std::string&& r_ref;
  const std::string& cl_ref;
};

int main() {
  lifetime_extender le = {std::string{}, "rvalue"}; // OK

  // どちらの参照先も生存期間内であり、安全に参照できる
  std::cout << le.r_ref << '\n'
            << le.cl_ref << '\n';
}
```

　この`le`の初期化式にある二つの`std::string`オブジェクトは右辺値であり、本来その寿命はその一連の式の終端、すなわち最初のセミコロン`;`までです。しかし、右辺値参照型（`std::string&&`）と`const`左辺値参照型（`const std::string&`）のメンバで受けているため、それらの参照に束縛されることでその寿命が延長され初期化後も安全に参照することができます。

　このことは普通に変数宣言をして初期化した時の規則と同じです。むしろ、同じことが集成体初期化という仮想的なコンストラクタを通しても起こるという事です。

```cpp
int main() {
  // 共にrvalueの寿命が延長される
  std::string&& r_ref = std::string{};
  const std::string& = "rvalue";
}
```

　上記の右辺値は、正確には値カテゴリという規格用語で*prvalue*と呼ばれるカテゴリの状態にあります。*pravlue*の右辺値が右辺値参照あるいは`const`左辺値参照を初期化したとき、その*prvalue*なオブジェクトを実体化した*lvalue*が参照に束縛され、そのオブジェクトの寿命が延長されます（一時オブジェクトから参照へ所有権が移行する）。

　ところで、右辺値にはもう一つ*xvalue*というカテゴリがあります。*xvalue*は`std::move`によって右辺値参照にキャストされた左辺値（*lvalue*）の事です。右辺値参照や`const`左辺値参照は*prvalue*と同様に*xvalue*を束縛できるので、*xvalue*で上記のような参照を初期化してもコンパイルは通ります。しかし、*xvalue*では*prvalue*の時に起こったような実体化が発生せず、所有権が移行しないので寿命は元のオブジェクト次第となります。

```cpp
int main() {
  // prvalueからの初期化
  lifetime_extender le = {std::string{}, "rvalue"};

  // lvalueオブジェクトを動的構築
  std::string* str1 = new std::string{};
  std::string* str2 = new std::string{"lvalue"};
    
  // xvalueから初期化
  lifetime_extender le2 = {std::move(*str1), std::move(*str2)};

  // 元のオブジェクトを殺す
  delete str1;
  delete str2;

  // Undefined Behavior...
  std::cout << le2.r_ref << '\n'
            << le2.cl_ref << '\n';
}
```

　これはかなり恣意的なコードですが、この様なことがライブラリの中に隠れているときなどに思わぬ罠を踏むかもしれません。集成体の参照メンバで寿命延長を狙う時は*prvalue*を意識しましょう。

　集成体の右辺値参照メンバというのはまず使わないでしょうが`const`左辺値参照メンバはたまに使うかもしれません。そして、その参照を*prvalue*で初期化するとしたらおそらく関数の戻り値で直接初期化する時ではないかと思います。

```cpp
// prvalueを返す関数
std::string f();

// xvalueを返す関数
std::string&& g();

struct str_cref {
  const std::string& str;
};

int main() {
  str_cref sc1 = {f()}; // OK、安全
  str_cref sc2 = {g()}; // OK、もしかしたら・・・
}
```

　この様な場合にこれらのこまごました事を頭の片隅に置いておくと何か役に立たないでもないかもしれません（というか*xvalue*を返す関数なんて作るべきではありませんが・・・）。

### `()`・・・

　そして、`()`による集成体初期化ではその様な*prvalue*の寿命延長が行われません。これは`()`による初期化がコンストラクタ呼び出しに近くなるようにされているためだと思われます。

```cpp
struct str_cref {
  const std::string& str;
};

int main() {
  str_cref sc1{"braces"};       // OK、安全
  str_cref sc2("parentheses");  // 未定義動作、ﾖｼ！
}
```

　こういう事もあるのでやはり集成体初期化には`{}`を常に使いましょう。`()`はライブラリの深部でクラス型と集成体型を統一的に扱うために存在するものであって、普段使いするものではありません。

　この`{}`と`()`の差異やここまで見てきた集成体初期化を見ていると分かるかもしれませんが、集成体初期化は集成体のメンバをほぼ直接初期化します。そのため、各メンバを普通の変数として並べてそれぞれ初期化したときと殆ど同じように初期化されます。一方、コンストラクタ呼び出しによる初期化はコンストラクタを通してそのメンバを初期化します。そのため、集成体初期化と比べると何かある種の壁を通して初期化しているような雰囲気で、普通の変数の初期化とは少し意味論が異なります。

\clearpage

# `std::array`

　ここでは、集成体の花形？たる`std::array`の実装を見てみます。とはいっても集成体なので難しいことは無く、次のようにほとんどそのまま実装されます。

```cpp
template<typename T, std::size_t N>
struct array {
  T[N] m_array;

  using value_type = T;
  using reference = T&;
  using const_reference = const T&;
  using pointer = T*;
  using const_pointer = const T*;
  using difference_type = std::ptrdiff_t;
  using size_type = std::size_t;
  using iterator = T*;
  using const_iterator = const T*;
  using reverse_iterator = std::reverse_iterator<iterator>;
  using const_reverse_iterator = std::reverse_iterator<const_iterator>;
};
```

　`std::array`の実装はこれでほとんど終わりです。後は愚直にメンバ関数を実装するだけ。なお、コピー・ムーブコンストラクタ/代入演算子は何もせずとも暗黙の`default`宣言がなされています（型`T`次第では`delete`されますが）。

　これをベースにすると、添え字演算子の実装は例えば次のようになります。

```cpp
template<typename T, std::size_t N>
struct array {
  T[N] m_array;

  constexpr reference operator[](size_type i) {
    return m_array[i];
  }

  constexpr const_reference operator[](size_type i) const {
    return m_array[i];
  }

  constexpr reference at(size_type i) {
    if (N <= i) throw std::out_of_range{};
    return m_array[i];
  }

  constexpr const_reference at(size_type i) const {
    if (N <= i) throw std::out_of_range{};
    return m_array[i];
  }
};
```

イテレータインターフェース（`begin()/end()`）は

```cpp
template<typename T, std::size_t N>
struct array {
  T[N] m_array;

  constexpr iterator begin() noexcept {
    return m_array;
  }

  constexpr const_iterator begin() const noexcept {
    return m_array;
  }

  constexpr iterator end() noexcept {
    return m_array + N;
  }

  constexpr const_iterator end() const noexcept {
    return m_array + N;
  }
};
```

　こんな感じで、`std::array`は単に生配列をラップする形で実装されています。

\clearpage

# 集成体進化の歴史

## C++98/03それ以前

　このあたりの仕様は仕様書が公開されていなかったりと不明なところが多いですが、ほとんどC言語の構造体そのままだったはずで、集成体初期化は最初から可能です。

　このころの集成体の仕様は次のようでした。そしてこれに沿っておけば全てのバージョンのC++で集成体となることができます。

- ユーザー宣言のコンストラクタを持たない
- 非`public`なメンバ変数を持たない
- 仮想関数を持たない
- 継承していない

　継承が禁止されている以外はC++20とほぼ変わりありません。C++11～C++20で集成体の要件が色々変わりましたが、結局は継承を許可しつつここに戻ってくることになります。

## C++11

- 集成体の要件
    - ユーザー提供のコンストラクタを持たない
- 言語機能
    - 縮小変換の禁止
    - 初期化式評価順序の規定

### 集成体の要件

　C++11での集成体の要件は次のようでした。

- ~~ユーザー宣言のコンストラクタを持たない~~
- \textcolor[rgb]{1,0.157,0}{ユーザー提供のコンストラクタを持たない}
- 非`public`なメンバ変数を持たない
- 仮想関数を持たない
- 継承していない
- \textcolor[rgb]{1,0.157,0}{デフォルトメンバ初期化されていない}

　クラスメンバのデフォルトメンバ初期化はC++11から導入されたのですが、おそらく議論が間に合わなかったためにC++11の集成体ではとりあえず禁止とされました。

#### ユーザー提供のコンストラクタを持たない

　

　「ユーザー宣言のコンストラクタを持たない」とはあらゆるコンストラクタを宣言してはいけないという意味ですが、「ユーザー提供のコンストラクタを持たない」とはユーザーによって定義されたコンストラクタを持たないという意味です。そして、ユーザー提供ではないコンストラクタとはC++11から導入されたコンストラクタの`default/delete`指定によって宣言されたコンストラクタの事です。

　つまり、（C++11の）集成体においては`default/delete`されたコンストラクタはあってもいいという事です。

```cpp
struct A {
  int n;
  A() = default;
  A(A&&) = delete;
};

int main() {
  A a = {1}; // OK、集成体初期化
}
```

### 縮小変換の禁止/初期化式評価順序の規定

　ちょっと前で`{}`による初期化時には縮小変換が禁止され、初期化式の評価順序が規定されていると説明しましたが、これは実はC++11から導入されました。どうやら一様初期化構文と共にもたらされたもののようです。

## C++14

- 集成体の要件
    - デフォルトメンバ初期化の許可
- 言語機能
    - ネストした波かっこ省略の許可

### 集成体の要件

　C++14での集成体の要件は次のようでした。

- ユーザー提供のコンストラクタを持たない
- 非`public`なメンバ変数を持たない
- 仮想関数を持たない
- 継承していない
- ~~デフォルトメンバ初期化されていない~~

　C++11から遅れること3年、デフォルトメンバ初期化が許可されました。これは特に疑問点の無い改善でしょう。

```cpp
struct A {
  int n = 10;
};

int main() {
  A a1 = {1}; // OK、n == 1
  A a2 = {};  // OK、n == 10
}
```

### ネストした波かっこ省略の許可

　これは配列の初期化時の集成体初期化ではネストする`{}`を省略できていたのをクラス型の集成体まで広げたものです。

```cpp
int main() {
  // 配列は以前からどちらもok
  int arr1[2][2] = {{0, 1}, {2, 3}};
  int arr2[2][2] = { 0, 1 ,  2, 3 };

  std::array<int, 4> arr3 = {{0, 1, 2, 3}}; // C++11では省略不可
  std::array<int, 4> arr4 = { 0, 1, 2, 3 }; // C++14から省略可
}
```

　`std::array`は内部に配列メンバを一つだけ持つ集成体で、C++11ではその集成体初期化時にはこのように`{}`を2つネストさせなければなりませんでした。外側の`{}`は集成体そのもの（`std::array`）の初期化で、内部の`{}`はそのメンバの集成体（この場合は`std::array`の内部配列）の初期化のためのものです。一方配列では多次元配列の初期化時に全てのかっこを取り払って一番外側のかっこだけにすることができます。C++14では多次元配列で許可されていたこの挙動を集成体全般に広げました。

## C++17

- 集成体の要件
    - 公開継承の許可
        - それに伴って、基底クラスを含めた集成体初期化の許可
    - 一部のコンストラクタ宣言の禁止
- 言語機能
    - 構造化束縛
    - クラステンプレートのテンプレート引数推論
      - ただし、推論補助が必要
- ライブラリ機能
    - `std::is_aggregate<T>`

### 集成体の要件

　C++17での集成体の要件は次のようでした。

- ~~ユーザー提供のコンストラクタを持たない~~
- ユーザー提供のコンストラクタ、\textcolor[rgb]{1,0.157,0}{`explicit`コンストラクタ、継承コンストラクタ}を持たない
- 非`public`なメンバ変数を持たない
- 仮想関数を持たない
- \textcolor[rgb]{1,0.157,0}{`public`以外の継承をしていない}

#### `explicit`コンストラクタを持たない

　

　コンストラクタの`explicit`指定は暗黙変換を禁止するためのものですが、`default/delete`指定と合わせて指定することができました。集成体でそれをすると、集成体でありながら集成体初期化できない謎の型を生み出すことができてしまいます。

```cpp
// これは集成体だけど・・・
struct A {
  int n;
  explicit A() = default;
};

int main() {
  A a1 = {1}; // NG
  A a2{1};    // NG
}
```

　`explicit`コンストラクタの禁止はこれを防止するための要件です。

#### `public`以外の継承をしていない、継承コンストラクタを持たない

　

　`public`継承に限って継承が許可されることとなりました。`public`継承だけならメンバとして持つのとほぼ同義であり集成体初期化を妨げるものではないので許可されたようです。

　その際、メンバがそうであるように基底クラスが集成体でなくても構わず、継承コンストラクタがあるとコンストラクタが定義されているのと同義なため継承コンストラクタを禁止しています。

```cpp
class B {
  int n;
public:
  B(int m) :n(m) {}
};

// 集成体
struct A1 : public B {
  int n2;
};

// 集成体じゃない
struct A2 : public B {
  int n2;
  using B::B;
};

int main() {
  A1 a1 = {1, 2}; // OK
  A2 a2 = {1, 2}; // NG
}
```

### クラステンプレートのテンプレート引数推論

　クラステンプレートのテンプレート引数推論はクラステンプレート初期化時の引数の型からそのクラステンプレートのテンプレートパラメータを推定するものです。

```cpp
template<typename T>
struct vec3 {
  T v1, v2, v3;
};

int main() {
  // std::vector<int>
  std::vector vec = {1, 2, 3, 4, 5};

  // std::pair<int, double>
  std::pair p = {1, 1.0};

  // vec3<double>ってなってほしいが、エラー
  vec3 v3 = {1.0, 2.0, 1.0};
}
```

　この様に、主としてコンストラクタの引数からそのテンプレートパラメータを補うことができ、めちゃくちゃ便利な神アプデです。ところが、この`vec3`のような型ではテンプレートパラメータが無いぞ！とエラーが出ます。残念なことに集成体初期化ではこの機能を使えません・・・

　そこで、同時に導入された推論補助（推論ガイド：*deduction guide*）を書いてやります。推論補助は初期化時の引数から素直にテンプレートパラメータを推定できない場合に、どのようにテンプレートパラメータを導出するかをコンパイラに教えてあげるものです。

```cpp
template<typename T>
struct vec3 {
  T v1, v2, v3;
};

// 推論補助
template<typename T>
vec3(T, T, T) -> vec3<T>;


int main() {
  // vec3<double>
  vec3 v3 = {1.0, 2.0, 1.0};
  // vec3<int>
  vec3 vi3 = {1, 2, 3};
}
```

　しかし、非集成体クラステンプレートならばこの推論補助と同等な推論を自動で行うので、本来必要のないはずのものです。

　ちなみに、この推論補助は`auto`の無い後置戻り値の関数宣言構文なので、関数宣言で出来ることは大体できます。かなり強力です。

###  `std::is_aggregate<T>`

　`std::is_aggregate<T>`は型`T`が集成体かどうかを判定するメタ関数で、おそらく実装はコンパイラマジックによって行われます。

```cpp
struct A {
  int n;
};

int main() {
  std::is_aggregate_v<std::vector<int>>;  // false
  std::is_aggregate_v<A>;  // true
}
```

　これと可変長テンプレートを使って集成体型のメンバの数を求める謎のメタ関数を作ることができたりします。

## C++20

- 集成体の要件
    - あらゆるコンストラクタ宣言の禁止
- 言語機能
    - 指示付初期化
    - `()`による集成体初期化
    - クラステンプレートのテンプレート引数推論への適合（推論補助なしで推論する）
    - 一貫比較

### 集成体の要件

　C++20での集成体の要件は次のようです。

- ~~ユーザー提供のコンストラクタ、`explicit`コンストラクタ、継承コンストラクタを持たない~~
- \textcolor[rgb]{1,0.157,0}{ユーザー宣言のコンストラクタ、継承コンストラクタを持たない}
- 非`public`なメンバ変数を持たない
- 仮想関数を持たない
- `public`以外の継承をしていない

　こうして、継承が可能になったこと以外は結局C++03までの要件に回帰しました。

#### ユーザー宣言のコンストラクタを持たない

　

　結局コンストラクタ宣言は禁止されたわけですが、`default/delete`ならば問題はなさそうに思えます。一体何がダメだったのでしょうか・・・？

　まず1つは、非集成体型においてデフォルト構築を禁止するためにコンストラクタの`delete`宣言を行っている場合に、同時に集成体の要件を満たすがために集成体初期化が可能になってしまうという問題がありました。

```cpp
struct delete_defctor {
  delete_defctor() = delete;
};

int main() {
  delete_defctor x;   // エラー、デフォルトコンストラクタは削除されている
  delete_defctor x{}; // OK、集成体初期化！？
}
```

　デフォルト構築を禁止しているのに集成体であることによって空のリストからの初期化が可能となってしまいます。また、この様なクラスがメンバを持っていると同じ理由から問題が起きます。

```cpp
struct delete_init_int {
  delete_defctor() = delete;
  delete_defctor(int) = delete; // intから構築してほしくない

  int n = 10;
};

int main() {
  delete_init_int x(3); // エラー、コンストラクタは削除されている
  delete_init_int x{3}; // OK、集成体初期化！？
}
```

　これがなぜ起きるのか、起こさないようにするにはどうすればいいのか？は中々難しい話であり、多くのC++ユーザーは別に知る必要もないお話になりがちです（ここまで読んでいれば分かるとは思います）。そんな些末な仕様の隅っこを多くのC++ユーザーが知る必要が無いようにする、というのが動機の一つです。また、この問題は丸かっこ集成体初期化を考慮するとより複雑になります・・・

　そしてもう一つは、`default/delete`宣言の位置によって集成体であるか無いかが変化してしまっていた問題です。

　コンストラクタ（というかメンバ関数全般）はその定義をクラス外で行うことができます。そして、`default/delete`宣言もクラス外で行うことができます。

```cpp
// C++17までは集成体
struct aggregate {
  aggregate() = default;

  int n;
};

// 集成体ではない
struct not_aggregate {
  not_aggregate();

  int n;
};

not_aggregate::not_aggregate() = default;


int main() {
  aggregate x{10};      // OK
  not_aggregate y{10};  // NG
}
```

　この2つのクラスは名前以外は同じ意味のクラスとなる筈ですが、集成体という観点からは同じにはなりません。クラス外のコンストラクタ定義は別の翻訳単位で行われている可能性もあるため、このようなケースを特別扱いすることも難しいです。

　主にこれらの問題から、集成体におけるコンストラクタの宣言は禁止されることになりました。これは破壊的変更であり、C++11～17の間に書かれたコンストラクタ宣言を持つ集成体はC++20以降集成体ではなくなることになります。

### `()`による集成体初期化と集成体型へのキャスト

　丸かっこによる集成体初期化が許可された流れで、`static_cast`による集成体型への明示的変換が許可されました。

　キャストに与えられた式から変換先の集成体の最初の要素への暗黙変換が可能であるときに、集成体型へのキャストが有効になります。

```cpp
struct Agg {
  int n;
  double v = 1.0;
};

int main() {
  auto a1 = static_cast<Agg>(10); // ok
  auto a2 = (Agg)10;  // OK、C形式キャスト
  auto a3 = Agg(10);  // OK、関数形式キャスト
}
```

　C形式キャストと関数形式キャストは可能なら`static_cast`によって処理されるので、それらでも集成体への明示的変換ができるようになります。

　これらキャストでは1つの式しか指定できないので、集成体の最初の要素以外は初期化されません。また、これらは集成体初期化ではなく明示的な変換なので縮小変換を含めたあらゆる暗黙変換が考慮されます。

### 集成体のテンプレート引数推論

　C++17では推論補助が無いとできなかった集成体に対するテンプレート引数推論が推論補助無しでできるようになります。

```cpp
template<typename T>
struct vec3 {
  T v1, v2, v3;
};

int main() {
  // vec3<double>
  vec3 v3 = {1.0, 2.0, 1.0};  // OK
  // vec3<int>
  vec3 vi3 = {1, 2, 3};       // OK
}
```

　推論補助の仕組みは、初期化子からマッチするコンストラクタと推論補助を関数テンプレートとして抽出しオーバーロード解決によって1つを選んだうえで、コンストラクタの場合はそのテンプレートパラメータ、推論補助の場合はその戻り値型をテンプレート引数として補うものです。C++17までは集成体初期化がそこでは考慮されていなかったため推論補助が必要となっていました。

　C++20からは、集成体`C`の初期化時の引数リスト`{x1, ..., xi}`について、対応する`C`の要素`ei`が過不足なくぴったりと存在している場合に、`C`の要素`ei`の型`Ti`によって`C(T1, ..., Ti)`という仮想的なコンストラクタを考慮して同じ手順でテンプレート引数推論を行うようになります。

　この時、その集成体のテンプレート引数にメンバとなっているクラステンプレートが依存していると`{}`省略が出来なくなります。

```cpp
template <typename T>
struct S {
  T x;
  T y;
};

template <typename T>
struct C {
  S<T> s; // テンプレートパラメータTに依存している
  T t;
};

C c1 = {1, 2};        // error
C c2 = {1, 2, 3};     // error、{}省略できない
C c3 = {{1u, 2u}, 3}; // ok, C<int>

template <typename T>
struct D { 
  S<int> s; // テンプレートな集成体だが、型は確定している
  T t; 
};

D d1 = {1, 2};    // error、{}省略するなら初期化子は3つ必要
D d2 = {1, 2, 3}; // ok、{}省略可能
```

### 一貫比較（*Consistent Comparison*）

　一貫比較とは`<=>`（宇宙船演算子）と`==`の2つの演算子から残りの全ての比較演算子を導出する仕組みの事です。その比較が基底クラスとメンバを宣言順に比較するもの（辞書式順序による比較、構造的な比較）であるならば、`default`指定によってコンパイラ任せにすることができます。  

　集成体は多くの場合いくつかのデータをまとめた名前付きタプルの様な使われ方をするので、その比較も構造的な比較で十分な場合が多いです。

```cpp
struct compareble {
  int n;
  double v;
  char str[5];  // 配列メンバは展開されて要素ごとの比較になる

  // この2つの演算子から残りの5つの演算子が導出される
  auto operator<=>(const compareble&) const = default;
  auto operator==(const compareble&) const = default;
};

int main () {
  compareble c1 = {1, 3.14, "abcd"};
  compareble c2 = {1, 3.14, "bcde"};

  c1 <  c2; // true
  c1 <= c2; // true
  c1 >  c2; // false
  c1 >= c2; // false
  c1 == c2; // false
  c1 != c2; // true
}
```

　一貫比較仕様による比較演算子自動生成によって、集成体のコードをさらに削減することができるようになります。


\clearpage

# 謝辞

　本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppmap(https://cppmap.github.io/ : ライセンスはCC0 パブリックドメイン)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Designated Initialization @ C++ - yohhoyの日記  
  https://yohhoy.hatenadiary.jp/entry/20170820/p1
- aggregateと初期化リストの不思議 - 本の虫  
  https://cpplover.blogspot.com/2010/09/aggregate.html
- C++11 Universal Initialization は、いつでも使うべきなのか - Qita  
  https://qiita.com/h2suzuki/items/d033679afde821d04af8
- Why does std::tuple break small-size struct call convention optimization in C++? - stackoverflow  
  https://stackoverflow.com/a/63723752
