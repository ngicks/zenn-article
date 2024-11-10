---
title: "[Go]ast(dst)と型情報からコードを生成する(partial-json patcher etc)"
emoji: "📊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## ast(dst)と型情報からコードを生成する(partial-json patcher etc)

こんにちは

この記事では`Go`のast(dst)と型情報を用いたコードジェネレーターの実装を例にしながらポイントや考慮すべきことをまとめます。
似たような感じでコードジェネレーターを作りたい人や、`go/ast`や`go/types`以下で実装される型や関数の使い方がわからなくてとっかかりがつかめない人(かつての私)が進みやすくなるかもしれないことを目指しています。

## Overview

プログラムを書いていると時たま、コードジェネレーターの吐いたコードの結果を受けてさらにコードを編集したいときがあります。例としては[github.com/oapi-codegen/oapi-codegen]が生成したコードの特定パス以下(e.g. `/config`)のrequest bodyをパスごとに保存できる簡単なconfig storeを作ったりなどですね。
`Go`のを引数にコードジェネレーターを作成する際、単にテキストファイルとしてソースコードを解析してもよいのですが、astや型情報をそのまま用いることができたほうが改行やコメントその他で意味論的に違いのないソースコードをうまく処理できるため、その観点からはできるならそうしたほうがいいと言えます。

そこで本記事では以下のようなことをします。

- `ast`と型情報の解析
  - [golang.org/x/tools/go/packages]を使います
- `ast`の書き換え
  - 実際には[github.com/dave/dst]を用います。
- `ast`のnode単位の部分的なプリント
- 型情報から`ast`への変換
- ファイルのimportの解析と連携
- struct tagの編集
- 型情報をグラフ化して、型定義の依存関係を上に向けて探索する

これらを具体的な実装物を通じて説明します。
概説的、網羅的にはなりません。あくまで今回実装したものの周りについてのみ説明します。

## 前提知識

- `Go`の構文ルールがわかる
- `Go`におけるjsonの取り扱いがわかる
- `ast`や型情報といったものがどういうものかある程度わかる
  - `Go`固有と思しき`ast`構造についてはいくらか触れますが、`ast`という言葉自体を解説しません。

構文ルール(や`Go`プロジェクトの始め方)は[Goで開発して3年のプラクティスまとめ](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part1)という一連の記事でまとめています・・・というか、`A Tour of Go`をやったら30分～数時間ぐらいで分かるよということだけのべておきます。

`json`を含めたデータの取り扱いは[Goで開発して3年のプラクティスまとめ(2/4)のデータのシリアライズ/デシリアライズの項目](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part2#%E3%83%87%E3%83%BC%E3%82%BF%E3%81%AE%E3%82%B7%E3%83%AA%E3%82%A2%E3%83%A9%E3%82%A4%E3%82%BA%2F%E3%83%87%E3%82%B7%E3%83%AA%E3%82%A2%E3%83%A9%E3%82%A4%E3%82%BA)や[GoのJSONのT | null | undefinedは\[\]Option\[T\]で表現できる]ですでにそこそこ深めに述べましたので、そちらを読んでいただければわかることもあるかもしれません。

基本的にはある程度実践的に`Go`を使ったことがあることを前提とします。

## Goとastと型情報とコードジェネレーター

一応[Go]とは何ぞやというところから触れていきます。

> https://go.dev/doc/
>
> The Go programming language is an open source project to make programmers more productive.
>
> Go is expressive, concise, clean, and efficient. Its concurrency mechanisms make it easy to write programs that get the most out of multicore and networked machines, while its novel type system enables flexible and modular program construction. Go compiles quickly to machine code yet has the convenience of garbage collection and the power of run-time reflection. It's a fast, statically typed, compiled language that feels like a dynamically typed, interpreted language.

[Go]はGoogleが開発しているオープンソースのプログラミング言語です。

C系の文法の言語で、特徴は大雑把に

- concurrentな関数の実行が言語機能として組み込まれています。
- `interface`によるdynamic dispatchがサポートされています。
- tupleはありません。関数は多値返却を行えます。
- エラーは`error` interfaceを実装する値であり、通例では多値返却の末尾の値として返します。
- 言語機能が絞られていていろいろできません。
- operatorのオーバーロードは存在しません。
- `array`(固定長シーケンスデータ: `[n]T`), `slice`(可変長アレイ: `[]T`), `map`(hash-map: `map[K]V`), `channel`(FIFOチャネル: `chan T`)などの組み込み型が特別扱いされており、これらを組み合わせてプログラムを構築します

シンプルであるがゆえにコードジェネレーターの実装が容易です。
マクロ機能は現状ありません。

[Go1.18]までgenericsが存在していなかったこともあり、上記の組み込み型のみがジェネリックに中身の型を取り換えられる型でした。

- `array`, `slice`, `map`はコンパイラが具体的な別の型に書き換えます。
- appending, insert, channel-send/receiveはコンパイラが具体的な関数に書き換えます。

これらの事情を反映して、ast上でも型情報上でも以下のように型として表現されます

- ast:
  - [*ast.ArrayType]
  - [*ast.MapType]
  - [*ast.ChanType]
- types:
  - [*types.Array]
  - [*types.Slice]
  - [*types.Map]
  - [*types.Chan]

ast上では`slice`も`array`も同じ`ArrayType`になります。`Len`がnilであれば`slice`です。

型上これらが出現するため追跡が容易です。
これが`Vec<T>`のような名前付き型であったり、カスタムデータコンテナだと特別扱いしたい型が増えて大変になっていたかもしれません。

## 実現したいもの

具体的にどういったものを実装するかについて述べます

https://zenn.dev/ngicks/articles/go-json-undefined-or-null-slice

で作った、[github.com/ngicks/und]以下で定義される型を用いると、

- [sliceund.Und]: JSONの`undefined | null | T`
- [sliceelastic.Elastic]: JSONの`undefined | null | (T | null)[]`

を`Go`のstruct fieldで表現することができます。

本記事ではこれらを用いて以下を実現するコードを生成するコードジェネレーターを実装します。

- Patcher
  - 対象となるstruct typeの、あらゆるフィールドを[sliceund.Und]でラップし、`json:",omitempty"`を付け足した別の型を定義し、それを元となった方にApplyできるメソッドを実装することでpartial jsonによるpatchを実現する
- Valdator
  - `und:""` struct tagの内容でsomeじゃないいけないとか、Elasticの場合は`[]T`がn要素以上ないといけないとかを決められるようにしてあるため、この情報を用いてvalidateを行う
- Plain
  - `und:""` struct tagの内容からsomeでないといけないなら`option.Option[T]`を`T`にアンラップしたような*Plain*な型を作成し、元となった型との相互変換を実現する。
  - こうすることで、Marshal/Unmarshalの界面ではlosslessで`undefined | null | T`をデータ構造に当てはめることができるため、`map[string]any`などを介さずにフィールドの有り無しをvalidateし、その後扱いやすい型に変換してから内部処理を行うことができます。

## 生成されるコードのイメージ

まずどういったコードを生成したら目標が実現できるかを思い描き、そこから何を実装すべきかについて考えます。

### Patcher

Patcherが実現したいのはpartial jsonを受けとって元となるデータ構造にパッチを当てることです。

そのため、Patchのフィールドが

- `undefined`である(入力jsonにフィールドが存在しない)とき更新しない
- `null`であるとき、ゼロ値で上書きる
- `T`であるときその値で上書きする

という挙動を実現します。

- 元となる型からpatchへの変換
- patch同士のmerge
- merge結果を元の型に逆変換

で、partial jsonによるpatchの挙動が実現できます。

つまり、以下が入力であるとき

```go
type All struct {
    Foo string
    Bar *int      `json:",omitempty"`
    Baz *struct{} `json:"baz,omitempty"`
    Qux []string

    Opt          option.Option[string] `json:"opt,omitzero"`
    Und          und.Und[string]       `json:"und"`
    Elastic      elastic.Elastic[string]
    SliceUnd     sliceund.Und[string]
    SliceElastic sliceelastic.Elastic[string]
}
```

以下が出力される

```go
type AllPatch struct {
    Foo sliceund.Und[string]    `json:",omitempty"`
    Bar sliceund.Und[*int]      `json:",omitempty"`
    Baz sliceund.Und[*struct{}] `json:"baz,omitempty"`
    Qux sliceund.Und[[]string]  `json:",omitempty"`

    Opt          sliceund.Und[string]         `json:"opt,omitempty"`
    Und          und.Und[string]              `json:"und,omitzero"`
    Elastic      elastic.Elastic[string]      `json:",omitzero"`
    SliceUnd     sliceund.Und[string]         `json:",omitempty"`
    SliceElastic sliceelastic.Elastic[string] `json:",omitempty"`
}

func (p *AllPatch) FromValue(v All) {
    *p = AllPatch{
        Foo:          sliceund.Defined(v.Foo),
        Bar:          sliceund.Defined(v.Bar),
        Baz:          sliceund.Defined(v.Baz),
        Qux:          sliceund.Defined(v.Qux),
        Opt:          option.MapOr(v.Opt, sliceund.Null[string](), sliceund.Defined[string]),
        Und:          v.Und,
        Elastic:      v.Elastic,
        SliceUnd:     v.SliceUnd,
        SliceElastic: v.SliceElastic,
    }
}

func (p AllPatch) ToValue() All {
    return All{
        Foo:          p.Foo.Value(),
        Bar:          p.Bar.Value(),
        Baz:          p.Baz.Value(),
        Qux:          p.Qux.Value(),
        Opt:          option.Flatten(p.Opt.Unwrap()),
        Und:          p.Und,
        Elastic:      p.Elastic,
        SliceUnd:     p.SliceUnd,
        SliceElastic: p.SliceElastic,
    }
}

func (p AllPatch) Merge(r AllPatch) AllPatch {
    return AllPatch{
        Foo:          sliceund.FromOption(r.Foo.Unwrap().Or(p.Foo.Unwrap())),
        Bar:          sliceund.FromOption(r.Bar.Unwrap().Or(p.Bar.Unwrap())),
        Baz:          sliceund.FromOption(r.Baz.Unwrap().Or(p.Baz.Unwrap())),
        Qux:          sliceund.FromOption(r.Qux.Unwrap().Or(p.Qux.Unwrap())),
        Opt:          sliceund.FromOption(r.Opt.Unwrap().Or(p.Opt.Unwrap())),
        Und:          und.FromOption(r.Und.Unwrap().Or(p.Und.Unwrap())),
        Elastic:      elastic.FromUnd(und.FromOption(r.Elastic.Unwrap().Unwrap().Or(p.Elastic.Unwrap().Unwrap()))),
        SliceUnd:     sliceund.FromOption(r.SliceUnd.Unwrap().Or(p.SliceUnd.Unwrap())),
        SliceElastic: sliceelastic.FromUnd(sliceund.FromOption(r.SliceElastic.Unwrap().Unwrap().Or(p.SliceElastic.Unwrap().Unwrap()))),
    }
}

func (p AllPatch) ApplyPatch(v All) All {
    var orgP AllPatch
    orgP.FromValue(v)
    merged := orgP.Merge(p)
    return merged.ToValue()
}
```

- [sliceund.Und]でラップしたフィールドには`json:",omitempty"`を付け足すことで、partial jsonのmarshal/unmarshalをできるようにします。
  - 付け足す、というのがキモです。元からあった`json` structタグはなるだけそのままにする必要があります・・・。
- `FromValue`で元となった型からpatchへ変換、入力patchと`Merge`でマージ、`ToValue`で元となった型に逆変換することでパッチの挙動を実現します。
- `Merge`は[github.com/ngicks/und]の機能をふんだんに使って`Or`をとることで実現します。
- 元の型->パッチ型な変換は元の型にメソッドとして実現するか、New*FooBar*Patchという関数で実現するかしたほうがよかったかもしれませんが、下記理由でしませんでした
  - なるだけ元の型には何も追加したくないので、メソッドの追加もしたくありません。
    - 追加すると名前被りのリスクがあります。
    - リスク回避のために自然に感じられないPrefixをつけて被りにくくするとかがありえますが、これを避けたいわけです
  - New*FooBar*Patch的な関数も同様で名前被りのリスクがあります。

### Validator

Validatorが実現したいのは、und type([github.com/ngicks/und]で定義される諸般の型)をfieldにもつstruct typeがあるとき、struct tagで`und:""`を指定すると、その内容に基づいてundefined / null / definedの状態などのvalidationを行うことです。

und typeのvalidationを行うためのvalidatorは[untag.UndOpt](https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/undtag#UndOpt)としてexportしてあるため、`und:""`の内容を解析して`UndOpt`を定義し、各フィールドをvalidateすることになります。`undtag`パッケージは循環依存を避けながら`undtag`パッケージ内でoption typeを用いるために、internal packageとしてコピーしたinternaloptionを使用しています。この微妙な理由から`UndOpt`自体を外部のパッケージ/モジュールが初期化することができません。そのため[undtag.UndOptExport](https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/undtag#UndOptExport)をexportしておき、これを通じて`UndOpt`を初期化するようにします。

入力が以下であるとき

```go
type All struct {
    Foo    string
    Bar    option.Option[string]        // no tag
    Baz    option.Option[string]        `und:"def"`
    Qux    und.Und[string]              `und:"def,und"`
    Quux   elastic.Elastic[string]      `und:"null,len==3"`
    Corge  sliceund.Und[string]         `und:"nullish"`
    Grault sliceelastic.Elastic[string] `und:"und,len>=2,values:nonnull"`
}
```

以下が出力される

```go
package validatortarget

import (
    "fmt"

    "github.com/ngicks/und"
    "github.com/ngicks/und/elastic"
    "github.com/ngicks/und/option"
    "github.com/ngicks/und/undtag"
    "github.com/ngicks/und/validate"
)

func (v All) UndValidate() error {
    {
        validator := undtag.UndOptExport{
            States: &undtag.StateValidator{
                Def: true,
            },
        }.Into()

        if !validator.ValidOpt(v.Baz) {
            return validate.AppendValidationErrorDot(
                fmt.Errorf("%s: value is %s", validator.Describe(), validate.ReportState(v.Baz)),
                "Baz",
            )
        }
    }
    {
        validator := undtag.UndOptExport{
            States: &undtag.StateValidator{
                Def: true,
                Und: true,
            },
        }.Into()

        if !validator.ValidUnd(v.Qux) {
            return validate.AppendValidationErrorDot(
                fmt.Errorf("%s: value is %s", validator.Describe(), validate.ReportState(v.Qux)),
                "Qux",
            )
        }
    }
    {
        validator := undtag.UndOptExport{
            States: &undtag.StateValidator{
                Def:  true,
                Null: true,
            },
            Len: &undtag.LenValidator{
                Len: 3,
                Op:  undtag.LenOpEqEq,
            },
        }.Into()

        if !validator.ValidElastic(v.Quux) {
            return validate.AppendValidationErrorDot(
                fmt.Errorf("%s: value is %s", validator.Describe(), validate.ReportState(v.Quux)),
                "Quux",
            )
        }
    }
    {
        validator := undtag.UndOptExport{
            States: &undtag.StateValidator{
                Null: true,
                Und:  true,
            },
        }.Into()

        if !validator.ValidUnd(v.Corge) {
            return validate.AppendValidationErrorDot(
                fmt.Errorf("%s: value is %s", validator.Describe(), validate.ReportState(v.Corge)),
                "Corge",
            )
        }
    }
    {
        validator := undtag.UndOptExport{
            States: &undtag.StateValidator{
                Def: true,
                Und: true,
            },
            Len: &undtag.LenValidator{
                Len: 2,
                Op:  undtag.LenOpGrEq,
            },
            Values: &undtag.ValuesValidator{
                Nonnull: true,
            },
        }.Into()

        if !validator.ValidElastic(v.Grault) {
            return validate.AppendValidationErrorDot(
                fmt.Errorf("%s: value is %s", validator.Describe(), validate.ReportState(v.Grault)),
                "Grault",
            )
        }
    }

    return nil
}
```

さらに、フィールドがvalidatorを実装する際にはそれを呼び出せるようにします。

```go
// Allは上記スニペットのAllです
type Dependent struct {
    Foo  All
    Bar  option.Option[All] `und:"required"`
}

func (v Dependent) UndValidate() (err error) {
    if err := v.Foo.UndValidate(); err != nil {
        return validate.AppendValidationErrorDot(
            err,
            "Foo",
        )
    }
    {
        validator := undtag.UndOptExport{
            States: &undtag.StateValidator{
                Def: true,
            },
        }.Into()

        if !validator.ValidOpt(v.Bar) {
            err = fmt.Errorf("%s: value is %s", validator.Describe(), validate.ReportState(v.Bar))
        }
        if err == nil {
            err = option.UndValidate(v.Bar)
        }
        if err != nil {
            return validate.AppendValidationErrorDot(
                err,
                "Bar",
            )
        }
    }
    return nil
}
```

こうすればund typeを含むstructが複数ネストした場合でもフィールドをすべてvalidateして回れるようになります。

[validate.AppendValidationErrorDot](https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/validate#AppendValidationErrorDot)と[validate.AppendValidationErrorIndex](https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/validate#AppendValidationErrorIndex)は深くネストしたフィールドのどこがvalidationエラーだったのか表示するためにフィールド名をアペンドしていけるようになっているヘルパーで、[ValidationError.Pointer](https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/validate#ValidationError.Pointer)メソッドでJSON pointerを取得できるようにしてあります。

### Plain

Plainが実現したいのは、struct fieldがund typeであり`und:""`タグが指定されているとき、このタグの内容に応じてフィールドをアンラップした*Plain*な型を作り、これと元となった方の相互変換を行うことです。

これを行うことのメリットは下記が実現できることです。

- 外部データのUnmarshal時には[sliceund.Und]でデータを受けとり、上記Validateによって存在しなければならないフィールドの検査を行ってから*Plain*な型に変換してプログラム内ではこれを処理する
- [Elasticsearch]に格納されたデータのUnmarshal時には[sliceelastic.Elastic]でデータを受けとり、例えば`string`でなければならない`keyword type`に事故的に`[]string`が格納されている際に、エラーでデータを受け付けないのではなく穏当にwarningのログを出して`[]string`の最初の値以外を無視する
  - 仕様変更により`[]T`を格納していたフィールドを`T`にリミットしたいときなどにも起こるケースかと思います

`Go`でJSONの`undefined | null | T`を表現しづらいという課題感は[GoのJSONのT | null | undefinedは\[\]Option\[T\]で表現できる]で説明しているのでそこを合わせて読んでいただければと思います。

- `und:""`で指定できるのは
  - `def`(=defined)
  - `null`
  - `und`(=undefined)
  - `required` = `def`のshorthand
  - `nullish` = `null,und`のshorthand
  - `len` = `Elastic`の長さを指定、
    - `len>n`, `len>=n`, `len==n`, `len<n`, `len<=n`でそれぞれ要素数の制限を指定できます
    - どうしてここまで柔軟な仕様に・・・？
  - `values` = `Elastic`の各要素の状態を指定
    - `values:nonnull`で各要素は`null`になってはならないことを表現できる。

これに合わせ、

- `Und`+`und:"def"` -> `T`
- `Und`+`und:"def,null"` -> `option.Option[T]`
- `Elastic`+`und:"def,len==n"` -> `[n]option.Option[T]`
- `Elastic`+`und:"len>2,values:nonnull"` -> `und.Und[[]T]`

みたいな感じでフィールドが変換された型を生成し、これと相互変換を行うことで*Plain*に感じられる型で内部処理を行えるようにします。
`len==n`の時、要素数`n`のarrayに変換されるのがやばいです。これによってgenericsによる処理ができなくなり、要素数ごとのアドホックな即時間数を生成する必要ができました。誰ですかこんな仕様を考えたやつは・・・。

つまり以下のような型が入力であるとき

```go
type Example struct {
    Foo   string                    `json:"foo"`
    Bar   option.Option[string]     `json:"bar" und:"required"`
    Baz   und.Und[string]           `json:"baz" und:"def"`
    Qux   und.Und[string]           `json:"qux" und:"def,null"`
    Quux  sliceelastic.Elastic[int] `json:"quux" und:"len==3"`
    Corge sliceelastic.Elastic[int] `json:"corge" und:"len>2,values:nonnull"`
}
```

以下が出力されるだろうということです

```go
type ExamplePlain struct {
    Foo   string                `json:"foo"`
    Bar   string                `json:"bar" und:"required"`
    Baz   string                `json:"baz" und:"def"`
    Qux   option.Option[string] `json:"qux" und:"def,null"`
    Quux  [3]option.Option[int] `json:"quux" und:"len==3"`
    Corge []int                 `json:"corge" und:"len>2,values:nonnull"`
}

func (v Example) UndPlain() ExamplePlain {
    return ExamplePlain{
        Foo: v.Foo,
        Bar: v.Bar.Value(),
        Baz: v.Baz.Value(),
        Qux: v.Qux.Unwrap().Value(),
        Quux: sliceund.Map(
            conversion.UnwrapElasticSlice(v.Quux),
            func(o []option.Option[int]) (out [3]option.Option[int]) {
                copy(out[:], o)
                return out
            },
        ).Value(),
        Corge: conversion.NonNullSlice(conversion.LenNAtLeastSlice(3, conversion.UnwrapElasticSlice(v.Corge))).Value(),
    }
}

func (v ExamplePlain) UndRaw() Example {
    return Example{
        Foo: v.Foo,
        Bar: option.Some(v.Bar),
        Baz: und.Defined(v.Baz),
        Qux: conversion.OptionUnd(true, v.Qux),
        Quux: sliceelastic.FromUnd(sliceund.Map(
            sliceund.Defined(v.Quux),
            func(s [3]option.Option[int]) []option.Option[int] {
                return s[:]
            },
        )),
        Corge: sliceelastic.FromUnd(conversion.NullifySlice(sliceund.Defined(v.Corge))),
    }
}
```

さらに、フィールドがこの`UndRaw`/`UndPlain`という変換メソッドをを実装する際にはそれを呼び出せるようにします。
ObjectにObjectやArrayがネストしているJSONは普通に存在していますから、これができないと実用に耐えないですね。

つまり以下のような、`IncludesImplementor`が存在すると

```go
package sub

type IncludesImplementor struct {
    Foo sub2.Foo[int]
}

---

package sub2

type Foo[T any] struct {
    T   T
    Yay string
}

func (f Foo[T]) UndPlain() FooPlain[T] {
    return FooPlain[T]{
        Nay: f.Yay,
    }
}


type FooPlain[T any] struct {
    T   T
    Nay string
}

func (f FooPlain[T]) UndRaw() Foo[T] {
    return Foo[T]{
        Yay: f.Nay,
    }
}
```

以下のように生成されます。

```go
type IncludesImplementorPlain struct {
    Foo sub2.FooPlain[int]
}

func (v IncludesImplementor) UndPlain() IncludesImplementorPlain {
    return IncludesImplementorPlain{
        Foo: v.Foo.UndPlain(),
    }
}

func (v IncludesImplementorPlain) UndRaw() IncludesImplementor {
    return IncludesImplementor{
        Foo: v.Foo.UndRaw(),
    }
}
```

## 実装すべき機能

### 機能

- 1. astおよび型情報の収集
- 2. struct tagの編集
- 3. import情報の連携
- 4. astのrewrite、およびrewriteによって生成されたコードと別の方法で生成されたコードを同じファイルに書き出せるようにする
- 5. メタデータの収集

`1.`には[golang.org/x/tools/go/packages]を用います。

`2.`は解析器と編集機能を実装します。[encoding/json v2(候補)について紹介してundefined | null | Tを表現する](https://zenn.dev/ngicks/articles/go-json-undefined-or-null-v2)で解析器までは実装していたのでこれを改造します。

`3.`はimport情報を保存し、ある`package path`に対して、ファイル内でどういったら`ident`でアクセスできるかを連携するための機能を実装します。これは、astのrewriteによって生成されるコードは既存のファイルのimport specを参照したコードになるため必要になります。

`4.`について、astの書き換えはコメントオフセットの狂いによっておかしな出力がされるという問題があるため実際には[github.com/dave/dst]で行います。node単位でprintは`go/printer`を用います。

`5.`は次のセクションで述べます

### 収集したいメタデータ

前述したようなコードを生成するにはどのようなメタデータを収拾する必要があるかについて考えます。

つまるところ以下を行いたいわけです

- 1. und typeをフィールドに含み、そのフィールドに有効な`und:""`タグがあることの検知
- 2. さらに、上記の型を含む型をの検知と、さらにその型を含む型・・・という感じで連鎖的な型の検知
  - 連鎖的に検知し、それぞれコード生成の対象となった場合には`UndValidate`/`UndRaw`などのメソッドを実装しているという「てい」にして取り扱います。
  - そうしないと、何度もコード生成を行わないと連鎖的にすべての型に対してメソッドが生成できませんので非常に不便です。
- 3. コード生成対象外の場合でも、`UndValidate`や`UndRaw` -> `UndPlain` -> `UndRaw`の循環的な変換をサポートする型の検知
  - これらを検知して、これらを含む型を連作的に検知します。そうしないと、分割されたモジュール間での連携ができなくなって非常に不便です。

`1.`に関してはastか型情報を用いればよいでしょう。テキストとしてソースコードを読み込んでもよいと思いますが、コメントなどでパーザーが混乱させられることもあるのでロバストとはいいがたいです(`/* comment */`構文だとあらゆる箇所にコメントを入れられます。)また、`struct {}`リテラルなどで改行を含んだフィールドや、`struct {Foo, Bar string}`のように複数のフィールドを1行で書いたりできるため、テキストとして解析は案外大変だったりします。

`2.`は型情報の依存性をグラフとして解析し、フィールドにund typeを含む型・・・以後`matched type`と呼ぶ・・・をまず見つけ、その型に依存している方に向けてtraverseすることで、そういった型をフィールドに含む型・・・以後`transitive`と呼ぶ・・・を見つけるという方式をとります。
当初は`map[ident]type`なマップに`matched`を記録しておき、これらをフィールドに含む型を`transitive`としてさらにマップに記録していく方式をとっていました。この方法には明確な欠点があって、ソースコード上の出現順序と依存関係が逆だと、型の数と同じだけ解析処理を走らせないと網羅的にすべての`transitive`を発見できず、非常に非効率だし思いのほか処理の使いまわしがききませんでした。

`3.`は型情報を解析して判定することとします。`types`には[types.AssignableTo](https://pkg.go.dev/go/types@go1.23.3#AssignableTo)、[types.Implements](https://pkg.go.dev/go/types@go1.23.3#Implements)、[types.Satisfies](https://pkg.go.dev/go/types@go1.23.3#Satisfies)などがありますが、そもそもこういった循環的な関係性をinterfaceで表現する方法がわかりません。`Go`のinterfaceにはSelf type的なものがないためおそらく表現できないんじゃないかと思います。そのため、もっと手続き的な方法で直接型情報をたどって検査します。

## astおよび型情報の収集: packages.Loadによるast/型情報の取得

astと型情報の解析は[golang.org/x/tools/go/packages]を用います。

astの素朴な解析は`go/token`, `go/ast`, `go/parser`を用いることで行えます。

```go
package main

import (
	"go/parser"
	"go/token"
)

func main() {
	fset := token.NewFileSet()
	/* *ast.File */file, err := parser.ParseFile(fset, "path/to/source/file", nil, parser.ParseComments|parser.AllErrors)
	if err != nil {
		// handle error
	}
}
```

さらに型チェックも同様に`go/types`, `go/importer`によって行えます

```diff go
package main

import (
+	"go/importer"
	"go/parser"
	"go/token"
+	"go/types"
)

func main() {
	fset := token.NewFileSet()
	/* *ast.File */file, err := parser.ParseFile(fset, "path/to/source/file", nil, parser.ParseComments|parser.AllErrors)
	if err != nil {
		// handle error
	}
+	conf := &types.Config{
+		Importer: importer.Default(),
+		Sizes:    types.SizesFor("gc", "amd64"),
+	}
+	pkg := types.NewPackage(pkgPath, files[0].Name.Name)
+	typeInfo := &types.Info{
+		Types:      make(map[ast.Expr]types.TypeAndValue),
+		Defs:       make(map[*ast.Ident]types.Object),
+		Uses:       make(map[*ast.Ident]types.Object),
+		Implicits:  make(map[ast.Node]types.Object),
+		Instances:  make(map[*ast.Ident]types.Instance),
+		Scopes:     make(map[ast.Node]*types.Scope),
+		Selections: make(map[*ast.SelectorExpr]*types.Selection),
+	}
+	chk := types.NewChecker(conf, fset, pkg, typeInfo)
+	err := chk.Files(file)
+	if err != nil {
+		// handle error
+	}
}
```

ただし直接使うには少し難しい部分があります。
それはロード対象のソースコードが外部のモジュールなどをインポートしているとき、(多分)それらを手動で事前に`fset`にセットしておくなどしなければならないことです。

なので、type-checkerを使いたいならば、`go list ./...`などでチェック対象のgo moduleの依存先を事前にリストしておき、リストされたモジュールの読み込みなどを先に済ませておき、importerの実装としてそれらを返せるようにしておく必要があります(多分)。

という上記のようなことを[golang.org/x/tools/go/packages]は行います。

中身をパパッと読む限り、[golang.org/x/tools/go/packages]は`go list -json ...`によって依存モジュールを列挙、依存関係をDAG化、グラフをdepth-firstの順番でロード、タイプチェックと一通りやってくれます。

[golang.org/x/tools/go/packages]でtype checkまで行うコードは以下のようになります

```go
import "golang.org/x/tools/go/packages"

func main() {
	cfg := &packages.Config{
		Mode: packages.NeedName |
			packages.NeedTypes |
			packages.NeedSyntax |
			packages.NeedTypesInfo |
			packages.NeedTypesSizes,
		Context: ctx,
		Dir:     dir,
	}
	pkgs, err := packages.Load(cfg, "variadic", "package/match", "patterns")
	if err != nil {
		// handle error
	}
}
```

ずいぶん簡単になりましたね。

PkgPath, Syntax(`[]*ast.File`), TypeInfo(`*types.Info`)を使いたい場合、以上のようにModeビットフラグを設定します。
理由はわかりませんが、`NeedTypesSizes`フラグもないと`*types.Info`の各フィールドがpopulateされません。

## struct tagの編集

struct tagの編集機能は以下で実装します。

https://github.com/ngicks/go-codegen/tree/abc928f3177ebdb817474777b6563d1596875b0c/codegen/structtag

これと言って語るべきことはないですね。

`Go`のstdの[reflect.StructTag.Lookup](https://pkg.go.dev/reflect@go1.23.3#StructTag.Lookup)を改変してkey-valueのペアに解析できるように変更し、
[encoding/json/v2 discussion](https://github.com/golang/go/discussions/63397)の[experimental実装のタグ解析部分](https://github.com/go-json-experiment/json/blob/ebd3a8989ca1eadb7a68e02a93448ecbbab5900c/fields.go#L350)を参考に、仕様をまねて`json:"name"`のname部分はsingle quotation(`'`)でescapeしてもよい、`option:value`という形式のオプションをとっても良いという形式にしてあります。

## import情報の連携

コードはast rewriteによって生成される部分と単なるテキストの書き出しで生成される部分があり、この時、既存のimport declをそのまま使いまわすため、importされたpackage pathに対してどのようなidentでアクセスできるかを把握し、生成されるコードはそのidentを使わなければなりません。

また、生成されるコードが既存のファイルに存在していなかったimportを追加したいのはよくあると思います。例えば`fmt.Errorf`を使用するために`"fmt"`を追加するといったことですね。
この場合、既存のファイルに存在していたimportと名前が被ってしまう可能性があります。
普通に`Go`を書いてても`crypto/rand`と`math/rand/v2`がどちらも`rand`なので被ってしまいますよね。

さらに、identを指定しないimport specはインポートされるパッケージがpackage clauseで付けた名前になります。つまり、

```go
import (
    "github.com/charmbracelet/bubbletea" /* tea */
)
```

上記は`bubbletea`が`package tea`で定義されるため、`tea`でアクセスできます

https://github.com/charmbracelet/bubbletea/blob/1feb60b44b74d9a3a7dc54b90ffbecc8ffd6b40d/tea.go#L10

package pathとpackage名が違う場合、linterが警告を出す場合がありますが、上記のようなpackage pathのbaseのサブストリングである場合は出ないようですね。
`gopls`の設定で`"ui.semanticTokens": true`でsemantic tokensを有効にしてあると、`tea`の部分が緑色で表示されてidentであることがわかります。

ということで以下を行うものとします

- `[]*packages.Package`を引数に、生成対象とその依存先のモジュールのimportをリストする(`dependencies` imports)
- またcode generatorが生成するコードで使用するimportを追加できるものとする(`extra` imports)
- code generatorは、package pathを引数に`*ast.Expr`や`*dst.Expr`を生成できる
- `*ast.File`を引数に、`ident`と`package path`の関係を洗い出す
- 上記の`extra` importsやcode generatorが`*ast.Expr`のために引き出したpackage pathなどを`missing` importsとして記録しておく
- `missing` importsを`*dst.File`の`Imports`や`GenDecls`のimport declにappendする

依存先も`packages.Load`で解析したいので`cfg`の`Mode`ビットに[packages.NeedImports](https://pkg.go.dev/golang.org/x/tools@v0.27.0/go/packages#NeedImports)|[packages.NeedDeps](https://pkg.go.dev/golang.org/x/tools@v0.27.0/go/packages#NeedDeps)も加えます。

`package.Visit`を使うとdependency orderでのvisitが可能です。適当にラップすれば[iterator](https://pkg.go.dev/iter)に変換できます。

https://github.com/ngicks/go-codegen/blob/abc928f3177ebdb817474777b6563d1596875b0c/codegen/imports/parser.go#L90-L100

`*types.Package`の`Scope()`で最上位スコープをとり、`Names()`ですべての名前を列挙します。`*types.Named`か`*types.Alias`がtype specで定義できる型なので、それだけが検出されるようにフィルターをかけます。これは最近出たiterator仕様をふんだんに使っています。割と読みやすい気がする。

https://github.com/ngicks/go-codegen/blob/3a61e3289667278c98bf85b55f8dfb37a2866f6d/codegen/imports/parser.go#L52-L73

`xiter`パッケージは以前の記事で作った[モジュール](https://github.com/ngicks/go-iterator-helper)下でベンダーされたものなので、`golang.org/x/exp`に存在しているわけではないことを述べておきます。

`[]*package.Package`から解析された型情報を`dependencies`, code generatorが追加したいimportを`extra`、`*ast.File`から解析された`ident` - `package path`の関係を`ident`として保存しておきます。`extra`およびランタイムで問い合わせられたpackage pathのなかで`ident`に存在しないものは`missing`に記録します。

https://github.com/ngicks/go-codegen/blob/3a61e3289667278c98bf85b55f8dfb37a2866f6d/codegen/imports/parser.go#L109-L117

下記のような関数で`ident`から`package path`に対応するidentを取り出そうとし、ない場合`dependencies`から取り出して`missing`に記録します。

https://github.com/ngicks/go-codegen/blob/3a61e3289667278c98bf85b55f8dfb37a2866f6d/codegen/imports/parser.go#L280

https://github.com/ngicks/go-codegen/blob/3a61e3289667278c98bf85b55f8dfb37a2866f6d/codegen/imports/parser.go#L296

https://github.com/ngicks/go-codegen/blob/3a61e3289667278c98bf85b55f8dfb37a2866f6d/codegen/imports/parser.go#L311

identが被った場合に備えて`_%d`でsuffixしながらマップに追加できるようにします。

https://github.com/ngicks/go-codegen/blob/3a61e3289667278c98bf85b55f8dfb37a2866f6d/codegen/imports/parser.go#L210-L227

最後に、`*dst.File`に`missing`の内容を追加することで、のちのnode単位のast printingで出力できるようにします。

https://github.com/ngicks/go-codegen/blob/3a61e3289667278c98bf85b55f8dfb37a2866f6d/codegen/imports/parser.go#L337-L394

## dstによるastのrewrite、node単位のプリント

`Go`のastは[astutil.Apply](https://pkg.go.dev/golang.org/x/tools/go/ast/astutil#Apply)があってastの書き換えがしやすいですが、実はast上コメントはバイトオフセットで表現されており、astノードの書き換えを行った時にこのオフセットが更新されないことで出力結果が狂ってしまうという問題があります。

そこで[github.com/dave/dst]を使用します。

> The go/ast package wasn't created with source manipulation as an intended use-case. Comments are stored by their byte offset instead of attached to nodes, so re-arranging nodes breaks the output. See this Go issue for more information.

> The dst package enables manipulation of a Go syntax tree with high fidelity. Decorations (e.g. comments and line spacing) remain attached to the correct nodes as the tree is modified.

とある通り、このライブラリはまさしくこの問題を解決するために開発されています。

以下のようにすることで変換を行います。

```go
var pkgs []*packages.Package

for _, pkg := range pkgs {
    for _, file := range pkg.Syntax {
        dec := decorator.NewDecorator(pkg.Fset)
        /* *dst.File */ df, err := dec.DecorateFile(file)
        if err != nil {
            // ...
        }
    }
}
```

変換前の`*ast.File`ないの`ast.Node`から`*dst.File`内部の変換先を参照するには以下のようにします

```go
var ts *ast.TypeSpec
dts := dec.Dst.Nodes[ts].(*dst.TypeSpec)
```

`dst`のrewriteは`astutil.Apply`と対応づく`dstutil.Apply`があるため、ほぼ同じように行えます。

書き換え自体はGo source codeと紐づくast表現の規則を覚えて気合と根性で何とかします。

```go
dstutil.Apply(
    dts.Type,
    func(c *dstutil.Cursor) bool {
        node := c.Node()
        switch field := node.(type) {
        default:
            return true
        case *dst.Field:
            // replace field...
            //
            // wrapping field type with sliceund.Und
            // *unmodified field type* -> sliceund.Und[*unmodified field type*]
            c.Replace(&dst.Field{
                Names: field.Names,
                Type: &dst.IndexExpr{
                    X: &dst.SelectorExpr{
                        X: &dst.Ident{
                            Name: "sliceund",
                        },
                        Sel: &dst.Ident{
                            Name: "Und",
                        },
                    },
                    Index: field.Type,
                },
                Tag:  field.Tag,
                Decs: field.Decs,
            })
            return false
        }
    },
    nil,
)
```

さらに、書き換えた`*dst.File`を`*ast.File`へ逆変換するには以下のようにします。

```go
res := decorator.NewRestorer()
/* *ast.File */ af, err := res.RestoreFile(df)
if err != nil {
    // ...
}
```

同様に`dst.Node`から`ast.Node`を引くことができます。

```go
var dts *dst.TypeSpec
ats := res.Ast.Nodes[dts].(*ast.TypeSpec)
```

astのNode単位でのprintには[printer.Fprint](https://pkg.go.dev/go/printer@go1.23.3#Fprint)を用います。
`dst`にも[decorator.Fprint](https://pkg.go.dev/github.com/dave/dst/decorator#Fprint)がありますが、こちらは`*dst.File`単位でしかprintできません。

```go
buf := new(bytes.Buffer)
err := printer.Fprint(buf, res.Fset, ats)
```

ですので、変換前の`ast.Node`を`dst`で書き換え、さらに`printer.Fprint`したい場合は以下のようなシーケンスになります。

```go
var originalNode ast.Node
var dec *decorator.Decorator
var res *decorator.Restorer

modifiedNode := dec.Dst.Nodes[originalNode]
modifiedAstNode := res.Ast.Nodes[modifiedNode]

var w io.Writer
err := printer.Fprint(w, res.Fset, modifiedAstNode)
if err != nil {
    // ...
}
```

## und struct tagを持つund typeのフィールドの検知

[go/types]で定義される型情報を用いて、type specを走査して`und:""` struct tagのついたund typeのフィールドを持つ型(=`matched` types)を見つけます。

型周りの詳しい話は以下を読むといいかもしれません。

https://github.com/golang/example/tree/master/gotypes

何気に(予定上)`Go1.24`から導入される`generic type aliases`に合わせた更新も入ってます。

### type specに対応するtype infoを探す

[go/types]で型を探索するには、

- [Scope.Lookup](https://pkg.go.dev/go/types@go1.23.3#Scope.Lookup)を使うか
- [Info](https://pkg.go.dev/go/types@go1.23.3#Info)の`Defs`フィールドを走査する

のいずれかをします。

`Defs`から探す場合はキーの型が`*ast.Ident`なのでast情報も同様に必要になります。
今回のケースに限ってはastも探索する前提なので`Defs`から探すこととします。

以下みたいな感じです。

```go
var info *types.Info
for _, f := range []*ast.File{...} {
	for _, decl := range f.Decls {
		genDecl, ok := decl.(*ast.GenDecl)
		if !ok {
			// func or bad decl
			continue
		}
		if genDecl.Tok != token.TYPE {
			// import, constant or variable spec
			continue
		}
		for _, spec := range genDecl.Specs {
			ts := spec.(*ast.TypeSpec)
			typeInfo := info.Defs[ts.Name] // types.Object
			switch typeInfo.Type().(type) {
				case *types.Alias:
					// alias...
				case *types.Named:
					// named...
			}
		}
	}
}
```

type specのidentで`Defs`を照会した場合、得られるのは名前付き型(`*types.Named`)もしくはalias(`*types.Alias`, `type A = B`)のみのようです。

### und struct tagを持つund typeのフィールドを見つける

こうして見つけた型がund typeかつ`und:""` struct tagがついているかは以下のように探索します。

```go
var st *types.Struct = typeInfo.Type().Underlying().(*types.Struct)
for i := range st.NumFields() {
	f := st.Field(i)
	undTagValue, ok := reflect.StructTag(st.Tag(i)).Lookup("und")
	if ok {
		undOpt, err := undtag.ParseOption(undTagValue)
		if err != nil {
			return err
		}
		if !isUndType(f.Type()) {
			return fmt.Errorf("tagged but not an und type is an error")
		}
        // found
	}
}
```

`Defs`から得られた`types.Object`は`Type`メソッドで`types.Type`が得られます。これがnamed、もしくはalias typeである場合、`Underlying`でunderlying typeを取得します。

`Underlying`の用語は[Go specのそれ](https://go.dev/ref/spec#Underlying_types)と一致しており、つまるところ以下のような感じです。

```go
type Foo struct {Foo string; Bar int}
//       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//       this part is underlying
```

`type Foo`のunderlying typeは`struct {Foo string; Bar int}`というわけです。

[*types.Struct]は[reflect.StructField](https://pkg.go.dev/reflect@go1.23.3#StructField)と違ってfieldではなく[*types.Struct]に`Tag`メソッドがあり、そこからstruct tagを取得します。

上記の`isUndType`の具体的実装は以下のようになります。

```go
func isUndType(ty types.Type) bool {
	named, ok := ty.(*types.Named)
	if !ok {
		return false
	}
	obj := named.Obj()
	pkg := obj.Pkg()
	if pkg == nil {
		// 組み込み型などの場合、Pkgからnilが帰ります。
		// named typeではerror型がnilを返します。
		// types.Objectを受けとるところではPkgのnil checkはしておくほうが無難ですね。
		return false
	}
	name := obj.Name()
	pkgPath := pkg.Path()
	switch [2]string{pkgPath, name} {
	case [2]string{"github.com/ngicks/und/option", "Option"},
		[2]string{"github.com/ngicks/und", "Und"},
		[2]string{"github.com/ngicks/und/elastic", "Elastic"},
		[2]string{"github.com/ngicks/und/sliceund", "Und"},
		[2]string{"github.com/ngicks/und/sliceund/elastic", "Elastic"}:
		return true
	default:
		return false
	}
}
```

`types.Object`の`Name`でunqualified nameが得られ、`Pkg().Path()`でパッケージのパスが得られるため、これを比較すればよいです。

## 型依存関係のグラフの作成

`matched type`(フィールドに`und:""` struct tagがついたund typeを含む型)を探し出し、さらにそれらの型に依存する型を依存グラフを上に向けてたどることですべて発見するために、型情報をグラフとします。

やることは以下です

- `*types.Named`(名前付き型)の列挙
  - `type Foo ...`として定義した型の中で`type A = B`というaliasを除いたものです。
- 型をnodeとし`*types.Named`から`*types.Named`への依存をedgeとして記録。
- `matcher`を受けとり、`*types.Named`が`matched`であるかを判別
  - `matcher`はund typeや`UndValidate`、`UndRaw`/`UndPlain`のような特別な関数を満たす外部の型にもマッチするようにします。
- `matched`から上へedgeをたどって`transitive` typeをとれるようにします。
  - transitの際、edgeを上にたどるかどうかを決める`edgeFilter`を受けとり、例えば`chan A`のような依存ではたどらないものとします。

例えば、下記のようなコードがあるとき、

```go
type A struct {
    Foo und.Und[int] `und:"required"`
}

type B struct {
    A A
}

type C struct {
    A map[string]A
}

type D struct {
    A []A
}

// or even
type E struct {
    A map[string]*[3][]chan A
}
```

各依存関係は以下のようになります。

- `B` -`{struct}`-> `A`
- `C` -`{struct, map}` -> `A`
- `D` -`{struct, slice}` -> `A`
- `E` -`{struct, map, pointer, array, slice}` -> `A`

`*types.Named`から`*types.Named`へのエッジをとるとき、`array`, `chan`, `map`, `pointer`, `slice`, `struct`のような無名の型を経由することができるため、これらの関係性をコード生成にかかわるレベルで記録するものとします。

今回生成したいコードは`JSON`など外部とのデータのやり取りに用いる型を対象とするため、`chan`はedgeとして有効ではないとしたくなります。
また、`und.Und[T]`の`T`が`UndValidate`や`UndRaw` -> `UndPlain`のような特定のinterfaceを満たす時、特別なハンドリングを行いたいので、`matcher`にマッチする型はtype argも記録するものとします。

そこで、Node, Edgeは以下の通りに定義します。

https://github.com/ngicks/go-codegen/blob/3a61e3289667278c98bf85b55f8dfb37a2866f6d/codegen/typegraph/type_graph.go#L54-L66

https://github.com/ngicks/go-codegen/blob/3a61e3289667278c98bf85b55f8dfb37a2866f6d/codegen/typegraph/type_graph.go#L88-L97

[Go1.18]からgenericsが導入されたため、親から子への依存はtype argによりばらばらにinstantiateされる可能性がありますが、nodeそのものはinstantiateされてない型の定義そのものです。そのため、child側だけはNodeとTypeをそれぞれ記録する必要があります。

グラフを図にすると以下のようになります。

![](/images/go-code-generation-from-ast-and-type-info-type-graph-concept.drawio.png)

`chan`は変換の対象にならないため、edgeのフィルタリングにより連続的に`D`も対象外になります。

![](/images/go-code-generation-from-ast-and-type-info-type-graph-concept-edge-filtering.drawio.png)

edgeの形成はNode間(`*types.Name`から`*types.Named`)のみの評価であるため評価は必ず終わりますが、edgeをたどる際には無限ループが生じうるため、注意が必要です。

Treeを形成するとほぼ必ず型的な再帰が起きます。

```go
type Tree struct {
	l, r   *Tree
	value  any
}
```

そこで、お決まりですが`visited map[*node]bool`なマップを用意し、1度visitしたnodeに再度visitすることがないようにします。

https://github.com/ngicks/go-codegen/blob/3a61e3289667278c98bf85b55f8dfb37a2866f6d/codegen/typegraph/type_graph.go#L514-L544

## `UndRaw`/`UndPlain`を実装する型の検知

前述のとおり、code generatorが生成することになる`UndRaw`/`UndPlain`は`T` -> `T'` -> `T`の循環的な変換メソッドです。
これらを実装する型を検知し、`implementor`として取り扱うこととします。`implementor`に依存している型も同様に`transitive`として扱うことで、`go module`間での円滑な連携を可能とします。

`Go`のinterfaceには`Self type`を表す方法がないため、`UndRaw`/`UndPlain`はinterfaceで表現することはできず、型情報を解析して実装をチェックするよりほかありません。

ある型のmethod setを`types`を通じて得るには[types.NewMethodSet](https://pkg.go.dev/go/types@go1.23.3#NewMethodSet)を用います。

```go
type Foo struct {
}

func (f Foo) MethodOnNonPointer() {
	//
}

func (f *Foo) MethodOnPointer() {
	//
}

---
var fooObj types.Object

mset := types.NewMethodSet(fooObj.Type())
for i, sel := range hiter.AtterAll(mset) {
    t.Logf("%d: %s", i, sel.Obj().Name())
    // 0: MethodOnNonPointer
}

mset = types.NewMethodSet(types.NewPointer(fooObj.Type()))
for i, sel := range hiter.AtterAll(mset) {
    t.Logf("%d: %s", i, sel.Obj().Name())
    // 0: MethodOnNonPointer
    // 1: MethodOnPointer
}
```

通常の`Go`のルールのとおり、pointer typeでなければpointer typeがreceiverのmethodは見えません。

method setの`At`メソッドでn番目のメソッドを取得できます。ここから得られる[\*types.Selection](https://pkg.go.dev/go/types@go1.23.3#Selection)の`Type`はおそらく`[\*types.Signature](https://pkg.go.dev/go/types@go1.23.3#Signature)のみです。

上記より、`UndRaw`/`UndPlain`を実装しているかは以下のようにチェックできます。

https://github.com/ngicks/go-codegen/blob/3a61e3289667278c98bf85b55f8dfb37a2866f6d/codegen/undgen/method_checker.go#L34-L102

やってることは簡単で、与えられた`*types.Named`が特定の名前(=`UndPlain`)のメソッドを持ち、それが返す型が特定の名前(=`UndRaw`)のメソッドをもち、呼び出し結果が元の型と一致するかを判定します。
少しだけややこしいのが、関数に与えられた`*types.Named`がinstantiateされていない場合、メソッドの返り値(`toType`)がtype param(例えば`T`)でinstantiateされて、type argをもってしまうため`types.Identical`がtrueを返さないということです。
その時は`*types.Signature`経由で返されたinstantiateされた型でもう一度関数を実行することで判定を行います。

[Go]: https://go.dev/
[Go1.18]: https://tip.golang.org/doc/go1.18
[Go1.23]: https://tip.golang.org/doc/go1.23
[GoのJSONのT | null | undefinedは\[\]Option\[T\]で表現できる]: https://zenn.dev/ngicks/articles/go-json-undefined-or-null-slice
[go/ast]: https://pkg.go.dev/go/ast@go1.23.3
[go/types]: https://pkg.go.dev/go/types@go1.23.3
[*ast.ArrayType]: https://pkg.go.dev/go/ast@go1.23.3#ArrayType
[*ast.MapType]: https://pkg.go.dev/go/ast@go1.23.3#MapType
[*ast.ChanType]: https://pkg.go.dev/go/ast@go1.23.3#ChanType
[*types.Array]: https://pkg.go.dev/go/types@go1.23.3#Array
[*types.Slice]: https://pkg.go.dev/go/types@go1.23.3#Slice
[*types.Map]: https://pkg.go.dev/go/types@go1.23.3#Map
[*types.Chan]: https://pkg.go.dev/go/types@go1.23.3#Chan
[*types.Struct]: https://pkg.go.dev/go/types@go1.23.3#Struct
[golang.org/x/tools/go/packages]: https://pkg.go.dev/golang.org/x/tools@v0.26.0/go/packages
[github.com/dave/dst]: https://github.com/dave/dst
[github.com/oapi-codegen/oapi-codegen]: https://github.com/oapi-codegen/oapi-codegen
[github.com/ngicks/und]: https://github.com/ngicks/und
[sliceund.Und]: https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/sliceund#Und
[sliceelastic.Elastic]: https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/sliceund/elastic#Elastic
[Elasticsearch]: https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro.html
