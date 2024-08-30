---
title: "Goのcode generatorの作り方: jenniferの使い方"
emoji: "🧰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## Goのcode generatorの作り方についてまとめる

`Go`のcode generationについてまとめようと思います。

前段の記事: [Goのcode generatorの作り方: 諸注意とtext/templateの使い方](https://zenn.dev/ngicks/articles/go-code-generation-in-ways-text-template)で

- Rationale: なぜGoでcode generationが必要なのか
- code generatorを実装する際の注意点など
- `io.Writer`に書き出すシンプルな方法
- `text/template`を使う方法
  - `text/template`のcode generationにかかわりそうな機能性。
  - 実際に`text/template`を使ったcode generatorのexample。

について述べました。

この記事では

- [github.com/dave/jennifer]を用いたcode generatorの実装

について述べます

さらに後続の記事で

- [Goのcode generatorの作り方: ast(dst)を解析して書き換える](https://zenn.dev/ngicks/articles/go-code-generation-in-way-ast-dst)で[astutil](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/ast/astutil)および[github.com/dave/dst]を用いる方法

について述べます。

[github.com/dave/jennifer]は、code generatorを作るためのライブラリで、`Go`のトークンや構文に紐づいた関数をメソッドチェーンで順番に呼び出すことでcodeを生成することができます。
`README.md`をしっかり読めば特に説明が必要なことはないのですが、いくらかコードサンプルを示すことで、READMEを読むための勘所をえられるような手助けをする記事となります。
そのため前段、後段の記事に比べてこの記事はずいぶん文字数が少ないです。

## 前提知識

- [The Go programming language](https://go.dev/)の基本的文法、プロジェクト構成などある程度Goを書けるだけの知識

## 環境

`Go`のstdに関するドキュメントおよびソースコードはすべて`Go1.22.6`のものを参照します。
[golang.org/x/tools](https://pkg.go.dev/golang.org/x/tools@v0.24.0)に関してはすべて`v0.24.0`を参照します。

コードを実行する環境は`1.22.0`です。

```
# go version
go version go1.22.0 linux/amd64
```

書いてる途中で`1.23.0`がリリースされちゃったんですがでたばっかりなんで`1.22.6`を参照したままです。ﾏﾆｱﾜﾅｶｯﾀ。。。

## github.com/dave/jennifer

[github.com/dave/jennifer]を利用する方法です。

このライブラリはcode generatorを作成するためのライブラリですので、前段の`text/template`を用いる方法よりもはるかに簡単に記述することが可能です。

[github.com/dave/jennifer]を使うと、

- `Go`のトークンや構文に対応づいた関数群をメソッドチェインで呼び出すことでコードを生成することができます。
- [Qual](https://github.com/dave/jennifer/tree/v1.7.0?tab=readme-ov-file#qual)によって自動的にimport declが管理されるため、同名のパッケージをインポートする際の名前被りも自動的に回避されます。
- `○○Func`系のメソッドや[Do](https://pkg.go.dev/github.com/dave/jennifer/jen@v1.7.0#Do)で関数を受けとることができるので容易にfor-loopを回したパラメータに基づく生成が可能です。

### 利点と欠点

利点:

- 書きやすい
  - `Qual`によってimport declを自動的に管理してくれるのでimportの名前かぶりに関して気を使う必要がない。
- ごちゃごちゃしてるように見えてメンテしやすい(体感上)
- 単なる`Go`コードであるので任意に分割して再利用できる

欠点:

- std外のライブラリをインポートしてしまう。
- ユーザーからファイルを通して入力を受けとる方法が特に決まっていない

### 基本的な使用方法

コードは以下でホストされます

https://github.com/ngicks/go-example-code-generation/tree/main/jennifer/basic

[README.md](https://github.com/dave/jennifer/tree/v1.7.0?tab=readme-ov-file#jennifer)でしっかり説明がなされているので特に説明することはないかと思います、APIの様式がわかる程度のことを書いておいたほうが読みやすいかもしれないので先にここでそれについて述べておきます。

#### 宣言、書き出し

`jen.NewFile`, `jen.NewFilePath`, `jen.NewFilePathName`のいずれかで`*jen.File`(=1つの`.go`ファイルに対応づくもの)をallocateし、そこからメソッドをいろいろ呼び出します。最後に`(*jen.File).Render`でファイルなどに生成したコードを書き出して終了します。

```go
package main

import (
	"bytes"

	"github.com/dave/jennifer/jen"
)

func main() {
	f := jen.NewFile("baz")
	f.Var() //...

	buf := new(bytes.Buffer)
	if err := f.Render(buf); err != nil {
		panic(err)
	}
	if err := os.WriteFile("path/to/dest", buf.Bytes(), fs.ModePerm); err != nil {
		panic(err)
	}
}
```

`Render`は[NoFormat](https://github.com/dave/jennifer/blob/v1.7.0/jen/file.go#L64)を`true`にしない限り[format.Sourceによってフォーマットをかける](https://github.com/dave/jennifer/blob/v1.7.0/jen/jen.go#L81-L89)挙動があります。そのため、出力が`Go`のソースコードとして正しくない場合にformat部分でエラーを吐くことがあります。
上記サンプルでは一旦`Render`の結果を`*bytes.Buffer`に受けてからファイルに書き出していますが、こうすることで、`format.Source`の成功まで`os.Create`による対象ファイルのtruncateを遅延しています。

#### print

`*jen.Statement`などの各typeにはデバッグ用途として`GoString`メソッドが実装されており、それによってコード断片状態の`*jen.Statement`などを書き出すことができます。
`fmt.Printf("%#v\n", v)`でprintするのが最も便利でしょう。

```go
decoratePrint := func(v any) {
	fmt.Println("---")
	fmt.Printf("%#v\n", v)
	fmt.Println("---")
	fmt.Println()
}

decoratePrint(jen.Var().Id("yay").Op("=").Lit("yay yay"))
/*
	---
	var yay = "yay yay"
	---
*/
```

以後のコードスニペットは`decoratePrint`の宣言は省略されます。

#### \*jen.File

[\*jen.File](https://pkg.go.dev/github.com/dave/jennifer/jen@v1.7.0#File)は[jen.NewFile](https://pkg.go.dev/github.com/dave/jennifer/jen@v1.7.0#NewFile), [jen.NewFilePath](https://pkg.go.dev/github.com/dave/jennifer/jen@v1.7.0#NewFilePath), [jen.NewFilePathName](https://pkg.go.dev/github.com/dave/jennifer/jen@v1.7.0#NewFilePathName)のいずれかで作成します。

それぞれは以下のように[Qual](https://github.com/dave/jennifer/tree/v1.7.0?tab=readme-ov-file#qual)を使った場合の挙動が違います。
`NewFilePath`, `NewFilePathName`は生成対象のパッケージパスを認識しますので、`Qual`が参照するのが生成対象そのものだった時は`PackageName`が省略されます。

```go
f = jen.NewFile("baz")
f.NoFormat = true
f.Qual("foo/bar/baz", "Wow")
decoratePrint(f)
/*
	---
	package baz

	import baz "foo/bar/baz"


	baz.Wow
	---
*/

f = jen.NewFilePath("foo/bar/baz")
f.NoFormat = true
f.Qual("foo/bar/baz", "Wow")
decoratePrint(f)
/*
	---
	package baz


	Wow
	---
*/

f = jen.NewFilePathName("foo/bar/baz", "hoge")
f.NoFormat = true
f.Qual("foo/bar/baz", "Wow")
decoratePrint(f)
/*
	---
	package hoge


	Wow
	---
*/
```

#### Package comment

`*jen.File`の`PackageComment`で`package clause`より先にコメントを書き出します。

```go
var f *jen.File

f = jen.NewFile("foo")
f.PackageComment("// Code generated by me. DO NOT EDIT.")
decoratePrint(f)
/*
	---
	// Code generated by me. DO NOT EDIT.
	package foo

	---
*/
```

#### if err != nil { return err }

```go
decoratePrint(jen.If(jen.Err().Op("!=").Nil()).Block(jen.Return(jen.Err())))
/*
	---
	if err != nil {
			return err
	}
	---
*/
```

#### struct def

```go
decoratePrint(jen.Type().Id("foo").Struct(
	jen.Id("A").String().Tag(map[string]string{"json": "a"}),
	jen.Id("B").Int().Tag(map[string]string{"json": "b", "bar": "baz"}),
))
/*
	---
	type foo struct {
			A string `json:"a"`
			B int    `bar:"baz" json:"b"`
	}
	---
*/
```

#### []T{}

`Values`で`{...}`をレンダーします。自分で`Line`を追加しない限り改行しません。

```go
decoratePrint(jen.Var().Id("bar").Op("=").Index(jen.Op("...")).String().Values(jen.Lit("foo"), jen.Lit("bar"), jen.Lit("baz")))
/*
	---
	var bar = [...]string{"foo", "bar", "baz"}
	---
*/
```

#### []T{}+自動改行

`Values`は自動的に1項目ごとに改行しません。`Custom`を用いると項目ごとに改行できます。

```go
decoratePrint(
	jen.Var().Id("bar").Op("=").Index(jen.Op("...")).String().
		Custom(jen.Options{
			Open:      "{",
			Close:     "}",
			Separator: ",",
			Multi:     true,
		},
			jen.Lit("foo"), jen.Lit("bar"), jen.Lit("baz"),
		),
)
/*
	---
	var bar = [...]string{
			"foo",
			"bar",
			"baz",
	}
	---
*/
```

#### 関数を受けとるメソッド: ○○Func

`○○Func`という風に`Func`が,例えば、`StructFunc`を用いると関数を受けることができるので、ここでfor-loopを回すなりするとよいでしょう。

```go
fields := []struct {
	name string
	def  *jen.Statement
}{
	{"foo", jen.String()},
	{"bar", jen.Int()},
	{"baz", jen.Op("*").Qual("bytes", "Buffer")},
}
decoratePrint(jen.Type().Id("foo").StructFunc(func(g *jen.Group) {
	for _, f := range fields {
		g.Id(f.name).ががdd(f.def)
	}
}))
/*
	---
	type foo struct {
			foo string
			bar int
			baz *bytes.Buffer
	}
	---
*/
```

#### 関数を受けとるメソッド: Do

同様に[Do](https://pkg.go.dev/github.com/dave/jennifer/jen@v1.7.0#Do)もコールバック関数を受け取ります。
`○○Func`と違って受け入れる関数のシグネチャは`func(s *jen.Statement)`です。

```go
	decoratePrint(jen.Type().Id("bar").Op("struct").Op("{").
		Do(func(s *jen.Statement) {
			for _, f := range fields {
				s.Id(f.name).Add(f.def).Line()
			}
		}).
		Op("}"),
	)
	/*
		---
		type bar struct {
		        foo string
		        bar int
		        baz *bytes.Buffer
		}
		---
	*/
```

いい例が思いつかなくて少々ぎこちない感じですが、`Do`で受け取ったコールバックで書きだす内容は何でもいいので任意に関数分割を行えます。

#### Add: \*jen.Statementや\*jen.Groupを追加する

import pathが長くなると`Qual`を何度も書くと冗長なので関数に切り分けたくなると思います。

`Add`を用いれば`Qual`など、繰り返すには長すぎる表現を関数に切り出せます。

```go
	randomIdentName := func(leng int) string {
		var buf strings.Builder
		buf.Grow(1 + 2*leng)
		_ = buf.WriteByte('_')
		for range leng {
			_, _ = buf.WriteString(fmt.Sprintf("%x", [1]byte{rand.N[byte](255)}))
		}
		return buf.String()
	}
	bytesQual := func(ident string) *jen.Statement {
		return jen.Qual("bytes", ident)
	}
	imageQual := func(ident string) *jen.Statement {
		return jen.Qual("image", ident)
	}
	decoratePrint(jen.Var().DefsFunc(func(g *jen.Group) {
		g.Id(randomIdentName(4)).Op("=").Op("*").Add(bytesQual("Buffer"))
		g.Id(randomIdentName(4)).Op("=").Add(imageQual("Image"))
	}))
	/*
		---
		var (
		        _76ef57e5 = *bytes.Buffer
		        _5d5cdb42 = image.Image
		)
		---
	*/
```

#### HACK: Idでコード片を挿入

`Id`などはノーチェックで渡されたstringの内容を書き出しているだけなのでGo source codeとして有効ならなんでも書き出すことができます。
ただし手動で`Line`で囲まないと改行なしでトークンが出力される可能性があるのでそこだけ注意が必要です。

```go
decoratePrint(jen.Type().Id("Yay").String().Line().Id(`func foo() bool { return true }`).Line().Type().Id("Nay").Bool())
/*
	type Yay string

	func foo() bool { return true }

	type Nay bool
*/
```

この方法で差し込まれたコード片はimport declを更新できないので新しいimportがここで追加される場合うまく機能しません。
import declの内容を追加するのは筆者が見たところ`Qual`のみです。
そのため何かしらのHACKをさらに重ねない限り、`text/template`の出力結果を`Id`で埋め込む・・・みたいなことはできません。

### jennifer example: enum

[text/template example: enum](https://zenn.dev/ngicks/articles/go-code-generation-in-ways-text-template#text%2Ftemplate-example%3A-enum)と同じものを[github.com/dave/jennifer]で再実装します。

関数呼び出しがどういったコードと対応づいているのかをコメントしておいたので、これでおそらく呼び出し方の様式がわかるでしょう。

```go
type EnumParam struct {
	PackageName string
	Name        string
	Variants    []string
	Excepts     []EnumExceptParam
}

type EnumExceptParam struct {
	Name             string
	ExceptName       string
	ExcludedValiants []string
}

func capitalize(s string) string {
	if len(s) == 0 {
		return s
	}
	if len(s) == 1 {
		return strings.ToUpper(s)
	}
	return strings.ToUpper(s[:1]) + s[1:]
}

func replaceInvalidChar(s string) string {
	// As per Go programming specification.
	// identifier = letter { letter | unicode_digit }.
	// https://go.dev/ref/spec#Identifiers
	return strings.Map(func(r rune) rune {
		if unicode.IsLetter(r) || r == '_' || unicode.IsDigit(r) {
			return r
		}
		return '_'
	}, s)
}

func fillName(p EnumExceptParam, name string) EnumExceptParam {
	p.Name = name
	return p
}

func main() {
	pkgPath := filepath.Join("jennifer", "go-enum", "example")
	err := os.MkdirAll(pkgPath, fs.ModePerm)
	if err != nil {
		panic(err)
	}

	param := EnumParam{
		PackageName: "example",
		Name:        "Enum",
		Variants:    []string{"foo", "b\"ar", "baz"},
		Excepts: []EnumExceptParam{
			{
				ExceptName:       "foo",
				ExcludedValiants: []string{"foo"},
			},
			{
				ExceptName:       "Muh",
				ExcludedValiants: []string{"foo", "b\"ar"},
			},
		},
	}

	f := jen.NewFile(param.PackageName)

	f.PackageComment("// Code generated by me. DO NOT EDIT.")

	out, err := os.Create(filepath.Join(pkgPath, "enum.go"))
	if err != nil {
		panic(err)
	}

	f.Type().Id(param.Name).String() // type Enum string

	// const (
	f.Const().DefsFunc(func(g *jen.Group) {
		for _, variant := range param.Variants {
			g.
				Id(param.Name + replaceInvalidChar(capitalize(variant))). // EnumFoo
				Id(param.Name).                                           // Enum
				Op("=").                                                  // =
				Lit(variant)                                              // "foo"\n
		}
	}) // )

	// var _EnumAll = [...]Enum
	f.Var().Id("_" + param.Name + "All").Op("=").Index(jen.Op("...")).Id(param.Name).
		ValuesFunc(func(g *jen.Group) { // {
			for _, variant := range param.Variants {
				g.Line().Id(param.Name + replaceInvalidChar(capitalize(variant))) // EnumFoo,
			}
			g.Line() // \n
		}) // }

	// func IsEnum(v Enum) bool
	f.Func().Id("Is" + param.Name).Params(jen.Id("v").Id(param.Name)).Bool().Block( // {
		jen.Return( // return
			jen.Qual("slices", "Contains").Call( // slices.Contains
				jen.Id("_"+param.Name+"All").Index(jen.Op(":")), // _EnumAll[:],
				jen.Id("v"), // v,
			),
		),
	) // }

	f.Line()

	for _, except := range param.Excepts {
		except = fillName(except, param.Name)
		// func IsEnumExceptFoo(v Enum) bool
		f.Func().Id("Is" + except.Name + "Except" + replaceInvalidChar(capitalize(except.ExceptName))).Params(jen.Id("v").Id(except.Name)).Bool().Block( // {
			jen.Return( // return
				jen.Op("!").Qual("slices", "Contains").Params( // !slice.Contains(
					jen.Line().Index().Id(except.Name).ValuesFunc(func(g *jen.Group) { //[]Enum{
						for _, e := range except.ExcludedValiants {
							g.Line().Id(param.Name + replaceInvalidChar(capitalize(e))) // EnumFoo,
						}
						g.Line()
					}), // },
					jen.Line().Id("v"), // v,
					jen.Line(),
				), // )
			),
		) // }
		f.Line()
	}

	err = f.Render(out)
	if err != nil {
		panic(err)
	}
}
```

## おわりに

前段の記事: [Goのcode generatorの作り方: 諸注意とtext/templateの使い方](https://zenn.dev/ngicks/articles/go-code-generation-in-ways-text-template)で

- Rationale: なぜGoでcode generationが必要なのか
- code generatorを実装する際の注意点など
- `io.Writer`に書き出すシンプルな方法
- `text/template`を使う方法
  - `text/template`のcode generationにかかわりそうな機能性。
  - 実際に`text/template`を使ったcode generatorのexample。

を述べました。

この記事では`Go`のcode generatorを作るためのライブラリである[github.com/dave/jennifer]の使い方を軽く紹介し、`text/template`で実装したenumのcode generatorを`jennifer`で再実装しました。

さらに後続の記事で、それぞれ以下について説明します。

- [Goのcode generatorの作り方: ast(dst)を解析して書き換える](https://zenn.dev/ngicks/articles/go-code-generation-in-way-ast-dst)で[astutil](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/ast/astutil)および[github.com/dave/dst]を用いる方法

[github.com/dave/jennifer]では`Go`のトークンや構文に対応づいた関数をメソッドチェーンで順繰りに呼び出すことでコードを生成します。
`○○Func`, `Do`で関数を受け取れるため、ここで反復可能なデータを取り扱うことができ、さらに`Add`も組み合わせると適当にgeneratorを分割することができます。
また`Qual`によって自動的にimport declが管理されるため、同名のパッケージをインポートする際の名前被りも自動的に回避されます。

`jennifer`は書きやすい反面、ユーザーから部分的なcode generatorを受けとって挙動をカスタマイズさせるような機能を作りづらいため、そういった機能が必要なケースでは`text/template`を組み合わせて使用するのがよいかと思います。

[Go]: https://go.dev/
[Rust]: https://www.rust-lang.org
[Visual Studio Code]: https://code.visualstudio.com/
[vscode]: https://code.visualstudio.com/
[github.com/dave/jennifer]: https://github.com/dave/jennifer
[github.com/dave/dst]: https://github.com/dave/dst
[text/template]: https://pkg.go.dev/text/template@go1.22.6
[go/ast]: https://pkg.go.dev/go/ast@go1.22.6
[golang.org/x/tools/go/packages]: https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages
