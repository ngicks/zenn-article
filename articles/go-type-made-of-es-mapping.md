---
title: "ElasticsearchのmappingからGoのTypeを作る"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Elasticsearch", "go"]
published: false
---

## Overview

- [Elasticsearch]にドキュメントとして格納される JSON を取り扱うための型
- [Query DSL]の強い型付けのヘルパー

の作成を目指し、前者である mapping からの[Go]の型を生成する code generator を作りました。

[Explicit Mapping]で運用する Index のみを想定対象とし、

- いくつかのヘルパータイプを用意し
  - `*[]T` を内部に持つことで、Elasticsearch が許容する`null` / `undefined` / `T[]` / `T`を全て格納できる型
  - Geopoint や Boolean のような複数のデータフォーマットを許容する mapping type と対応した型
- mapping をデコードするための型を定義し
- mapping を解析して text/template などでコードを生成する

ことでこれを実現しました。

自分で使うかもわからないため、力尽きる可能性もあるため、供養代わりに記事にしています。

## 想定読者

- [Go programming language](https://go.dev/) をある程度使ったことがある
  - [encoding/json]の挙動を知っている
- [Elasticsearch] をある程度使ったことがある
  - REST API などを通じて、JSON でドキュメント生成(PUT)をしたことがある
  - [Query DSL] JSON を組んで検索をしたことがある

ここで Elasticsearch が何であるかについて詳しくは語りません。(特に詳しくありませんので）

## 本記事のすること

最初に成果物である code generator のサンプルを乗せ、開発に至った経緯、開発する妥当性について説明します。
その後、実装のポイントに関して説明します。

## 成果物

https://github.com/ngicks/elastic-type

### Installation

実行ファイルを go install することで使用します。

```
go install github.com/ngicks/elastic-type/cmd/generate-es-type@latest
```

### Run

mapping を収めた json、任意でさらに二つのこのコードジェネレーター独自のオプションファイルを読み込ませる事で動作します。

```bash
generate-es-type -prefix-with-index-name -i ./example.json -out-high ./example_high.go -out-raw ./example_raw.go -global-option ./example_global_option.json -map-option ./example_map_option.json
```

```json: example.json
// This is what you fetch from <es_origin>/<index_name>/_mappings
{
  "example": {
    "mappings": {
      "dynamic": "strict",
      "properties": {
        "blob": {
          "type": "binary"
        },
        "bool": {
          "type": "boolean"
        },
        "date": {
          "type": "date",
          "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
        }
      }
    }
  }
}
```

```json: example_global_option.json
{
  "IsSingle": true,
  "TypeOption": {
    "date": {
      "IsRequired": true,
      "IsSingle": true
    }
  }
}
```

```json: example_map_option.json
{
  "blob": {
    "IsRequired": true,
    "IsSingle": false
  }
}
```

It generates:

```go: example_high.go
package example

import (
	"encoding/json"
	"time"

	estype "github.com/ngicks/elastic-type/es_type"
	"github.com/ngicks/flextime"
	typeparamcommon "github.com/ngicks/type-param-common"
)

type Example struct {
	Blob [][]byte        `json:"blob"`
	Bool *estype.Boolean `json:"bool"`
	Date ExampleDate     `json:"date"`
}

func (t Example) ToRaw() ExampleRaw {
	return ExampleRaw{
		Blob: estype.NewFieldSlice(t.Blob, false),
		Bool: estype.NewFieldSinglePointer(t.Bool, false),
		Date: estype.NewFieldSingleValue(t.Date, false),
	}
}

// ExampleDate represents elasticsearch date.
type ExampleDate time.Time

func (t ExampleDate) MarshalJSON() ([]byte, error) {
	return json.Marshal(t.String())
}

var parserExampleDate = flextime.NewFlextime(
	typeparamcommon.Must(flextime.NewLayoutSet(`2006-01-02 15:04:05`)).
		AddLayout(typeparamcommon.Must(flextime.NewLayoutSet(`2006-01-02`))),
)

func (t *ExampleDate) UnmarshalJSON(data []byte) error {
	tt, err := estype.UnmarshalEsTime(
		data,
		parserExampleDate.Parse,
		time.UnixMilli,
	)
	if err != nil {
		return err
	}
	*t = ExampleDate(tt)
	return nil
}

func (t ExampleDate) String() string {
	return time.Time(t).Format(`2006-01-02 15:04:05`)
}
```

and

```go: example_raw.go
package example

import (
	estype "github.com/ngicks/elastic-type/es_type"
)

type ExampleRaw struct {
	Blob estype.Field[[]byte]         `json:"blob"`
	Bool estype.Field[estype.Boolean] `json:"bool" esjson:"single"`
	Date estype.Field[ExampleDate]    `json:"date" esjson:"single"`
}

func (r ExampleRaw) MarshalJSON() ([]byte, error) {
	return estype.MarshalFieldsJSON(r)
}

func (t ExampleRaw) ToPlain() Example {
	return Example{
		Blob: t.Blob.ValueZero(),
		Bool: t.Bool.ValueSingle(),
		Date: t.Date.ValueSingleZero(),
	}
}
```

## きっかけ

仕事で Elasticsearch とやり取りする Node.js(TypeScript)のアプリを組んでいます。

最近になって Go を日常的に書くようになったのですが、Go でまた Mapping から大量のフィールドを定義していくのはとてつもなく面倒だし、

```go: 例.go
type BusinessDoc struct {
  TradeName string `json:"trade_name"`
}
```

このように struct tag を全部のフィールドに定義しないといけないのでどこかでミスしそうだし、プロジェクト毎に Date とか Boolean のようなヘルパータイプを定義するのは大変

```go: 別の例.go
type Stock struct {
  StoredAt ISODatetimeOrUnixMilli `json:"stored_at"`
  IsSold   Boolean                `json:"is_sold"`
}

// Elasticsearch default date type!
// that can be ISO datetime or long value representing unix epoch millis.
type ISODatetimeOrUnixMilli time.Time

func (d *ISODatetimeOrUnixMilli) UnmarshalJSON(data []byte) error {
  // do some decode here.
}

// Elasticsearch boolean type!
// which can be unmarshalled from JSON boolean literal or "true" / "false" / "" string literals!
type Boolean bool

func (b *Boolean) UnmarshalJSON(data []byte) error {
  // do some decode here.
}
```

ついでにいうと date はフォーマット任意かつ複数指定できるので一般化しようとするとなかなか面倒なコードを書くことになります。

ということで code generator を作ることにします。

## Rationale

- [Elasticsearch]は分散全文検索エンジンであり、 JSON を多用する
  - Elasticsearch は REST API などを通じて JSON でドキュメントを格納できる。
  - 検索も同様に JSON で表現できる [Query DSL] によって行うことができる。
  - ドキュメントは `PUT <index-name>/_mappings`で mapping の JSON を渡すことで、[Field data type(s)]で定められた型を持つデータフォーマットとして定義することもできる。
    - [date] / [date_nanos]はフォーマットを mapping によって設定できる
    - [constant_keyword](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/keyword.html#keyword)は mapping によって固定値を設定する
    - 事前に定義しない場合、ドキュメントの内容から mapping が自動作成される。
- Elasticsearch はフィールドの type が T であるとき、格納できる値がそれ以上に多種ある;
  - `T` / [`T[]` / `T[][]` (ネストした T のアレイ), (null | T)[]](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/array.html)
  - [`null` / `null[]`](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/null-value.html)
  - `undefined` (=キーが存在しない)
- 他方、Go では、これに対して十分な JSON のサポートをデフォルトでは持たない。
  - JSON の取り扱いは std では[encoding/json]を通じて行う
    - `Marshal`/`Unmarshal`などで struct から json への encode/decode が可能である
    - `Marshaler`/`Unmarshaler`/`TextMarshaler`/`TextUnmarshaler` interface を通じてある type がどのように encode/decode されるかを定義できる。
  - `encoding/json` では JSON(というか javascript)における`undefined`と`null`を 1 つの型から出力し分けるようなことができない。
  - Elasticsearch が持つ Geopoint などの特殊な型については当然定義されたものはない。
  - 上記のような `T` | `T[]` | `undefined` | `null`を全て格納できる型がない。
  - Unmarshal でデータを代入するには export されている必要があるので、一般的な JSON のキーの命名規則と一致しないことが多く、struct tag を書く必要がある
    - `encoding/json`など、reflect を使うため

:::details undefined と null をどちらも出力できる型を定義する方法がない
https://pkg.go.dev/encoding/json@go1.19.3

> Array and slice values encode as JSON arrays, except that []byte encodes as a base64-encoded string, and a nil slice encodes as the null JSON value.

> Pointer values encode as the value pointed to. A nil pointer encodes as the null JSON value.

> Interface values encode as the value contained in the interface. A nil interface value encodes as the null JSON value.

- 引用の通り、`null`はフィールドが nil であるとき Marshal を通じて出力されます。
  - もしくは MarshalJSON で`[]byte("null")`を返すこと。
- `undefined`を表現するためには struct tag で`omitempty`設定し、フィールドが struct 以外の型かつ zero value にします。

  - [Marshal 中でフィールドがスキップされるような処理になります](https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/encoding/json/encode.go;l=748;drc=8c17505da792755ea59711fc8349547a4f24b5c5;bpv=1;bpt=1)
  - [Array の場合は rv.Len() == 0 の場合のみです。](https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/encoding/json/encode.go;drc=d5de62df152baf4de6e9fe81933319b86fd95ae4;l=339)

- MarshalJSON メソッドで空のバイト列(` []byte(``) `)などを返すとエラーです。
  - [この記述](https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/encoding/json/indent.go;l=17;drc=8c17505da792755ea59711fc8349547a4f24b5c5)からわかるように、返すことが許されるのは、有効な JSON 文字列のみです。

型のみ(= MarshalJSON のみ)によって`undefined` / `null`を表現し分けることは、std の範疇ではできないようです。

:::

また、検索クエリもなかなかトリッキーで

- Nested / Object 型の場合サブフィールドはドットでつなげていく。例えば

```json: mapping.json
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "name": {
        "type": "nested",
        "properties": {
          "first": { "type": "text" },
          "last": { "type": "text" }
        }
      }
    }
  }
}
```

のような mapping があった時、

```json: query.json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name.first": "Alice" }},
        { "match": { "name.last": "Smith" }},
      ]
    }
  }
}
```

のように、Query に指定する場合はドットで結合したフィールドを指定します。

- Mappings の中に[Fields](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/multi-fields.html)を持つ type(`keyword`など)や、search-as-you-type なども同様に、ドットでサブフィールドを検索するようなことができる。

### つまり

- Elasticsearch のドキュメントの JSON を Encode / Decode するための型
  - mapping から生成する必要のある型があるので code generator
- 検索クエリの _strongly typed_ なヘルパー

を作れば Elasticsearch を取り扱う事が楽になり、アプリの業務に集中できるようになります。
また、これらのコードから Elasticsearch におけるデータハンドリングの pitfall などを伝えることも期待できます。

今回実現したのは前者までで、後者は今後やるかもしれない・・・という状態です。

## 課題

前述した取り組むべき課題をまとめます。

- [x] Elasticsearch のドキュメントの JSON を Encode / Decode するための型。
  - `null` / `undefined`をだし分けらる型がないこと
  - Date / Geopoint / Boolean など、複数の記法を許す field mapping type を Unmarshal/Marshal できる型がないこと
  - date はフォーマットによって型が変わるのでそれ専用の型と code generator が必要なこと
  - mapping の properties から Go の一般的な命名規則で名づけられたフィールドを持った型を生成する必要があること
  - 上記の問題から派生しますが、アプリケーション固有のルールとして必ず初期値が入っているフィールドなどが`null` / `undefined`を許容する必要はありませんので、`T`もしくは`[]T`となるように都合をつける方法を追加するとより良いでしょう。
- [ ] ETA TBD: 検索クエリの _strongly typed_ なヘルパー

## 解決法

### \*[]T を利用して undefined / null / T[] / T を表現する

Go は全ての変数が zero value で初期化されるためデータが「ない」状態を表現することは、ポインターなしではできません。

```go: 特にNumeric typeのとき顕著ですよね.go
var someNum int
otherNum := 0
fmt.Println(someNum) // `0`
fmt.Println(someNum == otherNum) // `true`

var someStr string
otherStr := ""
fmt.Println(someStr) // ``
fmt.Println(someStr == otherStr) // `true`
```

`*T`にした場合、`nil | T`の二つの表現できます。`**T`にすれば`nil | *nil | T`を表現できます。C 言語などでよく見る(?)ダブルポインター方式です。

![](/images/go-type-made-of-es-mapping/nil-tree.drawio.png)

筆者のユースケースでは `T[][]`と `null[]`のパターンは出てこないので無視することにします。
`(T | null)[]`のパターンもまた無視しますが、これはもしかしたら有用かもしれないので今後検討するかもしれません。

ここで、slice(`[]T`)も nil を取ることができることを思い出してください(zero value も nil ですね。) 今回取りたいデータは`nil slice`と空の`slice`を区別して表現する必要がありますので、`*[]T`を`**T`の代わりに使うこととします。

`undefined`は`zero value`, `null`は意図的に`nil`をセットしたものとして、

```go
type Field[T any] struct {
	inner *[]T
}

func (f Field[T]) IsUndefined() bool {
  return f.inner == nil
}

func (f Field[T]) IsNull() bool {
  return f.inner != nil && *f.inner == nil
}
```

のようにすると、上記 4 つのヴァリアントを表現可能となります。ただし、struct 型にするため`encoding/json`.Marshal で処理するとき、omitempty でフィールドを省略してもらうことはできませんので、後述の専用の Marshaler が必要になります。

実際の実装は[こちら](https://github.com/ngicks/elastic-type/blob/c12f4a90f959115db8e0de8731a028e24a5b113a/es_type/field.go)

#### Field[T any]むけ Marshaler

前節の`Field[T any]`は struct ですので、omitempty によるフィールドのスキップが利きません。
そこで、専用の Marshaler の定義が必要となります。その Marshaler の引数の型はあらかじめ決めておくことはできませんので、インターフェイスは以下のようになります

```go
func MarshalFieldsJSON(v any) ([]byte, error)
```

内部的には、reflect によって全てのフィールドを走査していく形で実装されることになるはずです。
しかるに以下に示される疑似コードのようになります。

```go
func MarshalFieldsJSON(v any) ([]byte, error) {
	out := []byte(`{`)

	rv := reflect.ValueOf(v)
	rt := rv.Type()

  // reflectで全部のフィールドをiterate overして
	for i := 0; i < rv.NumField(); i++ {
    // 内部の値をanyで取り出すmethodを持つinterfaceに向けてtype assertして
		value := rv.Field(i).Interface().(interface {
      IsUndefined() bool
      IsNull() bool
      GetValueInAny() any
    })

    if value.IsUndefined() {
      // undefinedならスキップして
      continue
    }

		field := rt.Field(i)
		name := getFieldName(field)

    // "<field_name>": that_value, ... になるようにバイト列を追記していく
    out = append(out, []byte(`"`+name+`":`)...)

    if value.IsNull() {
      out = append(out, []byte("null,")...)
      continue
    }

    // 値をanyとして取り出して
    var val any = value.GetValueInAny()

    // binary typeはGoの[]byteをそのまま使っていたりして、GoのMarshalに依存するようになっているので、
    // 素直にjson.Marshalを使う
    // []byteはjson.Marshalでstd encodingのbase64 stringに変換される。
    encoded, err := json.Marshal(val)
    if err != nil {
      return nil, err
    }

    out = append(out, encoded...)

		out = append(out, ',')
	}

	out = append(out, '}')

	return out, nil
}
```

実際にはもうちょっと凝った実装が必要になります。[実際の定義はこちら](https://github.com/ngicks/elastic-type/blob/c12f4a90f959115db8e0de8731a028e24a5b113a/es_type/marshal_field.go)

ということで上記に示される通り、instantiation なしに内部の値を取り出す必要があります。

実際には以下のように定義を行いました。

```go
type UninstantiatedField interface {
	ValueAny(mustSingle bool) any
	IsNull() bool
	IsUndefined() bool
}

func (f Field[T]) ValueAny(single bool) any
```

### 複数の記法を許す field data type に対応した型を作る

もうこれに関しては気合です。ドキュメントを読んで実装していきます。

例えば

- [boolean](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/boolean.html)
  - `true`/`false`/`"true"`/`"false"`/`""`(false 扱い)
- [date](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/date.html)
  - デフォルトでは`YYYY-MM-dd'T'HH:mm:ss.SSSZ` or `YYYY-MM-dd` or `Unix milli seconds`
  - 任意で複数のフォーマットを指定できる
- [geopoint](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/geo-point.html)
  - GeoJSON
  - Well-Known Text
  - {"lat": 123, "lon": 456}
  - \[`lon`, `lat`\]
  - `"lat,lon"`
  - Geohash

のような感じで、特定の型は複数の記法を許容します。date は上限はわかりませんが、任意で複数のフォーマットを指定できます。 geopoint は歴史的経緯で 6 種の記法を許容します。

実際に[field data type(s)]のページをそれぞれ見た限り、以下が特別な Unmarshaler の実装を必要としているようでした:

- [x] boolean
- [x] date for built-in es date formats
- [ ] histogram
- [x] geopoint
- [x] geoshape
- [ ] join
- [ ] ranges
- [ ] rank_feature/rank_features
- [ ] point
  - basically same as geopoint, but fewer supported data notations.
- [ ] shape
  - basically same as geoshape.
- [ ] version

README.md からのコピペです。チェックのついたものに関しては一応の実装が終わっています。date に関しては es がビルトインでサポートしているフォーマット名で指定できるフォーマット(例えば`strict_date_optional_time`や`basic_date_time`のような)のみの話で、それ以外の YYYY-MM-dd 形式で指定するフォーマットは後述の code generator を使います。

実装の仕方の例として Boolean の簡易版を以下に示します。

```go: boolean.go
type Boolean bool

// MarshalJSON implements Marshaler.
func (b Boolean) MarshalJSON() ([]byte, error) {
	return json.Marshal(bool(b))
}

// UnmarshalJSON implements Unmarshaler.
func (b *Boolean) UnmarshalJSON(data []byte) error {
	switch strings.Trim(string(data), " ") {
	case `true`, `"true"`:
		*b = Boolean(true)
		return nil
	case `false`, `"false"`, `""`:
		*b = Boolean(false)
		return nil
	}

  return fmt.Errorf("unknown: %s", string(data))
}

func (b Boolean) String() string {
	if bool(b) {
		return "true"
	} else {
		return "false"
	}
}
```

のような感じで Marshaler / Unmarshaler interface を実装することで、`encoding/json`にエンコードしてもらえるようにします。

実際の実装は[こちら](https://github.com/ngicks/elastic-type/blob/c12f4a90f959115db8e0de8731a028e24a5b113a/es_type/boolean.go)

現時点の実装では型としてはひとつの記法にしか Marshal できなような形で実装されていますが、Unmarhsal 時にフォーマットを記憶するように実装してもいいかもしれません。

### 複数のフォーマットを許容する date 型のコードジェネレーターを作る

Elasticsearch の[date] / [date_nanos]は mappings で任意数のフォーマットを指定し、いずれかのフォーマットのデータを格納できるようです。

Elasticsearch が Java の Apache Lucene を使用していることからわかるように、date のフォーマットの記法も java 方式の`yyyy-MM-dd`のようなものです。それに対して Go の time.Time のフォーマットは、`2006-01-02`のように、独特なトークンを使用します。

Go において`yyyy-MM-dd`な記法を解釈する時間のライブラリは[nleeper/goment](https://github.com/nleeper/goment)などがありますが、こちらは軽くコードを見た限り、Elasticsearch が使用する`yyyy-MM-dd'T'HH:mm:ss`のような`'`による non time-token のエスケープに対応していないようです。

ということで作りましたこちらです。

https://github.com/ngicks/flextime

このパッケージを使って`yyyy`な time format token を`2006`な Go の形式に変換し、複数のレイアウト全てに対して time.Parse を行う事で複数のフォーマットでの変換を可能とします。
Go の time がサポートしてない都合上 weekyear はサポートされません。

ジェネレータの動きを疑似コードで表現すると、

```go
// Elasticsearchのdate formatは||で区切ると複数指定できる
formats := strings.Split(mappings.dateField.format, "||")
// epoch_millisとepoch_secondを取り除いて
// `strict_date_optional_time`のようなbuilt-in formatを
// "yyyy-MM-dd['T'HH:mm:ss.999999999Z]"のようなフォーマットに変換する
strFormats, hasNumFormat, isMillis, _ := toTimeTokenFormat(formats)
// 上記の*flextime.LayoutSetに変換する
layouts := parseLayouts(strFormats)

buf := bytes.NewBuffer(make([]byte, 0))
// templateを実行する
_ = dateTypeTemplate.Execute(buf, Param{
  // 色々入れる
})
return buf.String()
```

みたいな感じで順繰りに変換して行きます。この辺は気合です。

### mappings から型を作る

mappings から型を作るためにはまず mappings を解析するために型を定義する必要があります。実際には`map[string]any`にデコードしても処理自体はできますが、できる限りデータ構造を明示したほうが何をしたいのかわかりやすいコードになるはずなので、事前に定義しで起きます。

ということで作りましたこちらです。

https://github.com/ngicks/elastic-type/blob/c12f4a90f959115db8e0de8731a028e24a5b113a/mapping

(実装してみてから気付いたのですが、[公式クライアントの typed api](https://github.com/elastic/go-elasticsearch/tree/main/typedapi/types)が mappings の型を定義していますね。今後はこちらを使うように定義を変更するかもしれません。）

さて、コードジェネレータについてですが、ざっくりいうと Object like type 用の Generator を組んで、Properties を順に走査しながら Field が Object like の場合だけ再帰し、それ以外の型についてはそれ用の型を生成します。

mappings の Root となる部分は[IndexState](https://github.com/elastic/go-elasticsearch/tree/main/typedapi/types/indexstate.go)の定義を見る限り、[TypeMapping](https://github.com/elastic/go-elasticsearch/tree/main/typedapi/types/typemapping.go)という型で定義されているようです。

```json
{
  "example": {
    "mappings": {
      // <- ここ
      "dynamic": "strict",
      "properties": {
        "foo": {
          "type": "keyword"
        },
        "bar": {
          "type": "long"
        }
      }
    }
  }
}
```

[TypeMapping](https://github.com/elastic/go-elasticsearch/tree/main/typedapi/types/typemapping.go)、[Nested type]、 [Object type]はそれぞれが同じ Dynamic、Properties を使用し、さらに Nested type と Object type のドキュメントを読み比べる限り、クエリに差はあっても Encode / Decode する JSON の型置換しては両者に違いはなさそうです。

そこで、ものすごいざっくりとした疑似コードでコードジェネレータを表現すると以下のようになります。

```go
type GeneratedType struct {
    TyName  string
    TyDef   string
    Imports []string
    Option  FieldOption
}

func generateObjectLike(mappings mapping.Properties) ([]GeneratedType, error) {
  childFields := map[fieldName]childTyName
  def := []GeneratedType

  for fieldName, prop := range someHowMakeItStableOrdered(mappings) {
    if isObject(prop) || isNested(prop) {
      // do recursive generation
      child, _ := generateObjectLike()
      def = append(def, child...)
      childFields[fieldName] = child.TyName
    } else {
      child, _ := generateField(prop)
      def = append(def, child)
      childFields[fieldName] = child.TyeName
    }
  }

  var buf bytes.Buffer
  objectTypeDefTemplate.Execute(&buf, Param{ /* use childFields */})

  return append(GeneratedType{ TyDef: buf.String(), /* fill rest for this object */}, def...), nil
}

func generateField(prop Property) (GeneratedType, error) {
  switch prop.Type {
  case mapping.Binary:
    return GeneratedType{TyName: "[]byte"}, nil
  /* ...snip... */
  }
}
```

### アプリ固有ルールを設定できるようにする

前節で作ったコードジェネレータに対して User Defined な設定値を追加できるようにします。

```go
type optStr string

const (
	Inherit optStr = ""
	None    optStr = ""
	True    optStr = "true"
	False   optStr = "false"
)

type MapOption map[string]FieldOption

type FieldOption struct {
	IsRequired                     optStr
	IsSingle                       optStr
	PreferStringBoolean            optStr // Boolean型がMarshalJSONで"true" / "false"にmarshalされるかどうか
	PreferredTimeMarshallingFormat string // Date型でmarshalに使うフォーマット。mappingsで指定されないフォーマットの場合error.
	PreferTimeEpochMarshalling     optStr // Date型がMarshalJSONでepoch_millis / epoch_secondにmarshalされるかどうか
	ChildOption                    MapOption
}

type GlobalOption struct {
	IsRequired                 optStr
	IsSingle                   optStr
	PreferStringBoolean        optStr
	PreferTimeEpochMarshalling optStr
	TypeOption                 TypeOption
	TypeNameGenerator          TypeNameGenerator
}

type TypeOption map[mapping.EsType]OptionForType

type OptionForType struct {
	IsRequired optStr
	IsSingle   optStr
}
```

とりあえず、(グローバル|フィールドごと|型ごと)に Required / Optional、Many / Single を設定できるようにしています。boolean、date 型だけちょっとややこしい設定を与えています。グローバル、型ごとの設定を与えるため、global-option < type-option < field-option となる優先順位を与えて設定値の継承が起きるようにしています。

## Conclusion

いかがでしたでしょうか。

ヘルパータイプの作成、mappings の解析、text/template を駆使して Elasticsearch の mapping から型を作るコードジェネレータを作成しました。
今後はこれらを拡張することで Query DSL のヘルパーとなる型を作成する予定です。

この記事を通じて Elasticsearch と Go の難しいところ、pitfall が伝わっていれば幸いです。

[elasticsearch]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/elasticsearch-intro.html
[explicit mapping]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/explicit-mapping.html
[query dsl]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/query-dsl.html
[go]: https://go.dev/
[field data type(s)]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping-types.html
[encoding/json]: https://pkg.go.dev/encoding/json@go1.19.3#Unmarshaler
[date]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/date.html
[date_nanos]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/date_nanos.html
[object type]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/object.html
[nested type]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/nested.html
