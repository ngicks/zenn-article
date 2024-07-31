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

`Go`のstdに関するドキュメントおよびソースコードはすべて`Go1.22.5`のものを参照します。

コードを実行する環境は`1.22.0`です。

```
# go version
go version go1.22.0 linux/amd64
```

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

### go:generate go run --mod=mod

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

ちなみに`Go 1.19`以降`flag`パッケージは`-x`でも`--x`でもオプションを受け付けるようになったので、`-mod=mod`でも`--mod=mod`でも同じ意味です。そういう意味でタイトルはわざとです。

## io.Writerに書くだけ

という風にタイトルをつけていますが特にこれと言って述べるべきことはこれにはありません。

例えば、`Go`の`runtime`では以下のような単にテキストを書くだけのcode generatorを見つけることができます。

https://github.com/golang/go/blob/go1.22.5/src/runtime/wincallback.go

生成対象は`.s`の[Go assembly](https://go.dev/doc/asm)ファイルですが、まあ言いたいことはかわらないのでいいとしましょう。

このコードによって以下のがファイルが生成されます。

https://github.com/golang/go/blob/go1.22.5/src/runtime/zcallback_windows_arm.s
https://github.com/golang/go/blob/go1.22.5/src/runtime/zcallback_windows_arm64.s
https://github.com/golang/go/blob/go1.22.5/src/runtime/zcallback_windows.s

これらのファイルは以下のような、ほぼ同じパターンを2000回繰り返すだけの単純なものです。

```
	MOVD	$i, R12
	B	runtime·callbackasm1(SB)
```

このように、本当に単純なコード断片を、同じパラメータを何度も使いまわすとかそういうことがない場合はこういう風に、
単なる`io.Writer`への書き出しで十分機能します。

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
  - `gopls`(言語サーバー)による支援はあるが、syntax highlightはまだ未実装
  - `for`がネストしだすと劇的に視認性が落ちる
  - 空白の取り扱いが難しい。
    - 筆者は無駄な改行を甘んじて受け入れている
  - `goimports`によってフォーマットをかけることでいくらか改善する

### 基本的な使用法

パラメータ、関数その他の呼び出しは`{{`と`}}`で囲まれたブロックの中で行います。

```tmpl
An example template.
Hello {{.Gopher}}.
Yay Yay.
```

というtemplateでは`{{.Gopher}}`の部分が入力のパラメータによって動的に変更されることになります。

このdelimiter(`{{`,`}}`)は[(\*Template).Delims](https://pkg.go.dev/text/template@go1.22.5#Template.Delims)設定変更できますが基本的に変更することは想定しません。

`template.New()`で新しい`*Template`をallocateし、`Parse`によってtemplateテキストを解析して`*Template`オブジェクトを得ます。

```go
var example = template.Must(template.New("").Parse(
	`An example template.
Hello {{.Gopher}}.
Yay Yay.
`,
))
```

`template.Must`は`(*Template, error)`を引数にとって、第二引数のエラーがnon-nilだった場合panicするヘルパー関数です。

```go
type sample struct {
	Gopher string
}

err := example.Execute(os.Stdout, sample{Gopher: "me"})
```

で、を渡した`io.Writer`に書き出します。`os.Stdout`を渡しているのでstdoutに書き出されます。

```
An example template.
Hello me.
Yay Yay.
```

上記がstdoutに出力されます。

### パラメータへのアクセス

`Execute`の第二引数にはパラメータを詰め込んだデータ構造を渡します。
`{{.}}`の`.`はcontextualな値で、トップレベルでは`Execute`に渡したデータそのものをさしています。
`.Gopher`のようにdot selectorで**_reflectでアクセスできるフィールドを指定する_**とそのフィールドのデータを取り出せます。

メソッドでもよいです。

```go
type sample struct {
	Gopher string
}

type sampleMethod1 struct {
}

func (s sampleMethod1) Gopher() string {
	return "method"
}

err := example.Execute(os.Stdout, sampleMethod1{})
/*
An example template.
Hello method.
Yay Yay.
*/
```

メソッドは第二返り値でエラーを返してもよいです。

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
fmt.Printf("template execution error:  %#v\n", err)
/*
---
An example template.
Hello ---
template execution error:  template: :2:8: executing "" at <.Gopher>: error calling Gopher: sample
*/
```

同様に、`map[K]V`でもいいです。

```go
err := example.Execute(os.Stdout, map[string]string{"Gopher": "map[string]string"})
/*
An example template.
Hello map[string]string.
Yay Yay.
*/
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

### sub-template

### .tmpl / .gotmpl拡張子で保存する

ソースコード中にstring literalとしてtemplateを記述することもできますが、個別のファイルに保存すると`gopls`(言語サーバー)の支援が受けられます。

https://github.com/golang/tools/blob/55d718e5dba2aaaa12d0a2ab2c11c7ac7eb84fcb/gopls/doc/features/templates.md

### embed.FS, ParseFS

## github.com/dave/jennifer

## ast(dst)-rewrite

１からastをくみ上げることでコードを生成することもできますが、それをやるならば上記の`text/template`か`github.com/dave/jennifer`を用いるほうが楽なはずなので、ここでは深く紹介しません。
その代わり、astや型情報

### astutilを使った書き換え

### dstを使った書き換え

https://stackoverflow.com/questions/31628613/comments-out-of-order-after-adding-item-to-go-ast

## post process: goimports

生成したコードは`goimports`によってフォーマットをかけてから書き出すとよいでしょう。
これにより、万一code generatorの実装ミスや、ユーザーが指定できるパラメータのvalidationががおかしくて生成されたコードが`Go`の文法を満たさない場合にエラーとして検知が可能です。

以下のコードでは`go run golang.org/x/tools/cmd/goimports@latest`するのではなく、システムにインストール済みの`goimports`を利用します。
なので、`checkGoimports`を他の生成ロジックより前に呼び出して、このプロセスが見ることができる位置に`goimports`が存在するかを確認しておくほうが無難です。

別に`go run`で`goimports`を呼び出しても問題ないことのほうが多いと思いますが、万一呼び出し側の環境に`goimports`がなく、モジュールのダウンロード/ビルドでエラーを起こした場合、よくわからないエラーメッセージを吐いてしまう可能性があります。
そういったエラーを切り分けたほうが面倒がないかと思って今回は`go run`を使わないサンプルになっています。
また、[VscodeのGo extensionがgoimportsをダウンロードする](https://github.com/golang/vscode-go/blob/0099728ac6e476c0dc016a03501e79d096fe918b/extension/tools/installtools/main.go)のでそもそも`goimports`がない環境をあまり想定していません。

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
