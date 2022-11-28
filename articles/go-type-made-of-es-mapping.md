---
title: "ElasticsearchのmappingからGoのTypeを作る"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Elasticsearch", "go"]
published: false
---

## Overview

[Elasticsearch] の JSON を Go で取り扱うためのヘルパーと、mapping からの型を生成する code genrator を作りました。

[Explicit Mapping]で運用する Index のみを想定対象とし、

- いくつかのヘルパータイプを用意し
  - \*\[\]T を内部に持つことで、Elasticsearch が許容する`null` / `undefined` / `T[]` / `T`を全て格納できる型
  - Geopoint や Boolean のような複数のデータフォーマットを許容する mapping type と対応した型
- mapping をデコードするための型を定義し
- mapping を解析して text/template などでコードを生成する

ことでこれを実現しました。

自分で使うかもわからないため、力尽きる可能性が高く、供養代わりに記事にしています。

## 想定読者

- Go programming language をある程度使ったことがある
  - `encode/json`の挙動を知っている
  - type parameters を使ったことがある。
- Elasticsearch をある程度使ったことがある
  - REST API などを通じて、PUT(ドキュメント作成)/ UPDATE をしたことがある
  - [Query DSL] を組んで検索をしたことがある

ここで Elasticsearch が何であるかについて詳しくは語りません。(特に詳しくありませんので）

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

仕事で Elasticsearch とやり取りする Node.js(Typescript)のアプリを組んでいます。

最近になって Go を日常的に書くようになったのですが、Node.js アプリの時にしたいくつかの失敗を Go のライブラリを書くネタついでに解決しようというワケです。

## Rationale

- [Elasticsearch]は分散全文検索エンジンであり、 JSON を多用する
  - Elasticsearch は REST API などを通じて JSON でドキュメントを格納できる。
  - 検索も同様に JSON で表現できる [Query DSL] によって行うことができる。
  - ドキュメントは [Field data type(s)] によってフィールドの意味を定義することができる
- Elasticsearch はフィールドの type が T であるとき、格納できる値がそれ以上に多種ある;
  - `T` / [`T[]` / `T[][]` (ネストした T のアレイ)](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/array.html)
  - [`null` / `null[]`](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/null-value.html)
  - `undefined` (=キーが存在しない)
- 他方、Go では、これに対して十分な JSON のサポートをデフォルトでは持たない。
  - JSON の取り扱いは std では[encoding/json]を通じて行う
    - `Marshal`/`Unmarshal`などで struct から json への encode/decode が可能である
    - `Marshaler`/`Unmarshaler`/`TextMarshaler`/`TextUnmarshaler` interface を通じてある type がどのように encode/decode されるかを定義できる。
  - `encoding/json` では JSON(というか javascript)における`undefined`と`null`をだし分けられない。
  - Elasticsearch が持つ Geopoint などの特殊な型については当然定義されたものはない。
  - 上記の多様な型を通常の T を型に持つフィールドに格納できない。

:::details undefined と null をだし分けられない
https://pkg.go.dev/encoding/json@go1.19.3

> Array and slice values encode as JSON arrays, except that []byte encodes as a base64-encoded string, and a nil slice encodes as the null JSON value.

> Pointer values encode as the value pointed to. A nil pointer encodes as the null JSON value.

> Interface values encode as the value contained in the interface. A nil interface value encodes as the null JSON value.

- 引用の通り、`null`は pointer type で nil の場合、もしくは UnmarshalJSON で`[]byte("null")`を返すことで出力できます。
- `undefined`を表現するためには struct tag で`omitempty`設定し、フィールドが struct 以外の型かつ zero value にします。
  - [Marshal 中でフィールドがスキップされるような処理になります](https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/encoding/json/encode.go;l=748;drc=8c17505da792755ea59711fc8349547a4f24b5c5;bpv=1;bpt=1)
  - [Array の場合は rv.Len() == 0 の場合のみです。](https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/encoding/json/encode.go;drc=d5de62df152baf4de6e9fe81933319b86fd95ae4;l=339)

UnmarhsalJSON メソッドで空のバイト列(` []byte(``) `)などを返すとエラーです。

https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/encoding/json/encode.go;l=480;drc=8c17505da792755ea59711fc8349547a4f24b5c5;bpv=1;bpt=1

https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/encoding/json/indent.go;l=52;drc=8c17505da792755ea59711fc8349547a4f24b5c5;bpv=1;bpt=1

返すことが許されるのは、有効な JSON 文字列のみです。

型のみ(= UnmarshalJSON のみ)によって`undefined` / `null`を表現し分けることは、std の範疇ではできないようです。

:::

また、検索クエリもなかなかトリッキーで、例えば

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

```json: query.json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name.first": "Alice" }},
        { "match": { "name.first": "Smith" }},
      ]
    }
  }
}
```

- Mappings の中に[Fields](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/multi-fields.html)を持つ type(`keyword`など)や、search-as-you-type なども同様に、ドットでサブフィールドを検索するようなことができる。

- Elasticsearch のドキュメントの JSON を Encode / Decode するための型
- 検索クエリの _strongly typed_ なヘルパー

を作れば Elasticsearch を取り扱う事が楽になり、アプリの業務に集中できるようになることが予測されます。

今回実現したのは前者までで、後者は今後やるかもしれない・・・という状態です。

## 課題

前述した取り組むべき課題をまとめます。

- [x] Elasticsearch のドキュメントの JSON を Encode / Decode するための型。
  - `null` / `undefined`をだし分けらる型がないこと
  - Date / Geopoint / Boolean など、複数の記法を許す field mapping type を Unmarshal/Marshal できる型がないこと
  - 上記の問題から派生しますが、アプリケーション固有のルールとして必ず初期値が入っているフィールドなどが`null` / `undefined`を許容する必要はありませんので、`T`もしくは`[]T`となるように都合をつける方法を追加するとより良い。
- [ ] ETA TBD: 検索クエリの _strongly typed_ なヘルパー

## 解決法

### \*[]T を利用して undefined / null / T[] / T を表現する

`*T`が`T`がデータが`ある状態`と`ない状態`を表現するならば、`**T`はない状態を二つ表現することができます。C 言語などでよく見た(?)ダブルポインター方式です。筆者のユースケースでは `T[][]`と `null[]`のパターンは出てこないので無視します。(T | null)[]のパターンもまた無視しますが、これは有用そうなケースも想像できますのでそのうちサポートするかもしれません。

ここで、slice(`[]T`)も nil を取ることができることを思い出してください。今回取りたいデータは`nil slice`と空の`slice`を区別して表現する必要がありますので、`**[]T`の代わりに`*[]T`で要件を満たすことになります。

`undefined`は`zero value`, `null`は意図的に`nil`をセットしたものとして、

```go
type Field[T any] struct {
	inner *[]T
}

func (f Field[T]) IsNull() bool {
     return val != nil && *val == nil
}

func (f Field[T]) IsUndefined() bool {
    return val == nil
}
```

のようにすると、上記すべてのヴァリアントを表現可能となります。

### 複数の記法を許す field mapping type

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

のような感じで、特定の型は複数の記法を許容します。特に geopoint が最もヴァリアントが多く、6 種の記法を許容します。

Boolean の場合は以下の用に実装します。

```go: boolean.go
type Boolean bool

func (b Boolean) MarshalJSON() ([]byte, error) {
	return json.Marshal(bool(b))
}

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

現時点の実装では型としてはひとつの記法にしか Marshal できなような形で実装されていますが、Unmarhsal 時にフォーマットを記憶するように実装してもいいかもしれません。

[elasticsearch]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/elasticsearch-intro.html
[explicit mapping]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/explicit-mapping.html
[query dsl]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/query-dsl.html
[field data type(s)]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping-types.html
[encoding/json]: https://pkg.go.dev/encoding/json@go1.19.3#Unmarshaler

```

```
