---
title: "Goのstruct fieldでJSONのundefinedとnullを表現する"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

# Overview

Go で JSON を扱うとき、Elasticsearch の update api に渡す JSON のような `null` と `undefined` をだし分けられるデータ構造や、それに対応する JSON marshaller が Go には std ではなく、軽く探したところ見つからなかったので、興味本位で作ってみました。

成果物はこちらです。

https://github.com/ngicks/undefinedablejson

この記事では

- どういう事で困っていたのか
- 既存の方法にはどういうものがあったのか
  - この話題に関連する今もってる知見をできる限り書いています。
- 実装を通じて`encoding/json`について得られた知見

などを書いていきます。

# 前提知識

- [Go programming language](https://go.dev/) の説明はしないので、ある程度知っている人じゃないと意味が分からないかもしれません。

# 環境

```bash
> go version
go version go1.20 linux/amd64.
```

ドキュメント、ソースコードは全て[Go 1.20](https://tip.golang.org/doc/go1.20)のものを参照していますが、ドキュメントそのものはしばらく変わっていませんのでそれより以前のバージョンでも同様であると予想します。また、[Go 1.18](https://tip.golang.org/doc/go1.18) で追加された generics を利用したソースコードを書きますので、記事中のサンプルコードは `Go 1.18` 以降でのみ動きます。

# 対象読者

- Go の struct field で`undefined | null | T`を表現する方法がわからない人
- Go で JSON を受け取る API を組むときに validation などで悩んでいる人
- `encoding/json`のポイントを知りたい人

# 言葉の定義

本投稿では以後 JSON の key が:

- ないことを `undefined`
- `null` であることを `null`
- ある型 `T` であること `T`
- アプリによって必須であると決められていることを`required`

と呼びます。

# 背景: 時たま困る Go における「データがない状態」の扱い

Go の言語設計のせいでは全くないのですが・・・

## Go の zero value

Go には [The zero value](https://go.dev/ref/spec#The_zero_value) の概念があるため、

> ... a variable or value is set to the zero value for its type: false for booleans, 0 for numeric types, "" for strings, and nil for pointers, functions, interfaces, slices, channels, and maps.

とある通り、変数も struct field も型に対応する zero value に初期化されます。

### zero value はフィールドがない判定に使われることがある

この zero value は、「データがない状態」の判定に使われることがあり、standard library の`reflect`にも例えば、[IsZero](https://pkg.go.dev/reflect@go1.20#Value.IsZero)というメソッドが提供されています。

他には [GORM](https://gorm.io/) という ORM の場合,

> // Struct
> db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
> // SELECT \* FROM users WHERE name = "jinzhu" AND age = 20 ORDER BY id LIMIT 1;
>
> https://gorm.io/docs/query.html

という風に、zero value であるフィールドはデータがセットされていないこととして扱う API が存在します。該当の zero value 判定は[ここなどでおこなわれています](https://github.com/go-gorm/gorm/blob/02b7e26f6b5dcdc49797cc44c26a255a69f3aff3/schema/field.go#L462)(`reflect`の IsZero も使われていますね)

### 外部データ(JSON)をバインドするときの zero value

同じように、JSON など外部データからの変換時、struct にデータをバインドする際に、

```go
type Sample struct {
	Foo string
	Bar int
}
```

のような struct があったとして、

```json
{"Foo": "", "Bar": 0}
{"Foo": null, "Bar": null}
{}
```

を入力した場合、バインドされた struct field の値は、全て区別がつきません。

外部 API として`""`(空白 string) や数値型で `0` があり得ない場合のみ、上記の `Sample` struct にデータをバインドするだけでいいことになります。

`""`がありえない API はそこまで珍しくない気がしますが、 `0` がありえないケースはそこそこ珍しいかなと思います。そこで基本的に以下のようにメンバーをポインターにすることになるのが普通かと思います。

```go
type Sample struct {
	Foo *string
	Bar *int
}
```

しかしこの場合でも、フィールドが`null`であった時と、`undefined`(JSON に key がなかった)時に区別がつきません。

## 外部システムとやり取りする時の`undefined`と`null`

### `undefined`と`null`を分けて扱われることがある

HTTP で JSON を送る UPDATE や PATCH のとき、`undefined`(=キーが存在しない)のとき field をスキップ、`null`のとき field をクリアするか`null`で上書き、`T`の時`T`で上書き、という挙動をさせる API があります。筆者もそういった API を書くことがあります。

広く使われている実例としては、[Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro.html) があります。Elasticsearch の update api では [partial document を送ること](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update.html#_update_part_of_a_document)でドキュメントの各 field を更新でき、`null` をセットすることで field を `null` で上書きできます。

## 既存の方法: required の強制、validation について

### map[string]any

ハッシュマップでは任意の key に対して、(`any` 型を取り扱える Go では)value にあらゆる型を格納可能なため、いったん `map[string]any` にデコードすれば key のあるなし、型の一致などの判定をおこなえます。

下記 JSON schema もライブラリの実装は `map[string]any` を使用しています。

### JSON schema

JSON のバイト列が意識される段階では validation そのものは JSON schema などで行うことができます。

JSON schema や OpenAPI spec の JSON schema 部分を読み込んで validation をかける事ができるライブラリはいくつかあります。

- https://github.com/xeipuuv/gojsonschema
- https://github.com/santhosh-tekuri/jsonschema
- https://github.com/qri-io/jsonschema

筆者は`github.com/santhosh-tekuri/jsonschema`を [echo](https://echo.labstack.com/) の Binder の実装の中で使って validation をかけるようなことしたことがあります。

実装のサンプルは長くなるのでドロップダウンに隠しておきます。

:::details echo.Binder の実装サンプル

- [playground](https://go.dev/play/p/t90GrV94HG8)
  - 依存モジュールが多すぎてタイムアウトするのでローカルで動かしてください。。。

サンプルで`echo.Context`をくみ上げる気が起きなかったので、その部分は単に compilation error が起きないことだけ見せています。

```go: こんな感じ.go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"strings"

	"github.com/labstack/echo/v4"
	"github.com/mitchellh/mapstructure"
	"github.com/santhosh-tekuri/jsonschema/v5"
)

type SampleValidator struct {
	Foo string
}

var schema *jsonschema.Schema

func init() {
	cmp := jsonschema.NewCompiler()
	err := cmp.AddResource("foo.json", strings.NewReader(`{
		"type": "object",
		"properties": {
			"Foo": {
				"type": "string"
			}
		},
		"required": ["Foo"]
	}`))
	if err != nil {
		panic(err)
	}

	schema = cmp.MustCompile("foo.json")
}

func (*SampleValidator) Validate(data map[string]any) error {
	return schema.Validate(data)
}

var _ echo.Binder = &ValidatingBinder{}

type ValidatingBinder struct{}

// Bind binds body to i. It ignores path params and query params.
func (b *ValidatingBinder) Bind(i interface{}, c echo.Context) (err error) {
	return bindValidating(c.Request().Body, i)
}

func bindValidating(r io.Reader, i any) error {
	// encodingを見てdecoderをさらに噛ませた方がいいんですが省略です。
	dec := json.NewDecoder(r)
	dec.UseNumber()

	if closer, ok := r.(io.Closer); ok {
		defer closer.Close()
	}

	jsonMap := map[string]any{}
	if err := dec.Decode(&jsonMap); err != nil {
		return err
	}

	// 省略:
	// token, err := dec.Token()
	// // input stream has additional chars and it is not a valid json token.
	// if err != nil { /*どうする？*/ };
	// // input stream has another json.
	// if token != nil { /*どうする？*/ }

	if validator, ok := any(i).(interface {
		Validate(data map[string]any) error
	}); ok {
		err := validator.Validate(jsonMap)
		if err != nil {
			return err
		}
	}

	err := mapstructure.Decode(jsonMap, i)
	if err != nil {
		return err
	}
	return nil
}

func main() {
	var (
		err error
		sv  SampleValidator
	)
	// 内部でjson.Decoderを使うので後続行にさらにjsonがあっても問題ありません。
	err = bindValidating(strings.NewReader("{\"Foo\":\"foo\"}{\"Foo\":\"bar\"}"), &sv)
	fmt.Printf("%+v\n", err) // <nil>
	fmt.Printf("%+v\n", sv)  // {Foo:foo}

	sv = SampleValidator{}
	err = bindValidating(strings.NewReader(`{}`), &sv)
	fmt.Printf("%+v\n", err) // jsonschema: '' does not validate with file:///<path to cwd>/foo.json#/required: missing properties: 'Foo'
	fmt.Printf("%+v\n", sv)  // {Foo:}
}
```

- `github.com/xeipuuv/gojsonschema` も `github.com/santhosh-tekuri/jsonschema` もどちらも JSON をいったん`map[string]any`に unmarshal しています
  - 入力が io.Reader であるときでも
- [github.com/mitchellh/mapstructure](https://github.com/mitchellh/mapstructure)の力を借りて、`map[string]any`から入力の struct にデータをコピーします。
  - 特にベンチを取ってるわけではないので断言はしませんが、json.Unmarshal をもう一度呼ぶよりは効率的だと予測します。

会社で書いたコードなのでうろ覚えですが、実際に使ったときは OpenAPI spec を [github.com/ghodss/yaml](https://github.com/ghodss/yaml)の`yaml.YAMLToJSON`で json byte に変換して schema 部分を json-pointer で指定して `MustCompile` を実行していました。実際には `MustCompile` も lazy に実行させていたような記憶もあります。

```go:こんなコードだった気がする.go
//go:embed oapi_spec.yaml
var yamlBin []byte

const schemaUrl = "mem://schema.json"

var jsonschemaCompiler = jsonschema.NewCompiler()

func init() {
	j, err := yaml.YAMLToJSON(yamlBin)
	if err != nil {
		panic(err)
	}
	err = jsonschemaCompiler.AddResource(schemaUrl, bytes.NewReader(j))
	if err != nil {
		panic(err)
	}
	jsonschemaCompiler.AssertContent = true
	jsonschemaCompiler.AssertFormat = true
}

func mustCompile(internalRef []string) *jsonschema.Schema {
	if len(internalRef) < 2 {
		panic("mustCompile: input internalRef must be len(s) > 1")
	} else if internalRef[0] != "#" {
		panic(fmt.Sprintf("mustCompile: first element of input internalRef must be # but %s", internalRef[0]))
	}
	escaped := make([]string, 0, len(internalRef)-1)
	for _, str := range internalRef[1:] {
		item := str
		// This is gonna be a json pointer string.
		// Below must be escaped into:
		//   - tilde(~) into ~0
		//   - forwardslash(/) into ~1
		// see https://datatracker.ietf.org/doc/html/rfc6901
		item = strings.ReplaceAll(item, "~", "~0")
		item = strings.ReplaceAll(item, "/", "~1")
		item = url.PathEscape(item)
		escaped = append(escaped, item)
	}
	return jsonschemaCompiler.MustCompile(schemaUrl + "#/" + strings.Join(escaped, "/"))
}
```

:::

## Partial JSON の受け側にはなれる

- 後のセクションでも触れますが、`json.Unmarshal`はキーがないとき単に何も代入しない動きをするため、「キーがあるときだけ上書き」の挙動自体は容易に実現可能です。
  - 上記の validation と合わせると堅牢な update 処理を行えます。
- JSON を`map[string]any`にデコードしてそれを上書きしてもいいでしょう。

```go
type Sample struct {
	Foo   string
	Bar   int
	Baz   float64
	Inner Inner
}

type Inner struct {
	Qux  string
	Quux string
}

// Update creates a updated Sample.
func (s Sample) Update(jsonBytes []byte) (Sample, error) {
	err := json.Unmarshal(jsonBytes, &s)
	if err != nil {
		return Sample{}, err
	}
	return s, nil
}

func main() {
	var s Sample

	s, _ = s.Update([]byte(`{"Foo":"foo","Bar":10,"Baz":10.15,"Inner":{"Qux":"qux","Quux":"quux"}}`))
	fmt.Printf("%+v\n", s)
	s, _ = s.Update([]byte(`{"Foo":"foo?","Inner":{"Quux":"q"}}`))
	fmt.Printf("%+v\n", s)
	s, _ = s.Update([]byte(`{"Bar":-20,"Baz":10.24}`))
	fmt.Printf("%+v\n", s)
}

/*
	{Foo:foo Bar:10 Baz:10.15 Inner:{Qux:qux Quux:quux}}
	{Foo:foo? Bar:10 Baz:10.15 Inner:{Qux:qux Quux:q}}
	{Foo:foo? Bar:-20 Baz:10.24 Inner:{Qux:qux Quux:q}}
*/
```

ネストした struct (今回の場合 Inner)がポインターだと `nil` 以外の時、新しいポインターを allocate しないので注意が必要です。

# 課題

Go の struct に JSON から変換/逆変換を行うときに struct 上では`undefined`と`null`と`T`が区別が付かないため

- 入力値の`required`や、`disallowNull`, `disallowUndefined`などの入力ルールを実現できない
  - 少なくとも`required`は API にリクエストを飛ばすクライアントの実装が typo で key 名を取り違えているときのチェックのために欲しい。
  - `map[string]any`へのデコードで実現できますが、最終的にデータのバインド先となる struct とフィールドと型が合わない場合でも一旦デコードを完了してしまうため非効率です。
    - この場合[DisallowUnknownFields](https://pkg.go.dev/encoding/json@go1.20#Decoder.DisallowUnknownFields)のようなことができないため非効率です。
- JSON を送信するとき、相手システムが `null` と `undefined` を分けて使う場合、だし分けるのに当該 struct 以上の追加のデータが必要であるため煩雑であること

ひとつ目の validation 周りの欲求はパフォーマンスにしか触れておらず、ベンチもとっていないので特に強く言えないですが、2 番目の欲求はちょっとリアルに困っているところです。

なので残った要求は

- 取り扱いやすい方法で`undefined | null | T`を struct field で表現する

ことです。

しかし後続セクションで述べる理由により、std 範疇では難しいことがわかります。

# Go の standard library における JSON "null", "undefined" の取り扱い

まず解決方法を考える前に Go の standard library で JSON を取り扱う`encoding/json`が`null`や`undefined`をどのように処理するのか確認します。

## 基本: 変換/逆変換

この記事をここまで読んでいて知らない人がいるかはわかりませんが、基礎情報として、変換/逆変換の方法について触れます。

Go では standard library の`encoding/json`によって JSON と Go の値との変換/逆変換を行います。

[playground](https://go.dev/play/p/oDZNB8D_ub6)

```go
type Sample struct {
	Foo string
	Bar int
}

func main() {
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
}
```

対象の type が`json.Marshaler`, `json.Unmarshaler`を実装している場合、そちらが優先して使われます。

[playground](https://go.dev/play/p/LnsWozze22p)

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

func main() {
	bin, err := json.Marshal(Sample2{Foo: "foo", Bar: 123})
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
}
```

stream でも処理が可能です。

[playground](https://go.dev/play/p/Gv15R4ZLkKW)

```go
type Sample struct {
	Foo string
	Bar int
}

func main() {
	buf := new(bytes.Buffer)
	encoder := json.NewEncoder(buf)

	err := encoder.Encode(Sample{Foo: "foo", Bar: 123})
	if err != nil {
		panic(err)
	}
	fmt.Printf("%s", buf.String()) // {Foo:foo Bar:123}

	decoder := json.NewDecoder(buf)

	var s Sample
	err = decoder.Decode(&s)
	if err != nil {
		panic(err)
	}
	fmt.Printf("%+v\n", s) // {"Foo":"foo","Bar":123}
}
```

## JSON null

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
>
> https://pkg.go.dev/encoding/json@go1.20.0

逆に json.Unmarshal では

- non pointer type `T`に対して `null` は代入しない
- pointer type `*T`に対して `null` は `nil` を代入

という挙動と記述されています。

type が`json.Unmarshaler`を実装する場合、JSON byte が`null`であるときでも呼び出されるため、`[]byte("null")`を好きに変換することができます。

## JSON undefined

JSON(JavaScript Object Notation)はその名前の通り JavaScript が元となっているため、JavaScript の事情を多分に含んでいます。

JavaScript には`null`とは別に`undefined`という「データがない状態」の表現が存在します。
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

## `encoding/json`では"undefined | null | T"のだし分けできない

前記の通り、型のデータを`undefined`(=フィールドをスキップする)とするか`null`とするかはデータではなくメタデータとして実装されています。

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

# 解決方法: "undefined | null | T"を表現できる type を作る

"go JSON undefined"などでググってみましたがこれを実現しているライブラリーは見つかりませんでした。おそらくはある気がしますが、気が向いたしすぐ作れる気がするので作ってみます。

- `undefined | null | T`を表現できる構造体を定義することとします。
  - `**T`のようなポインタータイプを base type とする defined type はメソッドを持てませんので struct にします。
  - struct は omiempty によるスキップがぜったいに起こりませんので専用の marshaller が必要です。
    - `",string"`オプションをサポートすると unmarshaller も必要になってしまったので作ります

## type を作る

### 問題: \*\*T はメソッドを持てない

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
  - `interface { IsUndefined() bool }`のような interface に対する type assertion を行うこともできません。
  - reflect を使う方法は取れると思いますが、[reflect.ValueOf は(現状は)かならず heap にデータを escape する](https://cs.opensource.google/go/go/+/refs/tags/go1.20:src/reflect/value.go;l=3144-3147)ので避けられるなら避けたほうがいいでしょう。

`encoding/json`も`reflect`を多用しますのでこれを機にせずに使えるようなアプリならば気になるのはメソッドが定義できない煩雑さの方でしょう。

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

func (o Option[T]) Value() T {
	return o.v
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

`Nullable[T]`は`Option[T]`とほぼ等価ですが、以下の定義はできません

- `type Nullable[T] = Option[T]`
  - type param のある type alias は spec で[定義されていない](https://go.dev/ref/spec#AliasDecl)
- `type Nullable[T] Option[T]`
  - Defined type は method set を[継承しない](https://go.dev/ref/spec#Type_definitions)ため。

そのため embedded で行きます。

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

func (f Undefinedable[T]) Value() T {
	return f.v.Value()
}
```

## 特化した Marshaller を作る

上記の`Undefinedable[T]`は struct であるので、前記の通り omitempty によるスキップ動作が起きません。そこで、専用の Marshaller を作成して`undefined`時に skip 可能にします。

### ナイーブな発想版

単純な発想によれば、reflect によって field の値を読み取りながら、上記の Undefinedable[T]の場合かつ Undefined の時だけ field をスキップする挙動で目的は達せます。

もちろん`Undefinedable[T]`と`Nullable[T]`に MarshalJSON と UnmarshalJSON の実装が必要ですが、自明であるので省略します。

`encoding/json`は内部で利用している key などを JSON string の escape を行う機能を外部に公開しないため、[github.com/mailru/easyjson](https://github.com/mailru/easyjson)の`jwriter`に任せてしまいます。

```go marshal_fields_json.go
func getFieldName(f reflect.StructField) string {
	return f.Name // 実際にはf.Tag.Get("json")でフィールド名も見る
}

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

### json.Marshal と同じ struct tag をサポートする

#### json.Marshal の struct tag

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

#### 実装する

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
+		quote := shouldQuote(field.Type(), field.Interface()) && string(marshalled) != "null"
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

## 特化した Unmarshaller を作る

`Undefinable[T]`、`Nullable[T]`が\`json:",string"\`を無視していれば`json.Unmarshal`でデコード可能だったのですが、考慮するようにしたので`json.Unmarshal`に対応する`UnmarshalFieldsJSON`が必要になってしまいました。

JSON バイト列の解析と iterate は [github.com/buger/jsonparser](https://github.com/buger/jsonparser) にすべて任せます。

```go
func UnmarshalFieldsJSON(data []byte, v any) error {
	rv := reflect.ValueOf(v)

	if rv.Kind() != reflect.Pointer || rv.IsNil() {
		return &json.InvalidUnmarshalError{Type: rv.Type()}
	}

	return unmarshalFieldsJSON(data, rv)
}

type fieldInfo struct {
	ty          reflect.StructField
	layoutIndex int  // index that can be used to retrieve type info with rv.Field(layoutIndex).
	anonymous   bool // embedded.
	tagged      bool // tagged with field name.
	quote       bool // field has ,string option.
}

func unmarshalFieldsJSON(data []byte, rv reflect.Value) error {
	rv = reflect.Indirect(rv)
	rt := rv.Type()

	if rt.Kind() != reflect.Struct {
		return fmt.Errorf("incorrect type. want = struct, but is %s", rt.Kind())
	}

	// Collect type info ahead of time
	// to avoid doing rv.FieldByName() each time it iterates over JSON byte slice.
	fieldNames := make(map[string]fieldInfo, rv.NumField())
	for i := 0; i < rv.NumField(); i++ {
		field := rt.Field(i)
		fieldName, opts, tagged, shouldSkip := GetFieldName(field)

		if shouldSkip {
			continue
		}

		// Untagged embedded field. Must go into recursively.
		if !tagged && field.Anonymous && field.Type.Kind() == reflect.Struct {
			err := unmarshalFieldsJSON(data, rv.Field(i))
			if err != nil {
				return err
			}
		}

		fieldNames[fieldName] = fieldInfo{
			ty:          field,
			layoutIndex: i,
			anonymous:   field.Anonymous,
			tagged:      tagged,
			quote:       OptContain(opts, "string") || OptContain(field.Tag.Get("und"), "string"),
		}
	}

	return jsonparser.ObjectEach(
		data,
		func(key, value []byte, dataType jsonparser.ValueType, offset int) error {
			info, ok := fieldNames[string(key)]
			if !ok {
				return nil
			}

			if dataType == jsonparser.String {
				// jsonparser trims wrapping double quotations. Get those back here.
				value = data[offset-len(value)-2 : offset]
			}

			if info.quote && string(value) != "null" {
				if value[0] != '"' || value[len(value)-1] != '"' {
					// mimicking json.Unmarshal behavior.
					return fmt.Errorf(
						"broken quotation. field ( %s tagged as %s ) is tagged with string option"+
							" but input value is neither 'null'"+
							" nor wrapped with double quotations. value = %s",
						info.ty.Name, string(key), string(value),
					)
				}
				value = value[1 : len(value)-1]
			}

			frv := rv.Field(info.layoutIndex)

			v := frv
			if v.Kind() != reflect.Pointer && v.Type().Name() != "" && v.CanAddr() {
				// adder this value so that we can find method of *T, not only ones for T.
				v = v.Addr()
			}

			if info.tagged && info.anonymous && frv.Type().Kind() == reflect.Struct {
				// tagged embedded field.
				err := unmarshalFieldsJSON(value, v)
				if err != nil {
					return err
				}
			}

			if decoder, ok := v.Interface().(json.Unmarshaler); ok {
				err := decoder.UnmarshalJSON(value)
				if err != nil {
					return err
				}
			} else {
				internalValue := v.Interface()
				err := json.Unmarshal(value, internalValue)
				if err != nil {
					return err
				}
			}

			return nil
		},
	)
}
```

上記のような感じです。[実際の実装ではもうちょっと凝ったことをしています。](https://github.com/ngicks/undefinedablejson/blob/main/serde.go)

## `encoding/json`の効率化をまねる

`unmarshalFieldsJSON`の fieldInfo の収集部分は、同じ reflect.Type に対して全く同じ処理をしますのでコンパイラが最適化をかけてくれそうな気もしますが、`reflect`が含まれているのでどうなるんだろう、と思いながら`encoding/json`を読み進めるとフィールドをどう処理するかっていう処理を[sync.Map でキャッシュしていますね。](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/encode.go;l=370;drc=d5de62df152baf4de6e9fe81933319b86fd95ae4;bpv=1)`reflect`であるから最適化が掛からないかはわからないですが、似たようなことをしている都合上、似たような対策が似たような効果を及ぼすと期待できます。

このキャッシュの部分は`sync.WaitGroup`を使って同期させる部分が存在していて、これは recursive な type 用とコメントがありますが、おそらく非同期的に同時に呼び出されてもいいような対策でもあると思います。[ここも真似っこで実装しておきました。](https://github.com/ngicks/undefinedablejson/blob/699a7f4463b59597ade11fbe700df99c415e74d4/serde.go#L131-L160)

また、struct type への string option は staticcheck `SA5008`に警告されてしまうので、warning 回避のために`und:"string"`でも`json:",string"`と同様の判定を行うように変更しました。

# 効果

- Elasticsearch の前に立って、update 用の partial document を受け取って validation をかけたり加工したり素通ししたりする API が自然に書けるようになる(はず)
- Elasticsearch の\_source フィールドで Elasticsearch から得られるドキュメントのフィールド名を絞った際、自然に指定しなかったフィールドが`undefined`なままで表現できるようになるはず
  - ただしこれは document が null 埋めされてるとか初期値がついている前提。

Elasticsearch は実のところあらゆるフィールドの値が `(T | null) | (T | null)[]`と定義されているので、もう少し努力が必要です。これは別の機会に実装しようと思っています。

# おわりに

いかかがでしたか？私はこの実装や調査を非常に楽しみました。最初は数時間で実装がすむ程度だと思って始めたのですが、思った以上に`encoding/json`の挙動の奥が深くて(考えれば当たり前ですが)、数日を消費されました。

ここまで書いておいてなんですが、本当にこの方法が最良なのでしょうか？他に方法があったり、他に誰かが似たようなことをすでにしている気がするんです。筆者が記事執筆前に軽く調べた限りそれらしい情報は出てこなかったですし、作ってみたかったので作ってみただけなので別にあったとしてもいいのですが。

今後の拡張としては

- code generator の追加
  - `reflect`の使用を避けるため。
- `required`, `disallowNull`などのバリデーションルールを追加し、Unmarshal 時に validate する
- `Option[T]`に`Map()`のような Rust の`Option<T>`風関数を追加して変更を容易にする。
- 似たようなことをしている package を探して、**あった場合このパッケージ破棄し、それに貢献する**。

実はこのライブラリ自体は Elasticsearch の JSON をいい感じに扱うために作っていたライブラリを一部切り出したものなので、筆者が Elasticsearch とやり取りする Go アプリを書くまで本格使用することがないため、問題点が明らかにならない可能性が高いです。
