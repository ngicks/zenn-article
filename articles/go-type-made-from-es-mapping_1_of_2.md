---
title: "ElasticsearchのmappingからGoのTypeを作る(1/2)"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Elasticsearch", "go"]
published: false
---

# Overview

- 1. Elasticsearchの概要について説明する
- 2. Elasticsearchについて、JSONを格納したり引き出したりする場合に必要な知識を調査し、明示する
- 3. 以下を達成する型を作るcode generatorを作成する
  - mapping.jsonからindexに格納されたJSONを容易に生成/消費できる
  - Plain / Rawと二つに分け、アプリの決定事項を反映した型、Elasticsearchが受け入れるすべての値からUnmarshalできる型とそれぞれする
    - 相互を適切に変換するメソッドを設ける
  - `"dynamic"`値が`"strict"`以外の時にmapping.jsonに載っていない数値を格納できる
- 4. 作成中に見つけたjenniferによるcode generationのポイントを述べる

この記事は1．と2.を述べ、3.、4．は後続の記事で述べます。

# 成果物

- Helper Types: [Elasticsearch]にドキュメントとして格納するJSONのフィールドをmarshal / unmarshalする型
- Code Generator: mappingからGoのstructを作るcode generator

を作りました。

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

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/generator/test/testdata

それを`genestype`に食わせて以下のコードを生成してあります。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/generator/test

# 前提知識

以下を有する

- [Go programming language](https://go.dev/)を使って開発が行える程度の理解度。
- [Elasticsearch]とやり取りするアプリケーションを開発できる程度の理解度。
  - indexの作成
  - documentの格納/取得

Elasticsearchついて、基本的な概要からJSONを生成、消費することにかかる説明はこの記事で行いますが十分足りてるかは不明です。Goに関しては一切説明を行いません。

- 本投稿のごく一部で何の説明もなしにTypeScriptの型表記がでてきます。知っている人か、でなければなんとなくで読んでください。

# 対象読者

- Elasticsearchとやり取りするアプリを書いてJSON構造がよくわからなくて困った人
- Elasticsearchのmappingに関する細かい話が知りたい人

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

# 目的

- mapping.jsonを解析してそこからJSONを生成/消費できる型を生成するcode generatorを作成する

# 目標

この記事では、そのために以下を調べます

- Elasticsearchがmappingに対してどのようなJSONを受け付けるのか
  - 逆にどのようなJSONを受け付けないのか
- その中で`string`や`int`や`map[string]any`などで表現できない特定のJSONのサブフィールドを必要とする[field data type(s)]

さらに、

- そのfield data typeを表現する型を作成する

# Elasticsearch

コンテキストを共有するためにElasticsearchの概要について説明します。筆者はElasticsearchに明るくないので基本的には引用元をあたってください。

## 概要

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

格納されたドキュメントはデフォルトでは１秒程度で検索可能状態となり、謳われる通り`near real-time`です。

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

Apache Luceneを包むことでこれらの挙動を実現していると述べており、種々の事情がここから透けてきています。ElasticsearchがShardという単位で管理を行う部分がありますが、これはLucene Indexのことのようです([参考](https://po3rin.com/blog/try-lucene))。

ElasticsearchのIndexというのはRDBで言うところのTableに近く、
同じschemaを共有するドキュメントを格納することができ、indexに対して検索をかけることができます。
このschemaを決めるのがmappingであり、mappingによってIndexに収めるJSON Documentの形を完全に、あるいは部分的に決めることができます。

## dynamic mapping / explicit mapping

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

Elasticsearchはschema-lessで稼働して検索されることも可能です。この場合は[Dynamic field mapping](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/dynamic-field-mapping.html#dynamic-field-mapping)で述べられるように、格納されたJSONの内容から自動的にmappingを推定します。データの形が推定されるために手元にデータをとりあえず検索可能にするなどのユースケースには便利なのかもしれません。

一方で、推定されるために

- [date detection](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/dynamic-field-mapping.html#date-detection)に検知されないフォーマットの時間が時間としてindexされない
- `"true"` / `"false"`がbooleanとしてindexされない

などのドローバックがあると述べられています。

mappingはindex作成時などに[明確に指定することができます。](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/explicit-mapping.html#explicit-mapping)

## mappingの指定/更新

Indexは[Create index API](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/indices-create-index.html)で作成します。このAPIで同時にmappingを指定することができます。このmappingがこの記事で呼ぶ`mapping.json`のことです。

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

引用のように一度作られたindexのフィールド(mapping)は減らすはできません。
追加は[Update mapping API](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/indices-put-mapping.html)により可能です。

## `"dynamic": "strict"`によるmappingの固定

mappingはしばしば完全に固定にされる(`"dynamic":"strict"`)ことがあります。

> 引用: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/dynamic.html#dynamic-parameters
>
> `strict` If new fields are detected, an exception is thrown and the document is rejected. New fields must be explicitly added to the mapping.

推定を防ぐことで後々のmapping拡張を許したり、ill-formedなJSONを検知するなどの目的でこの設定が行われると考えられます。また、自由なフィールド追加を許すと[mapping explosion](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping.html#mapping-limit-settings)という現象が起きる可能性があります。これはindexに任意の入力を許すことによりmappingが増殖してパフォーマンスが劣化することです。

ユーザーの任意な値を収める場合は[flattened]フィールドを使うか、objectのmapping設定で`"index":false`にするなどをしたほうが良いです。

この記事の目的は主に、`"dynamic":"strict"`なmappingからGoの型を生成することで知識をコードにすることです。

# お品書き

`mapping.json`から型を作るcode generatorを作るにあたり以下の作業が必要でした。

1. `undefined | null | T | (null | T)[]`をunmarshalできる型を作る。
2. Elasticsearchに格納できるJSONの形を調査する。
3. Helper typeを実装する: ElasticsearchのField data typesのうち、単なる`string`や`int`などでない型に対するhelperを作成する。
4. date formatの変換: Elasticsearchの理解するtime formatをGoが理解するそれに変換する部分を実装する。
5. code generatorの作成: [github.com/dave/jennifer]を使ったcode generatorを作る。

`4.`, `5.`は後続の記事で述べます。

# `undefined | null | T | (null | T)[]`をunmarshalできる型を作る。

すべてのフィールドは`undefined`であることが許され、ほとんどの[field data type(s)]において、`null`を受け付けます。[Arrays](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/array.html)の項に説明される通り、ほとんどの[field data types]において`T[] | T[][]`も同じく受け付けられます。`T[][]`は`T[]`にflattenされると述べられています。

上記のすべてはオペレーション上混在しうることが考えられるため、これらすべてからunmarshalできる型があるとGoのコードからの扱いがよくなります。(`T[][]`はサポートしないことにしました)

混在する状況として考えらるのは

- 全くデータを指定しない(`{}`)で初期化されたドキュメント
  - すべてのフィールドが`undefined`
- mappingが追加されてフィールドが拡張された場合の、それ以前にindexされたドキュメントのそのフィールドは`undefined`
- フィールドをクリアするために[update APIのpartial update](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/docs-update.html#_update_part_of_a_document)で`[]`、もしくは`null`で上書きされた場合
- サービスで運用されていくうちに、もとは`T`だったフィールドが`T[]`にしたい場合、`T`と`T[]`は混在

`undefined`と`null`、`T`と`T[]`の混在は、[reindex](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/docs-reindex.html#reindex-with-an-ingest-pipeline)とpipelineなどででデータをいじって、[alias](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/aliases.html)をreindex先に切り替えれば無停止でそれぞれ片方に統一できます。しかし、ドキュメント数が多いとメモリやCPU使用率が高い状態が長時間続くため、あえてやりたい理由はないでしょう。

上記の型を実装するには[前回の記事]でも述べた、[github.com/ngicks/und]を拡張し、`Elastic[T]`として実装しました。

https://github.com/ngicks/und/blob/fd0b45653fa93b1bb1ec1928253b563bd1d33eca/elastic/elastic.go#L12-L15

前回の記事で提示した`Undefinedable[T]`と`Nullable[T]`の組み合わせで上記のすべての型を表現することを実現しました。

https://github.com/ngicks/und/blob/fd0b45653fa93b1bb1ec1928253b563bd1d33eca/elastic/elastic.go#L184-L218

`UnmarshalJSON`の実装で`null, T and (null | T)[]`のいずれも受け付けられるようにしてあります。

https://github.com/ngicks/und/blob/fd0b45653fa93b1bb1ec1928253b563bd1d33eca/elastic/elastic.go#L175-L182

とある通り、値がある場合は必ず`T[]`に向けてエンコードされるように設計されています。
値が一つだけの場合に`T`にmarshalすべきか、`T[]`にすべきかは型レベルでは判別のつきづらい要素であったためです。

# Elasticsearchに格納できるJSONの形を調査する

## Elasticsearchのmappingとフィールドの形

前述のとおり、[Elasticsearch]ではIndexごとに格納するJSON documentの形をJSONで表現されrうmappingによって決めることができます。[spec](https://github.com/elastic/elasticsearch-specification/blob/76e25d34bff1060e300c95f4be468ef88e4f3465/specification/_types/mapping/TypeMapping.ts#L34-L56)によればトップは必ずJSON Objectであり、各フィールドは[field data type(s)]を指定することで、格納するデータの型とそれの意味が決められます。

前述のとおり、mappingはindex生成時などに事前に定義することができ、設定によって部分的に、あるいは完全にJSONの形を固定することができます。

mappingはJSONとして`PUT /<index_name>`にsettingとともに渡すことができます。

例えば、以下のようなmappingの場合

```json
{
  "mappings": {
    "dynamic": "strict",
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
- multi-value
  - => 複数値を受け付けるか
- null
  - => `null`をそのフィールドに格納できるか
- single valueのarrayを受け付けるか
  - `dense_vector`以外は許されました。おそらく[Arrays](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/array.html)の項に説明がある通り、`T[][]`は`T[]`にflattenされるからでしょう。

| field data type           | built-in / std       | 要mapping解析 | multi-value | null | 備考                                       |
| ------------------------- | -------------------- | ------------- | ----------- | ---- | ------------------------------------------ |
| [aggregate_metric_double] |                      | ⭕️            | ❌          |      |                                            |
| [alias]                   | `N/A`                | ⭕️            | ❌          | ❌   | いかなる値も受け付けない                   |
| [binary]                  | `[]byte`             |               |             |      |                                            |
| [boolean]                 |                      |               |             |      |                                            |
| [completion]              | `string`             |               |             | ❌   |                                            |
| [date]/[date_nanos]       |                      | ⭕️            |             |      |                                            |
| [dense_vector]            |                      | ⭕️            | ❌          | ❌   | `[[1,2,3]]`は受け付けない                  |
| [flattened]               | `map[string]any`     |               |             |      |                                            |
| [geo_point]               |                      |               |             |      |                                            |
| [geo_shape]               |                      |               |             |      |                                            |
| [histogram]               |                      |               | ❌          |      |                                            |
| [ip]                      | `netip.Addr`         |               |             |      |                                            |
| [join]                    |                      | ⭕️            | ❌          | ❌   |                                            |
| [keyword]                 | `string`             |               |             |      |                                            |
| [constant_keyword]        | `string`             | ⭕️            |             | ❌   | mapping.jsonで入れたワードしか受け付けない |
| [wildcard]                | `string`             |               |             |      |                                            |
| [nested]                  |                      | ⭕️            |             |      | code generatorからすると`object`と同じ     |
| [byte]                    | `int8`               |               |             |      |                                            |
| [double]                  | `float64`            |               |             |      |                                            |
| [float]                   | `float32`            |               |             |      |                                            |
| [half_float]              | `float32`            |               |             |      | goはnativeで`float16`をサポートしない      |
| [integer]                 | `int32`              |               |             |      |                                            |
| [long]                    | `int64`              |               |             |      |                                            |
| [scaled_float]            | `float64`            |               |             |      |                                            |
| [short]                   | `int16`              |               |             |      |                                            |
| [object]                  |                      | ⭕️            |             |      |                                            |
| [percolator]              | `map[string]any`     |               | ❌          | ❌   | QueryDSLを格納する用途                     |
| [point]                   |                      |               |             |      |                                            |
| [date_range]              |                      | ⭕️            |             |      |                                            |
| [double_range]            |                      |               |             |      |                                            |
| [float_range]             |                      |               |             |      |                                            |
| [integer_range]           |                      |               |             |      |                                            |
| [ip_range]                |                      |               |             |      |                                            |
| [long_range]              |                      |               |             |      |                                            |
| [rank_feature]            | `float64`            |               | ❌          |      |                                            |
| [rank_features]           | `map[string]float64` |               |             | ❌   | 同じキーが複数のobjectにあるとエラー       |
| [search_as_you_type]      | `string`             |               |             |      |                                            |
| [shape]                   |                      |               |             |      |                                            |
| [text]                    | `string`             |               |             |      |                                            |
| [match_only_text]         | `string`             |               |             |      |                                            |
| [token_count]             |                      |               |             |      | おそらくfieldプロパティ―以外では使えない？ |
| [unsigned_long]           | `uint64`             |               |             |      |                                            |
| [version]                 | `string`             |               |             |      | semverパッケージ使うほうがいいかもしれない |

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

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/generator/gen_aggregate_metric_double/gen.go#L24-L70

以上のように、フラグのon/offの全パターン網羅は`for`文で容易に実装できます。これによって事前にすべてのパターンを事前に生成しておけばよいのです。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/aggregate_metric_double.go

この型のうちmappingに対し適切なものをcode generatorによって選択してもらえばいいわけですね。

### boolean

[boolean]は以下の値を受けれるとドキュメントされています。

- trueして: `true`, `"true"`
- falseとして: `false`, `"false"`, `""`

boolをbase typeと持つ型とし、`MarshalJSON` / `UnmarshalJSON`を実装すればよいでしょう。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/boolean.go#L70-L83

困ったことに、stringの`"true"` / `"false"`を好むプロジェクトが存在する(筆者が実際に参加していました)ため、`MarshalJSON`で出すのがboolean literalになる型とstring literalになる型をそれぞれ作ってcode generatorの設定値でどちらを使うか決める決断を下しました。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/boolean.go#L10-L17

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/boolean.go#L39-L47

### geo_point

[geo_point]は以下の6つのフォーマットを受け付けます。

- [GeoJSON]
- [Well-Known Text]
- `{"lat":123,"lon":456}`
- `[lon, lat]`
- `"lat,lon"`
- geohash

多いですね。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/geopoint.go#L15-L147

頑張って実装しました。これで少なくとも公式のサンプルを全部パーズできます。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/geopoint.go#L149-L159

この型はシンプルな`{"lat":123,"lon":456}`フォーマットにMarshalします。
boolと同じく、どのフォーマットに対してmarshalするかを設定で決められるようにすればよかったと思いますが、力尽きてしまいました・・・。

### geo_shape

[geo_shape]は[GeoJSON]か[Well-Known Text]を受け付けます。

https://www.elastic.co/guide/en/elasticsearch/reference/8.4/geo-shape.html#input-structure

にある通り、GeoJSON Typeのうち
`"Feature"`と`"FeatureCollection"`以外を受け付けるとあります。

そこで実装は、

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/geoshape.go#L18-L44

特にデータフォーマットを制限したりせず、[github.com/go-spatial/geom](https://github.com/go-spatial/geom)に委譲してしまう実装にしました。内部の実装を読む限り、wktはbboxをサポートしていないのでそれを使われるとデコードできないですが、それ以外は網羅できています。

### histogram

[histogram]はドキュメントによるとalgorithm agnosticな値のセットであり、あらかじめaggregateされている値を入れておくものらしいです。

これはシンプルに

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/histogram.go

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

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/range.go

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

## date helper typeの実装

[date]および[date_nanos] field data
typeは`"format"`フィールドで指定されたフォーマットに従う`string`もしくは`number`を収めることができ、フォーマット通りに解釈されて時間としてインデックスされます。

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

- フォーマットは[DateTimeFormatter](https://docs.oracle.com/javase/8/docs/api/java/time/format/DateTimeFormatter.html)のフォーマットに従う([Cunstom Date Formats](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping-date-format.html#custom-date-formats))
- [`||`区切りの複数のフォーマットを指定できる](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/date.html#multiple-date-formats)
- 特定の文字列(e.g. `strict_date_optional_time`)は[built in format](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping-date-format.html#built-in-date-formats)として認識される
- `epoch_millis`と`epoch_second`を指定すると、Epochからのミリ秒、秒をJSONの`number`で指定できる。
- `"format"`を指定しない場合デフォルトは`"strict_date_optional_time||epoch_millis"`, `"strict_date_optional_time_nanos||epoch_millis"`にそれぞれなる
  - `"strict_date_optional_time"`は`yyyy-MM-dd'T'HH:mm:ss.SSSSSSZ or yyyy-MM-dd`です。
  - と、ドキュメントには載っていますが実際には9桁までデータを格納できます。おそらくIndex時にデータがドロップしています。

したがって以下が必要となります。

- `"format"`フィールドを読んでフォーマットの解析
- built in formatの事前展開展開
- DateTimeFormatterが理解するフォーマットをGoの`time.Parse`が理解するフォーマットに変換
  - optional section ( `[`, `]`)の展開
  - トークンごとの変換
- 複数のフォーマットでパーズができる型を定義
  - Marshal / Unmarshal時、フォーマットに`epoch_*`が含まれている場合numberも解釈する必要がある。

`"format"`フィールドの展開や、built in formatの内容からレイアウト変換などはcode generatorの行いますのでこのセクションでは述べません。

### optional sectionの展開

この機能は`optionalstring`という名前でパッケージにまとめてあります。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/estime/optionalstring

パパっと調べた限り、特定のトークンで囲まれたstringをoptionalとみなして展開し列挙する、というほしい機能を備えたパッケージは見つかりませんでした。
探し方が悪いだけな可能性が高いですが、いいんですこれは趣味プロジェクトなんだから作ってしまえば。

このパッケージは[github.com/prataprc/goparsec](https://github.com/prataprc/goparsec)というパーザコンビネータを利用して文字列を木構造に変更、木構造を展開してoptional sectionなしの文章に列挙します。

このパッケージは事前処理のために使われるため、パフォーマンスは重視されていません。あまり賢い実装をしているとは思えませんし、実際頭がこんがらがりながら木構造を展開する処理をかいていました。実際パーザコンビネータの吐くトークン列から文字列を列挙をすればいいのに木構造に落としなおしている時点で非効率なはずです。

実装の不備やバグは探せばいくらでもあると思いますがdate formatを展開するという用途には現状問題なく動作しています。

### time tokenの変換

こちらは以下のファイルで実装されています。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/estime/convert.go

中身は`time.Parse`を簡易化したような実装をしており、愚直にswitch-caseを書いてパフォーマンスを求めるより、トークンをテーブル化して実装の負担を減らす方針でいきました。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/estime/convert.go#L236-L258

こういったtableを作ることで、switch-caseの量を大分減らせます。

doc commentでも述べていますが、Goが同じ機能を持つトークンを持たない以下はサポートされません

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/estime/convert.go#L28-L41

とくにweekyear系トークンがないのでbuilt in date formatの中にいくつか使えないものが出てきます。

### 複数フォーマットでパーズできる型を定義

#### 複数のstringフォーマットを持つパーザ

これ以下のファイルで実装しました。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/estime/multi_layout.go

これはとっても簡単ですね。

複数のレイアウトを保持

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/estime/multi_layout.go#L9-L13

イニシャライズ時にlengthでdescending, 文字コードでdescendingでソート、dedupe,
validateし、

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/estime/multi_layout.go#L15-L55

順番にパーズを試みて成功したらそのまま値を返します。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/estime/multi_layout.go#L79-L89

これは、Elasticsearch自身のソースを参考にしました。どうやっているんだろう、と思って見に行くと単にパーザーをイテレートしながらパーズを繰り返していたので、なるほど、と思いに似たような処理にしています。

#### numberもパーズ/フォーマットできるパーザ

これは前述のMultiLayoutとnumberを変換できるパーザを組み合わせます。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/estime/estime.go#L51-L54

numberのパーザ/フォーマッタはElasticsearchのそれと一致したstring typeであると非常に楽です。つまり

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/estime/estime.go#L11-L13

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/estime/estime.go#L35-L39

switch-caseによって`time.UnixMilli`と`time.Unix`を呼び出せば所望の動作を実現できます。

https://github.com/ngicks/estype/blob/45f4eb8bad861432af49f2c333975855f2f0b78a/fielddatatype/estime/estime.go#L15-L24

# まとめ

- 1. Elasticsearchの概要について説明した
- 2. Elasticsearchについて、JSONを格納したり引き出したりする場合に必要な知識を調査し、明示した
  - 単に`string`, `int`などにできない型について型を定義し、必要であれば`json.Marshaler`, `json.Unmarshaler`を実装した。
  - [date] / [data_nanos]のために時間フォーマットの変換部と複数レイアウトを保持してパーズができる型を実装した

次回の記事で`mapping.json`から型を生成するcode generatorを実装します。

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
