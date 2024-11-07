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

## お題

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
  - `und:""` struct tagの内容でsomeじゃないいけないとか、Elasticの場合は`[]T`がn要素以上ないといけないとかを決められるようにしてあるため、この譲歩を用いてvalidateを行う
- Plain
  - `und:""` struct tagの内容からsomeでないといけないなら`option.Option[T]`を`T`にアンラップしたような*Plain*な型を作成し、元となった型との相互変換を実現する。

## 生成されるコードのイメージ

まずどういったコードを生成したら目標が実現できるかを思い描き、どういったらコードジェネレーターを実装するかを思い描きます

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
type All struct {
	Foo string
	Bar *string
	Baz *struct{}
	Qux []string

	UntouchedOpt      option.Option[int] `json:",omitzero"`
	UntouchedUnd      und.Und[int]       `json:",omitzero"`
	UntouchedSliceUnd sliceund.Und[int]  `json:",omitzero"`

	OptRequired       option.Option[string] `json:"opt_required,omitzero" und:"required"`
	OptNullish        option.Option[string] `json:",omitzero" und:"nullish"`
	OptDef            option.Option[string] `json:",omitzero" und:"def"`
	OptNull           option.Option[string] `json:",omitzero" und:"null"`
	OptUnd            option.Option[string] `json:",omitzero" und:"und"`
	OptDefOrUnd       option.Option[string] `json:",omitzero" und:"def,und"`
	OptDefOrNull      option.Option[string] `json:",omitzero" und:"def,null"`
	OptNullOrUnd      option.Option[string] `json:",omitzero" und:"null,und"`
	OptDefOrNullOrUnd option.Option[string] `json:",omitzero" und:"def,null,und"`

	UndRequired       und.Und[string] `json:",omitzero" und:"required"`
	UndNullish        und.Und[string] `json:",omitzero" und:"nullish"`
	UndDef            und.Und[string] `json:",omitzero" und:"def"`
	UndNull           und.Und[string] `json:",omitzero" und:"null"`
	UndUnd            und.Und[string] `json:",omitzero" und:"und"`
	UndDefOrUnd       und.Und[string] `json:",omitzero" und:"def,und"`
	UndDefOrNull      und.Und[string] `json:",omitzero" und:"def,null"`
	UndNullOrUnd      und.Und[string] `json:",omitzero" und:"null,und"`
	UndDefOrNullOrUnd und.Und[string] `json:",omitzero" und:"def,null,und"`

	ElaRequired       elastic.Elastic[string] `json:",omitzero" und:"required"`
	ElaNullish        elastic.Elastic[string] `json:",omitzero" und:"nullish"`
	ElaDef            elastic.Elastic[string] `json:",omitzero" und:"def"`
	ElaNull           elastic.Elastic[string] `json:",omitzero" und:"null"`
	ElaUnd            elastic.Elastic[string] `json:",omitzero" und:"und"`
	ElaDefOrUnd       elastic.Elastic[string] `json:",omitzero" und:"def,und"`
	ElaDefOrNull      elastic.Elastic[string] `json:",omitzero" und:"def,null"`
	ElaNullOrUnd      elastic.Elastic[string] `json:",omitzero" und:"null,und"`
	ElaDefOrNullOrUnd elastic.Elastic[string] `json:",omitzero" und:"def,null,und"`

	ElaEqEq elastic.Elastic[string] `json:",omitzero" und:"len==1"`
	ElaGr   elastic.Elastic[string] `json:",omitzero" und:"len>1"`
	ElaGrEq elastic.Elastic[string] `json:",omitzero" und:"len>=1"`
	ElaLe   elastic.Elastic[string] `json:",omitzero" und:"len<1"`
	ElaLeEq elastic.Elastic[string] `json:",omitzero" und:"len<=1"`

	ElaEqEquRequired elastic.Elastic[string] `json:",omitzero" und:"required,len==2"`
	ElaEqEquNullish  elastic.Elastic[string] `json:",omitzero" und:"nullish,len==2"`
	ElaEqEquDef      elastic.Elastic[string] `json:",omitzero" und:"def,len==2"`
	ElaEqEquNull     elastic.Elastic[string] `json:",omitzero" und:"null,len==2"`
	ElaEqEquUnd      elastic.Elastic[string] `json:",omitzero" und:"und,len==2"`

	ElaEqEqNonNullSlice      elastic.Elastic[string] `json:",omitzero" und:"values:nonnull"`
	ElaEqEqNonNullNullSlice  elastic.Elastic[string] `json:",omitzero" und:"null,values:nonnull"`
	ElaEqEqNonNullSingle     elastic.Elastic[string] `json:",omitzero" und:"values:nonnull,len==1"`
	ElaEqEqNonNullNullSingle elastic.Elastic[string] `json:",omitzero" und:"null,values:nonnull,len==1"`
	ElaEqEqNonNull           elastic.Elastic[string] `json:",omitzero" und:"values:nonnull,len==3"`
	ElaEqEqNonNullNull       elastic.Elastic[string] `json:",omitzero" und:"null,values:nonnull,len==3"`
}
```

以下が出力されるだろうということです

```go
type AllPlain struct {
	Foo string
	Bar *string
	Baz *struct{}
	Qux []string

	UntouchedOpt      option.Option[int] `json:",omitzero"`
	UntouchedUnd      und.Und[int]       `json:",omitzero"`
	UntouchedSliceUnd sliceund.Und[int]  `json:",omitzero"`

	OptRequired       string                `json:"opt_required,omitzero" und:"required"`
	OptNullish        conversion.Empty      `json:",omitzero" und:"nullish"`
	OptDef            string                `json:",omitzero" und:"def"`
	OptNull           conversion.Empty      `json:",omitzero" und:"null"`
	OptUnd            conversion.Empty      `json:",omitzero" und:"und"`
	OptDefOrUnd       option.Option[string] `json:",omitzero" und:"def,und"`
	OptDefOrNull      option.Option[string] `json:",omitzero" und:"def,null"`
	OptNullOrUnd      conversion.Empty      `json:",omitzero" und:"null,und"`
	OptDefOrNullOrUnd option.Option[string] `json:",omitzero" und:"def,null,und"`

	UndRequired       string                          `json:",omitzero" und:"required"`
	UndNullish        option.Option[conversion.Empty] `json:",omitzero" und:"nullish"`
	UndDef            string                          `json:",omitzero" und:"def"`
	UndNull           conversion.Empty                `json:",omitzero" und:"null"`
	UndUnd            conversion.Empty                `json:",omitzero" und:"und"`
	UndDefOrUnd       option.Option[string]           `json:",omitzero" und:"def,und"`
	UndDefOrNull      option.Option[string]           `json:",omitzero" und:"def,null"`
	UndNullOrUnd      option.Option[conversion.Empty] `json:",omitzero" und:"null,und"`
	UndDefOrNullOrUnd und.Und[string]                 `json:",omitzero" und:"def,null,und"`

	ElaRequired       []option.Option[string]                `json:",omitzero" und:"required"`
	ElaNullish        option.Option[conversion.Empty]        `json:",omitzero" und:"nullish"`
	ElaDef            []option.Option[string]                `json:",omitzero" und:"def"`
	ElaNull           conversion.Empty                       `json:",omitzero" und:"null"`
	ElaUnd            conversion.Empty                       `json:",omitzero" und:"und"`
	ElaDefOrUnd       option.Option[[]option.Option[string]] `json:",omitzero" und:"def,und"`
	ElaDefOrNull      option.Option[[]option.Option[string]] `json:",omitzero" und:"def,null"`
	ElaNullOrUnd      option.Option[conversion.Empty]        `json:",omitzero" und:"null,und"`
	ElaDefOrNullOrUnd elastic.Elastic[string]                `json:",omitzero" und:"def,null,und"`

	ElaEqEq option.Option[string]   `json:",omitzero" und:"len==1"`
	ElaGr   []option.Option[string] `json:",omitzero" und:"len>1"`
	ElaGrEq []option.Option[string] `json:",omitzero" und:"len>=1"`
	ElaLe   []option.Option[string] `json:",omitzero" und:"len<1"`
	ElaLeEq []option.Option[string] `json:",omitzero" und:"len<=1"`

	ElaEqEquRequired [2]option.Option[string]                `json:",omitzero" und:"required,len==2"`
	ElaEqEquNullish  und.Und[[2]option.Option[string]]       `json:",omitzero" und:"nullish,len==2"`
	ElaEqEquDef      [2]option.Option[string]                `json:",omitzero" und:"def,len==2"`
	ElaEqEquNull     option.Option[[2]option.Option[string]] `json:",omitzero" und:"null,len==2"`
	ElaEqEquUnd      option.Option[[2]option.Option[string]] `json:",omitzero" und:"und,len==2"`

	ElaEqEqNonNullSlice      und.Und[[]string]        `json:",omitzero" und:"values:nonnull"`
	ElaEqEqNonNullNullSlice  conversion.Empty         `json:",omitzero" und:"null,values:nonnull"`
	ElaEqEqNonNullSingle     string                   `json:",omitzero" und:"values:nonnull,len==1"`
	ElaEqEqNonNullNullSingle option.Option[string]    `json:",omitzero" und:"null,values:nonnull,len==1"`
	ElaEqEqNonNull           [3]string                `json:",omitzero" und:"values:nonnull,len==3"`
	ElaEqEqNonNullNull       option.Option[[3]string] `json:",omitzero" und:"null,values:nonnull,len==3"`
}

func (v All) UndPlain() AllPlain {
	return AllPlain{
		Foo:               v.Foo,
		Bar:               v.Bar,
		Baz:               v.Baz,
		Qux:               v.Qux,
		UntouchedOpt:      v.UntouchedOpt,
		UntouchedUnd:      v.UntouchedUnd,
		UntouchedSliceUnd: v.UntouchedSliceUnd,
		OptRequired:       v.OptRequired.Value(),
		OptNullish:        nil,
		OptDef:            v.OptDef.Value(),
		OptNull:           nil,
		OptUnd:            nil,
		OptDefOrUnd:       v.OptDefOrUnd,
		OptDefOrNull:      v.OptDefOrNull,
		OptNullOrUnd:      nil,
		OptDefOrNullOrUnd: v.OptDefOrNullOrUnd,
		UndRequired:       v.UndRequired.Value(),
		UndNullish:        conversion.UndNullish(v.UndNullish),
		UndDef:            v.UndDef.Value(),
		UndNull:           nil,
		UndUnd:            nil,
		UndDefOrUnd:       v.UndDefOrUnd.Unwrap().Value(),
		UndDefOrNull:      v.UndDefOrNull.Unwrap().Value(),
		UndNullOrUnd:      conversion.UndNullish(v.UndNullOrUnd),
		UndDefOrNullOrUnd: v.UndDefOrNullOrUnd,
		ElaRequired:       v.ElaRequired.Unwrap().Value(),
		ElaNullish:        conversion.UndNullish(v.ElaNullish),
		ElaDef:            v.ElaDef.Unwrap().Value(),
		ElaNull:           nil,
		ElaUnd:            nil,
		ElaDefOrUnd:       conversion.UnwrapElastic(v.ElaDefOrUnd).Unwrap().Value(),
		ElaDefOrNull:      conversion.UnwrapElastic(v.ElaDefOrNull).Unwrap().Value(),
		ElaNullOrUnd:      conversion.UndNullish(v.ElaNullOrUnd),
		ElaDefOrNullOrUnd: v.ElaDefOrNullOrUnd,
		ElaEqEq: conversion.UnwrapLen1(und.Map(
			conversion.UnwrapElastic(v.ElaEqEq),
			func(o []option.Option[string]) (out [1]option.Option[string]) {
				copy(out[:], o)
				return out
			},
		)).Value(),
		ElaGr:   conversion.LenNAtLeast(2, conversion.UnwrapElastic(v.ElaGr)).Value(),
		ElaGrEq: conversion.LenNAtLeast(1, conversion.UnwrapElastic(v.ElaGrEq)).Value(),
		ElaLe:   conversion.LenNAtMost(0, conversion.UnwrapElastic(v.ElaLe)).Value(),
		ElaLeEq: conversion.LenNAtMost(1, conversion.UnwrapElastic(v.ElaLeEq)).Value(),
		ElaEqEquRequired: und.Map(
			conversion.UnwrapElastic(v.ElaEqEquRequired),
			func(o []option.Option[string]) (out [2]option.Option[string]) {
				copy(out[:], o)
				return out
			},
		).Value(),
		ElaEqEquNullish: und.Map(
			conversion.UnwrapElastic(v.ElaEqEquNullish),
			func(o []option.Option[string]) (out [2]option.Option[string]) {
				copy(out[:], o)
				return out
			},
		),
		ElaEqEquDef: und.Map(
			conversion.UnwrapElastic(v.ElaEqEquDef),
			func(o []option.Option[string]) (out [2]option.Option[string]) {
				copy(out[:], o)
				return out
			},
		).Value(),
		ElaEqEquNull: und.Map(
			conversion.UnwrapElastic(v.ElaEqEquNull),
			func(o []option.Option[string]) (out [2]option.Option[string]) {
				copy(out[:], o)
				return out
			},
		).Unwrap().Value(),
		ElaEqEquUnd: und.Map(
			conversion.UnwrapElastic(v.ElaEqEquUnd),
			func(o []option.Option[string]) (out [2]option.Option[string]) {
				copy(out[:], o)
				return out
			},
		).Unwrap().Value(),
		ElaEqEqNonNullSlice:     conversion.NonNull(conversion.UnwrapElastic(v.ElaEqEqNonNullSlice)),
		ElaEqEqNonNullNullSlice: nil,
		ElaEqEqNonNullSingle: conversion.UnwrapLen1(und.Map(
			und.Map(
				conversion.UnwrapElastic(v.ElaEqEqNonNullSingle),
				func(o []option.Option[string]) (out [1]option.Option[string]) {
					copy(out[:], o)
					return out
				},
			),
			func(s [1]option.Option[string]) (r [1]string) {
				for i := 0; i < 1; i++ {
					r[i] = s[i].Value()
				}
				return
			},
		)).Value(),
		ElaEqEqNonNullNullSingle: conversion.UnwrapLen1(und.Map(
			und.Map(
				conversion.UnwrapElastic(v.ElaEqEqNonNullNullSingle),
				func(o []option.Option[string]) (out [1]option.Option[string]) {
					copy(out[:], o)
					return out
				},
			),
			func(s [1]option.Option[string]) (r [1]string) {
				for i := 0; i < 1; i++ {
					r[i] = s[i].Value()
				}
				return
			},
		)).Unwrap().Value(),
		ElaEqEqNonNull: und.Map(
			und.Map(
				conversion.UnwrapElastic(v.ElaEqEqNonNull),
				func(o []option.Option[string]) (out [3]option.Option[string]) {
					copy(out[:], o)
					return out
				},
			),
			func(s [3]option.Option[string]) (r [3]string) {
				for i := 0; i < 3; i++ {
					r[i] = s[i].Value()
				}
				return
			},
		).Value(),
		ElaEqEqNonNullNull: und.Map(
			und.Map(
				conversion.UnwrapElastic(v.ElaEqEqNonNullNull),
				func(o []option.Option[string]) (out [3]option.Option[string]) {
					copy(out[:], o)
					return out
				},
			),
			func(s [3]option.Option[string]) (r [3]string) {
				for i := 0; i < 3; i++ {
					r[i] = s[i].Value()
				}
				return
			},
		).Unwrap().Value(),
	}
}

func (v AllPlain) UndRaw() All {
	return All{
		Foo:               v.Foo,
		Bar:               v.Bar,
		Baz:               v.Baz,
		Qux:               v.Qux,
		UntouchedOpt:      v.UntouchedOpt,
		UntouchedUnd:      v.UntouchedUnd,
		UntouchedSliceUnd: v.UntouchedSliceUnd,
		OptRequired:       option.Some(v.OptRequired),
		OptNullish:        option.None[string](),
		OptDef:            option.Some(v.OptDef),
		OptNull:           option.None[string](),
		OptUnd:            option.None[string](),
		OptDefOrUnd:       v.OptDefOrUnd,
		OptDefOrNull:      v.OptDefOrNull,
		OptNullOrUnd:      option.None[string](),
		OptDefOrNullOrUnd: v.OptDefOrNullOrUnd,
		UndRequired:       und.Defined(v.UndRequired),
		UndNullish:        conversion.NullishUnd[string](v.UndNullish),
		UndDef:            und.Defined(v.UndDef),
		UndNull:           und.Null[string](),
		UndUnd:            und.Undefined[string](),
		UndDefOrUnd:       conversion.OptionUnd(false, v.UndDefOrUnd),
		UndDefOrNull:      conversion.OptionUnd(true, v.UndDefOrNull),
		UndNullOrUnd:      conversion.NullishUnd[string](v.UndNullOrUnd),
		UndDefOrNullOrUnd: v.UndDefOrNullOrUnd,
		ElaRequired:       elastic.FromOptions(v.ElaRequired...),
		ElaNullish:        conversion.NullishElastic[string](v.ElaNullish),
		ElaDef:            elastic.FromOptions(v.ElaDef...),
		ElaNull:           elastic.Null[string](),
		ElaUnd:            elastic.Undefined[string](),
		ElaDefOrUnd:       conversion.OptionOptionElastic(false, v.ElaDefOrUnd),
		ElaDefOrNull:      conversion.OptionOptionElastic(true, v.ElaDefOrNull),
		ElaNullOrUnd:      conversion.NullishElastic[string](v.ElaNullOrUnd),
		ElaDefOrNullOrUnd: v.ElaDefOrNullOrUnd,
		ElaEqEq: elastic.FromUnd(und.Map(
			conversion.WrapLen1(und.Defined(v.ElaEqEq)),
			func(s [1]option.Option[string]) []option.Option[string] {
				return s[:]
			},
		)),
		ElaGr:   elastic.FromUnd(und.Defined(v.ElaGr)),
		ElaGrEq: elastic.FromUnd(und.Defined(v.ElaGrEq)),
		ElaLe:   elastic.FromUnd(und.Defined(v.ElaLe)),
		ElaLeEq: elastic.FromUnd(und.Defined(v.ElaLeEq)),
		ElaEqEquRequired: elastic.FromUnd(und.Map(
			und.Defined(v.ElaEqEquRequired),
			func(s [2]option.Option[string]) []option.Option[string] {
				return s[:]
			},
		)),
		ElaEqEquNullish: elastic.FromUnd(und.Map(
			v.ElaEqEquNullish,
			func(s [2]option.Option[string]) []option.Option[string] {
				return s[:]
			},
		)),
		ElaEqEquDef: elastic.FromUnd(und.Map(
			und.Defined(v.ElaEqEquDef),
			func(s [2]option.Option[string]) []option.Option[string] {
				return s[:]
			},
		)),
		ElaEqEquNull: elastic.FromUnd(und.Map(
			conversion.OptionUnd(true, v.ElaEqEquNull),
			func(s [2]option.Option[string]) []option.Option[string] {
				return s[:]
			},
		)),
		ElaEqEquUnd: elastic.FromUnd(und.Map(
			conversion.OptionUnd(false, v.ElaEqEquUnd),
			func(s [2]option.Option[string]) []option.Option[string] {
				return s[:]
			},
		)),
		ElaEqEqNonNullSlice:     elastic.FromUnd(conversion.Nullify(v.ElaEqEqNonNullSlice)),
		ElaEqEqNonNullNullSlice: elastic.Null[string](),
		ElaEqEqNonNullSingle: elastic.FromUnd(und.Map(
			und.Map(
				conversion.WrapLen1(und.Defined(v.ElaEqEqNonNullSingle)),
				func(s [1]string) (out [1]option.Option[string]) {
					for i := 0; i < 1; i++ {
						out[i] = option.Some(s[i])
					}
					return
				},
			),
			func(s [1]option.Option[string]) []option.Option[string] {
				return s[:]
			},
		)),
		ElaEqEqNonNullNullSingle: elastic.FromUnd(und.Map(
			und.Map(
				conversion.WrapLen1(conversion.OptionUnd(true, v.ElaEqEqNonNullNullSingle)),
				func(s [1]string) (out [1]option.Option[string]) {
					for i := 0; i < 1; i++ {
						out[i] = option.Some(s[i])
					}
					return
				},
			),
			func(s [1]option.Option[string]) []option.Option[string] {
				return s[:]
			},
		)),
		ElaEqEqNonNull: elastic.FromUnd(und.Map(
			und.Map(
				und.Defined(v.ElaEqEqNonNull),
				func(s [3]string) (out [3]option.Option[string]) {
					for i := 0; i < 3; i++ {
						out[i] = option.Some(s[i])
					}
					return
				},
			),
			func(s [3]option.Option[string]) []option.Option[string] {
				return s[:]
			},
		)),
		ElaEqEqNonNullNull: elastic.FromUnd(und.Map(
			und.Map(
				conversion.OptionUnd(true, v.ElaEqEqNonNullNull),
				func(s [3]string) (out [3]option.Option[string]) {
					for i := 0; i < 3; i++ {
						out[i] = option.Some(s[i])
					}
					return
				},
			),
			func(s [3]option.Option[string]) []option.Option[string] {
				return s[:]
			},
		)),
	}
}
```

コード生成が楽になるようなヘルパーを定義してもこの生成量です。

さらに、フィールドがこの`UndRaw`/`UndPlain`という変換メソッドをを実装する際にはそれを呼び出せるようにします。
ObjectにObjectやArrayがネストしているJSONは普通に存在していますから、これができないと実用に耐えないですね。

つまり以下のような、`IncludesImplementor`が存在すると

```go
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

---
package sub

type IncludesImplementor struct {
	Foo sub2.Foo[int]
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

### 見つけたいもの

上記のすべてを叶えるためには

- 受け取ったパッケージ内のすべての型宣言を列挙
- 特定の型（i.e.`option.Option[T]`）を含むフィールドの検知
- 特定のメソッドを実装する型をフィールドに含む型の検知
- さらに上記の2つの検知にかかった型をフィールドに含む方を含む方を芋づる式に検知

する必要がある

### 検知方法の検討

型の依存性をグラフ化して依存関係を探索、条件を満たす型から上に向けてトラバースすることで生成対象の型を列挙する。

- `pkgs []*packages.Package`を引数にとる
- `pkgs`を全部探索して型を列挙する
- `matcher`を引数に取り、これにマッチする型をリストしておく
- 型同士の依存関係をエッジとして型依存グラフを形成する
- 依存関係探索時、`pkgs`外の`*types.Named`についても関してもmatcherを実行する。マッチする場合、externalとしてリストしておく
- マッチした型からupward traverseすることで、芋づる式の検知を可能にする

[Go]: https://go.dev/
[Go1.18]: https://tip.golang.org/doc/go1.18
[Go1.23]: https://tip.golang.org/doc/go1.23
[GoのJSONのT | null | undefinedは\[\]Option\[T\]で表現できる]: https://zenn.dev/ngicks/articles/go-json-undefined-or-null-slice
[*ast.ArrayType]: https://pkg.go.dev/go/ast@go1.23.2#ArrayType
[*ast.MapType]: https://pkg.go.dev/go/ast@go1.23.2#MapType
[*ast.ChanType]: https://pkg.go.dev/go/ast@go1.23.2#ChanType
[*types.Array]: https://pkg.go.dev/go/types@go1.23.2#Array
[*types.Slice]: https://pkg.go.dev/go/types@go1.23.2#Slice
[*types.Map]: https://pkg.go.dev/go/types@go1.23.2#Map
[*types.Chan]: https://pkg.go.dev/go/types@go1.23.2#Chan
[github.com/oapi-codegen/oapi-codegen]: https://github.com/oapi-codegen/oapi-codegen
[github.com/dave/dst]: https://github.com/dave/dst
[go/ast]: https://pkg.go.dev/go/ast@go1.22.6
[golang.org/x/tools/go/packages]: https://pkg.go.dev/golang.org/x/tools@v0.24.0/go/packages
[github.com/ngicks/und]: https://github.com/ngicks/und
[sliceund.Und]: https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/sliceund#Und
[sliceelastic.Elastic]: https://pkg.go.dev/github.com/ngicks/und@v1.0.0-alpha5/sliceund/elastic#Elastic
[Elasticsearch]: https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro.html
