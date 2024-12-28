---
title: "[Go]deep-clone method generatorを実装する"
emoji: "🖇️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## deep-clone method generatorを実装する

こんにちは

この記事では`Go`のpackage pattern(e.g. `./...`)を受けとって見つかった各型に`deep clone` methodを生成するcode generatorの実装に際して気を付けどころや方法などについて述べます。

`deep clone`とはここでは、ある値`a`とそれの`deep clone`である`b`があるとき、`a`か`b`一方へのデータアクセスが(そうであることを意図しない限り)もう一方に影響しないことをさします。

大雑把に言って以下のことを説明します。

- Rationale: どうしてcode generatorを実装するする必要があるか？
- 生成するmethodのsignatureについて(i.e. type paramをどうcloneするか)
- どういうコードを生成するかの事前想定
- Clone methodの実装方針
- code generatorの実装方針、考慮事項
  - [golang.org/x/tools/go/packages]による`ast`, `type information`の読み込み
  - 生成対象の型依存関係のグラフ化、グラフを逆にたどることで生成対象の型を列挙する。
  - 型情報を使った各型の判別
    - `Clone`を実装する型(_implementor_)か
    - `NoCopy`(assignによってコピーすると`go vet`が`copies lock value`と警告する型)か
    - assignによってcloneできる型か
  - field unwrapper: `[]map[string]*[5]T`のような型があるとき、`T`以外の部分のcloneは共通の処理を実装できるので、これをfield unwrapperと呼んで実装します。
  - など

対象読者のレベル感を会社の同僚に置くため基本的な概念の説明を多く含めようと考えています。([A Tour of Go](https://go.dev/tour/list)は最低限こなしている)
そのため`Go`やcomputer-scienceに長じた読者はある程度とばしながら読んでいただければと思います。
ただし`ast`や`type info`そのものの説明は過去の記事で何度か書いたので省きます。そこからかなり難易度が高くなってしまうかも。すみません。

## 対象環境

- `linux/amd64`
  - ただし`OS`/`arch`の差は外部ライブラリによって吸収されるので影響しないものと想定します。

検証は`go 1.23.2`、リンクとして貼るドキュメントは`1.23.4`のものになります。

```
# go version
go version go1.23.2 linux/amd64
```

各種ライブラリは以下のバージョンを用います。

```go.mod
require (
    github.com/dave/dst v0.27.3
    github.com/google/go-cmp v0.6.0
    github.com/ngicks/go-iterator-helper v0.0.18
    github.com/ngicks/und v1.0.0-alpha6
    github.com/spf13/cobra v1.8.1
    github.com/spf13/pflag v1.0.5
    golang.org/x/tools v0.28.0
    gotest.tools/v3 v3.5.1
)
```

`github.com/ngicks`から始まるgo moduleは自作のものです。地産地消です。これらはあんまり気にしないでください。

`Go 1.24`から[generic type alias](https://tip.golang.org/doc/go1.24#language)が導入されますが、この記事はこれを全く考慮していません。なんかそのままでうまく動きそうな気もしてる。

## Rationale: どうしてcode generatorを実装するする必要があるか？

### 型とassignによるコピー

`pointer`が含まれる型はassignでコピーしきれない`deep clone`に特別な方法がいるよねっていう基本的な話をします。
説明が不要な方は[deep-cloneを実現する方法](#deep-cloneを実現する方法)まで飛ばしてください。

他のプログラミング言語でもそうである通り、`Go`ではassign(`a = b`)によって値のコピーが起こります。
`int`, `string`, `bool`などのプリミティブな型は単純なassignのみで情報をコピーできるため、例えば`c := a`とした後、`c`への変更は`a`に影響しません(_and vice versa_)。
(ただし言語によっては`string`がimmutableでないことがあるので気を付ける。)

さらに、`struct`, `array`などが単にこれらのprimitiveな型のみを含む場合でもassignによってコピーが起きます。

```go
type sample struct { foo int; bar string }
s1 := sample{bar: "foo"}
s2 := s1
s2.bar = "bar"
fmt.Printf("s1.bar=%q, s2.bar=%q\n", s1.bar, s2.bar)
// s1.bar="foo", s2.bar="bar"
```

ただし、型が[pointer](https://go.dev/tour/moretypes/1)(`*T`)を含む場合は話が変わっています。
「含む」というのは下記をさします。

- (1) 型がpointerであるとき
  - `*T`
  - `slice`, `map`, `channel`, `func`, `interface`
    - これらは暗黙的にpointerを含む
- (2) (1)ないしは(1)へのtype aliasを[underlying type](https://go.dev/ref/spec#Underlying_types)とするとき
  - `type A B`とするとき、`B`が`underlying type`です
  - ただし`pointer`, `interface`を`underlying`とした型にはmethodを定義できません。
- (3) (1),(2)をstruct field, array elementに含むとき

これらの場合、単純なassignでは値の完全なコピーは行われません。
なぜなら、`pointer`がassignされた場合、そのアドレス値がコピーされるためです。
`pointer`は(リンク先の`A Tour of Go`で説明されている通り)メモリー上のある位置を指し示すアドレスの値です。いってしまえば`uint`です。
`*T`は実際上は`uint`なのだからassignによって`uint`の値がコピーされます。`uint`が指し示したさきの値をコピーしないことはこう考えると当たり前に聞こえるはず。

pointerは[dereference](https://go.dev/ref/spec#Address_operators)することでassignによるコピーが行うことができます

```go
num   /*  int */ := 10
nump  /* *int */ := &num
deref /*  int */ := *nump
num = 12
fmt.Printf("num=%d\nnump=%d\nderef=%d\n", num, *nump, deref)
// num=12
// nump=12
// deref=10
```

`Go`はmoduleやpackageによるコードの分割に対応しています。
`Go`における公開性のコントロールは、そのpackageのもっとも外側のスコープで定義されるident(=identifier、識別子、関数名とか変数名とかのこと)の先頭1文字がunicode upper caseになっているか否かで決まります。
upper caseなら公開されているのでpackage外からでもアクセスできます。逆ならアクセスできません。module local的な概念のあるモジュールシステムを備えた言語もありますが、`Go`にはありません。
これはstructのfieldに関しても同様です。
この何が問題かというと型がexportされないfieldに`pointer`を含むとき、そのfieldにアクセスして`deref`することができないのでコピーが行うことができません。

ある値へのアクセスが他の値に影響しないでほしいことはよくあります。
例えばプログラム実行時のときどきの状態を保存して比較したいときや、[data race](https://go.dev/doc/articles/race_detector)を防ぎながら同じデータを複数のgoroutineに渡して処理したいときなどです。

### deep cloneを実現する方法

`deep clone`を実現する一般的な方法について述べます。
知ってる人は[どうしてcode-generatorを実装する必要があるのか](#どうしてcode-generatorを実装する必要があるのか)まで飛ばしてください。

普通にデータを`deep clone`するには以下のような方法を用います。

- データ構造を`marshal`(serialize)して`unmarshal`(deserialize)する
- `reflect`によって動的なコピーを実装する
- code generatorによってdeep clonerを実装する

#### データ構造を`marshal`(serialize)して`unmarshal`(deserialize)する

javascriptに[structuredClone](https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone)があるので読者にはなじみのある方法なのではないでしょうか。あるいは`JSON.stringify`して`JSON.parse`することでdeep cloneを行ったことがあるかもしれません。

最初にPros(いいとこ) Cons(わるいとこ)を述べます。

- Pros:
  - 実装が容易
  - code generation不要
  - stdのみで終始する
    - サードパーティライブラリを用いる場合はそのライブラリに対する開発者による慣れの差ができやすいはずです
    - `go.mod`が何行か軽量になります。脆弱性スキャンとか`Dependabot`による頻繁なアラートが少し減るわけです。
- Cons:
  - バイナリを経由することによるパフォーマンス低下
  - `encoding/json`が`reflect`によって実装されるため、`export`されたフィールドしかコピーできない
    - `Go`の各フィールドはunicode upper caseで始まらないとパッケージ外からアクセスできません。
    - `reflect`はこのルールにのっとり`export`された`ident`(identifier)にしかアクセスしません。
  - バイナリフォーマットに変換するため、各フォーマットごとに表現力の限界がある。

表現力の限界の例は`JSON`には`pointer`に当たる機能がないというものがあります。そのため`JSON`では`ring buffer`のような循環構造を表現できません。
循環構造の表現が必須であれば`YAML`を用いるのが一つの解決方法かもしれません。`YAML`には[anchor](https://yaml.org/spec/1.2.2/#anchors-and-aliases)仕様がありますのでうまいことやれば循環グラフを表現可能です。

`Go`は`encoding/*`以下でいくつかのデータ変換パッケージを提供します。例えば以下のようなものがあります。

- [encoding/json](https://pkg.go.dev/encoding/json@go1.23.4): `JSON`と`Go`データ構造の相互変換機能
- [encoding/gob](https://pkg.go.dev/encoding/gob@go1.23.4): self-describingなバイナリと`Go`データ構造の相互変換機能

`gob`は多分`Go object`の略ですかね。`gob`は使ったことがないため筆者には特に何かを述べることができません。

例として`encoding/json`を用いた`deep clone`を以下に示します。

[playground](https://go.dev/play/p/_-rxkpR7ahv)

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Foo struct {
    Foo string
    Bar *Bar
}

type Bar struct {
    Baz int
    Qux float64
}

func main() {
    org := Foo{
        Foo: "foo",
        Bar: &Bar{
            Baz: 12,
            Qux: 10.24,
        },
    }

    bin, err := json.Marshal(org)
    if err != nil {
        // handle error. here, I just simply let it panic
        panic(err)
    }

    var cloned Foo
    err = json.Unmarshal(bin, &cloned)
    if err != nil {
        panic(err)
    }

    printFoo := func(f Foo) string {
        return fmt.Sprintf(`Foo{Foo:%q, Bar:Bar{Baz:%d, Qux:%f}}`, f.Foo, f.Bar.Baz, f.Bar.Qux)
    }

    fmt.Printf("org = %s\ncloned = %s\n", printFoo(org), printFoo(cloned))
    // org = Foo{Foo:"foo", Bar:Bar{Baz:12, Qux:10.240000}}
    // cloned = Foo{Foo:"foo", Bar:Bar{Baz:12, Qux:10.240000}}

    // modification to one does not affect the other.
    org.Bar.Baz = -20
    fmt.Printf("org = %s\ncloned = %s\n", printFoo(org), printFoo(cloned))
    // org = Foo{Foo:"foo", Bar:Bar{Baz:-20, Qux:10.240000}}
    // cloned = Foo{Foo:"foo", Bar:Bar{Baz:12, Qux:10.240000}}
}
```

#### `reflect`によって動的なコピーを実装する

[The Go Blog: The Laws of Reflection](https://go.dev/blog/laws-of-reflection)で説明されている通り、`Go`は`reflect`パッケージを備え、runtime(プログラム実行時)に変数から型情報を取り出して動的な処理を実現することができます。

[reflect](https://pkg.go.dev/reflect@go1.23.4)の`godoc`を見れば分かりますが、動的な変数の宣言を行うことができるため、当然これを用いればdeep cloneも実現できます。

Pros, Consは以下になります。

- Pros:
  - code generation不要
  - (ライブラリによるが)細かいコピーのルールを設定して動的に変更することができる。
- Cons:
  - それそのものな機能はstdにないため、自前実装を行うか、ライブラリを使用する必要があります。
  - `reflect`はexportされたデータにしかアクセスできないというルールがあるため、unexportなフィールドのdeep copyを行いたい場合には利用できません。
    - `unsafe`パッケージを使うことでこのルールを迂回できますが、名前の通り不安全であるので割愛します。

以下などで`reflect`ベースの`deep clone`が実装されています。

- https://github.com/jinzhu/copier
- https://github.com/ulule/deepcopier
- https://github.com/mitchellh/copystructure (Public archive)
  - [github.com/compose-spec/compose-go](https://github.com/compose-spec/compose-go)がかつて短い期間これを用いてdeep copyを実現していましたが、今見ると[27c7848d662](https://github.com/compose-spec/compose-go/commit/27c7848d662558f4f8b37a4e7a8183f55264abf0)で依存性から取り除かれていました。

実装例を以下に示します。
ただし、この例は完全な実装ではなく、かなり雑なものであるので雰囲気がわかる以上のものではないことを留意してください。

[playground](https://go.dev/play/p/XrrJ3zN54SE)

```go
package main

import (
    "fmt"
    "reflect"
)

type Foo struct {
    Foo string
    Bar *Bar
}

type Bar struct {
    Baz int
    Qux float64
}

func cloneReflect(org any) reflect.Value {
    orgRv := reflect.ValueOf(org)
    clonedRv := reflect.New(orgRv.Type()).Elem()

    for i := range orgRv.NumField() {
        fOrg := orgRv.Field(i)
        fCloned := clonedRv.Field(i)

        switch fOrg.Kind() {
        case reflect.String, reflect.Int, reflect.Float64:
            fCloned.Set(fOrg)
        case reflect.Pointer:
            if fOrg.IsNil() {
                continue
            }
            rv := reflect.New(fOrg.Type().Elem())
            rv.Elem().Set(cloneReflect(fOrg.Elem().Interface()))
            fCloned.Set(rv)
        }
    }
    return clonedRv
}

func main() {
    org := Foo{
        Foo: "foo",
        Bar: &Bar{
            Baz: 12,
            Qux: 10.24,
        },
    }

    printFoo := func(f Foo) string {
        return fmt.Sprintf(`Foo{Foo:%q, Bar:Bar{Baz:%d, Qux:%f}}`, f.Foo, f.Bar.Baz, f.Bar.Qux)
    }

    cloned := cloneReflect(org).Interface().(Foo)

    fmt.Printf("org = %s\ncloned = %s\n", printFoo(org), printFoo(cloned))
    // org = Foo{Foo:"foo", Bar:Bar{Baz:12, Qux:10.240000}}
    // cloned = Foo{Foo:"foo", Bar:Bar{Baz:12, Qux:10.240000}}

    // modification to one does not affect the other.
    org.Bar.Baz = -20
    fmt.Printf("org = %s\ncloned = %s\n", printFoo(org), printFoo(cloned))
    // org = Foo{Foo:"foo", Bar:Bar{Baz:-20, Qux:10.240000}}
    // cloned = Foo{Foo:"foo", Bar:Bar{Baz:12, Qux:10.240000}}
}
```

本題ではないため、`reflect`でこれが実現できることだけ示し、深い解説はしないものとします。
見てのとおり、自分で実装すると大変なので、ライブラリを使用したほうが良いと思います。

#### code generatorによってdeep clonerを実装する

[The Go Blog: Generating code](https://go.dev/blog/generate)などからわかる通り、`Go`にはcode generatorを呼び出すための`//go:generate` directiveが存在します。
([directive](https://tip.golang.org/doc/comment#syntax)というのは`//`のあとに` `(space)を含まないコメントのことです。これは`go doc`などに表示されなくなるような特別扱いがあります。[ここ](https://pkg.go.dev/cmd/compile#hdr-Compiler_Directives)などを参照。)
このdirective commentが書きこまれたgo source codeを指定して`go generate ./path/to/file.go`を実行すると、`//go:generate`の後に書かれたコマンドが実行される仕組みになっています。

`Go`自身も`//go:generate`を活用しており、

https://github.com/search?q=repo%3Agolang%2Fgo%20%2F%2Fgo%3Agenerate&type=code

と多用されていることがわかります。

このように、`Go`にとってcode generatorを使用するのは一般的なことです。

そもそもマクロが現状(`Go1.24`時点)ありませんし、それに類するツールもありません。[Go1.18]までgenericsもなかったため、code generatorを使わざるを得ないことも多くありました。

Pros, Consは以下になります。

- Pros:
  - exportされていないfieldのコピーが可能
  - 高速: `marshal`/`unmarshal`のオーバーヘッドや、`reflect`によるメモリアロケーションコストを回避できます。
- Cons:
  - コード変更のたびにcode generatorを動作させる必要がある。

Go1.23までではcode generatorのバージョン管理が面倒というconsも存在していますが、Go1.24以降は`go.mod`にtool directive付きで記録することが可能になるため、若干取り扱いがよくなりました。

これらの機能を提供するライブラリは

- https://github.com/ulule/deepcopier
- https://github.com/switchupcb/copygen
- https://github.com/reedom/convergen
- (私のやつ) https://github.com/ngicks/go-codegen

などがあります。

### どうしてcode generatorを実装する必要があるのか

前述から

- `marshal`/`unmarshal`用いる方法はバイナリを経由することのオーバーヘッドがかかる
- `reflect`を用いる方法ではexportされていないfieldのコピーができない

というデメリットがそれぞれあります。

code generatorを用いる方法は、ソース変更を行うたびに再生成が必要というデメリットはあるものの、上記の二つを克服しうるものであることは述べました。

ではなぜすでにcode generatorライブラリが存在している今の状況で新規に開発を行う必要があるのでしょうか？

- 1. **作りたかったから**です
- 2. どうもtype paramのある型に対する`deep clone`の生成をうまくこなすgeneratorがないっぽい？

`1.`に関しては何も言うことはありません。趣味なんだから作ってしまえばいいです。

`2.`に関してなのですが、例えば下記のようなコードがあるとき

```go
package paramcb

type A[T any] struct {
    A B[string, T]
    B C[[]string]
    C B[C[string], []C[string]]
}

type B[T, U any] struct {
    T T
    U U
}

type C[T any] struct {
    T T
}
```

これに対して前述の[github.com/ulule/deepcopier](https://github.com/ulule/deepcopier)を実行します。

```
go run github.com/globusdigital/deep-copy@latest -type A -type B -type C ./path/to/paramcb/
```

下記がstdoutに出力されます。

```go
// generated by /tmp/go-build4222039145/b001/exe/deep-copy -type A -type B -type C ./generator/cloner/internal/testtargets/paramcb/; DO NOT EDIT.

package paramcb

// DeepCopy generates a deep copy of A
func (o A) DeepCopy() A {
        var cp A = o
        if o.B.T != nil {
                cp.B.T = make([]string, len(o.B.T))
                copy(cp.B.T, o.B.T)
        }
        cp.C = o.C.DeepCopy()
        return cp
}

// DeepCopy generates a deep copy of B
func (o B) DeepCopy() B {
        var cp B = o
        if o.U != nil {
                cp.U = make([]C[string], len(o.U))
                copy(cp.U, o.U)
        }
        return cp
}

// DeepCopy generates a deep copy of C
func (o C) DeepCopy() C {
        var cp C = o
        return cp
}
```

type paramを無視していますね。

2年ほど前(2023年1月あたり)にこれらを含めてさらにもう1つか2つのcode generatorを試したことがあるんですが、セットアップがややこしいか正しいコードを吐いてくれなかった(`*map[string]Foo`のpointerの部分が無視されててコンパイルできないコードが吐かれたり)ため、作りたいなあという漠然とした思いだけが残っていました。
[前回の記事: \[Go\]ast(dst)と型情報からコードを生成する(partial-json patcher etc)](https://zenn.dev/ngicks/articles/go-code-generation-from-ast-and-type-info)で、ほぼ`deep cloner`のようなものを副作用的に作っていたためせっかくなので作ってみようということで作り始めました。

## 生成するdeep clone methodのsignature

生成する`deep clone` methodのsignatureを以下のように定めます。

```go
func (Type) Clone() Type
func (Type[T, U, V,...]) CloneFunc(cloneT func(T) T, cloneU func(U) U, cloneV func(V) V,...) Type[T, U, V,...]
```

`Copy`, `DeepCopy`, `Clone`, `DeepClone`など色々派閥あると思いますが今回は`Clone`で行きます。

シンプルな型には`Clone`メソッドを生成します。`Type`はnon-pointer type(`*T`でなく`T`)とします。これは各々メリット、デメリットあると思いますが`deep clone`の意図がデータの複製であるのでpointerは返さないものとします。

[type parameter](https://go.dev/ref/spec#Type_parameter_declarations)のある型には`CloneFunc`を生成します。こちらも同じくnon-pointer typeを返します。
それぞれのtype paramを複製するためのコールバック関数を`clone+{type param name}`で受け取ります。

type parameterやgenericsについては[A Tour of GoのType parameters](https://go.dev/tour/generics/1)で説明されているので説明は割愛します。

## 生成されるメソッドのイメージ

以下のような型があるとき

```go
type A struct {
    A string
    B int
    C *int
}
```

理想的には下記が生成されます。

```go
func (a A) Clone() A {
    return A{
        A: a.A,
        B: a.B,
        C: func(v *int) *int {
            if v == nil {
                return nil
            }
            vv := *v
            return &vv
        }(a.C),
    }
}
```

さらに、各フィールドは`[]T`や`map[K]V`がいくつネストしてもいいものとします。
つまり以下のような型があるとき、

```go
type B struct {
    A [][]string
    B map[string]map[string]int
}
```

理想的には下記のようなメソッドが生成されます。

```go
func (v B) Clone() B {
    return B{
        A: func(v [][]string) [][]string {
            if v == nil {
                return
            }
            out := make([][]string, len(v), cap(v))
            for k, v := range v {
                if v == nil {
                    continue
                }
                vv := make([]string, len(v), cap(v))
                copy(vv, v)
                out[k] = vv
            }
            return out
        }(b.A),
        B: func(v map[string]map[string]int) map[string]map[string]int {
            if v == nil {
                return
            }
            out := make(map[string]map[string]int)
            for k, v := range v {
                out[k] = maps.Clone(v)
            }
            return out
        }(b.B),
    }
}
```

加えて、type paramがある場合は`CloneFunc(cloneT func(T) T, cloneU func(U) U, ...)`のような形で各type paramをcloneするためのコールバック関数を受けとります。
type paramを指定されたfield、ないしはtype paramでinstantiateされた型のfieldの`clone`にはこれらのコールバック関数を使用します。

もちろん、生成対象となっている型(`A[T any]`)が別のtype paramを持つ型(`B[T, U any]`)を含み、それがtype param以外でinstantiateされている(`B[string, T]`)こともあり得ます。その場合は、instantiateに使われた型に対応したclonerをコールバック関数として渡します(`func(s string) string, cloneT`)。

つまり下記のような型があるとき、

```go
type A[T any] struct {
    A B[string, T]
}

type B[T, U any] struct {
    T T
    U U
}
```

下記が生成されます。

```go
func (a A[T]) CloneFunc(cloneT func(T) T) A[T] {
    return A[T]{
        A: a.A.CloneFunc(
            func(v string) string {
                return v
            },
            cloneT,
        ),
    }
}

func (b B[T, U]) CloneFunc(cloneT func(T) T, cloneU func(U) U) B[T, U] {
    return B[T, U]{
        T: cloneT(b.T),
        U: cloneU(b.U),
    }
}
```

## `Clone`/`CloneFunc`の実装方針

- 単なる代入でコピーできるものに関してはフィールドにその値を代入します。
- 各フィールドが単純な関数呼び出し(e.g. `maps.Clone`など)ですまない時、無名の関数を作成して即座に呼び出します。
  - scopeを分けることでコードを単純にします(生成されるコード、生成するコード両方を)
  - 呼び出さずに`CloneFunc`に渡すこともできます。
  - [inline](https://github.com/golang/go/blob/go1.23.4/src/cmd/compile/internal/inline/inl.go), [devirtualize](https://github.com/golang/go/blob/go1.23.4/src/cmd/compile/internal/devirtualize/devirtualize.go)などでコンパイラが最適化してくれることを期待します。
    - 実際にコンパイラがどういうコードを生成するのかは確認していません・・・
  - 全くな同じ定義の無名関数が複数あったら一つにまとめるような最適化もどこかにあるだろうと予測しています。(すみません。これは全く確認してないです。)
  - 定義が膨れることで読みにくくなるデメリットはありますが、生成物のまとまりがよくなるメリットがあります
  - 外部moduleにcommon partsをまとめてそれを呼び出す方法も考えられますが、バージョン管理が複雑になるため避けます。
- 生成時に読み込んだパッケージ群外で生成されたnamed typeに関しては、fieldとそれが指定する型がすべてexportされている場合に限ってad-hocな無名関数を生成してcloneします。

## code generator実装の基本方針

ここからは[前回の記事: \[Go\]ast(dst)と型情報からコードを生成する(partial-json patcher etc)](https://zenn.dev/ngicks/articles/go-code-generation-from-ast-and-type-info)との重複がありますが特に内容が前提となっているわけではありません。

実装前にあった当初の方針が下記になります。

- 1. 型情報を収集する
  - 高度な型の判別には型情報が不可欠です。
  - この手のcode generatorは大抵型情報を使っている印象です。
  - 型情報を使って、ある型に*method*(`Clone`/`CloneFunc`)を実装してもよいかを判別します
    - 例えば以下のような型のみを含む型は生成対象になりません。
      - channelを含む
      - NoCopyである(e.g. `*sync.Mutex`, `*sync.RWMutex`)もしくは
      - named typeであり
      - 生成対象のパッケージ群で定義されたものでなく
      - `implementor`(*method*を実装する)ではなく
      - `clone-by-assign`(non-pointerのみを含む型)でないとき
- 2. 型情報をグラフ化する
  - named typeをnodeとし、node間にedgeを描くことでグラフとします
  - グラフを辿ることで`Clone`/`CloneFunc`を実装することになる生成対象の型を含む型を探索します。
  - `A`(ある生成対象の型)を含む`B`(別の型)の`Clone`/`CloneFunc`実装内で`A`の`Clone`を呼び出していいのかを判断するには、グラフをたどって行く必要があります。
  - `A`を含む`B`を含む`C`という型がある場合、ソースコードからわかる依存関係は`C` -> `B` -> `A`ですが、やりたい判別には逆順の`A` -> `B` -> `C`で辿る必要があります。
  - そのためグラフを事前に作っておく必要があります。
- 3. field unwrapper: `[]map[string]*[5]T`という型があるとき、`[]map[string]*[5]`の部分と、`T`に分けて考え、前者側向けの共通処理を用意します。
  - 前者は`for-loop`などで展開することでコピー可能です
  - `T`の部分だけclone方法が型によって変わります。
- 4. channel, NoCopy type(e.g. `*sync.Mutex`, `*sync.RWMutex`), funcの取り扱いをユーザーに決めさせるためにConfigを受けとれるようにする

## 型情報を収集する

高度な型の判別のために型情報を収集します。`go/ast`, `go/types`で定義された各型がast, 型情報に対応しており、これらを使うことでgo source codeを解析した結果の型情報を取り扱うことができます。

### [golang.org/x/tools/go/packages]による`ast`, `type info`の読み込み

[golang.org/x/tools/go/packages]を用いるとGo source code解析してast、type info、そのほかのメタデータなどを得ることができます。

[go/ast], [go/types]がそれぞれast解析、astからtype info解析を行う機能を提供しているのですが、対象となるGo source codeが外部モジュールをimportしているとき、これをうまく読み込む手段をこれらは用意していません。
そのため、go moduleのdependency graphを形成して末端のnodeとなる、自分以外に何もimportしていないmoduleから順番に解析してimporterとしてtype infoを返すようなものを自作する必要があります。それをやってくれるのが[golang.org/x/tools/go/packages]なのです。

[golang.org/x/tools/go/packages]でastを解析して、type checkした結果を受けとるには以下のようにします

```go
import "golang.org/x/tools/go/packages"

func main() {
    cfg := &packages.Config{
        Mode: packages.NeedName |
            packages.NeedTypes |
            packages.NeedSyntax |
            packages.NeedTypesInfo |
            packages.NeedTypesSizes,
        Context: ctx,
        Dir:     dir,
    }
    pkgs, err := packages.Load(cfg, "variadic", "package/match", "patterns")
    if err != nil {
        // handle error
    }
}
```

`packages.Load`でpackage patternを受けとり、マッチするmoduleの各種情報を[\[\]\*packages.Package](https://pkg.go.dev/golang.org/x/tools@v0.28.0/go/packages#Package)として取得します。

内部的には`go list -deps -json package/pattern`によって依存をリストアップします。他の`go module`-awareなツールと同じように使う必要があります。
package patternが相対パスの場合はcwdから評価されます。`Dir`でcwdを変更できます。

`*packages.Config`の`Mode`がビットフラグでロードする情報をコントロールします。上記は`PkgPath`(string), `Types`(`*types.Package`), `Syntax`(`[]*ast.File`), `TypeInfo`(`*types.Info`), `TypesSizes`(`types.Sizes`)がロードされます。
`Mode`ビットは`Need{field name}`という名前(になっていないものもいくつかあるが)で各種定義されていますのでほしい情報に合わせてbitwise-ORをとります。

### 型情報を使った各型の判別

複雑な型や、interfaceで表現することができない型の条件は`go/types`以下で定義される型情報を直接走査して判定を行います。

#### `NoCopy`(assignによってコピーすると`go vet`が`copies lock value`と警告する型)の判別

`Lock` methodを備える型を直接(pointerによってindirectされずに)含む型はno-copyなどと言われて、代入や関数の引数に渡すことでコピーが起きると`go vet`で警告を受けます。
code generatorはこれらをコピーしないようなコードを生成する配慮が必要なので判別する必要があります。

```go
type noCopy struct{}
func(noCopy) Lock()

type noCopy2 struct {
    l noCopy
}

type notNoCopy struct {
    l *noCopy
}
```

上記の

- `noCopy`は`Lock` methodを実装するためNoCopyです。
- `noCopy2`はnon-pointerとして`noCopy`を含むためNoCopyです。
- `notNoCopy`はpointerとして`noCopy`を含むためNoCopyではありません。

pointerと言えるのは、`interface`, `map[K]V`, `[]T`を含みます。array(`[n]T`)はpointerではないので以下も`noCopy`です

```go
type noCopyArray struct {
    l [3]noCopy
}
```

`Clone`/`CloneFunc`はこれらをコピーしないように単にフィールドをzero valueのままほおっておくとか、含まれる型はそもそも生成対象から外すとか、pointerならpointerをコピーするとかをユーザーに選ばせたいので、no-copyの判定を行う必要があります。

以下のように実装します。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/matcher/matcher.go#L9-L49

`findMethod`の実装は以下

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/matcher/matcher.go#L101-L108

`asNamed`, `asInterface`, `as[T]`の実装は以下

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/matcher/matcher.go#L64-L77

この実装はgo vetのそれとは異なり、`sync.Locker`のような`interface`を[struct embedding](https://gobyexample.com/struct-embedding)することで`Lock`を実装しているnon-interface型もpointerではないとみなし、no-copy typeとして判定します。

つまり以下はこの実装ではNoCopyとして取り扱われますが、`go vet`は警告しません。

```go
type notNoCopy2 struct {
    sync.Locker
}
```

#### `implementor`(`Clone`/`CloneFunc`を実装する型)の判別

ある型`A`が内部に型`B`を持つとき、`B`が`implementor`(`Clone`/`CloneFunc`を実装する型)であるならば`A`の*method*(`Clone`/`CloneFunc`)は`B`の*method*を呼び出します。
`Clone`はともかく、`CloneFunc`は複雑かつ、`interface`として表現できない複雑な条件であるため、型情報を用いた判別を行います。

##### Clone

`Clone`の実装は以下のように判定します。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/matcher/method_checker.go#L101-L123

引数が`func (Type) Clone() Type`か`func (*Type) Clone() Type`というmethodを持つときtrueを返します。

`Clone`以外のmethod nameでもいいように、`Name`フィールドでパラメータ化してありますが実際の呼び出しは`Name: "Clone"`以外ですることはありません。

`asPointer`の実装は以下。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/matcher/matcher.go#L79-L90

[types.NewMethodSet]で、ある型が実装するmethod setを得ることができますが、`Go`の通常のinterfaceのルールと同じくnon-pointer型にはreceiverがnon-pointer型のmethodしか見せなくなっています。
すべてのmethodを見つけるために、型がpointerでない場合は`types.NewPointer`でラップすることでpointerに変換します。
ただし`interface`のpointerをとると逆に[types.NewMethodSet]はmethodを返さなくなるため、`interface`である型はpointerに包まないようにします。
[*types.Named](named type),[*types.Alias]もしくはinterface literal以外はmethodを実装することはないため、それ以外の型の場合はそのまま引数を返します。

`noArgSingleValue`の実装は以下

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/matcher/matcher.go#L110-L131

`unwrapPointer`の実装は以下

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/matcher/matcher.go#L92-L99

##### CloneFunc

`CloneFunc`の実装は以下のように判別します。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/matcher/method_checker.go#L125-L196

前述通り、`CloneFunc(cloneT func(T) T, cloneU func(U) U, ...)`というシグネチャであるかを判別します。処理の単純性のためにtype paramとclonerコールバック関数の順序は一致することを必須とします。
判定する型によっては`A[string, T]`のような感じで具体的な型だけでなく、さらに別のtype paramでinstantiateされていることがあります。
`types.Identical`はtype paramに対して単なる`pointer`同士の比較以上のことをしないように実装されているため、type paramはindexで比較する必要があります。

#### `clone-by-assign`(non-pointerのみを含む型)の判別

`clone-by-assign`(non-pointerのみを含む型)である場合は、生成対象のパッケージ群で定義されている型でない時でも単純にassignすればよいので、これを判別できるようにしておきます。これの具体例は[image/color.RGBA64](https://pkg.go.dev/image/color@go1.23.4#RGBA64)などですね。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/matcher/tester.go#L5-L31

わりかし単純です。ただし、`stepNext func(*types.Named) bool`を受けとってnamed typeに対してマッチするとき再帰しないで`false`を返す措置があります。
こういうシグネチャになっているのは、引数が生成対象のnamed typeであったり、`implementor`であったりして、*method*を実装しているとき、それらを`clone-by-assign`として取り扱わないようにしたいからです。その時にはそれらの*method*を呼び出すようにします。
とくに`implementor`に対しては*method*内でどういうフックを行っているか明らかでないのでとりあえず呼び出さないと実装者の意図に反する可能性があります。

## 型情報をグラフ化する

生成される`Clone`/`CloneFunc`メソッドは、生成対象の型がさらに*method*を実装する型を含んでいる場合はそれらを呼び出します。

```go
type A struct {
    B Implementor
}

func (a A) Clone() A {
    return A{
        B: a.B.Clone()
    }
}

type Implementor struct{
    // ...
}

func (i Implementor) Clone() Implementor {
    return Implementor{/*...*/}
}
```

上記で言うところの`Implementor`もさらに生成対象の型であるとき、これが実装する`Clone`もcode generatorによって生成されることになるので**初回実行時にはまだ存在していません。**
つまり`implementor`の判別を行うだけではこの`Clone` methodを発見することができません。

そのため、stableな出力結果を得るためには、型が参照する別のnamed typeが生成対象となっているのかを検知する必要があります。

型は別の型を含むことができ、さらにその型が別の方に依存していることがありまえます。これをここで`type dependency chain`と呼びます。
このchainをたどったとき、どこかに生成対象の型が含まれている場合、chainの上流にいる型も生成対象の型となります。

source codeや、それの解析結果自体が型、呼び出しの依存関係の順向きグラフとなっています。
今回知りたいのは型がどこから参照されているかという逆向きの情報です。
おそらく、この逆向きの情報を手軽に得る方法は存在しないため、特別な実装を必要としています。

(`gopls`のFind All Referencesの実装を見たら一般的にどうやって逆向きの情報の得るのかを調べれるなあと思うだけ思って調べてないです)

以下のpackageでそれを実装します。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/typegraph/type_graph.go

このグラフは作成時に渡された`[]*packages.Package`内部のnamed typeをすべて列挙し、named type同士の依存関係を親から子、子から親に相互に参照できるようにedgeでつなぎます。

型は`[]T`, `map[K]V`, `chan T`など無名の型に含まれることがあります。
この型グラフはnamed typeからnamed typeへの依存関係のみを焦点としますが、場合によっては特定の無名な型を経由して依存される場合、逆向きに型をたどりたくないことがあります。
例えば、`chan T`であるとき、`Clone` methodはこの`T`に何のアクションも起こせませんので、このedgeをたどる必要はありません。
そこで、edgeはこれらの無名の型を*edge route*と呼び、`[]T`, `map[K]V`それぞれを*edge route node*のstackとして別途記録しておきます。

さらに、matcherを受けとり、すべての型を列挙するときにこれを使用することで、関心のある型にマーキングを行います。
今回で言うと`clone-by-assign`(non-pointerな型しか含まれない型)か`implementor`(`Clone`/`CloneFunc`を実装するnamed type)を含む型が`Clone-able`としてマッチします。

`IterUpward`を以下のように実装し、matcherでmatchした型から依存関係を親側に向けてたどります。channelを含む*edge route*に対しては`Clone`/`CloneFunc`を呼び出すことはできないため、これらを含むedgeはフィルターして辿らないこととします。そのため`edgeFilter`を受けとるようになっています。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/typegraph/type_graph.go#L584-L614

`MarkDependant`を以下のように定義することで、`IterUpward`で辿られた型を`dependant`としてマークします。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/typegraph/type_graph.go#L572-L582

いずれかのマークがされた型にのみ`Clone`/`CloneFunc`を実装していきます。
逆に言うとマークされた型に対しては盲目的に(型情報によらずに)`Clone`/`CloneFunc`を呼び出してよいことになります。

## field unwrapper

`[]map[string]*[5]T`という型があるとき、`[]map[string]*[5]`の部分と、`T`に分けて考え、前者側向けの共通処理を用意します。

以下のようにoutermost(初期化と返却)、mid(中間経路)、inner end(`T`のclone expression)の三つに分けます。

```go
func (v []map[string]*[5]T) []map[string]*[5]T {
/* outermost */ out := make([]map[string]*[5]T, len(v), cap(v))
/*       mid */ for k, v := range v {
/*       mid */     next := make(map[string]*[5]T)
/*       mid */     for k, v := range v {
/*       mid */         next := new([5]T)
// ...
/* inner end */         for k, vv := range v {
/* inner end */             v[k] = cloneExpr(vv)
/* inner end */         }
// ...
/*       mid */     }
/*       mid */ }
/* outermost */ return out
}
```

outermostは返すclonedの初期化と返却を行います。
midは`for`(`slice`,`array`,`map`)か`if v != nil { v := *v }`(`pointer`)で引数の`v`を1つずつ*unwrap*していき、そのたび一つ内側の型のcloneされたオブジェクトを初期化します(`[]map[string]*[5]T` -> `map[string]*[5]T` -> `*[5]T` -> `[5]T`)
inner endは`T`ごとのclone expressionを記述します。`implementor`なら`Clone`/`CloneFunc`、`clone-by-assign`なら単に`vv`を代入するのみ、という感じです。

`Go`は通常、型推論が行われるため変数の型を宣言しないことも多いです。実際上記の`mid`では`for k, v := range v`とするときにiteration variable(`k`と`v`)の型を明確に記述していません。
しかし`map`, `slice`などの初期化に`make`組み込み関数を使うときに型シグネチャを必要とします。これを用いないと`slice`の`len`や`cap`を指定できません。
ですので`mid`を経由するたび1つ*unwrap*した型をテキストとして出力できなければなりません。(e.g. `[][]T` -> `[]T`)
そこで`types.Type`か`ast.Expr`を受け取って順繰りに*unwrap*する機能が必要です。`types.Type`は[types.TypeString](https://pkg.go.dev/go/types@go1.23.4#TypeString), `ast.Expr`は[printer.Fprint](https://pkg.go.dev/go/printer@go1.23.4#Fprint)でテキストとして出力可能だからです。

ここでは`types.Type`で行うこととします

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/method.go#L480-L494

`typegraph.EdgeKind`は前述の*edge route*の種類を表現するenum-likeな値です。
これを使わなくてもunwrapは成立するんですが、こうするとtypegraph情報との連携がうまくいっていない場合にtype-assertionのところでpanicするので便利です

`ast.Expr`でなく`types.Type`を使う理由はtype aliasをunaliasするのが型情報を必要とするからです。
例えばですが、下記のようにtype aliasが何度も重なっていることはあり得ます。

```go
type A = B

type B = C

type C struct {
    // ..
}
```

`type A = B`のast nodeを受け取っているとき`A`から`B`まではastのunwrapするだけで簡単に取り出せますが、`B`から`C`をたどるのはまさに型情報です。

上記よりfield unwrapperを`unwrapFieldAlongPath`として定義します。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/method.go#L496-L621

(返された関数を2度以上呼び出すと(そうなることを意図していないにもかかわらず)結果が変わるよくない実装になっています。参考にする人がいるかはわかりませんが注意してください。)

`fromTy, toTy types.Type`を引数に取ることで`fromTy -> toTy`な関数を出力します。のちの再利用を前提として変換先に別の型を指定できるようになっていますが、今回はclonerなのでこの二つは全く同じ`types.Type`が渡されることになります。
返り値の関数`unwrapper`で上記で言うclone exprを`wrappee func(string) string`として受け取とり`inner end`でそれを呼び出すようなfield unwrapperをテキストで出力します。

## code generatorの実装

### Configの定義

たとえ挙動を変えうる設定項目が一つもなくてもconfigを主体にAPIを設計しないとあとから設定項目を追加するのが破壊的変更なってしまいますので毎回何かを無理くりひねり出すんですが幸いにも今回はいくつかユーザーに取り扱いを決めてほしいものがあります。
そこで`Config`を以下のように定義しています。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/generator.go#L25-L28

今後項目が増えるかもしれませんが現在はこれだけです。

`MatcherConfig`は以下のように定義されます。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/matcher.go#L22-L45

これも項目が増えるかもしれませんが現時点ではこれだけです。`NoCopy`, `Channel`, `Func`, `Interface`のグローバルオプションをそれぞれ用意しています。
`CopyHandleIgnore`ならフィールドはclone対象にならず、clone後にはzero valueになります。`CopyHandleDisallow`ならこれを含む型は生成対象から除外されます。`CopyHandleCopyPointer`は、そのフィールドがpointerであるとき(=`*T`, interface, channelなど)の時のみコピーを行いそれ以外の時は`Ignore`として取り扱います。

`CustomHandlers`は後述します。

`Config`に`Generate` methodを実装します。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/generator.go#L52-L56

こうすれば`Config`を無視して何かをすることはできなくなります。

### in-place option: struct fieldにコメントをつけて挙動を変更できるようにする

挙動のfine tuningのためにstruct fieldにdirective commentをつけることでconfigをper-fieldレベルで上書きできるようにします。これを`in-place option`と呼びます

#### typegraphのデータにstruct fieldのコメントを格納できるよう拡張する

[型情報をグラフ化する](#型情報をグラフ化する)のところで説明した通り、今回実装するcode generatorは事前に型情報グラフ化して、その時に渡された`matcher`の結果で生成対象の型を決定します。
per-fieldレベルの設定によってtype graphのマッチする、しないが左右されるためtypegraphの機能としてper-fieldレベルのデータを収めることとします。

そこで以下のようにoptionを定義し、

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/typegraph/option.go#L3-L19

typegraphの`New`関数でOptionを受けとれるようにします。(破壊的変更を加えました)

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/typegraph/type_graph.go#L226-L232

`typegraph.Node`に`Priv`(private)データを含めるようにします。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/typegraph/type_graph.go#L64-L78

こういうのはC言語だとよく見るパターンですね。

`PrivParser`はmatcher呼び出しの直前で呼び出します。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/typegraph/type_graph.go#L308-L315

この`Priv`データ自体はtypegraphにとって関心のある所ではないため`any`になっています。データは利用者ごとに別々のものを用意したほうがよいでしょう。

下記のように`Priv`をtype paramにしてもよかったのですが、optionalなもののためにtype paramを追加するのはわかりにくくなるのでやめておきました。

```go
type Node[T any] any {
    // ...
    Priv *T
}
```

#### コメントを解析する

前述通り`*typegraph.Node`を受けとって型情報ないしはast情報を用いて`Priv`に情報をセットできるようになっています。

Priv dataは以下の`clonerPriv`として定義します。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/priv.go#L26-L36

これは前述のConfigをoverrideできるようにロジックを集約しておきます。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/priv.go#L38-L52

(`Interface`のオーバーライドが実装されていない！そのうち直ります。)

「struct fieldにdirective commentをつける」のstruct fieldについているコメントは下記で言うと`A`に付いているコメントは`// 3`, `/* 4 */`, `// 7`に当たる位置のみと定義します。

```go
type A struct {
    _                 int
    // 1

    // 2

    // 3
    /* 4 */ A /* 5 */ string /* 6 */ `json:"a"` // 7
    /* 8 */
    // 9
    B                 string /* 10 */
    // 11
    C                 int
    // 12
    // 13
}
```

解析は[github.com/dave/dst]を用いて、dstにいったん変換してから行います。
`dst`では`ast`と違い、コメントは直前もしくは直後のnodeにアタッチされているものとして取り扱われます。
Goの`go/ast`におけるコメントの取り扱いは単なるオフセットなので`dst`と比べると取り扱いがかなりややこしいです。

上記の場合,`A`, `B`, `C`についているコメントは`dst`上では

```go
st := dts.Type.(*dst.StructType)
a := st.Fields.List[1]
b := st.Fields.List[2]
c := st.Fields.List[3]

//
// // 2
//
// // 3
// /* 4 */ A /* 5 */ string /* 6 */ `json:"a"` // 7
//

a.Decs.Start
// [0] = "// 2"
// [1] = "\n"
// [2] = "// 3"
// [3] = "/* 4 */"
a.Decs.End
// [0] = "// 7"

//
// /* 8 */
// // 9
// B               string /* 10 */
//

b.Decs.Start
// [0] = "/* 8 */"
// [1] = "\n"
// [2] = "// 9"
b.Decs.End
// [0] = "/* 10 */"

//
// // 11
// C                 int
// // 12
// // 13
//

c.Decs.Start
// [0] = "// 11"
c.Decs.End
// [0] = "\n"
// [1] = "// 12"
// [2] = "// 13"
```

という風になります。`dst`上でもコメントの取り扱いは微妙であとにフィールドが続くかによって何がどこに入ってくるか変わってしまいます。

fieldにアタッチされたコメントは以下のように定義できます。

- `Start`: 最後の`\n`以後
- `End`: 1つ

これらのコメントを列挙する関数を`ParseFieldDirectiveCommentDst`として定義すると、以下のように各フィールドのコメントを解析できます。
(これそのものはシンプルなテキスト解析なので特にいうことはありません)

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/priv.go#L54-L111

### matcherの定義

与えられた型に対してどのようなコードを生成すればよいかを判別するmatcherを定義します。
どのようにハンドルすべきか(`type handleKind int`)を定義して、それを返す部分と、実際にコードを生成する部分を分けて実装します。前者がこのmatcherです。
こうすると以下の点で好都合です

- 型グラフ生成時のmatcherとして同じ処理を使いまわせます
- そもそもmatcher部分が巨大かつユーザーに内部状態を教えるためにloggerを受けとるため実際のコード生成はコード生成だけに集中しないとごちゃごちゃして読めたもんじゃないです
- config項目の拡充などでmatcher部分は今後も変わり続けますがコード生成部分は多分あんまりもう変わらないので分離できておくと差分が見やすくていいです

あるnamed typeから別のnamed type、あるいは他の型を含むことができない型(`int`のようなbasic typeや`func`, `interface`, `type param`など)までをたどり、*edge route node*とその最終的な型を引数にしてコールバック関数を呼ぶ`TraverseTypes`を定義し、これを活用します。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/typegraph/type_graph.go#L491-L542

`TraverseTypes`を使って型をとり、判別を行います。struct literalが含まれる場合は再帰処理で対応するのでstruct literalが出たら*traverse*を中断したり、custom handler(後述)にマッチしたらマッチする直前までのfield unwrapperを生成したいのでそこで処理を中断したりといろいろ考慮を加えます。

そしてmatcher本体ロジックは下記のクソデカswitch-case

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/matcher.go#L181-L446

デカい！デカくてzennのpreviewだと最後まで表示できていないですね(200行までの制限がかかっているようです)。

こういったコアロジックは長大になる傾向がある気がします。どのタイミングで分割するか考えていないですしばらくはデカいまま放置されると思います。

### 各型のclone

前述のmatcherの呼び出して、得られた`handleKind`をswitch-caseで分岐することで各型のcloneを生成します。前述のfield unwrapperがあるため、実際には`[][]T`のような型があるときには`T`のclone方法のみを生成します。

`T`が型グラフの構築時にmatchedとなった型については`Clone`/`CloneFunc`を呼び出す必要があるので、その考慮を加えるために`handleField`というラッパーを経由してmatcherを呼び出します。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/matcher.go#L508-L550

各型向けに呼び出します。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/method.go#L218-L224

switch-caseで分岐してそれぞれ向けのテキストを生成します。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/method.go#L260-L374

struct literalと`CloneFunc`のtype argに対して再帰呼び出しが必要でややこしいですがmatcher部分と分離したため割合単純なコードになっています。

### field unwrapperと組み合わせる

`unwrapFieldAlongPath`の呼び出しによって得られたfield unwrapperと組み合わせて最終的な`cloner func(string) string`が得られます

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/method.go#L253-L258

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/method.go#L376-L386

### source fileにsuffixを付けたファイルへ書き出し

あとはこれを書き出したら終わりです。

対象となった型が含まれていたファイル名+suffixなファイルに吐き出す方式をとるため以下で`suffixwriter`を定義します

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/suffixwriter/writer.go

[token.FileSet.Position](https://pkg.go.dev/go/token@go1.23.4#FileSet.Position)で得られる[token.Position](https://pkg.go.dev/go/token@go1.23.4#Position)からファイル名が得られます。
これを引数に`suffixwriter`を呼び出すことで所望の挙動を実現できます。

## build constraintsのコピー

[go command documentationのBuild constraintsの項](https://pkg.go.dev/cmd/go#hdr-Build_constraints)からわかる通り、`//go:build` directive commentや、ファイル名に`_linux`や`_amd64`などのsuffixをつけることでbuild constraintsを指定できます。

リンク先でも説明されていますがbuild constraintsとはどのような条件でこのファイルがpackageに含まれるかを決めるものです。

よくある使われ方はマルチプラットフォーム対応です。
プラットフォーム固有の機能やパラメータを同名の関数や定数をプラットフォームごとに定義しておき、`linux`向けとか`mac`向けとか、`amd64`向けとか`arm64`むけとか、そういったbuild constraintsでパッケージに含まれる定義を切り替えられるようにします。
そうすることで他のプラットフォーム非依存なコードから呼び出せるようにします。

他で言えば[wails](https://wails.io/)(ElectronやTauriの`Go`版でおおむね間違っていない)のdevビルドとprodビルドをbuild constraintで切り替えて、dev版ではweb viewにdev toolsを表示しておくとかそういう使い方も考えられます。しょっちゅう使われるものという印象もないですが、覚えておくと便利なときもあると思います。

`go`コマンドに組み込みのbuild constraintsには`GOOS`(`linux`, `windows`, `darwin`など)や`GOARCH`(`amd64`, `arm64`など)などがあります。
これらは`go env GOOS`などで確認できる値として勝手にbuild tagに設定されます。
そのほかの任意のbuild tagは[Command Documentation: go](https://pkg.go.dev/cmd/go#hdr-Compile_packages_and_dependencies)より、`go`コマンドのサブコマンドのうちビルド行うもの(`build`, `test`, etc)に`-tags` cliオプションを使って設定します。

C言語では[#ifdef](https://learn.microsoft.com/ja-jp/cpp/preprocessor/hash-ifdef-and-hash-ifndef-directives-c-cpp)などを活用して、ファイルの中でもconstraintに合わせて定数や関数の定義を切り替えることができます。ファイルの特定の行があるかどうかを制御する`if`のようなものですから、1つのファイル内で複数のconstraintによる分岐が行えます。
`Go`では一方で、1ファイルに1つしか`//go:build`が存在することが許さないようになっています。それぞれの環境向けのファイルを複数作っておき、それぞれで同名の関数/パラメータを定義することになります。

さて、build constraintを尊重する方法についてですが、以下の二つがあります。

- `//go:build`コメントをコピーする
- ファイルのsuffixがbuild constraintであるときはコピーする

### //go:buildコメントのコピー

コピーするというよりは`//go:build`コメント以外を消すというのが正しいです。

今夏実装したcode generatorでは生成元の型が定義されたファイルのpackage clause、import declをそのまま再利用して生成されるファイルに書き出します。
この時書き出すpackage commentを`//go:build`コメントのみになるようにフィルターします。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/codegen/comment.go#L14-L25

[go/build/constraint.IsGoBuild](https://pkg.go.dev/go/build/constraint@go1.23.4#IsGoBuild), [go/build/constraint.IsPlusBuild](https://pkg.go.dev/go/build/constraint@go1.23.4#IsPlusBuild)が定義されているのでこれをそのまま使います。

`IsPlusBuild`は`// +build`から始まる行に対してtrueを返します。これが`Go1.16`かそれ以前までに使われていた形式で、`Go1.17`でdeprecation、`Go1.18`から基本的に削除されているはずですが、一応簡単にサポートできるのでベストエフォートで対応してあります。後述する`Custom Handler`で`maps.Clone`を使用するため、暗黙的に`Go1.21`以上がこのcode generatorの対象バージョンとなっています。そのためサポートする必要自体はないと思っています。

こうしてtrimされたpackage-commentを`PrintFileHeader`内でprintしていきます。
この関数は出力されるファイルすべてに対して呼ばれるprinterでpackage comment, package clauseとimport declをすべて出力するものです。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/codegen/print.go#L251-L290

`printer.Fprint`がコメントだけとかそういうレベルのprintに対応していないのでちょっと頑張って出力しています。

### ファイルのsuffixがbuild constraintであるときはコピーする

`Go`はファイル名が`_test` suffixを除いたときに`_os`, `_arch`, `_os_arch`でsuffixされている場合暗黙的なbuild constraintsとして取り扱います。
サポートされるos, archの一覧は`go tool dist list -json`で得られます。

:::details go tool dist list -jsonの出力

```
# go tool dist list -json
[
        {
                "GOOS": "aix",
                "GOARCH": "ppc64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "android",
                "GOARCH": "386",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "android",
                "GOARCH": "amd64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "android",
                "GOARCH": "arm",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "android",
                "GOARCH": "arm64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "darwin",
                "GOARCH": "amd64",
                "CgoSupported": true,
                "FirstClass": true
        },
        {
                "GOOS": "darwin",
                "GOARCH": "arm64",
                "CgoSupported": true,
                "FirstClass": true
        },
        {
                "GOOS": "dragonfly",
                "GOARCH": "amd64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "freebsd",
                "GOARCH": "386",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "freebsd",
                "GOARCH": "amd64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "freebsd",
                "GOARCH": "arm",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "freebsd",
                "GOARCH": "arm64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "freebsd",
                "GOARCH": "riscv64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "illumos",
                "GOARCH": "amd64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "ios",
                "GOARCH": "amd64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "ios",
                "GOARCH": "arm64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "js",
                "GOARCH": "wasm",
                "CgoSupported": false,
                "FirstClass": false
        },
        {
                "GOOS": "linux",
                "GOARCH": "386",
                "CgoSupported": true,
                "FirstClass": true
        },
        {
                "GOOS": "linux",
                "GOARCH": "amd64",
                "CgoSupported": true,
                "FirstClass": true
        },
        {
                "GOOS": "linux",
                "GOARCH": "arm",
                "CgoSupported": true,
                "FirstClass": true
        },
        {
                "GOOS": "linux",
                "GOARCH": "arm64",
                "CgoSupported": true,
                "FirstClass": true
        },
        {
                "GOOS": "linux",
                "GOARCH": "loong64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "linux",
                "GOARCH": "mips",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "linux",
                "GOARCH": "mips64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "linux",
                "GOARCH": "mips64le",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "linux",
                "GOARCH": "mipsle",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "linux",
                "GOARCH": "ppc64",
                "CgoSupported": false,
                "FirstClass": false
        },
        {
                "GOOS": "linux",
                "GOARCH": "ppc64le",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "linux",
                "GOARCH": "riscv64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "linux",
                "GOARCH": "s390x",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "netbsd",
                "GOARCH": "386",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "netbsd",
                "GOARCH": "amd64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "netbsd",
                "GOARCH": "arm",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "netbsd",
                "GOARCH": "arm64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "openbsd",
                "GOARCH": "386",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "openbsd",
                "GOARCH": "amd64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "openbsd",
                "GOARCH": "arm",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "openbsd",
                "GOARCH": "arm64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "openbsd",
                "GOARCH": "ppc64",
                "CgoSupported": false,
                "FirstClass": false
        },
        {
                "GOOS": "openbsd",
                "GOARCH": "riscv64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "plan9",
                "GOARCH": "386",
                "CgoSupported": false,
                "FirstClass": false
        },
        {
                "GOOS": "plan9",
                "GOARCH": "amd64",
                "CgoSupported": false,
                "FirstClass": false
        },
        {
                "GOOS": "plan9",
                "GOARCH": "arm",
                "CgoSupported": false,
                "FirstClass": false
        },
        {
                "GOOS": "solaris",
                "GOARCH": "amd64",
                "CgoSupported": true,
                "FirstClass": false
        },
        {
                "GOOS": "wasip1",
                "GOARCH": "wasm",
                "CgoSupported": false,
                "FirstClass": false
        },
        {
                "GOOS": "windows",
                "GOARCH": "386",
                "CgoSupported": true,
                "FirstClass": true
        },
        {
                "GOOS": "windows",
                "GOARCH": "amd64",
                "CgoSupported": true,
                "FirstClass": true
        },
        {
                "GOOS": "windows",
                "GOARCH": "arm",
                "CgoSupported": false,
                "FirstClass": false
        },
        {
                "GOOS": "windows",
                "GOARCH": "arm64",
                "CgoSupported": true,
                "FirstClass": false
        }
]
```

:::

`JSON`は内容から以下のように定義できるので

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/suffixwriter/suffix.go#L17-L22

コマンドを呼び出して`json.Unmarshal`します

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/suffixwriter/suffix.go#L101-L111

`json.Unmarshal`の結果を`map[string]bool`に集めます

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/suffixwriter/suffix.go#L76-L99

`go`コマンドが入っていない環境は全く想定していませんが、一応ない場合はソースに埋めておいたものにfallbackするようにしてあります。(`go`コマンドがない環境では[golang.org/x/tools/go/packages]`.Load`が動作しない)

前述通りファイルの出力の際には`suffixwirter`で`.cloner`のようなsuffixを加えてファイル名に書き込みを行いますが、ここをbuild constraintsとなるsuffixが元のファイル名についている場合はsuffixの後ろに移動させるように考慮を加えます。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/suffixwriter/suffix.go#L113-L126

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/suffixwriter/suffix.go#L135-L172

とりあえず書いてテストが通る程度なので見てすぐわかる程度に非効率なコードですが当面はこうでいいとしています。

## Custom handler

cloneの処理はユーザーごとに別のものを与えたかったりすることは十分に想定できます。
そこで、Custom handlerを渡せるようにします。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/custom_handler.go#L38-L48

- `Matcher`がtrueを返す時にこのcustom handlerを実行します
  - 例えば`[]map[string]*[5]T`がstruct fieldの型であるとき、`Matcher`は`[]map[string]*[5]T`, `map[string]*[5]T`, `*[5]T`, `[5]T`, `T`を引数に何度も呼ばれます。
- `Imports`でこのcustom handlerが使用する外部パッケージを指定します。
- `Expr`で各種データを受けとって`func(s string) string`を返します。`s`はclone処理の引数にすべき変数の変数名です。`isFunc`がtrueのとき、返された`expr`は呼び出し可能な関数ですので、clonerはこれを呼び出すようなコードを生成します。

`CustomHandlerExprData`の`ImportMap`は`types.Qualifier`になれたりするようなものです。`types.TypeString`とともに使ってもよいようにしてあります。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/imports/parser.go#L389-L399

```go
typeExpr := types.TypeString(data.Ty, data.ImportMap.Qualifier(data.PkgPath))
```

で、マッチした型のテキスト表現が得られます(e.g. `foo.Bar[string]`)。もはや`CustomHandlerExprData`にあらかじめ評価済みのものを渡しておいてもいい気がしますが、custom handlerの実装によっては全くいらないことも多いのでやめておきました。

[matcherの定義](#matcherの定義)の項ですでに見せていますが、例えば`[]map[string]*[5]T`があるとき、`[5]T`にマッチするcustom handlerを提供すると、`[]map[string]*`までのfield unwrapperが出力され、`[5]T`に対してcustom handlerを呼び出します。

いくつかbuilt-inのcustom handlerを定義しておいています。

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/custom_handler.go#L50-L253

例えば、`[n]T`を単なる代入に変換するcustom handlerが適用すると以下のコードを生成した部分が

```go
var inner [5]string
for k, v := range /* [5]string */ v {
    inner[k] = v
}
```

以下のようになります。

```go
inner = v
```

現時点で以下のbuilt-in custom handlerが定義されています

- `T`がclone-by-assignなとき、`[]T`に対して`copy`ベースのcloneを実行する
  - forを回すより最適化された処理をするようなことを読んだのでやってますが特に確証はありません。
  - する意味なかったら泣きながら消します。
- `V`がclone-by-assignなとき、`map[K]V`に対して`maps.Clone`を呼び出す
  - これもforを回すより最適化された処理を呼び出していたのでこうしています。
- `time.Time`に対して`time.Date(v.Year(), v.Month(), v.Day(), v.Hour(), v.Minute(), v.Second(), v.Nanosecond(), v.Location())`を呼び出す
  - monotonic timer以外のコピーです。
  - 単なる代入だとmonotonic timerがコピーされて困ります。されたほうがいいかも？要今後の検討ですね
- `*big.Int`, `*big.Rat`, `*big.Float`に対してコピー処理を呼び出す。
  - `math/big`をあまり使わないので詳しくないですが、`big.New*`して`Set`を呼び出さないとコピーされない作りになっています
  - これらは内部的に`[]Word`を持つので暗黙的に`pointer`を持っています。
- `xml.Token`に対して`xml.CopyToken`を呼び出す。
  - `xml.Token`はinterfaceで、`[]byte`であることがありうるのでコピー必須です。
- basic type、もしくは既知の`clone-by-assign`、もしくはそれらのarrayに対して単なる代入を行う。
  - pointerを一切含まない型に関しては機械的に列挙可能なのでしておきました([これ](https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/clone_by_assign_types_std.generated.go))
  - [unique.Handle](https://pkg.go.dev/unique@go1.23.4#Handle)や[\*time.Location](https://pkg.go.dev/time@go1.23.4#Location)など、定義上、APIでの取り扱い上内部に`pointer`を含んでいてもそのまま代入すればいいものは目で確認しながらリスト化していってます。([ここ](https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/clone_by_assign_types_known.go))

やる気がでたら拡充します。

## cliとして呼び出せるようにする

[github.com/spf13/cobra](https://github.com/spf13/cobra)を使ってサブコマンドとして呼び出せるようにしてあります。

```
# go run github.com/ngicks/go-codegen/codegen@6a0b75516f057f51967eb566eaf255890f975192 cloner --help
cloner generates clone methods on target types.

cloner command generates 2 kinds of clone methods

1) Clone() for non-generic types
2) CloneFunc() for generic types.

CloneFunc requires clone function for each type parameters.

Example:

func (c C[T, U]) CloneFunc(cloneT func(T) T, cloneU func(U) U) C[T, U] {
        // ...
}

The cloner sub command, as other commands do, loads and parses Go source code files
by using "golang.org/x/tools/go/packages".Load
then it examines types defined in them whether if they are clone-able or not.
Multiple packages can be loaded and processed at once.
The type dependency chain is allowed to span across multiple packages and generated Clone method considers it.

The specified package path must be relative to the cwd, which can be changed by --dir option,
to limit the target packages to which the process can write generated code safely.

The clone-able is defines as
1) A struct type which has at least a field of
1-1) basic types or pointer of basic types
1-2) array, slice or map of 1-1).
1-3) channel, noCopy object (types with the Lock method, e.g. sync.Mutex or sync.Locker), func type when the configuration allows copying each of them.
1-4) a type that implements Clone or CloneFunc method
1-5) other 1) types.
2) A named type whose underlying type is array, slice or map of 1), basic types or pointer of basic type.

A field of deeply nested type, for example, []*[5]map[int]string is still considered as clone-able,
since bottom type, string, is a basic therefore clone-able type.
We call parts other than that ([]*[5]map[int]) _route_. And each element of them as _route node_
(i.e. for []*[5]map[int]string, route nodes are slice, pointer, map, in the order. The bottom type is string.)
Only disallowed _route node_ is interface literal. They are ignored silently.

The cloner sub command also allows per-field basis configuration by writing comments associated to it.
For example:

type Foo struct {
        //cloner:copyptr
        NoCopy *sync.Mutex
}

the cloner command generates

func (v Foo) Clone() Foo {
        return Foo{
                NoCopy: v.NoCopy,
        }
}

Without the comment, the cloner command ignores the type Foo since it has no clone-able fields other than that.

Usage:
  codegen cloner [flags] --pkg ./

Flags:
      --build-flags strings    a comma separated list of command-line flags to be passed through to the build system's query tool.
      --chan-copy             sets global option that copies channel fields
      --chan-disallow         sets global option that disallows channel fields
      --chan-ignore           sets global option that ignores channel fields.
      --chan-make             sets global option that makes new channel. Clone methods also copy the capacity of input channels.
  -d, --dir string            specifies the current working directory for the source code loader and other tools.
                              The path specified by --pkg flag is evaluated under this directory.
                              If empty cwd will be used.
      --dry                   enables dry run mode. any files will not be removed nor generated.
      --func-copy             sets global option that copies func fields
      --func-disallow         sets global option that disallow func fields.
      --func-ignore           sets global option that ignores func fields. func literal or named function type.
  -h, --help                  help for cloner
      --ignore-generated      You do not need this option.
                              If set, the type checker ignores ast nodes with comment //codegen:generated attached.
                              Useful for internal debugging.
      --interface-copy        sets global option that copies interface fields
      --interface-ignore      sets global option that ignores interface fields. func literal or named function type.
      --no-copy-copy          sets global option that copy pointer of no-copy object. Clone methods copy no-copy object if and only if field is pointer type.
      --no-copy-disallow      sets global option that disallow no-copy object. Types that contain no-copy type fields are not generation target.
      --no-copy-ignore        sets global option that ignores no-copy object. Clone methods just simply leave fields zero value.
  -p, --pkg ./...             [required] target package pattern. relative to dir. must start with "./". can be ./...
  -v, --verbose               verbose logs
```

[matcherの定義](#matcherの定義)で見せた`MatcherConfig`の各項目がcli flagとしてカスタマイズできるようになっています。説明がわかりにくい気がする・・・この辺は今後の改善事項ですね。

## 生成結果のexample

以下にexample typeとその生成結果が出力されます。

https://github.com/ngicks/go-codegen/tree/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/internal/testtargets

tree構造のcloneのexampleとか

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/internal/testtargets/tree/tree.go

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/internal/testtargets/tree/tree.clone.go

(exampleなのに本当に動作するbinary treeの実装を書いちゃった)

struct literalを含む型に対するexampleとか

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/internal/testtargets/structlit/lit.go

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/internal/testtargets/structlit/lit.clone.go

type aliasを含む場合のexampleとか

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/internal/testtargets/alias/alias.go

https://github.com/ngicks/go-codegen/blob/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/internal/testtargets/alias/alias.clone.go

build constraintsを含む場合のexampleとか

https://github.com/ngicks/go-codegen/tree/6a0b75516f057f51967eb566eaf255890f975192/codegen/generator/cloner/internal/testtargets/constraint

色々用意してあります。

## 今後

- prefer-slices-cloneオプションの追加
  - capの正確なコピーができない反面`slices.Clone`のほうがzero valueでの初期化を挟まない分パフォーマンスがよいです
  - capの正確なコピーが必要ないケースのほうが多そうなのでこちらがデフォルトになるかもしれないです。
- known clone by assignの拡充
  - まだ洗いきっていないstd libのpackageがあるので全部見る。
- overlayオプション
  - in-placeオプション(型定義やstruct fieldにコメントをつけて行う設定)と同じことを外部データからもできるようにする。
    - フィールド単位の無視とか、コピー方法の指定とかです。
  - 他のcode generatorによって生成された型にコメントをつけて回るのは現実的にしたくない運用だからそこをカバーしに行くためです。
    - [github.com/oapi-codegen/oapi-codegen]の生成するコードにさらに`Clone`を生成してみて、server interfaceとかに不要なのにcloneを生成して困っています。
- in-placeオプションの拡充
  - フィールドレベルでclonerの関数を指定したり
  - フィールドを無視させたり、単なるassignに変えたり
- templateによるcustom handlerの受付
  - 現状custom handlerはgoのプログラムとして呼び出す場合にのみ渡せますが、これをcli経由でも渡せるように整備するということです。
  - `text/template`で解釈できるテキストとしてcustom handlerを定義できればよいわけです。
  - `text/template`はテキストから呼び出せる関数を任意に定義可能なのでなんでもできるんですが、
  - それはそれとして1度もそういうことをしたことがないのでノウハウがないため大変ですね。
- 型がすでに`Clone`を実装していてそれが`cloner`が生成したものでないときは生成対象から外す。
- method receiverをほかのmethod receiverと同じものにする
  - 現状問答無用で`func(v Type) Clone() Type`と`v`をreceiverにしたmethodを生成しますが、このreceiverはgodoc上表示されるので見栄えが悪い
- ドキュメントを整備する
  - 現状これらのサブコマンドの説明がgit repositoryのトップに全然なくて誰も把握できてないと思います。
  - 誰が読むのかもわからないドキュメントを読みやすく整備するのは精神との戦いだったりしますね
  - そもそもぱっと読んでわかるドキュメントを備えたOSSってのが珍しくて、ソースを読んでようやくなるほどなってなることが多いです。多分大変なことなんですよね。
- [前回の記事の##おわりに](https://zenn.dev/ngicks/articles/go-code-generation-from-ast-and-type-info#%E3%81%8A%E3%82%8F%E3%82%8A%E3%81%AB)で触れた別のcode generatorを作っていく。
  - そこに上げている`wrapper`の必要性を日々感じていて困っているのでいったん切り上げて別テーマに行くのもありですね。

## おまけ: go/ast, go/typesお気をつけポインツ

### ast

#### type declのunderlying typeは`()`で囲むことができる

[The Go Programming Language SpecificationのTypesのところ](https://go.dev/ref/spec#Types)をよーく見たらわかるんですが、以下は合法です。

[playground](https://go.dev/play/p/8jwQW6DiA-g)

```go
type B (struct{})
```

これは見たことがない。ただ安易に[*ast.TypeSpec]を`typeSpec.Type.(*ast.StructType)`として判定を行うと、合法なソースコードが処理できないというissueが上がってくることもありうる・・・かもしれないですね。

一応考慮に入れるなら以下のようにすればよいでしょう。

[playground](https://go.dev/play/p/VYrWT_mDiMo)

```go
package main

import (
    "fmt"
    "go/ast"
    "go/parser"
    "go/token"
)

var src = `package foo

type A (((struct{})))
`

func main() {
    fset := token.NewFileSet()
    file, err := parser.ParseFile(fset, "foo.go", src, parser.ParseComments|parser.AllErrors)
    if err != nil {
        panic(err)
    }

    var ts *ast.TypeSpec

SEARCH:
    for _, dec := range file.Decls {
        genDecl, ok := dec.(*ast.GenDecl)
        if !ok {
            continue
        }
        if genDecl.Tok != token.TYPE {
            continue
        }
        for _, spec := range genDecl.Specs {
            ts = spec.(*ast.TypeSpec)
            break SEARCH
        }

    }
    handleStructType(ts)
}

func handleStructType(ts *ast.TypeSpec) {
    unwrapped := ts.Type

    var loopCount int
    const maxDepth = 100
    for {
        if loopCount >= maxDepth {
            panic("too deep parenthesis")
        }
        paren, ok := unwrapped.(*ast.ParenExpr)
        if !ok {
            break
        }
        unwrapped = paren.X
        loopCount++
    }
    st, ok := unwrapped.(*ast.StructType)
    if !ok {
        // handle
        return
    }
    fmt.Printf("structType = %v\n", st)
    // structType = &{24 0xc0000a81e0 false}
}
```

構文ルール上、`type A ((((struct {}))))`みたいにいくつもparenthesisがネストするのは許されています。
筆者環境では`gofumpt`によるフォーマットをかけていますがこれだと冗長なParenthesis(`()`) は削除されて一つになります。

システム外部から入力を受け付けるケースではparenthesisの深さに上限をつけないと`DoS`を可能にしてしまいます。
`Go`のソースコードを外部から受け付けるシステムがあるかは置いておいて、ですが。

#### \*ast.StructTypeのFieldは0個もしくは複数個のNamesを持つ。

`Go`のstructはは複数のFieldを一行に掛けますよね。

[\*ast.Field](https://pkg.go.dev/go/ast@go1.23.4#Field)の定義より、1つの`*ast.Field`は複数の`Names`を持つことがあります。

[*types.Struct]にはn番目のfieldを取得するAPIしかないため、astをベースに型情報との連携を行うとき、fieldの列挙が不正確だと不整合を起します。

```go
type A struct {
    Foo, Bar, Baz string
}
```

上記の`Foo`,`Bar`, `Baz`は1つの`*ast.Field`で表現されます。

さらに、この`Names`は0個の時もあります。以下のようにstruct embeddingが行われているときです。

```go
type A struct {
    Embedded
}
```

この時、`go/types`で定義される型情報上では`Embedded`という名前の1個のfieldとして取り扱われます。

すべてのFieldの名前を列挙するには以下のように定義するとよいでしょう。

```go
type FieldDescAst struct {
    Pos   int
    Name  string
    Field *ast.Field
}

func FieldAst(st *ast.StructType) iter.Seq[FieldDescAst] {
    return func(yield func(FieldDescAst) bool) {
        if st.Fields == nil || len(st.Fields.List) == 0 {
            return
        }
        pos := 0
        for i := 0; i < len(st.Fields.List); i++ {
            f := st.Fields.List[i]
            names := f.Names
            if len(names) == 0 {
                // embedded field
                unwrapped := f.Type
                var name string
            UNWRAP:
                for {
                    switch x := unwrapped.(type) {
                    default:
                        panic(fmt.Errorf("unknown type in an anonymous field: %T in %v", unwrapped, f.Type))
                    case *ast.Ident:
                        name = x.Name
                        break UNWRAP
                    case *ast.StarExpr:
                        unwrapped = x.X
                    case *ast.SelectorExpr:
                        unwrapped = x.Sel
                    case *ast.IndexExpr: // type param
                        unwrapped = x.X
                    case *ast.IndexListExpr:
                        unwrapped = x.X
                    }
                }
                if !yield(FieldDescAst{pos, name, f}) {
                    return
                }
                pos++
            } else {
                for _, name := range names {
                    if !yield(FieldDescAst{pos, name.Name, f}) {
                        return
                    }
                    pos++
                }
            }
        }
    }
}
```

[EmbeddedFieldのspec](https://go.dev/ref/spec#EmbeddedField)よりembedできるのはnamed typeかそれへのpointerのみです。type paramとtype paramへのポインターはembedできません。

- 型の名前となるidentifierは`*ast.Ident`
- ポインターは`*ast.StarExpr`
- `X.Sel`(=外部packageの型)は`*ast.SelectorExpr`
- `X[Index]`(=type paramが1つのgeneric type)は`*ast.IndexExpr`
- `X[Index1, Index2,...]`(=type paramが複数のgeneric type)は`*ast.IndexListExpr`

となります。

### types

#### unaliasしよう

[The Go Blog: What's in an (Alias) Name?](https://go.dev/blog/alias-names)より、`Go`では名前をaliasすることができます。
`Go 1.24`からaliasがtype paramを持てるようになります。

```go
type Foo[T, U comparable] struct{}

type (
    Foo2 = Foo[int, string]
    // Go 1.23かそれ以前だと非合法
    Foo3[T interface{ ~int | ~uint }] = Foo[int, T]
)
```

厄介なことに、type aliasはsliceなどの無名の型を含むことができます。

```go
type A = []B
type B = map[string]C
type C = chan D
type D struct {
    // ...
}
type E = struct{ Foo string }
type F = interface{ Look() }
```

struct literalやinterface literalにもaliasを行うことができます。

[types.Unalias](https://pkg.go.dev/go/types@go1.23.4#Unalias)を使うことで、aliasされていない方を取り出すことができます。

[playground](https://go.dev/play/p/HSE-CZrz943)

```go
package main

import (
    "fmt"
    "go/ast"
    "go/importer"
    "go/parser"
    "go/token"
    "go/types"
)

func parseStringSource(src string) (*token.FileSet, *ast.File, *types.Package) {
    fset := token.NewFileSet()
    f, err := parser.ParseFile(fset, "hello.go", src, parser.ParseComments|parser.AllErrors)
    if err != nil {
        panic(err)
    }
    conf := &types.Config{
        Importer: importer.Default(),
    }
    pkg := types.NewPackage("hello", "main")
    chk := types.NewChecker(conf, fset, pkg, nil)
    err = chk.Files([]*ast.File{f})
    if err != nil {
        panic(err)
    }
    return fset, f, pkg
}

func main() {
    _, _, pkg := parseStringSource(`package main

type A = B
type B = C
type C = D
type D struct {
    // ...
}

type AA = [][]B
`)

    a := pkg.Scope().Lookup("A")
    aa := pkg.Scope().Lookup("AA")
    fmt.Printf("type name = %s\n", types.TypeString(a.Type(), func(p *types.Package) string { return "" }))
    // type name = A
    fmt.Printf("unaliased A = %s\n", types.TypeString(types.Unalias(a.Type()), func(p *types.Package) string { return "" }))
    // unaliased A = D
    fmt.Printf("unaliased AA = %s\n", types.TypeString(types.Unalias(aa.Type()), func(p *types.Package) string { return "" }))
    // unaliased AA = [][]B
}
```

上記からわかる通り、最初のalias以外の型までunaliasしてくれます。

型を見たらとりあえず`types.Unalias`するぐらいの勢いでよいと思います。

#### type asserterを定義しよう

例えば以下のようなstructがあるとします。

```go
type A struct {
    F1 func()
    F2 Fn
}

type Fn func()
```

`F1`の型は[*types.Signature], `F2`の方は[*types.Named]です。
どちらも特定のsignatureを満たす`func`なわけですから、同じように取り扱いたいときはよくあると思います。

そこで以下のようなhelperを定義しておくと便利だと気づきました。

```go
func asUnderlying[T types.Type](ty types.Type) T {
    ty = types.Unalias(ty)
    t, ok := ty.(T)
    if ok {
        return t
    }
    named, ok := ty.(*types.Named)
    if !ok {
        // should be nil
        return *new(T)
    }
    ty = types.Unalias(named.Underlying())
    t, _ = ty.(T)
    return t
}
```

aliasingを無視し、元となっている型が`T`であるか、`T`がnamed typeならそのunderlying typeが`T`であるかをチェックします。

## 感想

そこそこ実用に耐えるレベルにはなってきたかな～？って感じですね。すぐできると思ったんですが、がっつりやってたのに1か月ぐらいかかっちゃいましたね。
あとは使ってみて荒を洗い出すところですね。とりあえず1回使ってみてtype aliasで`type A = []B`みたいな型があると動作しないのに気付いてから慌てて直したりしました。

他のcode generatorが吐いた型に対してさらに`Clone` methodを生成するという前からやりたかったことができるようになったことで、生活がほんの少し楽になりました。
他のcode generatorというのは[github.com/oapi-codegen/oapi-codegen]なんですが、時と場合によってstruct literalを含む型が吐かれたり、type aliasであれこれやられたり`*map[K]V`が吐かれたり、手で書いてたらめったに見ないことのオンパレードになってハードルが上がりまくるので大変でした。おかげでコーナーケースがある程度潰せていると思います。

[前回の記事](https://zenn.dev/ngicks/articles/go-code-generation-from-ast-and-type-info)で作成した`undgen`に次いで、結構な要素を使いまわした別のcode generatorを作ったわけですが、こうして別のものを作ってみるとどこが使いまわせてどこが全然関係ないところなのかが見えてきて頭の中がまとまるので結構面白いです。特に`typegraph`周りには`undgen`固有の残骸が残ってて、全く使いまわせないのに不適切なところに不適切なものを定義してしまっていることに気付きました。ここから広げた枝を改めて刈り込む工程があります。

これまで型情報をいじっていろいろやってきましたが、function body内の型情報とそれを使ったrewriteに関してはなにもやってこなかったのでそちらにも進んでいきたいと思っています。
さらにvscode/vimと連携してエディターからそれらの機能を呼び出せるようにするとか、やりたいネタはまだまだあります。

[Go]: https://go.dev/
[Go1.18]: https://tip.golang.org/doc/go1.18
[Go1.23]: https://tip.golang.org/doc/go1.23
[GoのJSONのT | null | undefinedは\[\]Option\[T\]で表現できる]: https://zenn.dev/ngicks/articles/go-json-undefined-or-null-slice
[go/ast]: https://pkg.go.dev/go/ast@go1.23.4
[go/types]: https://pkg.go.dev/go/types@go1.23.4
[*ast.ArrayType]: https://pkg.go.dev/go/ast@go1.23.4#ArrayType
[*ast.MapType]: https://pkg.go.dev/go/ast@go1.23.4#MapType
[*ast.ChanType]: https://pkg.go.dev/go/ast@go1.23.4#ChanType
[*types.Array]: https://pkg.go.dev/go/types@go1.23.4#Array
[*types.Slice]: https://pkg.go.dev/go/types@go1.23.4#Slice
[*types.Map]: https://pkg.go.dev/go/types@go1.23.4#Map
[*types.Chan]: https://pkg.go.dev/go/types@go1.23.4#Chan
[*types.Struct]: https://pkg.go.dev/go/types@go1.23.4#Struct
[*types.Named]: https://pkg.go.dev/go/types@go1.23.4#Named
[*types.Signature]: https://pkg.go.dev/go/types@go1.23.4#Signature
[*types.Alias]: https://pkg.go.dev/go/types@go1.23.4#Alias
[types.Object]: https://pkg.go.dev/go/types@go1.23.4#Object
[types.Type]: https://pkg.go.dev/go/types@go1.23.4#Type
[*types.Info]: https://pkg.go.dev/go/types@go1.23.4#Info
[types.NewMethodSet]: https://pkg.go.dev/go/types@go1.23.4#NewMethodSet
[*ast.File]: https://pkg.go.dev/go/ast@go1.23.4#File
[*ast.TypeSpec]: https://pkg.go.dev/go/ast@go1.23.4#TypeSpec
[golang.org/x/tools/go/packages]: https://pkg.go.dev/golang.org/x/tools@v0.28.0/go/packages
[github.com/dave/dst]: https://github.com/dave/dst
[github.com/oapi-codegen/oapi-codegen]: https://github.com/oapi-codegen/oapi-codegen
[github.com/ngicks/und]: https://github.com/ngicks/und
[sliceund.Und]: https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha6/sliceund#Und
[sliceelastic.Elastic]: https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha6/sliceund/elastic#Elastic
[Elasticsearch]: https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro.html
[github.com/go-json-experiment/json]: https://github.com/go-json-experiment/json
[*bufio.Writer]: https://pkg.go.dev/bufio@go1.23.4#Writer
[fmt.Fprintf]: https://pkg.go.dev/fmt@go1.23.4#Fprintf
[text/template]: https://pkg.go.dev/text/template@go1.23.4
[github.com/dave/jennifer]: https://github.com/dave/jennifer
