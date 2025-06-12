---
title: "gotipでencoding/json/v2を試す"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## gotipでencoding/json/v2を試す

長いこと[discussion](https://github.com/golang/go/discussions/63397)にあった`encoding/json/v2`ですが2025-01-31に以下のproposalに移行しました。

https://github.com/golang/go/issues/71497

[proposal]: https://github.com/golang/go/issues/71497

2025-04-18に`GOEXPERIMENT=jsonv2`下で利用できるようになるCLがマージされました。

https://go-review.googlesource.com/c/go/+/665796

すでに`go1.25rc1`で`GOEXPERIMENT`付きで試せるようになっています。
`iterator`のように問題なく進めば`Go 1.26`で正式実装になるでしょう。

筆者は以下の記事などで何度か`encoding/json/v2`について触れていますが

https://zenn.dev/ngicks/articles/go-json-undefined-or-null-v2

何が変わったかなどをついてこの記事で述べていきたいと思います。この記事は結構焼き増しです！！！

以後特にほかに述べられないとき`v1`とは`encoding/json`のことをさし、`v2`とは`encoding/json/v2`のことをさします。

## EDIT LOG

:::details edit logs

### 2025-06-12

`go1.25rc1`がリリースされたのでそれに合わせて変更しました。

### 2025-05-23

- [v2.Marshal / v2.Unmarshal](#v2.marshal-%2F-v2.unmarshal)を追加しました。一番ふつうの使い方です。

### 2025-05-14

- [v1からの顕著な変更](#v1からの顕著な変更)部分を増量しました。
- [jsontext.Decoder.StackPointer](#jsontext.decoder.stackpointer)部分もうちょい凝った処理にしました。
- [tee-ing版](#tee-ing版)増設して`*jsontext.Decoder`で読んだ内容を2つにteeする方法を述べました。
  :::

## サンプル

以下に上がります。

https://github.com/ngicks/go-play-encoding-json-v2

## 環境

```
$ go version
go version go1.25rc1 linux/amd64
```

## (いらなくなったので省略)gotipで開発できるようにする

`go 1.25rc1`がリリースされて不要になりました。

:::details gotipで開発できるようにする

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

:::

## プロジェクトを作成

```
$ GOTOOLCHAIN=go1.25rc1 go mod init github.com/ngicks/go-play-encoding-json-v2
go: creating new go.mod: module github.com/ngicks/go-play-encoding-json-v2
$ ls
go.mod
$ cat go.mod
module github.com/ngicks/go-play-encoding-json-v2

go 1.25rc1
```

プロジェクトを実行したり、goplsを起動するには若干面倒ですが下記の手順が必要です

```
$ export GOEXPERIMENT=
$ export PATH=$(GOTOOLCHAIN=go1.25rc1 go env GOROOT)/bin:$PATH
$ export GOEXPERIMENT=jsonv2
$ go version
go version go1.25rc1 linux/amd64
```

こうしないと`go`がパニックします。多分ですが`go` commandがビルドされた時点で存在していた`GOEXPERIMENT`以外が存在していることが条件です。
`PATH`で指定される`go`が`go1.24.4`とかだと`GOEXPERIMENT=jsonv2`が存在しません。

既存のプロジェクトに追加して遊ぶ場合は

```
//go:build (go1.25 && goexperiment.jsonv2) || go1.26
```

をファイルの先頭に付け足します。このbuild constraintによって、`go1.25`かつ`GOEXPERIMENT=jsonv2`のときのみコンパイル対象になります。
`go1.26`以降に`GOEXPERIMENT`なしになっているのは希望的観測なのですが多分大丈夫でしょう。

## 経緯

詳しくは前述の[discussion](https://github.com/golang/go/discussions/63397)に書かれているのでそこを読んでほしいのですが、適当にまとめ直すと

`encoding/json`にはいろいろ問題がありました。

- 機能が欠けている
  - `time.Time`のフォーマットを指定できない
  - 特定の値をomitできない(e.g. zero valueの`time.Time`)
  - slice(`[]T`)やmap(`map[K]V`)がnilのとき必ず`null`が出力され、`[]`や`{}`を出力する方法がない
  - 型をembedする以外に値をinlineする方法がない
  - strcutなどに向けてunmarshalするとき、想定されないフィールドが含まれるときにそれらをどこかにfallbackする機能がない
  - etc, etc.
- APIが変
  - `json.NewDecoder(r).Decode(v)`がよくされるが、`r io.Reader`から1つのJSON Valueを取り出すだけでそのあとにゴミデータがあった場合などにエラーにならない。
  - `Decoder`, `Encoder`にOptionを設定する方法があるが([SetEscapeHTML](https://pkg.go.dev/encoding/json@go1.24.3#Encoder.SetEscapeHTML)とか[DisallowUnknownFields](https://pkg.go.dev/encoding/json@go1.24.3#Decoder.DisallowUnknownFields)とかのこと)、`json.Marhsal`, `json.Unmarshal`にはない
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
  - Unmarshal時のGo structフィールド名とJSON objectのキー名とのマッチングがcase-insensitive。
    - `json:"name"`で指定していたとしてもcase-insensitiveでした。
  - underlying typeがnon-addressableである型の`MarshalJSON` / `UnmarshalJSON`が呼ばれない。
    - method receiverがpointerで`json.Marshal`に渡された値がnon-pointerだった時のことをさしています。
  - unmarshalのターゲットにnon-zeroな値を渡したときの挙動に一貫性がない
    - sliceはunmarshal後の長さが短くなる場合、`s[len(s):cap(s)]`の区間はzero化されていませんでした。
  - エラーに一貫性がなく、io error, syntax error, semantic errorが構造化されずにごっちゃに返されている。

破壊的変更を避けながらこれらを解消することはできるかもしれないが、デフォルトの挙動を[RFC 8259](https://tools.ietf.org/html/rfc8259)に準拠させるなどしたほうが良いと思われるので`v2`として破壊的変更を加えよう、みたいな感じです。

## v1からの顕著な変更

### encoding/json/jsontextとencoding/json/v2に分かれる

[proposal]に貼られていますが、以下のように構造が変化します。

![](https://raw.githubusercontent.com/go-json-experiment/json/6e475c84a2bf3c304682aef375e000771a318a5c/api.png)

`encoding/json/jsontext`を追加し、ここでJSONの文法を処理します。
`encoding/json/v2`を追加し、ここで`jsontext`で処理されたトークン情報を用いて、`v1`と同様に`reflect`などを使用して`Go`の値との相互的なやり取りを実現します。

### encoding/json/v2

[json.Marshal], [json.Unmarshal]とほぼ同等なものに加えて、`io.Reader`/`io.Writer`, `*jsontext.Encoder`/`*jsontext.Decoder`を引数に取る新しいAPIが追加されます。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal.go#L167

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal.go#L183

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal.go#L203

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal.go#L398

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal.go#L415

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal.go#L448

見てのとおり、`Options`という型でoptionを受け付けられるようになっており、ユーザー側で柔軟な挙動の変更が可能です。

### encoding/json/jsontext

[*json.Encoder]/[*json.Decoder]に当たるものとして`*jsontext.Encoder`/`*jsontext.Decoder`が実装されます。
それぞれ[io.Writer]/[io.Reader]を引数にとって初期化します。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/jsontext/encode.go#L84-L95

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/jsontext/decode.go#L117-L126

JSONのToken単位での読み書きをする`Write/ReadToken`と,JSON Value単位(`"foo"`のようなstring literalや`{"foo":"bar"}`のような1つのJSON Objectなど)での読み書きをする`Write/ReadValue`があり、`v1`に比べるとよりレキシカルな操作が可能です。

また、[*json.Decoder.More]の代わりに`PeekKind`があり、これによって次のtokenの種類(kind)をしることができます。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/jsontext/decode.go#L302-L309

(`v1`では読んじゃったらUnreadできないからすごい困ってたんですよ)

`*jsontext.Encoder`/`*jsontext.Decoder`ともに、`StackDepth`, `StackIndex`, `StackPointer`を実装しており、現在のnestの深さ、ある深さの開始Token(`{`なのか`[`なのか)、現在の位置のJSON Pointer([RFC 6901])をそれぞれ知ることができます。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/jsontext/decode.go#L1126-L1134

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/jsontext/decode.go#L1136-L1158

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/jsontext/decode.go#L1160-L1163

`jsontext.Token`は構造体、`jsontext.Value`は`[]byte`であり、interfaceではないので不要なbox化が起きなくなっています。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/jsontext/token.go#L32-L90

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/jsontext/value.go#L38-L47

### Options

多くの`v2` APIがうけつける`Options`型によって、encoder単位、呼び出し単位でふるまいのカスタマイズが可能になっています。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/internal/jsonopts/options.go#L7-L18

`Options`はinterfaceですが上記のとおり現状`encoding/json`以下のpackageからしか定義ができないようになっています。
逆に言うとこのハックによって`encoding/json/v2`, `encoding/json/jsontext`のそれぞれにふさわしいpackageで`Options`が定義されています。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/jsontext/options.go#L17-L45

上記のようにいろいろあります。

関連APIが`Options`を受け取るほか、`*jsontext.Encoder`/`*jsontext.Decoder`がこれを引きまわすようになっているため、`MarshalJSONTo`/`UnmarshalJSONFrom`の実装もこれらを受け取ることができます。

例えば`v2`で`v1`の[json.MarshalIndent]相当のことをするには`jsontext.WithIdent`を`v2.Marshal*`系APIに渡します。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/jsontext/options.go#L221-L255

`*jsontext.Encoder`はこの`Options`が渡されていた場合、streamへの書き出し時にObjectやArrayがnestするたびに`StackDepth`に応じたインデントを書きだします。`v1`では一旦Marshalをして`[]byte`を出力し、これを解析してインデントをつけなおすという遠回りな処理をしていましたが、`v2`では`Options`がencoderについて回ることで処理がずいぶんシンプルになっています。

encoder単位、呼び出し単位でのふるまいの変更ができる`Options`で顕著なものは`MarshalToFunc[T any]`/`UnmarshalFunc[T any]`です。
これらは、関数をあたることで特定の型`T`のmarshal, unmarshalのふるまいを変えることができます。
`v1`までは[json.Marshaler]/[json.Unmarshaler]を型に実装させるしかありませんでしたが、`v2`では呼び出しごとに変更することができるほか、closureを渡すことができるので、例えばある型が見つかるたび`channel`に送信するとか、カウントをインクリメントするとかもできます。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal_funcs.go#L204-L251

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/arshal_funcs.go#L287-L336

見てのとおりそれぞれencoder/decoderを受け取るため、`StackDepth`, `StackIndex`, `StackPointer`を用いて階層情報を取得しそれに基づいてふるまいを変えることもできるようになっています。

### struct tag

`json:""` struct tagも大幅な変更があります。

https://github.com/golang/go/blob/0e17905793cb5e0acc323a0cdf3733199d93976a/src/encoding/json/v2/fields.go#L469-L555

特に顕著なのは

- `case:value`, `format:value`などでフォーマットを指定できるように(のちにサンプルを示す)
  - 前述の`time.Time`が`RFC3339`以外でmarshal/unmarshalできなかった問題が解決します。
- `inline`で型をembedしなくてもembedしてた時みたいにmarshal/unmarshalされます
- `unknown`でstrcut fieldなどで指定していない(=不明な)フィールドをまとめて格納することができます。(のちにサンプルを示す)
- `json:"name"`のname部分をsingle quotation mark(`'`)でescape可能に
  - `json:"',\"'"`と設定すれば、`",\""`というフィールドが出力されます。

## discussion版からの顕著な変更

https://github.com/golang/go/issues/71497#issuecomment-2626483666

顕著な変更（主観）は

- `MarshalJSONV2`/`UnmarshalJSONV2` -> `MarshalJSONTo`/`UnmarshalJSONFrom`に改名
- `MarshalJSONTo`/`UnmarshalJSONFrom`がoptionsを受け取らなくなった。
  - 代わりに`jsontext.Encoder`/`jsontext.Deocder`が`Options`を返すことができるように。
- `jsontext.Encoder`/`jsontext.Decoder`の`StackPointer`が`string`の代わりに`jsontext.Pointer`を返すように。
- `jsontext.ObjectStart`/`jsontext.ObjectEnd` -> `jsontext.BeginObject`/`jsontext.EndObject`に変更

その他いろいろ追加されています。

## 使ってみる

### v2.Marshal / v2.Unmarshal

`Marshal`/`Unmarshal`の使用感はあまり変わりません。

ちょっとしたコツとして、`v2.Marshal`に渡す値は必ずaddressableなもの、つまりポインターにします。
しない場合、`v2`は一旦値をコピーしてポインターに変換しなおします。これはnon-addressableな値だとmethod receiverがpointerだと`reflect`経由では呼び出しができないためです。

こうしておくほうが若干パフォーマンスが良いです。

```go
//go:build (go1.25 && goexperiment.jsonv2) || go1.26

package main

import (
    "encoding/json/jsontext"
    "encoding/json/v2"
    "fmt"
    "time"
)

type A struct {
    Foo string    `json:"foo,omitzero"`
    Bar int       `json:"int,omitzero"`
    T   time.Time `json:"t,omitzero,format:RFC3339"`
    U   string    `json:"',\"'"`
}

func main() {
    a := A{
        Foo: "foo",
        Bar: 123,
        T:   time.Now(),
        U:   "um",
    }

    bin, err := json.Marshal(&a, jsontext.WithIndent("    "))
    if err != nil {
        panic(err)
    }
    fmt.Println(string(bin))
    /*
       {
           "foo": "foo",
           "int": 123,
           "t": "2025-05-23T21:47:23+09:00",
           ",\"": "um"
       }
    */
    a = *new(A)
    err = json.Unmarshal(bin, &a)
    if err != nil {
        panic(err)
    }
    fmt.Printf("%#v\n", a)
    // main.A{Foo:"foo", Bar:123, T:time.Date(2025, time.May, 23, 21, 47, 23, 0, time.Local), U:"um"}
}
```

### jsontext.Encoder

`*jsontext.Encoder`は[*json.Encoder]に当たるもので、言葉通りJSONのTokenやValueを`io.Writer`に書き込むことができます。

`WriteToken`、`WriteValue`で値を書き込みますが、内部のステートマシンが状態を覚えているので`:`とか`,`とかを手動で書き込む必要はないです。これはいいデザインですね。

`StackDepth`,`StackIndex`,`StackPointer`は`jsontext.Decoder`と共通なmethodでそれぞれ、

- 現在のJSON ObjectやJSON Arrayのnest回数
- i番目の階層の開始token: `0`, `{`, `[`のどれか
- JSON Pointer([RFC 6901])

が取得できます。

`UnusedBuffer`で`encoderState`に紐づくバッファーが利用できるので、これを利用するとよいというAPIのようです。内部のコメントを見ると`encoderState`のバッファーの未使用の部分をsliceで返すような実装をしていたけどやめたようなことがコメントで書かれています。見た限りずっとこのコメントが残されています。proposalになる時点でもこのmethod名が変わらなかったので実装されるときもこのままかもしれないですね。

[sninnet](https://github.com/ngicks/go-play-encoding-json-v2/blob/main/play/encoder_test.go)

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

    bufErr(enc.WriteToken(jsontext.BeginArray))
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
    t.Log(enc.StackIndex(2)) // encoder_test.go:54: [ 4

    bufErr(enc.WriteToken(jsontext.EndArray))
    assertDepth(enc, 1)
    bufErr(enc.WriteToken(jsontext.EndObject))
    assertDepth(enc, 0)

    if err != nil {
        panic(err)
    }
    expected := `{
    "foo": null,
    "baz": [
        "qux",
        123,
        "quux",
        [
            {
                "corge": null
            }
        ]
    ]
}
`
    if buf.String() != expected {
        t.Fatalf("not equal:\nexpected = %s\nactual  = %s", expected, buf.String())
    }
}
```

### jsontext.Decoder

`*jsontext.Decoder`は[*json.Decoder]に当たるもので、言葉通りJSON TokenやValueを`io.Reader`から読み込むことができます。

`PeekKind`で値を消費せずに`jsontext.Kind`を取得し、`ReadToken`, `ReadValue`で値を読み込みます。

`StackDepth`,`StackIndex`,`StackPointer`は`jsontext.Encoder`と共通なmethodでそれぞれ、

- 現在のJSON ObjectやJSON Arrayのnest回数
- i番目の階層の開始token: `0`, `{`, `[`のどれか
- JSON Pointer([RFC 6901](https://datatracker.ietf.org/doc/html/rfc6901))

が取得できます。

[snippet](https://github.com/ngicks/go-play-encoding-json-v2/blob/main/play/decoder_test.go)

```go
import (
    "bytes"
    "encoding/json"
    "encoding/json/jsontext"
    "io"
    "strings"
    "testing"
)

func TestDecoder(t *testing.T) {
    const input = `{
    "foo": null,
    "baz": [
        "qux",
        123,
        "quux",
        [
            {
                "corge": null
            }
        ]
    ]
}
`
    dec := jsontext.NewDecoder(strings.NewReader(input))

    expected := []any{
        jsontext.BeginObject,
        jsontext.String("foo"),
        jsontext.Null,
        jsontext.String("baz"),
        "peek",
        jsontext.BeginArray,
        jsontext.String("qux"),
        jsontext.Int(123),
        "peek",
        jsontext.String("quux"),
        jsontext.Value(`[{"corge":null}]`),
        jsontext.EndArray,
        jsontext.EndObject,
    }

    for _, tokenOrValue := range expected {
        idxKind, valueLen := dec.StackIndex(dec.StackDepth())
        t.Logf("depth = %d, index kind = %s, len at index = %d, stack pointer = %q", dec.StackDepth(), idxKind, valueLen, dec.StackPointer())
        /*
           decoder_test.go:46: depth = 0, index kind = <invalid jsontext.Kind: '\x00'>, len at index = 0, stack pointer = ""
           decoder_test.go:46: depth = 1, index kind = {, len at index = 0, stack pointer = ""
           decoder_test.go:46: depth = 1, index kind = {, len at index = 1, stack pointer = "/foo"
           decoder_test.go:46: depth = 1, index kind = {, len at index = 2, stack pointer = "/foo"
           decoder_test.go:46: depth = 1, index kind = {, len at index = 3, stack pointer = "/baz"
           decoder_test.go:46: depth = 1, index kind = {, len at index = 3, stack pointer = "/baz"
           decoder_test.go:46: depth = 2, index kind = [, len at index = 0, stack pointer = "/baz"
           decoder_test.go:46: depth = 2, index kind = [, len at index = 1, stack pointer = "/baz/0"
           decoder_test.go:46: depth = 2, index kind = [, len at index = 2, stack pointer = "/baz/1"
           decoder_test.go:46: depth = 2, index kind = [, len at index = 2, stack pointer = "/baz/1"
           decoder_test.go:46: depth = 2, index kind = [, len at index = 3, stack pointer = "/baz/2"
           decoder_test.go:46: depth = 2, index kind = [, len at index = 4, stack pointer = "/baz/3"
           decoder_test.go:46: depth = 1, index kind = {, len at index = 4, stack pointer = "/baz"
        */
        switch x := tokenOrValue.(type) {
        case string:
            t.Logf("peek = %s", dec.PeekKind())
        /*
           decoder_test.go:62: peek = [
           decoder_test.go:62: peek = string
        */
        case jsontext.Token:
            tok, err := dec.ReadToken()
            if err != nil && err != io.EOF {
                panic(err)
            }
            if tok.Kind() != x.Kind() {
                t.Errorf("not equal: expected(%v) != actual(%v)", x, tok)
            }
            switch tok.Kind() {
            case 'n': // null
            case 'f': // false
            case 't': // true
            case '"', '0': // string literal, number literal
                if tok.String() != x.String() {
                    t.Errorf("not equal: expected(%s) != actual(%s)", x, tok)
                }
            case '{': // end object
            case '}': // end object
            case '[': // begin array
            case ']': // end array
            }
        case jsontext.Value:
            val, err := dec.ReadValue()
            if err != nil && err != io.EOF {
                panic(err)
            }

            if !bytes.Equal(mustCompact(val), mustCompact(x)) {
                t.Errorf("not equal: expected(%q) != actual(%q)", string(x), string(val))
            }
        }
    }
}

func mustCompact(v jsontext.Value) jsontext.Value {
    err := v.Compact()
    if err != nil {
        panic(err)
    }
    return v
}
```

### jsontext.Decoder.StackPointer

`jsontext.Deocder.StackPointer`を活用すれば特定の`JSON Pointer`まで値を読み捨ててdecodeをするみたいなこともできます。

[snippet](https://github.com/ngicks/go-play-encoding-json-v2/blob/main/play/decoder_pointer_test.go)

```go
import (
    "bytes"
    "encoding/json/jsontext"
    "encoding/json/v2"
    "errors"
    "fmt"
    "io"
    "iter"
    "strconv"
    "strings"
    "testing"
)

var ErrNotFound = errors.New("not found")

func ReadJSONAt(dec *jsontext.Decoder, pointer jsontext.Pointer, read func(dec *jsontext.Decoder) error) (err error) {
    lastToken := pointer.LastToken()
    var idx int64 = -1
    if len(lastToken) > 0 && strings.TrimLeftFunc(lastToken, func(r rune) bool { return '0' <= r && r <= '9' }) == "" {
        idx, err = strconv.ParseInt(lastToken, 10, 64)
        if err == nil {
            pointer = pointer[:len(pointer)-len(lastToken)-1]
        } else {
            // I'm not really super sure this could happen.
            idx = -1
        }
    }

    currentDepth := 0
    for {
        _, err = dec.ReadToken()
        if errors.Is(err, io.EOF) {
            break
        }
        if err != nil {
            return err
        }
        p := dec.StackPointer()
        if pointer == p {
            if idx >= 0 {
                if dec.PeekKind() != '[' {
                    return ErrNotFound
                }
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
            if dec.PeekKind() == ']' {
                return ErrNotFound
            }
            return read(dec)
        }
        nextDepth := commonSegment(p, pointer)
        if nextDepth < currentDepth {
            // search depth should only increase
            break
        }
        currentDepth = nextDepth
    }
    return ErrNotFound
}

func commonSegment(target, pointer jsontext.Pointer) int {
    if pointer.Contains(target) {
        return strings.Count(string(target), "/") + 1
    }
    next, stop := iter.Pull(target.Tokens())
    defer stop()
    common := 0
    for p := range pointer.Tokens() {
        t, ok := next()
        if !ok {
            break
        }
        if t != p {
            break
        }
        common++
    }
    return common
}

func TestDecoder_Pointer(t *testing.T) {
    jsonBuf := []byte(`{"yay":"yay","nay":[{"boo":"boo"},{"bobo":"bobo"}],"foo":{"bar":{"baz":"baz"}}}`)

    type Boo struct {
        Boo string `json:"boo"`
    }
    type Bobo struct {
        Bobo string `json:"bobo"`
    }
    type Baz struct {
        Baz string `json:"baz"`
    }

    type testCase struct {
        pointer    jsontext.Pointer
        readTarget any
        expected   any
    }
    for _, tc := range []testCase{
        {"/foo/bar", Baz{}, Baz{"baz"}},
        {"/nay/0", Boo{}, Boo{"boo"}},
        {"/nay/1", Bobo{}, Bobo{"bobo"}},
        {"/yay/2", nil, nil},
        {"/foo/bar/baz/qux", nil, nil},
        {"/nay/2", nil, nil},
    } {
        t.Run(string(tc.pointer), func(t *testing.T) {
            err := ReadJSONAt(
                jsontext.NewDecoder(bytes.NewBuffer(jsonBuf)),
                tc.pointer,
                func(dec *jsontext.Decoder) error {
                    return json.UnmarshalDecode(dec, &tc.readTarget)
                },
            )
            if tc.readTarget == nil {
                if err != ErrNotFound {
                    t.Errorf("should be ErrNotFound, but is %q", err)
                }
                return
            }
            if err != nil && err != io.EOF {
                panic(err)
            }
            expected := fmt.Sprintf("%#v", tc.expected)
            read := fmt.Sprintf("%#v", tc.readTarget)
            if expected != read {
                t.Errorf("read not as expected: expected(%q) != actual(%q)", expected, read)
            }
        })
    }
}
```

~~実用しようと思うならたとえば`/foo/bar/baz`が与えられた時に`/foo/bar`を読み終わって`/foo/other`に移行したときに探索をやめるようなコードが必要ですが、例なので実装していません。~~

EDIT(2025-05-14): 実用レベルかはともかく要素が見つからないと確定したら早々にreturnするようにしました。

### v2.MarshalerTo / v2.UnmarshalerFrom

何度もやってる`undefined | null | T`を表現できる型を`v2.MarshalerTo`/`v2.UnmarshalerFrom`で実装してみます。

[snippet](https://github.com/ngicks/go-play-encoding-json-v2/blob/main/play/arshaler_to_from_test.go)

```go
import (
    "encoding/json/jsontext"
    "encoding/json/v2"
    "testing"
)

var (
    _ json.MarshalerTo     = Option[any]{}
    _ json.UnmarshalerFrom = (*Option[any])(nil)
    _ json.MarshalerTo     = Und[any]{}
    _ json.UnmarshalerFrom = (*Und[any])(nil)
)

type Option[V any] struct {
    some bool
    v    V
}

func None[V any]() Option[V] {
    return Option[V]{}
}

func Some[V any](v V) Option[V] {
    return Option[V]{some: true, v: v}
}

func (o Option[V]) IsZero() bool {
    return o.IsNone()
}

func (o Option[V]) IsNone() bool {
    return !o.some
}

func (o Option[V]) IsSome() bool {
    return o.some
}

func (o Option[V]) Value() V {
    return o.v
}

func (o Option[V]) MarshalJSONTo(enc *jsontext.Encoder) error {
    if o.IsNone() {
        return enc.WriteToken(jsontext.Null)
    }
    return json.MarshalEncode(enc, o.Value())
}

func (o *Option[V]) UnmarshalJSONFrom(dec *jsontext.Decoder) error {
    if k := dec.PeekKind(); k == 'n' {
        err := dec.SkipValue()
        if err != nil {
            return err
        }
        o.some = false
        o.v = *new(V)
        return nil
    }
    var v V
    err := json.UnmarshalDecode(dec, &v)
    if err != nil {
        return err
    }
    // preventing half-baked value left in-case of error in middle of decode
    // sacrificing performance
    o.some = true
    o.v = v
    return nil
}

type Und[V any] struct {
    opt Option[Option[V]]
}

func Undefined[V any]() Und[V] {
    return Und[V]{}
}

func Null[V any]() Und[V] {
    return Und[V]{opt: Some(None[V]())}
}

func Defined[V any](v V) Und[V] {
    return Und[V]{opt: Some(Some(v))}
}

func (u Und[V]) IsZero() bool {
    return u.IsUndefined()
}

func (u Und[V]) IsUndefined() bool {
    return u.opt.IsNone()
}

func (u Und[V]) IsNull() bool {
    return u.opt.IsSome() && u.opt.Value().IsNone()
}

func (u Und[V]) IsDefined() bool {
    return u.opt.IsSome() && u.opt.Value().IsSome()
}

func (u Und[V]) Value() V {
    return u.opt.Value().Value()
}

func (u Und[V]) MarshalJSONTo(enc *jsontext.Encoder) error {
    if !u.IsDefined() {
        return enc.WriteToken(jsontext.Null)
    }
    return json.MarshalEncode(enc, u.Value())
}

func (u *Und[V]) UnmarshalJSONFrom(dec *jsontext.Decoder) error {
    // should be with omitzero which handles absence of field.
    if k := dec.PeekKind(); k == 'n' {
        err := dec.SkipValue()
        if err != nil {
            return err
        }
        *u = Null[V]()
        return nil
    }
    var v V
    err := json.UnmarshalDecode(dec, &v)
    if err != nil {
        return err
    }
    *u = Defined(v)
    return nil
}

func TestArshalerToFrom(t *testing.T) {
    type sample struct {
        // null or string
        Foo Option[string]
        // undefined or string
        Bar Option[int] `json:",omitzero"`
        // undefined | null | bool
        Baz Und[bool] `json:",omitzero"`
    }

    type testCase struct {
        in        sample
        marshaled string
    }
    for _, tc := range []testCase{
        {sample{}, `{"Foo":null}`},
        {sample{Some(""), Some(0), Null[bool]()}, `{"Foo":"","Bar":0,"Baz":null}`},
        {sample{Some("foo"), Some(5), Defined(false)}, `{"Foo":"foo","Bar":5,"Baz":false}`},
        {sample{None[string](), None[int](), Defined(true)}, `{"Foo":null,"Baz":true}`},
    } {
        t.Run(tc.marshaled, func(t *testing.T) {
            bin, err := json.Marshal(tc.in)
            if err != nil {
                panic(err)
            }
            if string(bin) != tc.marshaled {
                t.Errorf("not equal: expected(%q) != actual(%q)", tc.marshaled, string(bin))
            }
            var unmarshaled sample
            err = json.Unmarshal(bin, &unmarshaled)
            if err != nil {
                panic(err)
            }
            if unmarshaled != tc.in {
                t.Errorf("not euql:\nexpected(%#v)\n!=\nactual(%#v)", tc.in, unmarshaled)
            }
        })
    }
}
```

### `json:",format:value"`

`json:",format:value"`でフォーマットを指定できます。現状は[組み込まれた](https://pkg.go.dev/github.com/go-json-experiment/json#example-package-FormatFlags)特定のものしか指定できません。
ユーザー指定のformatを持てるようにしようというのは[別proposal](https://github.com/golang/go/issues/71664)になっています。最初の実装時には入ってこないかもしれないです。

[snippet](https://github.com/ngicks/go-play-encoding-json-v2/blob/main/play/tag_format_test.go)

```go
import (
    "encoding/json/v2"
    "testing"
    "time"
)

func TestArshalerFormat(t *testing.T) {
    type sample struct {
        Foo map[string]string `json:",format:emitempty"`
        Bar []byte            `json:",format:array"`
        Baz time.Duration     `json:",format:units"`
        Qux time.Time         `json:",format:'2006-01-02'"`
    }

    s := sample{
        Foo: nil,
        Bar: []byte(`bar`),
        Baz: time.Minute,
        Qux: time.Date(2025, 0o5, 12, 22, 23, 22, 123456789, time.UTC),
    }

    bin, err := json.Marshal(s)
    if err != nil {
        panic(err)
    }
    expected := `{"Foo":{},"Bar":[98,97,114],"Baz":"1m0s","Qux":"2025-05-12"}`
    if string(bin) != expected {
        t.Errorf("not equal:\n%s\n!=\n%s", expected, string(bin))
    }
}
```

### `json:",unknown"`でフォールバック

`json:",unknown"`でstruct fieldなどで指定されていない(=不明な)フィールドをすべて格納できます。これが欲しかった。

`json.DiscardUnknownMembers(true)`で無視、`json.RejectUnknownMembers(true)`でエラーに挙動を変更できます。

[snippet](https://github.com/ngicks/go-play-encoding-json-v2/blob/main/play/tag_unknown_test.go)

```go
import (
    "encoding/json/v2"
    "maps"
    "testing"
)

func TestTagUnknown(t *testing.T) {
    type sample struct {
        X   map[string]any `json:",unknown"`
        Foo string
        Bar int
        Baz bool
    }

    input := []byte(`{"Foo":"foo","Bar":12,"Baz":true,"Qux":"qux","Quux":"what!?"}`)
    var s sample
    err := json.Unmarshal(input, &s)
    if err != nil {
        panic(err)
    }
    expected := map[string]any{"Qux": "qux", "Quux": "what!?"}
    if !maps.Equal(s.X, expected) {
        t.Errorf("not equal:\n%#v\n!=\n%#v", expected, s.X)
    }

    err = json.Unmarshal(input, &s, json.RejectUnknownMembers(true))
    if err == nil {
        t.Error("should cause an error")
    } else {
        t.Logf("%v", err)
        // tag_unknown_test.go:32: json: cannot unmarshal JSON string into Go play.sample: unknown object member name "Qux"
    }
}
```

### streaming decode

`v1`でやっていたようにstreaming deocdeも可能です。

[snippet](https://github.com/ngicks/go-play-encoding-json-v2/blob/main/play/streaming_decode_test.go#L12-L63)

```go
import (
    "bytes"
    "encoding/json/jsontext"
    "encoding/json/v2"
    "io"
    "reflect"
    "testing"
)

const streamDecodeInput = `{
    "foo": null,
    "bar": {
            "baz": [
                {"foo":"foo1"},
                {"foo":"foo2"},
                {"foo":"foo3"}
            ]
        }
}
`

func TestStreamingDecode(t *testing.T) {
    dec := jsontext.NewDecoder(bytes.NewReader([]byte(streamDecodeInput)))
    for dec.StackPointer() != jsontext.Pointer("/bar/baz") {
        _, err := dec.ReadToken()
        if err != nil {
            if err != io.EOF {
                panic(err)
            }
            break
        }
    }

    if dec.PeekKind() != '[' {
        panic("not array")
    }
    // discard '['
    _, err := dec.ReadToken()
    if err != nil {
        panic(err)
    }
    type sample struct {
        Foo string `json:"foo"`
    }
    var decoded []sample
    for dec.PeekKind() != ']' {
        var s sample
        err := json.UnmarshalDecode(dec, &s)
        if err != nil {
            panic(err)
        }
        decoded = append(decoded, s)
    }
    expected := []sample{{"foo1"}, {"foo2"}, {"foo3"}}
    if !reflect.DeepEqual(expected, decoded) {
        t.Errorf("not equal:\nexpected(%#v)\n!=\nactual(%#v)", expected, decoded)
    } else {
        t.Logf("decoded = %#v", decoded)
        // streaming_decode_test.go:60: decoded = []play.sample{play.sample{Foo:"foo1"}, play.sample{Foo:"foo2"}, play.sample{Foo:"foo3"}}
    }
}
```

### streaming deocde 2

`v2.WithUnmarshalers(v2.UnmarshalFromFunc(func (...) {...}))`で型ごとにunmarshalerを変更できます。
これを利用すればもっと簡単に(と言いつつコードはごちゃごちゃしますが)streaming decodeを行うことができます。

[snippet](https://github.com/ngicks/go-play-encoding-json-v2/blob/main/play/streaming_decode_test.go#L65-L115)

```go
import (
    "bytes"
    "encoding/json/jsontext"
    "encoding/json/v2"
    "io"
    "reflect"
    "testing"
)

const streamDecodeInput = `{
    "foo": null,
    "bar": {
            "baz": [
                {"foo":"foo1"},
                {"foo":"foo2"},
                {"foo":"foo3"}
            ]
        }
}
`

func TestStreamingDecode2(t *testing.T) {
    type Data struct {
        Foo string `json:"foo"`
    }

    type Bar struct {
        Baz []Data `json:"baz"`
    }

    type sample struct {
        Foo *int `json:"foo"`
        Bar Bar  `json:"bar"`
    }

    dataChan := make(chan Data)
    unmarshaler := json.WithUnmarshalers(json.UnmarshalFromFunc(func(dec *jsontext.Decoder, d *Data) error {
        type plain Data
        var p plain
        err := json.UnmarshalDecode(dec, &p)
        if err == nil {
            dataChan <- Data(p)
        }
        *d = Data(p)
        return err
    }))

    resultCh := make(chan []Data)
    go func() {
        var result []Data
        for d := range dataChan {
            result = append(result, d)
        }
        resultCh <- result
    }()

    var s sample
    err := json.Unmarshal([]byte(streamDecodeInput), &s, unmarshaler)
    if err != nil {
        panic(err)
    }
    close(dataChan)
    result := <-resultCh

    expected := []Data{{"foo1"}, {"foo2"}, {"foo3"}}
    if !reflect.DeepEqual(expected, result) {
        t.Errorf("not equal:\nexpected(%#v)\n!=\nactual(%#v)", expected, result)
    } else {
        t.Logf("decoded = %#v", result)
        // streaming_decode_test.go:111: decoded = []play.Data{play.Data{Foo:"foo1"}, play.Data{Foo:"foo2"}, play.Data{Foo:"foo3"}}
    }
}
```

## まだ足りてなさそうなところ

### token列からのunmarshalは簡単じゃなさそう。

`encoding/xml`には[xml.NewTokenDecoder]がありますが、`encoding/json/v2`にはこういったtoken readerがないため効率的なtee-ingができないかもしれないです。

ということで下記の`Either[L, R]`を例に出します。jsonからunmarshalするとき、左で成功すれば左、だめなら右でunmarshal、どっちかで成功すればよいというものです。tokenのtee-ingができないと一旦`JSON Value`をバッファーする必要があり、これは`v1`の[json.Unmarshaler]と同様にパフォーマンスが悪そうに思えます。こちらは`v2`の`Options`を伝搬できる違いがあり、実装が無意味というわけでもないです。

とはいえ下記のように`jsontext.Encoder`/`jsontext.Decoder`がinterfaceになることはないでしょうから当面(もしくはずっと)token列からのunmarshalはできないと思われます。

> - Make Encoder and Decoder an interface: The json.MarshalerTo and json.UnmarshalerFrom interfaces reference a concrete jsontext.Encoder and jsontext.Decoder implementation, which prevents use of a customer encoder or decoder. We considered making these an interface, but the performance cost of constantly calling a virtual method was expensive when a vast majority of usages are for the standard implementation.

追記(2025-05-13): ちょっと考えるとteeingもできたので追記しました。

#### Value-buffer版

[snippet](https://github.com/ngicks/go-play-encoding-json-v2/blob/main/play/arshaler_either_test.go)

```go
import (
    "encoding/json/jsontext"
    "encoding/json/v2"
    "fmt"
    "testing"
)

var (
    _ json.MarshalerTo     = Either[any, any]{}
    _ json.UnmarshalerFrom = (*Either[any, any])(nil)
)

// zero value is zero left.
type Either[L, R any] struct {
    isRight bool
    l       L
    r       R
}

func (e Either[L, R]) IsLeft() bool {
    return !e.isRight
}

func (e Either[L, R]) MarshalJSONTo(enc *jsontext.Encoder) error {
    if e.IsLeft() {
        return json.MarshalEncode(enc, e.Left())
    }
    return json.MarshalEncode(enc, e.Right())
}

func (e *Either[L, R]) UnmarshalJSONFrom(dec *jsontext.Decoder) error {
    val, err := dec.ReadValue()
    if err != nil {
        return err
    }

    var l L
    errL := json.Unmarshal(val, &l, dec.Options())
    if errL == nil {
        e.isRight = false
        e.l = l
        e.r = *new(R)
        return nil
    }

    var r R
    errR := json.Unmarshal(val, &r, dec.Options())
    if errR == nil {
        e.isRight = true
        e.l = *new(L)
        e.r = r
        return nil
    }

    return fmt.Errorf("Either[L, R]: unmarshal failed for both L and R: l = (%w), r = (%w)", errL, errR)
}

func TestArshalerEither(t *testing.T) {
    type testCase struct {
        in   string
        fail bool
    }
    for _, tc := range []testCase{
        {"\"foo\"", false},
        {"123", false},
        {"false", true},
    } {
        var e Either[string, int]
        err := json.Unmarshal([]byte(tc.in), &e)
        if (err != nil) != tc.fail {
            t.Errorf("incorrect!")
        }
        t.Logf("err = %v", err)
        /*
           arshaler_either_test.go:122: err = <nil>
           arshaler_either_test.go:122: err = <nil>
           arshaler_either_test.go:122: err = json: cannot unmarshal into Go play.Either[string,int]: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON boolean into Go string), r = (json: cannot unmarshal JSON boolean into Go int)
        */
    }
}
```

:::details 残りのmethod

```go
func Left[L, R any](l L) Either[L, R] {
    return Either[L, R]{isRight: false, l: l}
}

func Right[L, R any](r R) Either[L, R] {
    return Either[L, R]{isRight: true, r: r}
}


func (e Either[L, R]) IsRight() bool {
    return e.isRight
}

func (e Either[L, R]) Left() L {
    return e.l
}

func (e Either[L, R]) Right() R {
    return e.r
}

func (e Either[L, R]) Unpack() (L, R) {
    // for ? syntax discussed under https://github.com/golang/go/discussions/71460
    return e.l, e.r
}

func MapLeft[L, R, L2 any](e Either[L, R], mapper func(l L) L2) Either[L2, R] {
    if e.IsLeft() {
        return Left[L2, R](mapper(e.Left()))
    }
    return Right[L2](e.Right())
}

func (e Either[L, R]) MapLeft(mapper func(l L) L) Either[L, R] {
    return MapLeft(e, mapper)
}

func MapRight[L, R, R2 any](e Either[L, R], mapper func(l R) R2) Either[L, R2] {
    if e.IsRight() {
        return Right[L](mapper(e.Right()))
    }
    return Left[L, R2](e.Left())
}

func (e Either[L, R]) MapRight(mapper func(l R) R) Either[L, R] {
    return MapRight(e, mapper)
}
```

:::

#### tee-ing版

すごく迂遠なことをすればtokenのtee-ingが実装できます。

- `PeekKind`でJSON ObjectかJSON Arrayのときと、それ以外のときで分岐します。
  - `null`, `false`, `true`, `""`(String literal), `0`(Number literal)のいずれかである場合はどちらにせよValue単位の読み込みになるため、`ReadValue`を読んでunmarshalしておきます。
- JSON ObjectかJSON Arrayのとき、まず初めに`StackDepth`をとっておきます。`jsontext.Decoder.ReadToken`すると`{`か`[`を読むことでdepthが増加するはずですので、元のdepthに戻ったときそののJSON ObjectかJSON Arrayが読み終わったとみなせます。
- 二つ[io.Pipe]を作り、自家版[io.MultiWriter]みたいなものでwriter側を結合して1つの[io.Writer]にし、`*jsontext.Encoder`を作成します。
- `*jsontext.Decoder.ReadToken`で読み込んだtokenをこのencoderに書き込みます。
  - `ReadToken`も`ReadValue`も`,`や`:`の情報がドロップしてしまうため、これを完全に再建するためには`*jsontext.Encoder`の力が必要です。
- [io.MultiWriter]をそのまま使えないのは、片方の`v2.UnmarshalRead`が早期にエラー終了したとき、こちらには書き込まなくてよくなるのでそれをpipeのwriter側に伝えたいが、これに`CloseWithError`以外のうまい方法がないためです。
- あとはどこかのgoroutineがpanicした時に備えてrecoverしてre-panicできるように少し考慮を加えて完成です。

ものすごい大きなJSON Objectとかでない限りValue-buffer版のほうが効率いいと思うので微妙な気持ちになりますが、筆者が思いつけるのはここが限界です。
もっと賢い方法があったら教えてほしいです。

[snippet](https://github.com/ngicks/go-play-encoding-json-v2/blob/main/play/either_teeing/arshaler_either_test.go)

つまり以下のように`TeeDecoder`を定義します。

`v2.Unmarshal`が終わると`*jsontext.Decoder`は`v2`が持っているキャッシュプールに戻されるため、呼び出しが終わった後もdecoderを保持し続けるとrace conditionが生じます。
`TeeDecoder`は内部で作ったgoroutineが全部終了するようしっかりsyncをとる必要があります。

```go
type ReadCloseStopper interface {
    io.ReadCloser
    Stop(successful bool) // when called with true, stop tee-ing of both side. Otherwise stops the calling side.
}

type bufReader struct {
    mu     sync.Mutex
    closed bool
    r      *bytes.Reader
}

func (r *bufReader) Read(p []byte) (int, error) {
    r.mu.Lock()
    defer r.mu.Unlock()

    if r.closed {
        return 0, io.EOF
    }
    return r.r.Read(p)
}

func (r *bufReader) Close() error {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.closed = true
    return nil
}

func (r *bufReader) Stop(successful bool) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.closed = true
}

var (
    errStopped     = errors.New("stopped")
    errFailedEarly = errors.New("failed early")
)

type teeReader struct {
    mu     sync.Mutex
    closed bool
    r      *io.PipeReader
}

func (r *teeReader) Read(p []byte) (int, error) {
    r.mu.Lock()
    defer r.mu.Unlock()

    if r.closed {
        return 0, io.EOF
    }

    return r.r.Read(p)
}

func (r *teeReader) Close() error {
    r.mu.Lock()
    defer r.mu.Unlock()

    if r.closed {
        return nil
    }
    r.closed = true

    return r.r.Close()
}

func (r *teeReader) Stop(successful bool) {
    r.mu.Lock()
    defer r.mu.Unlock()

    if r.closed {
        return
    }
    r.closed = true

    err := errFailedEarly
    if successful {
        err = errStopped
    }
    r.r.CloseWithError(err)
}

type multiPipeWriter struct {
    maskedErr error
    wl, wr    *io.PipeWriter
}

func (w *multiPipeWriter) Write(b []byte) (n int, err error) {
    if w.wl != nil {
        var nl int
        nl, err = w.wl.Write(b)
        if err != nil {
            // failed write = the other side of pipe is closed with error or anything.
            w.wl = nil
            if !errors.Is(err, w.maskedErr) {
                w.wr.CloseWithError(err)
                w.wr = nil
                return
            }
        } else {
            n = nl
        }
        err = nil
    }

    if w.wr != nil {
        var nr int
        nr, err = w.wr.Write(b)
        if err != nil {
            w.wr = nil
            if !errors.Is(err, w.maskedErr) {
                w.wl.CloseWithError(err)
                w.wl = nil
                return
            }
        } else {
            n = nr
        }
        err = nil
    }
    if len(b) != n && err == nil {
        err = io.ErrClosedPipe
    }
    return
}

func (w *multiPipeWriter) CloseWithError(err error) error {
    if w.wl != nil {
        w.wl.CloseWithError(err)
        w.wl = nil
    }
    if w.wr != nil {
        w.wr.CloseWithError(err)
        w.wr = nil
    }
    return nil
}

var errBadDec = errors.New("bad decoder")

func TeeDecoder(dec *jsontext.Decoder, encOptions ...jsontext.Options) (l ReadCloseStopper, r ReadCloseStopper, wait func(), err error) {
    switch dec.PeekKind() {
    default:
        return nil, nil, func() {}, fmt.Errorf("%w: decoder peeked a non starting token %q", errBadDec, dec.PeekKind().String())
    case 'n', 'f', 't', '"', '0':
        val, err := dec.ReadValue()
        if err != nil {
            return nil, nil, func() {}, err
        }
        return &bufReader{r: bytes.NewReader(val)}, &bufReader{r: bytes.NewReader(val)}, func() {}, nil
    case '[', '{':
        prl, pwl := io.Pipe()
        prr, pwr := io.Pipe()

        var (
            wg       sync.WaitGroup
            panicVal any
        )
        wg.Add(1)
        go func() {
            defer wg.Done()
            var err error
            mw := &multiPipeWriter{errFailedEarly, pwl, pwr}
            defer func() {
                // it's possible that reading dec panicks
                if rec := recover(); rec != nil {
                    panicVal = rec
                    err = fmt.Errorf("panicked: %v", rec)
                }
                mw.CloseWithError(err)
            }()

            enc := jsontext.NewEncoder(mw, encOptions...)

            depth := dec.StackDepth()
            var tok jsontext.Token
            for {
                tok, err = dec.ReadToken()
                if err != nil {
                    return
                }

                err = enc.WriteToken(tok)
                if err != nil {
                    return
                }

                if dec.StackDepth() == depth {
                    break
                }
            }
        }()

        wait = func() {
            wg.Wait()
            if panicVal != nil {
                panic(panicVal)
            }
        }
        return &teeReader{r: prl}, &teeReader{r: prr}, wait, nil
    }
}
```

上記の`TeeDecoder`を用いると`UnmarshalJSONFrom`は下記のようになります。

```go
func (e *Either[L, R]) UnmarshalJSONFrom(dec *jsontext.Decoder) error {
    eitherErr := func(errL, errR error) error {
        return fmt.Errorf("Either[L, R]: unmarshal failed for both L and R: l = (%w), r = (%w)", errL, errR)
    }
    switch dec.PeekKind() {
    case 'n', 'f', 't', '"', '0':
        val, err := dec.ReadValue()
        if err != nil {
            return err
        }

        var l L
        errL := json.Unmarshal(val, &l, dec.Options())
        if errL == nil {
            e.isRight = false
            e.l = l
            e.r = *new(R)
            return nil
        }

        var r R
        errR := json.Unmarshal(val, &r, dec.Options())
        if errR == nil {
            e.isRight = true
            e.l = *new(L)
            e.r = r
            return nil
        }

        return eitherErr(errL, errR)
    case '{', '[': // maybe deep and large
        var wg sync.WaitGroup
        defer wg.Wait() // in case of panic

        var err error

        rl, rr, wait, err := TeeDecoder(dec)
        if err != nil {
            return err
        }
        defer func() {
            rl.Stop(false)
            rr.Stop(false)
            wait()
        }()

        var (
            l          L
            r          R
            errL, errR error
            panicVal   any
        )

        wg.Add(1)
        go func() {
            defer func() {
                if rec := recover(); rec != nil {
                    panicVal = rec
                }
                rr.Stop(false)
                wg.Done()
            }()
            errR = json.UnmarshalRead(rr, &r, dec.Options())
        }()

        errL = json.UnmarshalRead(rl, &l, dec.Options())
        rl.Stop(errL == nil)

        wg.Wait()
        if panicVal != nil {
            panic(panicVal)
        }

        if errL == nil {
            e.isRight = false
            e.l = l
            e.r = *new(R)
            return nil
        }

        if errR == nil {
            e.isRight = true
            e.l = *new(L)
            e.r = r
            return nil
        }

        return eitherErr(errL, errR)
    default: // invalid, '}',    ']'
        // syntax error
        _, err := dec.ReadValue()
        return err
    }
}
```

下記のようにテストを定義して挙動を確かめます。
`go test -count=100 -timeout=5s -race ./play/either_teeing/`してみていますがPASSしているのできちんとsyncできているようです。

```go
func TestArshalerEither(t *testing.T) {
    type testCase struct {
        in   string
        fail bool
    }
    for _, tc := range []testCase{
        {"\"foo\"", false},
        {"123", false},
        {"false", true},
        {"{\"foo\": false}", true},
    } {
        t.Run(tc.in, func(t *testing.T) {
            var e Either[string, int]
            err := json.Unmarshal([]byte(tc.in), &e)
            if (err != nil) != tc.fail {
                t.Errorf("incorrect!")
            }
            t.Logf("err = %v", err)
            /*
               === RUN   TestArshalerEither
               === RUN   TestArshalerEither/"foo"
                   arshaler_either_test.go:402: err = <nil>
               === RUN   TestArshalerEither/123
                   arshaler_either_test.go:402: err = <nil>
               === RUN   TestArshalerEither/false
                   arshaler_either_test.go:402: err = json: cannot unmarshal into Go play.Either[string,int]: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON boolean into Go string), r = (json: cannot unmarshal JSON boolean into Go int)
               === RUN   TestArshalerEither/{"foo":_false}
                   arshaler_either_test.go:402: err = json: cannot unmarshal into Go play.Either[string,int] after offset 13: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON object into Go string), r = (json: cannot unmarshal JSON object into Go int)
            */
        })
    }

    type sampleL struct {
        Foo []int
    }
    type sampleR struct {
        Bar map[string]string
    }
    for _, tc := range []testCase{
        {"\"foo\"", true},
        {"123", true},
        {"false", true},
        {"{\"foo\": false}", true},
        {"{\"Foo\": false}", true},
        {"{\"Foo\": [1,2,3]}", false},
        {"{\"Bar\": {\"foo\":\"foofoo\",\"bar\":\"barbar\"}}", false},
        {"{\"Foo\": [1,2,3}", true},     // syntax error
        {"{\"Bar\": {\"foo\":}}", true}, // syntax error
    } {
        t.Run(tc.in, func(t *testing.T) {
            var e Either[sampleL, sampleR]
            err := json.Unmarshal([]byte(tc.in), &e, json.RejectUnknownMembers(true))
            if (err != nil) != tc.fail {
                t.Errorf("incorrect!")
            }
            t.Logf("err = %v", err)
            /*
               === RUN   TestArshalerEither/"foo"#01
                   arshaler_either_test.go:440: err = json: cannot unmarshal into Go struct: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON string into Go play.sampleL), r = (json: cannot unmarshal JSON string into Go play.sampleR)
               === RUN   TestArshalerEither/123#01
                   arshaler_either_test.go:440: err = json: cannot unmarshal into Go struct: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON number into Go play.sampleL), r = (json: cannot unmarshal JSON number into Go play.sampleR)
               === RUN   TestArshalerEither/false#01
                   arshaler_either_test.go:440: err = json: cannot unmarshal into Go struct: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON boolean into Go play.sampleL), r = (json: cannot unmarshal JSON boolean into Go play.sampleR)
               === RUN   TestArshalerEither/{"foo":_false}#01
                   arshaler_either_test.go:440: err = json: cannot unmarshal into Go struct after offset 13: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON string into Go play.sampleL: unknown object member name "foo"), r = (json: cannot unmarshal JSON string into Go play.sampleR: unknown object member name "foo")
               === RUN   TestArshalerEither/{"Foo":_false}
                   arshaler_either_test.go:440: err = json: cannot unmarshal into Go struct after offset 13: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON boolean into Go []int within "/Foo"), r = (json: cannot unmarshal JSON string into Go play.sampleR: unknown object member name "Foo")
               === RUN   TestArshalerEither/{"Foo":_[1,2,3]}
                   arshaler_either_test.go:440: err = <nil>
               === RUN   TestArshalerEither/{"Bar":_{"foo":"foofoo","bar":"barbar"}}
                   arshaler_either_test.go:440: err = <nil>
               === RUN   TestArshalerEither/{"Foo":_[1,2,3}
                   arshaler_either_test.go:440: err = json: cannot unmarshal into Go struct within "/Foo": Either[L, R]: unmarshal failed for both L and R: l = (jsontext: read error: jsontext: invalid character '}' after array element (expecting ',' or ']') within "/Foo" after offset 14), r = (json: cannot unmarshal JSON string into Go play.sampleR: unknown object member name "Foo")
               === RUN   TestArshalerEither/{"Bar":_{"foo":}}
                   arshaler_either_test.go:440: err = json: cannot unmarshal into Go struct within "/Bar/foo": Either[L, R]: unmarshal failed for both L and R: l = (jsontext: read error: jsontext: missing value after object name within "/Bar/foo" after offset 15), r = (jsontext: read error: jsontext: missing value after object name within "/Bar/foo" after offset 15)
            */
        })
    }
}

```

`panic`が伝搬できてるかもテストしておきましょう。

```go
var panicVal any = "panicVal"

type panicReader struct {
    after io.Reader
    val   any
}

func (r *panicReader) Read(p []byte) (int, error) {
    n, err := r.after.Read(p)
    if err == io.EOF {
        panic(r.val)
    }
    return n, err
}

var _ json.UnmarshalerFrom = (*panicDecoder)(nil)

type panicDecoder struct{}

func (d *panicDecoder) UnmarshalJSONFrom(dec *jsontext.Decoder) error {
    panic(panicVal)
}

func TestArshalerEither_panic(t *testing.T) {
    t.Run("reader panics", func(t *testing.T) {
        defer func() {
            rec := recover()
            if rec != panicVal {
                t.Errorf("incorrect panic: want(%v) != got(%v)", panicVal, rec)
            }
        }()
        var e Either[map[int]any, map[string]any]
        json.UnmarshalRead(&panicReader{strings.NewReader(`{"foo":"foo",`), panicVal}, &e)
    })
    t.Run("left panics", func(t *testing.T) {
        defer func() {
            rec := recover()
            if rec != panicVal {
                t.Errorf("incorrect panic: want(%v) != got(%v)", panicVal, rec)
            }
        }()
        var e Either[panicDecoder, map[string]any]
        json.Unmarshal([]byte(`{"foo":"foo","bar":"bar"}`), &e)
    })
    t.Run("right panics", func(t *testing.T) {
        defer func() {
            rec := recover()
            if rec != panicVal {
                t.Errorf("incorrect panic: want(%v) != got(%v)", panicVal, rec)
            }
        }()
        var e Either[map[string]any, panicDecoder]
        json.Unmarshal([]byte(`{"foo":"foo","bar":"bar"}`), &e)
    })
}
```

:::details 前の

[snippet](https://github.com/ngicks/go-play-encoding-json-v2/blob/091868eb177d471bbd21455207ecee7eb174683a/play/either_teeing/arshaler_either_test.go)

```go
func (e *Either[L, R]) UnmarshalJSONFrom(dec *jsontext.Decoder) error {
    eitherErr := func(errL, errR error) error {
        return fmt.Errorf("Either[L, R]: unmarshal failed for both L and R: l = (%w), r = (%w)", errL, errR)
    }
    switch dec.PeekKind() {
    case 'n', 'f', 't', '"', '0':
        val, err := dec.ReadValue()
        if err != nil {
            return err
        }

        var l L
        errL := json.Unmarshal(val, &l, dec.Options())
        if errL == nil {
            e.isRight = false
            e.l = l
            e.r = *new(R)
            return nil
        }

        var r R
        errR := json.Unmarshal(val, &r, dec.Options())
        if errR == nil {
            e.isRight = true
            e.l = *new(L)
            e.r = r
            return nil
        }

        return eitherErr(errL, errR)
    case '{', '[': // maybe deep and large
        var wg sync.WaitGroup
        defer wg.Wait() // in case of panic

        prl, pwl := io.Pipe()
        prr, pwr := io.Pipe()
        defer func() {
            prl.Close()
            prr.Close()
        }()

        var (
            panicVal  any
            storeOnce sync.Once
        )
        recoverStoreOnce := func() {
            if rec := recover(); rec != nil {
                storeOnce.Do(func() {
                    panicVal = rec
                })
            }
        }

        errUnmarshalFailedEarly := errors.New("unmarshal failed early")
        errLeftSuccessful := errors.New("left successful")
        wg.Add(1)
        go func() {
            var err error
            defer func() {
                recoverStoreOnce()
                pwl.CloseWithError(err)
                pwr.CloseWithError(err)
                wg.Done()
            }()

            encl := jsontext.NewEncoder(pwl)
            encr := jsontext.NewEncoder(pwr)

            depth := dec.StackDepth()
            for {
                var tok jsontext.Token
                tok, err = dec.ReadToken()
                if err != nil {
                    return
                }

                errL := encl.WriteToken(tok)
                if errL != nil && !errors.Is(errL, errUnmarshalFailedEarly) {
                    err = errL
                    return
                }
                errR := encr.WriteToken(tok)
                if errR != nil && !errors.Is(errR, errUnmarshalFailedEarly) {
                    err = errR
                    return
                }
                if errL != nil && errR != nil {
                    err = errUnmarshalFailedEarly
                    return
                }

                if dec.StackDepth() == depth {
                    break
                }
            }
        }()

        var (
            l          L
            r          R
            errL, errR error
        )

        wg.Add(1)
        go func() {
            closed := false
            defer func() {
                recoverStoreOnce()
                if !closed {
                    prl.Close()
                }
                wg.Done()
            }()

            errL = json.UnmarshalRead(prl, &l, dec.Options())
            if errL != nil { // successful = tokens are fully consumed
                prl.CloseWithError(errUnmarshalFailedEarly)
            } else { // If decoding left succeeded, right is no longer needed.
                prr.CloseWithError(errLeftSuccessful)
            }
            closed = true
        }()

        errR = json.UnmarshalRead(prr, &r, dec.Options())
        if errR != nil && !errors.Is(errR, errLeftSuccessful) {
            prr.CloseWithError(errUnmarshalFailedEarly)
        }

        wg.Wait()
        if panicVal != nil {
            panic(panicVal)
        }

        if errL == nil {
            e.isRight = false
            e.l = l
            e.r = *new(R)
            return nil
        }

        if errR == nil {
            e.isRight = true
            e.l = *new(L)
            e.r = r
            return nil
        }

        return eitherErr(errL, errR)
    default: // invalid, '}',    ']'
        // syntax error
        _, err := dec.ReadValue()
        return err
    }
}
```

```go
func TestArshalerEither(t *testing.T) {
    type testCase struct {
        in   string
        fail bool
    }
    for _, tc := range []testCase{
        {"\"foo\"", false},
        {"123", false},
        {"false", true},
        {"{\"foo\": false}", true},
    } {
        var e Either[string, int]
        err := json.Unmarshal([]byte(tc.in), &e)
        if (err != nil) != tc.fail {
            t.Errorf("incorrect!")
        }
        t.Logf("err = %v", err)
        /*
           arshaler_either_test.go:254: err = <nil>
           arshaler_either_test.go:254: err = <nil>
           arshaler_either_test.go:254: err = json: cannot unmarshal into Go play.Either[string,int]: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON boolean into Go string), r = (json: cannot unmarshal JSON boolean into Go int)
           arshaler_either_test.go:254: err = json: cannot unmarshal into Go play.Either[string,int] after offset 13: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON object into Go string), r = (json: cannot unmarshal JSON object into Go int)
        */
    }

    type sampleL struct {
        Foo []int
    }
    type sampleR struct {
        Bar map[string]string
    }
    for _, tc := range []testCase{
        {"\"foo\"", true},
        {"123", true},
        {"false", true},
        {"{\"foo\": false}", true},
        {"{\"Foo\": false}", true},
        {"{\"Foo\": [1,2,3]}", false},
        {"{\"Bar\": {\"foo\":\"foofoo\",\"bar\":\"barbar\"}}", false},
        {"{\"Foo\": [1,2,3}", true},     // syntax error
        {"{\"Bar\": {\"foo\":}}", true}, // syntax error
    } {
        t.Run(tc.in, func(t *testing.T) {
            var e Either[sampleL, sampleR]
            err := json.Unmarshal([]byte(tc.in), &e, json.RejectUnknownMembers(true))
            if (err != nil) != tc.fail {
                t.Errorf("incorrect!")
            }
            t.Logf("err = %v", err)
            /*
                === RUN   TestArshalerEither/"foo"
                    arshaler_either_test.go:286: err = json: cannot unmarshal into Go struct: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON string into Go play.sampleL), r = (json: cannot unmarshal JSON string into Go play.sampleR)
                === RUN   TestArshalerEither/123
                    arshaler_either_test.go:286: err = json: cannot unmarshal into Go struct: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON number into Go play.sampleL), r = (json: cannot unmarshal JSON number into Go play.sampleR)
                === RUN   TestArshalerEither/false
                    arshaler_either_test.go:286: err = json: cannot unmarshal into Go struct: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON boolean into Go play.sampleL), r = (json: cannot unmarshal JSON boolean into Go play.sampleR)
                === RUN   TestArshalerEither/{"foo":_false}
                    arshaler_either_test.go:286: err = json: cannot unmarshal into Go struct after offset 13: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON string into Go play.sampleL: unknown object member name "foo"), r = (json: cannot unmarshal JSON string into Go play.sampleR: unknown object member name "foo")
                === RUN   TestArshalerEither/{"Foo":_false}
                    arshaler_either_test.go:286: err = json: cannot unmarshal into Go struct after offset 13: Either[L, R]: unmarshal failed for both L and R: l = (json: cannot unmarshal JSON boolean into Go []int within "/Foo"), r = (json: cannot unmarshal JSON string into Go play.sampleR: unknown object member name "Foo")
                === RUN   TestArshalerEither/{"Foo":_[1,2,3]}
                    arshaler_either_test.go:286: err = <nil>
                === RUN   TestArshalerEither/{"Bar":_{"foo":"foofoo","bar":"barbar"}}
                    arshaler_either_test.go:286: err = <nil>
                === RUN   TestArshalerEither/{"Foo":_[1,2,3}
                    arshaler_either_test.go:286: err = json: cannot unmarshal into Go struct within "/Foo": Either[L, R]: unmarshal failed for both L and R: l = (jsontext: read error: jsontext: invalid character '}' after array element (expecting ',' or ']') within "/Foo" after offset 14), r = (json: cannot unmarshal JSON string into Go play.sampleR: unknown object member name "Foo")
                === RUN   TestArshalerEither/{"Bar":_{"foo":}}
                    arshaler_either_test.go:286: err = json: cannot unmarshal into Go struct within "/Bar/foo": Either[L, R]: unmarshal failed for both L and R: l = (jsontext: read error: jsontext: missing value after object name within "/Bar/foo" after offset 15), r = (jsontext: read error: jsontext: missing value after object name within "/Bar/foo" after offset 15)
            */
        })
    }
}
```

:::

## おわりに

多分使えるようになるのは`Go 1.26`からでしょうからあと１年とすこし耐えましょう。

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

<!-- RFCs -->

[RFC 6901]: https://datatracker.ietf.org/doc/html/rfc6901

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
[*json.Decoder]: https://pkg.go.dev/encoding/json@go1.24.3#Decoder
[*json.Decoder.More]: https://pkg.go.dev/encoding/json@go1.24.3#Decoder.More
[*json.Encoder]: https://pkg.go.dev/encoding/json@go1.24.3#Encoder
[json.Marshal]: https://pkg.go.dev/encoding/json@go1.24.3#Marshal
[json.Marshaler]: https://pkg.go.dev/encoding/json@go1.24.3#Marshaler
[json.MarshalIndent]: https://pkg.go.dev/encoding/json@go1.24.3#MarshalIndent
[json.Unmarshaler]: https://pkg.go.dev/encoding/json@go1.24.3#Unmarshaler
[json.Unmarshal]: https://pkg.go.dev/encoding/json@go1.24.3#Unmarshal
[xml.NewTokenDecoder]: https://pkg.go.dev/encoding/xml@go1.24.3#NewTokenDecoder
[errors.New]: https://pkg.go.dev/errors@go1.24.3#New
[errors.Is]: https://pkg.go.dev/errors@go1.24.3#Is
[errors.As]: https://pkg.go.dev/errors@go1.24.3#As
[errors.Join]: https://pkg.go.dev/errors@go1.24.3#Join
[fmt.Errorf]: https://pkg.go.dev/fmt@go1.24.3#Errorf
[fs.ErrNotExist]: https://pkg.go.dev/io/fs@go1.24.3#ErrNotExist
[http.Server]: https://pkg.go.dev/net/http@go1.24.3#Server
[*http.Server]: https://pkg.go.dev/net/http@go1.24.3#Server
[io.EOF]: https://pkg.go.dev/io@go1.24.3#EOF
[io.MultiWriter]: https://pkg.go.dev/io@go1.24.3#MultiWriter
[io.Pipe]: https://pkg.go.dev/io@go1.24.3#Pipe
[io.Reader]: https://pkg.go.dev/io@go1.24.3#Reader
[io.Writer]: https://pkg.go.dev/io@go1.24.3#Writer
[syscall.Errno]: https://pkg.go.dev/syscall@go1.24.3#Errno
[text/template]: https://pkg.go.dev/text/template@go1.24.3
