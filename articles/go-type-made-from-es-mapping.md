---
title: "ElasticsearchのmappingからGoのTypeを作る"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Elasticsearch", "go"]
published: false
---

# Overview

- Helper Types:
  [Elasticsearch]にドキュメントとして格納するJSONのフィールドをmarshal /
  unmarshalする型
- Code Generator: mappingからGoのstructを作るcode generator

を作りました。

作る過程で知りえたあれこれを知見として残すことがこの記事の目的です。

成果物はこちらです。

https://github.com/ngicks/estype

READMEも整備中でありどんどん更新の可能性ありです。

# 前提知識

以下を有する

- [Go programming language](https://go.dev/)を使って開発が行える程度の理解度。
  - time.Parseの使い方、layoutのフォーマット。
- [Elasticsearch]とやり取りするアプリケーションを開発できる程度の理解度。
  - indexの作成
  - documentの格納/取得
  - QueryDSLを使った検索

記事の目的に反してしまうのでElasticsearchについては軽い説明にとどめます。筆者は詳しくないので深く立ち入ることができません。

- 本投稿のごく一部で何の説明もなしにTypeScriptの型表記がでてきます。知っている人か、でなければなんとなくで読んでください。

# 対象読者

- GoでElasticsearchとやり取りするアプリを書いてJSON構造がよくわからなくて困った人
- Elasticsearchのmappingに関する細かい話が知りたい人
- [github.com/dave/jennifer]の使い方について知りたい人
  - まあまあ詳しく書いてあります。

# 環境

作り出した時期が大分前なので、elasticsearchは8.4.3を対象に作られています。
ドキュメントも1つを除いてすべて8.4のものを参照しています。

```
# go version
go version go1.20.6 linux/amd64
# curl ${ELASTICSEARCH_URL}
{
  "name" : "cdf7a5d86cb7",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "raebKhvuRay4SB_eC704NQ",
  "version" : {
    "number" : "8.4.3",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "42f05b9372a9a4a470db3b52817899b99a76ee73",
    "build_date" : "2022-10-04T07:17:24.662462378Z",
    "build_snapshot" : false,
    "lucene_version" : "9.3.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

# 背景

筆者は業務でElasticsearchとやり取りするアプリをNode.jsで開発しています。その中で、いろいろなことで困りました。例えば:

- あるフィールドが`undefined`(JSONにキーが存在しない), `null`, `T`(keywordの場合stringのような), `T[]`のどれでも取りうることがあるため、typescriptの型定義が非常に煩雑であること
- JSON.parseしただけでは[Date]型がjavascriptのDate型に変換できず、アプリケーションの中で変換コードが散らばってしまう。
- [Boolean]型が`true` / `false`だけでなく`"true"` / `"false"`を受け付けるため、アプリ内で非常に煩雑なコードで判別することになってしまう。

後ろの二つは明らかに筆者が未熟でした。JSON.parseした後に、アプリに都合のいいフォーマットに変換する変換部を設けるべきでした。後悔は先に立たないものですね。

一方で、Goではある型が[json.Marshaler](https://pkg.go.dev/encoding/json#Marshaler),
[json.Unmarshaler](https://pkg.go.dev/encoding/json#Unmarshaler)を実装している場合、`json.Marshal()`,
`(*json.Encoder).Encode()`, `json.Unmarshal()`,
`(*json.Decoder).Decode()`などの対応する関数の中で、デフォルトの挙動の代わりにそれらが呼び出されるという挙動があります。

GoはEncode / Decodeの界面での変換、validationに関して意識しやすい設計になっているため、
ここにElasticsearchのindexに格納できるJSON documentとGo structと変換するいいブリッジをかけば同じ轍を踏まずに済みそうです。

## Elasticsearch

コンテキストを共有するためにElasticsearchについて少しだけ説明します。筆者はElasticsearchに明るくないので基本的には引用元をあたってください。

> 引用:
> https://www.elastic.co/guide/en/elasticsearch/reference/8.4/elasticsearch-intro.html
>
> Elasticsearch is the distributed search and analytics engine at the heart of
> the Elastic Stack. Logstash and Beats facilitate collecting, aggregating, and
> enriching your data and storing it in Elasticsearch.
>
> (中略)
>
> Elasticsearch provides near real-time search and analytics for all types of
> data. Whether you have structured or unstructured text, numerical data, or
> geospatial data, Elasticsearch can efficiently store and index it in a way
> that supports fast searches.

Elasticsearchは検索と分析を行う分散型ドキュメントストアであり、
REST APIを通じてJSONをDocumentとして格納することができます。

> 引用:
> https://www.elastic.co/guide/en/elasticsearch/reference/8.4/documents-indices.html
>
> Elasticsearch uses a data structure called an inverted index that supports
> very fast full-text searches.

格納されたドキュメントは、設定に基づいて解析され、
Inverted Indexに変換されます。これにより高速な検索を実現してるとのことです。

> 引用:
> https://www.elastic.co/guide/en/elasticsearch/reference/8.4/search-analyze.html
>
> While you can use Elasticsearch as a document store and retrieve documents and
> their metadata, the real power comes from being able to easily access the full
> suite of search capabilities built on the Apache Lucene search engine library.

Apache Luceneを包むことでこれらの挙動を実現していると述べており、
種々の事情がここから透けてきています。ElasticsearchがShardという単位で管理を行う部分がありますが、
これはLucene Indexのことのようです([参考](https://po3rin.com/blog/try-lucene))。

> 引用:
> https://www.elastic.co/guide/en/elasticsearch/reference/8.4/documents-indices.html
>
> Elasticsearch also has the ability to be schema-less, which means that
> documents can be indexed without explicitly specifying how to handle each of
> the different fields that might occur in a document.
>
> (中略)
>
> You can define rules to control dynamic mapping and explicitly define mappings
> to take full control of how fields are stored and indexed.

ElasticsearchのIndexというのはRDBで言うところのTableに近く、
同じschemaを共有するドキュメントを格納することができ、indexに対して検索をかけることができます。
このschemaを決めるのがmappingであり、
mappingによってIndexに収めるJSON Documentの形を完全に、あるいは部分的に決めることができます。

mappingはしばしば完全に固定にされる(`"dynamic":"strict"`)ことがあります。
これは、index explosionいわれる、形の決まっていないJSONを収めることによって無数のフィールドが作成され、
パフォーマンスが落ちる現象を避けるためや、間違った形のJSONを誤って納めたらすぐわかるようにするなどの意図が考えられます。

> 引用:
> https://www.elastic.co/guide/en/elasticsearch/reference/8.8/mapping-explosion.html
>
> Mappings cannot be field-reduced once initialized. Elasticsearch indices
> default to dynamic mappings which doesn’t normally cause problems unless it’s
> combined with overriding index.mapping.total_fields.limit. The default 1000
> limit is considered generous, though overriding to 10000 doesn’t cause
> noticable impact depending on use case. However, to give a bad example,
> overriding to 100000 and this limit being hit by mapping totals would usually
> have strong performance implications.

引用のように一度作られたindexのフィールド(mapping)は減らすことができないため、そのindexの用途によっては固定しておくほうが事故が少ないので良いでしょう。
書いてある通り、ユーザーの任意な値を収める場合は`"type":"flattened"`フィールドを使うとか、`"index":false`にしておくとかのほうがいいですね。

この記事の目的は主に、`"dynamic":"strict"`なmappingからGoの型を生成することで知識をコードにすることです。

# お品書き

それにあたって以下の作業が必要でした。

- `undefined | null | T | (null | T)[]`をunmarshalできる型を作る。
- Helper typeを実装する: ElasticsearchのField data typesのうち、json.Marshaler/json.Unmarshaler実装が必要なものに対して型を定義する。
  - Field data typesのドキュメント全部読む
- date formatの変換: Elasticsearchの理解するtime formatをGoが理解するそれに変換する部分を実装する。
- code generatorの作成: [github.com/dave/jennifer]を使ったcode generatorを作る。

これらによってElasticsearchを取り扱う際の知識をできるだけコード化することが目的となります。

# 大雑把な設計方針

- 少なくともfielddatatype(helper type)と生成されたコードが使うヘルパーと、それ以外でモジュールを分ける
  - code generatorにstd(`text/template`など)以外を使ってしまうと、生成されたコードに余計な依存が生じるため
    - これは脆弱性のfalse-positiveなどをもたらすはずなので、理想的には必要な依存以外が書かれていないほうがいいでしょう。
- date formatterの変換などはfielddatatypeの中で行う。
  - 現状ランタイムで解析をする予定はないですが、できてもよいでしょう。
- 生成されたコードが使うヘルパーモジュールを定義する
  - code generatorの負担を軽くするためです。
  - helper functionやbuf poolの二重定義を防ぐ方法がこれしか思いつかなかったからです。
    - 複数のmapping.jsonを解析して一つのパッケージに生成を行うと2重定義が起こりえていました。
- code generatorは二つのtypeを生成する
  - １つはplainであり、アプリケーションの決断によって決まる型です。
    - このkeywordフィールドは1つしか値を入れないので`T`であるとか
    - このフィールドは[ingest pipelines]やアプリの初期化処理で値を必ず入れるので、`*T`ではないとか
  - もう1つはrawであり、ありうるすべての型をパーズできるものです。
    - これは`elastic.Elastic[T]`を利用して、`undefined | null | T | (null | T)[]`をすべてパーズできるフィールドを持つ
    - `undefined`と`null`を出し分けられるので、update APIに利用できる
    - アプリの決断が変わり、`T`だったフィールドが`T[]`になったりするなど、混在している状態に対応できる
  - これら二つのconversion methodもあり、容易に相互変換できる
    - 可能である限りコピーは避ける
- code generatorは設定を受け付ける。
  - フィールドごとにoptional(`T`ではなく`*T`)であるか,single value(`T[]`でなく`T`である)かや、
  - Booleanはstringにmarshalすべきかなどを決められる。

# 実装

## `undefined | null | T | (null | T)[]`をunmarshalできる型を作る。

ElasticsearchのJSON Documentの各フィールドはほとんどのtypeにおいてこれらすべてを受け入れるため、
これらをうまく取り扱うことができる型を作ることで、アプリがする決め事を減らすことが目的となります。

アプリがする決め事の例は

- フィールドは[ingest pipelines]の中ですべて`null`で埋める
- `null`は使わない
- 必ず`T[]`にする

といったところでしょうか。

ところが、

- 新しいmappingが追加した時などは、そのフィールドは`null`埋めがされない(はず)。
  - ユースケースが拡張されて新しいフィールドが必要になったときです。
- フィールドに複数の値を格納するようにアプリを変更したとき、既存のドキュメントは`T`、新しいドキュメントは`T[]`の状態ができる
  - これも同じユースケースの拡張によって起きるはずです。

そういった型を実装するには[前回の記事]でも述べた、[github.com/ngicks/und]を拡張し、`Elastic[T]`として実装しました。

https://github.com/ngicks/und/blob/8b7839f325d719733510198e5082bf803cf3316b/elastic/elastic.go#L12-L15

前回の記事で提示した`Undefinedable[T]`と`Nullable[T]`の組み合わせで上記のすべての型を表現することを実現しました。

https://github.com/ngicks/und/blob/8b7839f325d719733510198e5082bf803cf3316b/elastic/elastic.go#L180-L214

`UnmarshalJSON`の実装で`null, T and (null | T)[]`のいずれも受け付けられるようにしてあります。

https://github.com/ngicks/und/blob/8b7839f325d719733510198e5082bf803cf3316b/elastic/elastic.go#L171-L178

とある通り、値がある場合は必ず`T[]`に向けてエンコードされるように設計されています。
`T`にmarshalすべきか、`T[]`かはたまた`undefined`になるべきなのかなど、型レベルでは判別のつきづらい要素であったためです。

## Helper Typeを実装する

### 必要性

Elasticsearchでは、Indexごとに格納するJSON documentの形をmappingによって決めることができます。[spec](https://github.com/elastic/elasticsearch-specification/blob/76e25d34bff1060e300c95f4be468ef88e4f3465/specification/_types/mapping/TypeMapping.ts#L34-L56)によればトップは必ずJSON Objectであり、各フィールドは[field data type(s)]を指定することで、格納するデータの型とそれの意味が決められます。

例えば、以下のようなmappingの場合

```json
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "blob": {
        "type": "binary"
      },
      "range": {
        "type": "integer_range"
      }
    }
  }
}
```

以下のJSONをそのIndexに格納することできます

```json
{
  "name": "alice",
  "blob": "Zm9vYmFyYmF6",
  "range": {
    "gte": 3,
    "lt": 5
  }
}
```

このようにmappingによってJSONの形と意味が決められます。

[text]はその言葉の通り、`analyzer`にって解析されるfull-textコンテンツです。JSONとしては単に`string`でよいです。ドキュメントによると、[binary]はbase64 encodeされたstring、[range]はサブフィールドに`gt`,`gte`,`lt`,`lte`を持つことでrangeを表現できる型です。

Elasticsearch上ではJSONとしてはstringであったりnumberであっても、特定のフォーマットでないとエラーとして拒否される`type`や、`range`のように特定のサブフィールドを持つことが期待されることもあります。

[binary]に関してはGoのコードとしては単に`[]byte`とするだけでよいです。

> 引用: https://pkg.go.dev/encoding/json#Marshal
>
> Array and slice values encode as JSON arrays, except that []byte encodes as a
> base64-encoded string, and a nil slice encodes as the null JSON value.

とあるように、`json.Marshal()`が`[]byte`をbase64にエンコードするように設計されているためです。
もし仮にぎりぎりまでデコードを遅延したいならばそれ用の特別な型が必要ですが、現状では行わない前提です。

他方、[range]など、決められた形のJSONを収める型に関しては、例えば以下のような型が定義されてあると、勘違いが少なく、実装の手間も少ないわけです。例えば

```go: range.go
type Range[T comparable] struct {
	Gt  *T `json:"gt,omitempty"`
	Gte *T `json:"gte,omitempty"`
	Lt  *T `json:"lt,omitempty"`
	Lte *T `json:"lte,omitempty"`
}
```

### 調査

[field data types]の各項目をすべて読んでいきます。

特定のデータフォーマットを要求してくるのは以下のテーブルのもので、「ヘルパー型の実装が必要」カラムに〇がついている型がstdの範疇でencode / decodeできない型です。

| type                      | ヘルパー型の実装が必要 | code generatorが必要 | 対応済み | 備考                                                                            |
| ------------------------- | ---------------------- | -------------------- | -------- | ------------------------------------------------------------------------------- |
| [alias]                   |                        |                      | 〇       | 値を入れてはいけない                                                            |
| [aggregate_metric_double] | 〇                     | 〇                   | 〇       | `"metrics"`で設定した値をサブフィールドに持つObject.                            |
| [binary]                  |                        |                      | 〇       | `[]byte`                                                                        |
| [boolean]                 | 〇                     | 〇                   | 〇       | string(`"true"`, `"false"`, `""`)も受け付けるため                               |
| [date]/[date_nanos]       | 〇                     | 〇                   | 〇       | JavaのDateTimeFormatを変換する必要有。formatによってはnumberも受け付ける        |
| [dense_vector]            |                        | 〇                   | 〇       | `[n]float64`: `n` = `"dims"`フィールドで定義。 `null`/multi-valueを許容しない。 |
| [geo_point]               | 〇                     |                      | 〇       | ６種類（！）のフォーマット                                                      |
| [geo_shape]               | 〇                     |                      | 〇       | [GeoJSON] or [Well-Known Text]                                                  |
| [histogram]               | 〇                     |                      | 〇       |                                                                                 |
| [ip]                      |                        |                      | 〇       | `netip.Addr`                                                                    |
| [join]                    |                        | 〇                   |          | `null`を許容しない。自分で使わなさそうだし大変そうだからとりあえず未実装に。    |
| [constant_keyword]        |                        | 〇                   | 〇       | 常に`undefined`にするかmapping.jsonで記述された値にする。`null`を許容しない。   |
| [nested]                  |                        | 〇                   | 〇       |                                                                                 |
| [object]                  |                        | 〇                   | 〇       |                                                                                 |
| [point]                   | 〇                     |                      |          | geopointとほぼ同じだが実装時に力尽き、未実装。                                  |
| [range]                   | 〇                     | 〇                   | 〇       |                                                                                 |
| [rank_features]           |                        |                      | 〇       | `map[string]float64`。`null`を許容しない。multi-valueを許容しない。             |
| [shape]                   | 〇                     |                      | 〇       | geoshapeのヘルパー型をそのまま使う                                              |
| [version]                 |                        |                      |          | semverパッケージ使う？検討中                                                    |

力尽きてしまったのでjoinとshapeは未実装ですが、他はすべて対応してあります。

### Helper types

#### aggregate metric double

[aggregate_metric_double]は、mappingの`"metrics"`で定義した値のみ値を格納できるようです。

```json
// 引用: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/aggregate-metric-double.html
{
  "mappings": {
    "properties": {
      "my-agg-metric-field": {
        "type": "aggregate_metric_double",
        "metrics": ["min", "max", "sum", "value_count"],
        "default_metric": "max"
      }
    }
  }
}
```

４種類のサブフィールドの組み合わせですから、15種類の型を定義しておけばよいです。そこで:

https://github.com/ngicks/estype/blob/9209b388817a5e7b15a5ff52668828a7f53c0862/generator/gen_aggregate_metric_double/gen.go#L24-L70

以上のように、フラグのon/offの全パターン網羅はfor文で容易に実装できます。これによって事前にすべてのパターンを事前に生成しておけばよいのです。

https://github.com/ngicks/estype/blob/9209b388817a5e7b15a5ff52668828a7f53c0862/fielddatatype/aggregate_metric_double.go

この型のうちmappingに対し適切なものをcode generatorによって選択してもらえばいいわけですね。

#### boolean

[boolean]は以下の値を受けれるとドキュメントされています。

- trueして: `true`, `"true"`
- falseとして: `false`, `"false"`, `""`

boolをbase typeと持つ型とし、`MarshalJSON` / `UnmarshalJSON`を実装すればよいでしょう。

https://github.com/ngicks/estype/blob/9209b388817a5e7b15a5ff52668828a7f53c0862/fielddatatype/boolean.go#L70-L83

困ったことに、stringの`"true"` / `"false"`を好むプロジェクトが存在する(筆者が実際に参加していました)ため、`MarshalJSON`で出すのがboolean literalになる型とstring literalになる型をそれぞれ作ってcode generatorの設定値でどちらを使うか決める決断を下しました。

https://github.com/ngicks/estype/blob/9209b388817a5e7b15a5ff52668828a7f53c0862/fielddatatype/boolean.go#L10-L17

https://github.com/ngicks/estype/blob/9209b388817a5e7b15a5ff52668828a7f53c0862/fielddatatype/boolean.go#L39-L47

#### geo_point

[geo_point]は以下の6つのフォーマットを受け付けます。

- [GeoJSON]
- [Well-Known Text]
- `{"lat":123,"lon":456}`
- `[lon, lat]`
- `"lat,lon"`
- geohash

多いですね。

https://github.com/ngicks/estype/blob/9209b388817a5e7b15a5ff52668828a7f53c0862/fielddatatype/geopoint.go#L15-L147

頑張って実装しました。これで少なくとも公式のサンプルを全部パーズできます。

https://github.com/ngicks/estype/blob/9209b388817a5e7b15a5ff52668828a7f53c0862/fielddatatype/geopoint.go#L149-L159

この型はシンプルな`{"lat":123,"lon":456}`フォーマットにMarshalします。
boolと同じく、どのフォーマットに対してmarshalするかを設定で決められるようにすればよかったと思いますが、力尽きてしまいました・・・。

#### geo_shape

[geo_shape]は[GeoJSON]か[Well-Known Text]を受け付けます。

https://www.elastic.co/guide/en/elasticsearch/reference/8.4/geo-shape.html#input-structure

にある通り、GeoJSON Typeのうち
`"Feature"`と`"FeatureCollection"`以外を受け付けるとあります。

そこで実装は、

https://github.com/ngicks/estype/blob/main/fielddatatype/geoshape.go#L18-L44

特にデータフォーマットを制限したりせず、[github.com/go-spatial/geom](https://github.com/go-spatial/geom)に委譲してしまう実装にしました。内部の実装を読む限り、wktはbboxをサポートしていないのでそれを使われるとデコードできないですが、それ以外は網羅できています。

#### histogram

[histogram]はドキュメントによるとalgorithm agnosticな値のセットであり、あらかじめaggregateされている値を入れておくものらしいです。

これはシンプルに

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/histogram.go

というだけです。

#### point / shape

見たところgeo_point, geo_shapeとほぼ同じなのですが、ドキュメントを読んでる時間がなく、shapeはgeo_shapeをそっくり使いまわします決断をしました。pointがgeo_pointとどう違うか確認が取れなかったため、実装を先送りしています。

#### range

[range]はその名の通り数値のrangeを表現するものです。8.4の時点では

- `integer_range`
- `float_range`
- `long_range`
- `double_range`
- `date_range`
- `ip_range`

があります。

`version_range`が存在しないのがちょっと気になるところですが、semverを数値に変換すれば実現可能なので優先度が低いんでしょうか。

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/range.go

各フィールドは`null`を許容しないため、`,omitempty`が必要です。試してないですが`Gt`と`Gte`、`Lt`と`Lte`は同時に存在してはいけないはずです。これに関しては特に型やメソッドによってvalidationをかけられるようにはしていません。

:::details 型制限をこれ以上きつくできなかった話

実際には以下のようにしたほうが型レベルできついのですが

```go
type Range[T interface {
	constraints.Integer | constraints.Float | netip.Addr | time.Time
}] struct{}
```

後述するElasticsearchのdateフォーマットをすべて理解できる型を作る際に以下のように、エラーになるため、できませんでした。

```go
type builtinDate time.Time

func init() {
	var n Range[builtinDate]
  // builtinDate does not satisfy interface{constraints.Integer | constraints.Float | netip.Addr | time.Time}
  // (builtinDate missing in ~int | ~int8 | ~int16 | ~int32 | ~int64 | ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr | ~float32 | ~float64 | net/netip.Addr | time.Time)
}
```

筆者の理解が正しければ、 structをunderlying typeとする型を、 base
typeとする型をconstraintにすることは現状できません。

つまり

```go
type Range[T interface {
	constraints.Integer | constraints.Float | ~netip.Addr | ~time.Time
}] struct{}

// invalid use of ~ (underlying type of netip.Addr is struct{addr netip.uint128; z *intern.Value})
// invalid use of ~ (underlying type of time.Time is struct{wall uint64; ext int64; loc *time.Location})
```

というエラーです。実際この型にtype paramを入れるのはcode
generatorなのでこの制限は特に問題ないとみなし、とりあえずで`comparable`にしてあります。

:::

## date formatの変換

[date]および[date_nanos] field data
typeは`"format"`フィールドで指定されたフォーマットに従う`string`を収めることができ、フォーマット通りに解釈されて時間としてインデックスされます。

```json
// 引用: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/date.html#multiple-date-formats
{
  "mappings": {
    "properties": {
      "date": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      }
    }
  }
}
```

これらのフォーマットは[DateTimeFormatter](https://docs.oracle.com/javase/8/docs/api/java/time/format/DateTimeFormatter.html)で解釈されるフォーマットです。

フォーマットを指定しない場合`"strict_date_optional_time||epoch_millis"`、`"strict_date_optional_time_nanos||epoch_millis"`がそれぞれデフォルトとして扱われます。

`strict_date_optional_time`などは、[built in format](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping-date-format.html#built-in-date-formats)として定義されており、ここに書かれた特定の文字列は特定のフォーマットとして認識されます。

`epoch_millis`、`epoch_second`はJSONのnumberを使ってやり取りされ、それぞれEpochからの経過時間をミリ秒、秒で表現できます。

つまりまとめると以下が必要です:

- `"format"`フィールドを読んでフォーマットの解析
- built in formatの展開
- DateTimeFormatterが理解するフォーマットをGoの`time.Parse`が理解するフォーマットに変換
  - optional section ( `[`, `]`)の展開
  - トークンごとの変換
- 複数のフォーマットでパーズができる型を定義
  - Marshal /
    Unmarshal時、フォーマットに`epoch_*`が含まれている場合numberも解釈する必要がある。

`"format"`フィールドの展開や、built in formatの内容からレイアウト変換などはcode generatorの行いますのでこのセクションでは述べません。

### optional sectionの展開

この機能は`optionalstring`という名前でパッケージにまとめてあります。

https://github.com/ngicks/estype/tree/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/optionalstring

パパっと調べた限り、特定のトークンで囲まれたstringをoptionalとみなして展開し列挙する、というほしい機能を備えたパッケージは見つかりませんでした。
探し方が悪いだけな可能性が高いですが、いいんですこれは趣味プロジェクトなんだから作ってしまえば。

このパッケージは[github.com/prataprc/goparsec](https://github.com/prataprc/goparsec)というパーザコンビネータを利用して文字列を木構造に変更、
木構造を展開してoptional sectionなしの文章に列挙します。

このパッケージは事前処理のために使われるため、パフォーマンスは重視されていません。
あまり賢い実装をしているとは思えませんし、実際頭がこんがらがりながら木構造を展開する処理をかいていました。
実装の不備やバグは探せばいくらでもあると思いますがdate formatを展開するという用途には現状問題なく動作しています。

### time tokenの変換

こちらは以下のファイルで実装されています。

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/convert.go

中身は`time.Parse`を簡易化したような実装をしており、愚直にswitch-caseを書いてパフォーマンスを求めるより、トークンをテーブル化して実装の負担を減らす方針でいきました。

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/convert.go#L236-L258

こういったtableを作ることで、switch-caseの量を大分減らせます。

doc commentでも述べていますが、
Goが同じ機能を持つトークンを持たない以下はサポートされません

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/convert.go#L28-L41

とくにweekyear系トークンがないのでbuilt in date formatの中にいくつか使えないものが出てきます。

### 複数フォーマットでパーズできる型を定義

#### 複数のstringフォーマットを持つパーザ

これ以下のファイルで実装しました。

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/multi_layout.go

これはとっても簡単ですね。

複数のレイアウトを保持

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/multi_layout.go#L9-L13

イニシャライズ時にlengthでdescending, 文字コードでdescendingでソート、dedupe,
validateし、

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/multi_layout.go#L15-L55

順番にパーズを試みて成功したらそのまま値を返します。

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/multi_layout.go#L79-L89

これは、Elasticsearch自身のソースを参考にしました。どうやっているんだろう、と思って見に行くと単にパーザーをイテレートしながらパーズを繰り返していたので、なるほど、と思いに似たような処理にしています。

#### numberもパーズ/フォーマットできるパーザ

これは前述のMultiLayoutとnumberを変換できるパーザを組み合わせます。

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/estime.go#L51-L54

numberのパーザ/フォーマッタはElasticsearchのそれと一致したstring typeであると非常に楽です。つまり

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/estime.go#L11-L13

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/estime.go#L35-L39

switch-caseによって`time.UnixMilli`と`time.Unix`を呼び出せば所望の動作を実現できます。

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/estime.go#L15-L24

## code generatorの作成

code generatorの目的はmapping.jsonを解析し、[Elasticsearch]にストアされるJSON Documentを円滑に生成/消費できるGoのstruct typeを作ることです。

そのためには:

- mapping.jsonの解析
- 型情報の生成
  - Nested / Object typeのrecursiveな生成
    - dynamicのinheritanceを考慮する
  - それ以外の`type`のフィールドには適切な型を選択する
  - dateやdense_vectorなど適切に生成する
  - 特定の型が`null`を受け付けなかったり、複数の値を受け付けなかったり、単数の値しかなくてもとにかくarrayを受け付けなかったりするのでそれらを考慮する。
- 型情報に基づいてcode generateを行う

### mapping.jsonの解析: specの実装

mapping.jsonを解析するには、mapping.jsonの内容を定義したGo structを定義するか、jsonをパーズせずに手続き的にキーを探索するかなどを行うことになります。

jsonをパーズしないまま探索を行う場合は、例えば、[github.com/tidwall/gjson](https://github.com/tidwall/gjson)を使います。

今回はのちのことを考えて静的なstructを定義してそれを使うこととします。

[github.com/elastic/go-elasticsearch]の`typedapi/types`以下には`IndexState`(index生成時に`/<index_name>`に`PUT`するJSON)や、`TypeMapping`が定義されています。当初はこれを使えばよいと思いましたが、`"type"`フィールドが存在しないとき`object`として扱われるというルールが正しく実装されていないことと、それを正しく修正する方法が不明であることからハンドポートを行いました。

#### specification

今回のこれを実装するためにあれこれ調べるまで全然知らなかったのですが、[github.com/elastic/elasticsearch-specification](https://github.com/elastic/elasticsearch-specification)で、Elasticsearchの種々のJSON Documentの型をtypescriptの型定義として実装してあり、これをcompilerでJSONの何かしらのschemaに変換できます。これによって別言語のクライアントを作成するようです。

goの公式clientライブラリである[github.com/elastic/go-elasticsearch]も、[typedapi](https://github.com/elastic/go-elasticsearch/tree/87bb1b42af071454319c73f91c6e5a35e7b6bc5b/typedapi)以下にここから生成されたコードがあります。

https://github.com/elastic/go-elasticsearch/blob/87bb1b42af071454319c73f91c6e5a35e7b6bc5b/typedapi/types/typemapping.go#L33-L53

ここでmappingの型が定義されています。

ではこれを使えば目的が達せられるのかと言えばそうでもないんです。

#### go-elasticsearchのspecificationの変換に関する問題

propertiesのデコード部分

https://github.com/elastic/go-elasticsearch/blob/87bb1b42af071454319c73f91c6e5a35e7b6bc5b/typedapi/types/typemapping.go#L152-L447

ここで、`"type"`フィールドが不在の場合がハンドルされていません。

> 引用: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/object.html
>
> You are not required to set the field type to object explicitly, as this is the default value.

ドキュメント曰く、`"type"`がないと`"type":"object"`とみなされます。

さらに悪いことに、PropertyにUnmarshalJSONが実装されているのではなく、Propertyをフィールドに持つ各種のstructにデコードのコードが分散しているため、1か所直したフォーク版をメンテすればいいというものではないようです。

go-elasticsearchのMakefileを見る限り、makeの範疇でこの型の生成を行っているわけではないようです。どう直していいやらわからないためPRも書けません。困りましたね。

#### ハンドポート版の実装

幸いなことに生成元のtypescript定義は前述のとおりわかっていますし、go-elasticsearchの各ソースファイルに生成元の定義が載っています。
それさえわかれば後は単純なテキスト置換で実装しなおすことは自体は簡単そうです。

https://github.com/ngicks/estype/tree/main/spec

specというモジュールとして再実装しました。

https://github.com/ngicks/estype/blob/main/spec/mapping/Property.go#L79-L81

こちらでは、ハンドポートであるのでPropertyにUnmarshalJSONが実装される形に変わっています。もちろん`"type"`フィールドの不在もハンドルされています。

https://github.com/ngicks/estype/blob/main/spec/mapping/Property.go#L87-L435

(ちなみにtypedapiの中にははhelper typeで実装したような(rangeのような)型の定義は含まれておらず、無駄な努力をしたわけではなさそうでした。よかったよかった。)

### 型情報の生成

生成されるコードはstructなのでフィールドの名前、型の名前、struct tagからなるため、適当に以下のように情報を定義します。

https://github.com/ngicks/estype/blob/main/generator/object.go#L28-L35

https://github.com/ngicks/estype/blob/main/generator/typeid.go#L31-L39

#### Nested / Object typeのrecursiveな生成

object / nestedは与えられたpropertiesをiterateしながら各フィールドの型情報を集め、自身の型をrenderし、その後にフィールドの型をrenderしていきます。
これを再帰で書くにはdryrunオプションが各関数に必要なので、このパッケージ内のcode genを行う関数はdryrun optionを第二引数で取ります。

https://github.com/ngicks/estype/blob/main/generator/object.go#L37

##### dynamic inheritanceを考慮する

mappingの`"dynamic"`の値によって、そのfield dataがmapping.jsonに載っていない値を持てるかが決まります。

> 引用: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/dynamic.html
>
> Inner objects inherit the dynamic setting from their parent object.

とある通り、上位のオブジェクトから値を継承するため、再帰的な型の生成にはcontext情報が必要となります。
nestedも同じく`"dynamic"`の値を継承します。これはElasticsearch 8.4.3相手に確認してあります。ドキュメントに明確に書かれてはいないですが、nested特殊版objectである、という記述はあります。

https://github.com/ngicks/estype/blob/main/generator/generate.go#L28-L44

再帰する際にnextでそのフィールドに進めるようにします

https://github.com/ngicks/estype/blob/main/generator/generate.go#L83-L100

早速 _unused. future update may..._ な値が出てきて力尽きを感じさせますが、とりあえず現状は十分機能しています。

値の比較には[前回の記事]で書いた`Option[T]`を拡張してRustの`Option<T>`風な`Or`や`OrElse`を追加することで対応します。

https://github.com/ngicks/estype/blob/main/generator/object.go#L49-L69

`strict`でなければいずれでもmapping.jsonに載っていない値をMarshal / Unmarshalするための追加フィールドが必要です。しかるに、`strict`かそれでないかだけが関心なのでこのようなコードになります。

#### その他の型情報の生成

https://github.com/ngicks/estype/blob/main/generator/field.go#L16-L87

ほとんどの型に関しては特にgenerateが必要ないのでテーブルにまとめてあります。ここで書いてある通り、multi-valueを受け付けないであるとか、nullを受け付けないであるとかをパラメータにしておくことで知識をコードとしておきました。ややこしいですね！

このテーブルにない型に関してもそれらしく生成されるようにしてあります。

### github.com/dave/jenniferによるcode generation

code generationは[github.com/dave/jennifer]を使用します。

#### text/templateを使わない理由

code generationを行うとき真っ先に思いつくのは[text/template](https://pkg.go.dev/text/template)を使う方法です。実際一度は検討しました。

というか1度`text/template`で同じようなコードを書いたことがあるんですよ

https://github.com/ngicks/elastic-type/blob/main/generate/date.go#L206-L262

これ、見ただけで何してるかわかりますか？
これは、mapping.jsonのformat解析済みのデータからdate型の生成を行っています。
平叙なテキストなので、変動のない部分が多ければ何してるかまではわかりやすいです。
しかし`if`や`range`が二つネストするともうお手上げです。メンテする自信はありません。

`text/template`を使う方法の問題点は

- `range`, `if`などがネストするとよくわからなくなる
- 改行や空白の制御がしにくい
- パラメータを渡すときにstructを定義する必要がある。
- 当然goのsyntax highlightがかからないので間違いに気づきにくい。

もちろんこれは、`text/template`が複雑なgo codeの生成に使われる際、使う側のテクニックを要するというだけの話です。

`text/template`の明確な良い点は

- 外部からtemplate textの入力を受け付けるような使い方ができる
  - dockerや互換cliの`--format`オプションは`text/template`が認識する文字列です
- go code以外にも使える
- templateから登録した任意の関数を呼び出せる。

などがあります。
明確にdata-drivenとドキュメントで述べてある通り、パラメータのstructが固定されていて、テンプレートが変化する用途でも威力を発揮します。

#### github.com/dave/jennifer

[awesome-go](https://github.com/avelino/awesome-go#generators)を見てみるとリストされているもので任意のgo codeの生成を行えるのはjenniferだけですね。
`golang code generation`と検索して出てくるのもjenniferくらいのものです。

#### jenniferを使ったcode generation

上記のdate生成の部分をjenniferで書きなおすと以下のようになります。

https://github.com/ngicks/estype/blob/main/generator/genestime/gen.go#L14-L170

うーんネストが深いですね。
実際に生成されるコードと記述順序を一致させようとするとネストが深くなりがちです。ただ、jenniferを利用するとGo codeのトークンにほぼ同じ名前の関数を順番に呼ぶだけなので、書きにくいと感じることはなかったです。

:::details jenniferを触ってて最初わかんなかったとこたち

最初に触ってすぐにはわからなかったことを書いていきます。これ別の記事に分けたほうがいいかな・・・

##### \*Tと書くとき

```go
jen.Op("*").Id("T")
```

operatorはすべて`Op()`です。何なら`Id("[]string")`や、`Id(*time.Time)`でも問題ありません。

##### forを回しながらコードを生成する

forでsliceやmapをイテレートしながら値に基づいてコードを生成するには、`jen.*Func`を呼び出します。

`if`や`for`の後の`block`は`BlockFunc`、struct宣言には`StructFunc`、structやmapの初期化には`ValuesFunc`といった感じです。

以下前述した収集した型情報から`type FooBar struct {...}`を生成するコードです。

https://github.com/ngicks/estype/blob/main/generator/object.go#L150-L157

以下のよく書くやつを生成するには

```go
if err != nil {
  return nil, err
}
```

https://github.com/ngicks/estype/blob/main/generator/additional_prop.go#L82-L84

##### byteリテラルをそのまま書く

```go
jen.LitByte('{')
```

と書くと

```go
byte(0x7b)
```

と出力されるんです。脳内に完璧なascii code tableのある方なら困ることはないかもしれませんが、できれば`'{'`とそのまま表示してほしいですね。

その場合、以下のようにすると`'{'`を出力できます。

```go
jen.Id(`'{'`)
```

##### 禁じ手: go codeを直接書く

禁じ手ですが、決まり切ったgo codeなのでjenniferのメソッドチェーン外で生成したい場合は

```go
jen.
  Line().
  Line().
  Id(`
func foo() {
  fmt.Prinln("bar")
}
`,
  ).
  Line().
  Line()
```

とするとよいでしょう。
調べた限り生のstringをそのまま入力させてくれるAPIはないです。
`Id()`は入力をそのまま出力するので`Line()`で改行を挟んでおけば任意のコードを書き込めます。

:::

### 生成されるコード

以下に置かれたデータを入力に

https://github.com/ngicks/estype/tree/main/generator/test/testdata

以下のような方が出力されます。

https://github.com/ngicks/estype/tree/main/generator/test

### テスト

https://github.com/ngicks/estype/blob/main/test.compose.yml

以上のcomposeを使って、elasticsearch 8.4.3相手に

- mapping.jsonでindexを作れるか
- 作られたindexに生成された型のサンプル入力を格納できるか
  - plain, raw両方に対して
- `null`やmulti-valueを許容しない方に対して、適切なrawへの返還がされているか

などをテストしてパスするのを確認しました。長かった・・・。

### cli

[elasticsearch]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/elasticsearch-intro.html
[ingest pipelines]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/ingest.html
[field data type(s)]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping-types.html
[field data types]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping-types.html
[text]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/text.html
[alias]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/field-alias.html
[aggregate_metric_double]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/aggregate-metric-double.html
[binary]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/binary.html
[boolean]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/boolean.html
[date]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/date.html
[date_nanos]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/date_nanos.html
[dense_vector]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/dense-vector.html
[geo_point]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/geo-point.html
[geo_shape]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/geo-shape.html
[histogram]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/histogram.html
[ip]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/ip.html
[join]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/parent-join.html
[constant_keyword]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/keyword.html#constant-keyword-field-type
[nested]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/nested.html
[object]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/object.html
[point]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/point.html
[range]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/range.html
[rank_features]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/rank-features.html
[shape]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/shape.html
[version]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/version.html
[GeoJSON]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/geo-shape.html#:~:text=GeoJSON%20or%20Well%2DKnown%20Text
[Well-Known Text]: https://docs.opengeospatial.org/is/12-063r5/12-063r5.html
[github.com/elastic/go-elasticsearch]: https://github.com/elastic/go-elasticsearch
[github.com/dave/jennifer]: https://github.com/dave/jennifer
[github.com/ngicks/und]: https://github.com/ngicks/und
[前回の記事]: https://zenn.dev/ngicks/articles/go-json-that-can-be-t-null-or-undefined
