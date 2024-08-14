---
title: "Goのcode generation: text/template"
emoji: "🧰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## Goのcode generationについてまとめる

`Go`のcode generationについてまとめようと思います。

この記事では

- Rationale: なぜGoでcode generationが必要なのか
- code generatorを実装する際の注意点など
- `io.Writer`に書き出すシンプルな方法
- `text/template`を使う方法
  - `text/template`のcode generationにかかわりそうな機能性について説明します。
  - 実際に`text/template`を使ってcode generatorを実装します。

について述べ、

後続の

- [Goのcode generation: jennifer](https://zenn.dev/ngicks/articles/go-code-generation-in-ways-jennifer)で[github.com/dave/jennifer]を用いる方法
- [Goのcode generation: ast(dst)-rewrite](https://zenn.dev/ngicks/articles/go-code-generation-in-way-ast-dst)で[astutil](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/ast/astutil)および[github.com/dave/dst]を用いる方法

についてそれぞれ述べます。

## 前提知識

- [The Go programming language](https://go.dev/)の基本的文法、プロジェクト構成などある程度`Go`を書けるだけの知識

## 環境

`Go`のstdに関するドキュメントおよびソースコードはすべて`Go1.22.6`のものを参照します。
[golang.org/x/tools](https://pkg.go.dev/golang.org/x/tools@v0.24.0)に関してはすべて`v0.24.0`を参照します。

コードを実行する環境は`1.22.0`です。

```
# go version
go version go1.22.0 linux/amd64
```

書いてる途中で`1.23.0`がリリースされちゃったんですがでたばっかりなんで`1.22.6`を参照したままです。ﾏﾆｱﾜﾅｶｯﾀ。。。

## Rationale: なぜGoでcode generationが必要なのか

[Go]は、`C`や[Rust](https://doc.rust-lang.org/book/ch19-06-macros.html)にあるようなマクロを言語機能として持っていないため、たびたびcode generationを行いますし、それを行うことは前提のようになっています。
それは`go generate`というサブコマンドが存在することや、`Go`のstd自身がそれを多用することから様式として存在していることがわかります。
また、`Go`にはgenericsによるある型セットに対する共通した処理と、`reflect`による型構造への動的な処理を実装できますが、これらは当然ソースコードの解析を必要とするような挙動は実装できませんので、それらを必要とする場合はcode generationが必要となります。

### go:generate

[The Go BlogのGenerating code](https://go.dev/blog/generate)にも書かれている通り、`Go 1.4`から`Go`には[go generate](https://pkg.go.dev/cmd/go#hdr-Generate_Go_files_by_processing_source)というサブコマンドが追加されました。
これはgo source codeに書かれた`//go:generate`マジックコメントのあとにスペースを挟んで書かれた任意をコマンドを、そのソースファイルの位置をcwdに指定して実行するというものです。
`go generate`は任意のコマンドを実行できますが、基本的にcode generatorを実行してコードを生成することを想定した仕組みです。

`Go`自身も`//go:generate`を活用しており、

https://github.com/search?q=repo%3Agolang%2Fgo%20%2F%2Fgo%3Agenerate&type=code

以上のように検索してみればたくさんヒットします。

### goのgeneric function

`Go`では[reflect](https://pkg.go.dev/reflect@go1.22.5)を使うことで型情報を`any`な値から取り出すことができ、これを元に動的な挙動を行うことができます。
また、[Go 1.18で追加されたGenerics](https://tip.golang.org/doc/go1.18#generics)を用いることで、ある制約を満たす複数の型に対して処理を共通化できます。

例えば、以下のようなサンプルを定義します。

サンプルでは、あるstruct(`Sample`)に対して、フィールド名と定義順が一致するが、型が`Patcher[T]`で置き換えられたstruct(`SamplePatch`)を用意することで、部分的なフィールドの変更(=Patch)をする挙動を`reflect`を使って実装できることを示します。

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

上記のサンプルでは`Sample`, `SamplePatch`に対してのみしか動作が確かめられていませんが、実際には条件を守るあらゆるstructのペアに対してPatchを行うことができます。

- `reflect`を使用することでstructなどのデータ構造に対して動的な処理を実装できます
- `generics`を利用することで任意の制約を満たす型に対して共通した処理を実装できます
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

**内部的な挙動は`reflect`で作りこむにしろ、具体的な型を当てたラッパーはcode generatorで作成したいということはよくあるはずだ、ということです。**
(`reflect`使わずcode generatorでこういう挙動をするコードを吐き出してもいいんですがここでは気にしません！)

- `generics`では[#49085](https://github.com/golang/go/issues/49085)がないためにmethodにtype paramを与えることができません。

つまり以下のようなことはできないということです。

```go
// compilation error
func (p Patch[T]) Convert[U any](converter func(t T) U) U {
	// ...
}
```

メソッドにtype paramが持てないため**複数の型にほぼ同じ処理のメソッドを実装したい場合はcode generatorを作ったほうがメンテが楽だったりすることもあるということです。**

また、双方ともに型情報に含まれないような情報を用いた処理を行えません。

例えば、code generatorである[golang.org/x/tools/cmd/stringer](https://pkg.go.dev/golang.org/x/tools/cmd/stringer)は以下のような、iotaを用いたenum風なconst定義に対して

https://github.com/golang/go/blob/go1.22.5/src/html/template/context.go#L80-L161

以下のように`String`メソッドを生成します。

https://github.com/golang/go/blob/go1.22.5/src/html/template/state_string.go

**これらはソースコードの解析その他を行わない限り不可能なことですので、こういったことをしたい場合はcode generatorが必要になります。**

## 4つの(おそらく)代表的な方法

大雑把に言って4つの方法が代表的なのではないかと思います

- simple text emitter: `io.Writer`にテキストを書くだけ
  - プログラムによってgo source fileとなるテキストを書きだすだけの方法です
- [text/template]を用いる方法
  - stdで実装されるテンプレートエンジンを用いる方法です
- [github.com/dave/jennifer]を用いる方法
  - サードパーティで実装されるcode generatorを記述するためのライブラリを用いる方法です
  - goのトークンや構文に対応した各種関数をメソッドチェーンで記述していく方式です。
- `ast`(dst)-rewriteを行う方法
  - Goのsource codeを解析しast(abstract syntax tree)を得てそれをもとにコードを生成する方法です。
  - `go/ast`, `go/parser`, `go/printer`などのstd libraryを用います。
  - `ast`で1からコードをくみ上げることも当然可能ですが、前述のいずれかの方法をとったほうが簡単なので、rewriteする方法についてのみ述べます
  - `ast`のrewriteではコメントのオフセット周りに問題があるため、[github.com/dave/dst]を代わりに用います

上記を整理しなおすを以下のような関係図になります

![code-generation-data-flow](/images/go-code-generation-in-ways-data-flow.drawio.png)

おおむねコード生成のためのメタデータ取得部と、コード生成部と、コードの書き出し部分、最後のポストプロセスとしてフォーマットに分かれると思います。

メタデータ取得部分は`ast`を用いる場合はgo source codeを入力とし、`go/parser`を用いて`ast`の解析します。`ast`は、さらに`type checker`で解析すること型情報を得ることもできます(e.g. [skeleton](https://github.com/golang/example/blob/39e772fc26705bb170db248e5372a81ed5ffd67f/gotypes/skeleton/main.go))。この一連の記事では`type checker`関連の話には踏み込みません。
他の方法では`JSON`, `YAML`のようなフォーマットで書かれたデータ構造(ここにcli引数も含む)を`json.Unmarshal`や`yaml.Unmarshal`してデータ構造にbindしたり、[text/template]向けのテキストを読み込んだりします。

コード生成部(図の`jennifer`,`simple text emitter`, `text/template`および`ast rewrite`部分のこと)では、えられたメタデータを元に`io.Writer`に書き出したり、[text/template]を用いるなど前述の方法の一部または全部を組み合わせて行います。`ast`をrewriteする方法では`go/printer`の機能を利用することで、あるnodeのみを出力するようなことができるので、これも他の方法と組み合わせて1つのテキストファイルを形成することができます。

コード生成部によってテキストを出力します。このテキストは有効なgo source codeの文法を満たしてさえいれば(i.e. `package pkgname`から始まるgo code)この時点でファイルとして書きだされている必要はありません。

go source codeのテキスト、またはテキストのストリームは[gofmt](https://pkg.go.dev/cmd/gofmt), [github.com/mvdan/gofumpt](https://github.com/mvdan/gofumpt), [golang.org/x/tools/cmd/goimports](https://pkg.go.dev/golang.org/x/tools/cmd/goimports)などのフォーマッターを用いることでフォーマットをできます。`goimports`は`gofmt`と同じルールでフォーマットを行ったうえで、import declが正しくなかった場合修正をこころみます。code generatorの実装する際、完璧なインデントを保ったり、使わないimportを削除したりが大変なことがあります。`Go`では不要なimportが存在するとコンパイルが通りませんので、`goimports`にそれらを修正してもらうことで楽ができます。

最後に、ファイルとして書きだされたgo source codeは[gopls](https://github.com/golang/tools/tree/master/gopls)の機能を用いてフォーマットをかけることができます。ユーザーの`gopls`設定を元にフォーマットを行いたい場合は便利かもしれませんが基本的にはしません。理由はよくわかっていませんが、`goimports`などを直接呼び出す方法に比べてずいぶん動作速度が遅い(0.1秒オーダーに比べて数秒オーダー)ためです。

### それぞれの方法の利点と欠点

- simple text emitter: `io.Writer`にテキストを書くだけ
  - 利点: すごいシンプルなのですぐかける
  - 欠点: シンプルなので複雑なケースに対応できない
    - パラメータを複数回使いまわすとか
    - ifで分岐するとか
    - ユーザーから入力を受けたい場合、などに対応しにくいです。
      - するならほかの方法を使うほうが良いです
- [text/template]を用いる方法
  - 利点: stdで終始できる
    - ここが最大の利点だと思います。std以外を全く何もimportしないで済みます
    - 複数templateへの分割、関数の任意な追加、ユーザーからtemplateの入力を受け付けなど複雑なケースに対応できます
  - 欠点: 読みにくい
    - `gopls`(`Go`の言語サーバー)によるsyntax highlightなどの支援を受けられますが、生来の複雑さを持っため`for`がネストしだす本当に読みにくいです。
    - そもそもtemplate用途なので、`Go`のcode generator専用にしつらえられた`jennifer`のほうが使いやすいのは当然ではあります。
- [github.com/dave/jennifer]を用いる方法
  - 利点: 読みやすい/書きやすい
    - `Go`の関数呼び出しをチェーンさせるだけなのでsyntax highlightがしっかりかかる
    - それぞれの関数は`Go`のトークンや構文ルールに一致するので、違和感は少ない
    - 単なる`Go`コードなので、任意に分割できる
    - importの取り扱いを自動的に行う機能があるため、楽
  - 欠点: とくにない？
    - しいて言えばユーザーからシリアライズされた部分的なtemplateを受けとる方法が特に決まっていないので、自ら実装する必要があります
    - ただしその場合、`text/template`のテンプレートを受け付けて別ファイルに出力すればよいだけにも思います。
    - サードパーティのライブラリをインポートすることなりますので、そこが気になるケースでは採用しずらいです。
- `ast`(dst)-rewriteを行う方法
  - 利点: 既存のgo source codeを入力とできる。
    - 入力をGo source codeとできるのは当然この方法だけです。
  - 欠点: astの変更や、１からastをくみ上げるのは手間がかかる
    - `Go`のソースコードを直接書きに行くほかの方法に比べてたった1つのトークンを書くだけでも何倍もの文字を打つ必要があってかなり面倒です。
    - そのためこの記事ではrewriteする方法しか想定しません。

## misc: codeを生成する際の注意点

どの方法にもよらない注意点などをここにまとめておきます。

### ファイル先頭に// Code generated ... DO NOT EDIT.をつける

[go generateのドキュメント](https://pkg.go.dev/cmd/go#hdr-Generate_Go_files_by_processing_source)にもある通り、
`^// Code generated .* DO NOT EDIT\.$`という正規表現にマッチする行が**package declarationより前**に含まれる場合、`go tool`はこれをcode generatorによって生成されたファイルであるとみなします。
`Code generated`の後の`.*`の部分にcode generatorのpackage pathを書いておくとどうやって生成したのかわかってよいのではないかと思います。

`Go1.21`より[ast.IsGenerated](https://pkg.go.dev/go/ast@go1.22.5#IsGenerated)という関数がexportされるようになったので、ast解析を行って`*ast.File`がえられており、それがcode generatorに生成されたファイルかの確認が行いたい場合はこれを用いるとよいでしょう。

### for-range-mapの部分で毎回異なる順序で生成してしまうことがあるので注意する

code generator実装の内部で`fo-range-map`をしてしまうと、実行ごとに異なる順序になることがあるため、そうならないための気遣いが必要です。

> https://go.dev/ref/spec#For_range
>
> The iteration order over maps is not specified and is not guaranteed to be the same from one iteration to the next.

`Go`の言語仕様により`for-range-map`の順序は未定義です。

code generatorが内部で`for-range-map`を行っており、これがそのまま結果の出力順序に反映されていると実行のたびに結果が異なることがありあえます。
筆者が利用するサードパーティのcode generatorの中にも、生成する度に順序の入れ替わるものがありますが、生成対象が多くなるにつれて出てくるdiffの量が多くなってセルフレビューが大変になっています。
基本的にそうならないように作ったほうが利用者とっては便利です。

代わりに`Go 1.22.x`以前では

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
`Go1.23`以降ならもっと簡単に

```go
// https://go.dev/play/p/2yRGLquakg8
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

これは`README.md`などの中で、あなたの作成したcode generatorの呼び出し方をどのように指示するかという話なんですが、

あなたの作るcode generatorが生成するコードが何かしらの外部モジュールを必要とし、それがcode generatorと同じモジュールで管理されているとき、以下のように、`go run -mod=mod`で実行するよう指示するとよいでしょう。

```
# 架空のURLを取り扱うのでexample.comのサブドメインとして書いています
# url自体は興味のあるところではありません！
//go:generate go run -mod=mod fully-qualified.example.com/package/path/cmd/path/to/main/pkg@version
```

> https://go.dev/ref/mod#build-commands
>
> -mod=mod tells the go command to ignore the vendor directory and to [automatically update](https://go.dev/ref/mod#go-mod-file-updates) go.mod, for example, when an imported package is not provided by any known module.

とある通り、`-mod=mod`で動作させるとcode generatorのバージョンが、生成物の配置先となるgo moduleの`go.mod`に追加されるなり更新されるなりするらしいです。

### post process: goimports

生成したコードは`goimports`によってフォーマットをかけてから書き出すとよいでしょう。
これにより、万一code generatorの実装ミスや、ユーザーが指定できるパラメータのvalidationががおかしくて生成されたコードが`Go`の文法を満たさない場合にエラーとして検知が可能です。

以下のコードでは`go run golang.org/x/tools/cmd/goimports@latest`するのではなく、システムにインストール済みの`goimports`を利用します。
なので、`checkGoimports`を他の生成ロジックより前に呼び出して、実行プロセスが見ることができる位置に`goimports`が存在するかを確認しておくほうが無難です。

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

一応補足ですが、code generatorは大体`io.Writer`に書き出すようになっているため、`goimports`にその出力を渡すには一旦`*bytes.Buffer`に出力するか、でなければ`io.Pipe`を使ってパイプを行います。
出力は`Go`のソースコードなのでメモリに置けないほど大きくなることは想定する必要がないはずです。
多量のファイルの一気に処理するとかでない限り`*byte.Buffer`を経由して持ちまわって問題ないでしょう。

## simple text emitter: io.Writerに書くだけのほうほう

まずsimple text emitterと呼んでいた単に`io.Writer`に`Go source code`を書き出すだけの方法について述べます。

とは言え実際シンプルなのであまり述べることはありません。

例えば、`Go`の`runtime`では以下のような単にテキストを書くだけのcode generatorを見つけることができます。

https://github.com/golang/go/blob/go1.22.5/src/runtime/wincallback.go

生成対象は`.s`の[Go assembly](https://go.dev/doc/asm)ファイルですが、まあ言いたいことはかわらないのでいいとしましょう。

このコードによって以下の3つのファイルが生成されます。

https://github.com/golang/go/blob/go1.22.5/src/runtime/zcallback_windows_arm.s
https://github.com/golang/go/blob/go1.22.5/src/runtime/zcallback_windows_arm64.s
https://github.com/golang/go/blob/go1.22.5/src/runtime/zcallback_windows.s

これらのファイルは以下のような、ほぼ同じパターンを2000(=`maxCallback`)回繰り返すだけの単純なものです。

```
	MOVD	$i, R12
	B	runtime·callbackasm1(SB)
```

このように、単純なコード断片を何度も書きだすだけのようなケースでは、単なる`io.Writer`への書き出しで十分機能します。

## text/templateを用いる方法

`text/template`を用いる方法について述べます。

`text/template`は高機能でぱっと見難しいので、code generatorを作る際にかかわりそうな機能性について説明し、最後にcode generatorを実装してみることとします。

### text/template

https://pkg.go.dev/text/template@go1.22.5

stdライブラリに組み込まれたテンプレート機能です。

テンプレートなので、既存の型となるテキストの特定の部分を入力によって切り替えるものなのですが、`if`や`for`などが機能として組み込まれているのでおおよそ何でもできてしまいます。

`html/template`も存在しますが、こちらは`html`を出力するための各種サニタイズを実装した`text/template`のラッパーみたいなものですので、テキストの出力に関しては`text/template`を使用します。
エディターの自動補完に任せると`html/template`のほうがimportされることがありますので、なぜか出力文字列がエスケープされていたらimportを確認しましょう。

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

他のエディターの場合、`gopls`を似たような感じで設定します。

syntax highlight以外の機能は現状でも機能しているように見えるので、そこが不要なら設定は不要です。
`"ui.semanticTokens"`を有効にするとtemplateのみならず、Go source codeそのもののトークンの色がかなり変わって表示されますのでびっくりするかもしれません。

現状`gopls`の`semanticTokens`はexperimentalですがもうすぐenabled by defaultになるかもしれません([#45313](https://github.com/golang/go/issues/45313#issuecomment-2161267130))。

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

というtemplate textでは`{{.Gopher}}`の部分が入力のパラメータによって動的に変更されることになります。
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

`Execute`の第二引数にはパラメータを詰め込んだデータを渡します。
`{{.}}`の`.`はcontextualな値で、トップレベルでは`Execute`に渡したデータそのものをさしています。

渡すパラメータは任意の`Go struct`か`map[K]V`であれば、dot selectorでフィールドの値か、keyに収められている値がそれぞれ取り出されることが書かれています。
structを指定する場合は`reflect`パッケージを使って値にアクセスしますので、`reflect`でアクセスできるフィールドを指定する必要があります(=Exported)。

```go
var example = template.Must(template.New("").Parse(
	`An example template.
Hello {{.Gopher}}.
Yay Yay.
`,
))

type sample struct {
	Gopher string
}

err := example.Execute(os.Stdout, sample{Gopher: "me"})
/*
An example template.
Hello me.
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

同様に、`map[K]V`でもいいです。`map[K]V`の場合は先頭が小文字なフィールドにもアクセスできます。
template textが先頭が小文字なフィールドにアクセスしていたら`map[K]V`を使うしかなくなってしまいます。
型を決められるstructのほうが管理が容易であるため、基本的には先頭が大文字なフィールドしか使われることはないと思います。

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
	/*
		---
		Hi you.
			Iterating at 0: foo
			Iterating at 1: bar
			Hey you this is empty
		---
		error: <nil>
	*/
	decoratingExecute(map[string]any{
		"Gopher":   "you",
		"Continue": "ok",
		"Iter":     []map[string]string{{"Field": "foo"}, {"Field": "bar"}, {}, {"Field": "baz"}},
	})
	/*
		---
		Hi you.
			Iterating at 0: foo
			Iterating at 1: bar
			Hey you this is empty
			Iterating at 3: baz

		---
		error: <nil>
	*/

	decoratingExecute(map[string]any{
		"Gopher": "you",
		"Iter":   map[string]map[string]string{"0": {"Field": "foo"}, "1": {"Field": "bar"}, "2": {"Field": "baz"}},
	})
	/*
		---
		Hi you.
			Iterating at 0: foo
			Iterating at 1: bar
			Iterating at 2: baz

		---
		error: <nil>
	*/
}
```

何気なく使っていますが、`{{- pipeline}}`, `{{pipeline -}}`で前の/後ろの空白を削除する機能があります。

> For this trimming, the definition of white space characters is the same as in Go: space, horizontal tab, carriage return, and newline.

この「空白」の条件はGo source codeのそれと一致します。割とこの挙動が難しいので筆者は場合により無駄な空白や改行を甘んじて受け入れています。

### Funcs: 関数の追加

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

関数の引数の型は何でもいいですが、入力パラメータと一致しなければエラーになるようです。見たところ[(reflect.Type).AssignableToがfalseの場合エラー](https://github.com/golang/go/blob/go1.22.5/src/text/template/exec.go#L852-L862)です。

### multiple-template

例示されるコードは以下でもホストされます。

https://github.com/ngicks/go-example-code-generation/blob/main/template/multiple

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

筆者もこの記事を書くまで全くわかっていなかったのですが、`*Template`は以下の通り`*common`という構造体で解析されたtemplateを保持し、この`*common`は`(*Template).New`で作成されたすべての`*Template`に共有されています。

https://github.com/golang/go/blob/go1.22.5/src/text/template/template.go#L13-L35

- `Parse`はこの`*common`を上書きします。そのため、`Parse`や`Funcs`は`*common`を共有するすべての`*Template`に影響します。
- 同名のtemplateを複数定義している場合などでは`Parse`する順序によって結果が変わることになります。
- template同士はヒエラルキーのないフラットな構造で、お互い名前で参照しあうことができます。

以下で複数のtemplateを使用するサンプルを示します。

`tmp2.Parse`などを呼び出すことで、ユーザーから渡されたtemplate definitionによって元のtemplate構造を上書きしてカスタマイズが行えることを示します。

複数のtemplateを用い、さらに`additional`というデフォルトでは何も出力しないtemplateを後から`Parse`によって追加することで、ユーザーからのtemplateの入力ができることを示します。

```go
var (
	tmp1 = template.Must(template.New("tmp1").Parse(
		`tmp2: {{template "tmp2" .Tmp2}}
tmp3: {{template "tmp3" .Tmp3}}
tmp4: {{template "tmp4" .Tmp4}}
{{block "additional" .}}{{end}}
`))
	tmp2 = template.Must(tmp1.New("tmp2").Parse(`{{.Yay}}`))
	_    = template.Must(tmp1.New("tmp3").Parse(`{{.Yay}}`))
	tmp4 = template.Must(tmp1.New("tmp4").Parse(`{{.Yay}}`))
)

type param struct {
	Tmp2, Tmp3, Tmp4 sub
}
type sub struct {
	Yay string
	Nay string
}

func main() {
	decoratingExecute := func(data any) {
		fmt.Println("---")
		err := tmp1.Execute(os.Stdout, data)
		fmt.Println("---")
		fmt.Printf("error: %v\n", err)
		fmt.Println()
	}
	data := param{
		Tmp2: sub{
			Yay: "yay2",
			Nay: "nay2",
		},
		Tmp3: sub{
			Yay: "yay3",
			Nay: "nay3",
		},
		Tmp4: sub{
			Yay: "yay4",
			Nay: "nay4",
		},
	}

	decoratingExecute(data)
	/*
		---
		tmp2: yay2
		tmp3: yay3
		tmp4: yay4

		---
		error: <nil>
	*/
	_, _ = tmp2.Parse(`{{.Nay}}`)
	decoratingExecute(data)
	/*
		---
		tmp2: nay2
		tmp3: yay3
		tmp4: yay4

		---
		error: <nil>
	*/

	_, _ = tmp4.New("additional").Parse(`{{.Tmp2.Yay}} and {{.Tmp3.Nay}}`)
	decoratingExecute(data)
	/*
		---
		tmp2: nay2
		tmp3: yay3
		tmp4: yay4
		yay2 and nay3
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
    // どうもGo vscode extensionが以下と同様の
    // デフォルト値を入れているような振る舞いをするので、
    // これでいいなら設定は不要と思われる
    "build.templateExtensions": ["gotmpl", "tmpl"]
    // ...other settings...
  }
  // ...other settings...
}
```

`"ui.semanticTokens": true`を有効にするとtemplateのみならず、`Go`のソースコードが全体的にトークンの色の付け方が変わるので、びっくりするかもしれません。

### embed.FS, ParseFS

例示されるコードは以下でもホストされます。

https://github.com/ngicks/go-example-code-generation/tree/main/template/parse-fs

ディレクトリにtemplateを保存して丸ごとソースに埋め込みたいというケースはあると思いますが、`go:embed`と`template.ParseFS`によりそれが可能です。

例としてファイルを以下のように配置します。
前述の`gopls`の支援を受けるために拡張子は`.tmpl`にしてあります。

```
.template/
|-- tmp1.tmpl
|-- tmp2.tmpl
|-- tmp3.tmpl
`-- tmp4.tmpl
```

各templateの中身のは[multiple-template](#multiple-template)の同名ものとそれぞれ変わりませんが、以下のように名前だけ若干変わります。

```tmpl
sub1: {{template "tmp2.tmpl" .Sub1}}
sub2: {{template "tmp3.tmpl" .Sub2}}
sub3: {{template "tmp4.tmpl" .Sub3}}
{{block "additional" .}}{{end}}
```

`ParseFS`でファイルを読み込むと以下の行の挙動により`Base`が名前になってしまうためです。

https://github.com/golang/go/blob/go1.22.5/src/text/template/helper.go#L172-L178

`gopls`の支援により以下ようなsyntax highlightがかかります。

![tmpl-syntax-highlighting-by-gopls](/images/go-code-generation-in-ways-tmpl-syntax-highlighting-by-gopls.png)

`main.go`と同階層にこのtemplateディレクトリがあるものとして、以下のようなコードで読み込んで実行します。
事項結果自体は[multiple-template](#multiple-template)のものと変わりません。

ポイントとしては`//go:embed`でディレクトリを指定すると、そのディレクトリまでのパス構造がそのまま保たれます。つまり`//go:embed foo/bar/baz`とすると、`embed.FS`は`foo/bar/baz`というパス以下に`baz`ディレクトリの中身を埋め込みます。今回の場合この`templates` FSの直下に`template`ディレクトリがあってその中に各ファイルがある状態となります。
また、`fs.FS`のルールにより、`./template`は適切なパスではないので`template`で指定します([fs.ValidPath](https://pkg.go.dev/io/fs@go1.22.5#ValidPath))。

`template.ParseFS`の第二引数にvariadicな`patterns ...string`を渡すことができますが、それぞれが[fs.Glob](https://pkg.go.dev/io/fs@go1.22.5#Glob)に渡されるのため、[path.Match](https://pkg.go.dev/path@go1.22.5#Match)の条件を満たす必要があります。

```go
//go:embed template
var templates embed.FS

var (
	root = template.Must(template.ParseFS(templates, "template/*"))
)

type param struct {
	Tmp2, Tmp3, Tmp4 sub
}
type sub struct {
	Yay string
	Nay string
}

func main() {
	root = root.Lookup("tmp1.tmpl")
	data := param{
		Tmp2: sub{
			Yay: "yay2",
			Nay: "nay2",
		},
		Tmp3: sub{
			Yay: "yay3",
			Nay: "nay3",
		},
		Tmp4: sub{
			Yay: "yay4",
			Nay: "nay4",
		},
	}
	fmt.Println("---")
	err := root.Execute(os.Stdout, data)
	fmt.Println("---")
	fmt.Printf("err: %v\n", err)
	/*
		---
		tmp2: yay2
		tmp3: yay3
		tmp4: yay4

		---
		err: <nil>
	*/

	_, _ = root.New("additional").Parse(`{{.Tmp2.Yay}} and {{.Tmp3.Nay}}`)

	fmt.Println()
	fmt.Println("---")
	err = root.Execute(os.Stdout, data)
	fmt.Println("---")
	fmt.Printf("err: %v\n", err)
	/*
		---
		tmp2: yay2
		tmp3: yay3
		tmp4: yay4
		yay2 and nay3
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
	baseNameCutExt := func(p string) string {
		p, _ = strings.CutSuffix(path.Base(p), path.Ext(p))
		return p
	}
	for _, tmpl := range tmpls {
		if tmpl.IsDir() {
			continue
		}
		if extTrimmed == nil {
			extTrimmed = template.New(baseNameCutExt(tmpl.Name()))
		}
		bin, err := templates.ReadFile(path.Join("template", tmpl.Name()))
		if err != nil {
			panic(err)
		}
		_ = template.Must(extTrimmed.New(baseNameCutExt(tmpl.Name())).Parse(string(bin)))
	}
}
```

### text/template example: enum

例示されるコードは以下でもホストされます。

https://github.com/ngicks/go-example-code-generation/tree/main/template/go-enum

code generatorとしてかかわりそうな機能は一通り説明したと思います。このまま終わってもいいんですが、code generatorという立て付けで記事を作っているのですから最後にcode generatorのサンプルを示します。

以下のざっくり仕様を満たすものを作ることとします

- `type Foo string`な、string-base typeのみを生成します。
- `const (...)`でvariantsを列挙し、
- `IsFoo`で入力がvariantsかどうかを判定します。
- これだけだとつまらないので「特定のvariantsではない」という判定も作れるようにします(`IsFooExceptBar`)

このExceptの生成部分はサンプルにするために無理くり別のtemplateにくくりだしていますが、このぐらいのサイズなら1つのままにしておいたほうが読みやすいと思います。

ポイント的には

- `{{{pipeline}}`という感じで`{`の後にaction(`{{pipeline}}`)を実行したい場合`{{"{"}}{{pipeline}}`としないといけない
- ユーザーの入力文字列を`Go`の`ident`(identifier)として出力するには`Go` specを満たすようにエスケープが必要([identifier = letter { letter | unicode_digit }.](https://go.dev/ref/spec#Identifiers))
  - 以下のサンプルでは`unicode.IsLetter(r) || unicode.IsDigit(r) || r == '_'`でないとき`_`に置き換えるという少々雑な処理でごまかしていますが、実際には`u1234`という感じでunicode番号に置き換えるとかそういうことをしたほうが良いのだと思います

このサイズでも結構読むのはしんどいと思います。とはいえ機能が豊富で何でもできるのは便利ですね。

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

### ユーザーからtemplateを入力させるときのimportの取り扱い

この記事の話はここまでで終わりでもよかったんですが、欠点のところで「importの取り扱いが難しい」と正直に述べてしまったので、それへのアンサーとしてどのように処理すべきかの例を示します。

みなさんご存じの通り、`Go`のimportはimportされるpackageにアクセスするためのqualifierを`import "packagePath"`で定義し、`qualifier.ExportedIdentifier`で各要素にアクセスします。
当然qualifierはidentifierなので名前のかぶりを起こすとcompilation errorですし、`html/template`と`text/template`のように名前が同じ、かつ異なるパッケージは当然のように存在します。そのため、かぶりが起きたときにqualifier名を被らない何かにfallbackする仕組みが必要です。
また、`math/rand/v2`の`v2`のようなmajor versionはパッケージ名にならないのが普通で、この場合`rand`がパッケージ名になりますのでこれを考慮した処理も必要になります。

ユーザーが入力するtemplateが他のパッケージをimportするためには、単にtemplate textのみを入力とすると、非常に面倒な解析処理とテキスト置換処理が必要になります。
基本的にはimport package群も同様に入力させるほうがよいでしょう。

以下で、具体的な処理方法などの例示を行います。

#### Goのimport declの記法

`The Go Programming Language specification`によると、import declarationは

> https://go.dev/ref/spec#Import_declarations
>
> ImportDecl = "import" ( ImportSpec | "(" { ImportSpec ";" } ")" ) .
> ImportSpec = [ "." | PackageName ] ImportPath .
> ImportPath = string_lit .

です。

- `import "importPath"`である場合、`importPath`で指定されたパッケージが`package foobar`で宣言しているパッケージ名がqualifierとなります。
  - 大抵の場合、`importPath`の末尾の要素とパッケージ名は一致します。(e.g. `"foo/bar/baz"`ならば`baz`)
- `import packageName "importPath"`である場合、`packageName`がqualifierとなります。
- `import . "importPath"`である場合、qualifierなしで`importPath`のexported identifierにアクセスできます
- `import _ "importPath"`である場合、exported identifierにはアクセスできませんが、`importPath`の`init`などが実行され、importによる副作用のみを実行できます。

qualifierはidentifierであるため、当然名前は被ってはいけません。
一方で`.`, `_`はidentifierを定義しません。

#### ユーザー入力のフォーマット

上記の通り、ユーザーに入力させるtemplate textがほかのパッケージに依存する場合、import pathも同様に入力に持たせるほうがよいでしょう。
例えば

```go
type UserInput struct {
	// package name of generated code.
	PackageName string
	// Imports describes dependencies to other packages of Template text.
	// Template will not use packages other than described in this field.
	// Template may use all, some of, or even none of imported packages in the generated code.
	//
	// Imports maps the import path (key) to the template arg name (value).
	// Template refers to import qualifiers by template arg name(value) and
	// it expects values are provided under Imports key,
	// e.g. Template may describe imports by map[string]string{"bytes": "Bytes"} and refer to it as {{.Imports.Bytes}}.
	//
	// The values are also allowed to be `.` or `_`.
	// In those cases, the generated code will have dot or underscore imports
	// and Template will not receive those values.
	Imports map[string]string
	// template text
	Template string
}
```

という感じです。この構造体と相互に変換可能なJSONやYAMLなどで入力を受け付けることになるでしょう。
doc commentをそれなりに丁寧に書いていますが、例としては以下のような入力を想定します。

```go
UserInput{
	Imports: map[string]string{
		"fmt":           "Fmt",
		"math/rand/v2":  "MathRand",
	},
	Template: `func example() string {
	buf := make([]byte, 0, 16)
	for range 8 {
		buf = {{.Imports.Fmt}}.Appendf(buf, "%x", {{.Imports.MathRand}}.N[byte](255))
	}
	return string(buf)
}
`
}
```

`Imports`で、keyにpackage path, valueに`Template`中で参照できる変数名を指定させます。
正直key-valueは逆のほうがいい気もするんですが、同一のパッケージに複数のqualifierでアクセスしたいケースはかなり珍しいと思うのでシンプルさためにこうしています。
dot importは生成されたコードのほかの部分を壊しかねないため許容しないほうがいい気もするんですが、今回は単に例示なので許しています。

`Template`が`text/template`で解析/実行が可能なtemplate textです。

#### importPathからqualifierを取り出す

`html/template`と`text/template`,`crypto/rand`と`math/rand`のように、std範疇ですら同名の別パッケージが存在します。
こういった同名パッケージをインポートしたい場合、qualifierをそれぞれ別名にしてかぶりを起こさないようにする必要があります。
被りが起きるかどうかを検査するために、まずlexicalな解析でpackage pathからqualifierを取り出す処理を記述する必要があります。

ここで、`importPath`の末尾の要素(e.g. `"foo/bar/baz"`のとき`baz`)と、`importPath`が指し示すパッケージが`package foobar`で宣言するパッケージ名が一致していることを前提とします。
この前提が崩れると、ユーザーにパッケージ名も入力させなければlexicalな処理で事足りる範疇を超えてしまい、type checkのようなことが必要になってしまうためです。
このサンプルでは一致しているもの思い込みます: 一致しないのはディレクトリ名を書き換えたけど`package`宣言を修正し忘れているときだけだと思います。別にするメリットは基本ないはずですね。

大抵は`path.Base(packagePath)`でよいのです。
ただし`math/rand/v2`のように、major version suffixがある場合はこれを無視するのが一般的な`Go`のやり口なので、このケースを特別に処理する必要があります。

```go
func qualFromPkgPath(pkgPath string) string {
	base := path.Base(pkgPath)
	if base == pkgPath {
		// contains no `/`
		return pkgPath
	}
	majorVersion, has := strings.CutPrefix(base, "v")
	if !has {
		// no major version.
		return base
	}
	if len(strings.TrimLeftFunc(majorVersion, func(r rune) bool {
		return '0' <= r && r <= '9'
	})) == 0 {
		// suffix is major version
		return path.Base(path.Dir(pkgPath))
	}
	return base
}
```

#### import specを生成する

ユーザーによってimportも入力されるため、メインとなるtemplateのimport decl部分はもはやあらかじめ書いておくことができなくなります。
そのため以下のように何かのパラメータをrangeする必要があります。

```tmpl: pkg.tmpl
// Code generated by me. DO NOT EDIT.
package {{.PackageName}}

import (
{{range .Imports}}	{{if .Name}}{{.Name}} {{end}}{{quote .PkgPath}}
{{end -}}
)

// ...rest of code...
```

importはqual nameとpackage pathから構成されるため、上記パラメータは以下のように定義できます。

```go
type ImportSpec struct {
	// Name is the import qualifier name. Maybe empty.
	// If empty, the qual must be lexically inferred from PkgPath.
	Name    string
	PkgPath string
}

type TemplateParam struct {
	Imports         []ImportSpec
}
```

前述のユーザー入力と、ユーザー入力でないtemplateが元からimportするpackageを組み合わせて`[]ImportSpec`を生成するには以下のようにします。

ポイントとしては、

- ユーザー入力の`map[string]string`をfor-range-mapしないようにする
  - してしまうと実行のたびに順序が異なるため
- qual名が被るがpackage pathが異なる場合、`_数字`でsuffixして被らなくします。

引数の`preDeclared`はユーザー入力でないtemplate部分のimport specs、`userImports`は`UserInput`の`Imports`です。

```go
func makeImportSpecs(preDeclared []ImportSpec, userImports map[string]string) []ImportSpec {
	importSpecs := slices.Clone(preDeclared)

	// maps qualifier name to package path.
	qualToPkgPath := make(map[string]string, len(importSpecs)+len(userImports))

	for _, spec := range importSpecs {
		if spec.Name == "." || spec.Name == "_" {
			continue
		}
		name := spec.Name
		if name == "" {
			name = qualFromPkgPath(spec.PkgPath)
		}
		qualToPkgPath[name] = spec.PkgPath
	}

	userPackagePaths := make([]string, 0, len(userImports))
	for k := range userImports {
		userPackagePaths = append(userPackagePaths, k)
	}
	slices.Sort(userPackagePaths)
USER_PKG:
	for _, pkgPath := range userPackagePaths {
		arg := userImports[pkgPath]
		switch arg {
		case ".", "_":
			importSpecs = append(importSpecs, ImportSpec{arg, pkgPath})
		default:
			name := qualFromPkgPath(pkgPath)
			org := name
			fallenBack := false
			for i := 0; ; i++ {
				knownPkgPath, has := qualToPkgPath[name]
				if knownPkgPath == pkgPath {
					continue USER_PKG
				}
				if !has {
					qualToPkgPath[name] = pkgPath
					break
				}
				fallenBack = true
				name = org + "_" + strconv.FormatInt(int64(i), 10)
			}
			if !fallenBack {
				name = ""
			}
			importSpecs = append(importSpecs, ImportSpec{name, pkgPath})
		}
	}

	slices.SortFunc(importSpecs, func(i, j ImportSpec) int {
		return strings.Compare(i.PkgPath, j.PkgPath)
	})

	importSpecs = slices.CompactFunc(importSpecs, func(i, j ImportSpec) bool {
		return i.Name == j.Name && i.PkgPath == j.PkgPath
	})

	return importSpecs
}
```

#### ユーザーのtemplateに渡すパラメータを生成する

前述のとおり、`Imports`のvalueの値でpackage qualifierにアクセスする仕様にしたため、ユーザーtemplateに渡されるパラメータも生成する必要があります。

```go
UserInput{
	Imports: map[string]string{
		"fmt":           "Fmt",
		"math/rand/v2":  "MathRand",
	},
	Template: `func example() string {
	buf := make([]byte, 0, 16)
	for range 8 {
		buf = {{.Imports.Fmt}}.Appendf(buf, "%x", {{.Imports.MathRand}}.N[byte](255))
	}
	return string(buf)
}
`
}
```

以下のパラメータをユーザーのtemplateに渡します。

```go
type UserTemplateArg struct {
	// Imports maps arg name to package qualifier.
	Imports map[string]string
}
```

以下のように変換します。引数の`specs`は前述のimport spec作成部分(`makeImportSpecs`)の返り値、`userImports`は`UserInput`の`Imports`です。

```go
func makeUserImportArg(specs []ImportSpec, userImports map[string]string) map[string]string {
	pkgNames := make(map[string][]string)
	for _, spec := range specs {
		if spec.Name == "." || spec.Name == "_" {
			continue
		}
		name := spec.Name
		if name == "" {
			name = qualFromPkgPath(spec.PkgPath)
		}
		pkgNames[spec.PkgPath] = append(pkgNames[spec.PkgPath], name)
	}

	userImportArg := make(map[string]string)
	for pkgPath, arg := range userImports {
		if arg == "." || arg == "_" {
			continue
		}
		userImportArg[arg] = pkgNames[pkgPath][0]
	}

	return userImportArg
}
```

複数のqualが同じpackage pathにアクセスすることがあり得るものとしてこういう処理になっていますが実際今回組んだサンプルでは1-1関係が保たれますので無用なオーバーヘッドです。

#### 実行

完成したexampleは以下でもホストされます

https://github.com/ngicks/go-example-code-generation/tree/main/template/handle-imports

以下のように実行します。特に解説していませんが、`goimports`によるフォーマットをかけてからファイルに出力するようにしています。

```go
package main

import (
	"bytes"
	"context"
	"errors"
	"fmt"
	"io"
	"io/fs"
	"os"
	"os/exec"
	"path"
	"path/filepath"
	"slices"
	"strconv"
	"strings"
	"text/template"
	"unicode"
)

var funcs = template.FuncMap{
	"quote": func(s string) string {
		return strconv.Quote(s)
	},
}

var pkg = template.Must(template.New("pkg").
	Funcs(funcs).
	Parse(
		`// Code generated by me. DO NOT EDIT.
package {{.PackageName}}

import (
{{range .Imports}}	{{if .Name}}{{.Name}} {{end}}{{quote .PkgPath}}
{{end -}}
)

var bufPool = &sync.Pool{
	New: func() any {
		return new(bytes.Buffer)
	},
}

func getBuf() *bytes.Buffer {
	return bufPool.Get().(*bytes.Buffer)
}

func putBuf(b *bytes.Buffer) {
	if b == nil || b.Cap() > 64<<10 {
		return
	}
	b.Reset()
	bufPool.Put(b)
}

{{block "user-input" .UserTemplateArg}}{{end}}
`))

type UserInput struct {
	// ...
}

type TemplateParam struct {
	// ...
}

type ImportSpec struct {
	// ...
}

type UserTemplateArg struct {
	// ...
}

func main() {
	err := checkGoimports()
	if err != nil {
		panic(err)
	}

	targetDir := filepath.Join("template", "handle-imports", "target")
	err = os.Mkdir(targetDir, fs.ModePerm)
	if err != nil && !errors.Is(err, fs.ErrExist) {
		panic(err)
	}

	userInput := UserInput{
		PackageName: "main",
		Imports: map[string]string{
			"bytes":         "Bytes",
			"crypto":        "Crypto",
			"crypto/rand":   "CryptoRand",
			"crypto/sha256": "_",
			"crypto/sha512": "_",
			"encoding/hex":  "Hex",
			"fmt":           ".",
			"io":            "Io",
			"math/rand/v2":  "MathRand",
		},
		Template: `func main() {
	randBuf := getBuf()
	defer putBuf(randBuf)

	var err error

	_, err = {{.Imports.Io}}.CopyN(randBuf, {{.Imports.CryptoRand}}.Reader, 16)
	if err != nil {
		panic(err)
	}
	for i := 0; i < 16; i++ {
		_ = randBuf.WriteByte({{.Imports.MathRand}}.N(byte(255)))
	}

	_, _ = Printf("rand bytes=%q\n", {{.Imports.Hex}}.EncodeToString(randBuf.Bytes()))

	h := {{.Imports.Crypto}}.SHA256.New()
	_, err = {{.Imports.Io}}.Copy(h, {{.Imports.Bytes}}.NewReader(randBuf.Bytes()))
	if err != nil {
		panic(err)
	}
	_, _ = Printf("sha256sum=%q\n", {{.Imports.Hex}}.EncodeToString(h.Sum(nil)))

	h = {{.Imports.Crypto}}.SHA512.New()
	_, err = {{.Imports.Io}}.Copy(h, {{.Imports.Bytes}}.NewReader(randBuf.Bytes()))
	if err != nil {
		panic(err)
	}
	_, _ = Printf("sha512sum=%q\n", {{.Imports.Hex}}.EncodeToString(h.Sum(nil)))
}
`,
	}

	_, err = pkg.New("user-input").Parse(userInput.Template)
	if err != nil {
		panic(err)
	}

	var buf bytes.Buffer
	specs := makeImportSpecs([]ImportSpec{{"", "bytes"}, {"", "sync"}}, userInput.Imports)
	err = pkg.Execute(&buf, TemplateParam{
		PackageName: userInput.PackageName,
		Imports:     specs,
		UserTemplateArg: UserTemplateArg{
			Imports: makeUserImportArg(specs, userInput.Imports),
		},
	})
	if err != nil {
		panic(err)
	}

	formatted, err := applyGoimports(context.Background(), &buf)
	if err != nil {
		panic(err)
	}

	targetFile := filepath.Join(targetDir, "main.go")
	f, err := os.Create(targetFile)
	if err != nil {
		panic(err)
	}
	defer f.Close()
	_, err = io.Copy(f, formatted)
	if err != nil {
		panic(err)
	}
}

func qualFromPkgPath(pkgPath string) string {
	// ...
}

func makeImportSpecs(preDeclared []ImportSpec, userImports map[string]string) []ImportSpec {
	// ...
}

func makeUserImportArg(specs []ImportSpec, userImports map[string]string) map[string]string {
	// ...
}

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

以下が生成されます。

```go
// Code generated by me. DO NOT EDIT.
package main

import (
	"bytes"
	"crypto"
	"crypto/rand"
	_ "crypto/sha256"
	_ "crypto/sha512"
	"encoding/hex"
	. "fmt"
	"io"
	rand_0 "math/rand/v2"
	"sync"
)

var bufPool = &sync.Pool{
	New: func() any {
		return new(bytes.Buffer)
	},
}

func getBuf() *bytes.Buffer {
	return bufPool.Get().(*bytes.Buffer)
}

func putBuf(b *bytes.Buffer) {
	if b == nil || b.Cap() > 64<<10 {
		return
	}
	b.Reset()
	bufPool.Put(b)
}

func main() {
	randBuf := getBuf()
	defer putBuf(randBuf)

	var err error

	_, err = io.CopyN(randBuf, rand.Reader, 16)
	if err != nil {
		panic(err)
	}
	for i := 0; i < 16; i++ {
		_ = randBuf.WriteByte(rand_0.N(byte(255)))
	}

	_, _ = Printf("rand bytes=%q\n", hex.EncodeToString(randBuf.Bytes()))

	h := crypto.SHA256.New()
	_, err = io.Copy(h, bytes.NewReader(randBuf.Bytes()))
	if err != nil {
		panic(err)
	}
	_, _ = Printf("sha256sum=%q\n", hex.EncodeToString(h.Sum(nil)))

	h = crypto.SHA512.New()
	_, err = io.Copy(h, bytes.NewReader(randBuf.Bytes()))
	if err != nil {
		panic(err)
	}
	_, _ = Printf("sha512sum=%q\n", hex.EncodeToString(h.Sum(nil)))
}
```

もちろん正しく動作します。

```
...# go run ./template/handle-imports/target/
rand bytes="5c65391ca536b23733d81e0c67d2f9ca1c183d0f028a339fbeba68c6f0bf2d16"
sha256sum="bb863290d8f0699dd9d7feb16c2bf44b340ba98ada8d37f3401538d82dbe70cf"
sha512sum="5f06276c8c00bb1bab175d2c1f3f92332a3383bd7bf2f8f550f59cf69a8d1af6cddaf5fc005d01d5bade14b4bd618019501deffbbfe92b9e62979226ebe80f21"
```

## おわりに

この記事では

- なぜcode generatorが必要なのか
- 一連の記事で述べることになる、代表的と思われる4つのcode generatorの実装方法についての概説
- code generatorの諸注意などについて
- `io.Writer`にテキストを書き出すだけのシンプルなcode generator
- `text/template`の使い方
- `text/template`を用いるcode generator

を述べました。

さらに後続の記事で、それぞれ以下について説明します。

- [Goのcode generation: jennifer](https://zenn.dev/ngicks/articles/go-code-generation-in-ways-jennifer)で[github.com/dave/jennifer]を用いる方法
- [Goのcode generation: ast(dst)-rewrite](https://zenn.dev/ngicks/articles/go-code-generation-in-way-ast-dst)で[astutil](https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/ast/astutil)および[github.com/dave/dst]を用いる方法

`text/template`は機能が豊富で柔軟にコード生成できますが、`Go`のsource codeを生成するための専用というわけではないので可読性を保ちながら記述するのに苦労します。
記事の最後のほうで説明した通り、importの取り扱いは結構面倒でいろいろな落とし穴が存在しえますね。
ただし、この一連の記事が説明する方法はそれぞれ組み合わせてよいので、用途に合わせて使い分け、組み合わせるのがよいでしょう。特にimport周りは`jennifer`はうまく取り扱ってくれます。
`text/template`はtemplate textをutf-8のテキストとして持ち回るため、ユーザーからの入力を受け付けて挙動をカスタマイズさせたい場合に特に便利だと思います。

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
