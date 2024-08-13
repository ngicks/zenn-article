---
title: "Goのcode generationまとめ: text/template,jennifer,ast(dst)-rewrite"
emoji: "🧰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## Goのcode generationについてまとめる

`Go`のcode generationについてまとめようと思います。

- `io.Writer`に書き出すシンプルな方法
- `text/template`を使う方法
- [github.com/dave/jennifer]を使う方法
- `ast`をrewriteする方法
  - `ast`をrewriteするには問題があるので、その問題について述べ
  - 実際には[github.com/dave/dst]を使う方法について述べます。

についてそれぞれ述べます。

それぞれに対してそれなりに詳しく書こうと思っています。

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

## Rationale: Goにおけるcode generation

[Go]は、`C`や[Rust](https://doc.rust-lang.org/book/ch19-06-macros.html)にあるようなマクロを言語機能として持っていないため、たびたびcode generationを行いますし、それを行うことは前提のようになっています。

### go:generate

[The Go BlogのGenerating code](https://go.dev/blog/generate)にも書かれている通り、`Go 1.4`から`Go`には[go generate](https://pkg.go.dev/cmd/go#hdr-Generate_Go_files_by_processing_source)というサブコマンドが追加されました。
これはgo source codeに書かれた`//go:generate`マジックコメントのあとにスペースを挟んで書かれた任意をコマンドを、そのソースファイルの位置をcwdに指定して実行するというものです。
これは任意のコマンドを実行できますが、基本的にcode generatorを実行してコードを生成することを支援する仕組みです。

`Go`自身も`//go:generate`を活用しており、

https://github.com/search?q=repo%3Agolang%2Fgo%20%2F%2Fgo%3Agenerate&type=code

以上のように検索してみればたくさんヒットします。

### goのgeneric function

`Go`は[reflect](https://pkg.go.dev/reflect@go1.22.5)によって型情報を`any`な値から取り出すことができ、これを元に動的な挙動を行うことができます。
また、[Go 1.18で追加されたGenerics](https://tip.golang.org/doc/go1.18#generics)を用いることで、ある程度複数の型に対して処理を共通化できます

例えば、以下のようなサンプルを定義します。

サンプルでは、あるstructに対して、フィールド名と定義順が一致するが、型が`Patcher[T]`で置き換えられたstructを用意することで、部分的なフィールドの変更をできる挙動を示します。

[playground](https://go.dev/play/p/_a85DOAZV7H)

```go
type Sample struct {
	Foo string
	Bar int
	Baz bool
}

type SamplePatch struct {
	Foo Patcher[string]
	Bar Patcher[int]
	Baz Patcher[bool]
}

type Patcher[T any] struct {
	Present bool
	V       T
}

func (p Patcher[T]) IsPresent() bool {
	return p.Present
}

func patch(target, patch any) {
	tgtRv := reflect.ValueOf(target).Elem()
	patchRv := reflect.ValueOf(patch)

	for i := 0; i < tgtRv.NumField(); i++ {
		ft := tgtRv.Field(i)
		fp := patchRv.Field(i)

		if !fp.Interface().(interface{ IsPresent() bool }).IsPresent() {
			continue
		}

		ft.Set(fp.Field(1))
	}
}

func main() {
	s := Sample{}
	fmt.Printf("0: %#v\n", s)
	// 0: main.Sample{Foo:"", Bar:0, Baz:false}

	patch(&s, SamplePatch{Foo: Patcher[string]{true, "foo"}})
	fmt.Printf("1: %#v\n", s)
	// 1: main.Sample{Foo:"foo", Bar:0, Baz:false}

	patch(
		&s,
		SamplePatch{
			Foo: Patcher[string]{true, "bar"},
			Bar: Patcher[int]{true, 123},
		},
	)
	fmt.Printf("2: %#v\n", s)
	// 2: main.Sample{Foo:"bar", Bar:123, Baz:false}

	patch(&s, SamplePatch{Baz: Patcher[bool]{true, true}})
	fmt.Printf("3: %#v\n", s)
	// 3: main.Sample{Foo:"bar", Bar:123, Baz:true}
}
```

- `reflect`を使用することでstructなどのデータ構造に対して動的な処理を実装できます
- `generics`を利用することで任意の型に対して共通した処理を実装できます
  - (サンプルでは特に示していないが)「特定の`interface`を実装する」という型制約をかけることもできます。

ただし、

- `reflect`を使って動的に処理を行う場合、静的に型を当てた関数で包んでtype assertionを行わない限り型安全性を失います。

つまりこういうことです。

```go
func patchSample(s *Sample, patcher SamplePatch) {
	patch(s, patcher)
}
```

`patch`の引数はどちらも`any`でしたが、実際には第一引数は`foobar`へのポインターで、第二引数は`foobarPatch`のnon-pointer型でないと想定通りの動作をしませんから、
こうやって具体的な型の書かれた関数を定義したほうが利用しやすいことになります。

このケースでは返り値がないのでピンときにくいかもしれませんが、`reflect.Value`から値を取り出そうと思うと`Interface()`メソッドで`any`型の値を取り出すしかありませんので、
`type assertion`を関数内で行うことで返り値の型を具体的なものにするのが普通だと思います。

- `generics`では[#49085](https://github.com/golang/go/issues/49085)がないためにmethodにtype paramを与えることができません。

つまり以下のようなことはできないということです。

```go
// compilation error
func (p Patch[T]) Convert[U any](converter func(t T) U) U {
	// ...
}
```

また、双方ともに型情報に含まれないような情報を用いた動的な処理を行えません。

例えば、[golang.org/x/tools/cmd/stringer](https://pkg.go.dev/golang.org/x/tools/cmd/stringer)は以下のような、iotaを用いたenum風なconst定義に対して

https://github.com/golang/go/blob/go1.22.5/src/html/template/context.go#L80-L161

以下のように`String`メソッドを生成します。

https://github.com/golang/go/blob/go1.22.5/src/html/template/state_string.go

これらはソースコードの解析その他を行わない限り不可能なことですので、依然code generatorが必要なことになります。

## 4つの(おそらく)代表的な方法

大雑把に言って4つの方法が代表的なのではないかと思います

- `io.Writer`にテキストを書くだけ
  - プログラムによってgo source fileとなるテキストを書きだすだけの方法です
- [text/template]を用いる方法
  - stdに組み込まれたtemplateパッケージを用いる方法です
  - `io.Writer`を用いる方法に比べると、名前付きでパラメータを定義できるのでより複雑なケースに対応できます。
- [github.com/dave/jennifer]を用いる方法
  - code generatorを記述するためのライブラリを用いる方法です
  - goのトークンや構文に対応した各種関数をメソッドチェーンで記述していく方式です。
    - `text/template`と違い、`Go`の関数呼び出しの羅列なのでsyntax highlightがしっかりかかる分読みやすいです。
  - go source code生成に特化しているため、この用途では`text/template`より使いやすいです。
  - ただし、`text/template`に比べてテンプレートをユーザーから入力として受け取るのが困難になります。
    - テンプレートをシリアライズする方法に関しては別段定義されていない。
- `ast`(dst)-rewriteを行う方法
  - astをもとにコードを生成する方法です。
  - `go/ast`, `go/parser`, `go/printer`などのstd libraryを用います。
  - `ast`で1からコードをくみ上げることも当然可能ですが、前述のいずれかの方法をとったほうが簡単なので、rewriteする方法についてのみ述べます

上記を整理しなおすを以下のような関係図になります

![code-generation-data-flow](/images/go-code-generation-in-ways-data-flow.drawio.png)

おおむねコード生成のためのメタデータ取得部と、コード生成部と、コードの書き出し部分、最後のポストプロセスとしてフォーマットやタイプチェックぐらいに分かれると思います。タイプチェックは図では省略されています。

メタデータ取得部分は`ast`を用いる場合はgo source codeを入力とし、`go/parser`を用いて`ast`の解析します。`ast`は、さらに`type checker`で解析すること型情報を得ることもできます(e.g. [skeleton](https://github.com/golang/example/blob/39e772fc26705bb170db248e5372a81ed5ffd67f/gotypes/skeleton/main.go))。
他の方法では`JSON`, `YAML`のようなフォーマットで書かれたデータ構造(ここにcli引数も含む)を`json.Unmarshal`や`yaml.Unmarshal`してデータ構造にbindしたり、[text/template]向けのテキストを読み込んだりします。

コード生成部では、えられたメタデータを`io.Writer`に書き出したり、[text/template]を用いるなど前述の方法の一部または全部を組み合わせて行います。`ast`をrewriteする方法では`go/printer`の機能を利用することで、あるast nodeのみを出力するようなことができるので、これも他の方法と組み合わせて1つのテキストファイルを形成することができます。

コード生成部によってテキストを出力します。このテキストは有効なgo source codeの文法を満たしてさえいれば(i.e. `package pkgname`から始まるgo code)この時点でファイルとして書きだされている必要はありません。

go source codeのテキスト、またはテキストのストリームは[gofmt](https://pkg.go.dev/cmd/gofmt), [github.com/mvdan/gofumpt](https://github.com/mvdan/gofumpt), [golang.org/x/tools/cmd/goimports](https://pkg.go.dev/golang.org/x/tools/cmd/goimports)などのフォーマッターを用いることでフォーマットをできます。`goimports`は`gofmt`と同じルールでフォーマットを行ったうえで、import declが正しくなかった場合修正をこころみます。`ast`を書き換える形でコードを生成する場合、使わなくなったimportが出てくることがあります。`Go`では、使用していないimport declが存在しているとコンパイルエラーになりますが、`goimports`でそれらを消してもらうと書き換え部の実装が楽になります。

最後に、ファイルとして書きだされたgo source codeは[gopls](https://github.com/golang/tools/tree/master/gopls)の機能を用いてフォーマットをかけることができます。ユーザーの`gopls`設定を元にフォーマットを行いたい場合は便利かもしれませんが基本的にはしません。理由はよくわかっていませんが、`goimports`などを直接呼び出す方法に比べてずいぶん動作速度が遅い(0.1秒オーダーに比べて数秒オーダー)ためです。

## codeを生成する際の注意点

### ファイル先頭に// Code generated ... DO NOT EDIT.をつける

[go generateのドキュメント](https://pkg.go.dev/cmd/go#hdr-Generate_Go_files_by_processing_source)にもある通り、
`^// Code generated .* DO NOT EDIT\.$`という正規表現にマッチする行が**ソースコード先頭**に含まれる場合、`go tool`はこれをcode generatorによって生成されたファイルであるとみなします。
`Code generated`の後の部分にcode generatorのpackage pathを書いておくとよいのではないかと思います。

Go1.21より[ast.IsGenerated](https://pkg.go.dev/go/ast@go1.22.5#IsGenerated)という関数がexportされるようになったので、もし`*ast.File`を経由して生成されたファイルかの確認が行いたい場合はこれを用いるとよいでしょう。

### for-range-mapはしないように気を付ける

> https://go.dev/ref/spec#For_range
>
> The iteration order over maps is not specified and is not guaranteed to be the same from one iteration to the next.

言語仕様によりfor-range-mapの順序は未定義であるので、code generatorが実行のたびに異なった順序でコードを出力しないように気を付けたほうがよいでしょう。
さもなければ、不要なdiffを生じさせてしまいます。
筆者が利用するサードパーティのcode generatorの中にも、生成する度に順序の入れ替わるものがありますが、生成対象が多くなるにつれて出てくるdiffの量が多くなってセルフレビューが大変になっています。
基本的にそうならないように作ったほうが利用者とっては便利です。

代わりに

```go
// https://go.dev/play/p/s5K782-FhKo
var m map[K]V
keys := make([]K, 0, len(m))
for k := range m {
	keys = append(keys, k)
}
slices.Sort(keys)
for _, k := range keys {
	_ = m[k]
	// ...
}
```

とすることで、stableな順序でmapをiterateできます。
Go1.23以降ならもっと簡単に

```go
// https://go.dev/play/p/2yRGLquakg8?v=gotip
var m map[K]V
keys := slices.Collect(maps.Keys(m))
slices.Sort(keys)
for _, k := range keys {
	_ = m[k]
	// ...
}
```

とできます。

### go:generate go run -mod=mod

code generatorがruntime(生成されたコードがimportして利用するヘルパー関数を定義したパッケージ)に依存し、code generator自身と同じGo moduleで管理される場合、

```
# 架空のURLを取り扱うのでexample.comのサブドメインとして書いていますが単なる例示で文字列そのものには意味はありません。
//go:generate go run -mod=mod fully-qualified.example.com/package/path/cmd/path/to/main/pkg@version
```

という風にcode generatorを動作させるようにあなたの各READMEの中で指示したほうが親切でしょう

> https://go.dev/ref/mod#build-commands
>
> -mod=mod tells the go command to ignore the vendor directory and to [automatically update](https://go.dev/ref/mod#go-mod-file-updates) go.mod, for example, when an imported package is not provided by any known module.

とある通り、code generatorのバージョンが、生成物の配置先となるgo moduleの`go.mod`に追加されるなり更新されるなりするらしいです。

## io.Writerに書くだけ

という風にタイトルをつけていますが特にこれと言って述べるべきことはこれにはありません。

例えば、`Go`の`runtime`では以下のような単にテキストを書くだけのcode generatorを見つけることができます。

https://github.com/golang/go/blob/go1.22.5/src/runtime/wincallback.go

生成対象は`.s`の[Go assembly](https://go.dev/doc/asm)ファイルですが、まあ言いたいことはかわらないのでいいとしましょう。

このコードによって以下のがファイルが生成されます。

https://github.com/golang/go/blob/go1.22.5/src/runtime/zcallback_windows_arm.s
https://github.com/golang/go/blob/go1.22.5/src/runtime/zcallback_windows_arm64.s
https://github.com/golang/go/blob/go1.22.5/src/runtime/zcallback_windows.s

これらのファイルは以下のような、ほぼ同じパターンを2000(=`maxCallback`)回繰り返すだけの単純なものです。

```
	MOVD	$i, R12
	B	runtime·callbackasm1(SB)
```

このように、単純なコード断片を何度も書きだすだけのようなケースでは、単なる`io.Writer`への書き出しで十分機能します。

## text/template

https://pkg.go.dev/text/template@go1.22.5

stdライブラリに組み込まれたテンプレート機能です。

`html/template`も存在しますが、こちらは`html`を出力するための各種サニタイズを実装した`text/template`のラッパーみたいなものですので、テキストの出力に関しては`text/template`を使用します。
エディターの自動補完に任せると`html/template`のほうが読み込まれていて気付かず出力結果のテキストが思った通りにならずに首をかしげることが何度かありました。その辺も注意してみておいたほうが良いです(さすがに今はもうそんなミスはしませんが)。

詳細な使い方の説明は上記の`text/template`のdoc comment、ないしは実装そのものに当たってほしいと思いますが、筆者は初めて読んだときあまりにピンときませんでした。
なのでcode generatorとして使うときにかかわりそうなところはここで説明しておきます。

### 利点と欠点

利点:

- stdのみで終始できる
- 十分柔軟で便利
- 何ならtemplateそのものをユーザーに入力させて、code generatorの挙動をカスタマイズさせるようなことができる
  - 当然テキストなので、cliやネットワーク経由でも容易に受け取ることができます。

欠点:

- code generationのためのものではない
  - [github.com/dave/jennifer]に比べると大分書きにくい
  - `for`がネストしだすと劇的に視認性が落ちる
  - 空白の取り扱いが難しい。
    - 筆者は無駄な改行を甘んじて受け入れている
    - 生成後のコードを`goimports`によってフォーマットをかけることでいくらか改善する
  - importの取り扱いが大変。
    - ユーザーにtemplateを入力させる系を想定すると、ここで新しく追加されたimportをどう取り扱うか、自分で決める必要があります。

### エディターのサポート(syntax highlight, Go to definition, etc...)

`gopls`の設定をしたうえでtemplateを`gotmpl`か`tmpl`でテキストとして保存すると`gopls`によるsyntax highlightなどのサポートを得られます(experimental)。

vscodeの場合、`settings.json`に以下を追加します。

```json
{
  // ...other settings...
  "gopls": {
    // ...other settings...
    "ui.semanticTokens": true,
    "build.templateExtensions": ["gotmpl", "tmpl"]
    // ...other settings...
  }
  // ...other settings...
}
```

他のエディターの場合、`gopls`の設定を似たような感じで設定します。

syntax highlight以外の機能は現状でも機能しているように見えるので、そこが不要なら設定は不要です。
`"ui.semanticTokens"`を有効にするとGo source codeのエディター上のトークンの色がかなり変わりますのでびっくりするかもしれません。

現状`gopls`の`semanticTokens`はexperimentalですがもうすぐenable by defaultになるかもしれません([#45313](https://github.com/golang/go/issues/45313#issuecomment-2161267130))。

### 基本的な使用法

例示されるコードは以下でもホストされます。

https://github.com/ngicks/go-example-code-generation/blob/main/template/basic

#### 構文

パラメータ、関数その他の呼び出しはdelimiter(`{{`と`}}`)で囲まれたブロックの中で行います。

```tmpl
An example template.
Hello {{.Gopher}}.
Yay Yay.
```

というtemplateでは`{{.Gopher}}`の部分が入力のパラメータによって動的に変更されることになります。
上記は、パラメータとして渡された任意のGo structの、`Gopher`というexported fieldの値でここを置き換えるという意味になります。

このdelimiter(`{{`,`}}`)は[(\*Template).Delims](https://pkg.go.dev/text/template@go1.22.5#Template.Delims)で任意の文字列に変更できます。
基本的には変えないほうが良いです: `gopls`のドキュメントにも、delimiterは変えられるが変えたら構文解析が機能しないようなことが書いてあります。
筆者はこの記事を書くまで変更できることすら知りませんでした。

#### 初期化、解析

`template.New()`で新しい`*Template`をallocateし、`Parse`によってtemplateテキストを解析して`*Template`オブジェクトを得ます。

```go
var example = template.Must(template.New("").Parse(
	`An example template.
Hello {{.Gopher}}.
Yay Yay.
`,
))
```

`template.New`には`name`を渡せますが、今回のように単一のtemplateしか解析しない場合は特に名づける必要はありません。
`template.Must`は`(*Template, error)`を引数にとって、第二引数のエラーがnon-nilだった場合panicするヘルパー関数です。

#### 実行(Execute)

上記で`Parse`から返された`*Template`オブジェクトのメソッドを呼び出すことでこのtemplateを実行できます。

```go
type sample struct {
	Gopher string
}

err := example.Execute(os.Stdout, sample{Gopher: "me"})
```

で、渡された`io.Writer`にtemplate実行結果を書き出します。
`os.Stdout`を渡しているのでstdoutに書き出されます。

```
An example template.
Hello me.
Yay Yay.
```

上記がstdoutに出力されます。

### パラメータへのアクセス

`Execute`の第二引数にはパラメータを詰め込んだデータ構造を渡します。
`{{.}}`の`.`はcontextualな値で、トップレベルでは`Execute`に渡したデータそのものをさしています。

渡すパラメータは任意の`Go struct`か`map[K]V`であれば、dot selectorでフィールドの値か、keyに収められている値がそれぞれ取り出されることが書かれています。
structを指定する場合は`reflect`パッケージを使って値にアクセスしますので、**reflectでアクセスできるフィールドを指定する**必要があります。

つまり、embedされたunexported structのexported fieldにはアクセスできます。

```go
type sample struct {
	Gopher string
}

type embedded struct {
	sample
}

_ = example.Execute(os.Stdout, embedded{sample{Gopher: "embedded"}})
/*
An example template.
Hello embedded.
Yay Yay.
*/
```

メソッドでもよいとドキュメントされています。

```go
type sampleMethod1 struct {
}

func (s sampleMethod1) Gopher() string {
	return "method"
}

_ = example.Execute(os.Stdout, sampleMethod1{})
/*
An example template.
Hello method.
Yay Yay.
*/
```

関数は全般的に第二返り値でエラーを返してもよいということがドキュメントされています。
関数から返されるエラーがnon-nilであるとその時点でtemplateの実行が止まって、そのエラーが`Execute`から返ってきます

```go
type sampleMethod2 struct {
	err error
}

func (s sampleMethod2) Gopher() (string, error) {
	return "method2", s.err
}

_ = example.Execute(os.Stdout, sampleMethod2{})
/*
An example template.
Hello method2.
Yay Yay.
*/
fmt.Println("---")
err := example.Execute(os.Stdout, sampleMethod2{err: errors.New("sample")})
fmt.Println("---")
fmt.Printf("error: %v\n", err)
/*
---
An example template.
Hello ---
error: template: :2:8: executing "" at <.Gopher>: error calling Gopher: sample
*/
```

同様に、`map[K]V`でもいいです。`map[K]V`の場合は先頭が小文字な(_unexported_)フィールドにもアクセスできます。
template textが小文字なフィールドにアクセスしていたら`map[K]V`を使うしかないので基本的にはそうなってることはないと思います。

```go
_ = example.Execute(os.Stdout, map[string]string{"Gopher": "from map[string]string"})
/*
An example template.
Hello from map[string]string.
Yay Yay.
*/

var accessingUnexported = template.Must(template.New("").Parse(
	`accessing unexported field: {{.unexportedField}}
`,
))

_ = accessingUnexported.Execute(os.Stdout, map[string]string{"unexportedField": "unexported field"})
/*
accessing unexported field: unexported field
*/

type unexported struct {
	unexportedField string
}

fmt.Println("---")
err := accessingUnexported.Execute(os.Stdout, unexported{})
fmt.Println("---")
fmt.Printf("error: %v\n", err)
/*
---
accessing unexported field: ---
error: template: :1:30: executing "" at <.unexportedField>: unexportedField is an unexported field of struct type main.unexported
*/
// reflectはこのフィールドにアクセスできないのでエラーが返される。
```

さらに、このdot selectorはchainさせることもできます。

```go
chained = template.Must(template.New("").Parse(`**chained**{{.Chain.Gopher}}
`))

type chainedData struct {
	v   any
	err error
}

func (c chainedData) Chain() (any, error) {
	return c.v, c.err
}


_ = chained.Execute(os.Stdout, chainedData{v: sampleMethod2{}})
/*
**chained**method2
*/

_ = chained.Execute(os.Stdout, chainedData{v: map[string]string{"Gopher": "map"}})
/*
**chained**map
*/
```

### 制御構文: range, if

例示されるコードは以下でもホストされます。

https://github.com/ngicks/go-example-code-generation/blob/main/template/control-flow

#### range

`range`で`Go`の`for-range`のようにデータをiterateできます。

> {{range pipeline}} T1 {{end}}
> The value of the pipeline must be an array, slice, map, or channel.
> If the value of the pipeline has length zero, nothing is output;
> otherwise, dot is set to the successive elements of the array,
> slice, or map and T1 is executed. If the value is a map and the
> keys are of basic type with a defined order, the elements will be
> visited in sorted key order.

とる通り、`range`が引数に取れるのは`array`, `slice`, `map`, `channel`のいずれかであり、`Go 1.23`リリース時点ではrange-over-funcはできないようです([#66107](https://github.com/golang/go/issues/66107)が未実装であるので)。
`map[K]V`に関しては`K`の型がbasicなordered typeである場合はソートしてからiterateを行うと書かれています。range-over-mapみたいに順序が未定義でないことに逆に注意が必要ですかね？

> {{range $index, $element := pipeline}}

という構文で、`Go`の`range`構文のように`index`と`element`を変数にセットします。

`range`は`{{end}}`までスコープを作り、このスコープ内では`{{.}}`は、iterateされているデータの各項目をさします。`[]T`なら`T`, `map[K]V`なら`V`になります。
上記の`$index`,`$element`ももちろんこのスコープ内でのみ有効です。

このスコープ内では`{{break}}`と`{{continue}}`を使用して`Go`の`break`と`continue`と同等の制御をできます。どちらにも`label`を指定できるという記載はありません。

> When execution begins, $ is set to the data argument passed to Execute, that is, to the starting value of dot.

とある通り、このスコープ内では`$`が`Execute`関数に渡されたデータになります。

#### if

`if`で、`Go`の`if`のように条件による分岐ができます

> {{if pipeline}} T1 {{end}}
> If the value of the pipeline is empty, no output is generated;
> otherwise, T1 is executed. The empty values are false, 0, any
> nil pointer or interface value, and any array, slice, map, or
> string of length zero.
> Dot is unaffected.

とある通り、emptyの条件は`false`, `0`, `nil`, `len(a)==0`であるとのことなので、falsyな値の判定の関数を作りこむ必要がない場面も多いでしょう。

#### example

以下で`range`と`if`を使ったexampleを示します。

```go
var (
	example = template.Must(template.New("").Parse(
		`Hi {{.Gopher}}.
{{range $idx, $el := .Iter}}    {{if not .}}Hey {{$.Gopher}} this is empty
	{{- if not $.Continue}}{{break}}{{end -}}
{{else}}Iterating at {{$idx}}: {{.Field}} {{end}}
{{end}}
`,
	))
)

func main() {
	decoratingExecute := func(data any) {
		fmt.Println("---")
		err := example.Execute(os.Stdout, data)
		fmt.Println("---")
		fmt.Printf("error: %v\n", err)
		fmt.Println()
	}

	decoratingExecute(map[string]any{
		"Gopher": "you",
		"Iter":   []map[string]string{{"Field": "foo"}, {"Field": "bar"}, {}, {"Field": "baz"}},
	})

	decoratingExecute(map[string]any{
		"Gopher":   "you",
		"Continue": "ok",
		"Iter":     []map[string]string{{"Field": "foo"}, {"Field": "bar"}, {}, {"Field": "baz"}},
	})

	decoratingExecute(map[string]any{
		"Gopher": "you",
		"Iter":   map[string]map[string]string{"0": {"Field": "foo"}, "1": {"Field": "bar"}, "2": {"Field": "baz"}},
	})
}
/*
---
Hi you.
    Iterating at 0: foo
    Iterating at 1: bar
    Hey you this is empty
---
error: <nil>

---
Hi you.
    Iterating at 0: foo
    Iterating at 1: bar
    Hey you this is empty
    Iterating at 3: baz

---
error: <nil>

---
Hi you.
    Iterating at 0: foo
    Iterating at 1: bar
    Iterating at 2: baz

---
error: <nil>

*/
```

何気なく使っていますが、`{{- pipeline}}`, `{{pipeline -}}`で前の/後ろの空白を削除する機能があります。

> For this trimming, the definition of white space characters is the same as in Go: space, horizontal tab, carriage return, and newline.

この「空白」の条件はGo source codeのそれと一致します。割とこの挙動が難しいので筆者は使いどころを選んでいます。

### 関数の追加

例示されるコードは以下でもホストされます。

https://github.com/ngicks/go-example-code-generation/blob/main/template/funcmap

template actionの中で実行できる関数は以下で定義される通りいろいろありますが

https://pkg.go.dev/text/template@go1.22.5#hdr-Functions

それ以外にも、[(\*Template).Funcs](https://pkg.go.dev/text/template@go1.22.5#Template.Funcs)で任意に追加できます。

関数はtemplate内で参照される前に追加されている必要がありますが、あとから上書きすることもできます。

以下でいろいろ試してみます。

```go
var (
	example = template.Must(
		template.
			New("").
			Funcs(template.FuncMap{"customFunc": func() string { return "" }}).
			Parse(
				`{{customFunc .}}
`,
			),
	)
)

func main() {
	decoratingExecute := func(funcs template.FuncMap, data any) {
		fmt.Println("---")
		err := example.Funcs(funcs).Execute(os.Stdout, data)
		fmt.Println("---")
		fmt.Printf("error: %v\n", err)
		fmt.Println()
	}

	decoratingExecute(nil, "foo")
	/*
		---
		---
		error: template: :1:2: executing "" at <customFunc>: wrong number of args for customFunc: want 0 got 1
	*/
	decoratingExecute(
		template.FuncMap{"customFunc": func(v any) string { return fmt.Sprintf("%s", v) }},
		"foo",
	)
	/*
		---
		foo
		---
		error: <nil>
	*/
	decoratingExecute(
		template.FuncMap{"customFunc": func(v ...any) string {
			fmt.Printf("customFunc: %#v\n", v)
			return "ah"
		}},
		"bar",
	)
	/*
		---
		customFunc: []interface {}{"bar"}
		ah
		---
		error: <nil>
	*/
	decoratingExecute(
		template.FuncMap{"customFunc": func(v string) string {
			fmt.Printf("customFunc: %#v\n", v)
			return "ah"
		}},
		"baz",
	)
	/*
		---
		customFunc: "baz"
		ah
		---
		error: <nil>
	*/
	decoratingExecute(
		template.FuncMap{"customFunc": func(v int) string {
			return "ah"
		}},
		"qux",
	)
	/*
		---
		---
		error: template: :1:13: executing "" at <.>: wrong type for value; expected int; got string
	*/
	type sample struct {
		Foo string
		Bar int
	}
	decoratingExecute(
		template.FuncMap{"customFunc": func(v any) int {
			fmt.Printf("customFunc: %#v\n", v)
			return v.(sample).Bar
		}},
		sample{Foo: "foo", Bar: 123},
	)
	/*
	   ---
	   customFunc: main.sample{Foo:"foo", Bar:123}
	   123
	   ---
	   error: <nil>
	*/
		decoratingExecute(
		template.FuncMap{"customFunc": func(v sample) string {
			return v.Foo
		}},
		sample{Foo: "foo", Bar: 123},
	)
	/*
		---
		foo
		---
		error: <nil>
	*/
}
```

関数の引数の型は何でもいいですが、入力パラメータと一致しなければエラーになるようです・・・と言ってる間に気になったのでソースを見ました。[(reflect.Type).AssignableToによる判定です。](https://github.com/golang/go/blob/go1.22.5/src/text/template/exec.go#L852-L862)

### sub-template

例示されるコードは以下でもホストされます。

https://github.com/ngicks/go-example-code-generation/blob/main/template/subtemplate

> {{template "name"}}
> The template with the specified name is executed with nil data.
>
> {{template "name" pipeline}}
> The template with the specified name is executed with dot set
> to the value of the pipeline.

とる通り、`template`で、別の名付けられたtemplateを、pipelineの評価結果を引数に実行できます。

別の名付けられたtemplateは`(*Template).New(name)`で作成し、返り値の`(*Template).Parse`を呼び出すか、
`{{define "name"}}_template definition_{{end}}`で定義することで作成することができます。

`{{block}}`は`{{define}}`して`{{template}}`するショートハンドです。

筆者もこの記事を書くまで全くわかっていなかったのですが、`*Template`は`*common`という構造体で解析されたtemplateを保持し、この`*common`は`(*Template).New`で作成されたすべての`(*Template)`に共有されています。
この`*common`を上書きしあうことでtemplate definitionを共有しています。
なので同名のtemplateを複数定義している場合などに`Parse`する順序に意味があることになりますね。

https://github.com/golang/go/blob/go1.22.5/src/text/template/template.go#L13-L35

以下で複数のtemplateを使用するサンプルを示します。

`sub1.Parse`などを呼び出すことで、ユーザーから渡されたtemplate definitionによって元のtemplate構造を上書きしてカスタマイズが行えることを示します。

```go
var (
	root = template.Must(template.New("root").Parse(
		`sub1: {{template "sub1" .Sub1}}
sub2: {{template "sub2" .Sub2}}
sub3: {{template "sub3" .Sub3}}
{{block "additional" .}}{{end}}
`))
	sub1 = template.Must(root.New("sub1").Parse(`{{.Yay}}`))
	_    = template.Must(root.New("sub2").Parse(`{{.Yay}}`))
	sub3 = template.Must(root.New("sub3").Parse(`{{.Yay}}`))
)

type param struct {
	Sub1, Sub2, Sub3 sub
}
type sub struct {
	Yay string
	Nay string
}

func main() {
	decoratingExecute := func(data any) {
		fmt.Println("---")
		err := root.Execute(os.Stdout, data)
		fmt.Println("---")
		fmt.Printf("error: %v\n", err)
		fmt.Println()
	}

	data := param{
		Sub1: sub{
			Yay: "yay1",
			Nay: "nay1",
		},
		Sub2: sub{
			Yay: "yay2",
			Nay: "nay2",
		},
		Sub3: sub{
			Yay: "yay3",
			Nay: "nay3",
		},
	}

	decoratingExecute(data)
	/*
		---
		sub1: yay1
		sub2: yay2
		sub3: yay3

		---
		error: <nil>
	*/
	_, _ = sub1.Parse(`{{.Nay}}`)
	decoratingExecute(data)
	/*
		---
		sub1: nay1
		sub2: yay2
		sub3: yay3

		---
		error: <nil>
	*/

	_, _ = sub3.New("additional").Parse(`{{.Sub1.Yay}} and {{.Sub2.Nay}}`)
	decoratingExecute(data)
	/*
		---
		sub1: nay1
		sub2: yay2
		sub3: yay3
		yay1 and nay2
		---
		error: <nil>
	*/
}
```

### .tmpl / .gotmpl拡張子で保存する

ソースコード中にstring literalとしてtemplateを記述することもできますが、個別のファイルに保存すると`gopls`(言語サーバー)の支援が受けられます。

https://github.com/golang/tools/blob/55d718e5dba2aaaa12d0a2ab2c11c7ac7eb84fcb/gopls/doc/features/templates.md

強調したくて前述しましたが、`gopls`を以下のように設定するとsyntax highlightがかかります。それ以外の機能は設定なしでも機能しているようです。

```json
{
  // ...other settings...
  "gopls": {
    // ...other settings...
    "ui.semanticTokens": true,
    "build.templateExtensions": ["gotmpl", "tmpl"]
    // ...other settings...
  }
  // ...other settings...
}
```

`"ui.semanticTokens": true`を有効にすると全体的にトークンの色の付け方が変わるので、びっくりするかもしれません。

### embed.FS, ParseFS

例示されるコードは以下でもホストされます。

https://github.com/ngicks/go-example-code-generation/tree/main/template/parse-fs

`//go:embed`によりtemplateを収めたディレクトリを丸ごとソースに埋め込み、`template.ParseFS`によって`fs.FS`をwalkしてそれぞれのファイルを`Parse`できます。

例としてファイルを以下のように配置します。
前述の`gopls`の支援を受けるために拡張子は`.tmpl`にしてあります。

```
.template/
|-- root.tmpl
|-- sub1.tmpl
|-- sub2.tmpl
`-- sub3.tmpl
```

各templateの中身のは[sub-template](#sub-template)の同名ものとそれぞれ変わりませんが、以下のように名前だけ若干変わります。

```tmpl
sub1: {{template "sub1.tmpl" .Sub1}}
sub2: {{template "sub2.tmpl" .Sub2}}
sub3: {{template "sub3.tmpl" .Sub3}}
{{block "additional" .}}{{end}}
```

`ParseFS`でファイルを読み込むと以下の行の挙動により`Base`が名前になってしまうためです。

https://github.com/golang/go/blob/go1.22.5/src/text/template/helper.go#L172-L178

`gopls`の支援により以下ようなsyntax highlightがかかります。

![tmpl-syntax-highlighting-by-gopls](/images/go-code-generation-in-ways-tmpl-syntax-highlighting-by-gopls.png)

`main.go`と同階層にこのtemplateディレクトリがあるものとして、以下のようなコードで読み込んで実行します。
事項結果自体は[sub-template](#sub-template)のものと変わりません。

ポイントとしては`//go:embed`でディレクトリを指定すると、そのディレクトリまでのパス構造がそのまま保たれるため、今回の場合、この`templates` FSの直下に`template`ディレクトリがあってその中に各ファイルがある状態となります。
また、`fs.FS`のルールにより、`./template`は適切なパスではないので`template`を指定します([fs.ValidPath](https://pkg.go.dev/io/fs@go1.22.5#ValidPath))。

`template.ParseFS`の第二引数にvariadicな`patterns ...string`を渡すことができますが、それぞれが[fs.Glob](https://pkg.go.dev/io/fs@go1.22.5#Glob)に渡されるのため、[path.Match](https://pkg.go.dev/path@go1.22.5#Match)の条件を満たす必要があります。

```go
//go:embed template
var templates embed.FS

var root = template.Must(template.ParseFS(templates, "template/*"))

type param struct {
	Sub1, Sub2, Sub3 sub
}
type sub struct {
	Yay string
	Nay string
}

func main() {
	root = root.Lookup("root.tmpl")
	data := param{
		Sub1: sub{
			Yay: "yay1",
			Nay: "nay1",
		},
		Sub2: sub{
			Yay: "yay2",
			Nay: "nay2",
		},
		Sub3: sub{
			Yay: "yay3",
			Nay: "nay3",
		},
	}
	fmt.Println("---")
	err := root.Execute(os.Stdout, data)
	fmt.Println("---")
	fmt.Printf("err: %v\n", err)
	/*
		---
		sub1: yay1
		sub2: yay2
		sub3: yay3

		---
		err: <nil>
	*/

	_, _ = root.New("additional").Parse(`{{.Sub1.Yay}} and {{.Sub2.Nay}}`)

	fmt.Println()
	fmt.Println("---")
	err = root.Execute(os.Stdout, data)
	fmt.Println("---")
	fmt.Printf("err: %v\n", err)
	/*
		---
		sub1: yay1
		sub2: yay2
		sub3: yay3
		yay1 and nay2
		---
		err: <nil>
	*/
}
```

各templateの名前から拡張子を取り除きたい場合は以下のように手動で挙動を作るしかないかと思います。

```go
//go:embed template
var templates embed.FS

var (
	extTrimmed *template.Template
)

func init() {
	tmpls, err := templates.ReadDir("template")
	if err != nil {
		panic(err)
	}
	cutExt := func(p string) string {
		p, _ = strings.CutSuffix(path.Base(p), path.Ext(p))
		return p
	}
	for _, tmpl := range tmpls {
		if tmpl.IsDir() {
			continue
		}
		if extTrimmed == nil {
			extTrimmed = template.New(cutExt(tmpl.Name()))
		}
		bin, err := templates.ReadFile(path.Join("template", tmpl.Name()))
		if err != nil {
			panic(err)
		}
		_ = template.Must(extTrimmed.New(cutExt(tmpl.Name())).Parse(string(bin)))
	}
}
```

### text/template example: enum

例示されるコードは以下でもホストされます。

https://github.com/ngicks/go-example-code-generation/tree/main/template/go-enum

code generatorとしてかかわりそうな機能は一通り説明したと思います。このまま終わってもいいんですが、code generatorという立て付けで記事を作っているのですから最後にcode generatorのサンプルを示します。

- `type Foo string`な、string-base typeのみを生成します。
- `const (...)`でvariantsを列挙し、
- `IsFoo`で入力がvariantsかどうかを判定します。
- これだけだとつまらないので「特定のvariantsではない」という判定も作れるようにします(`IsFooExceptBar`)

このExceptの生成部分はサンプルにするために無理くり別のtemplateにくくりだしていますが、このぐらいのサイズなら1つのままにしておいたほうが読みやすいと思います。

ポイント的には

- `{{{pipeline}}`という感じで`{`の後にactionを実行したい場合`{{"{"}}{{pipeline}}`としないといけない
- ユーザーの入力文字列を`Go`の`ident`(identifier)として出力するには`Go` specを満たすようにエスケープが必要([identifier = letter { letter | unicode_digit }.](https://go.dev/ref/spec#Identifiers))
  - 以下のサンプルでは`unicode.IsLetter(r) || unicode.IsDigit(r) || r == '_'`でないとき`_`に置き換えるという少々雑な処理でごまかしていますが、実際には`u1234`という感じでunicode番号に置き換えるとかそういうことをしたほうが良いのだと思います

このサイズでも結構読むのはしんどいと思います。機能が豊富で何でもできるのは便利ですね。

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

var funcs = template.FuncMap{
	"capitalize": func(s string) string {
		if len(s) == 0 {
			return s
		}
		if len(s) == 1 {
			return strings.ToUpper(s)
		}
		return strings.ToUpper(s[:1]) + s[1:]
	},
	"replaceInvalidChar": func(s string) string {
		// As per Go programming specification.
		// identifier = letter { letter | unicode_digit }.
		// https://go.dev/ref/spec#Identifiers
		return strings.Map(func(r rune) rune {
			if unicode.IsLetter(r) || r == '_' || unicode.IsDigit(r) {
				return r
			}
			return '_'
		}, s)
	},
	"quote": func(s string) string {
		return strconv.Quote(s)
	},
	"fillName": func(p EnumExceptParam, name string) EnumExceptParam {
		p.Name = name
		return p
	},
}

var (
	pkg = template.Must(template.New("package").Funcs(funcs).Parse(
		`// Code generated by me. DO NOT EDIT.
package {{.PackageName}}

import (
	"slices"
)

type {{.Name}} string

const (
{{range .Variants}}	{{$.Name}}{{replaceInvalidChar (capitalize .)}} {{$.Name}} = {{quote .}}
{{end -}}
)

var _{{.Name}}All = [...]{{.Name}}{{"{"}}{{range .Variants}}
	{{$.Name}}{{replaceInvalidChar (capitalize .)}},{{end}}
}

func Is{{.Name}}(v {{.Name}}) bool {
	return slices.Contains(_{{.Name}}All[:], v)
}

{{range .Excepts}}
{{template "except" (fillName . $.Name)}}{{end}}`))
	_ = template.Must(pkg.New("except").Parse(
		`func Is{{.Name}}Except{{replaceInvalidChar (capitalize .ExceptName)}}(v {{.Name}}) bool {
	return !slices.Contains(
		[]{{.Name}}{{"{"}}{{range .ExcludedValiants}}
			{{$.Name}}{{replaceInvalidChar (capitalize .)}},{{end}}
		},
		v,
	)
}
`))
)

func main() {
	pkgPath := filepath.Join("template", "go-enum", "example")
	err := os.MkdirAll(pkgPath, fs.ModePerm)
	if err != nil {
		panic(err)
	}

	f, err := os.Create(filepath.Join(pkgPath, "enum.go"))
	if err != nil {
		panic(err)
	}

	err = pkg.Execute(
		f,
		EnumParam{
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
		},
	)
	if err != nil {
		panic(err)
	}
}
```

これを実行すると以下を出力します

```go
// Code generated by me. DO NOT EDIT.
package example

import (
	"slices"
)

type Enum string

const (
	EnumFoo Enum = "foo"
	EnumB_ar Enum = "b\"ar"
	EnumBaz Enum = "baz"
)

var _EnumAll = [...]Enum{
	EnumFoo,
	EnumB_ar,
	EnumBaz,
}

func IsEnum(v Enum) bool {
	return slices.Contains(_EnumAll[:], v)
}


func IsEnumExceptFoo(v Enum) bool {
	return !slices.Contains(
		[]Enum{
			EnumFoo,
		},
		v,
	)
}

func IsEnumExceptMuh(v Enum) bool {
	return !slices.Contains(
		[]Enum{
			EnumFoo,
			EnumB_ar,
		},
		v,
	)
}
```

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

[text/template example: enum](#text%2Ftemplate-example%3A-enum)と同じものを[github.com/dave/jennifer]で再実装します。

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

## ast(dst)-rewrite

１からastをくみ上げることでコードを生成することもできますが、それをやるならば上記の`text/template`か`github.com/dave/jennifer`を用いるほうが楽なはずなので、ここでは深く紹介しません。
その代わり、astや型情報

### ソースコードの解析

#### go/parser

astは`go/token`, `go/parser`を用いて解析します。

`Go`のastはastと言いながら各Exprの位置情報が記録されています。これは`go/printer`による逆変換ができるようにするためかもしれません。

1ファイルのみを読み込むには以下のようにします。

出力は少々長くなるので省略しました。なので、以下のplaygroundで実行するか、[ソース](https://github.com/ngicks/go-example-code-generation/ast/print/ast.go)をコピーしてローカルで実行してみてください。

[playground](https://go.dev/play/p/QZ7x7sFeNWB)

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

`token.NewFileSet`で[\*token.FileSet](https://pkg.go.dev/go/token@go1.22.5#FileSet)をallocateして、[parser.ParseFile](https://pkg.go.dev/go/parser@go1.22.5#ParseFile)で第３引数を解析します。ドキュメントにある通り、[nil, []byte, string, io.Readerのいずれかを受け付け, nilの場合第二引数のfilenameを読み込みます](https://github.com/golang/go/blob/go1.22.5/src/go/parser/interface.go#L24-L42)。

#### ast.Print

解析された[\*ast.File](https://pkg.go.dev/go/ast@go1.22.5#File)を[ast.Print](https://pkg.go.dev/go/ast@go1.22.5#Print)もしくは[ast.Fprint](https://pkg.go.dev/go/ast@go1.22.5#Fprint)に渡すことで内部の構造をプリントすることができます。
これは要するに`reflect`によってgo structをwalkする関数です。ですので、`Go`のコードからどのようにastを扱えばいいのかがわかります。

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

[\*ast.GenDecl](https://pkg.go.dev/go/ast@go1.22.5#GenDecl)は

> A GenDecl node (generic declaration node) represents an import, constant, type or variable declaration. A valid Lparen position (Lparen.IsValid()) indicates a parenthesized declaration.

であるので、`*ast.File`のトップレベルにあるものは関数宣言以外はすべてこれになります。`var`や`type`は`var()`でグループを持てるため、`Spec`フィールドは`[]ast.Spec`というsliceになっています。`()`によるグルーピングがかかっていない場合はparenthesis(`(`)がないわけですからこのast nodeには`Lparen`と`Rparen`(省略されて表示されていないが)にはemptyな値が収められています。

`TypeParam`のindex(`[]`)の中身はstructの１つのfieldと同じ構文ルールが適用できるので`ast.FieldList`が使われていますね。ここはちょっと筆者的には驚きでした。

とまあそういった感じです。

#### golang.org/x/tools/go/packages

[ast.ParseDir](https://pkg.go.dev/go/parser@go1.22.5#ParseDir)が返す[\*ast.Package](https://pkg.go.dev/go/ast@go1.22.5#Package)が[Go1.22からdeprecatedになっている](https://tip.golang.org/doc/go1.22#minor_library_changes)ため、ディレクトリの中身を一気にパーズしたいとき何使えばいいんだよってなりますよね。

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

[\*packages.Config](https://pkg.go.dev/golang.org/x/tools@v0.23.0/go/packages#Config)をいろいろ設定し、[packages.Load](https://pkg.go.dev/golang.org/x/tools@v0.23.0/go/packages#Load)の第一引数として渡します。第二引数はvariadicなパターンで、`go`コマンドに渡すようなpackage patternを渡して読み込みたいパッケージを指定できます。

##### pattern

[packages.Load](https://pkg.go.dev/golang.org/x/tools@v0.23.0/go/packages#Load)の第二引数にはvairadicなpatternを渡し、これによってロードするパッケージを指定します。

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

[packages.Load](https://pkg.go.dev/golang.org/x/tools@v0.23.0/go/packages#Load)の第一引数には[\*packages.Config](https://pkg.go.dev/golang.org/x/tools@v0.23.0/go/packages#Config)を渡します。

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
  - コードを追ってみる限り`json.Marshal`してファイルシステムに書き出された後`-overlay`オプションに渡されます。
  - `-overlay`オプションそのものは今回の話題に対して重要ではないので、ここでは説明を避け[go Command Documentation](https://pkg.go.dev/cmd/go)を読むようにとだけ書いておきます。

##### packages.Visit

[packages.Visit](https://pkg.go.dev/golang.org/x/tools@v0.23.0/go/packages#Visit)で、ロードされたパッケージをインポートグラフ順にvisitできます。

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

[\*packages.Package](https://pkg.go.dev/golang.org/x/tools@v0.23.0/go/packages#Package)の`Fset`フィールドは`*token.FileSet`,`Syntax`フィールドは`[]*ast.File`なので、`ast.Print`などast情報を使った処理ができます。

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

[\*packages.Package](https://pkg.go.dev/golang.org/x/tools@v0.23.0/go/packages#Package)の.Typesフィールドは`*types.Package`なので、型情報を使って処理を行うことができます。

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

### directive commentとその解析方法

`Go`には[Compiler directiveをコメントとして書くことができます。](https://pkg.go.dev/cmd/compile#hdr-Compiler_Directives)
このdirective commentはdoc commentに出現しないようです。なので、directive commentでcode generatorに対して指示を出せるとdoc commentを邪魔せず、メタデータを別ファイルに分けることなく`Go`のsource codeに追加できるため便利かもしれません。
ただし、directive commentの解析をastから行うには若干の工夫がいるので以下でその方法を述べます。

#### directive comment

通常のコメントは下記のように、コメントの開始に半角スペースなどを１つ以上入れますが、directive commentは半角などを入れません。

```go
// non-directive comment
/* non-directive comment */

//directive comment
```

[compiler directive](https://pkg.go.dev/cmd/compile#hdr-Compiler_Directives)の項目では説明されていませんが、`//go:embed`のように色々なマジックコメントがwildに存在します。

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

astをトラバースするには以下などの関数を用います。どれもdepth-first orderです。

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

今回は`pre`のみを使っていきます。

今回は`*ast.GenDecl`を探索するので
`ast.Apply`に`*ast.File`を渡すと、[package comment, package name, Declsの順](https://github.com/golang/tools/blob/v0.24.0/go/ast/astutil/rewrite.go#L429-L436)でwalkしていくのでdefault branchで`return true`しないとうまいこと進んでくれません。

コメントを書き換えるのでnodeとコメントの関連付けを保つために[\*ast.CommentMap](https://pkg.go.dev/go/ast@go1.22.6#CommentMap)を先に作成します。

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
これは単に`type Foo string`をparseした結果を`ast.Print`して確かめた通りに記述しているだけです。

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

前述のとおり、`*ast.CommentGroup`の`Text`ではdirective commentが除外されてしまうので`List`を走査します。

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

`*astutil.Cursor`をそのまま受け取ります。呼び出し側でcursorは`*ast.GenDecl`をさしているので、Parent=`*ast.File`を`astutil.Apply`でwalkします。
仕様で説明した通り、特定のコメントがついた`const ()`を探して、あれば置き換え、なければ追加します。

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

#### \*ast.CommentMap

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
以下のように`decorator`パッケージを利用して`*dst.File`

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
ですので、doc commentに当たるindexの範囲を探索してからその範囲を先頭から走査したほうがよいでしょう。

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
	if len(targetDecoration.Start) > 0 {
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
+
		err = printer.Fprint(out, pkg.Fset, f)
		if err != nil {
			panic(err)
		}
	}
```

#### 結果

https://github.com/ngicks/go-example-code-generation/blob/main/ast/rewrite/dstutil

`ast`版ではうまくいかなかったに対し、

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

## post process: goimports

生成したコードは`goimports`によってフォーマットをかけてから書き出すとよいでしょう。
これにより、万一code generatorの実装ミスや、ユーザーが指定できるパラメータのvalidationががおかしくて生成されたコードが`Go`の文法を満たさない場合にエラーとして検知が可能です。

以下のコードでは`go run golang.org/x/tools/cmd/goimports@latest`するのではなく、システムにインストール済みの`goimports`を利用します。
なので、`checkGoimports`を他の生成ロジックより前に呼び出して、このプロセスが見ることができる位置に`goimports`が存在するかを確認しておくほうが無難です。

別に`go run`で`goimports`を呼び出しても問題ないことのほうが多いと思います。ただ、`go run`はモジュールが`$GOPATH`以下にキャッシュされていない場合などにダウンロードをしますから、ダウンロード関連のエラーも起きうるわけです。ここでダウンロード周りのエラーを起されても困るのでこうしています。

```go
func checkGoimports() error {
	_, err := exec.LookPath("goimports")
	return err
}

func applyGoimports(ctx context.Context, r io.Reader) (*bytes.Buffer, error) {
	cmd := exec.CommandContext(ctx, "goimports")
	cmd.Stdin = r
	formatted := new(bytes.Buffer)
	stderr := new(bytes.Buffer)
	cmd.Stdout = formatted
	cmd.Stderr = stderr
	err := cmd.Run()
	if err != nil {
		return nil, fmt.Errorf("goimports failed: err = %v, msg = %s", err, stderr.Bytes())
	}
	return formatted, nil
}
```

一応補足ですが、code generatorは大体`io.Writer`に書き出すようになっているため、`goimports`にその出力を渡すには一旦`*bytes.Buffer`に出力するか、でなければ`io.Pipe`を使ってパイプを行ってください。
出力は`Go`のソースコードなのでメモリに置けないほど大きくなることは想定する必要がないはずです。基本的には`*byte.Buffer`を経由して持ちまわって問題ありません。

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
