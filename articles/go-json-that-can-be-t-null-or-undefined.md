---
title: "Goのstruct fieldでJSONのundefinedとnullを表現する"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## Overview

TypeScript で書いていたアプリを Go に移植するときに少し困るのが JSON との変換、逆変換です。

Go では standard library の`encoding/json`によって JSON と Go の値との変換/逆変換を行います。`encoding/json`では JSON の値が不整合の場合、エラーにならず単にスキップするなど、思ったよりも厳密ではありません。

Go では、`null | T`もしくは`undefined | T`を表現するのに、`*T`を利用します。また、Go には zero value の概念があり、変数は必ずこの zero value に初期化されます。ポインターの場合、zero value は`nil`になります。`encoding/json`パッケージでは、JSON の`null`は`nil`へ変換されます。つまり、入力 JSON にキーがないことと、キーに`null`がセットされていることは、同様に`nil`として取り扱われます。

TypeScript のアプリでは入力 JSON を`JSON.parse`でパーズし、入力フォーマットの interface を定義し、[ts-auto-guard](https://github.com/rhys-vdw/ts-auto-guard)で生成した typeguard を使うことで厳密な入力値の validation をおこなっていました。
では Go ではどうするのでしょうか？

この記事では便利にこの問題を解決する方法を考えライブラリとして実装しました。

https://github.com/ngicks/undefinedablejson

## Prerequisites

- 特にないですが [Go programming language](https://go.dev/) の細かい説明はしないので、ある程度知っている人じゃないと意味が分からないかもしれません。

## Background

JSON はその名の通り JavaScript の Object の記法なので JavaScript らしい事情があります。
具体的には、フォーマットを事前に決めない方式であるのでキーはあらゆる型を取ることができることや、値がないことの表現に`undefined`と`null`の二つがあることです。
[JSON の定義](https://datatracker.ietf.org/doc/html/rfc8259)上`undefined`は存在しないのでシリアライズされるときにキーが消える挙動となります。

JSON を受け取る REST API などを作るとき、`PATCH` method のときに`undefined`と`null`があると便利で、

- `T`の場合、値を上書き
- `undefined`(= つまり key がない)とき、値を更新しない
- `null`のとき、値を空にする(= つまり`undefined`にするか、`null`で上書きする)

のようなことを表現できます。いくつかのシステム(e.g. Elasticsearch)はこれをうまく利用するため、そのようなシステムとうまくやり取りするための方法が必要とされています。

他方、Go は type だけでこの`(T | undefined | null)`を区別して取り扱う方法が std の範囲ではありません。そのためユーザーコードでの努力が求められます。

また、`encoding/json`は柔軟であり、数値のオーバーフローや non pointer type に対する`null`などは単にフィールドへの代入をスキップするなど、エラーになってほしいのにならないパターンもあります。

### validation したいだけなら json schema を使うことができる

json schema や OpenAPI spec の json schema 部分を読み込んで validation をかける事ができるライブラリはいくつかあります。

- https://github.com/xeipuuv/gojsonschema
- https://github.com/santhosh-tekuri/jsonschema
- https://github.com/qri-io/jsonschema

筆者は`github.com/santhosh-tekuri/jsonschema`を [echo](https://echo.labstack.com/) の Binder の実装の中で使って validation をかけるようなことしたことがあります。少々非効率な方法にはなりますが、十分ではありました。

### 追加のデータペイロードなしで null やフィールドをスキップする実装はできない(はず)

validation は json schema などで十分可能ではありますが、逆に json を出力際にキーをスキップしたり、null にしたりなどを任意に行うには struct のインスタンス以外の追加のデータが必要なままです。

## 解決したい課題: T | undefined | null を表現する type がないこと

ここで解決したい課題は以下となります。

- type で`T | undefined | null`を全て表現する
  - PATCH request の発行側として、など

### json.Marshal / Unmarshal のそれらの挙動を表現できない

`encoding/json`では以下のような挙動となっているため、課題を解決するにはこれだけでは不十分です。

#### Marshal

https://pkg.go.dev/encoding/json@go1.20.0

> Array and slice values encode as JSON arrays, except that []byte encodes as a base64-encoded string, and a nil slice encodes as the null JSON value.
>
> Pointer values encode as the value pointed to. A nil pointer encodes as the null JSON value.
>
> Interface values encode as the value contained in the interface. A nil interface value encodes as the null JSON value.

- 引用の通り、json.Marshal は以下の条件で`null`を出力します
  - フィールドの型が`*T | []T | map[T]U | interface`で値が`nil`であるとき
  - もしくは MarshalJSON で`[]byte("null")`を返したとき。
- `undefined`を表現するためには struct tag で`omitempty`設定し、フィールドが struct 以外の型かつ zero value にします。
  - [Marshal でフィールドがスキップされるような処理になります](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/encode.go;l=748;drc=8c17505da792755ea59711fc8349547a4f24b5c5;bpv=1;bpt=1)
  - [条件はここで網羅されています](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/encode.go;drc=d5de62df152baf4de6e9fe81933319b86fd95ae4;l=339)
- MarshalJSON メソッドで空のバイト列(`[]byte("")`)などを返すとエラーです。
  - [この記述](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/indent.go;l=17;drc=8c17505da792755ea59711fc8349547a4f24b5c5)からわかるように、返すことが許されるのは、有効な JSON 文字列のみです。

type のみ(= MarshalJSON のみ)によって`undefined` / `null`を表現し分けることは、std の範疇ではできないようです。

#### Unmarshal

- `omitempty`は無関係です
  - Unmarshal は入力される JSON の`[]byte`に含まれていないとき単に値を入れない挙動をするようなので、zero value のままです。`omitempty`タグに関する記述は json.Unmarshal の doc コメントにはなく無関係なことがわかります。
- pointer type に対して`null`が入力されると `nil` になります。
- 値の型が代入不可だとエラーになるのですが、`null`の場合は non pointer type に対しては代入しない挙動となります。そこから、`null`は zero-value に置き換えられるといっても間違いではありません。

> If a JSON value is not appropriate for a given target type, or if a JSON number overflows the target type, Unmarshal skips that field and completes the unmarshaling as best it can.

[sample code](https://go.dev/play/p/JH1mTO2pXho)のような挙動になります。

```go
type Sample struct {
	Foo string
	Bar *string
}

func (s Sample) Print() {
	var b string
	if s.Bar == nil {
		b = `<nil>`
	} else {
		b = *s.Bar
	}

	fmt.Printf("{Foo:%s,Bar:%s}\n", s.Foo, b)
}

for _, data := range [][]byte{
  []byte(`{"Foo":"foo","Bar":"bar"}`),
  []byte(`{"Foo":null,"Bar":null}`),
  []byte(`{}`),
} {
  var sample Sample
  err := json.Unmarshal(data, &sample)
  if err != nil {
    panic(err)
  }
  sample.Print()
}
/*
{Foo:foo,Bar:bar}
{Foo:,Bar:<nil>}
{Foo:,Bar:<nil>}
*/
```

## Prior Works

### JSON 同士の結合処理

- [github.com/evanphx/json-patch](https://github.com/evanphx/json-patch)によって JSON の結合処理を行ってくれるようです。
  - このライブラリは記事作成中に探したものなので、詳しい挙動はわかりませんが・・・

### decode 時の validation - jsonparser による

[github.com/buger/jsonparser](https://github.com/buger/jsonparser)のような、json 構造を解析できるライブラリの力によって、decode 時の validation は可能です。

```go
jsonparser.ObjectEach(
	[]byte(`{"Foo":"foo2","Bar":null}`),
	func(key, value []byte, dataType jsonparser.ValueType, offset int) error {
		if string(value) == `null` {
			fmt.Printf("%s is null!\n", string(key))
		} else {
			fmt.Printf("%s = %s\n", string(key), string(value))
		}
		return nil
	},
)
```

```
Foo = foo2
Bar is null!
```

### encode 時の null 埋め、キーの削除 - jwriter による

[github.com/mailru/easyjson](https://github.com/mailru/easyjson)の`jwriter`を利用することで、json の encode 処理を手書きするのがずいぶん楽になりますので、好きな処理を盛り込めます。

```go
func getFieldName(field reflect.StructField) string {
	// 実際にはstruct tagを解析する。
	return field.Name
}

func enc(v any, skip ...string) ([]byte, error) {
	rv := reflect.Indirect(reflect.ValueOf(v))

	writer := jwriter.Writer{}
	writer.RawByte('{')

	appendComma := false

fieldLoop:
	for i := 0; i < rv.NumField(); i++ {
		field := rv.Field(i)
		name := getFieldName(rv.Type().Field(i))

		for _, s := range skip {
			if name == s {
				continue fieldLoop
			}
		}

		if appendComma {
			writer.RawString(",")
		}
		appendComma = true

		writer.String(name)
		writer.RawString(":")

		writer.Raw(json.Marshal(field.Interface()))
	}

	writer.RawString("}")

	if writer.Error != nil {
		return nil, writer.Error
	}

	var buf bytes.Buffer
	buf.Grow(writer.Size())
	if _, err := writer.DumpTo(&buf); err != nil {
		return nil, err
	}

	return buf.Bytes(), nil
}

```

## 既存の方法の問題

struct でデータを持ちまわる際に、field の type で undefined と null を表現し分けられないので、上記のように追加のデータを必要として面倒です。もっとすっきりした解決方法が欲しいところです。

## 解決方法: undefined と null を表現できる type を作る

- 解決方法として、`(undefined | null | T)`を表現できる構造体を定義することとします。
  - `**T`のようなポインタータイプを base type とする defined type はメソッドを持てませんので struct にします。
  - struct は omiempty によるスキップがぜったいに起こりませんので専用の marshaller が必要です。

### 問題:\*\*T はメソッドを持てない

Go では「データがない」状態を表現することに`*T` を利用します。

`*T`が`undefined | T`、`null | T`を表現するならば、単純に`**T`で`undefined | null | T`を表現できます。

ただし、以下のようなメソッドの実装はできません。

```go
type Undefined[T any] **T

func (u Undefined[T]) IsUndefined() bool {
	return u == nil
}

func (u Undefined[T]) IsNull() bool {
	if u == nil {
		return false
	}
	return *u == nil
}
```

これは Go の言語仕様で明確に禁止されています。

> A receiver base type cannot be a pointer or interface type
>
> https://go.dev/ref/spec#Method_declarations

### 解法: boolean flag で defined を表現する。

ところで、std の`database/sql`には以下のようなデータ構造が定義されています。

https://pkg.go.dev/database/sql@go1.20#NullBool

```go
type NullBool struct {
	Bool  bool
	Valid bool // Valid is true if Bool is not NULL
}
```

これに`type param [T]`をつけ足して 2 段重ねにするだけ目的を達成できそうですね。単純な解法ですが、`*T`に引っ張られて見落としていました。

ということで定義は以下のようになります。

まず`Option[T]`を定義して

```go: option.go
type Option[T any] struct {
	some bool
	v    T
}

func (o Option[T]) IsSome() bool {
	return o.some
}

func (o Option[T]) IsNone() bool {
	return !o.IsSome()
}

func (o Option[T]) Value() *T {
	if !o.some {
		return nil
	}
	return &o.v
}

func (o Option[T]) MarshalJSON() ([]byte, error) {
	if o.IsNone() {
		return nullByte, nil
	}
	return json.Marshal(o.v)
}

func (o *Option[T]) UnmarshalJSON(data []byte) error {
	if string(data) == string(nullByte) {
		o.some = false
		return nil
	}

	o.some = true
	return json.Unmarshal(data, &o.v)
}
```

`Nullable[T]`は`Option[T]`とほぼ等価ですが、`type Nullable[T] = Option[T]`は type param があるときエイリアスできないコンパイラルールで不可能ですし、`type Nullable[T] Option[T]`では method set を継承しない go のルールがあるため意味がないので embed を使うことにします。

```go: field.go
type Nullable[T any] struct {
	Option[T]
}

func (n Nullable[T]) IsNull() bool {
	return n.IsNone()
}

func (n Nullable[T]) IsNonNull() bool {
	return n.IsSome()
}

type Undefinedable[T any] struct {
	Option[Option[T]]
}

func (u Undefinedable[T]) IsUndefined() bool {
	return u.IsNone()
}

func (u Undefinedable[T]) IsDefined() bool {
	return u.IsSome()
}

func (u Undefinedable[T]) IsNull() bool {
	if u.IsUndefined() {
		return false
	}
	return u.v.IsNone()
}

func (u Undefinedable[T]) IsNonNull() bool {
	if u.IsUndefined() {
		return false
	}
	return u.v.IsSome()
}

func (f Undefinedable[T]) Value() *T {
	if f.IsUndefined() {
		return nil
	}
	return f.v.Value()
}
```

### 専用の Marshaller を作る。

上記の`Undefinedable[T]`は struct であるので、前期の通り omitempty によるスキップ動作が起きません。そこで、専用の Marshaller を作成して`undefined`時に skip 可能にします。

#### ナイーブな発想版

単純な発想によれば、reflect によって field の値を読み取りながら、上記の Undefinedable[T]の場合かつ Undefined の時だけ field をスキップする挙動で目的は達せます。

もちろん`Undefinedable[T]`と`Nullable[T]`に MarshalJSON と UnmarshalJSON の実装が必要ですが、自明であるので省略します。

```go marshal_fields_json.go
func MarshalFieldsJSON(v any) ([]byte, error) {
	rv := reflect.ValueOf(v)

	if rv.Type().Kind() != reflect.Struct {
		panic("not a struct!")
	}

	writer := jwriter.Writer{}
	writer.RawByte('{')

	appendComma := false

	for i := 0; i < rv.NumField(); i++ {
		field := rv.Field(i)
		fieldName := getFieldName(rv.Type().Field(i))

		und, ok := field.Interface().(interface{ IsUndefined() bool })
		if ok && und.IsUndefined() {
			continue
		}

		if appendComma {
			writer.RawString(",")
		}
		appendComma = true

		writer.String(fieldName)
		writer.RawString(":")

		writer.Raw(json.Marshal(field.Interface()))
	}

	writer.RawString("}")

	if writer.Error != nil {
		return nil, writer.Error
	}

	var buf bytes.Buffer
	buf.Grow(writer.Size())
	if _, err := writer.DumpTo(&buf); err != nil {
		return nil, err
	}

	return buf.Bytes(), nil
}
```

この方法は reflect の使用などのせいで実行時の効率は悪いですがほとんどの処理を json.Marshal に委譲できるのでメンテが楽という利点があります。json string の escape も jwriter に委譲します。

#### json.Marshal と同じ struct tag をサポートする

##### json.Marshal の struct tag

json.Marshal は struct tag によってカスタマイズが可能です。上記ナイーブな発想版の Marshaller はこれらのタグを無視してしまいますので手動でサポートを加える必要がありますね。

json の struct tag は以下のような構造を取ります

`json:"name,opt,opt"`

タグは field name、そこからカンマ区切りでオプションが並びます。

- name 部分で marshal 後の json key がカスタマイズできます。
  - 全く別の名前にもできますが、基本的には Go の Visibility rule で PascalCase にしてある field 名を snake_case に変換するのが主な目的かと思います。
- name 部分を`-`にすると、field を無視します。
  - ただし`-,`とすると`-`を出力できます。
- omitempty オプションで field が zero value であるときスキップさせます。
  - [条件はここで網羅されており](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/encode.go;drc=d5de62df152baf4de6e9fe81933319b86fd95ae4;l=339)、見て分かる通り長さ 1 以上の Array と Struct は絶対に skip されません
- string オプションで encode / decode 時に`"`によって値を quote します。
  - string, int, unsigned int, float, bool のみに有効でそれ以外の型では無視されます。[条件はここに乗っています](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/encode.go;l=1275-1286;drc=d5de62df152baf4de6e9fe81933319b86fd95ae4;bpv=1)
  - おそらく`"true"`/`"false"`を使うシステムや、JavaScript のライブラリによっては [BigInt](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/BigInt) の JSON 表現に string 型を選ぶことがある事への対応かと思います。

##### 実装する

name 部分の取得、option の取得を以下のように実装します。

- `tagged`は struct tag で field 名を指定したか、shouldSkip はタグが`-`であったかを示します。

```go
func GetFieldName(field reflect.StructField) (fieldName string, options string, tagged bool, shouldSkip bool) {
	tagged = true
	fieldName, options, shouldSkip = GetJsonTag(field)
	if len(fieldName) == 0 {
		tagged = false
		fieldName = field.Name
	}
	return fieldName, options, tagged, shouldSkip
}

func GetJsonTag(field reflect.StructField) (name string, opt string, shouldSkip bool) {
	tag := field.Tag.Get("json")
	if tag == "-" {
		return "", "", true
	}
	name, opt, _ = strings.Cut(tag, ",")
	return name, opt, false
}
```

- OptContain で option が含まれているかの判定をします。
  - (`encoding/json`内部で使われているのとほぼ同じコードです)
  - json option は boolean flag なのでこの実装で十分です。

```go
func OptContain(options string, target string) bool {
	if len(options) == 0 {
		return false
	}
	var opt string
	for len(options) != 0 {
		opt, options, _ = strings.Cut(options, ",")
		if opt == target {
			return true
		}
	}
	return false
}
```

omitempty ルールを再現するために、Empty 判定をする関数を作ります。これはまるっきりドキュメントの通りに実装しただけです。

```go
// IsEmpty reports true if v should be skipped when tagged with omitemty, false otherwise.
func IsEmpty(v reflect.Value) bool {
	switch v.Kind() {
	// false
	case reflect.Bool:
		return !v.Bool()
		// 0
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		return v.Int() == 0
	case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64, reflect.Uintptr:
		return v.Uint() == 0
	case reflect.Float32, reflect.Float64:
		return math.Float64bits(v.Float()) == 0
		// empty array
	case reflect.Array:
		return v.Len() == 0
		// nil interface or pointer
	case reflect.Interface, reflect.Pointer:
		return v.IsNil()
		// empty map, slice, string
	case reflect.Map, reflect.Slice:
		return v.IsNil() || v.Len() == 0
	case reflect.String:
		return v.Len() == 0
	}
	return false
}
```

string ルールを再現するために quotable 判定を行う関数を定義します。

```go
func IsQuotable(ty reflect.Type) bool {
	// string options work only for
	// string, floating point, integer, or boolean types.
	switch ty.Kind() {
	case reflect.String,
		reflect.Float32, reflect.Float64,
		reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64,
		reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64, reflect.Uintptr,
		reflect.Bool:
		return true
	}

	return false
}
```

`Undefinedable[T]`にもこのルールが適用できるように拡張を行います。

```go
func (f Nullable[T]) IsQuotable() bool {
	var t T
	return IsQuotable(reflect.TypeOf(t))
}

func (f Undefinedable[T]) IsQuotable() bool {
	var t T
	return IsQuotable(reflect.TypeOf(t))
}

func shouldQuote(ty reflect.Type, value any) bool {
	if IsQuotable(ty) {
		return true
	}
	if quotable, ok := value.(interface{ IsQuotable() bool }); ok {
		return quotable.IsQuotable()
	}
	return false
}
```

`MarshalFieldsJSON`にもろもろの処理を入れていきます。

```diff go:marshal_fields_json.go
func MarshalFieldsJSON(v any) ([]byte, error) {
	rv := reflect.ValueOf(v)

	if rv.Type().Kind() != reflect.Struct {
		panic("not a struct!")
	}

	writer := jwriter.Writer{}
	writer.RawByte('{')

	appendComma := false

	for i := 0; i < rv.NumField(); i++ {
		field := rv.Field(i)
-		fieldName := getFieldName(rv.Type().Field(i))
+		fieldName, options, tagged, shouldSkip := GetFieldName(rv.Type().Field(i))
+
+		if shouldSkip {
+			// tagged as "-" (and not "-,")
+			continue
+		}
+
+		if OptContain(options, "omitempty") && IsEmpty(field) {
+			continue
+		}

		und, ok := field.Interface().(interface{ IsUndefined() bool })
		if ok && und.IsUndefined() {
			continue
		}

		if appendComma {
			writer.RawString(",")
		}
		appendComma = true

-		writer.String(fieldName)
-		writer.RawString(":")
-
-		writer.Raw(json.Marshal(field.Interface()))
+		var marshalled []byte
+		var err error
+		if !tagged && rv.Type().Field(i).Anonymous && field.Type().Kind() == reflect.Struct {
+			// only the untagged embedded struct field receives same treatment.
+			marshalled, err = MarshalFieldsJSON(field.Interface())
+			if len(marshalled) >= 2 {
+				marshalled = marshalled[1 : len(marshalled)-1]
+			}
+		} else {
+			marshalled, err = json.Marshal(field.Interface())
+			writer.String(fieldName)
+			writer.RawString(":")
+		}
+
+		quote := shouldQuote(field.Type(), field.Interface())
+		if quote {
+			writer.RawString("\"")
+		}
+		writer.Raw(marshalled, err)
+		if quote {
+			writer.RawString("\"")
+		}
	}

	writer.RawString("}")

	if writer.Error != nil {
		return nil, writer.Error
	}

	var buf bytes.Buffer
	buf.Grow(writer.Size())
	if _, err := writer.DumpTo(&buf); err != nil {
		return nil, err
	}

	return buf.Bytes(), nil
}
```

実装しながらこの処理 reflect を含んでるからコンパイラに最適化されないんじゃないか？と疑問に思い、`encoding/json`を読み進めるとフィールドをどう処理するかっていう処理を[sync.Map でキャッシュしていますね。](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/encode.go;l=370;drc=d5de62df152baf4de6e9fe81933319b86fd95ae4;bpv=1)　実際に reflect 処理を含んでいたら最適化がかからないかは今後の調査事項とさせていただきます。

非同期的に同時に呼び出されてもいいような複雑な処理を含んでいますがその部分を[簡易的に実装](https://github.com/ngicks/undefinedablejson/blob/main/serde.go#L125-L138)しました。

## おわりに

ここまで書いておいてなんですが、本当にこの方法が最良なのでしょうか？他に誰かが似たようなことをすでにしている気がするんです。筆者が記事執筆前に軽く調べた限りそれらしい情報は出てこなかったですし、作ってみたかったので作ってみただけなので別にあったとしてもいいのですが。

今後は

- code generator の追加
  - `reflect`の仕様を避けるため。
- `required`, `disallowNull`などのバリデーションルールを追加し、Unmarshal 時に validate する
- 似たようなことをしている package を探して、**あった場合それに貢献する**。

をすることになると思います。

実はこのライブラリ自体は Elasticsearch の JSON をいい感じに扱うために作っていたライブラリを一部切り出したものなので、筆者が Elasticsearch とやり取りする Go アプリを書くまで本格使用することがないため、問題点が明らかにならない可能性が高いです。

[go]: https://go.dev/
[encoding/json]: https://pkg.go.dev/encoding/json@go1.20.0#Unmarshaler
