---
title: "Goのcode generation: ast(dst)-rewrite"
emoji: "🧰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## Goのcode generationについてまとめる

`Go`のcode generationについてまとめようと思います。

前段の記事の

- [Goのcode generation: text/template](https://zenn.dev/ngicks/articles/go-code-generation-in-ways-text-template)で
  - Rationale: なぜGoでcode generationが必要なのか
  - code generatorを実装する際の注意点など
  - `io.Writer`に書き出すシンプルな方法
  - `text/template`を使う方法
    - `text/template`のcode generationにかかわりそうな機能性。
    - 実際に`text/template`を使ったcode generatorのexample。
- [Goのcode generation: jennifer](https://zenn.dev/ngicks/articles/go-code-generation-in-ways-jennifer)で
  - [github.com/dave/jennifer]の各機能
  - `text/template`で実装したcode generatorのexampleを`jennifer`で再実装

についてそれぞれ述べました。

この記事では

- [astutil](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/ast/astutil)および[github.com/dave/dst]を用いてgo source codeをrewriteする方法

について述べます

- astのパーズ方法
  - `go/parser`を用いる方法
  - [golang.org/x/tools/go/packages]を用いる方法
- 軽いastの解析方法やデバッグ方法
  - ast構造のprint: ast.Print
  - directive commentの解析
  - astのtraverse方法
- `astutil.Apply`でgo source codeのrewriteを実装します
- `astutil.Apply`ではコメントオフセットの狂いによってコメントの順序がおかしくなる問題について述べ
- [github.com/dave/dst]によってこの問題を起さずにast rewriteができることを述べます。
  - `dst`の紹介
  - astと`dst`の相互変換
  - `dst`でのコメントの取り扱い方法について
  - `dstutil.Apply`を使ったrewrite

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

## ast(dst)-rewrite

１からastをくみ上げることでコードを生成することもできますが、それをやるならば上記の`text/template`か`github.com/dave/jennifer`を用いるほうが楽なはずなので、ここでは深く紹介しません。
その代わり、astをもとにそれをrewriteする方法のみを取り扱います。

### 利点と欠点

- 利点
  - 既存のgo source codeを入力とできる。
- 欠点
  - astの変更や、１からastをくみ上げるのは手間がかかる
  - `Go`のソースコードを直接書きに行くほかの方法に比べてたった1つのトークンを書くだけでも何倍もの文字を打つ必要があってかなり面倒です。
  - そのためこの記事ではrewriteする方法しか想定しません。

`text/template`や[github.com/dave/jennifer]を使う方法に比べてずいぶん面倒です。
ではなぜこんなことをわざわざするのかというと

- editorのextensionを実装して、code actionとしてsource codeを書き換えたい
- code generatorの出力結果をさらに修正したい
- ユーザーの体験のため;
  - ある`Go`のtypeに対して何かの生成を行いたいとき、生成元は`Go`で書くのが最も一直線です。
  - `text/template`などを使う方法であげたYAMLやJSONのメタデータを書く方法では、メタデータから生成結果の想像がつかないとやや書きづらくなります。

仕事でcode generatorを実装する際には筆者的に正当化しずらい費用対効果なので(メタデータをYAMLなどで書かせる方法のコスパがよすぎるため)、なかなか実装する機会がありませんが、体験はいいので慣れておきたいと筆者的には思っていました。

### Go source codeの解析

source codeを解析してastをえる方法について述べます。

- 文字列・単一のファイルに対しては`go/parser`の[parser.ParseFile](https://pkg.go.dev/go/parser@go1.22.6#ParseFile)を利用します
- 複数のファイル・パッケージを解析するには[golang.org/x/tools/go/packages]を利用します。

#### go/parser

astは`go/token`, `go/parser`を用いて解析します。

`Go`のastはastと言いながら各Exprの位置情報が記録されています。これはlinter・その他で構文エラーの位置を表示するため、さらに`go/printer`による逆変換ができるようにするためなどの理由があるのだと思います。

1ファイルのみを読み込むには以下のようにします。

出力は少々長くなるので省略しました。なので、以下のplaygroundで実行するか、[ソース](https://github.com/ngicks/go-example-code-generation/tree/main/ast/print/ast)をコピーしてローカルで実行してみてください。

[playground](https://go.dev/play/p/cTZCHKH2pXA)

```go
package main

import (
	"go/ast"
	"go/parser"
	"go/token"
)

const src = `package target

import "fmt"

type Foo string

const (
	FooFoo Foo = "foo"
	FooBar Foo = "bar"
	FooBaz Foo = "baz"
)

func Bar(x, y string) string {
	if len(x) == 0 {
		return y + y
	}
	return fmt.Sprintf("%q%q", x, y)
}

type Some[T, U any] struct {
	Foo string
	Bar T
	Baz U
}

func (s Some[T, U]) Method1() {
	// ...nothing...
}

// comment slash slash


/*

comment slash star

*/
`

func main() {
	fset := token.NewFileSet()
	f, err := parser.ParseFile(fset, "./target/foo.go", src, parser.AllErrors|parser.ParseComments)
	if err != nil {
		panic(err)
	}
	_ = ast.Print(fset, f)
}
```

`token.NewFileSet`で[\*token.FileSet](https://pkg.go.dev/go/token@go1.22.6#FileSet)をallocateして、[parser.ParseFile](https://pkg.go.dev/go/parser@go1.22.6#ParseFile)で第３引数を解析します。ドキュメントにある通り、第3引数は[nil, []byte, string, io.Readerのいずれかを受け付け, nilの場合第二引数のfilenameを読み込みます](https://github.com/golang/go/blob/go1.22.6/src/go/parser/interface.go#L24-L42)。

#### ast.Print

解析された[\*ast.File](https://pkg.go.dev/go/ast@go1.22.6#File)を[ast.Print](https://pkg.go.dev/go/ast@go1.22.6#Print)もしくは[ast.Fprint](https://pkg.go.dev/go/ast@go1.22.6#Fprint)に渡すことで内部の構造をプリントすることができます。
これは要するに`reflect`によってgo structをwalkしながらprintする関数です。

前述通り上記サンプルコードの出力結果は長いので省略しますが、抜粋して一部を以下に例示します。

```go
type Some[T, U any] struct {
	//...
}
```

は以下のようなastになります。

```
// ...
   287  .  .  4: *ast.GenDecl {
   288  .  .  .  TokPos: ./target/foo.go:20:1
   289  .  .  .  Tok: type
   290  .  .  .  Lparen: -
   291  .  .  .  Specs: []ast.Spec (len = 1) {
   292  .  .  .  .  0: *ast.TypeSpec {
   293  .  .  .  .  .  Name: *ast.Ident {
   294  .  .  .  .  .  .  NamePos: ./target/foo.go:20:6
   295  .  .  .  .  .  .  Name: "Some"
   296  .  .  .  .  .  .  Obj: *ast.Object {
   297  .  .  .  .  .  .  .  Kind: type
   298  .  .  .  .  .  .  .  Name: "Some"
   299  .  .  .  .  .  .  .  Decl: *(obj @ 292)
   300  .  .  .  .  .  .  }
   301  .  .  .  .  .  }
   302  .  .  .  .  .  TypeParams: *ast.FieldList {
   303  .  .  .  .  .  .  Opening: ./target/foo.go:20:10
   304  .  .  .  .  .  .  List: []*ast.Field (len = 1) {
   305  .  .  .  .  .  .  .  0: *ast.Field {
   306  .  .  .  .  .  .  .  .  Names: []*ast.Ident (len = 2) {
   307  .  .  .  .  .  .  .  .  .  0: *ast.Ident {
   308  .  .  .  .  .  .  .  .  .  .  NamePos: ./target/foo.go:20:11
   309  .  .  .  .  .  .  .  .  .  .  Name: "T"
   310  .  .  .  .  .  .  .  .  .  .  Obj: *ast.Object {
   311  .  .  .  .  .  .  .  .  .  .  .  Kind: type
   312  .  .  .  .  .  .  .  .  .  .  .  Name: "T"
   313  .  .  .  .  .  .  .  .  .  .  .  Decl: *(obj @ 305)
   314  .  .  .  .  .  .  .  .  .  .  }
   315  .  .  .  .  .  .  .  .  .  }
// ...
```

[\*ast.GenDecl](https://pkg.go.dev/go/ast@go1.22.6#GenDecl)は

> A GenDecl node (generic declaration node) represents an import, constant, type or variable declaration. A valid Lparen position (Lparen.IsValid()) indicates a parenthesized declaration.

であるので、`*ast.File`のトップレベルにあるものは関数宣言以外はすべてこれになります。`var`や`type`は`var()`でグループを持てるため、`Spec`フィールドは`[]ast.Spec`というsliceになっています。`()`によるグルーピングがかかっていない場合はparenthesis(`(`)がないわけですからこのast nodeには`Lparen`と`Rparen`(省略されて表示されていないが)にはemptyな値が収められています。

`TypeParam`のindex(`[]`)の中身はstructの１つのfieldと同じ構文ルールが適用できるので`ast.FieldList`が使われていますね。ここはちょっと筆者的には驚きでした。

とまあそういった感じです。

#### golang.org/x/tools/go/packages

サンプルは以下でもホストされます

https://github.com/ngicks/go-example-code-generation/tree/main/ast/parse-by-packages

少し前ではあるパッケージ、つまりディレクトリの中にあるsource fileを一気に解析するには[ast.ParseDir](https://pkg.go.dev/go/parser@go1.22.6#ParseDir)を使えばよかったのですが、これが返す[\*ast.Package](https://pkg.go.dev/go/ast@go1.22.6#Package)が[Go1.22からdeprecatedになっている](https://tip.golang.org/doc/go1.22#minor_library_changes)ため、ディレクトリの中身を一気にパーズしたいとき何使えばいいんだよってなりますよね。

困ったので[github.com/golang/example](https://github.com/golang/example)を見ていると、[このコミット](https://github.com/golang/example/commit/1d6d2400d4027025cb8edc86a139c9c581d672f7)で[golang.org/x/tools/go/packages]を勧める文章に変わっていました。

ということで複数パッケージを一気にパーズするには[golang.org/x/tools/go/packages]を使えばよさそうですね。

```go
package main

import (
	"context"
	"fmt"
	"os"
	"os/signal"
	"syscall"

	"golang.org/x/tools/go/packages"
)

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()
	cfg := &packages.Config{
		Mode: packages.NeedName |
			packages.NeedFiles |
			packages.NeedCompiledGoFiles |
			packages.NeedImports |
			packages.NeedDeps |
			packages.NeedExportFile |
			packages.NeedTypes |
			packages.NeedSyntax |
			packages.NeedTypesInfo |
			packages.NeedTypesSizes |
			packages.NeedModule |
			packages.NeedEmbedFiles |
			packages.NeedEmbedPatterns,
		Context: ctx,
		Logf: func(format string, args ...interface{}) {
			fmt.Printf("log: "+format, args...)
			fmt.Println()
		},
	}
	pkgs, err := packages.Load(cfg, "io", "./ast/parse-by-packages/target")
	if err != nil {
		panic(err)
	}
}
```

こんな感じでモジュールをロードします。

[\*packages.Config](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages#Config)をいろいろ設定し、[packages.Load](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages#Load)の第一引数として渡します。第二引数はvariadicなパターンで、`go`コマンドに渡すようなpackage patternを渡して読み込みたいパッケージを指定できます。

##### pattern

[packages.Load](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages#Load)の第二引数にはvairadicなpatternを渡し、これによってロードするパッケージを指定します。

> https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages#Package
>
> Load passes most patterns directly to the underlying build tool. The default build tool is the go command. Its supported patterns are described at https://pkg.go.dev/cmd/go#hdr-Package_lists_and_patterns. Other build systems may be supported by providing a "driver"; see [The driver protocol].
>
> All patterns with the prefix "query=", where query is a non-empty string of letters from [a-z], are reserved and may be interpreted as query operators.
>
> Two query operators are currently supported: "file" and "pattern".
>
> The query "file=path/to/file.go" matches the package or packages enclosing the Go source file path/to/file.go. For example "file=~/go/src/fmt/print.go" might return the packages "fmt" and "fmt [fmt.test]".
>
> The query "pattern=string" causes "string" to be passed directly to the underlying build tool. In most cases this is unnecessary, but an application can use Load("pattern=" + x) as an escaping mechanism to ensure that x is not interpreted as a query operator if it contains '='.

デフォルトでこれらを引数に[go list](https://pkg.go.dev/cmd/go#hdr-List_packages_or_modules)を呼び出すので、それに渡すことができるパターンを指定することができます。具体的に言うと`./...`で`cwd`以下のすべてのパッケージにマッチさせることができます。

##### \*package.Config

[packages.Load](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages#Load)の第一引数には[\*packages.Config](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages#Config)を渡します。

ちょっとわかりにくいところがあるのでそこだけ説明します。

- `Mode`: ビットフラグで何をロードするのか制御します。
  - 前述のサンプルが現時点(`v0.24.0`)でexportされている全てです。
  - [LoadMode](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages#LoadMode)のdoc commentにある通り現時点ではバグがあるみたいです。
- `Dir`: `go list`などの`cwd`を指定できます。
  - `Dir`が指定するパッケージの`go.mod`に、`Load`に渡すパターンに一致するモジュールがないと以下のようなエラーを吐きます。

```
github.com/hack-pad/hackpadfs: packages.Error{Pos:"", Msg:"no required module provides package github.com/hack-pad/hackpadfs; to add it:\n\tgo get github.com/hack-pad/hackpadfs", Kind:1}
```

- `Overlay`
  - コードを追ってみる限り内容をtemp fileとしてそれぞれ書き出し、`-overlay`オプションに渡すことができるjsonファイルを書き出してから`-overlay`オプションに書き出したjsonファイルのパスを渡します。
  - `-overlay`オプションそのものは今回の話題に対して重要ではないので、ここでは説明を避け[go Command Documentation](https://pkg.go.dev/cmd/go)を読むようにとだけ書いておきます。

##### packages.Visit

[packages.Visit](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages#Visit)で、ロードされたパッケージをインポートグラフ順にvisitできます。

```go
	pkgs, err := packages.Load(cfg, "io", "./ast/parse-by-packages/target")
	if err != nil {
		panic(err)
	}
	packages.Visit(
		pkgs,
		func(p *packages.Package) bool {
			for _, err := range p.Errors {
				fmt.Printf("pkg %s: %#v\n", p.PkgPath, err)
			}
			if len(p.Errors) > 0 {
				return true
			}

			fmt.Printf("package path: %s\n", p.PkgPath)

			return true
		},
		nil,
	)
	/*
package path: io
package path: errors
package path: internal/reflectlite
package path: internal/abi
package path: internal/goarch
package path: unsafe
package path: internal/unsafeheader
// 中略
package path: internal/race
package path: sync/atomic
package path: github.com/ngicks/go-example-code-generation/ast/parse-by-packages/target
package path: fmt
package path: internal/fmtsort
package path: reflect
package path: internal/itoa
// 中略
package path: internal/testlog
package path: io/fs
package path: path
*/
```

##### pkgs[i].Syntax: []\*ast.File

[packages.Load](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages#Load)の返り値は`[]*packages.Package`です。
[\*packages.Package](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages#Package)の各フィールドが解析結果です。

`Load`時、`LoadMode`に`packages.NeedSyntax`をつけると

- `Fset`フィールド: `*token.FileSet`
- `Syntax`フィールド: `[]*ast.File`

が読み込まれます。これによりパッケージ内のファイルすべてを`parser.ParseFile`するのと同等の挙動が得られます。

当然`ast.Print`などast情報を使った処理ができます。

```go
	pkgs, err := packages.Load(cfg, "io", "./ast/parse-by-packages/target")
	if err != nil {
		panic(err)
	}

	targetPkg := pkgs[1]

	for _, f := range targetPkg.Syntax {
		ast.Print(targetPkg.Fset, f)
		fmt.Printf("\n\n")
	}
```

##### pkgs[i].Types: \*types.Package

`Load`時、`LoadMode`に`packages.NeedTypes`をつけると`Types`フィールドに`*types.Package`が解析結果として代入されます。
これにより型情報を使った処理を行うことができます。

```go
	pkgs, err := packages.Load(cfg, "io", "./ast/parse-by-packages/target")
	if err != nil {
		panic(err)
	}

	ioPkg := pkgs[0]
	targetPkg := pkgs[1]

	foo := targetPkg.Types.Scope().Lookup("Foo")
	fmt.Printf("foo: %#v\n", foo)
	// foo: &types.TypeName{object:types.object{parent:(*types.Scope)(0xc004758660), pos:4034135, pkg:(*types.Package)(0xc0047586c0), name:"Foo", typ:(*types.Named)(0xc0067f0930), order_:0x2, color_:0x1, scopePos_:0}}

	r := ioPkg.Types.Scope().Lookup("Reader")

	pfoo := types.NewPointer(foo.Type())

	if types.AssignableTo(pfoo, r.Type()) {
		fmt.Printf("%s satisfies %s\n", pfoo, r)
		// *github.com/ngicks/go-example-code-generation/ast/parse-by-packages/target.Foo satisfies type io.Reader interface{Read(p []byte) (n int, err error)}
	}

	mset := types.NewMethodSet(pfoo)
	for i := 0; i < mset.Len(); i++ {
		meth := mset.At(i).Obj()
		sig := meth.Type().Underlying().(*types.Signature)
		fmt.Printf(
			"%d: func (receiver=%s name=*%s)(func-name=%s)(params=%s) (results=%s)\n",
			i, sig.Recv().Name(), foo.Name(), meth.Name(), sig.Params(), sig.Results(),
		)
		// 0: func (receiver=f name=*Foo)(func-name=Read)(params=(p []byte)) (results=(int, error))
	}
```

`types`の話はちょっとしたおまけなのでこれ以上詳しくしません。

### directive commentとその解析方法

`Go`はマジックコメントで[Compiler directiveを書貸せるようになっています。](https://pkg.go.dev/cmd/compile#hdr-Compiler_Directives)
このdirective commentはdoc commentに出現しないようです。なので、directive commentでcode generatorに対して指示を出せるとdoc commentを邪魔せず、メタデータを別ファイルに分けることなく`Go`のsource codeに追加できるため便利かもしれません。
ただし、directive commentの解析をastから行うには若干の工夫がいるので以下でその方法を述べます。

#### directive comment

通常のコメントは下記のように、コメントの開始に半角スペースなどを１つ以上入れますが、directive commentは半角などを入れません。

```go
// non-directive comment
/* non-directive comment */

//directive comment
```

[compiler directive](https://pkg.go.dev/cmd/compile#hdr-Compiler_Directives)の項目では説明されていませんが、`//go:embed`のように他にも色々なマジックコメントが存在します。

[staticcheckの//lint:ignore directive](https://staticcheck.dev/docs/configuration/#line-based-linter-directives)や、[golangci-lintのnolint directive](https://golangci-lint.run/usage/false-positives/#nolint-directive)などサードパーティのツール、特にlinterなどがこのdirective commentを利用して挙動の調節が行えるようになっています。

#### \*ast.CommentGroupの解析方法

directive commentは[\*ast.CommentGroup](https://pkg.go.dev/go/ast@go1.22.6#CommentGroup)の[Textメソッド](https://pkg.go.dev/go/ast@go1.22.6#CommentGroup.Text)から除外される挙動があるため、これを解析したい場合は`List`を直接走査する必要があります。

[playground](https://go.dev/play/p/ELsSS2X18bl)

```go
package main

import (
	"fmt"
	"go/parser"
	"go/token"
)

const src = `package target

// non-directive:comment

/*non-directive:comment*/

//directive:comment

/*line foo: 10 */

`

func main() {
	fset := token.NewFileSet()
	f, err := parser.ParseFile(fset, "./target/foo.go", src, parser.AllErrors|parser.ParseComments)
	if err != nil {
		panic(err)
	}
	for _, cg := range f.Comments {
		fmt.Println(fset.Position(cg.Pos()), cg.Text())
	}
	/*
		./target/foo.go:3:1 non-directive:comment

		./target/foo.go:5:1 non-directive:comment

		./target/foo.go:7:1

	*/

	fmt.Println("---")

	for _, cg := range f.Comments {
		fmt.Print(fset.Position(cg.Pos()))
		for _, comment := range cg.List {
			fmt.Println(comment.Text)
		}
	}
	// ./target/foo.go:3:1// non-directive:comment
	// ./target/foo.go:5:1/*non-directive:comment*/
	// ./target/foo.go:7:1//directive:comment
}
```

[この行でdirective commentが除外されますが](https://github.com/golang/go/blob/go1.22.6/src/go/ast/ast.go#L110-L130)、`/**/`スタイルのコメントには全く機能しないので、上記のCompiler directiveのドキュメントにも反した挙動のように思えます。なるだけ`/**/`スタイルのコメントは使わないほうが混乱が少なくていいのかもしれません。

### astのtraverse方法

astを解析して得ることができても、その中から特定の探したいパターンを探せなければ意味のある処理を行うことができません。

そこで`Go`は以下の関数などでastをトラバースする方法を提供しています。

- [ast.Inspect](https://pkg.go.dev/go/ast@go1.22.6#Inspect)
  - astをwalkします
- [ast.Walk](https://pkg.go.dev/go/ast@go1.22.6#Walk)
  - astをwalkしますがこちらは[Visitor interface](https://pkg.go.dev/go/ast@go1.22.6#Visitor)を受けるため、ステートマシン的に状態を切り替えられます。
- [\*inspector.Inspector](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/ast/inspector#Inspector)
  - 型(`*ast.TypeSpec`など)によるフィルターをかけたnodeの探索を行います。
  - [WithStack](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/ast/inspector#Inspector.WithStack)でマッチしたnodeに到達するまでのrootからのast nodeのstackを取得できるので、上位エレメントの構造がこうなら、みたいな条件付けでマッチできるのだと思います。
  - この記事を書くための調査で知った機能なので、あまり使ったことがありません。そのため詳しくはわかりません。
- [astutil.Apply](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/ast/astutil#Apply)
  - astをwalkしますが、あるast nodeにstep inする前に呼び出されるコールバック(`pre`)と、walkしきあったとに呼び出されるコールバック(`post`)を渡して処理を行えます。
  - [ApplyFunc](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/ast/astutil#ApplyFunc)(`pre`と`post`)には[\*Cursor](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/ast/astutil#Cursor)が渡され、これによってast nodeを`Delete`, `Replace`などができます。
    - また、[\*ast.File](https://pkg.go.dev/go/ast@go1.22.6#File)の`Decls`フィールドや、[\*ast.FieldList](https://pkg.go.dev/go/ast@go1.22.6#FieldList)の`List`フィールドのようなsliceにnodeを挿入する`InsertBefore`, `InsertAfter`があります。

### astutil.Applyを使ったrewrite

コードサンプルは以下でホストされます

https://github.com/ngicks/go-example-code-generation/blob/main/ast/rewrite/astutil

[astutil.Apply](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/ast/astutil#Apply)を使ったrewriteのサンプルを示します。
今までのサンプルと違い、新しいファイルに書き出すよりも既存のsource codeをrewriteするものを想定します。
この理由は

- そのほうが後述の問題が噴出しやすい
- 記事の目的から微妙にそれるが、editorのcode actionでコードを書き換える物を作りたいのでその前段として

です。

#### 仕様

ざっくり以下のような仕様で作ります。

- string-based typeのみを生成対象とする
- `//enum:variants=`から始まるコメントでcomma separatedなvariantsを記述できる
  - これはdirective commentでもよい
- このvariantsを元に、変数名を`型名+variant`、値がvariantのstring literalを`const ()`で列挙する
- すでに生成済みのconstの`*ast.GenDecl`がある場合はそれを`Replace`し、ない場合は追加します。
  - 生成された`*ast.GenDecl`ブロックに`//enum:generated_for=型名`をつけることで識別可能にします。

#### ターゲット

生成のターゲットを以下とします。

```go
package target

//enum:variants=foo,bar,baz
type Enum string

//enum:variants=foo,bar,baz
type Enum2 string

//enum:generated_for=Enum2
const (
	Enum2Foo = "foo"
)
```

#### parse

前述の[golang.org/x/tools/go/packages]を使ってロードするものとします。

```go
	cfg := &packages.Config{
		Mode: packages.NeedName |
			packages.NeedImports |
			packages.NeedTypes |
			packages.NeedSyntax,
	}
	pkgs, err := packages.Load(cfg, "./ast/rewrite/target")
	if err != nil {
		panic(err)
	}

	pkg := pkgs[0]
```

#### astutil.Apply

astをrewriteするのでSyntaxをfor-rangeします。

`astutil.Apply`では、あるast nodeにstep inする前に呼び出されるコールバック(`pre`)と、walkしきあったとに呼び出されるコールバック(`post`)を渡して処理を行えます。
今回は`pre`のみを用います。

今回は`*ast.GenDecl`を探索するので`*ast.File`を`astutil.Apply`に渡します。
その際のステップ順序は[package comment, package name, Declsの順](https://github.com/golang/tools/blob/v0.24.0/go/ast/astutil/rewrite.go#L429-L436)であるのでdefault branchで`return true`しないとうまいこと進んでくれません。

```go
	for _, f := range pkg.Syntax {
		astutil.Apply(
			f,
			func(c *astutil.Cursor) bool {
				n := c.Node()
				switch x := n.(type) {
				default:
					return true
				case *ast.FuncDecl:
				case *ast.GenDecl:
					// ...
				}
				return false
			},
			nil,
		)
	}
```

#### 対象タイプの探索

ますこの`Apply`の中で`//enum:variants=`のマジックコメントがついたtype specを探します。

今回は~~めんどくさいので~~簡易化のため、`type ()`で複数のtype specを書くパターンを禁止し、`type Foo string`な単体の`GenDecl`のみを対象とします。

```go
	astutil.Apply(
		f,
		func(c *astutil.Cursor) bool {
			n := c.Node()
			switch x := n.(type) {
			default:
				return true
			case *ast.FuncDecl:
			case *ast.GenDecl:
				if x.Tok != token.TYPE {
					break
				}

				if len(x.Specs) == 1 {
					name, ok := isStringBasedType(x.Specs[0])
					if !ok {
						break
					}
					param, ok := parseDirective(x.Doc)
					if !ok {
						break
					}
					param.Name = name
					addOrReplaceEnum(c, param)
				}
			}
			return false
		},
		nil,
	)
```

時々忘れちゃいますが、defaultでfallthroughが起きないだけで`Go`のswitch-case文はbreakが使えます。

`type Foo string`なstring-based typeかどうかは以下のように判定します。
こういった判定は、astを`ast.Print`して確かめた通りに実装するとよいです。

```go
func isStringBasedType(spec ast.Spec) (string, bool) {
	typ, ok := spec.(*ast.TypeSpec)
	if !ok {
		return "", false
	}
	id, ok := typ.Type.(*ast.Ident)
	if !ok {
		return "", false
	}
	return typ.Name.Name, id.Name == "string"
}
```

前述のとおり、`*ast.CommentGroup`の`Text`メソッドではdirective commentが除外されてしまうので`List`を走査します。

```go
type EnumParam struct {
	Name     string
	Variants []string
}

func parseDirective(cg *ast.CommentGroup) (EnumParam, bool) {
	for _, comment := range cg.List {
		c := strings.TrimLeftFunc(stripMarker(comment.Text), unicode.IsSpace)
		c, isDirection := strings.CutPrefix(c, "enum:variants=")
		if !isDirection {
			continue
		}
		return EnumParam{Variants: strings.Split(c, ",")}, true
	}
	return EnumParam{}, false
}

func stripMarker(text string) string {
	if len(text) < 2 {
		return text
	}
	switch text[1] {
	case '/':
		return text[2:]
	case '*':
		return text[2 : len(text)-2]
	}
	return text
}
```

#### Replace or Insert

replaceする部分です。

仕様で説明した通り、特定のコメントがついた`const ()`を探して、あれば置き換え、なければ追加します。

追加する際には`(*Cursor).InsertAfter`で、対象タイプの直後にコードを挿入したいため、対象となる`GenDecl`を指した状態の`*Cursor`をそのまま受け取れると都合がよいのでそうします。
既に作成された`const ()`ブロックを探すには、もう1度`Parent`=`*ast.File`を`Apply`で探索します。

```go
func addOrReplaceEnum(c *astutil.Cursor, param EnumParam) {
	found := false
	astutil.Apply(
		c.Parent(),
		func(c *astutil.Cursor) bool {
			node := c.Node()
			switch x := node.(type) {
			default:
				return true
			case *ast.FuncDecl:
			case *ast.GenDecl:
				if x.Tok != token.CONST {
					break
				}
				if !isGeneratedFor(x.Doc, param.Name) {
					break
				}
				found = true
				newDecl := astVariants(param, x.TokPos)
				c.Replace(newDecl)
			}
			return false
		},
		nil,
	)
	if !found {
		newDecl := astVariants(param, c.Node().(*ast.GenDecl).Specs[0].Pos()+30)
		c.InsertAfter(newDecl)
	}
}

func isGeneratedFor(cg *ast.CommentGroup, fotTy string) bool {
	for _, comment := range cg.List {
		c := strings.TrimLeftFunc(stripMarker(comment.Text), unicode.IsSpace)
		s, ok := strings.CutPrefix(c, "enum:generated_for=")
		if !ok {
			return false
		}
		if s == fotTy {
			return true
		}
	}
	return false
}
```

以下のようなブロックは

```go
const (
	EnumFoo = "foo"
	EnumBar = "bar"
)
```

astでは以下のように構成できます。

```go
func astVariants(param EnumParam, pos token.Pos) *ast.GenDecl {
	return &ast.GenDecl{
		Doc: &ast.CommentGroup{
			List: []*ast.Comment{
				{
					Slash: pos,
					Text:  "//enum:generated_for=" + param.Name,
				},
			},
		},
		TokPos: token.Pos(int(pos) + len("//enum:generated_for="+param.Name) + 1),
		Tok:    token.CONST,
		Lparen: 1,
		Specs:  mapParamToSpec(param),
		Rparen: 2,
	}
}

func mapParamToSpec(param EnumParam) []ast.Spec {
	specs := make([]ast.Spec, len(param.Variants))
	for i, variant := range param.Variants {
		specs[i] = &ast.ValueSpec{
			Names:  []*ast.Ident{{Name: param.Name + capitalize(variant)}},
			Type:   &ast.Ident{Name: param.Name},
			Values: []ast.Expr{&ast.BasicLit{Kind: token.STRING, Value: strconv.Quote(variant)}},
		}
	}
	return specs
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
```

#### node移動時にコメントの整合性を保つ: \*ast.CommentMap

> https://pkg.go.dev/go/ast@go1.22.6#File
>
> For correct printing of source code containing comments (using packages go/format and go/printer), special care must be taken to update comments when a File's syntax tree is modified: For printing, comments are interspersed between tokens based on their position. If syntax tree nodes are removed or moved, relevant comments in their vicinity must also be removed (from the [File.Comments] list) or moved accordingly (by updating their positions). A CommentMap may be used to facilitate some of these operations.

とある通り、nodeの移動や削除を行う場合`*ast.CommentMap`を使います。
上記にはnodeの追加に対しては何も言及がありません。おそらく、追加はこれを使ってもうまく動きません(後述)。
うまくいかないのであんまり紹介する意義がないかもですが、お作法として`*ast.CommentMap`を使うようにこのサンプルを書き換えます。

```diff go
	for _, f := range pkg.Syntax {
+		cm := ast.NewCommentMap(pkg.Fset, f, f.Comments)
		astutil.Apply(
			f,
			func(c *astutil.Cursor) bool {
				n := c.Node()
				switch x := n.(type) {
				default:
					return true
				case *ast.FuncDecl:
				case *ast.GenDecl:
					if x.Tok != token.TYPE {
						break
					}

					if len(x.Specs) == 1 {
						name, ok := isStringBasedType(x.Specs[0])
						if !ok {
							break
						}
						param, ok := parseDirective(x.Doc)
						if !ok {
							break
						}
						param.Name = name
-						addOrReplaceEnum(c, param)
+						addOrReplaceEnum(c, param, cm)
					}
				}
				return false
			},
			nil,
		)
+		f.Comments = cm.Comments()
	}
```

```diff go
-func addOrReplaceEnum(c *astutil.Cursor, param EnumParam) {
+func addOrReplaceEnum(c *astutil.Cursor, param EnumParam, cm ast.CommentMap) {
	found := false
	astutil.Apply(
		c.Parent(),
		func(c *astutil.Cursor) bool {
			node := c.Node()
			switch x := node.(type) {
			default:
				return true
			case *ast.FuncDecl:
			case *ast.GenDecl:
				if x.Tok != token.CONST {
					break
				}
				if !isGeneratedFor(x.Doc, param.Name) {
					break
				}
				found = true
				newDecl := astVariants(param, x.TokPos)
+				delete(cm, x)
+				cm[newDecl] = append(cm[newDecl], newDecl.Doc)
				c.Replace(newDecl)
			}
			return false
		},
		nil,
	)
	if !found {
		newDecl := astVariants(param, c.Node().(*ast.GenDecl).Specs[0].Pos()+30)
+		cm[newDecl] = append(cm[newDecl], newDecl.Doc)
		c.InsertAfter(newDecl)
	}
}
```

#### 生成

[printer.Fprint](https://pkg.go.dev/go/printer@go1.22.6#Fprint)で結果をプリントできます。

```go
	for _, f := range pkg.Syntax {
		filename := filepath.Base(pkg.Fset.Position(f.FileStart).Filename)

		astutil.Apply(
			f,
			func(c *astutil.Cursor) bool {
				// ...
			},
			nil,
		)

		out, err := os.Create(filepath.Join(generatedDir, filename))
		if err != nil {
			panic(err)
		}

		err = printer.Fprint(out, pkg.Fset, f)
		if err != nil {
			panic(err)
		}
	}
```

#### 結果

以下の入力をもとに

```go
package target

//enum:variants=foo,bar,baz
type Enum string

//enum:variants=foo,bar,baz
type Enum2 string

//enum:generated_for=Enum2
const (
	Enum2Foo = "foo"
)
```

以下を出力します。

```go
package target

//enum:variants=foo,bar,baz
type Enum string

//enum:variants=foo,bar,baz
//enum:generated_for=Enum
const (
	EnumFoo	Enum	= "foo"
	EnumBar	Enum	= "bar"
	EnumBaz	Enum	= "baz"
)

type Enum2 string

//enum:generated_for=Enum2
const (
	Enum2Foo	Enum2	= "foo"
	Enum2Bar	Enum2	= "bar"
	Enum2Baz	Enum2	= "baz"
)
```

なんかおかしいですね。

### astutil.Applyの問題点

`astutil.Apply`はdoc commentに特に触れられていないですが、うまくいかないパターンがあります。

#### 問題のあるパターン

例えば、下記のサンプルを前節のcode replacerにかけてみます。

```go
package target

// free floating comment 1

func Foo() {
	// nothing
}

//enum:variants=foo,bar,baz,qux,quux,corge
type EnumWithComments string

// free floating comment 2

func Bar() {
	// nothing
}

//enum:variants=foo,bar,baz
type EnumWithComments2 string

// free floating comment 3

//enum:generated_for=EnumWithComments2
const (
	EnumWithComments2Foo EnumWithComments2 = "foo"
)

/* free floating comment 4


 */
```

以下を出力します。コメントの位置関係が破綻しています！

```go
package target

// free floating comment 1

func Foo() {
	// nothing
}

//enum:variants=foo,bar,baz,qux,quux,corge
type EnumWithComments string

// free floating comment 2
//enum:generated_for=EnumWithComments

// nothing
const (
	EnumWithCommentsFoo	EnumWithComments	= "foo"
	EnumWithCommentsBar	EnumWithComments	= "bar"
	EnumWithCommentsBaz	EnumWithComments	= "baz"
	EnumWithCommentsQux	EnumWithComments	= "qux"
	EnumWithCommentsQuux	EnumWithComments	=

	//enum:variants=foo,bar,baz
	"quux"
	EnumWithCommentsCorge	EnumWithComments	= "corge"
)

func Bar() {

}

type EnumWithComments2 string

//enum:generated_for=EnumWithComments2
const (
	EnumWithComments2Foo	EnumWithComments2	= "foo"
	EnumWithComments2Bar	EnumWithComments2	= "bar"
	EnumWithComments2Baz	EnumWithComments2	= "baz"
)

/* free floating comment 4


 */
```

#### Commentはバイトオフセットで管理され、nodeの追加は想定されていない

実は`ast`パッケージにおけるコメントの表現はすべてバイトオフセットでしかなく、別段、前後のnodeに対する関連性が定義されていません。

`parser.ParseFile`や`printer.Fprint`が`token.FileSet`を引数にとることかわかる通り、ファイルのオフセット関係は`FileSet`の中に記録されます。この中で、パッケージ内のファイルをパッケージ内での絶対値オフセットに変更しています。

そのため、nodeを追加してしまうとオフセットの関係が狂って容易におかしな結果を出力してしまいます。

`*ast.GenDecl`などについているdoc commentとしてのコメントがついて回りますが、これは単に解析時にコメントのオフセットとトークン(`var`や`type`)のオフセットを比較して間に改行がない場合に関連しているとしてフィールドにセットしているだけのようです。

下記のstackoverflowでworkaround方法が述べられていますが、

https://stackoverflow.com/questions/31628613/comments-out-of-order-after-adding-item-to-go-ast

`parser.ParseFile`で解析する前に追加する分のバイトサイズだけをソース末尾をover-allocateしておき、nodeを追加するときに追加分だけ[AddLine](https://pkg.go.dev/go/token@go1.22.6#File.AddLine)で行を追加するとうまくいくようです。
あらかじめutf-8で何バイト追加するか判明していないと成立しないため、この方法でうまくやっていくビジョンが見えませんね。

しかしこのstackoverflowのaccepted answerにある通り、質問者自身がこの問題を解決するためのパッケージを作っており、これがうまく動作するので以降でその紹介をします。(よく見るとこの質問者は`jennifer`の作者のdaveです！)

### github.com/dave/dst

[github.com/dave/dst]は[github.com/dave/jennifer]と同作者が作ったコメントのオフセットを正しくキープしながらastの操作ができるライブラリです。

`ast`から変更が加わっており、コメントは前後のNodeに関連づくようになり、free floating commentの概念がなくなっています。

#### astからdstへの変換

https://github.com/ngicks/go-example-code-generation/blob/main/ast/print/dst

dstを利用するためには`*ast.File`をまず用意します。これは今まで通り`parser.ParseFile`を呼び出したり、[golang.org/x/tools/go/packages]を利用します。
以下のように`decorator`パッケージを利用して`*ast.File`を`*dst.File`に*decorate*します。

```go
package main

import (
	"go/parser"
	"go/token"

	"github.com/dave/dst"
	"github.com/dave/dst/decorator"
)

const src = `package target

import "fmt"

type Foo string

const (
	FooFoo Foo = "foo"
	FooBar Foo = "bar"
	FooBaz Foo = "baz"
)

func Bar(x, y string) string {
	if len(x) == 0 {
		return y + y
	}
	return fmt.Sprintf("%q%q", x, y)
}

type Some[T, U any] struct {
	Foo string
	Bar T
	Baz U
}

func (s Some[T, U]) Method1() {
	// ...nothing...
}

// comment slash slash


/*

comment slash star

*/
`

func main() {
	fset := token.NewFileSet()
	f, err := parser.ParseFile(fset, "./target/foo.go", src, parser.AllErrors|parser.ParseComments)
	if err != nil {
		panic(err)
	}
	dec := decorator.NewDecorator(fset)
	df, err := dec.DecorateFile(f)
	if err != nil {
		panic(err)
	}
	_ = dst.Print(df)
}
```

#### ast.Nodeとdst.Nodeの相互変換

[\*decorator.Decorator](https://pkg.go.dev/github.com/dave/dst/decorator#Decorator)および[\*decorator.Restorer](https://pkg.go.dev/github.com/dave/dst/decorator#Restorer)には[Map](https://pkg.go.dev/github.com/dave/dst/decorator#Map)フィールドがあり、`DecorateFile`や`RestoreFile`を呼び出し後にnodeの対応関係がマップに記録されるため、これによって相互変換ができます。

```go
	dec := decorator.NewDecorator(fset)
	_, err = dec.DecorateFile(f)
	if err != nil {
		panic(err)
	}

	// ast.Nodeと対応するdst.Nodeを取り出す。
	dn := dec.Dst.Nodes[f.Decls[0]]

	restorer := decorator.NewRestorer()
	_, err = restorer.RestoreFile(df)
	if err != nil {
		panic(err)
	}

	// dst.Nodeと対応するast.Nodeを取り出す。
	an := restorer.Ast.Nodes[dn]

	fmt.Println()
	err = printer.Fprint(os.Stdout, restorer.Fset, an) // import "fmt"
	if err != nil {
		panic(err)
	}
	fmt.Println()
```

### dstutil.Applyを使った書き換え

https://github.com/ngicks/go-example-code-generation/blob/main/ast/rewrite/dstutil

[github.com/dave/dst]には`astutil`に対応する`dstutil`パッケージがあり、ほぼ同じ使用感で実装されています。
前述の`astutil.Apply`のcode rewriterを`dstutil.Apply`を使って実装しなおします。

#### dstutil.Apply

[golang.org/x/tools/go/packages]を用いたロードまでは全く一緒です。
dstutil.Applyの前に`decorator.DecorateFile`

```diff go
	for _, f := range pkg.Syntax {
+		df, err := decorator.DecorateFile(pkg.Fset, f)
+		if err != nil {
+			panic(err)
+		}
-		astutil.Apply(
+		dstutil.Apply(
-			f,
+			df,
-			func(c *astutil.Cursor) bool {
+			func(c *dstutil.Cursor) bool {
				n := c.Node()
				switch x := n.(type) {
				default:
					return true
-				case *ast.FuncDecl:
+				case *dst.FuncDecl:
-				case *ast.GenDecl:
+				case *dst.GenDecl:
					// ...
				}
				return false
			},
			nil,
		)
	}
```

#### 対象タイプの探索

`isStringBasedType`は引数の型を`dst.Spec`に変えただけで、他は`astutil`版と全く変わりありません。

```go
func isStringBasedType(spec dst.Spec) (string, bool) {
	typ, ok := spec.(*dst.TypeSpec)
	if !ok {
		return "", false
	}
	id, ok := typ.Type.(*dst.Ident)
	if !ok {
		return "", false
	}
	return typ.Name.Name, id.Name == "string"
}
```

コメントのパーズ部分(`parseDirective`)はけっこうな変更になります。

```go
type EnumParam struct {
	Name     string
	Variants []string
}

func parseDirective(decorations dst.GenDeclDecorations) (EnumParam, bool) {
	for i := len(decorations.Start) - 1; i >= 0; i-- {
		line := decorations.Start[i]
		if len(strings.TrimSpace(line)) == 0 {
			// start of comments groups that is not associated to gen decl.
			break
		}
		c := stripMarker(line)
		c, isDirection := strings.CutPrefix(c, "enum:variants=")
		if !isDirection {
			continue
		}
		return EnumParam{Variants: strings.Split(c, ",")}, true
	}
	return EnumParam{}, false
}
```

まず、コメントは[dst.GenDeclDecorations](https://pkg.go.dev/github.com/dave/dst@v0.27.3#GenDeclDecorations)のような構造体に変更されています。
`dst`のdoc commentで述べられる通り、`Go`のコメントは思いのほか自由にかけるので、各部に対応したフィールドに単に`[]string`で格納する形になっています。

```go
/*Start*/
const /*Tok*/ ( /*Lparen*/
	a, b = 1, 2
	c    = 3
) /*End*/

/*Start*/
const /*Tok*/ d = 1 /*End*/
```

doc commentに当たるのは[dst.NodeDecs](https://pkg.go.dev/github.com/dave/dst@v0.27.3#NodeDecs)の`Start`ですのでこれだけを解析します。

例えば以下のようなnodeと`dst.Print`すると以下のようになります。

```go
// free floating comment

// doc comment
func (s Some[T, U]) Method1() {
	// ...nothing...
}

// comment slash slash


/*

comment slash star

*/

//   925  .  .  .  Decs: dst.FuncDeclDecorations {
//   926  .  .  .  .  NodeDecs: dst.NodeDecs {
//   927  .  .  .  .  .  Before: EmptyLine
//   928  .  .  .  .  .  Start: dst.Decorations (len = 3) {
//   929  .  .  .  .  .  .  0: "// free floating comment"
//   930  .  .  .  .  .  .  1: "\n"
//   931  .  .  .  .  .  .  2: "// doc comment"
//   932  .  .  .  .  .  }
//   933  .  .  .  .  .  End: dst.Decorations (len = 6) {
//   934  .  .  .  .  .  .  0: "\n"
//   935  .  .  .  .  .  .  1: "\n"
//   936  .  .  .  .  .  .  2: "// comment slash slash"
//   937  .  .  .  .  .  .  3: "\n"
//   938  .  .  .  .  .  .  4: "\n"
//   939  .  .  .  .  .  .  5: "/* \n\ncomment slash star\n\n*/"
//   940  .  .  .  .  .  }
//   941  .  .  .  .  .  After: None
//   942  .  .  .  .  }
//   943  .  .  .  }
//   944  .  .  }
//   945  .  }
```

free floating commentも`Start`にひとまとめに入れれらます。
つまり、`Start`は末尾から操作して`\n`などの空白のみの行までをクリップして探索すると`ast`版と同等ということになります。

上記の`parseDirective`実装では単に末尾から探索していますが、`ast`版は先頭から探索なので、挙動差が生じています。~~これは単なる手抜きですので~~実際にはこういった実装はしないほうがよいでしょう。
doc commentに当たるindexの範囲を探索し、その範囲を先頭から走査すると挙動差が生じません。

#### Replace or Insert

この部分も`astutil`版の型表記を`dst`に変えただけでほとんど変更はありません。
`CommentMap`やトークンオフセットは不要なので消えます。

```diff go
-func addOrReplaceEnum(c *astutil.Cursor, param EnumParam, cm ast.CommentMap) {
+func addOrReplaceEnum(c *dstutil.Cursor, param EnumParam) {
	found := false
-	astutil.Apply(
+	dstutil.Apply(
		c.Parent(),
-		func(c *astutil.Cursor) bool {
+		func(c *dstutil.Cursor) bool {
			node := c.Node()
			switch x := node.(type) {
			default:
				return true
-			case *ast.FuncDecl:
+			case *dst.FuncDecl:
-			case *ast.GenDecl:
+			case *dst.GenDecl:
				if x.Tok != token.CONST {
					break
				}
				if !isGeneratedFor(x.Doc, param.Name) {
					break
				}
				found = true
-				newDecl := astVariants(param, x.TokPos)
-				delete(cm, x)
-				cm[newDecl] = append(cm[newDecl], newDecl.Doc)
-				c.Replace(newDecl)
+				c.Replace(astVariants(param, x.Decs))
			}
			return false
		},
		nil,
	)
	if !found {
-		newDecl := astVariants(param, c.Node().(*ast.GenDecl).Specs[0].Pos()+30)
-		cm[newDecl] = append(cm[newDecl], newDecl.Doc)
-		c.InsertAfter(newDecl)
+		c.InsertAfter(astVariants(param, dst.GenDeclDecorations{}))
	}
}

-func isGeneratedFor(cg *ast.CommentGroup, fotTy string) bool {
+func isGeneratedFor(decorations dst.GenDeclDecorations, fotTy string) bool {
-	for _, comment := range cg.List {
+	for i := len(decorations.Start) - 1; i >= 0; i-- {
+		line := decorations.Start[i]
+		if len(strings.TrimSpace(line)) == 0 {
+			break
+		}
-		c := strings.TrimLeftFunc(stripMarker(comment.Text), unicode.IsSpace)
+		c := stripMarker(line)
		s, ok := strings.CutPrefix(c, "enum:generated_for=")
		if !ok {
			return false
		}
		if s == fotTy {
			return true
		}
	}
	return false
}
```

`const ()`ブロックを生成する部分は丸っと変わります。
前述のとおり、`dst.GenDeclDecorations`の`Start`がdoc commentに当たりますが、nodeのdoc commentの直前にfree floating commnetがあった場合は`Start`に一緒くたに入ってしまうため、末尾から探索して`\n`が見つかる場合にはそのindex以降にdoc commentを追記する形にします。こうすることで書き換えたいコメント以外を保つことができます。

他の部分は型名の`ast`の部分を`dst`に変える以外の変更はありません。

```go
func astVariants(param EnumParam, targetDecoration dst.GenDeclDecorations) *dst.GenDecl {
	if len(targetDecoration.Start) > 0 && targetDecoration.Start[len(targetDecoration.Start)-1] != "//enum:generated_for="+param.Name {
		var i int
		for i = len(targetDecoration.Start) - 1; i >= 0; i-- {
			if targetDecoration.Start[i] == "\n" {
				break
			}
		}
		if i < 0 {
			i = len(targetDecoration.Start) - 1
		}
		targetDecoration.Start = append(slices.Clone(targetDecoration.Start[:i]), "\n", "//enum:generated_for="+param.Name)
	} else {
		targetDecoration.Start = []string{"//enum:generated_for=" + param.Name}
	}
	return &dst.GenDecl{
		Decs:   targetDecoration,
		Tok:    token.CONST,
		Lparen: true,
		Specs:  mapParamToSpec(param),
		Rparen: true,
	}
}
```

#### 生成

[\*decorator.Restorer](https://pkg.go.dev/github.com/dave/dst/decorator#Restorer)で書き換えた`*dst.File`を`*ast.File`に逆変換し、[printer.Fprint](https://pkg.go.dev/go/printer@go1.22.6#Fprint)で結果をプリントできます。

```diff go
	for _, f := range pkg.Syntax {
		filename := filepath.Base(pkg.Fset.Position(f.FileStart).Filename)
+		df, err := decorator.DecorateFile(pkg.Fset, f)
+		if err != nil {
+			panic(err)
+		}
-		astutil.Apply(
+		dstutil.Apply(
-			f,
+			df,
-			func(c *astutil.Cursor) bool {
+			func(c *dstutil.Cursor) bool {
				// ...
			},
			nil,
		)

		out, err := os.Create(filepath.Join(generatedDir, filename))
		if err != nil {
			panic(err)
		}

+		restorer := decorator.NewRestorer()
+		af, err := restorer.RestoreFile(df)
+		if err != nil {
+			panic(err)
+		}
-		err = printer.Fprint(out, pkg.Fset, f)
+		err = printer.Fprint(out, pkg.Fset, af)
		if err != nil {
			panic(err)
		}
	}
```

#### 結果

https://github.com/ngicks/go-example-code-generation/blob/main/ast/rewrite/dstutil

`ast`版ではうまくいかなかったのに対し、

```go
package target

// free floating comment 1

func Foo() {
	// nothing
}

//enum:variants=foo,bar,baz,qux,quux,corge
type EnumWithComments string

// free floating comment 2

func Bar() {
	// nothing
}

//enum:variants=foo,bar,baz
type EnumWithComments2 string

// free floating comment 3

//enum:generated_for=EnumWithComments2
const (
	EnumWithComments2Foo EnumWithComments2 = "foo"
)

/* free floating comment 4


 */
```

以下を生成します。コメントの位置関係が完全に保たれていることがわかります。

```go
package target

// free floating comment 1

func Foo() {
	// nothing
}

//enum:variants=foo,bar,baz,qux,quux,corge
type EnumWithComments string

//enum:generated_for=EnumWithComments
const (
	EnumWithCommentsFoo	EnumWithComments	= "foo"
	EnumWithCommentsBar	EnumWithComments	= "bar"
	EnumWithCommentsBaz	EnumWithComments	= "baz"
	EnumWithCommentsQux	EnumWithComments	= "qux"
	EnumWithCommentsQuux	EnumWithComments	= "quux"
	EnumWithCommentsCorge	EnumWithComments	= "corge"
)

// free floating comment 2

func Bar() {
	// nothing
}

//enum:variants=foo,bar,baz
type EnumWithComments2 string

// free floating comment 3

//enum:generated_for=EnumWithComments2
const (
	EnumWithComments2Foo	EnumWithComments2	= "foo"
	EnumWithComments2Bar	EnumWithComments2	= "bar"
	EnumWithComments2Baz	EnumWithComments2	= "baz"
)

/* free floating comment 4


 */
```

### dstを使用し続けるリスク

`ast`と違い、[github.com/dave/dst]はサードパーティ、かつ個人メンテのライブラリですから`Go`の進化についていけないリスクは常に抱えています。

例えばGo1.18で[IndexListExpr](https://pkg.go.dev/go/ast@go1.23rc2#IndexListExpr)が追加されました。`1.18`と言えばgenericsが追加されたアップデートです。genericsのために構文が拡張されたので(instantiation時に複数の型がある場合の表記, e.g. `[int, string, *bytes.Buffer]`が今までの構文上存在しなかった)、このexprが追加されたわけです。

現状の`dst`は上記には対応済みであるので現状のあらゆるコードにうまく機能するはずです。今後構文の追加があれば、同様にexprが追加されてそれについていけなくなるという可能性があるわけです。

Go1.23ではexprの追加はありません。逆に言って`1.0.0`から追加されたast nodeは上記のみです。
`ast`は非常にstableであり、おそらく`Go` teamもなるだけtokenもexprも追加したくはないでしょうから今後の追加の可能性も少ないでしょう。

ですのでおそらく今後数年はまずもって使い続けられると筆者は見積もっています。
exprが追加されてなおかつモジュールオーナーが非活発的な場合、筆者も頑張って貢献して直します。

## おわりに

前段の記事で

- [Goのcode generation: text/template](https://zenn.dev/ngicks/articles/go-code-generation-in-ways-text-template)で
  - Rationale: なぜGoでcode generationが必要なのか
  - code generatorを実装する際の注意点など
  - `io.Writer`に書き出すシンプルな方法
  - `text/template`を使う方法
    - `text/template`のcode generationにかかわりそうな機能性。
    - 実際に`text/template`を使ったcode generatorのexample。
- [Goのcode generation: jennifer](https://zenn.dev/ngicks/articles/go-code-generation-in-ways-jennifer)で
  - [github.com/dave/jennifer]の各機能
  - `text/template`で実装したcode generatorのexampleを`jennifer`で再実装

についてそれぞれ述べました。

この記事では、

- astのパーズ方法
  - `go/parser`を用いる方法
  - [golang.org/x/tools/go/packages]を用いる方法
- 軽いastの解析方法やデバッグ方法
  - ast構造のprint: ast.Print
  - directive commentの解析
  - astのtraverse方法
- `astutil.Apply`でgo source codeのrewriteを実装します
- `astutil.Apply`ではコメントオフセットの狂いによってコメントの順序がおかしくなる問題について述べ
- [github.com/dave/dst]によってこの問題を起さずにast rewriteができることを述べます。
  - `dst`の紹介
  - astと`dst`の相互変換
  - `dst`でのコメントの取り扱い方法について
  - `dstutil.Apply`を使ったrewrite

を述べました。

`Go`のsource codeを解析してastをえて、それを解析してrewriteを行うためにかかわりそうな要素について紹介しました。
もう少し凝ったexampleを乗せてもいいかなと思いましたが、それは別の記事に分離しようかと思います(書くかはわかりませんが)。

`text/template`や[github.com/dave/jennifer]を用いてコードを生成したほうがはるかに簡単なので、この方法を利用することは少ないと思います。
linterのcode actionのようなものを実装したいときや、code generatorの生成結果をさらに変更するなどのケースで便利かなと思います。

実装する機会は少ないかもしれない・・・少なくとも筆者的に仕事でやるには正当化しずらい手間です・・ですが、やれると体験がよいので覚えておくとよいかもしれません。

[Go]: https://go.dev/
[Rust]: https://www.rust-lang.org
[Visual Studio Code]: https://code.visualstudio.com/
[vscode]: https://code.visualstudio.com/
[github.com/dave/jennifer]: https://github.com/dave/jennifer
[github.com/dave/dst]: https://github.com/dave/dst
[text/template]: https://pkg.go.dev/text/template@go1.22.6
[go/ast]: https://pkg.go.dev/go/ast@go1.22.6
[golang.org/x/tools/go/packages]: https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages
