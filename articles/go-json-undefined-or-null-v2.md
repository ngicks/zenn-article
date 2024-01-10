---
title: "encoding/json v2(候補)について紹介してundefined | null | Tを表現する"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

# はじめに

以前書いた[Goのstruct fieldでJSONのundefinedとnullを表現する](https://zenn.dev/ngicks/articles/go-json-that-can-be-t-null-or-undefined)では、[jsoniter](https://github.com/json-iterator/go)の [Extension](https://pkg.go.dev/github.com/json-iterator/go#Extension) を駆使していろいろ頑張ることで`undefined | null | T`を出し分けることが実現できることを確認しました。記事中では同様に、`encoding/json/v2`でstdライブラリとして同様のことができるようになるかもしれないということも触れました。

先日(`2023-10-05T17:14:54Z`)、記事内で触れた[issue comment](https://github.com/golang/go/issues/5901#issuecomment-907696904)の筆者が[encoding/json/v2](https://github.com/golang/go/discussions/63397)というタイトルのdiscussionを作りました。
`v2`はproposalに近づいてきました。

そこで、この`encoding/json/v2`の候補版の実装を用いて以前の記事と同じようなことを実現します。

# やること

この記事は以下を行います。

- `encoding/json/v2`のモチベーションを紹介する
- `encoding/json/v2`の候補実装である[github.com/go-json-experiment/json](https://github.com/go-json-experiment/json)(以降`v2`と呼ばれる)の実装やAPI構造を紹介する
- `v2`でstruct fieldのみで`undefined | null | T`を実現する

# 想定読者

- [Go programming language](https://go.dev/) で [encoding/json](https://pkg.go.dev/encoding/json)の機能を十分理解している人。
- `encoding/json/v2`の候補版実装について興味がある人

# encoding/json/v2

## モチベーション

- discussion: [encoding/json/v2](https://github.com/golang/go/discussions/63397)

ざっくりdiscussionの中で述べられるモチベーションを列挙します。かっこ(`()`)で囲まれた部分は筆者の注釈です。

`v2`で解決したい`v1`の欠点を以下の４つの観点で述べる

- Missing Functionality
  - `time.Time`のフォーマットを指定できない
  - 特定の値をomitできない
    - (e.g. `time.Time`のzero valueを`omitempty`でomitしたいのに`struct`には決して`omitempty`が機能しない)
  - `slice`や`map`が`nil`であるとき空の`Array`(`[]`), `Object`(`{}`)を出力できない
  - `embed`以外の方法で出力結果に`Go type`をinlineできないこと
    - (i.e. Go structで定義したfield以外は全部`map[string]any`に詰めるようなことができない)
- API deficiencies
  - `io.Reader`からうまくjsonをdecodeする方法がない
    - `json.NewDecoder(r).Decode(v)`がよくされるがこれは誤りである: `Decode`は1つの有効なJSON valueだけを取り出すので、末尾にゴミデータがある場合にエラーにならない。
      - (エラーになってほしい場合、`Decode`を呼んだ後に`dec.More()`が`true`を返すときエラーを返すような処理をユーザーが書く必要があります)
  - `Decoder`, `Encoder`にOptionを設定する方法があるが([SetEscapeHTML](https://pkg.go.dev/encoding/json#Encoder.SetEscapeHTML)とか[DisallowUnknownFields](https://pkg.go.dev/encoding/json#Decoder.DisallowUnknownFields)とかのこと)、`json.Marhsal`, `json.Unmarshal`にはない
  - `json.Compact`, `json.Indent`, `json.HTMLEscape`などが`bytes.Buffer`を使っていること。より柔軟な`[]byte`, `io.Writer`などを使わないこと。
    - (これは筆者も大分不思議に思っていました。筆者の使い方的には困らないので別によかったのですが、`[]byte`や`io.Writer`を使うAPIのほうが驚きは少ないですね)
- Performance limitations
  - `MarshalJSON`メソッドが返り値の`[]byte`をallocateすることを強制してしまう。同様に、このsemanticsは`json`パッケージに返り値のバリデーションと、インデント付けの両方で解析を必要としてしまう。
  - `UnmarshalJSON`メソッドが一つの完全な、かつ、末尾にデータのないJSON valueを必要とする。`json`パッケージは`UnmarshalJSON`を呼び出す前に1つのJSON valueが終わるまで解析が必要であり、さらに`UnmarshalJSON`の実装でJSON valueを解析しなおす必要がある。`UnmarshalJSON`の呼び出しが再帰してしまうと劇的なパフォーマンス低下をもたらす。
  - streaming encoder APIが存在しない。`Encoder.EncodeToken` proposalは通過したが、実装はされていない([#40127](https://github.com/golang/go/issues/40127))
  - `json.Token`はinterfaceであるため、JSON numberやstringをbox化するときにallocateしてしまう。
  - steaming APIが存在しない: 理屈上最も大きなJSON Tokenのみバッファーすればよいはずだが、`Encoder.Encode`および`Decoder.Decode`はJSON全体をバッファーしてしまう。
- Behavioral flaws
  - 時間とともにJSONに関する標準は増えている ([RFC 4627](https://tools.ietf.org/html/rfc4627), [RFC 7159](https://tools.ietf.org/html/rfc7159), [RFC 7493](https://tools.ietf.org/html/rfc7493), [RFC 8259](https://tools.ietf.org/html/rfc8259))。一般的に言って、これらのRFCは徐々により厳密な定義になっていっているが、`encoding/json`は新しいRFCに適合していない。例えば、[RFC 8259](https://tools.ietf.org/html/rfc8259)ではinvalidなUTF-8のシーケンスを許さないが、`encoding/json`はそれらを許容する。
    - ([Go v1のリリース日は2012-03-28](https://go.dev/doc/devel/release#go1)であり、[RFC 7159](https://tools.ietf.org/html/rfc7159)が2014-03、, [RFC 7493](https://tools.ietf.org/html/rfc7493)は2015-03, [RFC 8259](https://tools.ietf.org/html/rfc8259)は2017-12です)
  - Unmarshal時のGo struct fieldとJSON objectのキー名とのマッチングがでcase-insensitiveである。
  - underlying typeがnon-addressableである型の`MarshalJSON` / `UnmarshalJSON`が呼ばれない。しかし、この挙動を修正することもbreaking changeである。
  - `json.Unmarshal(data, &v)`のターゲット(=`v`)がzero valueでない時に起きるマージの挙動に一貫性がない: 例えばnon-zeroなsliceを使用した場合、Unmarshal後のsliceのlengthとcapacityの間の値はゼロ化されずにマージされる。
  - 一貫しないエラー値: 現在の`encoding/json`の返すエラーは構造化されている部分とされていない部分があり一貫しない。実際には3つのクラスのエラーが起きるはずである: 文法エラー、意味論エラー、I/Oエラー。

Behavioral flawsの部分は破壊的変更なしに修正できないし、`json`パッケージにオプションという形で実装することはできるが、望ましい挙動がデフォルトでないことは不幸なことである。
デフォルトの挙動を変える必要があることから`v2`の必要性を示唆する。

(キー名とのマッチングがcase-insensitiveなのかなり驚きました。
[diff_test.go](https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/diff_test.go)を見ると`json:"name"`で名前を明確に指定していたとしてもcase-insensitiveなんですね。知らなかった。すごい驚きです。)

## 実装

- implementation: [github.com/go-json-experiment/json](https://github.com/go-json-experiment/json)

この記事では実装はすべて以下のコミットの状態で確認されています。

```
Commit: 2e55bd4e08b08427ba10066e9617338e1f113c53
Parents: 54c864be5b8da112b1492c72087512969f2fdea4
Author: Evan Jones <ej@evanjones.ca>
Committer: GitHub <noreply@github.com>
Date: Fri Nov 03 2023 08:28:22 GMT+0900 (Japan Standard Time)
```

実装は[discussion](https://github.com/golang/go/discussions/63397)で述べられた`v1`のよくなかったところを改善し、`v1`から続くコアコンセプトである、`unsafe`を使わない、読みやすくセキュア、`Easy to use(hard to misuse)`をそのまま反映したようなものになっています。

以下はdiscussion上に貼られた`encoding/json/v2`の構造を表した図への直リンクです

![](https://raw.githubusercontent.com/go-json-experiment/json/6e475c84a2bf3c304682aef375e000771a318a5c/api.png)

大分雰囲気が変わっています。

- jsonの文法的な変換を取り扱う`jsontext`パッケージ
- jsonの意味論的な変換を取り扱う`json`パッケージ

の2つに分割されるようになり、`Encoder` / `Decoder`が中心的に取り扱われるようになりました。
`json`パッケージは従来通り`reflect`を通してmarshaler / unmarshalerを作成して`sync.Map`にキャッシュするなどの挙動を行いますが、`jsontext`はjsonを`[]byte`や`io.Reader` / `io.Writer`から読み書きする機能のみを取り扱います。`jsontext`は「比較的軽量な依存ツリーであるので`TinyGo`/ `GopherJS` / `WASI`のようなバイナリの肥大化が気になるアプリケーションに適している」とのことです。

### Encoder / Decoder

`v2`では`Marshal`/`Unmarshal`のみならず、`MarshalWrite`, `MarshalEncode`, `UnmarshalRead`, `UnmarshalDecode`が追加され、用途に応じて使い分けられるようになりました。下記の通り、今まで通りの`any` -> `[]byte`の変換、`io.Writer`へ直接書き込み、`*jsontext.Encoder`への書き出しとそれぞれ対応しています。

https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/arshal.go#L163

https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/arshal.go#L176

https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/arshal.go#L189

`v1`では[encodeState](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/encode.go;l=250-261), [decodeState](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/decode.go;l=210-220)というステーマシンを定義して、内部的にそれらを呼び出すことでそれらの挙動が実現されていました(呼び出し箇所: [json.Marshal](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/encode.go;l=159), [Encoder.Encode](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/stream.go;l=206), [json.Unmarshal](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/decode.go;l=101), [Decoder](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/stream.go;l=17))。

`v2`では同様にステートマシンを利用しますが、`jsontext.Encoder`, `jsontext.Decoder`はこれらをラップしたものとして定義されています。

https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/jsontext/encode.go#L46-L48

https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/jsontext/decode.go#L76-L78

これをinternal packageとして定義されたexporterを利用して取り出して利用しています。

https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/arshal_default.go#L121-L127

`v1`で問題だったのはモチベーションで述べられていた通り、

- 不要なallocateが生じてしまう
- `MarshalJSON` / `UnmarshalJSON`の呼び出しごとに`encodeState`/`decodeState`がallocateされてしまう
- [(&json.Encoder{}).SetEscapeHTML](https://pkg.go.dev/encoding/json@go1.21.5#Encoder.SetEscapeHTML)などのoptionが`MarshalJSON`実装に伝えられない

であり、
`v2`では以下のように、`MarshalJSONV2`/`UnmarshalJSONV2`は`*jsontext.Encoder`/`*jsontext.Decoder`および`jsontext.Options`を受け取るデザインにすることでそれらを回避しています。

https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/arshal_methods.go#L50-L64

https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/arshal_methods.go#L81-L99

下記のように`UnmarshalJSONV2`の中でデフォルトの`json.Unmarshal`の挙動を利用したい場合でもallocateを避けることができます。

```go
var (
	defaultTime time.Time
)

// foo is an example type which wraps time.Time.
//
// If f is unmarshaled from []byte(`null`),
// it falls back to defaultTime instead of being the zero value.
type foo struct {
	time.Time
}

func (f *foo) UnmarshalJSONV2(dec *jsontext.Decoder, opt jsontext.Options) error {
	// PeekKindの返り値はjsontext.Kindです。
	// 'n'はnullのことです。
	if dec.PeekKind() == 'n' {
		// SkipValueで値を読み飛ばしておきます。
		err := dec.SkipValue()
		if err != nil {
			return err
		}
		f.Time = defaultTime
		return nil
	}
	err := json.UnmarshalDecode(dec, &f.Time, opt)
	if err != nil {
		return err
	}
	return nil
}
```

#### jsontext.Encoder

`WriteToken`、`WriteValue`で値を書き込みますが、内部のステートマシンが状態を覚えているので`:`とか`,`とかを手動で書き込む必要はないです。これはいいデザインですね。

`UnusedBuffer`で`encoderState`に紐づくバッファーが利用できるので、これを利用するとよいというAPIのようです。

```go
package main

import (
	"bytes"
	"fmt"
	"strings"

	"github.com/go-json-experiment/json/jsontext"
)

func main() {
	buf := new(bytes.Buffer)
	enc := jsontext.NewEncoder(buf, jsontext.WithIndent("    "))
	for _, t := range []jsontext.Token{
		jsontext.ObjectStart,
		jsontext.String("foo"),
		jsontext.Null,
		jsontext.String("baz"),
		jsontext.ObjectStart,
		jsontext.String("qux"),
		jsontext.Int(123),
		jsontext.String("quux"),
	} {
		err := enc.WriteToken(t)
		off := enc.OutputOffset()
		depth := enc.StackDepth()
		k, length := enc.StackIndex(depth)
		fmt.Printf("off = %d, kind = %s, depth = %d, length = %d, err = %v\n", off, k, depth, length, err)
	}
	v := enc.UnusedBuffer()
	v = append(v, []byte(`[`)...)
	v = append(v, []byte(`{"corge":null}`)...)
	v = append(v, []byte(`]`)...)
	_ = enc.WriteValue(v)
	_ = enc.WriteToken(jsontext.ObjectEnd)
	_ = enc.WriteToken(jsontext.ObjectEnd)
	fmt.Println(buf.String())
	depth := enc.StackDepth()
	fmt.Printf("depth = %d\n", depth)
	/*
off = 1, kind = {, depth = 1, length = 0, err = <nil>
off = 11, kind = {, depth = 1, length = 1, err = <nil>
off = 17, kind = {, depth = 1, length = 2, err = <nil>
off = 28, kind = {, depth = 1, length = 3, err = <nil>
off = 31, kind = {, depth = 2, length = 0, err = <nil>
off = 45, kind = {, depth = 2, length = 1, err = <nil>
off = 50, kind = {, depth = 2, length = 2, err = <nil>
off = 66, kind = {, depth = 2, length = 3, err = <nil>
{
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

depth = 0
	*/
}
```

#### jsontext.Decoder

`jsontext.Decoder`では`PeekKind`で値を消費せずに`jsontext.Kind`を取得し、`ReadToken`, `ReadValue`で値を読み込めます。

jsonをparseするときに面倒な「今objectがいくつネストしているか」というのが`StackDepth`で取得できます。

```go
package main

import (
	"fmt"
	"strings"

	"github.com/go-json-experiment/json/jsontext"
)

func main() {
	dec := jsontext.NewDecoder(strings.NewReader(`{"foo":"bar", "baz":{"qux":123, "quux":[{"corge":null}]}}`))
	var (
		t   jsontext.Token
		err error
	)
	for err == nil {
		off := dec.InputOffset()
		kind := dec.PeekKind()
		t, err = dec.ReadToken()
		depth := dec.StackDepth()
		_, length := dec.StackIndex(depth)
		pointer := dec.StackPointer()
		fmt.Printf("off = %d, kind = %s, token = %s, depth = %d, length = %d, pointer = %s, err = %v\n", off, kind, t, depth, length, pointer, err)
	}
	/*
off = 0, kind = {, token = {, depth = 1, length = 0, pointer = , err = <nil>
off = 1, kind = string, token = foo, depth = 1, length = 1, pointer = /foo, err = <nil>
off = 6, kind = string, token = bar, depth = 1, length = 2, pointer = /foo, err = <nil>
off = 12, kind = string, token = baz, depth = 1, length = 3, pointer = /baz, err = <nil>
off = 19, kind = {, token = {, depth = 2, length = 0, pointer = /baz, err = <nil>
off = 21, kind = string, token = qux, depth = 2, length = 1, pointer = /baz/qux, err = <nil>
off = 26, kind = number, token = 123, depth = 2, length = 2, pointer = /baz/qux, err = <nil>
off = 30, kind = string, token = quux, depth = 2, length = 3, pointer = /baz/quux, err = <nil>
off = 38, kind = [, token = [, depth = 3, length = 0, pointer = /baz/quux, err = <nil>
off = 40, kind = {, token = {, depth = 4, length = 0, pointer = /baz/quux/0, err = <nil>
off = 41, kind = string, token = corge, depth = 4, length = 1, pointer = /baz/quux/0/corge, err = <nil>
off = 48, kind = null, token = null, depth = 4, length = 2, pointer = /baz/quux/0/corge, err = <nil>
off = 53, kind = }, token = }, depth = 3, length = 1, pointer = /baz/quux/0, err = <nil>
off = 54, kind = ], token = ], depth = 2, length = 4, pointer = /baz/quux, err = <nil>
off = 55, kind = }, token = }, depth = 1, length = 4, pointer = /baz, err = <nil>
off = 56, kind = }, token = }, depth = 0, index = <invalid json.Kind: '\x00'>, length = 1, pointer = , err = <nil>
off = 57, kind = <invalid json.Kind: '\x00'>, token = <invalid json.Token>, depth = 0, index = <invalid json.Kind: '\x00'>, length = 1, pointer = , err = EOF
	*/
}
```

`StackPointer`でJSON Pointer (RFC 6901)が取得できます。これはJSON全体をデコードせずにJSON Pointer一致する任意の値まで読み飛ばすのに使うんでしょうか？

以下のようにすればJSON Pointerに一致するまで読み飛ばすことができますね。

```go
package main

import (
	"bytes"
	"errors"
	"fmt"
	"io"
	"strconv"
	"strings"

	"github.com/go-json-experiment/json"
	"github.com/go-json-experiment/json/jsontext"
)

func readJsonAt(data io.Reader, pointer string, read func(dec *jsontext.Decoder) error) (err error) {
	i := strings.LastIndex(pointer, "/")
	var idx int64 = -1
	if i > 0 && strings.IndexFunc(pointer[i+1:], func(r rune) bool { return '0' <= r && r <= '9' }) >= 0 {
		idx, err = strconv.ParseInt(pointer[i+1:], 10, 64)
		if err != nil {
			panic(err)
		}
		pointer = pointer[:i]
	}

	fmt.Printf("pointer = %s, last idx = %d\n", pointer, idx)

	dec := jsontext.NewDecoder(data)
	for {
		_, err = dec.ReadToken()
		if errors.Is(err, io.EOF) {
			break
		}
		if err != nil {
			return err
		}
		p := dec.StackPointer()
		fmt.Printf("current pointer = %s\n", p)
		if pointer == p {
			if idx >= 0 {
				// skip '['
				_, err = dec.ReadToken()
				if err != nil {
					return err
				}
				for ; idx > 0; idx-- {
					err := dec.SkipValue()
					if err != nil {
						return err
					}
				}
			}
			return read(dec)
		}
	}
	return nil
}

func main() {
	jsonBuf := []byte(`{"yay":"yay","nay":[{"boo":"boo"},{"bobo":"bobo"}],"foo":{"bar":{"baz":"baz"}}}`)

	type gibberish struct {
		Boo  string `json:"boo"`
		Bobo string `json:"bobo"`
		Baz  string `json:"baz"`
	}
	for _, pointer := range []string{"/foo/bar", "/nay/0", "/nay/1"} {
		var gib gibberish
		found := false
		err := readJsonAt(
			bytes.NewBuffer(jsonBuf),
			pointer,
			func(dec *jsontext.Decoder) error {
				found = true
				return json.UnmarshalDecode(dec, &gib)
			},
		)
		fmt.Printf("decoded = %#v, found = %t, err = %v\n", gib, found, err)
	}
}

/*
pointer = /foo/bar, last idx = -1
current pointer =
current pointer = /yay
current pointer = /yay
current pointer = /nay
current pointer = /nay
current pointer = /nay/0
current pointer = /nay/0/boo
current pointer = /nay/0/boo
current pointer = /nay/0
current pointer = /nay/1
current pointer = /nay/1/bobo
current pointer = /nay/1/bobo
current pointer = /nay/1
current pointer = /nay
current pointer = /foo
current pointer = /foo
current pointer = /foo/bar
decoded = main.gibberish{Boo:"", Bobo:"", Baz:"baz"}, found = true, err = <nil>
pointer = /nay, last idx = 0
current pointer =
current pointer = /yay
current pointer = /yay
current pointer = /nay
decoded = main.gibberish{Boo:"boo", Bobo:"", Baz:""}, found = true, err = <nil>
pointer = /nay, last idx = 1
current pointer =
current pointer = /yay
current pointer = /yay
current pointer = /nay
decoded = main.gibberish{Boo:"", Bobo:"bobo", Baz:""}, found = true, err = <nil>
*/
```

### struct tag周りの挙動変更

すべてのoptionは[ここ](https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/fields.go#L432-L446)を参照
詳細な挙動の変更は[diff_test.go](https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/diff_test.go)を参照ください。

- optionが追加、optionの挙動変更

  - `omitzero`: フィールドが型に対応する _zero_ である場合、フィールドがMarshal時にスキップされる
    - _zero_ とは、[`reflect.Value{}.IsZero`](https://pkg.go.dev/reflect#Value.IsZero)がtrueを返すような値、もしくは`interface { IsZero() bool }`を実装する場合、`true`が返されるような値のことです。
  - `time.Time{}.IsZero`が`true`を返す時フィールドをスキップしたいという要望は多くあったのでそれに対応した実装です。
  - `omitempty`: `omitzero`の追加に伴い、`omitempty`はJSONとして _empty_ な値をスキップするように変わりました
    - _epmty_ な値とは`null`, `""`, `{}`, `[]`のいずれかのことであり、`MarshalJSON`および`MarshalJSONV2`でこれらを返した場合にもskipされるようです。
    - ここに`0`が含まれていないのはJSON的に`0`, `-0`, `0.000`などの複数のバリエーションで表現可能だからとのことです。([参考](https://github.com/golang/go/discussions/63397#discussioncomment-7201224))
  - `format`: `time.Time`や`[]byte`および`[N]byte`などに任意のフォーマットを設定できます。
    - 今までは[`time.Time{}`が実装する`MarshalJSON`](https://cs.opensource.google/go/go/+/refs/tags/go1.21.5:src/time/time.go;l=1343)で定義された`time.RFC3339Nano`以外のフォーマットを利用したい場合は`MarshalJSON`を実装した新しい型を定義するほかなかったですが、`v2`ではstruct tagのみで設定できるようになりました。
  - `format:emitnull`を指定することで、`nil slice`や`nil map`がマーシャル時に`null`を出力するようになります。
  - `inline`, `unknown`: モチベーションのところでも述べられていたインライン化するフィールドを指定するオプションです。`map[string]any`なフィールドに`inline`オプションを付けておけば、structに定義されていない値はこのフィールドにすべて格納されます。

- single quoteでエスケープすることが許されるように
  - `v1`はstruct tagは単純にcomma-separatedな文字列であり、`json:"'\,field\,'"`のようなオプションは許されていませんでした。
  - これは実際にはJSONのフィールドとしてはありえます。

`v1`ではtagの解析は以下のような実装でした。

```go
// quoted from https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/tags.go;bpv=0

// parseTag splits a struct field's json tag into its name and
// comma-separated options.
func parseTag(tag string) (string, tagOptions) {
	tag, opt, _ := strings.Cut(tag, ",")
	return tag, tagOptions(opt)
}

// Contains reports whether a comma-separated list of options
// contains a particular substr flag. substr must be surrounded by a
// string boundary or commas.
func (o tagOptions) Contains(optionName string) bool {
	if len(o) == 0 {
		return false
	}
	s := string(o)
	for s != "" {
		var name string
		name, s, _ = strings.Cut(s, ",")
		if name == optionName {
			return true
		}
	}
	return false
}
```

`v2`からはoptionはsingle-quoteによってescapeしてもよいcomma-separatedな文字列という扱いになるようです。

https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/fields.go#L486-L543

### めちゃくちゃ読みやすい

ざっくり大雑把に読み進めていますが`v1`に比べてものすごい読みやすいです。

歴史が浅いこと、それによってコンパイラの最適化をよりあてにできるようになったこと、セキュリティーが重視されるためパフォーマンスよりも可読性が優先されていることなどが要因であると思われます。
作者他の実力の高さと経験がよくわかりますね。

例えば以下の`foldName`では`mid-stack inliner`によってinline化可能なことが述べられていますが、

https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/fold.go#L15-L20

[#19348](https://github.com/golang/go/issues/19348)のこの[issueコメント](https://github.com/golang/go/issues/19348#issuecomment-480994586)を見ると、`mid-stack inliner`の実装時期は`2019-04-09`のあたりのようです。コンパイラの発達によって関数を分割してもパフォーマンスが落ちにくくなりつつあるから読みやすいコードでも大丈夫になっているのだと思います（がコンパイラには全く詳しくないので多分そうなんだろうなぐらいの感想です）

すごく読みやすいのでこれ以上実装について記事内で説明する必要性を感じなくなってきましたのでこの辺にしておきます。

# `v2`で`undefined | null | T`を表現する

冒頭で述べた通り、`v2`ならstdの範疇で`undefined | null | T`が表現可能になります。

## `omitzero`を使う

`omitzero`オプションが追加されたため、`interface { IsZero() bool }`を実装し、`IsZero`メソッド内で`IsUndefined`呼び出せば`undefined`時にフィールドのスキップができます。

```go
package main

import (
	"fmt"

	"github.com/go-json-experiment/json" // github.com/go-json-experiment/json v0.0.0-20231102232822-2e55bd4e08b0
	"github.com/go-json-experiment/json/jsontext"
)

type opt[V any] struct {
	valid bool
	v     V
}

type und[V any] struct {
	opt opt[opt[V]]
}

func Undefined[V any]() und[V] {
	return und[V]{}
}

func Null[V any]() und[V] {
	return und[V]{
		opt: opt[opt[V]]{
			valid: true,
		},
	}
}

func Defined[V any](v V) und[V] {
	return und[V]{
		opt: opt[opt[V]]{
			valid: true,
			v: opt[V]{
				valid: true,
				v:     v,
			},
		},
	}
}

func (u *und[V]) IsZero() bool {
	return u.IsUndefined()
}

func (u *und[V]) IsUndefined() bool {
	return !u.opt.valid
}

func (u *und[V]) IsNull() bool {
	return !u.IsUndefined() && !u.opt.v.valid
}

func (u *und[V]) Value() V {
	if u.IsUndefined() || u.IsNull() {
		var zero V
		return zero
	}
	return u.opt.v.v
}

var _ json.MarshalerV2 = (*und[any])(nil)

func (u *und[V]) MarshalJSONV2(enc *jsontext.Encoder, opt json.Options) error {
	if u.IsUndefined() || u.IsNull() {
		return enc.WriteToken(jsontext.Null)
	}
	return json.MarshalEncode(enc, u.Value(), opt)
}

var _ json.UnmarshalerV2 = (*und[any])(nil)

func (u *und[V]) UnmarshalJSONV2(dec *jsontext.Decoder, opt json.Options) error {
	var v V
	if dec.PeekKind() == 'n' {
		err := dec.SkipValue()
		if err != nil {
			return err
		}
		u.opt.valid = true
		u.opt.v.valid = false
		u.opt.v.v = v
		return nil
	}
	err := json.UnmarshalDecode(dec, &v)
	if err != nil {
		return err
	}
	u.opt.valid = true
	u.opt.v.valid = true
	u.opt.v.v = v
	return nil
}

func main() {
	type some struct {
		Foo und[string] `json:",omitzero"`
		Bar string
	}
	for _, v := range []und[string]{
		Defined[string]("foo"),
		Defined[string](""),
		Null[string](),
		Undefined[string](),
	} {
		bin, err := json.Marshal(some{Foo: v, Bar: "bar"})
		fmt.Printf("bin = %s, err = %+#v\n", bin, err)
		var decoded some
		err = json.Unmarshal(bin, &decoded)
		fmt.Printf("value = %v, undefined = %t, null = %t, err = %+#v\n", decoded.Foo.Value(), decoded.Foo.IsUndefined(), decoded.Foo.IsNull(), err)
	}
}
/*
bin = {"Foo":"foo","Bar":"bar"}, err = <nil>
value = foo, undefined = false, null = false, err = <nil>
bin = {"Foo":"","Bar":"bar"}, err = <nil>
value = , undefined = false, null = false, err = <nil>
bin = {"Foo":null,"Bar":"bar"}, err = <nil>
value = , undefined = false, null = true, err = <nil>
bin = {"Bar":"bar"}, err = <nil>
value = , undefined = true, null = false, err = <nil>
*/
```

簡単ですね！

ただしこれだと`omitzero`オプションをフィールドごとに設定する必要があり、struct fieldだけで表現できてはいません。

## \*struct fieldだけで\*`undefined | null | T`を表現する

struct fieldだけで`undefined | null | T`を実現するためにもう少し工夫してみます。

`v2`のフィールドスキップの挙動は以下の行で実装されています

https://github.com/go-json-experiment/json/blob/2e55bd4e08b08427ba10066e9617338e1f113c53/arshal_default.go#L933-L948

当然ではありますが部分的にロジックを取り出せるようにはなっていませんので、特定のinterfaceを実装するとき`structField.omitzero = true`にするというようなことはできません。
(`//go:linkname`で関数を呼び出せても難しそう)

ところで、`v2`には以下のように特定の型のMarshaler, Unmarshalerを差し替えるoptionがありますので、こちらを利用することにします。

```go
marshaller := json.MarshalFuncV2[some](func(e *jsontext.Encoder, s some, o json.Options) error {
	// ...snip...
})
out, err := json.Marshal(v, json.WithMarshalers(marshaller))
```

[`reflect.StructOf`](https://pkg.go.dev/reflect@go1.21.5#StructOf)を利用するとランタイムで型を生成できるので、struct tagを差し替えた型を作成し、元の型から生成した型へ変換をかけてからデフォルトの`json.MarshalEncode`の挙動を呼び出すだけで機能を実現できます。

### struct tagだけ差し替えた型を作成する

Goのstructは別の型をembedできたり、embedした型が再帰できたりと、reflectを使った型の変換は簡単そうで厄介なのですが、struct tagをいじるだけならば以下のようなコードで十分です。

```go
package faketagencoder

import "reflect"

type Skipper func(reflect.Type) bool

func SkipImplementor(rt reflect.Type) Skipper {
	return func(t reflect.Type) bool {
		return t.Implements(rt) ||
			(t.Kind() == reflect.Pointer && t.Elem().Implements(rt)) ||
			reflect.PointerTo(t).Implements(rt)
	}
}

func SkipNot(s Skipper) Skipper {
	return func(t reflect.Type) bool {
		return !s(t)
	}
}

func SkipAnonymous() Skipper {
	return func(t reflect.Type) bool {
		if t.Kind() == reflect.Pointer {
			t = t.Elem()
		}
		if t.Kind() != reflect.Struct {
			return false
		}
		for i := 0; i < t.NumField(); i++ {
			if t.Field(i).Anonymous {
				return true
			}
		}
		return false
	}
}

func CombineSkipper(skippers ...Skipper) Skipper {
	return func(t reflect.Type) bool {
		for _, skipper := range skippers {
			if skipper(t) {
				return true
			}
		}
		return false
	}
}

type TagMutator func(reflect.StructField) reflect.StructTag

func MutateTag(
	rt reflect.Type,
	skipAdvancing Skipper,
	mutateTag TagMutator,
) reflect.Type {
	fields := make([]reflect.StructField, rt.NumField())
	for i := 0; i < rt.NumField(); i++ {
		field := rt.Field(i)

		typ := field.Type

		if !skipAdvancing(typ) {
			if typ.Kind() == reflect.Struct {
				typ = MutateTag(typ, skipAdvancing, mutateTag)
			} else if typ.Kind() == reflect.Pointer {
				elem := typ.Elem()
				if elem.Kind() == reflect.Struct {
					elem = MutateTag(elem, skipAdvancing, mutateTag)
					typ = reflect.PointerTo(elem)
				}
			}
		}

		fields[i] = reflect.StructField{
			Name:      field.Name,
			PkgPath:   field.PkgPath,
			Type:      typ,
			Tag:       mutateTag(field),
			Offset:    field.Offset,
			Index:     field.Index,
			Anonymous: field.Anonymous,
		}
	}

	return reflect.StructOf(fields)
}
```

多分この実装では再帰のある型だとstack overflowしますね。
`sync.Pool`に型をキャッシュするようにしてすでに作成済みの型はキャッシュから引き出すようにするとかそういった処理が必要ですがこのコードは実用するつもりがないのでまあこのままでいいでしょう。

### struct tagにoptionを追加する

[前回の記事のこの部分](https://zenn.dev/ngicks/articles/go-json-that-can-be-t-null-or-undefined#jsoniter-%E3%81%AE-extension-%E3%81%A7%E4%BD%95%E3%81%A8%E3%81%8B%E3%81%99%E3%82%8B)で`reflect.StructTag`を引数に`omitempty`がなければ追加するという処理を書きましたが、
前述のとおり`v2`はstruct tag optionがsingle-quotationでエスケープされた文字列や、`format:RFC3339`のように`:`で区切りの文字列を許すように拡張されたそれに合わせた処理が必要です。

```go
package faketagencoder

// This file uses modified Go programming language standard library.
// So keep it credited.
//
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.
//
// Modified parts are governed by a license that is described in ./LICENSE.

import (
	"errors"
	"fmt"
	"io"
	"reflect"
	"strconv"
	"strings"
	"unicode"
	"unicode/utf8"
)

var (
	ErrUnpairedKey = errors.New("unpaired key")
)

func AddOption(tag string, opt string, ignoreIf func(t reflect.Type) bool) TagMutator {
	return func(sf reflect.StructField) reflect.StructTag {
		if ignoreIf(sf.Type) {
			return sf.Tag
		}
		added, err := AddTagOption(sf.Tag, tag, opt)
		if err != nil {
			// downstream may return same error
			return sf.Tag
		}
		return added
	}
}

type Tag struct {
	Key   string
	Value string
}

func (t Tag) Flatten() string {
	return t.Key + ":" + strconv.Quote(t.Value)
}

func ParseStructTag(tag reflect.StructTag) ([]Tag, error) {
	var out []Tag

	for tag != "" {
		// Skip leading space.
		i := 0
		for i < len(tag) && tag[i] == ' ' {
			i++
		}
		tag = tag[i:]
		if tag == "" {
			break
		}

		// Scan to colon. A space, a quote or a control character is a syntax error.
		// Strictly speaking, control chars include the range [0x7f, 0x9f], not just
		// [0x00, 0x1f], but in practice, we ignore the multi-byte control characters
		// as it is simpler to inspect the tag's bytes than the tag's runes.
		i = 0
		for i < len(tag) && tag[i] > ' ' && tag[i] != ':' && tag[i] != '"' && tag[i] != 0x7f {
			i++
		}
		if i == 0 || i+1 >= len(tag) || tag[i] != ':' || tag[i+1] != '"' {
			return nil, fmt.Errorf("%w: input has no paired value, rest = %s", ErrUnpairedKey, string(tag))
		}
		name := string(tag[:i])
		tag = tag[i+1:]

		// Scan quoted string to find value.
		i = 1
		for i < len(tag) && tag[i] != '"' {
			if tag[i] == '\\' {
				i++
			}
			i++
		}
		if i >= len(tag) {
			return nil, fmt.Errorf("%w: name = %s has no paired value, rest = %s", ErrUnpairedKey, name, string(tag))
		}
		quotedValue := string(tag[:i+1])
		tag = tag[i+1:]

		value, err := strconv.Unquote(quotedValue)
		if err != nil {
			return nil, err
		}
		out = append(out, Tag{Key: name, Value: value})
	}

	return out, nil
}

func StructTagOf(tags []Tag) reflect.StructTag {
	var buf strings.Builder
	for _, tag := range tags {
		buf.Write([]byte(tag.Flatten()))
		buf.WriteByte(' ')
	}

	out := buf.String()
	if len(out) > 0 {
		out = out[:len(out)-1]
	}
	return reflect.StructTag(out)
}

// AddTagOption returns a new StructTag which has value added for tag.
// It assumes tag options are formatted as `tag:"name,opt,opt"` style.
// The names, and opts are allowed to be quoted by single quotation marks.
func AddTagOption(t reflect.StructTag, tag string, option string) (reflect.StructTag, error) {
	tags, err := ParseStructTag(t)
	if err != nil {
		return "", err
	}

	hasTag := false
	for i := 0; i < len(tags); i++ {
		if tags[i].Key != tag {
			continue
		}

		hasTag = true

		hasValue := false

		value := tags[i].Value
		// first, skip name.
		if len(value) > 0 && !strings.HasPrefix(value, ",") {
			n := len(value) - len(strings.TrimLeftFunc(value, func(r rune) bool {
				return !strings.ContainsRune(",\\'\"`", r) // reserve comma, backslash, and quotes
			}))
			if n == 0 {
				_, n, err = readTagOption(value)
				if err != nil {
					return "", err
				}
			}
			value = value[n:]
		}

		for len(value) > 0 {
			if value[0] != ',' {
				return "", fmt.Errorf("malformed option, %s", tags[i].Value)
			} else {
				value = value[1:]
				if len(value) == 0 {
					return "", fmt.Errorf("malformed option, %s", tags[i].Value)
				}
			}

			opt, n, err := readTagOption(value)
			if err != nil {
				return "", err
			}

			value = value[n:]
			if len(value) > 0 && value[0] == ':' {
				if strings.HasPrefix(option, opt+":") {
					hasValue = true
					break
				}
				value = value[len(":"):]
				_, n, err := readTagOption(value)
				if err != nil {
					return "", err
				}
				value = value[n:]
			}

			if option == opt {
				hasValue = true
				break
			}
		}

		if !hasValue {
			if !strings.HasPrefix(option, ",") {
				tags[i].Value += ","
			}
			tags[i].Value += option
		}
		break
	}

	if !hasTag {
		tags = append(tags, Tag{Key: tag, Value: option})
	}

	return StructTagOf(tags), nil
}

func readTagOption(s string) (opt string, n int, err error) {
	if len(s) == 0 {
		return "", 0, io.ErrUnexpectedEOF
	}

	switch r, _ := utf8.DecodeRuneInString(s); {
	case r == '_' || unicode.IsLetter(r): // Go ident
		n = len(s) - len(strings.TrimLeftFunc(s, func(r rune) bool {
			return r == '_' || unicode.IsLetter(r) || unicode.IsNumber(r)
		}))
		return s[:n], n, nil
	case r == '\'': // escaped
		return unescape(s)
	default:
		return "", 0, fmt.Errorf("invalid character: %s", s)
	}
}

func unescape(s string) (unescaped string, n int, err error) {
	i := 0
	if s[0] == '\'' {
		i = 1
	}

	escaping := false
	escaped := []byte{'"'}
	for i < len(s) {
		r, rn := utf8.DecodeRuneInString(s[i:])
		switch {
		case escaping:
			if r == '\'' {
				escaped = escaped[:len(escaped)-1]
			}
			escaping = false
		case r == '\\':
			escaping = true
		case r == '"':
			escaped = append(escaped, '\\')
		case r == '\'':
			escaped = append(escaped, '"')
			i += 1
			out, err := strconv.Unquote(string(escaped))
			if err != nil {
				return "", 0, fmt.Errorf("invalid escaped string: string must be escaped by single quotes, input = %s", s)
			}
			return out, i, nil
		}
		escaped = append(escaped, s[n:][:rn]...)
		i += rn
	}
	return "", 0, fmt.Errorf("invalid escaped string: single-quoted string missing terminating single-quote: %s", s)
}
```

### 呼び出す

以下のように呼び出します。
`reflect`に依存する都合上exported fieldしかコピーできませんが、よく考えたら`json.Marshal`もunexported fieldを無視しますので問題ありませんでした。

作っておいてなんですが、`reflect`によるコピーが生じるぶんパフォーマンス的にもメモリー的にも負荷が上がるはずなので`omitzero`を手書きでしたほうがいいと思います！

```go
package main

import (
	"fmt"
	"reflect"

	"github.com/go-json-experiment/json" // github.com/go-json-experiment/json v0.0.0-20231102232822-2e55bd4e08b0
	"github.com/go-json-experiment/json/jsontext"
	"github.com/ngicks/faketagencoder"
)

// 省略

type Undefinedable interface {
	IsUndefined() bool
}

var (
	undefinedableType = reflect.TypeOf((*Undefinedable)(nil)).Elem()
	jsonV1Marshaller  = reflect.TypeOf((*json.MarshalerV1)(nil)).Elem()
	jsonV2Marshaller  = reflect.TypeOf((*json.MarshalerV2)(nil)).Elem()
)

func setExported(l, r reflect.Value) {
	for i := 0; i < l.NumField(); i++ {
		fl := l.Field(i)
		fr := r.Field(i)
		if fl.Type() == fr.Type() {
			fl.Set(fr)
		} else {
			setExported(fl, fr)
		}
	}
}

func main() {
	type Nested struct {
		Nah und[string] `json:"nah"`
		Yay int         `json:"yay"`
	}
	type some struct {
		Foo und[string] `json:"foo"`
		Bar string      `json:"bar"`
		Baz Nested      `json:"baz"`
		Nested
	}
	mutated := faketagencoder.MutateTag(
		reflect.TypeOf(some{}),
		faketagencoder.CombineSkipper(
			faketagencoder.SkipImplementor(jsonV1Marshaller),
			faketagencoder.SkipImplementor(jsonV2Marshaller),
		),
		faketagencoder.AddOption(`json`, `,omitzero`, faketagencoder.SkipNot(faketagencoder.SkipImplementor(undefinedableType))),
	)

	fmt.Printf("mutated type = %+v\n", mutated)
/*
	mutated type = struct { Foo main.und[string] "json:\"foo,omitzero\""; Bar string "json:\"bar\""; Baz struct { Nah main.und[string] "json:\"nah,omitzero\""; Yay int "json:\"yay\"" } "json:\"baz\""; Nested struct { Nah main.und[string] "json:\"nah,omitzero\""; Yay int "json:\"yay\"" } }
*/

	marshaller := json.MarshalFuncV2[some](func(e *jsontext.Encoder, s some, o json.Options) error {
		rv := reflect.ValueOf(s)
		v := reflect.New(mutated).Elem()
		setExported(v, rv)
		return json.MarshalEncode(e, v.Interface(), o)
	})

	for _, v := range []some{
		{},
		{
			Foo: Defined("foo"),
			Bar: "bar",
			Baz: Nested{
				Nah: Defined("nah"),
				Yay: 20,
			},
			Nested: Nested{
				Nah: Defined("nah"),
				Yay: -231,
			},
		},
	} {
		out, err := json.Marshal(v, json.WithMarshalers(marshaller), jsontext.WithIndent("    "))
		if err != nil {
			panic(err)
		}
		fmt.Printf("%s\n", out)
	}
	/*
		{
		    "bar": "",
		    "baz": {
		        "yay": 0
		    },
		    "yay": 0
		}
		{
		    "foo": "foo",
		    "bar": "bar",
		    "baz": {
		        "nah": "nah",
		        "yay": 20
		    },
		    "nah": "nah",
		    "yay": -231
		}
	*/
}
```

# おわりに

この記事では`v2`のモチベーション、API、部分的な実装と`v1`との差異について紹介し、`undefined | null | T`が`v2`によって実現可能なことをしましました。
さらに、\*struct fieldだけで\*`undefined | null | T`を実現するために`v2`のoptionについて少し探索しました。

[このコメント曰く](https://github.com/golang/go/discussions/63397#discussioncomment-7438359)proposalになる前にdiscussionを受けたAPIの変更と`v1`との互換性レイヤーを実装するとのことなので、
`v2`がこのまま実装されるわけでもなさそうですし、すぐリリースということでもなさそうです。
非常に楽しみです。
