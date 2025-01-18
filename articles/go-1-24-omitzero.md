---
title: 'Go1.24からjson:",omitzero"でなんでもomit可能に'
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## Go1.24から`json:",omitzero"`なんでもomit可能になる

以下がGo1.24のDraft Release Notesです。現時点(2025-01-18)ではDraftですがリリースされるとURLはそのままでDraftの文言が消されます。

https://tip.golang.org/doc/go1.24#encodingjsonpkgencodingjson

上記の通り、`encoding/json`に変更が入ります。

> When marshaling, a struct field with the new omitzero option in the struct field tag will be omitted if its value is zero. If the field type has an IsZero() bool method, that will be used to determine whether the value is zero. Otherwise, the value is zero if it is the zero value for its type.

筆者はこれまで下記の記事のとおりにstructなどの値が`omitempty`によってomitされないことにあらがおうとごちゃごちゃしてきたわけですが、

https://zenn.dev/ngicks/articles/go-json-that-can-be-t-null-or-undefined
https://zenn.dev/ngicks/articles/go-json-undefined-or-null-v2
https://zenn.dev/ngicks/articles/go-json-undefined-or-null-slice

これからはstdの範疇でもっと簡単にこれらが実現できるわけです。

この記事は上記の記事群で求めていたテーマの最終着地点です。

## 環境

```
# go install golang.org/dl/go1.24rc2@latest
go: downloading golang.org/dl v0.0.0-20250116195134-55ca457114df
# go1.24rc2 download
Downloaded   0.0% (    3121 / 77743067 bytes) ...
Downloaded   6.0% ( 4685792 / 77743067 bytes) ...
Downloaded  15.3% (11878320 / 77743067 bytes) ...
Downloaded  24.5% (19070832 / 77743067 bytes) ...
Downloaded  33.8% (26262784 / 77743067 bytes) ...
Downloaded  42.9% (33340624 / 77743067 bytes) ...
Downloaded  51.1% (39713952 / 77743067 bytes) ...
Downloaded  60.3% (46907040 / 77743067 bytes) ...
Downloaded  69.6% (54098992 / 77743067 bytes) ...
Downloaded  78.9% (61308464 / 77743067 bytes) ...
Downloaded  88.1% (68500416 / 77743067 bytes) ...
Downloaded  96.3% (74890128 / 77743067 bytes) ...
Downloaded 100.0% (77743067 / 77743067 bytes)
Unpacking /root/sdk/go1.24rc2/go1.24rc2.linux-amd64.tar.gz ...
Success. You may now run 'go1.24rc2'
# go1.24rc2 version
go version go1.24rc2 linux/amd64
```

## 前提知識

- `encoding/json`の使い方を知っていること
- `JSON`とは何かを知っていること。
  - とりあえず[RFC8259](https://datatracker.ietf.org/doc/html/rfc8259)を読めばOK

## 前提: `json:",omitempty"`はstructをomitしない

`Go`のstd libraryの範疇では`encoding/json`パッケージを用いてstructなどのdata typeとJSONとの相互変換を行います。

`JSON`は、時と場合によって`Object`のfieldを省略することがあります。
`encoding/json`ではstruct tagで`json:",omitempty"`を指定することで、`empty value`であるときfieldの省略(=omit)を行う挙動があります。

`empty value`の判定は以下で行われます。

https://github.com/golang/go/blob/go1.24rc2/src/encoding/json/encode.go#L318-L330

見てのとおり、

- array, map, slice, string: len == 0
- bool, intとそのvariant(int32のような), uintとそのvariant(uint16のような), float32/float64, interface, pointer: zero value

であるときに`empty`であるとみなされます。

この条件にstructがないことから、structがemptyとみなされることがないことがわかります。
(`chan T`もこの中で無視されていますが、そもそもchanを含むstructのmarshalはサポートされていないためエラーになります。)

## 課題: time.Timeのような型がomitできない

例えば`time.Time`のようなstructをunderlying typeとする型がomitできません。

https://github.com/golang/go/blob/go1.24rc2/src/time/time.go#L140-L161

`time.Time`はstructであり、すべてのfieldがunexportであるので型そのものが値みたいなものですが、今まではomitできませんでした。

## omitzero

### issue

ということで何年も前からstructも含めてzero valueならomitする機能ほしいよねっていうissueは上がっていました。

https://github.com/golang/go/issues/45669

ついに上記が採用された形でcloseされました！

大雑把に以下の3つが検討されて`omitzero`に終着した形になります。

- `omitempty`の挙動を変える
- `MarshalJSON`実装で`nil`を返させる or 特殊なエラーを返させる
- `omitnil`

`omitzero`が現実的になったのはおそらく`reflect.Value.IsZero`が最適化されて高速化されたからでしょうね。
実際、`isEmptyValue`の実装のなかで[`IsZero`が呼ばれるようになったのはGo1.22から](https://github.com/golang/go/blob/go1.22.0/src/encoding/json/encode.go#L306-L318)で[Go1.21まででは型ごとに細かいチェックを行っていました](https://github.com/golang/go/blob/go1.21.0/src/encoding/json/encode.go#L304-L320)。
`Go1.22`で[CL411478](https://go-review.googlesource.com/c/go/+/411478)が適用されたことで`reflect.Value.IsZero`が高速化されたことでこうなったらようです。

### 実装

https://github.com/golang/go/blob/go1.24rc2/src/encoding/json/encode.go#L715-L718
https://github.com/golang/go/blob/master/src/encoding/json/encode.go#L1187-L1219

上記のように、`json:",omitzero"`がつけられていると、

- 型が`interface { IsZero() bool }`を実装している場合、これがtrueを返すとき
- もしくは[reflect.Value.IsZero](https://pkg.go.dev/reflect@go1.24rc2#Value.IsZero)がtrueを返すとき

のいずれかの時fieldがomitされます。

fieldの型がnon pointerで`IsZero`のmethod receiverがpointer typeのときの考慮が特筆すべき点ですね。

### 挙動

[playground](https://go.dev/play/p/OJtG9DLpZpu?v=gotip)

例えば以下のように型を定義します。

```go
type foo struct {
    Bar bar `json:",omitzero"`
}
```

zero valueのときomitされるのがわかります。

```go
var f foo
bin, err := json.MarshalIndent(f, "", "    ")
if err != nil {
    panic(err)
}
fmt.Printf("%s\n", bin)
// {}
f.Bar.F1 = "foo"
bin, err = json.MarshalIndent(f, "", "    ")
if err != nil {
    panic(err)
}
fmt.Printf("%s\n", bin)
// {
//     "Bar": {
//         "F1": "foo",
//         "F2": 0
//     }
// }
```

前述通り、`interface { IsZero() bool }`が実装されるとき、こちらが優先して使われます。
`time.Time`の`IsZero`はwall clockもしくはmonotonic timerがzeroであるときtrueを返します。
`time.Time`はfieldに[\*time.Location](https://pkg.go.dev/time@go1.24rc2#Location)を含むため、zero valueではないが`IsZero`がtrueを返すことがあります。

```go
type times struct {
    Empty time.Time `json:",omitempty"`
    Zero  time.Time `json:",omitzero"`
}
```

以下のように、zero valueでない`time.Time`もomitされます。

```go
t := times{
    Empty: time.Time{}.In(time.Local),
    Zero:  time.Time{}.In(time.Local),
}
bin, err := json.MarshalIndent(t, "", "    ")
if err != nil {
    panic(err)
}
fmt.Printf("is zero: %t\n", reflect.ValueOf(t.Zero).IsZero())
fmt.Printf("%s\n", bin)
// is zero: false
// {
//     "Empty": "0001-01-01T00:00:00Z"
// }
```

### v2とのcompatibility

[discussion: encoding/json/v2](https://github.com/golang/go/discussions/63397)で述べられている[experimental実装](https://github.com/go-json-experiment/json)でもほぼ同じ実装になっています。

https://github.com/go-json-experiment/json/blob/master/arshal_default.go#L1056-L1061
https://github.com/go-json-experiment/json/blob/master/fields.go#L200-L217

[このコメント](https://github.com/golang/go/issues/45669#issuecomment-2215356195)から`encoding/json`への`omitzero`の追加は、立ち位置的には仮想的なv2からのバックポートということになります。

## JSONのT | null | undefinedはOption[Option[T]]で表現できる

`Go1.23`以前ではJSONのT | null | undefinedを単なるstruct fieldで表現するには`[]Option[T]`を用いる必要がありました。
これは前述のとおり`,omitempty`で任意の型を収められるcontainer typeをomitさせようと思うとslice(`[]T`), map(`map[K]V`)を用いる必要があったためです。
下記の記事であれこれ述べました。

https://zenn.dev/ngicks/articles/go-json-undefined-or-null-slice

この方法には値がuncomparableになってしまうという明確な問題がありました。

`Go1.24`以降では`omitzero`が実装されるため`Option[Option[T]]`で同じ目的を達成できます。
この場合、`Option`の実装をcomparableにしておけば`T`がcomparableである限り`Option[Option[T]]`もcomparableとなります。

### 型の定義

以下のようにstructをunderlyingとした`Option[T]`を定義し、

https://github.com/ngicks/und/blob/v1.0.0-alpha8/option/opt.go#L18-L22

これを2段重ねて`Option[Option[T]]`とすることでT | null | undefinedを表現し分けられるようになります。

https://github.com/ngicks/und/blob/v1.0.0-alpha8/und.go#L34-L36

そしてこの型に`IsZero`を実装します。

https://github.com/ngicks/und/blob/v1.0.0-alpha8/und.go#L95-L98

中身はboolean flagを確認するだけです。

https://github.com/ngicks/und/blob/v1.0.0-alpha8/und.go#L110-L113

`,omitzero`がこの型のfieldをomitできるのでこれでよくなりました！

### sample

以下のsnippetを`go1.24rc2 run github.com/ngicks/und/example@v1.0.0-alpha8`で実行すると、コメントされたような結果がprintされます。

`und.Und`が`Option[Option[T]]`、`sliceund.Und`が`[]Option[T]`をベースとする型です。

```go
package main

import (
    "encoding/json"
    "fmt"

    "github.com/ngicks/und"
    "github.com/ngicks/und/elastic"
    "github.com/ngicks/und/option"

    "github.com/ngicks/und/sliceund"
    sliceelastic "github.com/ngicks/und/sliceund/elastic"
)

type sample1 struct {
    Foo  string
    Bar  und.Und[nested1]              `json:",omitzero"`
    Baz  elastic.Elastic[nested1]      `json:",omitzero"`
    Qux  sliceund.Und[nested1]         `json:",omitzero"`
    Quux sliceelastic.Elastic[nested1] `json:",omitzero"`
}

type nested1 struct {
    Bar  und.Und[string]            `json:",omitzero"`
    Baz  elastic.Elastic[int]       `json:",omitzero"`
    Qux  sliceund.Und[float64]      `json:",omitzero"`
    Quux sliceelastic.Elastic[bool] `json:",omitzero"`
}

type sample2 struct {
    Foo  string
    Bar  und.Und[nested2]              `json:",omitempty"`
    Baz  elastic.Elastic[nested2]      `json:",omitempty"`
    Qux  sliceund.Und[nested2]         `json:",omitempty"`
    Quux sliceelastic.Elastic[nested2] `json:",omitempty"`
}

type nested2 struct {
    Bar  und.Und[string]            `json:",omitempty"`
    Baz  elastic.Elastic[int]       `json:",omitempty"`
    Qux  sliceund.Und[float64]      `json:",omitempty"`
    Quux sliceelastic.Elastic[bool] `json:",omitempty"`
}

func main() {
    s1 := sample1{
        Foo:  "foo",
        Bar:  und.Defined(nested1{Bar: und.Defined("foo")}),
        Baz:  elastic.FromValue(nested1{Baz: elastic.FromOptions(option.Some(5), option.None[int](), option.Some(67))}),
        Qux:  sliceund.Defined(nested1{Qux: sliceund.Defined(float64(1.223))}),
        Quux: sliceelastic.FromValue(nested1{Quux: sliceelastic.FromOptions(option.None[bool](), option.Some(true), option.Some(false))}),
    }

    var (
        bin []byte
        err error
    )
    bin, err = json.MarshalIndent(s1, "", "    ")
    if err != nil {
        panic(err)
    }
    fmt.Printf("marshaled by with omitzero =\n%s\n", bin)
    // see? undefined (=zero value) fields are omitted with json:",omitzero" option.
    // ,omitzero is introduced in Go 1.24. For earlier version Go, see example of sample2 below.
    /*
        marshaled by with omitzero =
        {
            "Foo": "foo",
            "Bar": {
                "Bar": "foo"
            },
            "Baz": [
                {
                    "Baz": [
                        5,
                        null,
                        67
                    ]
                }
            ],
            "Qux": {
                "Qux": 1.223
            },
            "Quux": [
                {
                    "Quux": [
                        null,
                        true,
                        false
                    ]
                }
            ]
        }
    */

    s2 := sample2{
        Foo:  "foo",
        Bar:  und.Defined(nested2{Bar: und.Defined("foo")}),
        Baz:  elastic.FromValue(nested2{Baz: elastic.FromOptions(option.Some(5), option.None[int](), option.Some(67))}),
        Qux:  sliceund.Defined(nested2{Qux: sliceund.Defined(float64(1.223))}),
        Quux: sliceelastic.FromValue(nested2{Quux: sliceelastic.FromOptions(option.None[bool](), option.Some(true), option.Some(false))}),
    }

    bin, err = json.MarshalIndent(s2, "", "    ")
    if err != nil {
        panic(err)
    }
    fmt.Printf("marshaled with omitempty =\n%s\n", bin)
    // You see. Types defined under ./sliceund/ can be omitted by encoding/json@go1.23 or earlier.
    /*
        marshaled with omitempty =
        {
            "Foo": "foo",
            "Bar": {
                "Bar": "foo",
                "Baz": null
            },
            "Baz": [
                {
                    "Bar": null,
                    "Baz": [
                        5,
                        null,
                        67
                    ]
                }
            ],
            "Qux": {
                "Bar": null,
                "Baz": null,
                "Qux": 1.223
            },
            "Quux": [
                {
                    "Bar": null,
                    "Baz": null,
                    "Quux": [
                        null,
                        true,
                        false
                    ]
                }
            ]
        }
    */
}
```

sliceやarrayに含まれる*undefined*である`und.Und[T]`が`null`を出力するのはECMAの`JSON.stringify`と挙動が一致しているためちょうどいい感じになっています。

## おわりに

もう`omitempty`つかわなくていいかも。

`Go`も歴史が深くなってきて、昔はこうだったけど今はこうすべき見たいなtipsがいくつか出てきました。
例えば`for k, v := range`でiterator variableをshadowingしたほうがよかったのは[Go1.22で修正された](https://tip.golang.org/doc/go1.22#language)のでやらなくてよくなりました。
`omitzero`もそう言ったものの一つです。`omitempty`に比べて仕様が明快なのでこっちを使っていくほうが初心者には優しいと思います。(実際筆者はemptyの判定の条件を初めて見たとき混乱しました。)
そういうののを集めたtips集を作ってメンテしていったほうがいいかもしれませんね。多分文法とstd libraryの範疇に話をとどめればそんなに大きなものにはなりませんし。
