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
- 実装の方式、考慮事項
  - [golang.org/x/tools/go/packages]による`ast`, `type information`の読み込み
  - 生成対象の型依存関係のグラフ化、グラフを逆にたどることで生成対象の型を列挙する。
  - 型情報を使った各型の判別
    - `Clone`を実装する型(_implementor_)か
    - `NoCopy`(assignによってコピーすると`go vet`が`copies lock value`で怒る型)か
    - assignによってcloneできる型か
  - など

対象読者のレベル感を会社の同僚に置くため基本的な概念の説明を多く含めようと考えています。([A Tour of Go](https://go.dev/tour/list)は最低限こなしている)
そのため`Go`やcomputer-scienceに長じた読者はある程度とばしながら読んでいただければと思います。

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
    github.com/ngicks/go-iterator-helper v0.0.16
    github.com/ngicks/und v1.0.0-alpha6
    github.com/spf13/cobra v1.8.1
    github.com/spf13/pflag v1.0.5
    golang.org/x/tools v0.28.0
    gotest.tools/v3 v3.5.1
)
```

`github.com/ngicks`から始まるgo moduleは自作のものです。これらはあんまり気にしないでください。

`Go 1.24`から[generic type alias](https://tip.golang.org/doc/go1.24#language)が導入されますが、この記事はこれを全く考慮していません。

## Rationale: どうしてcode generatorを実装するする必要があるか？

### 型とassignによるコピー

他のプログラミング言語でもそうである通り、`Go`ではassign(`a = b`)によって値のコピーが起こります。
`int`, `string`, `bool`などのプリミティブな型は単純なassignのみで情報をコピーできるため、例えば`c := a`とした後、`c`への変更は`a`に影響しません(_and vice versa_)。
(ただし、`Go`以外では、言語によっては`string`がimmutableでないことがあるので気を付ける)

さらに、`struct`, `array`などが単にこれらのprimitiveな型のみを含む場合でもassignによってコピーが起きます。

```go
type sample struct { foo int; bar string }
s1 := sample{bar: "foo"}
s2 := s1
s2.bar = "bar"
fmt.Printf("s1.bar=%q, s2.bar=%q\n", s1.bar, s2.bar)
// s1.bar="foo", s2.bar="bar"
```

ただし、これらの型が[pointer](https://go.dev/tour/moretypes/1)(`*T`)であるときや、pointerを含む型の場合は話が異なってきます。
それもそのはずで、assignによるコピーは、pointerが指し示すアドレス値をコピーしているような形になるためです。
pointerが指し示した値自体はコピーしないため、pointerをassignでコピーした値は参照先を共有してしまいます。
pointerは(リンク先の`A Tour of Go`で説明されている通り)メモリー上のある位置を指し示すアドレスの値です。いってしまえば`uint`です。
`*T`は実際上は`uint`なのだからassignによって`uint`の値がコピーされます。`uint`が指し示したさきの値をコピーしないことはこう考えると当たり前に思えるはずです。

pointerは[dereference](https://go.dev/ref/spec#Address_operators)することでassignによるコピーが行うことができます

```go
num   /*  int */ := 10
nump  /* *int */ := &num
deref /*  int */ := *nump
num = 12
fmt.Printf("num=%d\nnump=%d\nderef=%d\n", num, *nump, deref)
// num=12
// ump=12
// deref=10
```

`Go`では、ご存じのとおり`type A B`によって新しい型を定義できます。ここで`A`をtype nameなどと言い、これが新しく定義される型の名前です。`A`は名前のある型なので、named typeなどと型システム(`go/types`)上では呼ばれます。
`B`は[underlying type](https://go.dev/ref/spec#Underlying_types)と呼ばれ、これが`A`のデータ構造となります。
`B`には`struct`で新しいデータセットを指定したり、 `pointer`(`*T`), `array`(`[n]T`), `slice`(`[]T`), `map`(`map[K]V`), `channel`(`chan T`)のような組み込み型を指定することができます。
`B`には別のnamed typeを指定することもできます。この場合`A`は`B`のメソッドセットを除いて、データ構造だけを継承します。`Go`では`B(A)`で構造の同じ型同士を変換できますから、`B`のメソッドを使用したい場合は明示的に型を変換します。

`struct`はfieldに別の型のpointer(e.g. `*int`)を指定できますし、`slice`, `map`, `channel`は暗黙的に内部でpointerを持ちます。
そのため、それらを`underlying type`とする型は、値そのものがnon-pointerであってもその値の型がpointerを含むためassignではコピーできないことがあります。
pointerはderefしてassignしてindirectしなおしてコピーされた値を指し示すpointerを得なければ*clone*することができません。

`Go`では、ident(identifier=変数名や関数名のような識別子)の先頭がunicode upper caseになっているものがexport(=それを定義したパッケージ以外からアクセスできる)されるという公開性のルールがあります。そのため、pointerを含む型がその部分をexportしない場合はコピーすることは困難となります。

ある値へのアクセスが他の値に影響しないでほしいことはよくあります。
例えばプログラム実行時のときどきの状態を保存して比較したいときや、[data race](https://go.dev/doc/articles/race_detector)を防ぎながら同じデータを複数のgoroutineに渡して処理したいときなどです。

### deep cloneを実現する方法

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

表現力の限界の例は`JSON`にはポインターに当たる機能がないというものがあります。そのため`JSON`では`ring buffer`のような循環構造を表現できません。
循環構造の表現が必須であれば`YAML`を用いるのが一つの解決方法かもしれませんん。`YAML`には[anchor](https://yaml.org/spec/1.2.2/#anchors-and-aliases)仕様があります。

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

例としては以下のようになります。ただし、この例は完全な実装ではなく、かなり雑なものであるので雰囲気がわかる以上のものではないことを留意してください。

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

以下などで`reflect`ベースの`deep clone`が実装されています。

- https://github.com/jinzhu/copier
- https://github.com/ulule/deepcopier
- https://github.com/mitchellh/copystructure (Public archive)
  - [github.com/compose-spec/compose-go](https://github.com/compose-spec/compose-go)がかつて短い期間これを用いてdeep copyを実現していましたが、今見ると[27c7848d662](https://github.com/compose-spec/compose-go/commit/27c7848d662558f4f8b37a4e7a8183f55264abf0)で依存性から取り除かれていました。

#### code generatorによってdeep clonerを実装する

[The Go Blog: Generating code](https://go.dev/blog/generate)などからわかる通り、`Go`にはcode generatorを呼び出すための`//go:generate` directiveが存在します。
([directive](https://tip.golang.org/doc/comment#syntax)というのは`//`のあとに` `(space)を含まないコメントのことです。これは`go doc`などに表示されなくなるような特別扱いがあります。[ここ](https://pkg.go.dev/cmd/compile#hdr-Compiler_Directives)などを参照。)
このdirective commentが書きこまれたgo source codeを指定して`go generate ./path/to/file.go`を実行すると、`//go:generate`の後に書かれたコマンドが実行される仕組みになっています。

`Go`自身も`//go:generate`を活用しており、

https://github.com/search?q=repo%3Agolang%2Fgo%20%2F%2Fgo%3Agenerate&type=code

と多用されていることがわかります。

このように、`Go`にとってcode generatorを使用するのは一般的なことです。

そもそもマクロが現状(`Go1.24`時点)ありませんし、それに類するツールもありません。[Go1.18]までgenericsもなかったため、code generatorを使わざるを得ないことも多くありました。
この辺の話は[昔の記事: "Goのcode generatorの作り方: 諸注意とtext/templateの使い方"の"Rationale: なぜGoでcode generationが必要なのか"](https://zenn.dev/ngicks/articles/go-code-generation-in-ways-text-template#rationale%3A-%E3%81%AA%E3%81%9Cgo%E3%81%A7code-generation%E3%81%8C%E5%BF%85%E8%A6%81%E3%81%AA%E3%81%AE%E3%81%8B)で述べましたので併せて読むといいかもしれません。

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
生成された`CloneFunc`がさらに別の型の`CloneFunc`を呼び出す場合、`cloneT`などのコールバックを受け渡してうまくcloneを行っていきます。

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
- 生成時に読み込んだパッケージ群外で生成されたnamed typeのclone方法は全く感知しません。代わりに特定の型にマッチすると指定したclone functionを呼び出すカスタムハンドラーのサポートをするものとします。これによって外部の型のcloneも可能にします。
  - 今後の拡張でexportされたfieldしか持たない型に関してはコピーを行うコードを吐くように変えるかもしれません。現状は放置です。

## code generator実装の基本方針

ここからは[前回の記事: \[Go\]ast(dst)と型情報からコードを生成する(partial-json patcher etc)](https://zenn.dev/ngicks/articles/go-code-generation-from-ast-and-type-info)がありますが特に内容が前提となっているわけではありません。

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

`*packages.Config`の`Mode`がビットフラグでロードする情報をコントロールします。上記は`PkgPath`(string), `Types`(`*types.Package`), `Syntax`(`[]*ast.File`), `TypeInfo`(`*types.Info`), `TypesSizes`(`types.Sizes`)がロードされます。
`Mode`ビットは`Need{field name}`という名前(になっていないものもいくつかあるが)で各種定義されていますのでほしい情報に合わせてbitwise-ORをとります。

### 型情報を使った各型の判別

複雑な型や、interfaceで表現することができない型の条件は`go/types`以下で定義される型情報を直接走査して判定を行います。

#### `NoCopy`(assignによってコピーすると`go vet`が`copies lock value`と警告される型)の判別

`Lock` methodを備える型を直接(pointerによってindirectされずに)含む型はno-copyなどと言われて、代入や関数の引数に渡すことでコピーが起きると`go vet`で警告を受けます。

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

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/matcher.go#L9-L49

`findMethod`の実装は以下

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/matcher.go#L101-L108

`asNamed`, `asInterface`, `as[T]`の実装は以下

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/matcher.go#L64-L77

この実装はgo vetのそれとは異なり、`sync.Locker`のような`interface`を[struct embedding](https://gobyexample.com/struct-embedding)することで`Lock`を実装するnon-interface型もpointerではないとみなし、no-copy typeとして判定します。

つまり以下はこの実装ではNoCopyとして取り扱われますが、`go vet`は警告しません

```go
type notNoCopy2 struct {
    sync.Locker
}
```

#### `implementor`(`Clone`/`CloneFunc`を実装する型)の判別

ある型`A`が内部に型`B`を持つとき、`B`が`implementor`(`Clone`/`CloneFunc`を実装する型)であるならば`A`の*method*(`Clone`/`CloneFunc`)は`B`の*method*を呼び出します。
`Clone`はともかく、`CloneFunc`は複雑かつ、`interface`として表現できない複雑な条件であるため、型情報を用いた判別を行います。

`Clone`の実装は以下のように判定します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/method_checker.go#L101-L123

`Clone`以外のmethod nameでもいいように、`Name`フィールドでパラメータ化してありますが実際の呼び出しは`Name: "Clone"`以外ですることはありません。

`asPointer`の実装は以下。
[types.NewMethodSet]で、ある型が実装するmethod setを得ることができますが、`Go`の通常のinterfaceのルールと同じくnon-pointer型にはreceiverがnon-pointer型のmethodしか見せなくなっています。
すべてのmethodを見つけるために、型がpointerでない場合は`types.NewPointer`でラップすることでpointerに変換します。
ただし`interface`のpointerをとると逆に[types.NewMethodSet]はmethodを返さなくなるため、`interface`である型はpointerに包まないようにします。
[*types.Named](named type)もしくはinterface literal以外はmethodを実装することはないため、それ以外の型の場合はそのまま引数を返します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/matcher.go#L79-L90

`noArgSingleValue`の実装は以下

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/matcher.go#L110-L131

`unwrapPointer`の実装は以下

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/matcher.go#L92-L99

ここで敷く条件は、`Clone`のreceiverはpointerでもnon-pointerでもよいが、返す型はnon-pointerでなければならないというものです。

`CloneFunc`の実装は以下のように判別します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/method_checker.go#L125-L196

前述通り、`CloneFunc(cloneT func(T) T, cloneU func(U) U, ...)`というシグネチャであるかを判別します。処理の単純性のために、`type A[T, U any]`に対して`CloneFunc(cloneU func(U) U, cloneT func(T) T)`のような感じで、type paramとclonerコールバック関数の順序が変わることを許さないようにしています。
判定する型によっては`A[string, T]`のような感じで具体的な型や別のtype paramでinstantiateされていることがあるため、単純に`types.Identical`だけでは判別できないことに留意します。

#### `clone-by-assign`(non-pointerのみを含む型)の判別

`clone-by-assign`(non-pointerのみを含む型)である場合は、生成対象のパッケージ群で定義されている型でない時でも単純にassignすればよいので、これを判別できるようにしておきます。これの具体例は[log/slog.Attr](https://pkg.go.dev/log/slog@go1.23.4#Attr)などですね。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/tester.go#L5-L31

わりかし単純です。named typeが含まれるときの扱いをフラグで渡せるようにしておくような特別な措置があります。
これはnamed typeが`implementor`であることがありうるので、それが自明でないときにnamed typeが含まれていれば即座にfalseを返せるようにするためにあります。

## 型情報をグラフ化する

生成される`Clone`/`CloneFunc`メソッドは、struct fieldに`Clone`/`CloneFunc`を実装する型が含まれる場合、これらを優先的に呼び出すことになりますが、生成対象となる型が更に別の生成対象となる型を含んでいる場合、初回実行時にはそれらのmethodは生成前であるため存在しません。
そのため、stableな出力結果を得るためには、型が参照する別のnamed typeが生成対象となっているのかを検知する必要があります。
その過程で型情報をすべて列挙するため、この時点で型に`Clone`/`CloneFunc`を実装してよいか、無視すべきかを判別しておきます。

source codeや、それの解析結果自体が型、呼び出しの依存関係の順向きグラフとなっています。
今回知りたいのは型がどこから参照されているかという逆向きの情報であるため、それを洗い出すための特別な実装が必要となります。

以下のpackageでそれを実装します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/typegraph/type_graph.go

このグラフは作成時に渡された`[]*packages.Package`内部のnamed typeをすべて列挙し、named type同士の依存関係を親から子、子から親に相互に参照できるようにedgeでつなぎます。
named typeからnamed typeへの依存関係のみを焦点とするため、例えば`[]T`や`map[K]V`など、無名のslice, array, map, channel型などを*edge route*と呼び、別途記録します。
matcherを受けとり、これを型の列挙中に使用することで、関心のある型にマーキングを行います。
今回で言うと`clone-by-assign`(non-pointerな型しか含まれない型)か`implementor`(`Clone`/`CloneFunc`を実装するnamed type)を含む型が`Clone-able`としてマッチします。

`IterUpward`を以下のように実装し、matcherでmatchした型から依存関係を親側に向けてたどります。channelを含む*edge route*に対しては`Clone`/`CloneFunc`を呼び出すことはできないため、これらを含むedgeはフィルターして辿らないこととします。そのため`edgeFilter`を受けとるようになっています。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/typegraph/type_graph.go#L580-L642

`MarkDependant`を以下のように定義することで、`IterUpward`で辿られた型を`dependant`としてマークします。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/typegraph/type_graph.go#L568-L578

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
midは`for`(`slice`,`array`,`map`)か`if v != nil`(`pointer`)で引数の`v`を1つずつ*unwrap*していき、そのたび一つ内側の型のcloneされたオブジェクトを初期化します(`[]map[string]*[5]T` -> `map[string]*[5]T` -> `*[5]T` -> `[5]T`)
inner endは`T`ごとのclone expressionを記述します。`implementor`なら`Clone`/`CloneFunc`、`clone-by-assign`なら単に`vv`を代入するのみ、という感じです。

`Go`は通常、型推論が行われるため変数の型を宣言しないことも多いです。実際上記の`mid`では`for k, v := range v`とするときにiteration variable(`k`と`v`)の型を明確に記述していません。
ただし上記の例のように`map`, `slice`などの初期化には`make`組み込み関数を用います。これを用いないと`slice`の`len`や`cap`を指定できません。
`make`は型シグネチャを引数として必要とする特殊な関数です。ですので`mid`を経由するたび`ast.Expr`の1つ*unwrap*した型をテキストとして出力できなければなりません。
そこで`types.Type`か`ast.Expr`を受け取って順繰りに*unwrap*する機能が必要です。`types.Type`は[types.TypeSTring](https://pkg.go.dev/go/types@go1.23.4#TypeString), `ast.Expr`は[printer.Fprint](https://pkg.go.dev/go/printer@go1.23.4#Fprint)でテキストとして出力可能だからです。
どちらでもいいのですが、`ast.Expr`を使うと細かい改行の表現などをそのまま保つことができるのでこちらを用いることとします。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/method.go#L421-L433

`typegraph.EdgeKind`は前述の*edge route*の種類を表現するenum-likeな値です。
これを使わなくてもunwrapは成立するんですが、こうするとtypegraph情報との連携がうまくいっていない場合にtype-assertionのところでpanicするので便利です。

上記よりfield unwrapperを`unwrapFieldAlongPath`として定義します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/method.go#L435-L552

`fromExpr, toExpr ast.Expr`を引数に取ることで`fromExpr -> toExpr`な関数を出力します。clonerなのでこの二つは全く同じ`ast.Expr`が渡されることになります。
返り値の関数`unwrapper`で上記で言うclone exprを`wrappee func(string) string`として受け取とり`inner end`でそれを呼び出すようなfield unwrapperをテキストで出力します。

## code generatorの実装

### Configの定義

たとえ挙動を変えうる設定項目が一つもなくてもconfigを主体にAPIを設計しないとあとから設定項目を追加するのが破壊的変更なってしまいます。
今回作成するcode generatorはchannelやNoCopy typeの取り扱いをユーザーに決めてもらおうと思っていたので`Config`を以下のように定義しています。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/generator.go#L25-L28

今後項目が増えるかもしれませんが現在はこれだけです。

`MatcherConfig`は以下のように定義されます。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/matcher.go#L22-L43

これも項目が増えるかもしれませんが現時点ではこれだけです。`NoCopy`, `Channel`, `Func`のグローバルオプションをそれぞれ用意しています。
`CopyHandleIgnore`ならフィールドはclone対象にならず、clone後にはzero valueになります。`CopyHandleDisallow`ならこれを含む型は生成対象から除外されます。`CopyHandleCopyPointer`は、そのフィールドがpointerであるとき(=`*T`, interface, channelなど)の時のみコピーを行いそれ以外の時は`Disallow`として取り扱います。

`CustomHandlers`は後述します。

`Config`に`Generate` methodを実装します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/generator.go#L52-L56

こうすれば`Config`を無視して何かをすることはできなくなります。

### struct fieldにコメントをつけて挙動を変更できるようにする

挙動のfine tuningのためにstruct fieldにdirective commentをつけることでconfigをper-fieldレベルで上書きできるようにします。

#### typegraphのデータにstruct fieldのコメントを格納できるよう拡張する

[型情報をグラフ化する](#型情報をグラフ化する)のところで説明した通り、今回実装するcode generatorは事前に型情報グラフ化して、その時に渡された`matcher`の結果で生成対象の型を決定します。
per-fieldレベルの設定によってtype graphのマッチする、しないが左右されるためtypegraphの機能としてper-fieldレベルのデータを収めることとします。

そこで以下のようにoptionを定義し、

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/typegraph/option.go#L3-L19

`typegraph.Node`に`Priv`(private)データを含めるようにします。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/typegraph/type_graph.go#L60-L74

こういうのはC言語だとよく見るパターンですね。

`PrivParser`はmatcher呼び出しの直前で呼び出します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/typegraph/type_graph.go#L304-L311

この`Priv`データ自体はtypegraphにとって関心のある所ではないため`any`になっています。データは利用者ごとに別々のものを用意したほうがよいでしょう。

```go
type Node[T] any {
    // ...
    Priv *T
}
```

としてもよかったのですが、optionalなもののためにtype paramを追加するのはわかりにくい気がしてやめておきました。

#### コメントを解析する

前述通り`*typegraph.Node`を受けとって型情報ないしはast情報を用いて`Priv`に情報をセットできるようになっています。

Priv dataはいかように`clonerPriv`として定義します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/priv.go#L26-L36

これは前述のConfigをoverrideできるようにロジックを集約しておきます。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/priv.go#L38-L52

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

[\*ast.Field](https://pkg.go.dev/go/ast@go1.23.4#Field)の定義より、1つの`*ast.Field`は複数の`Name`を持てることに注意します。

```go
type A struct {
    Foo, Bar, Baz string
}
```

上記の`Foo`,`Bar`, `Baz`は1つの`*ast.Field`で表現されます。
型情報である`*types.Struct`が備える`Field`メソッドは定義順でn番目のフィールドを取得するAPIです。その点の違いを認識しておく必要があります。
全く同じコメントがアタッチしてあるという解析結果になるにしろすべての`Names`を列挙する必要があります。

解析はdstにいったん変換してから行います。コメントのstableな取り扱いのためには必須です。Goの`go/ast`におけるコメントの取り扱いは単なるオフセットなのでかなりややこしいです

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

//
// /* 8 */
// // 9
// B               string /* 10 */
//

// [0] = "// 7"
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

- `Start`は最後の`\n`以後
- `End`は一行

がフィールドにアタッチされたコメントということになります。

これらのコメントを列挙する関数を`ParseFieldDirectiveCommentDst`として定義すると、以下のように各フィールドのコメントを解析できます。
(これそのものはシンプルなテキスト解析なので特にいうことはありません)

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/priv.go#L54-L111

### matcherの定義

与えられた型に対してどのようなコードを生成すればよいかを判別するmatcherを定義します。
どのようにハンドルすべきか(`type handleKind int`)を定義して、それを返す部分と、実際にコードを生成する部分を分けて実装します。前者がこのmatcherです。
こうすると以下の点で好都合です

- 型グラフ生成時のmatcherとして同じ処理を使いまわせます
- そもそもmatcher部分が巨大かつユーザーに内部状態を教えるためにloggerを受けとるため実際のコード生成はコード生成だけに集中しないとごちゃごちゃして読めたもんじゃないです
- config項目の拡充などでmatcher部分は今後も変わり続けますがコード生成部分は多分あんまりもう変わらないので分離できておくと差分が見やすくていいです

クソデカswitch-case

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/matcher.go#L144-L340

ログも含めて現状200行ぐらいですがまあまあ見通しがいいので関数分割はまだしなくてよさそう。

### 各型のclone

前述のmatcherの呼び出して、得られた`handleKind`をswitch-caseで分岐することで各型のcloneを生成します。前述のfield unwrapperがあるため、実際には`[][]T`のような型があるときには`T`のclone方法のみを生成します。

`T`が型グラフの構築時にmatchedとなった型については`Clone`/`CloneFunc`を呼び出す必要があるので、その考慮を加えるために`handleField`というラッパーを経由してmatcherを呼び出します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/matcher.go#L379-L420

各型向けに呼び出します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/method.go#L223-L228

switch-caseで分岐してそれぞれ向けのテキストを生成します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/method.go#L255-L406

struct literalと`CloneFunc`のtype argに対して再帰呼び出しが必要でややこしいですがmatcher部分と分離したため割合単純なコードになっています。

### field unwrapperと組み合わせる

`unwrapFieldAlongPath`の呼び出しによって得られたfield unwrapperと組み合わせて最終的な`cloner func(string) string`が得られます

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/method.go#L248-L253

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/method.go#L408-L418

### source fileにsuffixを付けたファイルへ書き出し

あとはこれを書き出したら終わりです。

対象となった型が含まれていたファイル名+suffixなファイルに吐き出す方式をとるため以下で`suffixwriter`を定義します

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/suffixwriter/writer.go

[token.FileSet.Position](https://pkg.go.dev/go/token@go1.23.4#FileSet.Position)で得られる[token.Position](https://pkg.go.dev/go/token@go1.23.4#Position)からファイル名が得られます。
これを引数に`suffixwriter`を呼び出すことで所望の挙動を実現できます。

## build constraintsのコピー

[go command documentationのBuild constraintsの項](https://pkg.go.dev/cmd/go#hdr-Build_constraints)からわかる通り、`//go:build` directive commentや、ファイル名に`_linux`や`_amd64`などのsuffixをつけることでbuild constraintsを指定できます。

C言語では[#ifdef](https://learn.microsoft.com/ja-jp/cpp/preprocessor/hash-ifdef-and-hash-ifndef-directives-c-cpp)などを活用して、ファイルの中でもconstraintに合わせて定数や関数の定義を切り替えることができます。ファイルの特定の行があるかどうかを制御する`if`のようなものですから、ファイル内で、例えば`linux`と`windows`向けの定義を複数かくとかできます。
`Go`では一方で、1ファイルに1つしか`//go:build`が存在することが許さないようになっています。

さて、build constraintを尊重する方法についてですが、以下の二つがあります

- `//go:build`コメントをコピーする
- ファイルのsuffixがbuild constraintであるときはコピーする

### //go:buildコメントのコピー

コピーするというよりは`//go:build`コメント以外を消すというのが正しいです。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/codegen/comment.go#L14-L25

[go/build/constraint.IsGoBuild](https://pkg.go.dev/go/build/constraint@go1.23.4#IsGoBuild), [go/build/constraint.IsPlusBuild](https://pkg.go.dev/go/build/constraint@go1.23.4#IsPlusBuild)が定義されているのでこれをそのまま使います。

`IsPlusBuild`は`// +build`から始まる行に対してtrueを返します。これが`Go1.16`かそれ以前までに使われていた形式で、`Go1.18`あたりからほぼdeprecationを迎えているはずですが、一応簡単にサポートできるのでベストエフォートで対応してあります。たしか`// +build`のほうは複数行にまたがって書くことができるのでうまく判定できないと思います。

こうしてtrimされたpackage-commentを`PrintFileHeader`内でprintしていきます。
この関数は出力されるファイルすべてに対して呼ばれるprinterでpackage clauseとimport declをすべて出力するものです。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/codegen/print.go#L249-L288

`printer.Fprint`がコメントだけとかそういうレベルのprintに対応していないのでちょっと頑張って出力しています。

### ファイルのsuffixがbuild constraintであるときはコピーする

`Go`はファイル名が`_test` suffixを除いたときに`_os`, `_arch`, `_os_arch`でsuffixされている場合暗黙的なbuild constraintsとして取り扱います。
サポートされるos, archの一覧は`go tool dist list`で得られます。

:::details go tool dist listの出力

```
# go tool dist list
aix/ppc64
android/386
android/amd64
android/arm
android/arm64
darwin/amd64
darwin/arm64
dragonfly/amd64
freebsd/386
freebsd/amd64
freebsd/arm
freebsd/arm64
freebsd/riscv64
illumos/amd64
ios/amd64
ios/arm64
js/wasm
linux/386
linux/amd64
linux/arm
linux/arm64
linux/loong64
linux/mips
linux/mips64
linux/mips64le
linux/mipsle
linux/ppc64
linux/ppc64le
linux/riscv64
linux/s390x
netbsd/386
netbsd/amd64
netbsd/arm
netbsd/arm64
openbsd/386
openbsd/amd64
openbsd/arm
openbsd/arm64
openbsd/ppc64
openbsd/riscv64
plan9/386
plan9/amd64
plan9/arm
solaris/amd64
wasip1/wasm
windows/386
windows/amd64
windows/arm
windows/arm64
```

:::

この内容を`/`でsplitしてありうるprefixの一覧を`map[string]bool`に集めます。
`Go`にはzero valueが概念がありますから`O(1)`で存在チェックをしたい場合には`map[K]bool`をよく利用します。`map[K]struct{}`とするよりコードが1行短くなります。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/suffixwriter/suffix.go#L69-L107

`go`コマンドが入っていない環境は全く想定していませんが、一応ない場合はソースに埋めておいたものにfallbackするようにしておきます。

前述通りファイルの出力の際には`suffixwirter`で`.cloner`のようなsuffixを加えてファイル名に書き込みを行いますが、ここをbuild constraintsとなるsuffixをさらに末尾に加えるように改変します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/suffixwriter/suffix.go#L109-L152

とりあえず書いてテストが通る程度なので見てすぐわかる程度に非効率なコードですが当面はこうでいいとしています。

## Custom handler

cloneの処理はユーザーごとに別のものを与えたかったりすることは十分に想定できます。
そこで、Custom handlerを渡せるようにします。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/generator/cloner/custom_handler.go#L39-L49

- `Matcher`がtrueを返す時にこのcustom handlerを実行します
  - 例えば`[]map[string]*[5]T`がstruct fieldの型であるとき、`Matcher`は`[]map[string]*[5]T`, `map[string]*[5]T`, `*[5]T`, `[5]T`を引数に何度も呼ばれます。
- `Imports`でこのcustom handlerが使用する外部パッケージを指定します。
- `Expr`で各種データを受けとって`func(s string) string`を返します。`s`は`Matcher`がtrueを返す型の変数のidentのテキストです。`isFunc`がtrueのとき、返された`expr`は呼び出し可能な関数ですので、clonerはこれを呼び出すようなコードを生成します。

`CustomHandlerExprData`の`ImportMap`は`types.Qualifier`になれたりするようなものです。`types.TypeString`とともに使ってもよいようにしてあります。
`AstExpr`、`Ty`はその型のexpressionとなります。どっちをベースに型をテキストに変換してもよいようにしてあります。

https://github.com/ngicks/go-codegen/blob/15a24b031aac0b07d8ed2b2471fac88b5ca8caeb/codegen/imports/parser.go#L398-L408

[matcherの定義](#matcherの定義)の項ですでに見せていますが、例えば`[]map[string]*[5]T`があるとき、`[5]T`にマッチするcustom handlerを提供すると、`[]map[string]*`までのfield unwrapperが出力され、`[5]T`に対してcustom handlerを呼び出します。

いくつかbuilt-inのcustom handlerを定義しておいています。

https://github.com/ngicks/go-codegen/blob/15a24b031aac0b07d8ed2b2471fac88b5ca8caeb/codegen/generator/cloner/custom_handler.go#L51-L186

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

- `[]T`に対して`copy`ベースのcloneを実行する
- `map[K]V`に対して`maps.Clone`を呼び出す
- `time.Time`に大して`time.Date(v.Year(), v.Month(), v.Day(), v.Hour(), v.Minute(), v.Second(), v.Nanosecond(), v.Location())`を呼び出す
  - monotonic timer以外のコピーです。
  - 単なる代入だとmonotonic timerがコピーされて困ります。されたほうがいいかも？要今後の検討ですね
- basic type、もしくは既知の`clone-by-assign`、もしくはそれらのarrayを単なる代入に変える

目下拡充予定です。

## cliとして呼び出せるようにする

[github.com/spf13/cobra](https://github.com/spf13/cobra)を使ってサブコマンドとして呼び出せるようにしてあります。

```
# go run github.com/ngicks/go-codegen/codegen@15a24b031aac0b07d8ed2b2471fac88b5ca8caeb cloner --help
go: downloading github.com/ngicks/go-codegen/codegen v0.0.0-20241221100251-15a24b031aac
go: downloading github.com/ngicks/go-codegen v0.0.0-20241221100251-15a24b031aac
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
We call parts other than that ([]*[5]map[int]) _route_. And each element of them as _route node_.
Only disallowed _route node_ is struct and interface literal. They are ignored silently.

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
      --chan-copy             copies channel fields
      --chan-disallow         disallows channel fields
      --chan-ignore           ignores channel fields.
      --chan-make             makes new channel. Clone methods also copy the capacity of input channels.
  -d, --dir string            specifies the current working directory for the source code loader and other tools.
                              The path specified by --pkg flag is evaluated under this directory.
                              If empty cwd will be used.
      --dry                   enables dry run mode. any files will be removed nor generated.
      --func-copy             copies func fields
      --func-disallow         disallow func fields.
      --func-ignore           ignores func fields. func literal or named function type.
  -h, --help                  help for cloner
      --ignore-generated      You do not need this option.
                              If set, the type checker ignores ast nodes with comment //codegen:generated attached.
                              Useful for internal debugging.
      --no-copy-copy          copy pointer of no-copy object. Clone methods copy no-copy object if and only if field is pointer type.
      --no-copy-disallow      disallow no-copy object. Types that contain no-copy type fields are not generation target.
      --no-copy-ignore        ignores no-copy object. Clone methods just simply leave fields zero value.
  -p, --pkg ./                target package pattern. relative to dir. must start with ./. can be `./...`
  -v, --verbose               verbose logs
```

[matcherの定義](#matcherの定義)で見せた`MatcherConfig`の各項目がcli flagとしてカスタマイズできるようになっています。説明がわかりにくい気がする・・・この辺は今後の改善事項ですね。
あとstruct literalはサポートしているのでメッセージが間違っていますね。当初はサポートせんでええわと思っていたんですが(実装がめんどくさかったから)、[github.com/oapi-codegen/oapi-codegen](https://github.com/oapi-codegen/oapi-codegen)でデフォルトに近い設定のまま[ResponseObject](https://swagger.io/specification/#response-object)の下に直接responseのschemaを記述するとstruct literalが出力されるため、サポートしないとまずいなと思ってサポートに加えた経緯があります。open-api specは自分側にコントロールがないことが多いからですね・・・

## 生成結果のexample

以下にexample typeとその生成結果が出力されます。

https://github.com/ngicks/go-codegen/tree/15a24b031aac0b07d8ed2b2471fac88b5ca8caeb/codegen/generator/cloner/internal/testtargets

tree構造のcloneのexampleとか

https://github.com/ngicks/go-codegen/blob/15a24b031aac0b07d8ed2b2471fac88b5ca8caeb/codegen/generator/cloner/internal/testtargets/tree/tree.go

https://github.com/ngicks/go-codegen/blob/15a24b031aac0b07d8ed2b2471fac88b5ca8caeb/codegen/generator/cloner/internal/testtargets/tree/tree.clone.go

(exampleなのに本当に動作するbinary treeの実装を書いちゃった)

struct literalを含む型に対するexampleとか

https://github.com/ngicks/go-codegen/blob/15a24b031aac0b07d8ed2b2471fac88b5ca8caeb/codegen/generator/cloner/internal/testtargets/structlit/lit.go

https://github.com/ngicks/go-codegen/blob/15a24b031aac0b07d8ed2b2471fac88b5ca8caeb/codegen/generator/cloner/internal/testtargets/structlit/lit.clone.go

build constraintsを含む場合のexampleとか

https://github.com/ngicks/go-codegen/tree/15a24b031aac0b07d8ed2b2471fac88b5ca8caeb/codegen/generator/cloner/internal/testtargets/constraint

色々用意してあります。

## 今後

- unexport fieldを持たないstructおよびそれを含む`map`, `slice`-base typeのclonerをad-hocに吐き出す
- known clone by assignの拡充
  - stdを全部洗う
- overlayオプション
  - struct fieldにコメントをつけることでfine tuningが行えますが、外部データからも全く同じことができるようにする。
  - 他のcode generatorによって生成された型にコメントをつけて回るのは現実的にしたくない運用だからそこをカバーしに行くためです。
    - これは要するに筆者自身が欲している機能です
- in-placeオプションの拡充
  - フィールドレベルでclonerの関数を指定したり
  - フィールドを無視させたり、単なるassignに変えたり
- templateによるcustom handlerの受付
  - 現状custom handlerはgoのプログラムとして呼び出す場合にのみ渡せますが、これをcli経由でも渡せるように整備するということです。
  - `text/template`で解釈できるテキストとしてcustom handlerを定義できれば
- ドキュメントを整備する
  - 現状これらのサブコマンドの説明がgit repositoryのトップに全然なくて誰も把握できてないと思います。
  - 誰が読むのかもわからないドキュメントを読みやすく整備するのは精神との戦いだったりしますね
  - そもそもぱっと読んでわかるドキュメントを備えたOSSってのが珍しくて、ソースを読んでようやくなるほどなってなることが多いです。多分大変なことなんですよね。

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
