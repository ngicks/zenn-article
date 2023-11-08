---
title: "GoでJSONのundefinedとnullを表現するv2(候補)版"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

# TL;DR

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
		_, err := dec.ReadToken()
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

これでおわりです。

これだけだとあんまりなのでもう少し詳しく触れていきましょう。

# 想定読者

- [Go programming language](https://go.dev/) で [encoding/json](https://pkg.go.dev/encoding/json)の機能を十分理解している人。

# GoでJSONのundefinedとnullを表現するv2(候補)版

以前書いた[Goのstruct fieldでJSONのundefinedとnullを表現する](https://zenn.dev/ngicks/articles/go-json-that-can-be-t-null-or-undefined)では、[jsoniter](https://github.com/json-iterator/go)の [Extension](https://pkg.go.dev/github.com/json-iterator/go#Extension) を駆使していろいろ頑張ることで`undefined | null | T`を出し分けることが実現できることを確認しました。記事中では同様に、`encoding/json/v2`でstdライブラリとして同様のことができるようになるかもしれないということも触れました。

先日(`2023-10-05T17:14:54Z`)、記事内で触れた[issue comment](https://github.com/golang/go/issues/5901#issuecomment-907696904)の筆者が[encoding/json/v2](https://github.com/golang/go/discussions/63397)というタイトルのdiscussionを作りました。

# encoding/json/v2

## Background

- discussion: [encoding/json/v2](https://github.com/golang/go/discussions/63397)

ざっくりdiscussionの中で述べられるモチベーションを列挙します。

- Missing Functionality
  - `time.Time`のフォーマットを指定できない
  - 特定の値をomitできない
    - e.g. `time.Time`のzero valueを`omitempty`でomitしたいのに`struct`には決して`omitempty`が機能しない
  - `slice`や`map`が`nil`であるとき空の`Array`(`[]`), `Object`(`{}`)を出力できない
  - `embed`以外の方法で出力結果に`Go type`をinlineできないこと
    - i.e. Go structで定義したfield以外は全部`map[string]any`に詰めるようなことができない
    - [compose-goのProjectの例](https://github.com/compose-spec/compose-go/blob/8df318e42a041ea3a1e1f5abbbd615619c9ee86c/types/project.go#L45)。この場合`gopkg.in/yaml.v3`にinline化機能があるようのでyamlとの読み書きでは問題にならない。
    - `encoding/json`でそのようなことをする場合、MarshalJSON / UnmarshalJSONの中で手動でハンドルする必要がある([自作の例](https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/test/additional_prop_escape.go#L37-L131))
- API deficiencies
  - `io.Reader`からうまくjsonをdecodeする方法がない
    - `json.NewDecoder(r).Decode(v)`がよくされるがこれは誤りである: `Decode`は1つの有効な`json` tokenだけを取り出すので、末尾にゴミデータがある場合にエラーにならない。
      - (エラーになってほしい場合、`Decode`を呼んだ後に`dec.More()`が`true`を返すときエラーを返すような処理をユーザーが書く必要があります)
  - `Decoder`, `Encoder`にOptionを設定する方法があるが([SetEscapeHTML](https://pkg.go.dev/encoding/json#Encoder.SetEscapeHTML)とか[DisallowUnknownFields](https://pkg.go.dev/encoding/json#Decoder.DisallowUnknownFields)とかのこと)、`json.Marhsal`, `json.Unmarshal`にはない
  - `json.Compact`, `json.Indent`, `json.HTMLEscape`などが`bytes.Buffer`を使っていること。より柔軟な`[]byte`, `io.Writer`などを使わないこと。
    - (これは筆者も大分不思議に思っていました。筆者の使い方的には困らないので別によかったのですが、`[]byte`や`io.Writer`を使うAPIのほうが好ましいですね)
- Performance limitations
  - `MarshalJSON`は返り値が`[]byte`であるのでallocateが生じること。`UnmarshalJSON`は引数には`[]byte`をとり、プロトコル上前後に空白のない1つのjson tokenのみ含むようにする必要があるので、呼び出し側がtokenの解析とフォーマットを行う必要があること。さらに、`UnmarshalJSON`の中で`[]byte`を解析する必要があるので二重に解析が必要であること。
  - streaming encoder APIが存在しない。`Encoder.EncodeToken` proposalは通過したが、実装はされていない([#40127](https://github.com/golang/go/issues/40127))
  - `json.Token`はinterfaceであるため、JSON numberやstringをbox化するときにallocateしてしまう。
  - steaming APIが存在しない: 理屈上最も大きなJSON Tokenのみバッファーすればよいはずだが、`Encoder.Encode`および`Decoder.Decode`はJSON全体をバッファーしてしまう。
- Behavioral flaws
  - 時間とともにJSONに関する標準は増えている ([RFC 4627](https://tools.ietf.org/html/rfc4627), [RFC 7159](https://tools.ietf.org/html/rfc7159), [RFC 7493](https://tools.ietf.org/html/rfc7493), [RFC 8259](https://tools.ietf.org/html/rfc8259))。一般的に言って、これらのRFCは徐々により厳密な定義になっていっているが、`encoding/json`は新しいRFCに適合していない。例えば、[RFC 8259](https://tools.ietf.org/html/rfc8259)ではinvalidなUTF-8のシーケンスを許さないが、`encoding/json`はそれらを許容する。
    - ([Go v1のリリース日は2012-03-28](https://go.dev/doc/devel/release#go1)であり、[RFC 7159](https://tools.ietf.org/html/rfc7159)が2014-03、, [RFC 7493](https://tools.ietf.org/html/rfc7493)は2015-03, [RFC 8259](https://tools.ietf.org/html/rfc8259)は2017-12です)
  - Unmarshal時のGo struct fieldとJSON objectのキー名とのマッチングがでcase-insensitiveである。
  - underlying typeがnon-addressableである型の`MarshalJSON` / `UnmarshalJSON`が呼ばれない。しかし、この挙動を修正することもbreaking changeである。
  - `json.Unmarshal(data, &v)`のターゲット(=`v`)がzero valueでない時に起きるマージの挙動に一貫性がない: 例えばnon-zeroなsliceを使用した場合、Unmarshal後のsliceのlengthとcapacityの間の値はゼロ化されずにマージされる。
  - 一貫しないエラー値: 現在の`encoding/json`の返すエラーは構造化されている部分とされていない部分があり一貫しない。実際には3つのクラスのエラーが起きるはずである: 文法エラー、意味論エラー(JSONとしては正しいが、対応付けられたGo typeへ向けて変換ができない. JSON stringをGo intへ変換しようとするときなど。)、I/Oエラー。

## Implementation

- implementation: [github.com/go-json-experiment/json](https://github.com/go-json-experiment/json)

この記事では実装はすべて以下のコミットの状態で確認されています。

```
Commit: 2e55bd4e08b08427ba10066e9617338e1f113c53
Parents: 54c864be5b8da112b1492c72087512969f2fdea4
Author: Evan Jones <ej@evanjones.ca>
Committer: GitHub <noreply@github.com>
Date: Fri Nov 03 2023 08:28:22 GMT+0900 (Japan Standard Time)
```

以下はdiscussion上に貼られた`encoding/json/v2`の構造を表した図への直リンクです

![](https://raw.githubusercontent.com/go-json-experiment/json/6e475c84a2bf3c304682aef375e000771a318a5c/api.png)

大分雰囲気が変わっています。

- jsonの文法的な変換を取り扱う`jsontext`パッケージ
- jsonの意味論的な変換を取り扱う`json`パッケージ

の2つに分割されるようになり、`Encoder` / `Decoder`が中心的に取り扱われるようになりました。
`json`パッケージは従来通り`reflect`を通してmarshaler / unmarshalerを作成して`sync.Map`にキャッシュするなどの挙動を行いますが、`jsontext`は「比較的軽量な依存ツリーであるので`TinyGo`/ `GopherJS` / `WASI`のようなバイナリの肥大化が気になるアプリケーションに適している」とのことです。

- `MarshalJSON` / `UnmarshalJSON`が効率的に
  - `encoding/json`では[encodeState](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/encode.go;l=250-261), [decodeState](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/decode.go;l=210-220)というステーマシンを定義して、内部的にそれらを呼び出すことでそれらの挙動が実現されていました(呼び出し箇所: [json.Marshal](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/encode.go;l=159), [Encoder.Encode](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/stream.go;l=206), [json.Unmarshal](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/decode.go;l=101), [Decoder](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/encoding/json/stream.go;l=17))。
  - `v2`からは逆に`Encoder` / `Decoder`が外部に存在しており、`encoderState` / `decoderState`はそれぞれのunexported fieldとなっています。
  - v1では`MarshalJSON` / `UnmarshalJSON`の実装の中で`json.Marshal`や`json.Unmarshal`を使っている場合にはこれらのステートマシンが呼び出しごとに再生成されていましたが、新しい`MarshalJSONV2` / `UnmarshalJSONV2`は引数で`Encoder` / `Decoder`と`json.Options`を受けとるようになったため、ずいぶんと効率的になるようになりました。
  - v1では`decodeState`は`sync.Pool`などを使って再利用されていませんでしたが、`v2`からは`decoderState`もキャッシュされてるようになっています。
