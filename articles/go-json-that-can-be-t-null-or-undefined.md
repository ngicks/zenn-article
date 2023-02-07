---
title: "Goのstruct fieldでJSONのundefinedとnullを表現する"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## Overview

Go で JSON を扱うとき、Elasticsearch の update api に渡す JSON のような `null` と `undefined` をだし分けられるデータ構造や、それに対応する JSON marshaller が欲しくて軽く探したところ、見つからなかったので興味本位で作ってみました。

成果物はこちらです。

https://github.com/ngicks/undefinedablejson

この記事では

- どういう事で困っていたのか
- 既存の方法にはどういうものがあったのか
- 実装の仕方を通じて`encoding/json`について

などを書いていきます。

## 前提知識

- 特にないですが [Go programming language](https://go.dev/) の細かい説明はしないので、ある程度知っている人じゃないと意味が分からないかもしれません。

## 対象読者

- Go の struct field で`undefined | null | T`をだし分けたい人
- `encoding/json`のポイントを知りたい人

## 背景

筆者は業務で TypeScript をよく書きます。Go はほぼ完全に趣味でしか使っていないですが、業務でねじ込めそうなところがあればできる限りねじ込んでいこうとしているところです。そこで困るのが、TypeScript では難なくできていたことが Go でできないことがあることで、導入をそこでためらわせられたくないので解決方法を日頃考えています。

TypeScript のアプリでは入力 JSON を`JSON.parse`でパーズし、入力フォーマットの `interface` を定義し、[ts-auto-guard](https://github.com/rhys-vdw/ts-auto-guard)で生成した typeguard を使うことで型的な整合性のチェックをし、その後アプリの validation ロジックを手組して厳密な validation を実現していました。

TypeScript は有効な JavaScript を全てをうまく取り扱えるように努力がされているため、当然 JavaScript の`undefined`と`null`という二つの「データがない状態」を使うことができました。ただ、「データがない状態」が特別な努力なく複数種類ある言語は知っている限り珍しいため、これらを使いこなしたコードを書いてしまうと他の言語への移植で少し困ります。

Go でも当然、「データがない状態」の表現は存在しますが、それには通常(知っている限り)`*T`というポインターを使います。当然 2 種類はありません。そこでいくつかの困りごとが発見されました。

### 困りごと

Go の struct に JSON から変換/逆変換を行うときに struct 上では`undefined`(=JSON に key がなかった)と`null`と、`T`が区別が付かないため

- 入力値の`required`(=必須キー)や、`null`を許容するなどの入力ルールを実現できない
- また、同様に相手システムが `null` と `undefined` を分けて使う場合、だし分けるのに当該 struct 以上の追加のデータが必要であるため煩雑であること

という悩みがありました。

### `undefined`と`null`を分けて扱うシステムがあった

HTTP で JSON を送る UPDATE や PATCH のとき、`undefined`(=キーが存在しない)のとき field をスキップ、`null`のとき field をクリアするか`null`で上書き、`T`の時`T`で上書き、という挙動をさせる API があることがあり、筆者もそのような API を構築することがあります。

実例としては、Elasticsearch の update api では [partial document を送ること](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update.html#_update_part_of_a_document)でドキュメントの各 field を更新でき、`null` をセットすることで field を `null` で上書きできます。

### validation だけなら JSON からの変換時に JSON schema などで行える

JSON のバイト列が意識される段階では validation そのものは JSON schema などで行うことができます。

JSON schema や OpenAPI spec の JSON schema 部分を読み込んで validation をかける事ができるライブラリはいくつかあります。

- https://github.com/xeipuuv/gojsonschema
- https://github.com/santhosh-tekuri/jsonschema
- https://github.com/qri-io/jsonschema

筆者は`github.com/santhosh-tekuri/jsonschema`を [echo](https://echo.labstack.com/) の Binder の実装の中で使って validation をかけるようなことしたことがあります。

```go: こんな感じ.go
var _ echo.Binder = &ValidatingBinder{}

type ValidatingBinder struct{}

// Bind binds body to i. It ignores path params and query params.
func (b *ValidatingBinder) Bind(i interface{}, c echo.Context) (err error) {
	jsonBytes, err := io.ReadAll(c.Request().Body)
	if err != nil {
		return err
	}
	c.Request().Body.Close()

	if validator, ok := any(i).(interface{ Validate(data []byte) error }); ok {
		err := validator.Validate(jsonBytes)
		if err != nil {
			return err
		}
	}

	err = json.Unmarshal(jsonBytes, i)
	if err != nil {
		return err
	}

	return nil
}

```

Bind 対象の i に Validate メソッドの実装をしておけば validation も一緒にかけてくれます。筆者は Validate の実装の中で JSON schema を使っていました。OpenAPI spec の yaml を解析して JSON pointer を自動生成するなど追加で少し面倒なコードを書く必要がありましたが、簡単な実装ならこれで十分ですね。

## 課題: struct field だけで"undefined | null | T"を表現することは(std 範疇では)できない

validation は上記の方法でイイ感じにできそうなのがわかりました。(軽く書いてますがこの方法にたどり着くまで結構時間がかかりました)。

なので残った要求は

- 取り扱いやすい方法で`undefined | null | T`を struct field で表現する

ことです。

しかし後続セクションで述べる理由により、std 範疇では難しいことがわかります。

## Go の standard library における JSON "null", "undefined" の取り扱い

まず解決方法を考える前に Go の standard library で JSON を取り扱う`encoding/json`が`null`や`undefined`をどのように処理するのか確認します。

### 基本: 変換/逆変換

まず基本的な情報として、変換/逆変換の方法について触れます。

Go では standard library の`encoding/json`によって JSON と Go の値との変換/逆変換を行います。

```go
type Sample struct {
	Foo string
	Bar int
}

// []byte, errorを返す。
bin, err := json.Marshal(Sample{"foo", 123})
if err != nil {
	panic(err)
}
fmt.Printf("%s\n", string(bin)) // {"Foo":"foo","Bar":123}

var s Sample
err = json.Unmarshal(bin, &s)
if err != nil {
	panic(err)
}
fmt.Printf("%+v\n", s) // {Foo:foo Bar:123}
```

対象の type が`json.Marshaler`, `json.Unmarshaler`を実装している場合、そちらが優先して使われます。

```go
type Sample2 struct {
	Foo string
	Bar int
}

func (s Sample2) MarshalJSON() ([]byte, error) {
	return []byte(fmt.Sprintf("{\"foo\":\"f_%s\",\"bar\":\"%d\"}", s.Foo, s.Bar)), nil
}

func (s *Sample2) UnmarshalJSON(data []byte) error {
	mm := make(map[string]any, 2)
	err := json.Unmarshal(data, &mm)
	if err != nil {
		return err
	}

	s.Foo = strings.TrimLeft(strings.TrimLeft(mm["foo"].(string), "f"), "_")
	num, _ := strconv.ParseInt(mm["bar"].(string), 10, 64)
	s.Bar = int(num)
	return nil
}

bin, err = json.Marshal(Sample2{Foo: "foo", Bar: 123})
if err != nil {
	panic(err)
}
fmt.Printf("%s\n", string(bin)) // {"foo":"f_foo","bar":"123"}

var s2 Sample2
err = json.Unmarshal(bin, &s2)
if err != nil {
	panic(err)
}
fmt.Printf("%+v\n", s2) // {Foo:foo Bar:123}
```

### JSON null

> Array and slice values encode as JSON arrays, except that []byte encodes as a base64-encoded string, and a nil slice encodes as the null JSON value.
>
> Pointer values encode as the value pointed to. A nil pointer encodes as the null JSON value.
>
> Interface values encode as the value contained in the interface. A nil interface value encodes as the null JSON value.
>
> https://pkg.go.dev/encoding/json@go1.20.0

`encoding/json`では引用の通り、json.Marshal は以下の条件で`null`を出力します

- pointer type で nil のとき。
  - フィールドの型が`*T | []T | map[T]U | interface`で値が`nil`であるとき
- もしくは json.Marshaler(`MarshalJSON`メソッド) で`[]byte("null")`を返したとき。

(`[]byte`が `base64 string` になるのがちょっとびっくりしますね。)

> ...Unmarshal first handles the case of the JSON being the JSON literal null. In that case, Unmarshal sets the pointer to nil.
>
> Because null is often used in JSON to mean “not present,” unmarshaling a JSON null into any other Go type has no effect on the value and produces no error.
> https://pkg.go.dev/encoding/json@go1.20.0

逆に json.Unmarshal では

- non pointer type `T`に対して `null` は代入しない
- pointer type `*T`に対して `null` は `nil` を代入

という挙動と記述されています。

type が`json.Unmarshaler`を実装する場合、JSON byte が`null`であるときでも呼び出されるため、`[]byte("null")`を好きに変換することができます。

### JSON undefined

JSON というか、JavaScript には`null`とは別に`undefined`という「データがない状態」の表現が存在します。
[JSON の定義](https://datatracker.ietf.org/doc/html/rfc8259)上`undefined`は存在しないのでシリアライズされるときにキーが消える挙動となります。

`json.Marshal`では、omitempty という struct tag で当該 struct field のスキップを行います。

> The "omitempty" option specifies that the field should be omitted from the encoding if the field has an empty value, defined as false, 0, a nil pointer, a nil interface value, and any empty array, slice, map, or string.
>
> https://pkg.go.dev/encoding/json@go1.20.0

- 設定時 [Marshal でフィールドがスキップされるような処理になります](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/encode.go;l=748;drc=8c17505da792755ea59711fc8349547a4f24b5c5;bpv=1;bpt=1)
- [条件はここで網羅されています](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/encode.go;drc=d5de62df152baf4de6e9fe81933319b86fd95ae4;l=339)
- MarshalJSON メソッドで空のバイト列(`[]byte("")`)などを返すとエラーです。
  - [この記述](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/indent.go;l=17;drc=8c17505da792755ea59711fc8349547a4f24b5c5)からわかるように、返すことが許されるのは、有効な JSON 文字列のみです。

`json.Unmarshal`では、

- JSON バイト列の中に対応する key がない場合、単に代入しない挙動となります。
  - zero-value のまま置かれると思ってもいいでしょう。

### `encoding/json`では"undefined | null | T"のだし分けできない

前記の通り、`undefined`の表現(=フィールドをスキップする)と`null`はデータではなくメタデータとして実装されています。

```go
type Sample3 struct {
	Foo string  `json:",omitempty"`
	Bar *string `json:",omitempty"`
	Baz *string
}


func main() {
	var emptyStr string
	bar := "bar"
	baz := "baz"
	must := func(v []byte, err error) []byte {
		return v
	}
	fmt.Printf("%s\n", string(must(json.Marshal(Sample3{Foo: "foo", Bar: &bar, Baz: &baz}))))
	fmt.Printf("%s\n", string(must(json.Marshal(Sample3{Foo: "", Bar: &emptyStr, Baz: &emptyStr}))))
	fmt.Printf("%s\n", string(must(json.Marshal(Sample3{Foo: "", Bar: nil, Baz: nil}))))
}
/*
	{"Foo":"foo","Bar":"bar","Baz":"baz"}
	{"Bar":"","Baz":""}
	{"Baz":null}
*/
```

こういう挙動です。

type のみ(= `json.Marshaler` / `json.Unmarshaler` のみ)によって`undefined` / `null`を表現し分けることは、std の範疇ではできなさそうなことは確認が取れました。

## 解決方法: "undefined | null | T"を表現できる type を作る

"go JSON undefined"などでググってみましたがこれを実現しているライブラリーは見つかりませんでした。おそらくはある気がしますが、気が向いたしすぐ作れる気がするので作ってみます。

- `undefined | null | T`を表現できる構造体を定義することとします。
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

そのため、便利な helper method の定義ができません。

- `**T`を引数にとる関数を定義すると煩雑です。
- method set が定義できないなら`any`な値が`Undefined`型であるのかを判別するのが煩雑になります。
  - go の generics の制限によって、メソッドは type param を持てませんので、具体的な型を指定せずに`Undefined`型に対しての [type assertion](https://go-tour-jp.appspot.com/methods/15) ができません。
  - `interface { IsUndefined() bool }`のような interface に対する type assertion を行うこともできません。
  - reflect を使う方法は取れると思いますが、[reflect.ValueOf は(現状は)かならず heap にデータを escape する](https://cs.opensource.google/go/go/+/refs/tags/go1.20:src/reflect/value.go;l=3144-3147)ので避けられるなら避けたほうがいいでしょう。

### 解法: boolean flag で defined を表現する。

ところで、std の`database/sql`には以下のようなデータ構造が定義されています。

https://pkg.go.dev/database/sql@go1.20#NullBool

```go
type NullBool struct {
	Bool  bool
	Valid bool // Valid is true if Bool is not NULL
}
```

Go 1.18 で追加された Generics を利用すればこれをあらゆる type `T`について定義できますし、これを 2 段重ねにするだけ目的を達成できそうですね。単純な解法ですが、`*T`に引っ張られて見落としていました。

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

var nullByte = []byte(`null`)

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

上記の`Undefinedable[T]`は struct であるので、前記の通り omitempty によるスキップ動作が起きません。そこで、専用の Marshaller を作成して`undefined`時に skip 可能にします。

#### ナイーブな発想版

単純な発想によれば、reflect によって field の値を読み取りながら、上記の Undefinedable[T]の場合かつ Undefined の時だけ field をスキップする挙動で目的は達せます。

もちろん`Undefinedable[T]`と`Nullable[T]`に MarshalJSON と UnmarshalJSON の実装が必要ですが、自明であるので省略します。

`encoding/json`は内部で利用している key などの string の escape を行う機能を外部に公開しないため、[github.com/mailru/easyjson](https://github.com/mailru/easyjson)の jwriter に委譲します。

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

json.Marshal は struct tag によってカスタマイズが可能です。上記ナイーブな発想版の Marshaller はこれらのタグを無視してしまいますので手動でサポートを加える必要がありますね。完全無視する挙動でも問題ないといえば問題ありませんが実用上の問題が出そうなので実装してみましょう。

json の struct tag は以下のような構造を取ります

`json:"name,opt,opt"`

タグは field name、そこからカンマ区切りでオプションが並びます。

- name 部分で marshal 後の JSON key がカスタマイズできます。
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
+		// json.Marshal does not quote the `null` value.
+		quote := shouldQuote(field.Type(), field.Interface()) && string(marshalled) == "null"
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

このキャッシュの部分は`sync.WaitGroup`を使って同期させる部分が存在していて、これはおそらく非同期的に同時に呼び出されてもいいようになっているのだと思います。この処理を読み込みや再現はそこそこ複雑そうだったのでとりあえずの処置として[簡易的な実装](https://github.com/ngicks/undefinedablejson/blob/main/serde.go#L125-L138)で済ませています。

また、struct type への string option は staticcheck `SA5008`警告されてしまうので、warning 回避のために`und:"string"`でも同様の判定を行うように変更しました。

## おわりに

ここまで書いておいてなんですが、本当にこの方法が最良なのでしょうか？他に誰かが似たようなことをすでにしている気がするんです。筆者が記事執筆前に軽く調べた限りそれらしい情報は出てこなかったですし、作ってみたかったので作ってみただけなので別にあったとしてもいいのですが。

今後は

- code generator の追加
  - `reflect`の使用を避けるため。
- `required`, `disallowNull`などのバリデーションルールを追加し、Unmarshal 時に validate する
- 似たようなことをしている package を探して、**あった場合それに貢献する**。

をすることになると思います。

実はこのライブラリ自体は Elasticsearch の JSON をいい感じに扱うために作っていたライブラリを一部切り出したものなので、筆者が Elasticsearch とやり取りする Go アプリを書くまで本格使用することがないため、問題点が明らかにならない可能性が高いです。
