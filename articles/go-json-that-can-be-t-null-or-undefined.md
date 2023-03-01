---
title: "Goのstruct fieldでJSONのundefinedとnullを表現する"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

# TL;DR

- Elasticsearch (の update API)のような JSON における`null`と`undefined`(JSON に key がない)状態をうまく使い分けるシステムに送る JSON を struct を marshal するだけでいい感じに作りたい。
- std の`encoding/json`でうまいことやるのは無理そうだった。
- `Option[T]`を定義して、`Option[Option[T]]`を`undefined | null | T`を表現する型とした。
- [jsoniter](https://github.com/json-iterator/go)の Extension を駆使して`undefined`のとき field を skip できるようにした。

# Overview

Go で JSON を扱うとき、Elasticsearch の update api に渡す JSON のような `null` と `undefined` をだし分けられるデータ構造や、それに対応する JSON marshaller が Go には std ではなく、軽く探したところ見つからなかったので、興味本位で作ってみました。

成果物はこちらです。

https://github.com/ngicks/und

この記事では

- どういう事で困っていたのか
- 既存の方法にはどういうものがあったのか
  - この話題に関連する今もってる知見をできる限り書いています。
- 実装を通じて`encoding/json`について得られた知見

などを書いていきます。

# 前提知識

- [Go programming language](https://go.dev/) の細かい説明はしてますが全体の説明はしないので、ある程度知っている人じゃないと意味が分からないかもしれません。
- 本投稿のごく一部で何の説明もなしに TypeScript の型表記がでてきます。知っている人か、でなければなんとなくで読んでください。

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

この zero value は、「データがない状態」の判定に使われることがあり、

例えば[GORM](https://gorm.io/) という ORM の場合,

```go
type User struct {
  ID           uint
  Name         string
  Email        *string
  Age          uint8
  Birthday     *time.Time
  MemberNumber sql.NullString
  ActivatedAt  sql.NullTime
  CreatedAt    time.Time
  UpdatedAt    time.Time
}

// Struct
db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
// SELECT \* FROM users WHERE name = "jinzhu" AND age = 20 ORDER BY id LIMIT 1;
```

https://gorm.io/docs/query.html より引用

という風に、zero value であるフィールドはデータがセットされていないこととして扱う API が存在します。該当の zero value 判定は[ここなどでおこなわれています](https://github.com/go-gorm/gorm/blob/02b7e26f6b5dcdc49797cc44c26a255a69f3aff3/schema/field.go#L462)(`reflect`の [IsZero](https://pkg.go.dev/reflect@go1.20#Value.IsZero) などが使われていますね)

### 外部データ(JSON)をバインドするときの zero value

同じように、JSON など外部データからのデコード時、struct にデータをバインドする際に、

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

を入力した場合、バインドされた struct field の値はいずれの場合も zero value でありますので、入力がなんであったかという区別がつきません。

外部 API として`""`(空白 string) や数値型で `0` があり得ない場合のみ、上記の `Sample` struct にデータをバインドするだけでいいことになります。

`""`がありえない API はそこまで珍しくない気がしますが、 `0` がありえないケースはそこそこ珍しいかなと思います。そこで基本的に以下のようにメンバーをポインターにすることになるのが普通かと思います。

```go
type Sample struct {
	Foo *string
	Bar *int
}
```

この場合、入力値が「データがない状態」を示すとき、対応する field の値は`nil`になります。しかしこの場合でも、フィールドが`null`であった時と、`undefined`(JSON に key がなかった)時の区別がつきません。

## 外部システムとやり取りする時の`undefined`と`null`

### `undefined`と`null`を分けて扱われることがある

HTTP で JSON を送る PUT や PATCH method において、`undefined`(=キーが存在しない)のとき field をスキップ、`null`のとき field をクリアするか`null`で上書き、`T`の時`T`で上書き、という挙動をさせる API があります。筆者もそういった API を書くことがあります。

広く使われている実例としては、[Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro.html) があります。Elasticsearch の update api では [partial document を送ること](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update.html#_update_part_of_a_document)でドキュメントの各 field を更新でき、`null` をセットすることで field を `null` で上書きできます。

## 既存の方法: required の強制、validation について

### map[string]any

もっとも単純な発想は`map[string]any`に JSON のあらゆる値をいったんデコードして、key の存在チェックや型の整合性などを取ることです。

Go のハッシュマップは`map[T]U`の記法で表現されます。`T`が key、`U`が value の型です。

`any`は Go1.18 で追加された`interface{}`のエイリアスです。Go の interface は [MethodElem](https://go.dev/ref/spec#MethodElem) の集合であり、それを実装するあらゆる型が代入可能です。`any`は何のメソッドも指定されていないためどのような値でも代入可能です。

JSON は TypeScript でいうところの

```ts
type JSONLit = string | number | boolean | null | JSONArray | JSONObject;

interface JSONArray extends Array<JSONLit> {}

interface JSONObject {
  [key: string]: JSONLit;
}
```

であるので厳密には`map[string]any`ではないんですが、Array 以外の primitive 値を送ることなんかめったにないと思いますし、API の後方互換性を保ったまま情報を増やすのがもっとも容易なのは結局 Object なのでそれ以外の場合の考慮はいったん外しましょう。

`map[string]any`にデコードすれば key のあるなし、型の一致などの判定をおこなえます。

### JSON schema

JSON のバイト列が意識される段階では validation そのものは JSON schema などで行うことができます。

JSON schema を読み込んで validation をかける事ができるライブラリはいくつかあります。

- https://github.com/xeipuuv/gojsonschema
- https://github.com/santhosh-tekuri/jsonschema
- https://github.com/qri-io/jsonschema

筆者は`github.com/santhosh-tekuri/jsonschema`を [echo](https://echo.labstack.com/) の Binder の実装の中で使って validation をかけるようなことしたことがあります。

実装のサンプルは長くなるのでドロップダウンに隠しておきます。

:::details json schema で validation をおこなう echo.Binder の実装サンプル

サンプルで`echo.Context`をくみ上げる気が起きなかったので、その部分は単に compilation error が起きないことだけ見せています。

MustCompile は[JSON Pointer](https://www.rfc-editor.org/rfc/rfc6901)を受け付ける仕様なので OpenAPI spec の yaml でも JSON byte に変換できればコンパイル可能です。

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

func (*SampleValidator) Validate(data any) error {
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

	var jsonVal any
	if err := dec.Decode(&jsonVal); err != nil {
		return err
	}

	// 省略:
	// token, err := dec.Token()
	// // input stream has additional chars and it is not a valid json token.
	// if err != nil { /*どうする？*/ };
	// // input stream has another json.
	// if token != nil { /*どうする？*/ }

	if validator, ok := any(i).(interface {
		Validate(data any) error
	}); ok {
		err := validator.Validate(jsonVal)
		if err != nil {
			return err
		}
	}

	err := mapstructure.Decode(jsonVal, i)
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

	sv = SampleValidator{}
	err = bindValidating(strings.NewReader(`null`), &sv)
	fmt.Printf("%+v\n", err) // jsonschema: '' does not validate with file:///<path to cwd>/foo.json#/type: expected object, but got null
	fmt.Printf("%+v\n", sv)  // {Foo:}
}
```

`null` リテラルもしっかりチェックしてくれますね！

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

struct field がポインターである場合、`nil`の場合のみ新しいポインターを allocate するので今回のように immutable な実装にしたい時はポインターを使わない方がいいでしょう。

# 課題

Go の struct に JSON からエンコード/デコードを行うときに struct 上では`undefined`と`null`と`T`が区別が付かないため

- 入力値の`required`や、`disallowNull`, `disallowUndefined`などの入力ルールを実現できない
  - 少なくとも`required`は API にリクエストを飛ばすクライアントの実装が typo で key 名を取り違えているときのチェックのために欲しい。
    - [DisallowUnknownFields](https://pkg.go.dev/encoding/json@go1.20#Decoder.DisallowUnknownFields)によって余分なキーを許可しないことで対応可能ではあります。
  - `map[string]any`へのデコードで実現できますが、最終的にデータのバインド先となる struct とフィールドと型が合わない場合でも一旦デコードを完了してしまうため非効率です。
    - この場合 `DisallowUnknownFields` のようなことができないため非効率です。
- JSON を送信するとき、相手システムが `null` と `undefined` を分けて使う場合、だし分けるのに当該 struct 以上の追加のデータが必要であるため煩雑であること

ひとつ目の validation 周りの欲求はパフォーマンスにしか触れておらず、ベンチもとっていないので特に強く言えないですが、2 番目の欲求はちょっとリアルに困っているところです。

なので残った要求は

- 取り扱いやすい方法で`undefined | null | T`を struct field で表現する

ことです。

しかし後続セクションで述べる理由により、std 範疇では難しいことがわかります。

# Go の standard library における JSON "null", "undefined" の取り扱い

まず解決方法を考える前に Go の standard library で JSON を取り扱う`encoding/json`が`null`や`undefined`をどのように処理するのか確認します。

## 基本: エンコード/デコード

この記事をここまで読んでいて知らない人がいるかはわかりませんが、基礎情報として、エンコード/デコードの方法について触れます。

Go で標準的な方法で JSON を取り扱うには standard library の`encoding/json`を使います。`json.Marshal`によってエンコード、`json.Unmarshal`によってデコードを行います。

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

stream でも処理が可能です。Decoder は[最低 512 バイト](https://cs.opensource.google/go/go/+/refs/tags/go1.20:src/encoding/json/stream.go;l=157;drc=169203f3ee022abf66647abc99fd483fd10f9a54)ずつ読みこみながら[JSON literal が終わるまで reader を読む](https://cs.opensource.google/go/go/+/refs/tags/go1.20:src/encoding/json/stream.go;l=62-63;drc=169203f3ee022abf66647abc99fd483fd10f9a54)ようですね。

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

:::details 若干厄介な null リテラルの取り扱い

`null` が入力値であり対象が non-pointer type であるとき単に代入しない動きであるので `null` リテラルが入力であればデコード自体が完全にスキップされ、**エラーになりません**。

[playground](https://go.dev/play/p/ywwNGq-LBYz)

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Sample struct {
	Foo string
	Bar int
}

func main() {
	var s Sample
	err := json.Unmarshal([]byte(`null`), &s)
	if err != nil {
		panic(err)
	}
	fmt.Printf("%+v\n", s) // {Foo: Bar:0}
}
```

`var jsonVal any`にデコードするか、でなければ `null` リテラルはデコード前に判定する必要があります。
:::

## JSON undefined

JSON(JavaScript Object Notation)はその名前の通り JavaScript が元となっているため、JavaScript の事情を多分に含んでいます。

JavaScript には`null`とは別に`undefined`という「データがない状態」の表現が存在します。
[JSON の定義](https://datatracker.ietf.org/doc/html/rfc8259)上`undefined`は存在しないのでシリアライズされるときにキーが消える挙動となります。

`json.Marshal`では、omitempty という struct tag で当該 struct field のスキップを行います。

> The "omitempty" option specifies that the field should be omitted from the encoding if the field has an empty value, defined as false, 0, a nil pointer, a nil interface value, and any empty array, slice, map, or string.
>
> https://pkg.go.dev/encoding/json@go1.20.0

- 設定時 [Marshal でフィールドがスキップされるような処理になります](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/encode.go;l=748;drc=8c17505da792755ea59711fc8349547a4f24b5c5;bpv=1;bpt=1)
- [empty であるかの条件はここで網羅されています](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/encode.go;drc=d5de62df152baf4de6e9fe81933319b86fd95ae4;l=339)
  - 条件の通り struct は決して empty と判定されることはありません。
- [この記述](https://cs.opensource.google/go/go/+/refs/tags/go1.20.0:src/encoding/json/indent.go;l=17;drc=8c17505da792755ea59711fc8349547a4f24b5c5)からわかるように、`MarshalJSON`メソッドで返すことが許されるのは、有効な JSON 文字列のみです。
  - つまりここで empty な値を返して、フィールドをスキップしてもらうようなことはできません。

`json.Unmarshal`では、

- JSON バイト列の中に対応する key がない場合、単に代入しない挙動となります。
  - zero value のまま置かれると思ってもいいでしょう。

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

# 関連 issue

- https://github.com/golang/go/issues/5901
- https://github.com/golang/go/issues/11939

似たような悩みに基づく issue が出ています。色々理由があって`encoding/json`に入ることはないようですが

https://github.com/golang/go/issues/5901#issuecomment-907696904

にある通り、`v2`の`encoding/json`に値に基づいてフィールドをスキップするような挙動が追加されるかもしれません。

とにかくしばらくはなさそうです。

# 解決方法: "undefined | null | T"を表現できる type を作る

"go JSON undefined"などでググってみましたがこれを実現しているライブラリーは見つかりませんでした。おそらくはある気がしますが、気が向いたしすぐ作れる気がするので作ってみます。

- `undefined | null | T`を表現できる構造体を定義することとします。
  - `**T`のようなポインタータイプを base type とする defined type はメソッドを持てませんので struct にします。
  - struct は omiempty によるスキップがぜったいに起こりませんので専用の marshaller が必要です。

## type を作る

### 問題: \*\*T はメソッドを持てない

Go では「データがない状態」を表現することに通常は`*T` を利用します。

`*T`が`undefined | T`もしくは`null | T`を表現するならば、単純に`**T`で`undefined | null | T`を表現できます。

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

`encoding/json`も`reflect`を多用しますのでこれを気にせずに使えるようなアプリならば気になるのはメソッドが定義できない煩雑さの方でしょう。

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

### 考慮すべきエッジケース

特化したものを作るのですが、特化の範囲はあくまで入力が struct のみと想定するのみです。そこ以外はある程度`encoding/json`の挙動に寄せないと呼び出し側に不要な気遣いを生じさせ、後々困る可能性があります。

`json.Marshal`は struct tag によってフィールド名が指定できる都合上、被ったフィールドをまとめて全部消す動作になっています。この辺のエッジケースがどういう風に動作するかをまず確認しましょう。

```go
type OverlappingKey1 struct {
	Foo string
	Bar string `json:"Baz"`
	Baz string
}
// OverlappingKey1{Foo: "foo", Bar: "bar", Baz: "baz"},
// ↓
// {"Foo":"foo","Baz":"bar"}
// tagが優先

type OverlappingKey2 struct {
	Foo string
	Bar string `json:"Bar"`
	Baz string `json:"Bar"`
}
// OverlappingKey2{Foo: "foo", Bar: "bar", Baz: "baz"}
// ↓
// {"Foo":"foo"}
// 同名のtagはどちらも削除

type OverlappingKey3 struct {
	Foo string
	Bar string `json:"Baz"`
	Baz string
	Qux string `json:"Baz"`
}
// OverlappingKey3{Foo: "foo", Bar: "bar", Baz: "baz", Qux: "qux"}
// ↓
// {"Foo":"foo"}
// tag名で被り+元のstruct field名で被りの場合でも全部まとめて消されますね。

type Sub1 struct {
	Foo string
	Bar string `json:"Bar"`
}

type OverlappingKey4 struct {
	Foo string
	Bar string
	Baz string
	Sub1
}
// OverlappingKey4{Foo: "foo", Bar: "bar", Baz: "baz", Sub1: Sub1{Foo: "foofoo", Bar: "barbar"}}
// ↓
// {"Foo":"foo","Bar":"bar","Baz":"baz"}
// Embeddedの場合、上の階層にあるほうが優先。

type Recursive1 struct {
	R string `json:"r"`
	Recursive2
}

type Recursive2 struct {
	R  string `json:"r"`
	RR string `json:"rr"`
	*OverlappingKey5
}

type OverlappingKey5 struct {
	Foo string
	Recursive1
}
// OverlappingKey5{Foo: "foo", Recursive1: Recursive1{R: "r", Recursive2: Recursive2{R: "r2", RR: "rr"}}},
// ↓
// {"Foo":"foo","r":"r","rr":"rr"}
// 型の再帰が起きた時、1周まではエンコードされるがその後は無視される挙動のようですね。
```

`encoding/json`のソースコードをよくよく読んでると優先ルールは[ここで記述されています](https://cs.opensource.google/go/go/+/refs/tags/go1.20:src/encoding/json/encode.go;l=1333-1388;bpv=1)ね。index は struct 中での定義順のことで、len(index) > 1 の時 embed された struct であることがわかります。なので、優先ルールは、

- 階層の浅さ(embed されているものが優先されない)
- tag されているか
- 定義順

で決められており、同名のフィールドが複数ある場合は`len(fields[0].index) == len(fields[1].index) && fields[0].tag == fields[1].tag`の場合、その名前は消す、という挙動ですね。

再帰に関しては[ここや](https://cs.opensource.google/go/go/+/refs/tags/go1.20:src/encoding/json/encode.go;l=384-399;bpv=1)、[ここなど](https://cs.opensource.google/go/go/+/refs/tags/go1.20:src/encoding/json/encode.go;l=1234-1237;bpv=1)合わせ技でなっているのだと思われます。

std はさすが、あらゆるエッジケースが考慮されていますね。

### jsoniter の Extension で何とかする

さすがに上記のエッジケースを埋めるものを個人で完全にメンテし続ける自信がなくなってきたので、なんとか他の方法でできないか探します。

`encoding/json`は部分的なロジックを取り出せるつくりにはなっておらず(不用意に露出させれば変更の自由が損なわれるのだから当然です。)、サードパーティのライブラリを当てにするしかありません。

幸いにも json 関連のライブラリは数多く存在しており、[github.com/json-iterator/go](https://github.com/json-iterator/go)(以後 jsoniter と呼びます)が内部の挙動を Extension の仕組みによってカスタマイズ可能かつ interface 的には`encoding/json`と互換なようですのでこちらを使うことにします。

jsoniter は[ValEncoder](https://pkg.go.dev/github.com/json-iterator/go#ValEncoder)という interface で型に対する encoder を定義しています。この interface は IsEmpty という今回使いたいドンピシャの機能を露出していますのでこれを利用します。

```go
var config = jsoniter.Config{
	EscapeHTML:             true,
	SortMapKeys:            true,
	ValidateJsonRawMessage: true,
}.Froze()

func init() {
	config.RegisterExtension(&UndefinedableExtension{})
}

type IsUndefineder interface {
	IsUndefined() bool
}

var undefinedableTy = reflect2.TypeOfPtr((*IsUndefineder)(nil)).Elem()

// undefinedableEncoder fakes Encoder so that
// the undefined Undefinedable fields are considered to be empty.
type undefinedableEncoder struct {
	ty  reflect2.Type
	org jsoniter.ValEncoder
}

func (e undefinedableEncoder) IsEmpty(ptr unsafe.Pointer) bool {
	val := e.ty.UnsafeIndirect(ptr)
	return val.(IsUndefineder).IsUndefined()
}

func (e undefinedableEncoder) Encode(ptr unsafe.Pointer, stream *jsoniter.Stream) {
	e.org.Encode(ptr, stream)
}

// UndefinedableExtension is the extension for jsoniter.API.
// This forces jsoniter.API to skip undefined Undefinedable[T] when marshalling.
type UndefinedableExtension struct {
}

func (extension *UndefinedableExtension) UpdateStructDescriptor(structDescriptor *jsoniter.StructDescriptor) {
	if structDescriptor.Type.Implements(undefinedableTy) {
		return
	}

	for _, binding := range structDescriptor.Fields {
		if binding.Field.Type().Implements(undefinedableTy) {
			enc := binding.Encoder
			binding.Encoder = undefinedableEncoder{ty: binding.Field.Type(), org: enc}
		}
	}
}

// ... rest of interface ...
```

jsoniter.Register...などは string で type 名を指定して Encoder を登録できますが、この type 名は`reflect2.Type#String`によってランタイムで判別されます。Generics である場合、例えば`Undefinedable[T]`は`T`ごとに別々の type 名を持つことになります。ここで煩雑なコードによってありあえる`T`ごとに Encoder を登録するよりも`interface{ IsUndefined() bool }`を実装するものに対してまとめて適用したほうが楽なのでここではそうしています。

さて、IsEmpty を内部的に IsUndefined に差し替えることができました。ただ、これだけではフィールドに`omitempty`を設定する必要があるため、まだ目標に届きません。

ここで、reflect2.StructField が interface であることに気付きました。つまり、

```diff go
+ // fakingTagField implements reflect2.StructField interface,
+ // faking the struct tag to pretend it is always tagged with ,omitempty option.
+ type fakingTagField struct {
+ 	reflect2.StructField
+ }
+
+ func (f fakingTagField) Tag() reflect.StructTag {
+ 	t := f.StructField.Tag()
+ 	if jsonTag, ok := t.Lookup("json"); !ok {
+ 		return reflect.StructTag(`json:",omitempty"`)
+ 	} else {
+ 		splitted := strings.Split(jsonTag, ",")
+ 		hasOmitempty := false
+ 		for _, opt := range splitted {
+ 			if opt == "omitempty" {
+ 				hasOmitempty = true
+ 				break
+ 			}
+ 		}
+
+ 		if !hasOmitempty {
+ 			return reflect.StructTag(`json:"` + strings.Join(splitted, ",") + `,omitempty"`)
+ 		}
+ 	}
+
+ 	return t
+ }
+
// UndefinedableExtension is the extension for jsoniter.API.
// This forces jsoniter.API to skip undefined Undefinedable[T] when marshalling.
type UndefinedableExtension struct {
}

func (extension *UndefinedableExtension) UpdateStructDescriptor(structDescriptor *jsoniter.StructDescriptor) {
	if structDescriptor.Type.Implements(undefinedableTy) {
		return
	}

	for _, binding := range structDescriptor.Fields {
		if binding.Field.Type().Implements(undefinedableTy) {
			enc := binding.Encoder
+			binding.Field = fakingTagField{binding.Field}
			binding.Encoder = undefinedableEncoder{ty: binding.Field.Type(), org: enc}
		}
	}
}
```

という感じで、`Undefinedable[T]`の時だけ常に omitempty タグがあるかのようにふるまうようにします。少々ハッキーですがこれで思った通りの挙動をになります。

:::details jsoniter も encoding/json と動作が一致しないという話。

上記のエッジケースを入力すると jsoniter も encoding/json と一致しない結果を返しました。よりにもよって一番考慮されないエッジケースを選んでしまったようです。

さらに、`,string`オプションが`null`や struct など対応していない型も quote していまいます。

issue にもしておきました。

https://github.com/json-iterator/go/issues/657

`,string`オプションはあまりにも挙動が違うのでサポートしたほうがいいかもですが、他は考慮するほどでもないですかね？

https://github.com/json-iterator/go/pull/659
https://github.com/json-iterator/go/pull/660

これらの問題を修正する PR を出しておきましたが、非活発的なようなのでほっとかれるかもしれません。
`unsafe.Pointer`は今まで使ったことなかったんですがおかげでちょっと慣れちゃいました。

:::

## 実装

https://github.com/ngicks/und

色々綺麗にしてパッケージとして公開しておきました。これで会社でもコードを使えるというワケです。

実際にはもう少し色々綺麗にしていて、

- `nullable.Null[T]()`と`undefinedable.Null[T]()`のような感じで同名関数を持てるようにパッケージを分割
- StructTag のパージングを本格的なものにして、json 以外のタグを全く変更しないように
- NewEncoder/NewDecoder も API として公開するように

など、実用に耐えなくもないラインを目指してます。

# 効果

- `JSON` の `undefined` と `null` をうまく使い分けるシステム(例えば、Elasticsearch) の前に立って、update 用の partial document を受け取って validation をかけたり加工したり素通ししたりする API が自然に書けるようになる(はず)
- Elasticsearch の\_source フィールドで Elasticsearch から得られるドキュメントのフィールド名を絞った際、自然に指定しなかったフィールドが`undefined`なままで表現できるようになるはず
  - ただしこれは document が null 埋めされてるとか初期値がついている前提。

Elasticsearch は実のところあらゆるフィールドの値が `undefined | (T | null) | (T | null)[]`と定義されているので、もう少し努力が必要です。これは別の機会に実装しようと思っています。

# おわりに

いかかがでしたか？私はこの実装や調査を非常に楽しみました。
想像よりも数段`encoding/json`の挙動が奥が深く、周辺ライブラリの多さや調査項目の多さに結構な時間を持っていかれました。なんだか結果的に jsoniter の便利さを伝えるだけの記事になってしまったような気がします。

似たようなことをしている人はいると思うんですが、今回の実装はほとんど外部ライブラリ頼みでミニマルにできたので自分で作ったものでも結構使えるものになったと思います。

今後の課題は

- 今回の成果物を Elasticsearch を相手にするシステムで使ってみて改善する。
- jsoniter の Extension 部分をもうちょっと一般的にして使いやすくする
  - `time.Time` で `t.IsZero() == true`のとき empty 扱いするなども同様にできますので、そういった extension を作ってもいいでしょう。
- 普通はこういうお困りごとはどうやって解決されているのかを調べる。

などでしょうか。

以上です。ありがとうございました。
