---
title: "gotipでencoding/json/v2を試す"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## gotipでencoding/json/v2を試す

長いこと[discussion](https://github.com/golang/go/discussions/63397)にあった`encoding/json/v2`ですが2025-01-31に以下のproposalに移行しました。

https://github.com/golang/go/issues/71497

[proposal]: https://github.com/golang/go/issues/71497

2025-04-15に`GOEXPERIMENT=jsonv2`下で利用できるようになるCLがマージされました。

https://go-review.googlesource.com/c/go/+/665796

このままいけば`Go 1.25`で`GOEXPERIMENT`付きで試せるようになり、問題なければ`Go 1.26`で実装という感じでしょうか。

筆者は以下の記事などで何度か`encoding/json/v2`について触れていますが

https://zenn.dev/ngicks/articles/go-json-undefined-or-null-v2

何が変わったかなどをついてこの記事で述べていきたいと思います。

## サンプル

以下に上がります。

https://github.com/ngicks/go-play-encoding-json-v2

## 環境

```
$ go version
go version devel go1.25-0e17905793 Fri Apr 18 08:24:07 2025 -0700 linux/amd64
```

## gotipで開発できるようにする

```
go install golang.org/dl/gotip@latest
go download 665796
export PATH=$(gotip env GOROOT)/bin/:$PATH
export GOEXPERIMENT=jsonv2
```

[gotip]を入れて前述のCLをダウンロードします。マージされていますので別にmasterの先頭を入れてもいいと思います。

`PATH`をいじって`go`コマンドが`gotip`のものになるようにします。
[Go 1.21]から[GOTOOLCHAIN](https://go.dev/doc/toolchain)という概念が追加されています。
それまでの`Go`は`go`コマンドのバージョンが今コンパイルの対象となっている`Go module`のそれより低かろうがコンパイルを試みるような挙動だったようですが、
gotoolchain追加後は現在呼び出された`go`コマンドよりも`go.mod`の内容が新しければ自動的にtoolchainをダウンロードする挙動になっています。

ここで不都合なのが`gotip`で最新を落として`gotip mod init`すると`go.mod`には最新の次のバージョンで記載されます。つまり今回だと`go 1.25`です。
このtoolchainは現在存在しませんからダウンロードしようとするところでエラーになります。

最後に`GOEXPERIMENT`を設定しておきます。

## 経緯

詳しくは前述の[discussion](https://github.com/golang/go/discussions/63397)に書かれているのでそこを読んでほしいのですが、要点だけを述べると

`encoding/json`にはいろいろ問題がありました。

- 機能が欠けている
  - `time.Time`のフォーマットを指定できない
  - 特定の値をomitできない(e.g. zero valueの`time.Time`)
  - slice(`[]T`)やmap(`map[K]V`)がnilのとき必ず`null`が出力され、`[]`や`{}`を出力する方法がない
  - 型をembedする以外に値をinlineする方法がない
  - strcutなどに向けてunmarshalするとき、想定されないフィールドが含まれるときにそれらをどこかにfallbackする機能がない
  - etc, etc.
- APIが変
  - `json.NewDecoder(r).Decode(v)`がよくされるが、`r io.Reader`から1つのJSON Valueと取り出すだけでそのあとにゴミデータがあった場合などにエラーにならない。
  - `json.Compact`, `json.Indent`, `json.HTMLEscape`などが`bytes.Buffer`を使っていること。より柔軟な`[]byte`, `io.Writer`などを使わないこと。
- パフォーマンスが悪い
  - `MarshalJSON`のinterfaceが`[]byte`を返すものであり、毎回allocationが要求される
  - `UnmarshalJSON`のinterfaceが`[]byte`を受け取るものであり、`encoding/json`はJSON Valueの終端までいったん解析し、`json.Compact`をかけてから渡していた。また、`UnmarshalJSON`の実装側でももう1度JSON Valueの解析が行われることになる。
  - `json.MarshalIndent`を呼び出すとindentのついていないjsonを書きだした後に再度解析を行ってindentを挿入するような処理になっている。
- streaming APIが存在しない。
  - `Encoder.EncodeToken` proposalは通過したが、実装はされていない([#40127](https://github.com/golang/go/issues/40127))
  - `json.Token`はinterfaceであるため、JSON numberやstringをbox化するときにallocateしてしまう。
  - 理屈上最も大きなJSON Tokenのみバッファーすればよいはずだが、`Encoder.Encode`および`Decoder.Decode`はJSON Value単位でバッファーしまてしまう。
- 挙動が変
  - 準拠しているRFCが古い。下記の時間とともに以下のようにRFCが更新されて行っている。`encoding/json`が準拠するのは`RFC 7159`。`v2`が準拠するのは`RFC 8259`。
    - [RFC 4627](https://tools.ietf.org/html/rfc4627)(2006-07)
    - [RFC 7159](https://tools.ietf.org/html/rfc7159)(2014-03)
    - [RFC 7493](https://tools.ietf.org/html/rfc7493)(2015-03)
    - [RFC 8259](https://tools.ietf.org/html/rfc8259)(2017-12)
    - [Go v1のリリース日は2012-03-28](https://go.dev/doc/devel/release#go1)
    - (顕著な変更は無効な文字、例えばペアになっていないsurrogate pairとかが許されていたことです。)
  - array(`[n]T`)の長さが不一致でもunmarshalが成功する
  - `json:",string"`オプションが数値以外にも適用される、また、`[]int`などにrecursiveに適用されない。
  - Unmarshal時のGo structフィールド名とJSON objectのキー名とのマッチングがでcase-insensitive。
    - `json:"name"`で指定していたとしてもcase-insensitiveでした。
  - underlying typeがnon-addressableである型の`MarshalJSON` / `UnmarshalJSON`が呼ばれない。
    - method receiverがpointerで`json.Marshal`に渡された値がnon-pointerだった時のことをさしています。
  - unmarshalのターゲットにnon-zeroな値を渡したときの挙動に一貫性がない
    - sliceはunmarshal後の長さが短くなる場合、`s[len(s):cap(s)]`の区間はzero化されていませんでした。
  - エラーに一貫性がなく、io error, syntax error, semantic errorが構造化されずにごっちゃに返されている。

破壊的変更を避けながらこれらを解消することはできるかもしれないが、デフォルトの挙動を[RFC 8259](https://tools.ietf.org/html/rfc8259)に準拠させるなどしたほうが良いと思われるので`v2`として破壊的変更を加えよう、みたいな感じです。

## 顕著な変更

[proposal]に貼られていますが、以下のように構造が変化します。

![](https://raw.githubusercontent.com/go-json-experiment/json/6e475c84a2bf3c304682aef375e000771a318a5c/api.png)

`encoding/json/jsontext`を追加し、ここでJSONの文法を処理します。
`encoding/json/v2`を追加し、ここで`jsontext`で処理されたトークン情報を用いて、`v1`と同様に`reflect`などを使用して`Go`の値との相互的なやり取りを実現します。

[json.Marshal], [json.Unmarshal]とほぼ同等なものに加えて、`io.Reader`/`io.Writer`, `*jsontext.Encoder`/`*jsontext.Decoder`を引数に取る新しいAPIが追加されます。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal.go#L167

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal.go#L183

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal.go#L203

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal.go#L398

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal.go#L415

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal.go#L448

見てのとおり、`Options`という型でoptionを受け付けられるようになっており、ユーザー側で柔軟な挙動の変更が可能です。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/internal/jsonopts/options.go#L7-L18

`Options`はinterfaceですが上記のとおり現状`encoding/json`以下のパッケージからしか定義ができないようになっています。

[json.Marshaler], [json.Unmarshaler]加えて以下の`MarshalerTo`と`UnmarshalerFrom`が追加されます。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal_methods.go#L48-L62

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal_methods.go#L79-L96

これらを型に実装させれば、前述の経緯のところで述べた[json.Unmarshaler]使用時の極端なパフォーマンス劣化を避けることができます。

## 使ってみる

### jsontext.Encoder

`WriteToken`、`WriteValue`で値を書き込みますが、内部のステートマシンが状態を覚えているので`:`とか`,`とかを手動で書き込む必要はないです。これはいいデザインですね。

`StackDepth`で現在のJSON ObjectやJSON Arrayのnest回数がわかります。これは自分で実装すると面倒なので助かります。

`UnusedBuffer`で`encoderState`に紐づくバッファーが利用できるので、これを利用するとよいというAPIのようです。内部のコメントを見ると`encoderState`のバッファーの未使用の部分をsliceで返すような実装をしていたけどやめたようなことがコメントで書かれています。見た限りずっとこのコメントが残されています。proposalになる時点でもこのmethod名が変わらなかったので実装されるときもこのままかもしれないですね。

`StackIndex`で現在の`StackDepth`までの間の階層の開始tokenがわかります。これは`0`, `{`, `[`のどれかになります。親が`{`のときのみ～みたいな条件の違うエンコーディングが必要な時に用いるんでしょうか。

```go
import (
	"bytes"
	"encoding/json/jsontext"
	"testing"
)

func TestEncoder(t *testing.T) {
	buf := new(bytes.Buffer)
	enc := jsontext.NewEncoder(buf, jsontext.WithIndent("    "))

	var err error
	bufErr := func(e error) {
		if err != nil {
			return
		}
		err = e
	}

	assertDepth := func(enc *jsontext.Encoder, depth int) {
		if enc.StackDepth() != depth {
			t.Errorf("wrong depth: expected = %d, actual = %d", depth, enc.StackDepth())
		}
	}

	assertDepth(enc, 0)
	bufErr(enc.WriteToken(jsontext.BeginObject))
	assertDepth(enc, 1)

	bufErr(enc.WriteToken(jsontext.String("foo")))
	bufErr(enc.WriteToken(jsontext.Null))
	bufErr(enc.WriteToken(jsontext.String("baz")))

	bufErr(enc.WriteToken(jsontext.BeginObject))
	assertDepth(enc, 2)

	bufErr(enc.WriteToken(jsontext.String("qux")))
	bufErr(enc.WriteToken(jsontext.Int(123)))
	bufErr(enc.WriteToken(jsontext.String("quux")))
	if enc.OutputOffset() == int64(buf.Len()) {
		t.Errorf("immediately flushed at %d", enc.OutputOffset())
	}
	v := enc.UnusedBuffer()
	v = append(v, []byte(`[`)...)
	v = append(v, []byte(`{"corge":null}`)...)
	v = append(v, []byte(`]`)...)
	assertDepth(enc, 2)
	bufErr(enc.WriteValue(v))
	assertDepth(enc, 2)

	t.Log(enc.StackIndex(0)) // encoder_test.go:52: <invalid jsontext.Kind: '\x00'> 1
	t.Log(enc.StackIndex(1)) // encoder_test.go:53: { 4
	t.Log(enc.StackIndex(2)) // encoder_test.go:54: { 4

	bufErr(enc.WriteToken(jsontext.EndObject))
	assertDepth(enc, 1)
	bufErr(enc.WriteToken(jsontext.EndObject))
	assertDepth(enc, 0)

	if err != nil {
		panic(err)
	}
	expected := `{
    "foo": null,
    "baz": {
        "qux": 123,
        "quux": [
            {
                "corge": null
            }
        ]
    }
}
`
	if buf.String() != expected {
		t.Fatalf("not equal:\nexpected = %s\nactual  = %s", expected, buf.String())
	}
}
```

### jsontext.Decoder

`PeekKind`で値を消費せずに`jsontext.Kind`を取得し、`ReadToken`, `ReadValue`で値を読み込みます。

`jsontext.Encoder`同様`StackDepth`で現在のJSON ObjectやJSON Arrayのnest回数が取得できます。

`jsontext.Encoder`と同様に`StackIndex`で現在の`StackDepth`までの間の階層の開始tokenがわかります。これは`0`, `{`, `[`のどれかになります。

`StackPointer`でJSON Pointer ([RFC 6901](https://datatracker.ietf.org/doc/html/rfc6901))が取得できます。decodeでエラー発生時にこれをエラーテキストに含ませるとわかりやすくていいかもしれません。

<!-- other languages referenced -->

[Java]: https://www.java.com/
[TypeScript]: https://www.typescriptlang.org/
[python]: https://www.python.org/
[C]: https://www.c-language.org/
[C++]: https://isocpp.org/
[Rust]: https://www.rust-lang.org
[The Rust Programming Language 日本語]: https://doc.rust-jp.rs/book-ja/
[Lua]: https://www.lua.org/

<!-- other lib/SDKs referenced -->

[Node.js]: https://nodejs.org/en
[deno]: https://deno.com/
[tokio]: https://tokio.rs/

<!-- editors -->

[Visual Studio Code]: https://code.visualstudio.com/
[vscode]: https://code.visualstudio.com/
[neovim]: https://neovim.io/

<!-- tools -->

[git]: https://git-scm.com/
[Git Credential Manager]: https://github.com/git-ecosystem/git-credential-manager?tab=readme-ov-file
[Docker]: https://www.docker.com/
[Dockerfile]: https://docs.docker.com/build/concepts/dockerfile/
[Elasticsearch]: https://www.elastic.co/docs/solutions/search

<!-- Go versions -->

[Go]: https://go.dev/
[Go 1.11]: https://go.dev/doc/go1.11
[Go 1.14]: https://go.dev/doc/go1.14
[Go 1.18]: https://go.dev/doc/go1.18
[Go 1.21]: https://go.dev/doc/go1.21
[Go 1.22]: https://go.dev/doc/go1.22
[Go 1.23]: https://go.dev/doc/go1.23
[Go 1.24]: https://go.dev/doc/go1.24

<!-- Go doc links -->

[A Tour of Go]: https://go.dev/tour/welcome/
[GOAUTH]: https://pkg.go.dev/cmd/go#hdr-GOAUTH_environment_variable

<!-- Go tools -->

[gotip]: https://pkg.go.dev/golang.org/dl/gotip
[gopls]: https://github.com/golang/tools/tree/master/gopls
[github.com/go-task/task]: https://github.com/go-task/task

<!-- references to spec -->

[type assertion]: https://go.dev/ref/spec#Type_assertions
[type switch]: https://go.dev/ref/spec#Type_switches

<!-- references to sdk library -->

[panic]: https://pkg.go.dev/builtin@go1.24.3#panic
[json.Marshal]: https://pkg.go.dev/encoding/json@go1.24.3#Marshal
[json.Marshaler]: https://pkg.go.dev/encoding/json@go1.24.3#Marshaler
[json.Unmarshaler]: https://pkg.go.dev/encoding/json@go1.24.3#Unmarshaler
[json.Unmarshal]: https://pkg.go.dev/encoding/json@go1.24.3#Unmarshal
[errors.New]: https://pkg.go.dev/errors@go1.24.3#New
[errors.Is]: https://pkg.go.dev/errors@go1.24.3#Is
[errors.As]: https://pkg.go.dev/errors@go1.24.3#As
[errors.Join]: https://pkg.go.dev/errors@go1.24.3#Join
[fmt.Errorf]: https://pkg.go.dev/fmt@go1.24.3#Errorf
[fs.ErrNotExist]: https://pkg.go.dev/io/fs@go1.24.3#ErrNotExist
[http.Server]: https://pkg.go.dev/net/http@go1.24.3#Server
[*http.Server]: https://pkg.go.dev/net/http@go1.24.3#Server
[io.EOF]: https://pkg.go.dev/io@go1.24.3#EOF
[io.Reader]: https://pkg.go.dev/io@go1.24.3#Reader
[io.Writer]: https://pkg.go.dev/io@go1.24.3#Writer
[syscall.Errno]: https://pkg.go.dev/syscall@go1.24.3#Errno
[text/template]: https://pkg.go.dev/text/template@go1.24.3
