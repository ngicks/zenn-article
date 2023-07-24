---
title: "ElasticsearchのmappingからGoのTypeを作る"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Elasticsearch", "go"]
published: false
---

# Overview

- Helper Types:
  [Elasticsearch]にドキュメントとして格納するJSONのフィールドをmarshal / unmarshalする型
- Code Generator: mappingからGoのstructを作るcode generator

を作りました。

作る過程で知りえたあれこれを知見として残すことがこの記事の目的です。

成果物はこちらです。

https://github.com/ngicks/estype

以下でインストールし、

```
# go install github.com/ngicks/estype/cmd/genestype@latest
```

以下のようなオプションを受け付けます。

```
root@16cb5614efe3:/mnt/git/github.com/ngicks/estype# genestype --help
Usage of genestype:
  -c string
        path to config file.
        see definition of github.com/ngicks/estype/generator.GeneratorOption.
  -m string
        path to mapping.json.
        You can use one that can be fetched from '<index_name>/_mapping',
        or one that you've sent when creating index.
  -o string
        [optional] path to output generated code. (default "--")
  -p string
        package name of generated code.
```

サンプルで用意してあるmapping.jsonとオプションは以下に格納され

https://github.com/ngicks/estype/tree/main/generator/test/testdata

それを`genestype`に食わせて以下のコードを生成してあります。

https://github.com/ngicks/estype/tree/main/generator/test

# 前提知識

以下を有する

- [Go programming language](https://go.dev/)を使って開発が行える程度の理解度。
  - time.Parseの使い方、layoutのフォーマット。
- [Elasticsearch]とやり取りするアプリケーションを開発できる程度の理解度。
  - indexの作成
  - documentの格納/取得
  - QueryDSLを使った検索

基本的な説明はできる限りするよう心がけますが、十分足りているかは不明です。GoとElasticsearchについて両方ともを使って開発した経験があることを前提とします。

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

一方で、Goではある型が[json.Marshaler](https://pkg.go.dev/encoding/json#Marshaler),[json.Unmarshaler](https://pkg.go.dev/encoding/json#Unmarshaler)を実装している場合、`json.Marshal()`,`(*json.Encoder).Encode()`, `json.Unmarshal()`,`(*json.Decoder).Decode()`などの対応する関数の中で、デフォルトの挙動の代わりにそれらが呼び出されるという挙動があります。

GoはEncode / Decodeの界面での変換、validationに関して意識しやすい設計になっているため、ここにElasticsearchのindexに格納できるJSON documentとGo structと変換するいいブリッジをかけば同じ轍を踏まずに済みそうです。

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

格納されたドキュメントはデフォルトでは１秒程度で検索可能状態となり、謳われる通り`near real-time`ということです。

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
このschemaを決めるのがmappingであり、mappingによってIndexに収めるJSON Documentの形を完全に、あるいは部分的に決めることができます。

mappingはしばしば完全に固定にされる(`"dynamic":"strict"`)ことがあります。
これは、index explosionいわれる、形の決まっていないJSONを収めることによって無数のフィールドが作成され、パフォーマンスが落ちる現象を避けるためや、間違った形のJSONを誤って納めたらすぐわかるようにするなどの意図が考えられます。

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
- Elasticsearchに格納できるJSONの形を調査する。
- Helper typeを実装する: ElasticsearchのField data typesのうち、単なる`string`や`int`などでない型に対するhelperを作成する。
- date formatの変換: Elasticsearchの理解するtime formatをGoが理解するそれに変換する部分を実装する。
- code generatorの作成: [github.com/dave/jennifer]を使ったcode generatorを作る。

これらによってElasticsearchを取り扱う際の知識をできるだけコード化することが目的となります。

# `undefined | null | T | (null | T)[]`をunmarshalできる型を作る。

ElasticsearchのJSON Documentの各フィールドはほとんどのtypeにおいてこれらすべてを受け入れるため、これらをうまく取り扱うことができる型を作ることで、アプリがする決め事を減らすことが目的となります。

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

これらすべては[reindex](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/docs-reindex.html#reindex-with-an-ingest-pipeline)とpipelineなどででデータをいじって、[alias](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/aliases.html)をreindex先に切り替えれば無停止でこれらのルールの変換をできるはずです。しかし、ドキュメント数が多いとメモリやCPU使用率が高い状態が長時間続くため、しないでいいならしたくないでしょう。

上記の型を実装するには[前回の記事]でも述べた、[github.com/ngicks/und]を拡張し、`Elastic[T]`として実装しました。

https://github.com/ngicks/und/blob/8b7839f325d719733510198e5082bf803cf3316b/elastic/elastic.go#L12-L15

前回の記事で提示した`Undefinedable[T]`と`Nullable[T]`の組み合わせで上記のすべての型を表現することを実現しました。

https://github.com/ngicks/und/blob/8b7839f325d719733510198e5082bf803cf3316b/elastic/elastic.go#L180-L214

`UnmarshalJSON`の実装で`null, T and (null | T)[]`のいずれも受け付けられるようにしてあります。

https://github.com/ngicks/und/blob/8b7839f325d719733510198e5082bf803cf3316b/elastic/elastic.go#L171-L178

とある通り、値がある場合は必ず`T[]`に向けてエンコードされるように設計されています。
`T`にmarshalすべきか、`T[]`かはたまた`undefined`になるべきなのかなど、型レベルでは判別のつきづらい要素であったためです。

# Elasticsearchに格納できるJSONの形を調査する

## Elasticsearchのmappingとフィールドの形

Elasticsearchでは、Indexごとに格納するJSON documentの形をmappingによって決めることができます。[spec](https://github.com/elastic/elasticsearch-specification/blob/76e25d34bff1060e300c95f4be468ef88e4f3465/specification/_types/mapping/TypeMapping.ts#L34-L56)によればトップは必ずJSON Objectであり、各フィールドは[field data type(s)]を指定することで、格納するデータの型とそれの意味が決められます。

前述のとおり、mappingはindex生成時などに事前に定義することができ、設定によって部分的に、あるいは完全にJSONの形を固定することができます。

mappingはJSONとして`PUT /<index_name>`にsettingとともに渡すことができます。

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

ドキュメントによると

- [text]はその言葉の通り、`analyzer`にって解析されるfull-textコンテンツです。
- [binary]はbase64 encodeされたstring
- [range]はサブフィールドに`gt`,`gte`,`lt`,`lte`を持つことでrangeを表現できる型です。

`text`や`keyword`は`string`、`integer`や`double`は`int32`や`float64`に単にすればよいですが、それ以外は大なり小なりフォーマットに意味があります。

## Elasticsearchの各field data typeのふるまい

8.4の[field data types]の項をすべて読み、以下を一通りチェックしました。tableの以下の各項目と対応しています。

- built-in / std
  - => Goのbuilt-in typeとstdの範疇で表現できるか
- 要mapping解析
  - => mappingの値によって型が変わったりcode generationで生成される内容が変わるか。
- disallow multi-value
  - => 複数値を受け付けるか
- disallow null
  - => `null`をそのフィールドに格納できるか
- single valueのarrayを受け付けるか
  - `dense_vector`以外は許されました。おそらく[Arrays](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/array.html)の項に説明がある通り、`T[][]`は`T[]`にflattenされるからでしょう。

記事の都合上ここに書いてありますが作ってて、テストしてたら「あれエラー吐くな」って思って調べました。こんな作らなかったらこんな細かく挙動を見なかったと思います。

| field data type           | built-in / std       | 要mapping解析 | disallow multi-value | disallow null | 備考                                       |
| ------------------------- | -------------------- | ------------- | -------------------- | ------------- | ------------------------------------------ |
| [aggregate_metric_double] |                      | 〇            | 〇                   |               |                                            |
| [alias]                   | `N/A`                | 〇            | 〇                   | 〇            | いかなる値も受け付けない                   |
| [binary]                  | `[]byte`             |               |                      |               |                                            |
| [boolean]                 |                      |               |                      |               |                                            |
| [completion]              | `string`             |               |                      | 〇            |                                            |
| [date]/[date_nanos]       |                      | 〇            |                      |               |                                            |
| [dense_vector]            |                      | 〇            | 〇                   | 〇            | `[[1,2,3]]`も受け付けない                  |
| [flattened]               | `map[string]any`     |               |                      |               |                                            |
| [geo_point]               |                      |               |                      |               |                                            |
| [geo_shape]               |                      |               |                      |               |                                            |
| [histogram]               |                      |               | 〇                   |               |                                            |
| [ip]                      | `netip.Addr`         |               |                      |               |                                            |
| [join]                    |                      | 〇            | 〇                   | 〇            |                                            |
| [keyword]                 | `string`             |               |                      |               |                                            |
| [constant_keyword]        | `string`             | 〇            |                      | 〇            | mapping.jsonで入れたワードしか受け付けない |
| [wildcard]                | `string`             |               |                      |               |                                            |
| [nested]                  |                      | 〇            |                      |               | code generatorからすると`object`と同じ     |
| [byte]                    | `int8`               |               |                      |               |                                            |
| [double]                  | `float64`            |               |                      |               |                                            |
| [float]                   | `float32`            |               |                      |               |                                            |
| [half_float]              | `float32`            |               |                      |               | goはnativeで`float16`をサポートしない      |
| [integer]                 | `int32`              |               |                      |               |                                            |
| [long]                    | `int64`              |               |                      |               |                                            |
| [scaled_float]            | `float64`            |               |                      |               |                                            |
| [short]                   | `int16`              |               |                      |               |                                            |
| [object]                  |                      | 〇            |                      |               |                                            |
| [percolator]              | `map[string]any`     |               | 〇                   | 〇            | QueryDSLを格納する用途                     |
| [point]                   |                      |               |                      |               |                                            |
| [date_range]              |                      | 〇            |                      |               |                                            |
| [double_range]            |                      |               |                      |               |                                            |
| [float_range]             |                      |               |                      |               |                                            |
| [integer_range]           |                      |               |                      |               |                                            |
| [ip_range]                |                      |               |                      |               |                                            |
| [long_range]              |                      |               |                      |               |                                            |
| [rank_feature]            | `float64`            |               | 〇                   |               |                                            |
| [rank_features]           | `map[string]float64` |               |                      | 〇            | 同じキーが複数のobjectにあるとエラー       |
| [search_as_you_type]      | `string`             |               |                      |               |                                            |
| [shape]                   |                      |               |                      |               |                                            |
| [text]                    | `string`             |               |                      |               |                                            |
| [match_only_text]         | `string`             |               |                      |               |                                            |
| [token_count]             |                      |               |                      |               | おそらくfieldプロパティ―以外では使えない？ |
| [unsigned_long]           | `uint64`             |               |                      |               |                                            |
| [version]                 | `string`             |               |                      |               | semverパッケージ使うほうがいいかもしれない |

# Helper Typeを実装する

## 必要性

上記のように、ものによっては単なるbuilt-in typeでElasticsearchのfield data typeを表現できません。

前記の[text], [binary], [range]のうち、`text`は単なる`string`,`binary`は`[]byte`とすればよいでしょう。

> 引用: https://pkg.go.dev/encoding/json#Marshal
>
> Array and slice values encode as JSON arrays, except that []byte encodes as a
> base64-encoded string, and a nil slice encodes as the null JSON value.

とあるように、`json.Marshal()`が`[]byte`をbase64にエンコードするように設計されているためです。
もし仮にぎりぎりまでデコードを遅延したいならばそれ用の特別な型が必要ですが、ひとまずはそのことは考えないようにしましょう。

他方、[range]など、決められた形のJSONを収める型に関しては、例えば以下のような型が定義されてあると、勘違いが少なく、実装の手間も少ないわけです。

```go: range.go
type Range[T comparable] struct {
	Gt  *T `json:"gt,omitempty"`
	Gte *T `json:"gte,omitempty"`
	Lt  *T `json:"lt,omitempty"`
	Lte *T `json:"lte,omitempty"`
}
```

## 実装

前述のテーブルの`built-in / std`が空白であった型に関してhelper typeを定義します。

力尽きてしまったのでjoinとpointは未実装です。自分で使う機会があれば実装するかもしれません。

### aggregate metric double

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

以上のように、フラグのon/offの全パターン網羅は`for`文で容易に実装できます。これによって事前にすべてのパターンを事前に生成しておけばよいのです。

https://github.com/ngicks/estype/blob/9209b388817a5e7b15a5ff52668828a7f53c0862/fielddatatype/aggregate_metric_double.go

この型のうちmappingに対し適切なものをcode generatorによって選択してもらえばいいわけですね。

### boolean

[boolean]は以下の値を受けれるとドキュメントされています。

- trueして: `true`, `"true"`
- falseとして: `false`, `"false"`, `""`

boolをbase typeと持つ型とし、`MarshalJSON` / `UnmarshalJSON`を実装すればよいでしょう。

https://github.com/ngicks/estype/blob/9209b388817a5e7b15a5ff52668828a7f53c0862/fielddatatype/boolean.go#L70-L83

困ったことに、stringの`"true"` / `"false"`を好むプロジェクトが存在する(筆者が実際に参加していました)ため、`MarshalJSON`で出すのがboolean literalになる型とstring literalになる型をそれぞれ作ってcode generatorの設定値でどちらを使うか決める決断を下しました。

https://github.com/ngicks/estype/blob/9209b388817a5e7b15a5ff52668828a7f53c0862/fielddatatype/boolean.go#L10-L17

https://github.com/ngicks/estype/blob/9209b388817a5e7b15a5ff52668828a7f53c0862/fielddatatype/boolean.go#L39-L47

### geo_point

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

### geo_shape

[geo_shape]は[GeoJSON]か[Well-Known Text]を受け付けます。

https://www.elastic.co/guide/en/elasticsearch/reference/8.4/geo-shape.html#input-structure

にある通り、GeoJSON Typeのうち
`"Feature"`と`"FeatureCollection"`以外を受け付けるとあります。

そこで実装は、

https://github.com/ngicks/estype/blob/main/fielddatatype/geoshape.go#L18-L44

特にデータフォーマットを制限したりせず、[github.com/go-spatial/geom](https://github.com/go-spatial/geom)に委譲してしまう実装にしました。内部の実装を読む限り、wktはbboxをサポートしていないのでそれを使われるとデコードできないですが、それ以外は網羅できています。

### histogram

[histogram]はドキュメントによるとalgorithm agnosticな値のセットであり、あらかじめaggregateされている値を入れておくものらしいです。

これはシンプルに

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/histogram.go

というだけです。

### point / shape

見たところgeo_point, geo_shapeとほぼ同じなのですが、ドキュメントを読んでる時間がなく、shapeはgeo_shapeをそっくり使いまわします決断をしました。pointがgeo_pointとどう違うか確認が取れなかったため、実装を先送りしています。

### range

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

# date formatの変換

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

## optional sectionの展開

この機能は`optionalstring`という名前でパッケージにまとめてあります。

https://github.com/ngicks/estype/tree/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/optionalstring

パパっと調べた限り、特定のトークンで囲まれたstringをoptionalとみなして展開し列挙する、というほしい機能を備えたパッケージは見つかりませんでした。
探し方が悪いだけな可能性が高いですが、いいんですこれは趣味プロジェクトなんだから作ってしまえば。

このパッケージは[github.com/prataprc/goparsec](https://github.com/prataprc/goparsec)というパーザコンビネータを利用して文字列を木構造に変更、
木構造を展開してoptional sectionなしの文章に列挙します。

このパッケージは事前処理のために使われるため、パフォーマンスは重視されていません。
あまり賢い実装をしているとは思えませんし、実際頭がこんがらがりながら木構造を展開する処理をかいていました。
実装の不備やバグは探せばいくらでもあると思いますがdate formatを展開するという用途には現状問題なく動作しています。

## time tokenの変換

こちらは以下のファイルで実装されています。

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/convert.go

中身は`time.Parse`を簡易化したような実装をしており、愚直にswitch-caseを書いてパフォーマンスを求めるより、トークンをテーブル化して実装の負担を減らす方針でいきました。

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/convert.go#L236-L258

こういったtableを作ることで、switch-caseの量を大分減らせます。

doc commentでも述べていますが、
Goが同じ機能を持つトークンを持たない以下はサポートされません

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/convert.go#L28-L41

とくにweekyear系トークンがないのでbuilt in date formatの中にいくつか使えないものが出てきます。

## 複数フォーマットでパーズできる型を定義

### 複数のstringフォーマットを持つパーザ

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

### numberもパーズ/フォーマットできるパーザ

これは前述のMultiLayoutとnumberを変換できるパーザを組み合わせます。

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/estime.go#L51-L54

numberのパーザ/フォーマッタはElasticsearchのそれと一致したstring typeであると非常に楽です。つまり

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/estime.go#L11-L13

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/estime.go#L35-L39

switch-caseによって`time.UnixMilli`と`time.Unix`を呼び出せば所望の動作を実現できます。

https://github.com/ngicks/estype/blob/c6ed9fb0db8fa145d20fe407394c598e51083903/fielddatatype/estime/estime.go#L15-L24

# code generatorの作成

code generatorの目的はmapping.jsonを解析し、[Elasticsearch]にストアされるJSON Documentを円滑に生成/消費できるGoのstruct typeを作ることです。

そのためには:

- mapping.jsonの解析し
- ユーザーから設定値を受けとり
- 型情報に基づいてcode generateを行う

最初のほうのセクションでのべた通り、

- `T`と`T[]`
- `undefined`と`null`

などが混在することを許しながら、Goのほかのコードで円滑に消費できる平叙な型を生成するのがこのcode generatorの目的です。

そのため、`Plain`と`Raw`の２つのタイプと、相互に変換を行うメソッドを実装する方針になっています。

`Plain`は以下のサンプル生成コードのように「普通の」Go structのようなものです

https://github.com/ngicks/estype/blob/main/generator/test/all.go#L14-L58

`Raw`は`undefined | (null | T) | (null | T)[]`を許容するstructです

https://github.com/ngicks/estype/blob/main/generator/test/all.go#L107-L151

`Plain`はユーザーから設定値を受けとって、`T`、`[]T`、`*T`、`*[]T`のいずれであるかなどを決めます。

## mapping.jsonの解析: specの実装

mapping.jsonを解析するには、mapping.jsonの内容を定義したGo structを定義して`json.Unmarshal()`するか、jsonをパーズせずに手続き的にキーを探索するかなどを行うことになります。

jsonをパーズしないまま探索を行う場合は、例えば、[github.com/tidwall/gjson](https://github.com/tidwall/gjson)を使います。

今回はのちのことを考えて静的なstructを定義してそれを使うこととします。

[github.com/elastic/go-elasticsearch]の`typedapi/types`以下には`IndexState`(index生成時に`/<index_name>`に`PUT`するJSON)や、`TypeMapping`が定義されています。当初はこれを使えばよいと思いましたが、`"type"`フィールドが存在しないとき`object`として扱われるというルールが正しく実装されていないことと、それを正しく修正する方法が不明であることからハンドポートを行いました。

### specification

今回のこれを実装するためにあれこれ調べるまで全然知らなかったのですが、[github.com/elastic/elasticsearch-specification](https://github.com/elastic/elasticsearch-specification)で、Elasticsearchの種々のJSON Documentの型をtypescriptの型定義として実装してあり、これをcompilerでJSONの何かしらのschemaに変換できます。これによって別言語のクライアントを作成するようです。

goの公式clientライブラリである[github.com/elastic/go-elasticsearch]も、[typedapi](https://github.com/elastic/go-elasticsearch/tree/87bb1b42af071454319c73f91c6e5a35e7b6bc5b/typedapi)以下にここから生成されたコードがあります。

https://github.com/elastic/go-elasticsearch/blob/87bb1b42af071454319c73f91c6e5a35e7b6bc5b/typedapi/types/typemapping.go#L33-L53

ここでmappingの型が定義されています。

ではこれを使えば目的が達せられるのかと言えばそうでもないんです。

### go-elasticsearchのspecificationの変換に関する問題

propertiesのデコード部分

https://github.com/elastic/go-elasticsearch/blob/87bb1b42af071454319c73f91c6e5a35e7b6bc5b/typedapi/types/typemapping.go#L152-L447

ここで、`"type"`フィールドが不在の場合がハンドルされていません。

> 引用: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/object.html
>
> You are not required to set the field type to object explicitly, as this is the default value.

ドキュメント曰く、`"type"`がないと`"type":"object"`とみなされます。

さらに悪いことに、PropertyにUnmarshalJSONが実装されているのではなく、Propertyをフィールドに持つ各種のstructにデコードのコードが分散しているため、1か所直したフォーク版をメンテすればいいというものではないようです。

go-elasticsearchのMakefileを見る限り、makeの範疇でこの型の生成を行っているわけではないようです。どう直していいやらわからないためPRも書けません。困りましたね。

### ハンドポート版の実装

幸いなことに生成元のtypescript定義は前述のとおりわかっていますし、go-elasticsearchの各ソースファイルに生成元の定義が載っています。
それさえわかれば後は単純なテキスト置換で実装しなおすことは自体は簡単そうです。

https://github.com/ngicks/estype/tree/main/spec

specというモジュールとして再実装しました。

https://github.com/ngicks/estype/blob/main/spec/mapping/Property.go#L79-L81

こちらでは、ハンドポートであるのでPropertyにUnmarshalJSONが実装される形に変わっています。もちろん`"type"`フィールドの不在もハンドルされています。

https://github.com/ngicks/estype/blob/main/spec/mapping/Property.go#L87-L435

(ちなみにtypedapiの中にははhelper typeで実装したような(rangeのような)型の定義は含まれておらず、無駄な努力をしたわけではなさそうでした。よかったよかった。)

## code generatorの実装

このセクションではcode generator考慮すべきことを述べます。

### dynamic inheritance

mappingの`"dynamic"`の値によって、そのfield dataがmapping.jsonに載っていない値を持てるかが決まります。

> 引用: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/dynamic.html
>
> Inner objects inherit the dynamic setting from their parent object.

とある通り、上位のオブジェクトから値を継承するため、再帰的な型の生成にはcontext情報が必要となります。
nestedも同じく`"dynamic"`の値を継承します。これはElasticsearch 8.4.3相手に確認してあります。ドキュメントに明確に書かれてはいないですが、nested特殊版objectである、という記述はあります。

`"dynamic":"strict"`以外はあればmapping.jsonに載っていないフィールドも受け付けるので、生成されるコードはこれをうまく格納できるフィールドと`MarshalJSON` / `UnmarshalJSON`を実装する必要があります。

### null/multi-valueを許容しない型を考慮する

[Elasticsearchの各field data typeのふるまい](#elasticsearchの各field-data-typeのふるまい)で述べた通り、一部のfield data typeはnullやmulti-valueをがあるとパーズ時にエラーとなります。これらはユーザーが渡す設定値よりも優先されますので、そのような挙動を作ります。

## github.com/dave/jenniferによるcode generation

code generationは[github.com/dave/jennifer]を使用します。

### text/templateを使わない理由

code generationを行うとき真っ先に思いつくのは[text/template](https://pkg.go.dev/text/template)を使う方法です。実際一度は検討しました。

というか1度`text/template`で同じようなコードを書いたことがあるんですよ

https://github.com/ngicks/elastic-type/blob/main/generate/date.go#L197-L295

これ、見ただけで何してるかわかりますか？
これは、mapping.jsonのformat解析済みのデータからdate型の生成を行っています。
ほとんどは平叙なテキストなので、入力された値を取り扱う部分がなければを書いているかはわかりやすいです。
`if`や`range`が二つネストするともうお手上げです。メンテする自信はありません。

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
ドキュメントが明確に述べる通り、data-drivenな使い道が主な用途でしょう。

### github.com/dave/jennifer

[awesome-go](https://github.com/avelino/awesome-go#generators)を見てみるとリストされているもので任意のgo codeの生成を行えるのはjenniferだけですね。
`golang code generation`と検索して出てくるのもjenniferくらいのものです。

### jenniferを使ったcode generation

上記のdate生成の部分をjenniferで書きなおすと以下のようになります。

https://github.com/ngicks/estype/blob/main/generator/genestime/gen.go#L14-L170

うーんネストが深いですね。
実際に生成されるコードと記述順序を一致させようとするとネストが深くなりがちです。ただ、jenniferを利用するとGo codeのトークンにほぼ同じ名前の関数を順番に呼ぶだけなので、書きにくいと感じることはなかったです。

### jenniferのcode generationレシピ

最初に触ってすぐにはわからなかったことを書いていきます。これ別の記事に分けたほうがいいかな・・・

#### 基本

メソッドチェーンで書いていきます。

https://go.dev/play/p/8KuGlxMIjX3

```go
// https://go.dev/play/p/8KuGlxMIjX3
package main

import (
	"io"
	"os"

	"github.com/dave/jennifer/jen"
)

func main() {
	f := jen.NewFile("main")
	f.Func().Id("double").Params(jen.Id("v").Int()).Int().Block(
		jen.Return(jen.Id("v").Op("*").Lit(int(2))),
	).
		Line()

	f.Func().Id("main").Params().Block(
		jen.Qual("fmt", "Println").Call(jen.Id("double").Call(jen.Lit(5))),
	)

	var out io.Writer = os.Stdout
	if err := f.Render(out); err != nil {
		panic(err)
	}
}
```

出力されるコードは

```go
package main

import "fmt"

func double(v int) int {
	return v * 2
}

func main() {
	fmt.Println(double(5))
}
```

#### \*Tと書くとき

```go
jen.Op("*").Id("T")
```

operatorはすべて`Op()`です。何なら`Id("[]string")`や、`Id(*time.Time)`でも問題ありません。

#### forを回しながらコードを生成する

forでsliceやmapをイテレートしながら値に基づいてコードを生成するには、`jen.*Func`を呼び出します。

`if`や`for`の後の`block`は`BlockFunc`、struct宣言には`StructFunc`、structやmapの初期化には`ValuesFunc`といった感じです。

以下前述した収集した型情報から`type FooBar struct {...}`を生成するコードです。

https://github.com/ngicks/estype/blob/main/generator/object.go#L150-L157

#### if err != nil ...を生成する

以下のよく書くやつを生成するには

```go
if err != nil {
  return nil, err
}
```

https://github.com/ngicks/estype/blob/main/generator/additional_prop.go#L82-L84

#### byteリテラルをそのまま書く

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

#### 禁じ手: go codeを直接書く

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
- `null`やmulti-valueを許容しない方に対して、

などをテストしてパスするのを確認しました。長かった・・・。

### cli

cliからも呼び出せるように実行ファイルも作ってあります。

```
# go install github.com/ngicks/estype/cmd/genestype@latest
root@16cb5614efe3:/mnt/git/github.com/ngicks/estype# genestype --help
Usage of genestype:
  -c string
        path to config file.
        see definition of github.com/ngicks/estype/generator.GeneratorOption.
  -m string
        path to mapping.json.
        You can use one that can be fetched from '<index_name>/_mapping',
        or one that you've sent when creating index.
  -o string
        [optional] path to output generated code. (default "--")
  -p string
        package name of generated code.
```

それぞれのフィールドの型名を決定する関数を渡すオプションがありませんが、ほかのオプションはわらせるようになりました。

サンプルの型もこれによって生成されています。

# まとめ

- Elasticsearchの概要について説明した
- Elasticsearchについて、JSONを格納したり引き出したりする場合に必要な知識を調査し、明示した
- 以下を達成する型を作るcode generatorを作成した
  - mapping.jsonからindexに格納されたJSONを容易に生成/消費できる
  - Plain / Rawと二つに分け、アプリの決定事項を反映した型、Elasticsearchが受け入れるすべての値からUnmarshalできる型とそれぞれする
    - 相互を適切に変換するメソッドを設ける
  - `"dynamic"`値が`"strict"`以外の時にmapping.jsonに載っていない数値を格納できる

# おわりに

雑多な知識を詰め込めるだけ詰め込んであるので、またスクロールバーが小指の先ほどの大きさになってしまいました。

- これについて調べなかったら一生[github.com/elastic/elasticsearch-specification](https://github.com/elastic/elasticsearch-specification)の存在を知らなかったかもしれないので、やってよかった
- [github.com/dave/jennifer]によるcode generationがすごく快適で、code generationいつでも任せてくださいって感じになれてうれしい。

今後の課題は

- 実際に使ってみて、使い勝手が悪いかなどを確かめる。

最近Elasticsearchをいじくる業務から離れてしまって使う機会が確保できるか微妙です。

[elasticsearch]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/elasticsearch-intro.html
[ingest pipelines]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/ingest.html
[field data type(s)]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping-types.html
[field data types]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping-types.html
[GeoJSON]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/geo-shape.html#:~:text=GeoJSON%20or%20Well%2DKnown%20Text
[Well-Known Text]: https://docs.opengeospatial.org/is/12-063r5/12-063r5.html
[github.com/elastic/go-elasticsearch]: https://github.com/elastic/go-elasticsearch
[github.com/dave/jennifer]: https://github.com/dave/jennifer
[github.com/ngicks/und]: https://github.com/ngicks/und
[前回の記事]: https://zenn.dev/ngicks/articles/go-json-that-can-be-t-null-or-undefined
[range]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/range.html

<!-- links to field data types -->

[aggregate_metric_double]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/aggregate-metric-double.html
[alias]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/field-alias.html
[binary]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/binary.html
[boolean]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/boolean.html
[completion]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/completion.html
[date]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/date.html
[date_nanos]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/date_nanos.html
[dense_vector]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/dense-vector.html
[flattened]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/flattened.html
[geo_point]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/geo-point.html
[geo_shape]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/geo-shape.html
[histogram]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/histogram.html
[ip]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/ip.html
[join]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/join.html
[keyword]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/keyword.html
[constant_keyword]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/keyword.html#constant-keyword-field-type
[wildcard]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/keyword.html#wildcard-field-type
[nested]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/nested.html
[byte]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/number.html
[double]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/number.html
[float]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/number.html
[half_float]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/number.html
[integer]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/number.html
[long]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/number.html
[scaled_float]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/number.html
[short]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/number.html
[object]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/object.html
[percolator]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/percolator.html
[point]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/point.html
[date_range]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/range.html
[double_range]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/range.html
[float_range]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/range.html
[integer_range]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/range.html
[ip_range]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/range.html
[long_range]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/range.html
[rank_feature]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/rank-feature.html
[rank_features]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/rank-features.html
[search_as_you_type]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/search-as-you-type.html
[shape]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/shape.html
[text]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/text.html
[match_only_text]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/text.html#match-only-text-field-type
[token_count]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/token-count.html
[unsigned_long]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/unsigned-long.html
[version]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/version.html
