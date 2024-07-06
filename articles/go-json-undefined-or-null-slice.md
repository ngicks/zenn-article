---
title: "GoのT | null | undefinedは[]Option[T]でよかった"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## GoのT | null | undefinedは[]Option[T]でよかった

- [github.com/oapi-codegen/nullable]が`map[bool]T`をベースに`encoding/json`にオミットされることが可能な`T | null | undefined`な型を定義していた
  - ([以前の記事]時点では全然思いついてなかった)
- `type Option[T any] struct{valid bool; v T}`を定義して`type undefinedable []Option[T]`としたほうがパフォーマンスが出るんじゃないかと思って試した。
  - ほんのちょっぴりパフォーマンスがよかった。

## はじめに

`JSON`を相互に送りあうシステムではたびたび`T`(フィールドにある型の値がある, _defined_, _specified_, _present_)、`null`(`null`がフィールドにセットされている)、`undefined`(フィールドが存在しない, _undefined_, _unspecified_, _absent_)を使い分けることがあります。これは`Go`のstdが敷く「structとバイト列と相互変換する」というデータ変換の様式の中で表現するのが難しく、特別な努力を要していました。

[以前の記事]で似たようなテーマについていろいろ述べて解決法までを示しました。それから1年ほどたって色々知見が筆者の脳内で整理できたり、そもそも大がかりなこと(エンコーダーの用意とか)をしなくても`[]Option[T]`を利用すれば`encoding/json`にオミットされることが可能かつ`T | null | undefined`を表現できる型を定義できることに気付きました。

この記事は[以前の記事]の置き換えを意図しており、それをobsoleteにするものとして書いています。そのためこの記事だけを読めばいいだけになるようにします。
この目的から[以前の記事]とこの記事は大部分が重複し、一部を追加し後半の大分部分を削除するような記事になります。
(ただし前半部分も文章をリファインして`xml`の話を含めるようにしたなどのアップデートをしてあります)
必要に応じて読者には記事をスキップしてほしいと思います。

## Overview(TL;DR)

- [Elasticsearch](の update API)のような`JSON`における`null`と`undefined`(`JSON`にフィールドがない)状態をうまく使い分けるシステムに送る`JSON`を structをmarshalするだけでいい感じに作りたい。
- `encoding/json`の挙動を利用し、`map[K]V`もしくは`[]T`をベースとする型を工夫することで可能なことが分かった。
- some/noneを表現できる型として`Option[T]`を定義して、`[]Option[T]`を`T | null | undefined`を表現する型とした。
  - [github.com/oapi-codegen/nullable](https://github.com/oapi-codegen/nullable)は`map[bool]T`として利用するが、`[]T`のほうが動作が速いんじゃないかという仮説があった

## やること

- `Go`で`T | null | undefined`がなぜ表現しにくいかについて説明します
- `T | null | undefined`のユースケースとして`Elasticsearch`のpartial updateを説明します
- 普通、フィールドのあるなしをどうやってチェックするなどをwildに存在する広く使われるライブラリの例を引用指定説明します
- 解決法を二つ紹介します。
  - `encoding/json`とすでに互換性のある方法(`[]Option[T]`)
  - `encoding/json/v2`のみで使えるもっと効率的な方法(`Option[Option[T]]`)

## 前提知識

[Go programming language]の細かい説明はしてますが全体の説明はしないので、ある程度知っている人じゃないと意味が分からないかもしれません。

# 環境

```
# go version
go version go1.22.0 linux/amd64
```

ただし`Go 1.18`で追加されたgenericsを用いる以外はバージョン依存な要素はないはずなので、`Go 1.18`ではおおむね通じる話をします。

## 対象読者

- `Go`のstruct fieldで`T | null | undefined`を表現する方法がわからない人
- Go で JSON を受け取る API を組むときに validation などで悩んでいる人
- `encoding/json`のポイントを知りたい人

## おさらい: 時たま困る「データがない状態」の扱い

### Goのzero valueとデータ相互変換

#### Goではすべての変数はzero valueに初期化される

`Go`には[The zero value](https://go.dev/ref/spec#The_zero_value)の概念があるため、

> ... a variable or value is set to the zero value for its type: false for booleans, 0 for numeric types, "" for strings, and nil for pointers, functions, interfaces, slices, channels, and maps.

とある通り、`Go`ではあらゆる変数が型に対応した`zero value`に初期化されます

#### Go valueからのエンコード時、「フィールドがない」の表現にzero valueが使われることがある

`zero value`はstructの「フィールドがない」という表現にたびたび使われます。

structと他の表現の相互変換機能でそういった機能がよくつかわれます。

`encoding/gob`は`zero value`をオミット(=出力先のデータに出現しなくなる)する挙動があります。

> https://pkg.go.dev/encoding/gob@go1.22.5#hdr-Encoding_Details
>
> If a field has the zero value for its type (except for arrays; see above), it is omitted from the transmission.

`encoding/json`, `encoding/xml`はstruct tagで`,omitempty`を指定すると`zero value`がオミットされます。

> https://pkg.go.dev/encoding/json@go1.22.5#Marshal
>
> The "omitempty" option specifies that the field should be omitted from the encoding if the field has an empty value, defined as false, 0, a nil pointer, a nil interface value, and any empty array, slice, map, or string.

> https://pkg.go.dev/encoding/xml@go1.22.5#Marshal
>
> a field with a tag including the "omitempty" option is omitted if the field value is empty. The empty values are false, 0, any nil pointer or interface value, and any array, slice, map, or string of length zero.

この様式に影響されているからなのか、サードパーティのライブラリでもstructと他の表現の相互変換時に「ない」を`zero value`で表現するものがあります。

例えば`Go` valueとurlのquery paramの相互変換を行う[github.com/pasztorpisti/qs](https://github.com/pasztorpisti/qs)にも同様に`,omitempty`オプションがあります。

> https://pkg.go.dev/github.com/pasztorpisti/qs#Marshal
>
> When a field is marshaled with the omitempty option then the field is skipped if it has the zero value of its type.

他には[GORM](https://gorm.io/)というORMの場合、

```go
// https://gorm.io/docs/query.html

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

という風に、`zero value`が「ない」として扱われます(実際に発行される`SELECT`の`WHERE`句に値を指定していないフィールドが出現していません。)。
`zero value`判定は[ここなどでおこなわれています](https://github.com/go-gorm/gorm/blob/02b7e26f6b5dcdc49797cc44c26a255a69f3aff3/schema/field.go#L462)(`reflect`の[IsZero](https://pkg.go.dev/reflect@go1.20#Value.IsZero)などが使われていますね)

#### 外部データからGo valueへのデコード時のzero valueは曖昧

外部データから`Go` valueへのデコード後にあるフィールドが`zero value`であった時、フィールドに対応するデータがなかったのか、`zero value`に対応した値が存在していたのか判別が付きません。

以下のsnippetで示される通り、入力値が`""`, `0`などの`zero value`であるときと`{}`のようなフィールドが空だった時どちらも(`JSON`の場合は`null`も同様に)unmarshal後の値が同じになります。

[playground](https://go.dev/play/p/R-T7d-GLJS-)

```go
type Sample struct {
	XMLName xml.Name `xml:"sample"`
	Foo     string
	Bar     int
}

func main() {
	for _, input := range []string{
		`{"Foo": "foo", "Bar": 123}`,
		`{"Foo": "", "Bar": 0}`,
		`{"Foo": null, "Bar": null}`,
		`{}`,
	} {
		var s Sample
		err := json.Unmarshal([]byte(input), &s)
		if err != nil {
			panic(err)
		}
		fmt.Printf("input = %s,\nunmarshaled = %#v\n\n", input, s)
		/*
			input = {"Foo": "foo", "Bar": 123},
			unmarshaled = main.Sample{XMLName:xml.Name{Space:"", Local:""}, Foo:"foo", Bar:123}

			input = {"Foo": "", "Bar": 0},
			unmarshaled = main.Sample{XMLName:xml.Name{Space:"", Local:""}, Foo:"", Bar:0}

			input = {"Foo": null, "Bar": null},
			unmarshaled = main.Sample{XMLName:xml.Name{Space:"", Local:""}, Foo:"", Bar:0}

			input = {},
			unmarshaled = main.Sample{XMLName:xml.Name{Space:"", Local:""}, Foo:"", Bar:0}
		*/
	}
	for _, input := range []string{
		`<sample><Foo>foo</Foo><Bar>123</Bar></sample>`,
		`<sample><Foo></Foo><Bar>0</Bar></sample>`,
		`<sample></sample>`,
	} {
		var s Sample
		err := xml.Unmarshal([]byte(input), &s)
		if err != nil {
			panic(err)
		}
		fmt.Printf("input = %s,\nunmarshaled = %#v\n\n", input, s)
		/*
			input = <sample><Foo>foo</Foo><Bar>123</Bar></sample>,
			unmarshaled = main.Sample{XMLName:xml.Name{Space:"", Local:"sample"}, Foo:"foo", Bar:123}

			input = <sample><Foo></Foo><Bar>0</Bar></sample>,
			unmarshaled = main.Sample{XMLName:xml.Name{Space:"", Local:"sample"}, Foo:"", Bar:0}

			input = <sample></sample>,
			unmarshaled = main.Sample{XMLName:xml.Name{Space:"", Local:"sample"}, Foo:"", Bar:0}
		*/
	}
}
```

そのため、普通はそういった`""`や`0`がデータとしてありうるケースでは代わりに`*string`などをフィールドの型として指定します。

ただし、この場合でも`JSON`のデコードではUnmarshal後の`Go` structのフィールドの値が`nil`だったことから、入力が`null`だったのか、`undefined`だったのか(フィールドが存在しなかった)のかは判別がつきません。
`xml`には`null`にあたる値の表現がないはずなのでこちらの場合は`*T`なフィールドを指定するだけで十分なはずです。

[playground](https://go.dev/play/p/YM8wrDe-WsD)

```go
type Sample struct {
	XMLName xml.Name `xml:"sample"`
	Foo     *string
	Bar     *int
}

func main() {
	for _, input := range []string{
		`{"Foo": "foo", "Bar": 123}`,
		`{"Foo": "", "Bar": 0}`,
		`{"Foo": null, "Bar": null}`,
		`{}`,
	} {
		var s Sample
		err := json.Unmarshal([]byte(input), &s)
		if err != nil {
			panic(err)
		}
		fmt.Printf("input = %s,\nunmarshaled = %#v\n\n", input, s)
		/*
			input = {"Foo": "foo", "Bar": 123},
			unmarshaled = main.Sample{XMLName:xml.Name{Space:"", Local:""}, Foo:(*string)(0xc0000141c0), Bar:(*int)(0xc0000121f0)}

			input = {"Foo": "", "Bar": 0},
			unmarshaled = main.Sample{XMLName:xml.Name{Space:"", Local:""}, Foo:(*string)(0xc0000141f0), Bar:(*int)(0xc000012230)}

			input = {"Foo": null, "Bar": null},
			unmarshaled = main.Sample{XMLName:xml.Name{Space:"", Local:""}, Foo:(*string)(nil), Bar:(*int)(nil)}

			input = {},
			unmarshaled = main.Sample{XMLName:xml.Name{Space:"", Local:""}, Foo:(*string)(nil), Bar:(*int)(nil)}
		*/
	}
}
```

### T | null | undefined、あるいはpartial JSONのユースケース

`T | null | undefined`を使い分けたいユースケースに`JSON` documentを受け付けるAPIのpartial updateがあります。

`JSON`をやり取りするAPIはたびたびpartial JSON documentを送りあい、存在する(`undefined`でない)フィールドだけ更新し、`T`であればその値に、`null`であればデータを空に更新するようなプラクティスが普通に存在します。

ここではその具体例として`Elasticsearch`という全文検索データストアの[Update part of a document](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update.html#_update_part_of_a_document)を例とします。

[Elasticsearch]は[Apache Lucene](https://lucene.apache.org/)をベースとした全文検索エンジンで`JSON`でドキュメントのストア、検索その他もろもろができます。

そもそも筆者が`T | null | undefined`をうまいこと`Go`で取り扱いたかったのは`Elasticsearch`と相互にやり取りするアプリを`Go`に移植できるか検討していたからなんでした。

例えば以下のような[mapping.json](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html)(SQLでいうところの`CREATE TABLE`みたいなもの)で[indexを作成](https://elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html)(`index`はSQLでいうところのtableみたいなもの)すると

```json: mapping.json
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "kwd": {
        "type": "keyword"
      },
      "long": {
        "type": "long"
      },
      "text": {
        "type": "text"
      }
    }
  }
}
```

以下の`Go` structと相互変換できる`JSON document`をindexに格納し、検索することができます。

```go
type Example struct {
	Kwd             string                               `json:"kwd"`
	Long            int64                                `json:"long"`
	Text            string                               `json:"text"`
}
```

ドキュメントは以下のように、`JSON`でpartial documentを送ると当該のフィールドだけを更新することができます。

```
curl -X POST "localhost:9200/test/_update/1?pretty"\
    -H 'Content-Type: application/json'\
    -d '{"doc": {"kwd": "new keyword"}}'
```

さらに、以下のようにフィールドに`null`を指定すると当該のフィールドを`null`に更新することができます。この場合、`null`はドキュメントにフィールドがないと同じ扱いなので、フィールドを空にすることができます。

```
curl -X POST "localhost:9200/test/_update/1?pretty"\
    -H 'Content-Type: application/json'\
    -d '{"doc": {"kwd": null}}'
```

### 普通の方法: anyを介したvalidation

`JSON`などのフィールドがある(`T`である)/ない(`undefined`)/`null`であるとかのvalidationを行うライブラリは普通、`JSON`を`any`(`map[string]any`)へとデコードして、そこを介してフィールドの有り無しなどをチェックします。

例えば`JSON schema`から`Go` typeを作成するライブラリ[github.com/omissis/go-jsonschema](https://github.com/omissis/go-jsonschema)は、`UnmarshalJSON`を以下のように生成します。

https://github.com/omissis/go-jsonschema/blob/main/tests/data/core/object/object.go

見てのとおり、`map[string]any`と`Plain`それぞれに対して`json.Unmarshal`を呼び出します。
`map[string]any`のほうを使ってフィールドのあるなしをチェックし、`Plain`のほうを結果としてreceiverに代入します。
`type Plain ObjectMyObject`は[`method set`を引き継がない](https://go.dev/ref/spec#Type_definitions)がデータ構造は同じな型を定義するためにこうしています。

他の例を挙げると、[OpenAPI spec](https://github.com/OAI/OpenAPI-Specification)を用いてJSONのvalidationを行うライブラリの[github.com/getkin/kin-openapi](https://github.com/getkin/kin-openapi/blob/2692f43ba21c89366b2a221a86be520b87539352/openapi3filter/validate_request.go#L275)も同様に、`JSON`を`any`にデコードし、これを利用してvalidationを行います。

https://github.com/getkin/kin-openapi/blob/2692f43ba21c89366b2a221a86be520b87539352/openapi3filter/validate_request.go#L275

https://github.com/getkin/kin-openapi/blob/2692f43ba21c89366b2a221a86be520b87539352/openapi3filter/req_resp_decoder.go#L1284-L1292

ちなみにこのライブラリは[github.com/oapi-codegen/echo-middleware](https://github.com/oapi-codegen/echo-middleware)などを経由して[github.com/oapi-codegen/oapi-codegen](https://github.com/oapi-codegen/oapi-codegen)で生成されたサーバーのmiddlewareとして利用できます。

### Partial JSONの受け側にはなれる

> https://pkg.go.dev/encoding/json@go1.22.5#Marshal
>
> ... Array and slice values encode as JSON arrays, except that []byte encodes as a base64-encoded string, and a nil slice encodes as the null JSON value.
>
> ... Pointer values encode as the value pointed to. A nil pointer encodes as the null JSON value.
>
> ... Interface values encode as the value contained in the interface. A nil interface value encodes as the null JSON value.

上記より、`[]byte`以外の`[]T`(slice), `any`(interface), `map[K]V`(map)、`*T`(ポインター型)はzero valueと`JSON`の`null`との相互変換となります。

パパっと読むと`map[K]V(nil)`の相互変換のしかたはいまいち書かれていない気がしますが実際`null`へ変換されます。(別言語で作ったクライアントと`Go`で作ったサーバーとのやり取りで`null`を受けたクライアントがクラッシュしたことが何度もある)

> https://pkg.go.dev/encoding/json@go1.22.5#Unmarshal
>
> ...Unmarshal first handles the case of the JSON being the JSON literal null. In that case, Unmarshal sets the pointer to nil.
>
> Because null is often used in JSON to mean “not present,” unmarshaling a JSON null into any other Go type has no effect on the value and produces no error.

上記より、`JSON`にフィールドが存在しない場合、`Go`フィールドにはなにも代入しません。
`JSON`フィールドに`null`がセットされている場合、pointer type `*T`に対しては`nil`を代入し、**non-pointer type Tに対しては何も代入しない挙動になります**。

この挙動より、wacky valueを用いればPartial JSONの受け側には十分なれます。

[playground](https://go.dev/play/p/1eB1P3Oj7lm)

```go
type Sample struct {
	Foo *string
	Bar *int
}

func (s Sample) GoString() string {
	foo := "<nil>"
	if s.Foo != nil {
		foo = *s.Foo
	}
	var bar any = "<nil>"
	if s.Bar != nil {
		bar = *s.Bar
	}

	return fmt.Sprintf("{Foo:%q,Bar:%v}", foo, bar)
}

func main() {
	var (
		wackyStr = "wacky"
		wackyInt = -9999999
	)
	s := Sample{
		Foo: &wackyStr,
		Bar: &wackyInt,
	}
	for _, input := range []string{
		`{}`,
		`{"Bar": null}`,
		`{"Foo": ""}`,
	} {
		err := json.Unmarshal([]byte(input), &s)
		if err != nil {
			panic(err)
		}
		fmt.Printf("input = %s,\nunmarshaled = %#v\n\n", input, s)
		/*
			input = {},
			unmarshaled = {Foo:"wacky",Bar:-9999999}

			input = {"Bar": null},
			unmarshaled = {Foo:"wacky",Bar:<nil>}

			input = {"Foo": ""},
			unmarshaled = {Foo:"",Bar:<nil>}
		*/
	}
}
```

API界面をまたがないwacky valueの出現は筆者的にはドキュメントがしっかりされていれば許せる範疇ですが、やらんでいいならやりたくはないでしょうね。これはAPI界面に出て来る(ドキュメント上に`"wacky"`ならば暗黙的に無視されると載る)のでやったらダメなタイプのやつです。
同僚の作ったAPIにwacky valueを使った特別なステートの表現があって心停止したことが何度もあるので皆さんはAPIをまたぐwacky valueを許容しないでください。tagged union的なものを定義しましょう。

## Genericsを利用して`T | null | undefined`を表現する型を作る

`T | null | undefined`はつまり1つの型で3つのステートを表現したいわけです。こういった型は`Go 1.18`以前では表現しにくかったのですが、以降は[genericsが導入された](https://tip.golang.org/doc/go1.18#generics)ので用意かつ可能となりました。

### できないパターン: \*\*T

一般的に`null | T`は`*T`で表現されるわけですから、単純な話`**T`を用いればいいわけです。実際`**T`自体はOpenSSLなどのAPIで見たことがあります。

ただし以下のように`**T`をreceiverに指定したmethodはコンパイルできません。

[playground](https://go.dev/play/p/rnverJK2HX-)

```go
type Undefined[T any] **T

// ./prog.go:9:7: invalid receiver type Undefined[T] (pointer or interface type)
func (u Undefined[T]) IsUndefined() bool {
	return u == nil
}

// ./prog.go:13:7: invalid receiver type Undefined[T] (pointer or interface type)
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

関数の引数に`**T`をとってもいいですが、そもそもメソッド自体が`C`で頻出した「第一引数に特定のstructをとる関数群」を便利にできるように構造化したようなもののはずなので、元も子もないような先祖返りとなってしまいますね。

### できるパターン: boolean flag で defined を表現する。

ところで、std の`database/sql`には以下のようなデータ構造が定義されています。

https://pkg.go.dev/database/sql@go1.22.5#Null

```go
type Null[T any] struct {
	V     T
	Valid bool
}
```

これを2段重ねにするだけ目的を達成できそうですね。単純な解法ですが、`*T`に引っ張られて見落としていました。

ということで`Rust`風な`Option[T]`型を以下のように定義します。

```go
// Option represents an optional value.
type Option[T any] struct {
	some bool
	v    T
}

func Some[T any](v T) Option[T] {
	return Option[T]{
		some: true,
		v:    v,
	}
}

func None[T any]() Option[T] {
	return Option[T]{}
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

func (o Option[T]) MarshalJSON() ([]byte, error) {
	if !o.some {
		return []byte(`null`), nil
	}
	return json.Marshal(o.v)
}

func (o *Option[T]) UnmarshalJSON(data []byte) error {
	if string(data) == "null" {
		o.some = false
		var zero T
		o.v = zero
		return nil
	}

	var v T
	err := json.Unmarshal(data, &v)
	if err != nil {
		return err
	}
	o.some = true
	o.v = v
	return nil
}
```

そしてこれを2段重ねすることで`T | null | undefined`を表現できる型とします。

```go
type Und[T any] struct {
	opt option.Option[option.Option[T]]
}

func Defined[T any](t T) Und[T] {
	return Und[T]{
		opt: option.Some(option.Some(t)),
	}
}

func Null[T any]() Und[T] {
	return Und[T]{
		opt: option.Some(option.None[T]()),
	}
}

func Undefined[T any]() Und[T] {
	return Und[T]{}
}

func (u Und[T]) IsDefined() bool {
	return u.opt.IsSome() && u.opt.Value().IsSome()
}

func (u Und[T]) IsNull() bool {
	return u.opt.IsSome() && u.opt.Value().IsNone()
}

func (u Und[T]) IsUndefined() bool {
	return u.opt.IsNone()
}

func (u Und[T]) Value() T {
	if u.IsDefined() {
		return u.opt.Value().Value()
	}
	var zero T
	return zero
}

func (u Und[T]) MarshalJSON() ([]byte, error) {
	if !u.IsDefined() {
		return []byte(`null`), nil
	}
	return json.Marshal(u.opt.Value().Value())
}

func (u *Und[T]) UnmarshalJSON(data []byte) error {
	if string(data) == "null" {
		*u = Null[T]()
		return nil
	}

	var t T
	err := json.Unmarshal(data, &t)
	if err != nil {
		return err
	}

	*u = Defined(t)
	return nil
}
```

## 課題: encoding/jsonはstructをskipしない

## 解決法1: map[T]U, []Tはomitemptyでskip可能

## 解決法2: encoding/json/v2(の候補版を使う)

### map[bool]Tを使う実装: [github.com/oapi-codegen/nullable](https://github.com/oapi-codegen/nullable)

### []Option[T]も使える

## 実装

### Option[T]

### Und[T] []Option[T]

## ベンチマーク

3パターンの入力(フィールドがない/`null`/ある)をUnmarshal/Marshalするラウンドトリップのパフォーマンスをとって比較してみます。

https://github.com/ngicks/und/blob/c1689d2e6c9e6a5f010a78fa6fb36401f709a9da/internal/bench/beanch_test.go

このベンチを筆者環境で実行します(`Docker Desktop`で構築された`wsl2`上の`docker container`)

```
# go version
go version go1.22.0 linux/amd64
# go test -bench . -benchmem
goos: linux
goarch: amd64
pkg: github.com/ngicks/und/internal/bench
cpu: AMD Ryzen 9 7900X 12-Core Processor
BenchmarkSerdeNullableV1-24       589605              1858 ns/op            1362 B/op         32 allocs/op
BenchmarkSerdeMapV1-24            609927              1878 ns/op            1362 B/op         32 allocs/op
BenchmarkSerdeSliceV1-24          642109              1742 ns/op            1250 B/op         30 allocs/op
BenchmarkSerdeNullableV2-24       661170              1712 ns/op             786 B/op         23 allocs/op
BenchmarkSerdeMapV2-24            742197              1600 ns/op             633 B/op         21 allocs/op
BenchmarkSerdeSliceV2-24          704398              1558 ns/op             665 B/op         22 allocs/op
BenchmarkSerdeNonSliceV2-24       782139              1457 ns/op             633 B/op         20 allocs/op
PASS
ok      github.com/ngicks/und/internal/bench    8.056s
```

V1は`encoding/json`+`,omitempty`オプション、V2は[github.com/go-json-experiment/json]+`,omitzero`オプションをさします。
Nullableは[github.com/oapi-codegen/nullable]の`Nullable[T]`型、Mapは自家版`map[bool]T`実装(なくていいんですが`Nullable[T]`とほぼ同じ実装なので、`go get`せずにベンチで比較するために作ってありました)、sliceは`[]Option[T]`ベースの型、NonSliceは`Option[Option[T]]`ベースの型のことをさします。

実行するたび当然数値は変わりますが傾向的に速度の順序はこの通りで入れ替わることはありません。`[]Option[T]`のほうが速いでまあ多分間違いなさそうですね。

## おわりに

[Go programming language]: https://go.dev/
[以前の記事]: https://zenn.dev/ngicks/articles/go-json-that-can-be-t-null-or-undefined
[Elasticsearch]: https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro.html
[github.com/oapi-codegen/nullable]: https://github.com/oapi-codegen/nullable
[github.com/go-json-experiment/json]: https://github.com/go-json-experiment/json
