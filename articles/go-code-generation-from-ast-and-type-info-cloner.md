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

`Go`では、値は代入(`a = b`)によってコピーします。
ただし、[pointer](https://go.dev/tour/moretypes/1)(`*T`)のコピーはポインターのアドレスの数値のコピーのコピーです。
`*T`という変数は実際には値のアドレスを格納した`uint`であり、`Go`の型システムが自動的に指し示された値を操作するような変換を行っていると考えるとよいと思います。
行いたいコピーがポインターが指し示している内容をコピーである場合は単なるassignでは不十分ということになります。

たとえ値がポインターであったときでも、`if v != nil { copied = *v }`という風に`*`で[pointer indirection](https://go.dev/ref/spec#Address_operators)を行うことで値のコピーを行うことができます。
ただし、`Go`の型は(ほかの言語でもそうであるように)`struct`などで別の型を含むことができるため、値がpointerを内部的に持つ型である場合はpointerのコピーが起きてしまいます。

`Go`では、ご存じのとおり`type A B`によって新しい型を定義できます。ここで`A`をtype nameなどと言い、これが新しく定義される型の名前です。`A`は名前のある型なので、named typeなどと型システム(`go/types`)上では呼ばれます。
`B`は[underlying type](https://go.dev/ref/spec#Underlying_types)と呼ばれ、これが`A`のデータ構造となります。
`B`には`struct`で新しいデータセットを指定したり、 `pointer`(`*T`), `array`(`[n]T`), `slice`(`[]T`), `map`(`map[K]V`), `channel`(`chan T`)のような組み込み型を指定することができます。
`B`には別のnamed typeを指定することもできます。この場合`A`は`B`のメソッドセットを除いて、データ構造だけを継承します。`Go`では`B(A)`で構造の同じ型同士を変換できますから、`B`のメソッドを使用したい場合は明示的に型を変換します。

`struct`はfieldに別の型のpointer(e.g. `*int`)を指定できますし、`slice`, `map`, `channel`は暗黙的に内部でpointerを持ちます。
そのため、それらを`underlying type`とする型は、値そのものがnon-pointerであってもその値の型がpointerを含むため複製できないことがあります。

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

`Go`は`encoding/*`以下にいくつかのデータ変換パッケージを提供します。例えば以下のようなものがあります。

- [encoding/json](https://pkg.go.dev/encoding/json@go1.23.4): `JSON`と`Go`データ構造の相互変換機能
- [encoding/gob](https://pkg.go.dev/encoding/gob@go1.23.4): self-describingなバイナリと`Go`データ構造の相互変換機能

`gob`は多分`Go object`の略ですかね。

また、バイナリフォーマットに変換するため、各フォーマットごとに表現力の限界があり、場合によっては正しくdeep cloneを実現できないことにも留意してください。

例えば

- `JSON`にはポインターに当たる機能がないため循環構造を変換できません。
  - `YAML`には[anchor](https://yaml.org/spec/1.2.2/#anchors-and-aliases)仕様があるためこちらを用いるのが解決法かもしれません。

`gob`は使ったことがないため筆者には特に何かを述べることができません。

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
  - 細かいコピーのルールは定義できません。

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

[The Go Blog: Generating code](https://go.dev/blog/generate)などからわかる通り、`Go`はcode generationを多用する文化がコミュニティーの中の存在します。

そもそもマクロが現状(`Go1.24`時点)ありませんし、それに類するツールもありません。[Go1.18]までgenericsもなかったため、code generatorを使わざるを得ないことも多くありました。
この辺の話は[昔の記事: "Goのcode generatorの作り方: 諸注意とtext/templateの使い方"の"Rationale: なぜGoでcode generationが必要なのか"](https://zenn.dev/ngicks/articles/go-code-generation-in-ways-text-template#rationale%3A-%E3%81%AA%E3%81%9Cgo%E3%81%A7code-generation%E3%81%8C%E5%BF%85%E8%A6%81%E3%81%AA%E3%81%AE%E3%81%8B)で述べましたので併せて読むといいかもしれません。

[reflect](https://pkg.go.dev/reflect@go1.23.4)の`godoc`を見れば分かりますが、動的な変数の宣言を行うことができるため、当然これを用いればdeep cloneも実現できます。

Pros, Consは以下になります。

- Pros:
  - exportされていないfieldのコピーが可能
  - 高速: `marshal`/`unmarshal`のオーバーヘッドや、`reflect`によるメモリアロケーションコストを回避できます。
  - 依存性が減る: code generatorそのものは`go.mod`に含める必要がない
- Cons:
  - コード変更のたびにcode generatorを動作させる必要がある。
  - code generatorのバージョン管理も必要

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

さらに、code generatorを用いる方法は、ソース変更を行うたびに再生成が必要というデメリットはあるものの、上記の二つを克服しうるものであることは述べました。

ではなぜcode generatorライブラリが存在している今の状況で新規に開発を行う必要があるのでしょうか？

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

シンプルな型には`Clone`メソッドを生成します。`Type`はnon-pointer type(`*T`でなく`T`)とします。これは各々メリット、デメリットあると思いますが`deep clone`の意図がデータの複製であるのでポインターは返えさないものとします。

[type parameter](https://go.dev/ref/spec#Type_parameter_declarations)のある型には`CloneFunc`を生成します。こちらも同じくnon-pointer typeを返します。
それぞれのtype paramを複製するためのコールバック関数を`clone+{type param name}`で受け取ります。

type parameterやgenericsは[A Tour of GoのType parameters](https://go.dev/tour/generics/1)で説明されているので説明は割愛します。

## 実装の方式、考慮事項

ここからは[前回の記事: \[Go\]ast(dst)と型情報からコードを生成する(partial-json patcher etc)](https://zenn.dev/ngicks/articles/go-code-generation-from-ast-and-type-info)がありますが特に内容が前提となっているわけではありません。

### [golang.org/x/tools/go/packages]による`ast`, `type info`の読み込み

[golang.org/x/tools/go/packages]を用いるとGo source code解析してast、type info、そのほかのメタデータなどを得ることができます。

[go/ast], [go/types]がそれぞれast解析、astからtype info解析を行う機能を提供しているのですが、対象となるGo source codeが外部モジュールをimportしているとき、これをうまく読み込む手段をこれらは用意していません。
そのため、go moduleのdependency graphを形成して末端のnodeとなる、自分以外に何もimportしていないmoduleから順番に解析してimporterとしてtype infoを返すようなものを自作する必要があるのですが、それをやってくれるのが[golang.org/x/tools/go/packages]なのです。

[golang.org/x/tools/go/packages]でtype checkまで行うコードは以下のようになります

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

`packages.Load`でpackage patternを受けとり、マッチするmoduleの各種情報を[\*packages.Package](https://pkg.go.dev/golang.org/x/tools@v0.28.0/go/packages#Package)として取得します。

`*packages.Config`の`Mode`がロードする情報をコントロールします。上記は`PkgPath`(string), `Types`(`*types.Package`), `Syntax`(`[]*ast.File`), `TypeInfo`(`*types.Info`), `TypesSizes`(`types.Sizes`)がロードされます。
`Mode`ビットは`Need{field name}`という名前(になっていないものもいくつかあるが)で各種定義されています。

### 生成対象の型依存関係のグラフ化、グラフを逆にたどることで生成対象の型を列挙する。

生成される`Clone`/`CloneFunc`メソッドは、struct fieldに`Clone`/`CloneFunc`を実装する型が含まれる場合、これらを優先的に呼び出すことになりますが、生成対象となる型が更に別の生成対象となる型を含んでいる場合、初回実行時にはそれらのmethodは生成前であるため存在しません。
そのため、stableな出力結果を得るためには、型が参照する別のnamed typeが生成対象となっているのかを検知する必要があります。

source codeや、それの解析結果自体がある意味依存関係のグラフとなっています。ただ、型がどこから参照されているかという情報は筆者が知る限り含まれていないため、この逆方向の依存関係を洗い出すための特別な実装が必要となります。

以下のpackageでそれを実装します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/typegraph/type_graph.go

方式としては単純で、渡された`[]*packages.Package`内部のnamed typeをすべて列挙し、named type同士の依存関係を親から子、子から親に相互に参照できるようにedgeでつなぎます。
matcherを受けとり、これを型の列挙中に使用することで、関心のある型にマーキングを行います。
今回で言うと`clone-by-assign`(non-pointerな型しか含まれない型)か`implementor`(`Clone`/`CloneFunc`を実装するnamed type)を含む型が`Clone-able`としてマッチします。

`IterUpward`を以下のように実装し、matcherでmatchした型から依存関係を親側に向けてたどります。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/typegraph/type_graph.go#L580-L642

`MarkDependant`を以下のように定義することで、`IterUpward`で辿られた型を`dependant`としてマークします。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/typegraph/type_graph.go#L568-L578

マークされた型は`Clone`/`CloneFunc`を実装することになるため、これらの方に対しては盲目に(型情報によらずに)`Clone`/`CloneFunc`を呼び出すコードを吐き出してよいことになります。

### 型情報を使った各型の判別

複雑な型や、interfaceで表現することができない型の条件は`go/types`以下で定義される型情報を直接走査して判定を行います。

#### `Clone`/`CloneFunc`を実装する型(_implementor_)

`Clone`/`CloneFunc`の判別は`go/types`以下で定義される型情報を用います。

`Clone`の実装は以下のように判定します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/method_checker.go#L101-L123

`findMethod`の実装は以下

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/matcher.go#L101-L108

`asPointer`の実装は以下
`Go`の通常のinterfaceのルールと同じく`types.NewMethodSet`はnon-pointer型にはreceiverがnon-pointer型のmethodしか見せなくなっています。
そのため、`interface`ではないときはpointerにwrapしてから`types.NewMethodSet`に渡します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/matcher.go#L79-L90

`noArgSingleValue`の実装は以下

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/matcher.go#L110-L131

`unwrapPointer`の実装は以下

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/matcher.go#L92-L99

ここで敷く条件は、`Clone`のreceiverはpointerでもnon-pointerでもよいが、返す型はnon-pointerでなければならないというものです。

#### `NoCopy`(assignによってコピーすると`go vet`が`copies lock value`で怒る型)か

`Lock` methodを備える型を直接(pointerによってindirectされずに)含む型はno-copyなどと言われて、代入や関数の引数に渡すことでコピーが起きると`go vet`で警告を受けます。

`Clone`/`CloneFunc`はこれらをコピーしないように単にフィールドをzero valueのままほおっておくとか、含まれる型はそもそも生成対象から外すとか、pointerならpointerをコピーするとかをユーザーに選ばせたいので、no-copyの判定を行う必要があります。

以下のように実装します。

https://github.com/ngicks/go-codegen/blob/99bf20cadbdeafb66781c16882fffc765baab9a2/codegen/matcher/matcher.go#L9-L49

この実装はgo vetのそれとは異なり、`sync.Locker`のような`interface`を[struct embedding](https://gobyexample.com/struct-embedding)して`Lock`を実装するnon-interface型もpointerではないとみなし、no-copy typeとして判定します。

#### assignによってcloneできる型か

### build constraintsのコピー

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
