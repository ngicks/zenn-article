---
title: "Goのcode generation: jennifer"
emoji: "🧰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## Goのcode generationについてまとめる

`Go`のcode generationについてまとめようと思います。

前段の記事: [Goのcode generation: text/template](https://zenn.dev/ngicks/articles/go-code-generation-in-ways-text-template)で

- Rationale: なぜGoでcode generationが必要なのか
- code generatorを実装する際の注意点など
- `io.Writer`に書き出すシンプルな方法
- `text/template`を使う方法
  - `text/template`のcode generationにかかわりそうな機能性について説明します。
  - 実際に`text/template`を使ってcode generatorを実装します。

について述べました。

この記事では

- [github.com/dave/jennifer]を用いたcode generatorの実装

について述べます

さらに後続の記事で

- [Goのcode generation: ast(dst)-rewrite](https://zenn.dev/ngicks/articles/go-code-generation-in-way-ast-dst)で[astutil](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/ast/astutil)および[github.com/dave/dst]を用いる方法

についてそれぞれ述べます。

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

このライブラリは`Go`のトークンや構文に対応づいた関数群をメソッドチェインで呼び出すことでコードを生成していきます。
対応づいているのですぐにすらすらかけるようになると思いますし、[Qual](https://github.com/dave/jennifer?tab=readme-ov-file#qual)によって自動的にimport declが追加されていくので超便利です。
`foobarFunc`系のメソッドで関数を受けとることができるので容易にfor-loopを回した生成が可能です。

### 利点と欠点

利点:

- 書きやすい
  - import declを自動的に調節してくれるのでimportの名前かぶりに関して気を使う必要がない。
- ごちゃごちゃしてるように見えてメンテしやすい(体感上)
- 単なる`Go`コードであるので任意に分割して再利用できる

欠点:

- std外のライブラリをインポートしてしまう。
- ユーザーからファイルを通して入力を受けとる方法が特に決まっていない

### 基本的な使用方法

[README.md](https://github.com/dave/jennifer?tab=readme-ov-file#jennifer)でしっかり説明がなされているので特に説明することはないかと思います、APIの様式がわかる程度のことを書いておいたほうが読みやすいかもしれないので先にここでそれについて述べておきます。

#### 宣言、書き出し

基本的には`jen.NewFile`,`jen.NewFilePath`,`jen.NewFilePathName`のいずれかでファイルを作り、そこからメソッドをいろいろ呼び出します。最後に`(*jen.File).Render`でファイルに生成したコードを書き出して終了します。

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

`Render`は[NoFormat](https://github.com/dave/jennifer/blob/3f94e7e1799d54504d53f8f56a079d2e2353a4cb/jen/file.go#L64)を`true`にしない限り[format.Sourceによってフォーマットをかける](https://github.com/dave/jennifer/blob/3f94e7e1799d54504d53f8f56a079d2e2353a4cb/jen/jen.go#L81-L89)挙動があります。
一旦`*bytes.Buffer`に内容を受けると、`truncate(2)`によって書き出し先のファイルが0byteにtruncateされたのちフォーマットエラーによって何も書きだされないのを防ぐことができます。
理屈上別ファイルに書き出して`rename(2)`しない限り電断などのabnormal exitで変更途中のファイル状態は観測しうるのであんまり気にしないでもいいといえばいいかも。

#### print

デバッグ用途として`GoString`が実装されており、それによってコード断片状態の`*jen.Statement`を書き出すことができます。
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

[\*jen.File](https://pkg.go.dev/github.com/dave/jennifer/jen@v1.7.0#File)は[jen.NewFile](https://pkg.go.dev/github.com/dave/jennifer/jen@v1.7.0#NewFile),[jen.NewFilePath](https://pkg.go.dev/github.com/dave/jennifer/jen@v1.7.0#NewFilePath),[jen.NewFilePathName](https://pkg.go.dev/github.com/dave/jennifer/jen@v1.7.0#NewFilePathName)のいずれかで作成します。

それぞれは以下のように[Qual](https://github.com/dave/jennifer/tree/master?tab=readme-ov-file#qual)を使った場合の挙動が違います。
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

`*jen.File`の`PackageComment`で`package`キーワードより先にコメントを書き出します。

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

#### fooFunc

`○○Func`や[Do](https://pkg.go.dev/github.com/dave/jennifer/jen@v1.7.0#Do)を用いると関数を受けることができるので、ここでfor-loopを回すなりするとよいでしょう。

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
		g.Id(f.name).Add(f.def)
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

ただしこの方法で差し込まれたコード片はimport declを更新できないので新しいimportがここで追加される場合うまく機能しません。
import declの内容を追加するのは筆者が見たところ`Qual`のみです。

かなり邪道ですね。

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

## まとめ

## おわりに

[Go]: https://go.dev/
[C++]: https://en.wikipedia.org/wiki/C%2B%2B
[Node.js]: https://nodejs.org/en
[TypeScript]: https://www.typescriptlang.org/
[python]: https://www.python.org/
[Rust]: https://www.rust-lang.org
[The Rust Programming Language 日本語]: https://doc.rust-jp.rs/book-ja/
[Visual Studio Code]: https://code.visualstudio.com/
[vscode]: https://code.visualstudio.com/
[git]: https://git-scm.com/
[github.com/dave/jennifer]: https://github.com/dave/jennifer
[github.com/dave/dst]: https://github.com/dave/dst
[text/template]: https://pkg.go.dev/text/template@go1.22.5
[go/ast]: https://pkg.go.dev/go/ast@go1.22.5
[golang.org/x/tools/go/packages]: https://pkg.go.dev/golang.org/x/tools@v0.23.0/go/packages
