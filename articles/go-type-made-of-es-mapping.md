---
title: "ElasticsearchのmappingからGoのTypeを作る"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Elasticsearch", "go"]
published: false
---

## Overview

Elasticsearch の JSON を Go で取り扱うためのヘルパーと、mapping からの型を生成する code genrator を作りました。

[Explicit Mapping](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/explicit-mapping.html)で運用する Index のみを想定対象とし、

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
    - これについては他の減で出も構いません。単に、type parameters 風の記法を使ってこの記事が掛かれるというだけです。
- Elasticsearch をある程度使ったことがある
  - REST API などを通じて、PUT(ドキュメント作成)/ UPDATE をしたことがある
  - Query DSL を組んで検索をしたことがある

ここで Elasticsearch が何であるかについて詳しくは語りません。語れるほど知っていません。

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

```go
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

```go
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

仕事で[Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/elasticsearch-intro.html)を操作する TypeScript の Node.js アプリを書いています。

最近になって(仕事以外で)Go をたくさん書くようになって、ずいぶん気に入りました。Go で既存のアプリを書きなおしたい・・・と考えるのですが、以下のようにいろいろと作業が必要な点がありました。

## 前提

Elasticsearch は[データ構造を定義せずに JSON などでドキュメントを格納することができます](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping.html#mapping-dynamic)が、対照に、[事前にデータ構造を部分的、あるいは完全に定義する](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/explicit-mapping.html)こともとできます。

前者については採用するとここで語るべきことは何もないのでこの記事は後者の、特に`"dynamic": "strict"`場合についてのみ取り扱います。

## Elasticsearch に格納する JSON を取り扱いの困難さ

### 問題点 1: null / undefined が区別される

Elasticsearch の[REST での Update は明確に undefined と null の区別します](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/docs-update.html#_update_part_of_a_document)

しかし、Go は普通、このような null / undefined を分けて表現する方法を持たないため、User code での工夫が求められます。

:::details undefined を表現する方法がないことを示す encoding/json からの引用
https://pkg.go.dev/encoding/json@go1.19.3

> Array and slice values encode as JSON arrays, except that []byte encodes as a base64-encoded string, and a nil slice encodes as the null JSON value.

> Pointer values encode as the value pointed to. A nil pointer encodes as the null JSON value.

> Interface values encode as the value contained in the interface. A nil interface value encodes as the null JSON value.

引用の通り、undefined にするための方法は特に示されていません。

UnmarhsalJSON メソッドで空のバイト列(` []byte(``) `)などを返すとエラーです。

https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/encoding/json/encode.go;l=480;drc=8c17505da792755ea59711fc8349547a4f24b5c5;bpv=1;bpt=1

https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/encoding/json/indent.go;l=52;drc=8c17505da792755ea59711fc8349547a4f24b5c5;bpv=1;bpt=1

返すことが許されるのは、有効な JSON 文字列のみです。

また、`omitempty`タグを設定した場合、[MarshalJSON がそもそも呼ばれないの](https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/encoding/json/encode.go;l=748;drc=8c17505da792755ea59711fc8349547a4f24b5c5;bpv=1;bpt=1)で、このメソッドで挙動をコントロールすることはできないようです。

:::

### 問題点 2: フィールドは`null` / `null[]` / `T` / `T[]`のすべてが格納可能

あるフィールドの型を`T`とするとき、そのフィールドは[`null`, `null[]`](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/null-value.html), `T`, [`T[]`, `T[][]`](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/array.html)のいずれも格納可能です。

こちらも同じく Go の通常`[]T`と`T`を同じフィールドに格納する方法を持ちません。

### 問題点 3: 複数の記法を許容する型

複数の mapping type で複数の記法を許すものがあります。Go の built-in type は複数入力を許すものは通常ありません。

例として:

- [boolean](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/boolean.html)は`true`/`false`/`"true"`/`"false"`/`""`(false 扱い)のいずれかでよい(https://www.elastic.co/guide/en/elasticsearch/reference/8.4/boolean.html)は
- [date](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/date.html)はデフォルトでは`YYYY-MM-dd'T'HH:mm:ss.SSSZ` or `YYYY-MM-dd` or `Unix milli seconds`のいずれか
- [geopoint](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/geo-point.html)は 6 つの異なる記法のいずれか
  - GeoJSON
  - Well-Known Text
  - {"lat": 123, "lon": 456}
  - \[`lon`, `lat`\]
  - `"lat,lon"`
  - Geohash

## 既存のライブラリ

https://pkg.go.dev/ と google で軽く検索をかけた限り、似たようなことをやっているライブラリは、少なくとも Go にはありませんでした。あったら悲しくなります。

すくなくとも問題 1 は取り扱っているものがある気がしますが、2 と同時となると特殊な用途になりそう・・・なのであまり探していません。

[公式の go-elasticsearch クライアント](https://github.com/elastic/go-elasticsearch)で TypedApi なるものが実装されていたので少し期待しましたが、これはあくまで`_source`フィールドなどには触らないようですし、型付けを強制したりもしません。

## 解決法

### 解決法 1: \*[]T を利用して undefined / null / T[] / T を表現する

- Elasticsearch のドキュメントを読むと、`undefined / null / T[] / T`以外にも`null[]`や`(T | null)[]`、`T[][]`が許容されているなど、もっといろいろあるのですが今回はそれらは考慮から外すことにしました。
  - これは単に「私(筆者)のユースケース上それらが出てこないから」です

`*T`が`T`が`ある状態`と`ない状態`を表現するならば、`**T`はない状態を二つ表現することができます。C 言語などでよく見た(?)ダブルポインター方式です。

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

:::details Defined Type(type Field[T any] \*[]T)ではない理由

defined type だと、defined type の defined type を作った時メソッドの解結先が base type になってしまうからです。(この場合は`*[]T`)

// TODO: リファレンスを追加する

```go
package main

import (
	"fmt"
	"time"
)

type foo time.Time

func (f foo) String() string {
	return "nah"
}

type bar foo

func main() {
	// nah
	fmt.Printf("%s\n", foo(time.Now()).String())
	// bar(time.Now()).String undefined (type bar has no field or method String
	fmt.Printf("%s\n", bar(time.Now()).String())
}
```

これは Embed した場合でも同じ挙動をします

```go
// こういうとき
type foo struct {
	time.Time
}

type bar foo
```

:::
