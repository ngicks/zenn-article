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
`Go`のソースコードをを引数にコードジェネレーターを作成する際、単にテキストファイルとしてソースコードを解析してもよいのですが、astや型情報を用いることができたほうが改行やコメントその他で意味論的に違いのないソースコードを違いなく処理できるため、その観点からはできるならそうしたほうがいいと言えます。

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

- Partial-JSONによるPatchを実現するPatch
- 特定の型([github.com/ngicks/und]で定義される型)の状態をvalidateするValidator
- 上記特定の型を、struct tagに応じて*Plain*な型に変換するPlain

の三つを実装し、それらの構成要素で重要なものを解説します。
概説的、網羅的にはなりません。あくまで今回実装したものの周りについてのみ説明します。

## 前提知識

- `Go`の構文ルールがわかる
- `Go`におけるjsonの取り扱いがわかる
- `ast`や型情報といったものがどういうものかある程度わかる
  - `Go`固有と思しき`ast`構造についてはいくらか触れますが、`ast`という言葉自体を解説しません。
- [Go1.23]で追加されたiterator仕様と[xiter proposal](https://github.com/golang/go/issues/61898)で載せられたadapter群を知っている
  - サンプルコードは何も言わずにそれらを使います。
  - 見たらなんとなくで意味は分かるとは思います。

構文ルール(や`Go`プロジェクトの始め方)は[Goで開発して3年のプラクティスまとめ](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part1)という一連の記事でまとめています・・・というか、`A Tour of Go`をやったら30分～数時間ぐらいで分かるよということだけのべておきます。

`json`を含めたデータの取り扱いは[Goで開発して3年のプラクティスまとめ(2/4)のデータのシリアライズ/デシリアライズの項目](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part2#%E3%83%87%E3%83%BC%E3%82%BF%E3%81%AE%E3%82%B7%E3%83%AA%E3%82%A2%E3%83%A9%E3%82%A4%E3%82%BA%2F%E3%83%87%E3%82%B7%E3%83%AA%E3%82%A2%E3%83%A9%E3%82%A4%E3%82%BA)や[GoのJSONのT | null | undefinedは\[\]Option\[T\]で表現できる]ですでにそこそこ深めに述べましたので、そちらを読んでいただければわかることもあるかもしれません。

コードの中で`xiter`パッケージを使います。これは以前の記事で作った[モジュール](https://github.com/ngicks/go-iterator-helper)下でベンダーされたものなので、`golang.org/x/exp`に`xiter`パッケージが存在しているわけではありません。

基本的にはある程度実践的に`Go`を使ったことがあることを前提とします。

## 対象環境

- `linux/amd64`
  - ただし`OS`/`arch`の差は外部ライブラリによって吸収されるので影響しないものと想定します。

検証は`go 1.23.2`、リンクとして貼るドキュメントは`1.23.3`のものになります。

```
# go version
go version go1.23.2 linux/amd64
```

各種ライブラリは以下のバージョンを用います。特に[golang.org/x/tools](https://pkg.go.dev/golang.org/x/tools)は作成途中に`v0.27.0`がリリースされていますがこのバージョンで本記事に書いたいくつかの挙動が改善されているかもしれないので注意してください。

```
require (
	github.com/dave/dst v0.27.3
	github.com/google/go-cmp v0.6.0
	github.com/ngicks/go-iterator-helper v0.0.16-0.20241102133946-d622279c83c3
	github.com/ngicks/und v1.0.0-alpha5.0.20241108225608-67d88238795b
	github.com/spf13/cobra v1.8.1
	github.com/spf13/pflag v1.0.5
	golang.org/x/tools v0.26.0
	gotest.tools/v3 v3.5.1
)
```

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
- 上記の型に加えて`struct`, `string`, `int`などをベースとする(`underlying`)型を定義することができ、それらにメソッドを定義することができます。

シンプルであるがゆえにコードジェネレーターの実装が容易です。
マクロ機能は現状ありません。

[Go1.18]までgenericsが存在しておらず、それまでは上記の組み込み型のみがジェネリックに中身の型を取り換えられる型でした。

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

を`Go`のstruct fieldで表現することができます。(ただし`json:",omitempty"`を必要とする)

本記事ではこれらを用いて以下を実現するコードを生成するコードジェネレーターを実装します。

- Patcher
  - Partial JSONを受けとってデータの部分的更新(Patch)を行うことができる型を生成する。
- Validator
  - `und:""` struct tagで指定された内容に従い、フィールドのタグが`und:"required"`なら`defined`でなければならない、という風にルールを設定してvalidateを行う
  - 前述の記事で課題感は説明しましましたが、`Go`で普通にやると`T | null | undefined`をstruct fieldで表現しきるのは難しく、`null`であったか`undefined`であったかを区別できません。
  - その差が重要なときに`sliceund.Und`を利用しますが、その際に`undefined`であったならvalidではないと判定するためにこのvalidatorを使います。
- Plain
  - `und:""` struct tagで指定された内容に従い、例えばのフィールドの型が`und.Und[T]`でstruct tagが`und:"required"`なら型を`T`に*unwrap*した*Plain*な型を作成する。
  - 元となった型(_Raw_)との相互変換を実現する。
  - こうすることで、Marshal/Unmarshalの界面ではlosslessで`undefined | null | T`や`undefined | null | T | (T | null)[]`をデータ構造に当てはめ、validationなどを実施したうえで界面以外で処理するのに都合のいいデータ構造に変換してから後続の処理を行うことができます。

## 生成されるコードのイメージ

まずどういったコードを生成したら目標が実現できるかを思い描き、そこから具体的に何を実装すべきかについて考えます。

この記事で一番話したかったのは[機能の実装](#機能の実装)のところなんですが、たてつけ上説明しないと意味不明なのでここでどういったコードを生成するか紹介します。
興味なかったらとばしてください。

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
  - 付け足す、というのがキモです。元からあった`json` structタグはなるだけそのままにする必要があります。
- `FromValue`で元となった型からpatchへ変換、入力patchと`Merge`でマージ、`ToValue`で元となった型に逆変換することでパッチの挙動を実現します。
- `Merge`は[github.com/ngicks/und]の機能をふんだんに使って`Or`をとることで実現します。

元の型->パッチ型な変換は元の型にメソッドとして実現するか、New*FooBar*Patchという関数で実現するかしたほうがよかったかもしれませんが、下記理由でしませんでした。

- なるだけ元の型には何も追加したくないので、メソッドの追加もしたくありません。
  - 追加すると名前被りのリスクがあります。
  - リスク回避のために自然に感じられないPrefixをつけて被りにくくするとかがありえますが、これを避けたいわけです
- New*FooBar*Patch的な関数も同様で名前被りのリスクがあります。

### Validator

Validatorが実現したいのは、und type([github.com/ngicks/und]で定義される諸般の型)をfieldにもつstruct typeがあるとき、struct tagで`und:""`を指定すると、その内容に基づいてundefined / null / definedの状態などのvalidationを行うことです。
こうすることで、フィールドが`null`であってもいいけど`undefined`であることは許さない、というのを実現できます。これ自体は一旦`map[string]any`にUnmarshalすることで実現可能だったんですが、und typeを使う方法に比べると若干非効率的です：`map[string]any`へ変換すると想定しないフィールドを無視することができないためです。

- validator自体は[github.com/ngicks/und]で実装済みです
  - [untag.UndOpt](https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/undtag#UndOpt)としてexportしてあります
  - ただしこの型はinternal packageをフィールドに使っているため外部モジュールから初期化できません。
    - `github.com/ngicks/und/option`をinternalとしてvendorしておいたinternal版optionを使っているからです。
      - 実装上の都合で`undtag`は`option`からも依存されています。
    - `undtag`自体が元はinternal packageだったのでこれでいいと思っていたんです。
  - 苦肉の策として[undtag.UndOptExport](https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/undtag#UndOptExport)をexportしておき、これを通じて`UndOpt`を初期化するようにします。
    - こちらは`option.Option[T]`の代わりにポインターを使っています。

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

以下の`UndValidate`メソッドが出力される

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

[validate.AppendValidationErrorDot](https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/validate#AppendValidationErrorDot)と[validate.AppendValidationErrorIndex](https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/validate#AppendValidationErrorIndex)は深くネストしたフィールドのどこがvalidationエラーだったのか表示するためにフィールド名をエラーにappendできるヘルパーで、内部的には[ValidationError](https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/validate#ValidationError)でエラーをラップします。

### Plain

Plainが実現したいのは、struct fieldがund typeであり`und:""`タグが指定されているとき、このタグの内容に応じてフィールドをアンラップした*Plain*な型を作り、これと元となった型(_Raw_)との相互変換を行うことです。

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

みたいな感じでフィールドが変換された型を生成し、これと相互変換を行うことで*Plain*に感じられる型でMarshal/Unmarshal以外の処理を行えるようにします。

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

以下の、*Plain*型,`UndPlain`/`UndRaw`メソッドが出力されます。

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

さらに、フィールドがこの`UndRaw`/`UndPlain`という変換メソッドをを実装する(これを`implementor`と呼ぶ。生成対象になった型をフィールドに含む型も同様に`implementor`のように取り扱われるが、こちらは`dependant`と呼ばれる)際にはそれを呼び出せるようにします。
ObjectにObjectやArrayがネストしているJSONは普通に存在していますし、それを表現する`Go`の型は各部を別々のnamed typeとして定義するのが筆者の知る限り普通なことなので、これができないと実用に耐えません。

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

## 基本方針

設計にかかる基本的な方針を述べます。

- astのrewriteで実現する
  - 今回生成するものは`Go`のソースコードを受けとり、定義された型のフィールドを置き換えるなどするため、astをそのまま用いることができれば独自に実装しなければならないものが減ります。
  - `go/types`以下で実装される型情報だけを使っても生成できるのですが、この場合**コメント情報が消える**ようです。
    - 今回実装するものは元となる型について回っているコメントもそのまま生成されるコードに残したい意図があります。コメントがなくなってフィールドの意図がわからなくなると困るだろうということです。
- `go/types`以下で定義される型情報も用いる
  - 前述のとおり、`UndValidate`および`UndPlain`/`UndRaw`を実装する型がフィールドに含まれる場合、そちらを呼び出すようなコードを作成します。
  - `UndValidate`はastから容易に実装しているかを判別可能ですが、`UndPlain`/`UndRaw`は`T` -> `T'` -> `T`という循環的な変換を行うため、astよりも高度な型情報が必要になります。
  - また、生成の対象になった型をフィールドに含む型(以後`dependant`と呼ばれる)を探索するには、型情報を用いる必要があります。
- メソッドの作成部分(`ApplyPatch`や`UndPlain`など)は、単なるテキスト書き出しで行う
  - `github.com/dave/jennifer`は便利ですが、importを外部から管理しづらいため使うことができませんでした。
  - `text/template`はif/elseで生成内容が大幅に変わる今回のようなケースでは煩雑であるので採用しませんでした。
  - 要するにこれら二つが想定する使い方をできなさそうなので使いません。
- ast rewriteで生成した部分とテキストで書きだされたメソッド群は同じファイルに書き込む
  - こうすることでまとめて消しやすくします。
- 生成元となった型を含むソースコードのパスに`und_patch`のようなサフィックスをしたファイルに書き出す。
  - 関連性をわかりやすくしつつ、いらなくなった時に削除しやすくします。

## 実装すべき機能

前述の基本方針に従いながら実現したいコードを生成するためにはどのような機能が必要かを述べます。

### 機能

- 1. astおよび型情報の収集
- 2. struct tagの編集
- 3. import情報の連携
- 4. astのrewrite、およびrewriteによって生成されたコードと別の方法で生成されたコードを同じファイルに書き出せるようにする
- 5. astおよび型情報を使ったメタデータの収集

`1.`には[golang.org/x/tools/go/packages]を用います。

`2.`はPatch作成のためにast rewrite時に`json:",omitempty"`を各フィールドに追加するために必要です。
struct tagの解析器と編集機能を実装します。[encoding/json v2(候補)について紹介してundefined | null | Tを表現する](https://zenn.dev/ngicks/articles/go-json-undefined-or-null-v2)で解析器までは実装していたのでこれを改造します。

`3.`はimport情報を保存し、ある`package path`に対して、ファイル内でどういった`ident`でアクセスできるかを引き出したり、元のファイルには存在していなかったimport specを[PackageName](https://go.dev/ref/spec#PackageName)が被らないように追加したりするために必要です。

これは、astのrewriteによって生成されるコードは生成元のファイルのimport declのpackage pathを使ってしまうので必要となります。つまり

```go
package foo

import (
    "github.com/ngicks/und"
    baaaaaar "example.com/foo/bar"
)

type Foo struct {
    A und.Und[baaaaaar.Bar] `und:"required"`
}
```

というコードがあるとき、`Plain`機能で生成されるコードは以下のようになります。ast上では`baaaaaar`は単なる文字列なので、

```go
type FooPlain struct {
    A baaaaaar.Bar
}
```

となります。単にテキストとして書き出されるメソッド群(`UndValidate`, `UndPlain`/`UndRaw`)でもこの`baaaaaar`という名前を用いることで、元のソースからの統一感を損なわないようにします。そのためimportを記録し、`PackagePath`から`ident`(この場合`baaaaaar`)を引き出せる情報ストアが必要です。

`4.`について、astの書き換えはコメントオフセットの狂いによっておかしな出力がされるという問題があるため実際には[github.com/dave/dst]で行います。node単位でprintは`go/printer`を用います。

`5.`は次のセクションで述べます

### 収集したいメタデータ

前述したようなコードを生成するにはどのようなメタデータを収拾する必要があるかについて考えます。

つまるところ以下を行いたいわけです

- 1. und typeをフィールドに含み、そのフィールドに有効な`und:""`タグがあることの検知し、これを**生成対象の型**とする。
- 2. さらに、上記の型を含む型をの検知と、さらにその型を含む型・・・という感じで連鎖的な型(`dependant`)の検知
  - 連鎖的に検知し、それぞれコード生成の対象となった場合には`UndValidate`/`UndRaw`などのメソッドを実装しているという「てい」にして取り扱います。
  - そうしないと、何度もコード生成を行わないと連鎖的にすべての型に対してメソッドが生成できませんので非常に不便です。
- 3. コード生成対象外の場合でも、`UndValidate`や`UndRaw` -> `UndPlain` -> `UndRaw`の循環的な変換を実装する型(`implementor`)の検知
  - これらを検知して、これらを含む型を連作的に検知します。そうしないと、分割されたモジュール間での連携ができなくなって非常に不便です。
- 4. 連鎖的に検知された型(`dependant`)から、生成されることになるはずの型を`*types.Named`として生成する
  - `implementor`の`UndPlain`/`UndRaw`による変換先の型は、型情報から取得可能ですが、`dependant`はまだ型を書き出していないので、`implementor`同様の方法では変換先を得られません
  - ただしcode generatorはどのような型を書き出すことになるのかを知っています。
  - `implementor`と`dependant`をほぼ同じように取り扱いたいならば、`dependant`の変換先も同じフォーマットで得られたほうが良いです。

`1.`には型情報を用います。

`2.`は型情報の依存性をグラフとして解析し、フィールドにund typeを含む型・・・以後`matched type`と呼ぶ・・・をまず見つけ、その型に依存している方に向けてtraverseすることで、そういった型をフィールドに含む型・・・以後`dependant`と呼ぶ・・・を見つけるという方式をとります。
当初は`map[ident]type`なマップに`matched`を記録しておき、これらをフィールドに含む型を`dependant`として同じマップに記録していく方式をとっていました。この方法には明確な欠点があって、ソースコード上の出現順序と依存関係が逆だと、型の数と同じだけ解析処理を走らせないと網羅的にすべての`dependant`を発見できず、非常に非効率だし思いのほか処理の使いまわしがききませんでした。

`3.`は型情報を解析して判定することとします。`types`には[types.AssignableTo](https://pkg.go.dev/go/types@go1.23.3#AssignableTo)、[types.Implements](https://pkg.go.dev/go/types@go1.23.3#Implements)、[types.Satisfies](https://pkg.go.dev/go/types@go1.23.3#Satisfies)などがありますが、これらを用いて`UndPlain`/`UndRaw`の検知を実現することはおそらくできません。
これらの方法は、型や値が`interface`を満たすかどうかのチェックを`types`の機能として提供しますが、そもそも`UndRaw`/`UndPlain`による`T` -> `T'` -> `T`の変換はinterfaceとして表現する方法がわかりません。`Go`のinterfaceにはSelf type的なものがないためおそらく表現できないんじゃないかと思います。そのため、もっと手続き的な方法で型情報をたどって検査します。

`4.`は、型情報を用いて元となった型(_Raw_)から`*types.Named`を作成します。後述です。

## 機能の実装

### astおよび型情報の収集: packages.Loadによるast/型情報の取得

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
    /* *ast.File */file, err := parser.ParseFile(
        fset,
        "path/to/source/file",
        nil,
        parser.ParseComments|parser.AllErrors,
    )
    if err != nil {
        // handle error
    }
}
```

さらに型チェックも同様に`go/types`, `go/importer`によって行えます

```diff go
package main

import (
+   "go/importer"
    "go/parser"
    "go/token"
+   "go/types"
)

func main() {
    fset := token.NewFileSet()
    /* *ast.File */file, err := parser.ParseFile(fset, "path/to/source/file", nil, parser.ParseComments|parser.AllErrors)
    if err != nil {
        // handle error
    }
+   conf := &types.Config{
+       Importer: importer.Default(),
+       Sizes:    types.SizesFor("gc", "amd64"),
+   }
+   pkg := types.NewPackage(pkgPath, files[0].Name.Name)
+   typeInfo := &types.Info{
+       Types:      make(map[ast.Expr]types.TypeAndValue),
+       Defs:       make(map[*ast.Ident]types.Object),
+       Uses:       make(map[*ast.Ident]types.Object),
+       Implicits:  make(map[ast.Node]types.Object),
+       Instances:  make(map[*ast.Ident]types.Instance),
+       Scopes:     make(map[ast.Node]*types.Scope),
+       Selections: make(map[*ast.SelectorExpr]*types.Selection),
+   }
+   chk := types.NewChecker(conf, fset, pkg, typeInfo)
+   err := chk.Files(file)
+   if err != nil {
+       // handle error
+   }
}
```

ただし直接使うには少し難しい部分があります。
type checkerは外部モジュールをimportする際に、`*types.Config`の`Importer`フィールドで受け取ったImporterを使用しますが、stdの[go/importer](https://pkg.go.dev/go/importer@go1.23.3)で提供されるipmorter実装は`go module` awareではないのか、デフォルトでは`go get`でキャッシュされたモジュールをロードしてくれません。

ではどうするのかというと、importerを自作する必要があります。

- `go list --json --deps=true -- ./package/specifier`で依存を含むすべてのソースコードを列挙
- モジュールのdependency graphを作成、末端(=なにもimportしない)から順次ロード
- `Importer`実装として、Package pathを与えられたら`*types.Package`を返すように、interfaceを実装する。

上記の課題を解決してくれるのが[golang.org/x/tools/go/packages]です。

中身を読む限り、[golang.org/x/tools/go/packages]は`go list --json --deps=true -- ./package/specifier`によって依存モジュールを列挙、依存関係をDAG化、グラフをdepth-firstの順番でロード、type checkと上記のことを一通りやってくれます。
上で上げたナイーブな実装とは違い、CGOやPGOの考慮や`go list`以外のdriverのサポート、package patternが多すぎる際にmax safe cli arg以下になるようにコマンド実行の分割、ロードのconcurrent化、type check失敗時の考慮などしっかり作りこまれています。

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

`packages.Load`で[[]\*packages.Package](https://pkg.go.dev/golang.org/x/tools@v0.26.0/go/packages#Package)が返され、これの各種フィールドがastや型情報となります。
`Package`の`PkgPath`, `Syntax`(`[]*ast.File`), `TypeInfo`(`*types.Info`)を使いたい場合、上記のように`Mode`ビットフラグを設定します。
`NeedTypesSizes`フラグもないと`*types.Info`の各フィールドがpopulateされません。

### 特定のast.Nodeの無視

`packages.Config`には[ParseFile](https://pkg.go.dev/golang.org/x/tools@v0.27.0/go/packages#Config.ParseFile)という項目があり、これによってロードの挙動をカスタマイズ可能です。

```go
    ParseFile func(fset *token.FileSet, filename string, src []byte) (*ast.File, error)
```

なにも指定されなければ下記と同等のコードが使用されます

```go
    return parser.ParseFile(fset, filename, src, parser.AllErrors|parser.ParseComments)
```

これを利用し、**デバッグ時に限り特定のコメントがついたノードをParse時に無視する**ものとします。

- デバッグ目的では無視したい
  - 今回動作させたいcode generatorは対象のディレクトリにコードを書き込みます。
  - 型情報用いるため、ソースコードはパッケージ単位で処理されますが、２度目以降の生成では生成したコードも型チェックの対象に入ってしまい、結果が変わりえてしまいます。
  - これによってgeneratorの実装不備がわかりにくくなります。実際できてると思ったらできてなかったというのが何度かおきました。
- 本番では無視したくない
  - ユーザーがパッケージ内に生成された型/メソッドに依存するコードを追加したとき、code generatorがそれらのnodeを無視してしまうとでtype check時にエラーが起きます。

無視できるようにするために、生成されるコードの各Declには`//undgen:generated`というコメントを必ず付与することとします。

`//undgen:`で始まるコメントをパーズする機能を`ParseUndComment(cg *ast.CommentGroup)`として定義していることを前提とすると、単純な発想では以下のような実装を用いれば`//undgen:generated`というコメントがついたノードを無視できます。

```go
func ParseFile(fset *token.FileSet, filename string, src []byte) (*ast.File, error) {
    f, err := parser.ParseFile(fset, filename, src, p.mode)
    if err != nil {
        return f, err
    }

    f.Decls = slices.AppendSeq(
        f.Decls[:0],
        xiter.Filter(
            func(decl ast.Decl) bool {
                var (
                    direction UndDirection
                    ok        bool
                    err       error
                )
                switch x := decl.(type) {
                case *ast.FuncDecl:
                    direction, ok, err = ParseUndComment(x.Doc)
                case *ast.GenDecl:
                    direction, ok, err = ParseUndComment(x.Doc)
                    if direction.generated {
                        return false
                    }
                    x.Specs = slices.AppendSeq(
                        x.Specs[:0],
                        xiter.Filter(
                            func(spec ast.Spec) (pass bool) {
                                var (
                                    direction UndDirection
                                    ok        bool
                                    err       error
                                )
                                switch x := spec.(type) { // IMPORT, CONST, TYPE, or VAR
                                default:
                                    return true
                                case *ast.ValueSpec:
                                    direction, ok, err = ParseUndComment(x.Comment)
                                case *ast.TypeSpec:
                                    direction, ok, err = ParseUndComment(x.Comment)
                                }
                                if !ok || err != nil {
                                    // no error at this moment
                                    return true
                                }
                                return !direction.generated
                            },
                            slices.Values(x.Specs),
                        ),
                    )
                }
                if !ok || err != nil {
                    // no error at this moment
                    return true
                }
                return !direction.generated
            },
            slices.Values(f.Decls),
        ),
    )
    return f, err
}
```

しかし上記のコードは以下の２点において正しくありません

- unused import
  - 削除されたノードによってのみ参照されていたimportが存在するとき、unused importが生じます。
  - `Go`はunused importをcompilation errorとします
    - specを見る限りunused importやunused variableについての記述がないので、おそらくですが言語仕様ではく実装の制限です。
- commentが取り残される
  - `go/ast`はコメントをバイトオフセットとして取り扱います。
  - `Decl`を削除しても、`*ast.File.Comments`にコメントはすべて残っているため、print時にこれらが現れてしまします。

そこでさらに、

- importを修正するために[goimports](https://pkg.go.dev/golang.org/x/tools/cmd/goimports)と同等の機能を使用してimportを修正してもらいます
- ノード削除時にはそのノードと、アタッチされたコメントのオフセットを記録し、`*ast.File.Comments`がその範囲に収まる場合はそれを削除する機能を加えます。
  - 単に`Decl`にアタッチされたコメントを消しただけでは、function bodyやdeclの中でフィールドにアタッチされたコメントが削除されませんので範囲で削除する必要があります。

ということですべて盛り込むと以下になります。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/ignore_undgen_generated_files.go#L17-L182

- `type tokenRange []token.Pos`で消したDeclのrangeを記録し、その間にあるコメントすべてを削除します。
- 一旦`printer.Fprint`で`*ast.File`をテキストで出力し、
- `"golang.org/x/tools/imports".Process`でunused importを削除してもらいます。
  - これは`goimports`が内部で使っているのと同じ機能です。
  - parse前のソースコードが正しくコンパイル可能であるという前提が成立している限り、`Process`が新しくインポートを追加することはありません。
- フォーマット結果をもう一度`parser.ParseFile`で解析して結果を返します。

### struct tagの編集

struct tagの編集機能は以下で実装します。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/structtag/tag.go

単なるテキスト処理であり、特筆すべきことはないため詳細な説明は省きます。

`Go`のstdの[reflect.StructTag.Lookup](https://pkg.go.dev/reflect@go1.23.3#StructTag.Lookup)を改変してkey-valueのペアに解析できるように変更し、
[encoding/json/v2 discussion](https://github.com/golang/go/discussions/63397)の[experimental実装のタグ解析部分](https://github.com/go-json-experiment/json/blob/ebd3a8989ca1eadb7a68e02a93448ecbbab5900c/fields.go#L350)を参考に、仕様をまねて`json:"name"`のname部分はsingle quotation(`'`)でescapeしてもよい、`option:value`という形式のオプションをとっても良いという形式にしてあります。

### import情報の連携

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

上記は`bubbletea`が`package tea`で定義されるため、`tea`でアクセスできます。

https://github.com/charmbracelet/bubbletea/blob/1feb60b44b74d9a3a7dc54b90ffbecc8ffd6b40d/tea.go#L10

この挙動はspec上実装依存であると述べられており、別のコンパイラが提供された違った挙動をすることもあり得ますが、まあ基本的にこの挙動は保たれるでしょう。

package pathのbase nameとpackage nameが違う場合、linterがpackage nameでimportすべきだと警告を出す場合がありますが、上記のようなpackage pathのbaseのサブストリングである場合は出ないようですね。
`gopls`の設定で`"ui.semanticTokens": true`でsemantic tokensを有効にしてあると、`tea`の部分が緑色(色はカラースキームによって異なる)で表示されて`tea`でこのpackageにアクセスできることがわかります。

ということで以下を行うものとします

- `[]*packages.Package`を引数に、生成対象とその依存先のモジュールのimportをリストする(`dependencies` imports)
- またcode generatorなどの外部のパッケージが任意にimportを追加できるものとする(`extra` imports)
- code generatorは、package pathを引数に`*ast.SelectorExpr`や`*dst.SelectorExpr`を生成できる
- `*ast.File`を引数に、`ident`と`package path`の関係を洗い出す
- 上記の`extra` importsやcode generatorが`*ast.SelectorExpr`のために引き出したpackage pathのうち、`*ast.File`に含まれていなかったものを`missing` importsとして記録しておく
- `missing` importsを`*dst.File`の`Imports`や`GenDecls`のimport declにappendする

上記より[\*packages.PackageのImportsフィールド](https://pkg.go.dev/golang.org/x/tools@v0.27.0/go/packages#Package.Imports)も`packages.Load`によってpopulateされほしいので`*packages.Config`の`Mode`ビットに[packages.NeedImports](https://pkg.go.dev/golang.org/x/tools@v0.27.0/go/packages#NeedImports)|[packages.NeedDeps](https://pkg.go.dev/golang.org/x/tools@v0.27.0/go/packages#NeedDeps)も加えます。

`[]*package.Package`を列挙するには`packages.Visit`を呼び出します。`Visit`は`[]*package.Packages`をdependency orderかつ重複を排除しながらtraverseする機能を呈要します。
適当にラップすれば[iterator](https://pkg.go.dev/iter)に変換できます。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/imports/parser.go#L90-L100

`[]*package.Package`から解析された型情報を`dependencies`, code generatorが追加したいimportを`extra`、`*ast.File`から解析された`ident` - `package path`の関係を`ident`として保存しておきます。`extra`およびcode generator動作中に問い合わせられたpackage pathのなかで`ident`に存在しないものは`missing`に記録します。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/imports/parser.go#L109-L117

下記のような関数で`ident`から`package path`に対応するidentを取り出そうとし、ない場合`dependencies`から取り出して`missing`に記録します。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/imports/parser.go#L280

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/imports/parser.go#L296

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/imports/parser.go#L311

identが被った場合に備えて`_%d`でsuffixしながらマップに追加できるようにします。これにより`math/rand/v2`をインポート済みのファイルに`crypto/rand`を追加しようとすると、`import rand_1 "crypto/rand"`という風に追加されることになります。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/imports/parser.go#L210-L227

最後に、`*dst.File`に`missing`の内容を追加することで、のちのnode単位のast printingで追加されたimportも出力できるようにします。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/imports/parser.go#L337-L394

### dstによるastのrewrite、node単位のプリント

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

`astutil.Apply`の代替となる`dstutil.Apply`があるため、rewriteはastと同じように行えます。

書き換え自体はGo source codeと紐づくast表現の規則を覚えて気合と根性で何とかします。
[ast.Fprint](https://pkg.go.dev/go/ast@go1.23.3#Fprint)でastの構造をprintできるのでソースを解析してprintしてを繰り返して構造を覚えます。

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
                    Index: field.Type,// *unmodified field type*
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

ですので、変換前の`ast.Node`から、modifyしてrestoreした後の`ast.Node`をたどってprintするには以下のようにします。

```go
var originalNode ast.Node

dec := decorator.NewDecorator(fset)
df /* *dst.File */, err := dec.DecorateFile(afile)
if err != nil {
    // ...
}

dNode := dec.Dst.Nodes[originalNode]

modify(dNode)

res := decorator.NewRestorer()
_ /* *ast.File */, err := res.RestoreFile(df)
if err != nil {
    // ...
}
modifiedAstNode := res.Ast.Nodes[dNode]

var w io.Writer
err := printer.Fprint(w, res.Fset, modifiedAstNode)
if err != nil {
    // ...
}
```

### und struct tagを持つund typeのフィールドの検知

[go/types]で定義される型情報を用いて、type specを走査して`und:""` struct tagのついたund typeのフィールドを持つ型(=`matched` types)を見つけます。

型周りの詳しい話は以下を読むといいかもしれません。

https://github.com/golang/example/tree/master/gotypes

何気に(予定上)`Go1.24`から導入される`generic type aliases`に合わせた更新も入ってます。

#### type specに対応するtype infoを探す

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
            typeInfo /* types.Object */ := info.Defs[ts.Name]
            switch ty := typeInfo.Type().(type) {
                case *types.Alias:
                    // alias...
                case *types.Named:
                    // named...
            }
        }
    }
}
```

type specのidentで`Defs`を走査した場合、得られるのは名前付き型([*types.Named])もしくはalias([\*types.Alias](https://pkg.go.dev/go/types@go1.23.3#Alias), `type A = B`)のみのようです。

#### und struct tagを持つund typeのフィールドを見つける

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

`Defs`から得られた[types.Object]の`Type`メソッドで[types.Type] interfaceが得られます。前述どおり実際型はnamedかaliasのみです。
aliasは無視するものとしてnamedの場合、ここで得られているのは名前だけですので、具体的なstruct fieldを探索するためにはそれの`underlying type`を`Underlying`メソッドで取り出します。

`Underlying`の用語は[Go specのそれ](https://go.dev/ref/spec#Underlying_types)と一致しており、つまるところ以下のような感じです。

```go
type Foo struct {Foo string; Bar int}
//       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//       this part is underlying
```

`type Foo`のunderlying typeは`struct {Foo string; Bar int}`というわけです。

[*types.Struct]は[reflect.StructField](https://pkg.go.dev/reflect@go1.23.3#StructField)と違ってfieldではなく[*types.Struct]に`Tag`メソッドがあり、それから`i`番目のフィールドのstruct tagを取得します。

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

[types.Object]の`Name`でunqualified nameが得られ、`Pkg().Path()`でパッケージのパスが得られるため、これを比較すればよいです。
コメントにある通り、`error`組み込み型は、組み込み型だからpackageが存在しませんがnamed typeです。なので`Pkg()`がnilを返します。nilチェックは必須です。

### 型依存関係のグラフの作成

`matched type`(フィールドに`und:""` struct tagがついたund typeを含む型)を探し出し、さらにそれらの型に依存する型を依存グラフを上に向けてたどることですべて発見するために、型情報をグラフとします。

やることは以下です

- [*types.Named]\(名前付き型\)の列挙
  - `type Foo ...`として定義した型の中で`type A = B`というaliasを除いたものです。
- 型をnodeとし[*types.Named]から[*types.Named]への依存をedgeとして記録。
- `matcher`を受けとり、`*types.Named`が`matched`であるかを判別
  - `matcher`はund typeや`UndValidate`、`UndRaw`/`UndPlain`のような特別な関数を満たす外部の型にもマッチするようにします。
    - マッチしたもので、`Load`で得た`[]*packages.Package`で直接ロードされたpackage以外は`external`としてマークします。
- `matched`から上へedgeをたどって`dependant` typeをとれるようにします。
  - transitの際、edgeを上にたどるかどうかを決める`edgeFilter`を受けとり、例えば`chan A`のような依存ではたどらないものとします。

#### \*types.Namedの列挙/nodeの記録

[*types.Named]の列挙は[astおよび型情報の収集: packages.Loadによるast/型情報の取得](#astおよび型情報の収集%3A-packages.loadによるast%2F型情報の取得)で説明した通り、`[]*packages.Package`の`Syntax`(\[\][*ast.File])をiterateして見つかった各[*ast.TypeSpec]のNameで`TypesInfo`([*types.Info])の`Defs`を引きます。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/typegraph/type_graph.go#L239-L298

型情報にはどのtype specがグルーピングされていたとか、コメントとかは直接現れないため`*ast.GenDecl`, `*ast.TypeSpec`でもフィルタリングをかけられるようにします。
`matcher`にマッチしたとき、nodeのMatchedビットを立てます。

`Node`は以下のように定義されます

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/typegraph/type_graph.go#L57-L69

#### edgeの記録

Edgeは以下の通りに定義します。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/typegraph/type_graph.go#L91-L100

[*types.Named]から[*types.Named]への経路をたどり、`map`,`slice`,`array`,`pointer`,`channel`のような無名の型の情報を`Stack`として記録します。
以下のように順繰りに型を*unwrap*しながら経路情報を記録します。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/typegraph/type_graph.go#L419-L462

各`node`は[*types.Named]であるため、`Underlying`でtraverseをかけます。
たどり着いたnamed typeが`matcher`にマッチしたとき、`node`としてすでに格納されていないならば外部タイプであるので、`external`ビットを立てます。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/typegraph/type_graph.go#L324-L375

`external`としてマッチした時のみ、type argも記録します。type argの記録時にはnamed type出ないことも許容します。
`und.Und[T]`の`T`が`UndValidate`や`UndRaw` -> `UndPlain`のような特定のinterfaceを満たす時、特別なハンドリングを行いたのでtype argの記録が必要でした。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/typegraph/type_graph.go#L377-L417

`edge`は親子に双方に描きます。グラフをtraverseするときは`edge`を子から親に向けてたどりますが、code generatorは子の情報を使うからです。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/typegraph/type_graph.go#L464-L490

#### graphの例とedge filteringについて

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

`{struct, map}`のような形で、`array`, `chan`, `map`, `pointer`, `slice`, `struct`のような無名の型をたどった経路を記録します。これが前述の`Stack`です。

[Go1.18]からgenericsが導入されたため、親から子への依存はtype argによりばらばらにinstantiateされる可能性がありますが、nodeそのものはinstantiateされてない型の定義そのものです。そのため、child側だけはNodeとTypeをそれぞれ記録する必要があります。

![](/images/go-code-generation-from-ast-and-type-info-type-graph-node-can-be-accessed-differently.drawio.png)

`Foo` nodeには複数のtype argをもってedgeが書かれることになります。

グラフを図にすると以下のようになります。

![](/images/go-code-generation-from-ast-and-type-info-type-graph-concept.drawio.png)

今回生成したいコードは`JSON`など外部とのデータのやり取りに用いる型を対象とするため、`chan`を`Stack`に含む`edge`は対象にしません。
そこで`edge`のtraverse時に`edge`をフィルターする機能を持つものとします。

上記の図では`Und`から`C`の`edge`は`chan`を含むものしかないため、それ以上辿らないものとします。
連鎖的に`D`も`dependant`ではないという風に取り扱います。`A`はmatchするため`matched`、`B`は連鎖的に`dependant`として判定されます。

![](/images/go-code-generation-from-ast-and-type-info-type-graph-concept-edge-filtering.drawio.png)

#### graphのtraverse

graphのtraversalは`matched`、`external`を起点に`edge`を親に向けてたどります。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/typegraph/type_graph.go#L504-L534

`edgeの`形成は`Node`間(`*types.Name`から`*types.Named`)のみの評価であるため評価は必ず終わりますが、edgeをたどる際には無限ループが生じうるため、注意が必要です。
例えばTree型は型的に再帰することで木構造を形成することが多いため、この場合nodeが循環します。visit処理はこれらで無限ループに陥らないようなケアが必要です。

```go
type Tree struct {
    l, r   *Tree
    value  any
}
```

そこで、お決まりですが`visited map[*node]bool`なマップを用意し、1度visitしたnodeに再度visitすることがないようにします。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/typegraph/type_graph.go#L536-L566

### `UndRaw`/`UndPlain`を実装する型の検知

前述のとおり、code generatorが生成することになる`UndRaw`/`UndPlain`は`T` -> `T'` -> `T`の循環的な変換メソッドです。
これらを実装する型を検知し、`implementor`として取り扱うこととします。`implementor`に依存している型も同様に`dependant`として扱うことで、`go module`間での円滑な連携を可能とします。

`Go`のinterfaceには`Self type`を表す方法がないため、`UndRaw`/`UndPlain`はinterfaceで表現することはできず、型情報を解析して実装をチェックするよりほかありません。

ある型のmethod setを`types`を通じて得るには[types.NewMethodSet](https://pkg.go.dev/go/types@go1.23.3#NewMethodSet)を用います。
例を下に示します。

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
基本的に[*types.Named]を受けとって`types.NewPointer`でポインターに包んでからmethod setをとることになります。

method setの`At`メソッドでn番目のメソッドを[\*types.Selection](https://pkg.go.dev/go/types@go1.23.3#Selection)として得られます。
これは[\*types.Signature](https://pkg.go.dev/go/types@go1.23.3#Signature)なのでtype assertionしてから利用します。

以上より`UndRaw`/`UndPlain`を実装しているかは以下のようにチェックできます。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/method_checker.go#L34-L102

[*types.Named]を受けとって`types.NewPointer`で包んでからmethod setを取得し、所望の名前のメソッドを探します。
返り値の型も`types.NewPointer`で包んでからmethod setを取得し、所望の名前のメソッドを探して、それの返り値が最初に入力された型かをチェックします。

ここで考慮しなければならないのが、入力されたnamed type `ty`がinstantiateされていない場合です。instantiateされていない型、つまり`type Foo[T any]`のような型からメソッドの返り値をとると、そのtype param `T`でinstantiateされた`FooPlain[T]`が返ります。`FooPlain[T]`の`UndRaw`から返ってくる型は`Foo[T]`であり、`type Foo[T any]`という具体的にinstantiateされていないtype paramだけを持つ状態で食い違うため同じ型ではないと判定されます。
そのため元の型`ty`が`TypeParam`を持つが`TypeArg`を持たない(=instantiateされていない)ときはメソッドが返した型でもう1度`isConversionMethodImplementor`を実行します。

### \*types.Namedの生成

`UndRaw`/`UndPlain`を実装する型の、`UndPlain`で返される型は上記`*types.Named`の探索によって行われます。

`dependant`は`UndPlain`を実装するものとして取り扱われますが、こちらの場合はコードが生成されていないため上記と同じ`*types.Named`を探索しただけでは変換先の型を取り出すことができませんが、`implementor`と同じように`*types.Named`で変換先を渡せると扱いを統一できてよいのでそうします。

そこで、変換前の`*types.Named`をベースに変換後の型を生成します。
`types.NewNamed`でメソッドセットを受けとりますが、これ自体に作成した`*types.TypeName`が必要であるため関数分離の都合上callbackを受けとってメソッドセットを作成します。この例では元となった型の`Underlying`をそのまま`SetUnderlying`に渡しますが、ここに渡す型を`interface`、`map`など好きな型に変えることで任意のnamed typeを作ることができます。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_common.go#L111-L131

```go
    aa := types.TypeString(instantiated, (*types.Package).Name)
    _ = aa
```

はデバッガで見るように残してあるだけなの気にしないでください。(dead code eliminationで消えるはずなのでそのままでも問題ないはず)

具体的な呼び出し例は以下になります。
元の型に+`"Plain"`をつけた名前で型を作り、メソッドは`UndRaw`だけを持ちます。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_plain.go#L155-L187

この`UndRaw`が参照されることは今回の実装では一度もなかったですが、実験的に`types.TypeString`でプリントして正しくシグネチャが作成できていることは確認しています。

この辺の処理はtype checkerそのものを参考にしました。
`types`のdoc commentを読むだけでは少々分かりにくかったですが、type checkerはinstantiateまでやりますから、全く同じ方法はとっていませんが参考になりました。

## code generatorの実装

最後に具体的な今回実装したかったcode generatorの実装方法に入っていきます。

やりたいことは大まかに二つで

- 入力となる型を受けとって変更し、テキストとして出力
- 入力をreceiverとしたメソッド(Patcher)、生成した型をreceiverとしたメソッド(Validator/Plain)をテキストとして出力

これらに対して、

- 型の書き出し => dstをrewriteして`printer.Fprint`
  - package clause, import specも`printer.Fprint`でprintします。
- メソッドの出力 => [*bufio.Writer] + [fmt.Fprintf]

を行います。

まず`printer.Fprint`によるprintの方法と`bufio.Writer`+`fmt.Fprint`による書き出しの方針と利点などの説明をし、`Patch`,`Validator`,`Plain`の具体的な変換や生成方法について述べてます。

### printer.Fprintによるprint

#### package, importのprint

package clause, import specのprintは以下のようにします。

```go
var (
    w io.Writer
    af *ast.File
)

_, err := fmt.Fprintf("%s %s\n\n", token.PACKAGE.String(), af.Name.Name)
if err != nil {
    // error...
}

for i, dec := range af.Decls {
    genDecl, ok := dec.(*ast.GenDecl)
    if !ok {
        continue
    }
    if genDecl.Tok != token.IMPORT {
        // it's possible that the file has multiple import spec.
        // but it always starts with import spec.
        break
    }
    err := printer.Fprint(w, fset, genDecl)
    if err != nil {
        // error...
    }
    _, err = io.WriteString(w, "\n\n")
    if err != nil {
        // error...
    }
}

// successful
```

正しく構成されたastならば必ずファイルはimport specから始まりますので、import spec以外の`*ast.GenDecl`が見つかるまで`Decls`をループで回せばよいです。
import decl自体が複数あることは許されているのでそこには注意しましょう。

```go
package foo

import "fmt"
import "crypto/rand"
import "net/http"
// ...
// こういうのもたまに見る
```

#### typeのprint

前述した通り型情報を事前にグラフ化してたどりながら生成していきますが、それぞれの`*TypeNode`は以下のように、`*ast.TypeSpec`も収集してあります。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/typegraph/type_graph.go#L57-L69

そのため、前述の「original ast.Node -> modified dst.Node -> modified ast.Node」を順繰りに参照し、`Fprint`することができます。

ただし、`*ast.TypeSpec`は`type`キーワードがないので手動で出力する必要があります。`type`キーワードがくっついてるのは`*ast.GenDecl`のほうです。
つまり、下記のような関係です。

```go
type Foo struct {}
//^^^^^^^^^^^^^^^^ GenDecl
//   ^^^^^^^^^^^^^ TypeSpec
```

これは`*ast.GenDecl`が複数の`*ast.TypeSpec`をもてることを考えると事情が理解しやすいかもしれません。

```go
type (
    Foo struct{}
    //^^^^^^^^^^ TypeSpec
    Bar struct{}
    //^^^^^^^^^^ TypeSpec
    Baz struct{}
    //^^^^^^^^^^ TypeSpec
)
//^^^^^^^^^^^^^^ GenDecl
```

ということで、`printer.Fprint`の前に`type`キーワード、`' '`(スペース)を出力しておきます。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_plain.go#L110-L112

(上記の`ats`は`*ast.TypeSpec`)

### \*bufio.Writer + fmt.Fprintfによるメソッドの書き出し

メソッドの書き出しには[*bufio.Writer]と[fmt.Fprintf]を用います。

```go
var w io.Writer
bufw := bufio.NewWriter(w)
defer bufw.Flush()
printf := func(format string, args ...any) {
    fmt.Fprintf(bufw, format, args...)
}
```

理由は単純で、エラーの発生もバッファーしておけることです。

https://github.com/golang/go/blob/go1.23.3/src/bufio/bufio.go#L673-L690

https://github.com/golang/go/blob/go1.23.3/src/bufio/bufio.go#L632-L635

このことで細かいエラーハンドリングを生成途中のコードから隠すことができます。
defer内で`Flush`を呼ぶことで実際の書き出しを行いながらバッファーしたエラーを回収することができます。

```go
func generateFancyMethods(w io.Writer) (err error) {
    bufw := bufio.NewWriter(w)
    defer func() {
        fErr := bufw.Flush()
        if err == nil {
            err = fErr
        }
    }()
    printf := func(format string, args ...any) {
        fmt.Fprintf(bufw, format, args...)
    }

    printf(
        `func (fancy *Fancy) SuperGoodMethodName() string {
            return %q + %q + %q
        }
`,
        "foo", "bar", "baz",
    )
    // continue printing...
}
```

上記の`bufio.Writer`でラップするのはヘルパーを定義して、以後はこちらを使います。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_common.go#L73-L80

printする際には[fmtのExplicit argument indexes](https://pkg.go.dev/fmt@go1.23.3#hdr-Explicit_argument_indexes)を用いると便利です。
リンク先でも述べられていますが、format stringの中で`%[d]verb`(dは任意の1-indexed integer)とすると`d`番目の引数をprintできます。今回作りたいcode generatorはこれだけで事足りてしまいます。

[playground](https://go.dev/play/p/iiUdIcaEHcJ)

```go
package main

import "fmt"

func main() {
    fmt.Printf(
        "%[1]s, %[1]s, %[3]s, %[2]s, %[3]s\n",
        "foo", "bar", "baz",
    )
    // foo, foo, baz, bar, baz
}
```

code generatorを作るとなると[text/template]か[github.com/dave/jennifer]が思いつくかと思いますが、下記がそれらを使わない理由です。

- [text/template]は煩雑
  - 条件分岐によって生成されるコードがかなり変わるため、`text/template`で書ききると煩雑です
  - `Go`でif/elseをたくさん書いて生成する内容が変わるようなケースだと不向きと思います
  - [dockerが--formatオプションでtext/templateを受け付けます](https://docs.docker.com/engine/cli/formatting/)が、こういったデータが先行しており、ユーザー入力によって出力を自由に変更できるようにするとき、より価値を発揮すると思います。
- [github.com/dave/jennifer]はimportの連携ができない
  - `jennifer`内部的にimportを管理してqualifierを自動的に調節してくれますが、今回のケースのようにimport周りを外部かコントロールしたい、というのは見たところできないようです
  - 基本的に１ファイルまるごど`jennifer`で出力するのが想定なようですので、今回のように複数のやり口を組み合わせるときには不向き、というか想定していないのを感じます。

### Patcher

[実現したいもの#Patcher](#patcher)で述べたものを実装します。

今回生成するものの中でもっとも簡単です。

Patch typeは元の型のフィールドの型が`T`であるとき、`sliceund.Und[T]`で置き換え、`json:",omitempty"`をstruct tagに追加します。
フィールドの型がund typeであるときは、意図的なので何の変換もしないものとします。ただし、`option.Option`であるときは特別に`sliceund.Und[T]`に変換します。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_patcher.go#L182-L266

`sliceund`, `sliceund/elastic`には`json:",omitempty"`を追加することで`undefined`の時`json.Marshal`でフィールドがスキップされるようにします。`und`および`elastic`は`encoding/json/v2`もとい[github.com/go-json-experiment/json]でMarshal時にスキップできるように`json:",omitzero"`を追加します。

残りのメソッド群も実装していきます。

[Go1.18]以降追加されたgenericsによりtype paramが存在する型の場合receiverの型表記にもtype paramを表記する必要があります。

```go
type Foo[T any] struct {
    // ...
}

func (f Foo[T]) Foo() {}
// [T]がないとコンパイルしない
```

そのためtype paramは事前に出力しておきます。型情報からやってもastからやってもいいですがここではastから出力します。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_patcher.go#L307-L321

実装自体は気合と根性ですね。ここに関しては先に実装イメージを書いてそれを出力できるコードを書いただけ、という感じです。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_patcher.go#L337-L431

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_patcher.go#L433-L522

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_patcher.go#L524-L607

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_patcher.go#L609-L644

### Validator

Validatorは、`und:""` struct tagのついたund type fieldに対してstruct tagに応じたvalidationを行うか、フィールドが`implementor`もしくは`dependant`である場合、実装を呼び出します。

生成したいコードのイメージは[実現したいもの#Validator](#validator)を再び参照してください。

#### 追加要件

ただし追加の要件として、

- `implementor`|`dependant`はpointer typeでもよいこととします。
  - 大きなstructはpointerにしたいことは結構あるかと思います。
- フィールドが`map[string][][5]map[int]sliceund.Und[string]`のように無名の型で深くネストすることを許します。
  - つまり、edgeがmap, array, sliceを持つことを許します。
  - ここまで極端なことをすることは少ないかと思いますが`[][]T`や`map[string]map[string]T`をフィールドに指定すること自体はそう珍しくないと思います。
- und typeにラップされた`implementor`|`dependant`(`sliceund.Und[Implementor]`)の`UndValidate`も呼び出すようにします

つまり下記のようにpointer typeの場合、nilチェックを挟むことになります。

```diff go
type Dependant struct {
    // ...
+    FooP *All
    // ...
}

func (v Dependent) UndValidate() (err error) {
    // ...
+    {
+        if v.FooP != nil {
+            err = v.FooP.UndValidate()
+        }
+        if err != nil {
+            return validate.AppendValidationErrorDot(
+                err,
+                "FooP",
+            )
+        }
+    }
    // ...
}
```

以下のコードでフィールドが

- 無名の型で深くネストした場合
- und typeにラップされた`implementor`|`dependant`の場合に`UndValidate`も呼び出す

例を示します。

`Go`にはスコープごとに変数を再定義できる仕様があるため`for-range`がネストするたび同名の変数を再使用できています。この仕様がなければもう少しcode generatorの実装難易度が上がっていました。

```go
type Implementor struct {
    Opt option.Option[string] `und:"required"`
}

type DeeplyNested struct {
    A []map[string][5]und.Und[Implementor] `und:"required"`
}

//undgen:generated
func (v DeeplyNested) UndValidate() (err error) {
    {
        validator := undtag.UndOptExport{
            States: &undtag.StateValidator{
                Def: true,
            },
        }.Into()

        v := v.A

        for k, v := range v {
            for k, v := range v {
                for k, v := range v {
                    if !validator.ValidUnd(v) {
                        err = fmt.Errorf("%s: value is %s", validator.Describe(), validate.ReportState(v))
                    }
                    if err == nil {
                        err = und.UndValidate(v)
                    }

                    if err != nil {
                        err = validate.AppendValidationErrorIndex(
                            err,
                            fmt.Sprintf("%v", k),
                        )
                        break
                    }
                }

                if err != nil {
                    err = validate.AppendValidationErrorIndex(
                        err,
                        fmt.Sprintf("%v", k),
                    )
                    break
                }
            }

            if err != nil {
                err = validate.AppendValidationErrorIndex(
                    err,
                    fmt.Sprintf("%v", k),
                )
                break
            }
        }

        if err != nil {
            return validate.AppendValidationErrorDot(
                err,
                "A",
            )
        }
    }
    return
}
```

`validate.AppendValidationErrorIndex`でフィールドのセレクタをエラー情報にappendします。こうすることで`validation failed at .A[1][foo][3].Opt: must be defined: value is none`のようなエラーメッセージを表示できるようにして、どのフィールドがvalidation errorになったのかわかるようにします。

#### 実装

前述のとおり、型情報からstruct tagを取得できます

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_validator.go#L223

`undtag.ParseOption`として解析機能がexportしてあるのでこのstruct tagの解析自体はこれを呼び出すだけです。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_validator.go#L232-L238

前述のとおりですが、`undtag.ParseOption`の解析結果である`undtag.UndOpt`はinternal packageとしてvendorされた`option`を利用するため、これ自体を外部パッケージが初期化できません。
そのため`undtag.UndOptExport`を出力して`Into`メソッドを呼び出すことで`undtag.UndOpt`を得ます。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_validator.go#L343-L392

`map[string][][]A`のように深くネストした型のAを取り出すためのunwrapperを出力します

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_validator.go#L148-L173

少しわかりにくいですかね？
今回許すnamed typeへの経路は`map`, `slice`, `array`のみですが、これらすべては`for k, v := range value {}`で処理可能です。
そのため、ネストしたのと同数回`for-range loop`を行えば深くネストしたnamed typeを取り出せます。

そのため、

```go
func unwrapOne(innerExpr string) string {
    return fmt.Sprintf(
        `for k, v := range {
            %s
        }
`,
        innerExpr,
    )
}
```

という風にします。
`%s`に内側の`expr`(expression)を渡します。渡されるのは`for-loop` expressionか`validator`の呼び出しです。こうすればいくらでも入れ子状に`for-loop`を書き出すことができます。

さらに、

```diff go
func unwrapOne(innerExpr string) string {
    return fmt.Sprintf(
        `for k, v := range {
            %s
+           if err != nil {
+               err = validate.AppendValidationErrorIndex(
+                   err,
+                   fmt.Sprintf("%%v", k),
+               )
+               break
+           }
        }
`,
        innerExpr,
    )
}
```

とエラー時にbreakさせることでエラー発生時に順次innermostのループだけを抜けさせることで、すべてのループでそれぞれ`validate.AppendValidationErrorIndex`を呼び出せるようにします。

unwrapperをappendしていく順序と実際に呼び出すべき順序は逆であるので`slices.Backward`で逆順に適用していきます

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_validator.go#L319-L323

あとは`implementor`|`dependant`なら呼び出すとか、`implementor`|`dependant`がpointer typeならnilチェックをするとかそういった細かい気遣いを加えて完成です。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_validator.go#L103-L341

書いてたときはなかなかしんどかったですがその甲斐あってそこそこきれいにまとまりました。

### Plain

Plain変換はこの3つのテーマの中でもっとも複雑です。

生成したいコードのイメージは[実現したいもの#Plain](#plain)を再び参照してください。

#### 追加要件

ただし、Validator同様追加の要件として、

- `implementor`はpointer typeでもよいこととします。
- フィールドが`map[string][][5]map[int]sliceund.Und[string]`のように深くネストすることを許します。
- und typeにラップされた`implementor`|`dependant`(`sliceund.Und[Implementor]`)も変換されるようにします。
  - 型の変換(e.g. `sliceund.Und[Implementor]` -> `sliceund.Und[ImplementorPlain]`)
  - `UndPlain`/`UndRaw`の呼び出し

#### Plain typeへのast rewrite

##### field unwrapper

```go
type DeeplyNested struct {
    A []map[string][5]und.Und[Implementor] `und:"required"`
}
```

上記ではund typeである`und.Und`はslice, map, arrayにラップされています。ast rewriteは上記の`und.Und[Implementor]`を(`und:"required"`であるから)`ImplementorPlain`に変更したいわけですから、mapやsliceの部分は一切触れる必要がありません。
そのため、mapやarrayをたどって目的の型のexpressionを取り出します。

前述通り、どのように目的の型がラップされるかは`*TypeDependencyEdge`に記録済みですのでこれを利用します。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_plain_type.go#L35-L50

取り出した`dst.Expr`そのものに別のexprを代入したくなるケースを考慮して`*dst.Expr`を返すようにします。

後続の変換メソッド生成処理で*unwrap*したdst nodeと*wrap*されたままのdst nodeが必要なのでここでそれらを記録しておきます。

##### implementor|dependantのrewrite

`und:""` struct tagの付けられていない`implementor`|`dependant`、もしくはund typeにラップされた`implementor`|`dependant`は`UndPlain`の変換先の型名に取り換えます。

変換された[*types.Named]を`*dst.SelectorExpr`に変換して前節で*unwrap*された`dst.Node`に代入します。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_plain_type.go#L130-L175

`plainConverter`は、`implementor`と`dependant`を一緒くたにしてnamed typeからnamed typeへの変換をするための関数で実装は以下のようになります

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_plain.go#L148-L189

ややすっきりしない作りですが

- `implementor`は型情報をたどって変換先の`*types.Named`を取り出し
- `dependant`は前述の`makeRenamedType`で`UndRaw`だけを実装する`*types.Named`を作成して返します。

##### und typeのrewrite

上記のfield unwrapperによって取り出された`dst.Expr`を書き換えます。
und typeは現状、必ずtype paramを1つ持つので、必ず`*dst.IndexExpr`となります。

ここから先は面倒で複雑な変換を行います。

例えば、`option.Option[T]`, `und.Und[T]`のフィールドに`und:"required"` struct tagがついている場合、_Plain_ typeのフィールドの型は`T`となります。

この操作はast(dst)のrewriteで行います。

前述の例、`und.Und[T]`をstringでinstantiateした`und.Und[string]`でastの構造をしめします。

![](/images/go-code-generation-from-ast-and-type-info-ast-node-structure.drawio.png)

`und:"required"`がついている場合、`string`で置き換えるので、`expr = expr.(*ast.IndexExpr).Index`という代入操作をします。

![](/images/go-code-generation-from-ast-and-type-info-ast-node-structure-swap.drawio.png)

結果として`string`のみが残ります。

![](/images/go-code-generation-from-ast-and-type-info-ast-node-structure-swap-result.drawio.png)

例えばほかにも`und.Und`部分を`option.Option`に書き換えるのならば、図の`SelectorExpr`部分を任意に置き換えればできますし、`und.Und[T]`を`und.Und[[]T]`に置換するのも`Index`部分を`*ast.ArrayType`に置き換え、置き換え前の`Index`のexprを`*ast.ArrayType`の`Elt`フィールドに代入すればできます。

こういう感じでパターンを網羅していきます。

まず最初に`und.Und[T]`の`T`が`implementor`|`dependant`である場合、`UndPlain`で変換された先に取り換えます。

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

これに従い、

- `option.Option[T]`は
  - `def && (null || und)`なら変更なし
  - `def`: `option.Option[T]` -> `T`
  - `null||und` -> `Empty`
- `und.Und[T]`は
  - `def && null && und` -> 変更なし
  - `def && (null || und)`: `und.Und[T]` -> `option.Option[T]`
  - `null && und` -> `option.Option[Empty]`
  - `def`: `und.Und[T]` -> `T`
  - `null || und` -> `Empty`

という風に変換していきます。
`null || und`のような特殊なケースのために[Empty](https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/conversion#Empty)という`json.Marshal`時に`MarshalJSON`実装でnullを返す`[]struct{}`ベースの型を定義し、そちらを使うようにします。

`elastic.Elastic[T]`の変換はもっとパターンが多くなってややこしいです。

`def&&null&&und`かつ`len`オプションがない、もしくは`==`以外の指定で、さらに`values:nonnull`が指定されていないとき型の変換は必要ないのでreturnします。

そうでない場合、`elastic.Elastic[T]` -> `und.Und[[]option.Option[T]]`という変換をかけます。ここから先のパターンは少なくとも必ずこの型には変換されます。

- `len==1`の場合、`[]option.Option[T]`部分はsliceである必要はないので`und.Und[[]T]` -> `und.Und[T]`と変換します。
- `len==n`の場合、`und.Und[[]T]` -> `und.Und[[n]T]`に変換します。

さらに、`values:nonnull`が指定されている場合、`und.Und[[]option.Option[T]]` -> `und.Und[[]T]`に変換します。
`len==1`だった場合はこの時点で`und.Und[option.Option[T]]`であるので、`und.Und[T]`に変換します。

最後に`def,null,und`の状態に応じた変換をかけます。

- `def && null && und`: 変更なし
- `def && (null || und)`: `und.Und[T]` -> `option.Option[T]`
- `null && und`: -> `option.Option[Empty]`
- `de`: `und.Und[T]` -> `T`
- `null || und`: -> `Empty`

上記すべてを盛り込むと下記のように実装されます

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_plain_type.go#L185-L325

#### UndPlain/UndRaw method

`UndPlain`メソッドで*Raw* -> *Plain*の変換、`UndRaw`メソッドで*Plain* -> *Raw*の相互変換を実装します。

型の変換時と同じく、und typeや`implementor`がslice, map, arrayにラップされていることは許されているので、mapやarrayをたどって目的の値までたどり、値と`und:""` struct tagに応じた変換処理をかけます。

しかるに、field unwrappingと変換で二つの工程の分けることができます。

##### field unwrapper

mapやsliceを*unwrap*して関心のある型にたどり着くための方法について述べます。
型変換時と同様に`*TypeDependencyEdge`に型の経路を記録済みですのでこれを利用します。

以下のような型あるとき、

```go
type DeeplyNested struct {
    A []map[string][5]und.Und[Implementor] `und:"required"`
}
```

以下のように、値を順次`for-range`で下っていき、変換となるexpressionを呼び出します。

```go
func(v DeeplyNested) UndRaw() DeeplyNestedPlain {
    return DeeplyNestedPlain{
        A: func(v []map[string][5]und.Und[Implementor]) []map[string][5]ImplementorPlain {
            for k, v := range v {
                // ...
                var (
                    k int
                    v und.Und[Implementor]
                )
                out[k] = conversion(v)
                // ...
            }
        }(v.A)
    }
}
```

これをcode generatorで生成するには、code genreatorに有利な単純な表現の繰り返しでこれを実現しなければなりません。
単純な発想では以下のような繰り返しになるんですが、

```go
func(v []map[string][]V) []map[string][]VPlain {
    out := make([]map[string][]VPlain)

    inner := out
    for k /* int */, v /* map[string][]V */ :=range v /* []map[string][]V */ {
        outer := inner
        inner := make(map[string][]VPlain, len(v))
        for k /* string */, v /* []V */ := range v {
            outer := inner
            inner := make([]VPlain, len(v))
            for k, v := range v{
                // この繰り返し？
            }
            outer[k] = inner
        }
        outer[k] = inner
    }

    return out
}
```

これではarrayが経路に含まれるとarrayのコピーによってunused writeが生じます。

```go
// ...
            outer := inner
            inner := [5]T
            for k, v := range v{
                //
            }
            outer[k] = inner
            // unused write to array index t16 + 1:int unusedwrite(default)
// ...
```

そこで、outerの定義はinnerへのアドレスをとることとします。

```go
// ...
            outer := &inner
            inner := [5]T
            for k, v := range v{
                //
            }
            (*outer)[k] = inner
// ...
```

表現を初期化部、中間経路、終端と三つに分けてそれぞれを`func(expr string) string`とします。

```go
// 初期化部
 func(expr string) string {
    return fmt.Sprintf(
        `func (v %s) %s {
            out := %s

            inner := out
            %s

            return out
        }(%s)`,
        input /* map[string]V */,
        output/* map[string]VPlain */,
        initializer(toExpr, s[0].Kind) /* make(map[string]VPlain, len(v)) */,
        expr /* 中間経路(終端) */,
        fieldExpr /* v.Aなど */,
    )
}

// 中間経路
func(s string) string {
    return fmt.Sprintf(
        `for k, v := range v {
            outer := &inner
            inner := %s
            %s
            (*outer)[k] = inner
        }`,
        initializerExpr/* make([]T, len(v))など */, s/* 中間経路(終端) */,
    )
}

// 終端(このsがV -> VPlainの変換expression)
func(s string) string {
    return fmt.Sprintf(
        `for k, v := range v {
            inner[k] = %s
        }`,
        s,
    )
}
```

`wrappers []func(string) string`を定義し、これらを順次詰め込みます`[初期化, 経路, 経路, ..., 終端]`という順列でappendすることとし、

```go
expr := wrappee("v")
for _, wrapper := range slices.Backward(wrappers) {
    expr = wrapper(expr)
}
```

という風に、`slices.Backward`で逆順で適用すればよいということになります。

ad hocな即時間数を毎度書くため、フィールドの変換前(_Raw_)、変換後(_Plain_)の型をそれぞれ明示的に示す必要があり、さらに`make(T, len(v))`を毎回呼ぶために経路上の中間となる型の表現もすべて書き出す必要があります。
前述のとおり経路の情報はすでに保存済みであるので、それを利用した以下の関数を定義します。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_plain_method.go#L26-L34

これにより`map[string][]V` -> `[]V` -> `V`という感じで順次unwrapすることができます。`ast.Expr`は`printer.Fprint`でnode単位でprint可能ですので、printした結果をテキストとして前述の関数群に渡します。

全部を組み合わせて以下のように`unwrapFieldAlongPath`を定義します。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_plain_method.go#L36-L110

##### conversion

型の変換時と同様に型と`und:""` strcut tagの内容に基づいて変換する関数を定義します。

コードの生成量を減らすために[github.com/ngicks/und]側で変換のためのランタイムを提供します。

https://github.com/ngicks/und/blob/67d88238795b9e837e9bfce9aeaf839dc4084899/conversion/conversion.go
https://github.com/ngicks/und/blob/67d88238795b9e837e9bfce9aeaf839dc4084899/conversion/back_conversion.go

これらのコードは`und`モジュール自体が使うことは一切ありません。そういったものをそこに定義するのはそれはそれで邪道に思いますが、生成されたコードが依存するランタイムを減らせてばらばらにバージョン管理しなくていいのが明確なメリットとなります。

変換自体は型の変換で説明したことをコード的に行うのみです。

- `und.Und[T] -> T`: `(und.Und[T]{}).Value()`
- `und.Und[T] -> option.Option[T]`: `(und.Und[T]{}).Unwrap().Value()`

逆変換は

- `T -> und.Und[T]`: `und.Defined(t)`
- `option.Option[T] -> und.Und[T]`: `conversion.OptionUnd(false, opt)`

こういう感じです。

- `len==1`の時`[]T` -> `T`の変換が行われますが、この時変換メソッドは生成の都合で`[]T` -> `[1]T` -> `T`というステップで行う決断を下しました。
- `[]T` -> `[n]T`への変換はad hocな即時間数を毎回定義します
  - 全く同じ関数を何度も定義することになりますが、内容が同じであればコンパイラがいい感じに1つの関数に減らしてくれるでしょうから気にしません。
  - 逆にいうとそういった最適化に協力するために即時関数を生成する際にはなるだけ変数をキャプチャしないようにします。
- `UndRaw`/`UndPlain`を呼び出すには上記の`und`モジュールの`conversion`パッケージの協力を得ずに、即時間数を生成します
  - genericsだとtype constraintの都合上receiverがポインタであっても、ノンポインタであってもよいとすることが難しいためです。
  - `implementor`は`UndRaw`/`UndPlain`をpointer receiverの上に実装してもよいですし、`implementor type A`があるときはstrcutフィールド上で`fieldName *A`であってもよくなります。

以下のように定義されます。

_Raw_ -> _Plain_
https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_plain_to_plain.go

_Plain_ -> _Raw_
https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_plain_to_raw.go

Raw ↔ Plain変換部の呼び出し。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/undgen/gen_plain_method.go#L112-L481

`UndRaw`と`UndPlain`の生成は意外にも互いにほとんど同じ処理で上記の*Raw* -> *Plain*と*Plain* -> *Raw*の各部を取り換える以外はほとんど共通です。

## github.com/spf13/cobraを利用した実行ファイル化

[github.com/spf13/cobra](https://github.com/spf13/cobra)を利用して実装した機能をサブコマンドで呼び出せる実行ファイルにまとめます。

```
go run github.com/ngicks/go-codegen/codegen@latest undgen plain -v --ignore-generated --dir ./path/to/target --pkg ./...
```

という感じで呼び出せます(と書いといてなんですがこのモジュール外から呼び出したことないです！)

以下の4つのファイルでまとめます。`undgen`というサブコマンドの更にサブコマンドで`patch`/`validator`/`plain`を呼び出せます。
`cobra`を使うと複数のコマンドを簡単にまとめられて助かります。

https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/cmd/undgen.go
https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/cmd/undgen_patch.go
https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/cmd/undgen_plain.go
https://github.com/ngicks/go-codegen/blob/7dbb755aecf626c70586719602b078f2ca3df708/codegen/cmd/undgen_validator.go

## 生成結果

生成サンプル用の型と結果は以下に格納されます。

https://github.com/ngicks/go-codegen/tree/2a35a98a9c52910efb646ac714b307bd9a43710a/codegen/undgen/internal/testtargets

## おわりに

筆者がここ数年ずっとやりたいと思いながらできていなかった、astと型情報をメタデータとするcode generatorの実装をようやくできるようになりました。

これを作り出したきかっけは業務でpartial jsonを使ったpatchを行うと都合のいい場面が出たからなんですが、例によって例のごとく、その時はその場限りな方法で解決してしまったため、今回作ったものを使う機会は逃してしまっています。

さて今後についてですが

`undgen`については現状の実装から大きく変わることはないと思いますが、いくつかの変更を予測しています。

- リファクタ: もう少しまとめられそうなコードが重複しているので整理しなおします。
- `und:"und"`がついたときのplain typeの対応するフィールドを`*T`にする
  - `T | null`は`option.Option[T]`で表現できますが、`T | undefined`は`json:",omitmepty"`のついた`*T`である必要があるためです。
  - そうしなければ、`json.Marshal`などで出力する際には`Raw`に一度変換しなおさなければフィールドが`null`で出力されてしますため、少し不便ですね。
  - `Plain`だけを使っても運用が通用したほうが便利ではあると思うためそうなるように検証を重ねていこうかなと思っています。
- もう一つは、さらなるオプションの追加です。
  - type-suffixオプション: 現状、生成される型は元の型名+`Patch`|`Plain`の名前がつきます。これが固定だと少し具合が悪いかなと思います。
  - denylistオプション: また、今は`validator`,`plain`は`//undgen:ignore`というコメントがついていない型はすべて生成対象となってしまいます。これはこのcode genreatorが複数のパッケージを同時に処理することを前提とするため、cli引数からallowlist/denylistを受けとるのが煩雑であるためこういった決断を下していました。ここをもう少し見直してdenylistを受けとれるようにしたほうが良いかなあと思っています。

さらに、今回作ったものを通じて型情報の操作に習熟したのでもっと違うものも作れるようになりました。今後はそちらも作って行くことになるかと思います。

- wrapper: struct fieldにinterfaceを含む型に対して、そのinterfaceを実装するようにメソッドを生成します。
  - すべての挙動はそのフィールドのinterface実装に移譲するが、引数やinterfaceかからの返り値を加工したいときに使うcode generatorです。
  - 例1: [afero.Fs](https://pkg.go.dev/github.com/spf13/afero#Fs)をラップして、引数が[fs.ValidPath](https://pkg.go.dev/io/fs@go1.23.3#ValidPath)を満たすように変換する
  - 例2: `afero.Fs`をラップして、引数と返り値をすべて記録し、テストに使う。
  - メソッドが多いinterfaceのラッパーを定義するのがしんどかったので、カスタマイズ性を犠牲にせずに楽に生成できる仕組みを整えておきたいと筆者は常々思っています。
- cloner: 型に対して`Clone`メソッドを生成してdeep-cloneを可能とします
  - いくつかのOSS実装を試したことがあるんですが、例えば`*map[K]V`というフィールドを含むstructに対して生成するとpointerであることが想定されていなくて生成されたコードがコンパイルできなかったりします。
  - 今回実装したcode genreatorの処理のほとんどがdeep clonerの生成に用いることができるためじゃあ作ればよくないかと思っています。
    - 型がcopy-by-assignなのか検知する機能以外もうほとんど実装終わってる気がするんですよ。
      - [noCopy](https://github.com/golang/go/issues/8005#issuecomment-190753527)型の検知とかもですね
    - field unwrapperのところはもうほとんどdeep cloneです。
  - `undgen`でさぼったtype paramの追跡が必須なのでそこが少々課題ですが
- option: よくあるfunctional optionパターンを実装するのですが、unexported fieldそれぞれがoption interfaceを満たす型として生成します。
  - 例えば`type A struct{foo string}`があるとき、`type Option interface { apply(a *A) Option }`が定義され、
  - `type fooOption string`, `func (o fooOption) apply(a *A) Option { prev := a.foo; a.foo = string(o); return fooOption(prev) }`という風に実装します
  - よくある`type Option func(a *A)`と違って比較できます。
    - 仕様上の制限で関数はnilとしか比較できない。
    - 型として定義することで、fieldの型がcomparableならcomparableのままにできます。
  - すべてのフィールドに対して個別に型を定義するのは手間すぎてやる気が起きなかったんですが、code generatorを整備しておけば現実的に可能ですね。
- なんとかpostfix系: code generatorの結果を受けてさらにfixするとか置き換えるとかする
  - oapi-codegen-postfix: [#970](https://github.com/oapi-codegen/oapi-codegen/issues/970)でも指摘されていますがoneOfを指定するとmarshalがおかしくなります。これをfixする。
    - 理由は単純で`type B`に`MarshalJSON`実装をしているとき、`type A B`で定義した`A`を`json.Encoder`に渡しているから起きています。`type A B`はmethod setを継承していないため`B`の`MarshalJSON`が呼び出されないため必ず`{}`が出力されます。
    - 今(`v2.4.1`)確認しても修正されていなかったのでまだやる価値はある。
- ident-mover: ファイル単位、exportされたident単位でパッケージに入っていたものを別のパッケージに移動させる
  - リファクタ(？)の中でも頻繁に困るのは元は同じパッケージで定義していたものを別のパッケージに切り出す時の書き換えです
  - 現在進行形でリファクタで苦労しています。
  - `GoLand`にはこういったものが最初から同梱されてるんですかね？

型情報とdst-rewriteを活用すれば別ファイルに書き出さないタイプのcode generator、つまりリファクタツールでもなんでも作れちゃいますね

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
[*types.Named]: https://pkg.go.dev/go/types@go1.23.3#Named
[types.Object]: https://pkg.go.dev/go/types@go1.23.3#Object
[types.Type]: https://pkg.go.dev/go/types@go1.23.3#Type
[*types.Info]: https://pkg.go.dev/go/types@go1.23.3#Info
[*ast.File]: https://pkg.go.dev/go/ast@go1.23.3#File
[*ast.TypeSpec]: https://pkg.go.dev/go/ast@go1.23.3#TypeSpec
[golang.org/x/tools/go/packages]: https://pkg.go.dev/golang.org/x/tools@v0.26.0/go/packages
[github.com/dave/dst]: https://github.com/dave/dst
[github.com/oapi-codegen/oapi-codegen]: https://github.com/oapi-codegen/oapi-codegen
[github.com/ngicks/und]: https://github.com/ngicks/und
[sliceund.Und]: https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/sliceund#Und
[sliceelastic.Elastic]: https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/sliceund/elastic#Elastic
[Elasticsearch]: https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro.html
[github.com/go-json-experiment/json]: https://github.com/go-json-experiment/json
[*bufio.Writer]: https://pkg.go.dev/bufio@go1.23.3#Writer
[fmt.Fprintf]: https://pkg.go.dev/fmt@go1.23.3#Fprintf
[text/template]: https://pkg.go.dev/text/template@go1.23.3
[github.com/dave/jennifer]: https://github.com/dave/jennifer
