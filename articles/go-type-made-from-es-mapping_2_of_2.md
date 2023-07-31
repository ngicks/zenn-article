---
title: "ElasticsearchのmappingからGoのTypeを作る(2/2)"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Elasticsearch", "go"]
published: true
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

この記事は[part1]の続きで`3.`、`4`について述べます

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

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/test/testdata

それを`genestype`に食わせて以下のコードを生成してあります。

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/test

# 前提知識

以下を有する

- [Go programming language](https://go.dev/)を使って開発が行える程度の理解度。
- [Elasticsearch]とやり取りするアプリケーションを開発できる程度の理解度。
  - indexの作成
  - documentの格納/取得

[part1]でElasticsearchの基本述べているので、それ以上を前提知識としてあります。Goについては一切説明を行いません。

- 本投稿のごく一部で何の説明もなしにTypeScriptの型表記がでてきます。知っている人か、でなければなんとなくで読んでください。

# 対象読者

- Elasticsearchとやり取りするアプリを書いてJSON構造がよくわからなくて困った人
- Goのcode generationで躓きがちなところを知りたい人

# 環境

作り出した時期が大分前なので、elasticsearchは8.4.3を対象に作られています。
ドキュメントもすべて8.4のものを参照しています。

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

# code generatorの作成

[part1]では、Elasticsearchに収めるJSONの注意点や、特定の形のJSONを要求する[field data types]について調べました。この記事ではcode generatorを作ることで、`mapping.json`からElasticsearchのあるindexに格納するJSONを容易に生成/消費できる型を生成します。

そのためには:

- mapping.jsonの解析して型情報を取得し
- ユーザーから設定値を受けとり
- 設定と型情報に基づいてcode generateを行う

以降では`mapping.json`の解析に関する情報と、code generateにおける注意点について述べます。

[part1]でのべた通り、

- `T`と`T[]`
- `undefined`と`null`

などが混在することを許しながら、Goのほかのコードで円滑に消費できるplainでidiomaticな型を生成するのがこのcode generatorの目的です。これらの目的は互いに矛盾するため、それぞれの目的を達成する型をそれぞれ作り、ブリッジとなる相互変換メソッドを設けることとします。

そのため、`Plain`と`Raw`の２つのタイプと、相互に変換を行うメソッドを実装する方針になっています。

`Plain`は以下のサンプル生成コードのように「普通の」Go structのようなものです

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/test/all.go#L14-L58

`Raw`は`undefined | (null | T) | (null | T)[]`を許容するstructです

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/test/all.go#L109-L153

[part1]で説明した`elastic.Elastic[T]`などを使うことでこれを実現します。[前回の記事]で説明した通り`serde`パッケージで`Marshal`した場合のみフィールドのスキップが起きます。

`Plain`はユーザーから設定値を受けとって、`T`、`[]T`、`*T`、`*[]T`のいずれであるかなどを決めます。

## mapping.jsonの解析: specの実装

mapping.jsonを解析するには、mapping.jsonの内容を定義したGo structを定義して`json.Unmarshal()`するか、jsonをパーズせずに手続き的にキーを探索するかなどを行うことになります。

(jsonをパーズしないまま探索を行う場合は、例えば、[github.com/tidwall/gjson](https://github.com/tidwall/gjson)を使います。)

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

https://github.com/elastic/go-elasticsearch/issues/696

go-elasticsearchのMakefileを見る限り、makeの範疇でこの型の生成を行っているわけではないようです。どう直していいやらわからないためPRも書けません。困りましたね。

:::message

執筆中にもう直されてリリースされちゃいました。なのでここに書かれた内容は古いです。

https://github.com/elastic/go-elasticsearch/releases/tag/v8.9.0

@non_exhaustiveタグが付いているデータはプラグインによって拡張されてもよい。そして、Propertyはそのタグが付いているとのことです。

私の書いたハンドポートは全くその辺を考慮してないのでそのうちこちらを使うように修正します。

:::

### ハンドポート版の実装

幸いなことに生成元のtypescript定義は前述のとおりわかっていますし、go-elasticsearchの各ソースファイルに生成元の定義が載っています。
それさえわかれば後は単純なテキスト置換で実装しなおすことは自体は簡単そうです。

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/spec

specというモジュールとして再実装しました。

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/spec/mapping/Property.go#L79-L81

こちらでは、ハンドポートであるのでPropertyにUnmarshalJSONが実装される形に変わっています。もちろん`"type"`フィールドの不在もハンドルされています。

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/spec/mapping/Property.go#L87-L435

(ちなみにtypedapiの中にははhelper typeで実装したような(rangeのような)型の定義は含まれておらず、無駄な努力をしたわけではなさそうでした。よかったよかった。)

## code generatorの実装

このセクションではcode generator考慮すべきことを述べます。

実装の詳細については述べません。今までのセクションと違い、詳細に説明しても、別段ElasticsearchやGoへの理解が深まるわけでもありませんので。GoやElasticsearchや周辺ライブラリの知識が深まりそうなところだけポイントとして説明します。

### dynamic inheritance

mappingの`"dynamic"`の値によって、そのfield dataがmapping.jsonに載っていない値を持てるかが決まります。

> 引用: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/dynamic.html
>
> Inner objects inherit the dynamic setting from their parent object.

とある通り、上位のオブジェクトから値を継承するため、再帰的な型の生成にはcontext情報が必要となります。
nestedも同じく`"dynamic"`の値を継承します。これはElasticsearch 8.4.3相手に確認してあります。ドキュメントに明確に書かれてはいないですが「nestedは特殊版objectである」という記述はあります。

`"dynamic":"strict"`以外はあればmapping.jsonに載っていないフィールドも受け付けるので、生成されるコードはこれをうまく格納できるフィールドと`MarshalJSON` / `UnmarshalJSON`を実装する必要があります。

そこで、`strict`以外の場合、`AdditionalProps_ map[string]any`フィールドを追加し、MarshalJSONはこんな感じ、

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/test/dynamic.go#L123-L171

UnmarshalJSONはこんな感じで生成されます

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/test/dynamic.go#L173-L222

ポイントは

- MarshalJSONの結果であるJSONは`encoding/json`の挙動を模倣する
  - `<`, `>`, `&`のようなhtmlに使われるキーワードを`\u003c`のようにunicode escapeする
  - `Plain`ならば、`,omitempty`タグされているとき、`reflect.Value#IsZero`判定を行ったうえでゼロならフィールドをスキップする
  - `Raw`ならば、`IsUndefined`のときフィールドをスキップする
  - `AdditionalProps_`以外のフィールドは定義順に出力する
    - `mapping.json`解析時に`sort.Strings`でソートされるのでフィールド名をascendingの順です。
  - `AdditionalProps_`はキー名を`sort.Strings`でソートした順序で出力。
    - 有名な話ですが`map[K]V`を`range`オペレータでイテレートするとき、ランダムな順序になるように仕様が定義されています。
    - `encoding/json`は`sort.Strings`でソートすることで結果をstableにします。
- 生成されるGo codeはGoの[FieldDecl]に従い、exportされている必要がある
  - [identifier]になるように、[letter]以外をunicode escapeする
  - `_`がprefixされているとき、exportフィールドにするために`_`-suffixに変換する
  - [Operators and punctuation]を除くようにunicode escapeする

[FieldDecl]: https://go.dev/ref/spec#FieldDecl
[identifier]: https://go.dev/ref/spec#identifier
[letter]: https://go.dev/ref/spec#letter
[Operators and punctuation]: https://go.dev/ref/spec#Operators_and_punctuation

`json.Marshal`の特定文字のunicode escapeと`map`のstable化の挙動はは以下のようになります。

```go
// https://go.dev/play/p/qQdZ_FhJEUp
package main

import (
	"encoding/json"
	"fmt"
)

type Sample struct {
	Foo string `json:"<foo>"`
}

func main() {
	bin, _ := json.Marshal(Sample{Foo: "<bar>&&"})
	fmt.Printf("%s\n", bin)
	bin, _ = json.Marshal(map[string]string{"<foo>": "<bar>&&"})
	fmt.Printf("%s\n", bin)
	bin, _ = json.Marshal(map[string]string{"a": "", "A": "", "b": "", "B": "", "c": "", "C": ""})
	fmt.Printf("%s\n", bin)
}
/*
{"\u003cfoo\u003e":"\u003cbar\u003e\u0026\u0026"}
{"\u003cfoo\u003e":"\u003cbar\u003e\u0026\u0026"}
{"A":"","B":"","C":"","a":"","b":"","c":""}
*/
```

見てのとおり、`map`のキーはunicodeで`ascending`順です(asciiコード表で分かる通り`'A' < 'a'`ですね)

unicode escapeされてても普通はdecode時にunescapeされるっぽいのであんまりこの辺は心配しなくても大丈夫です。すくなくともjavascriptは以下のように`unescape`してくれます。

```
# deno
Deno 1.32.4
exit using ctrl+d, ctrl+c, or close()
REPL is running with all permissions allowed.
To specify permissions, run `deno repl` with allow flags.
> JSON.parse(`{"\u003cfoo\u003e":"\u003cbar\u003e\u0026\u0026"}`)
{ "<foo>": "<bar>&&" }
```

`mapping.json`に記載できる値は有効なjsonならば何でもよいのか、✨のようなemojiでも許されました(前記のとおり8.4のみで確認)。
しかし、Goにおける[FieldDecl]は[IdentifierList](https://go.dev/ref/spec#IdentifierList)であり、[identifier]は

```
identifier = letter { letter | unicode_digit } .
```

[unicode_digit]: https://go.dev/ref/spec#unicode_digit

です。つまり、unicodeのletter categoryと`"_"`とnumber category以外はすべてescapeする必要があります。emojiはLetter categoryではないようです。

Goにはこの辺のことをする処理がぱっと調べた限り`strconv`にいろいろ実装されているのですが、`strings.Builder`と組み合わせて使うには微妙に不都合なので適当に実装しなおしてあります。

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/generate.go#L121-L164

エスケープ処理は`strconv`の中身を見て実装しなおしています。

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/generate.go#L166-L177

utf8は4byteまであり得るので、2byte以下の場合は`\u1234`、それ以上の場合は`\u12345678`になるようにする以外は`MSB`から順にhex encodeするといういつもの奴ですね。

激しい例ですと以下のようにコードを生成します

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/test/additional_prop_escape.go#L16-L20

### null/multi-valueを許容しない型を考慮する

[Elasticsearchの各field data typeのふるまい](https://zenn.dev/ngicks/articles/go-type-made-from-es-mapping_1_of_2#elasticsearchの各field-data-typeのふるまい)で述べた通り、一部のfield data typeはnullやmulti-valueをがあるとパーズ時にエラーとなります。これらはユーザーが渡す設定値よりも優先されますので、そのような挙動を作ります。

そこで、内部的な[field data type]に対応する型を表すための型を作り、そこに上記の各性質を反映するようにフィールドを定義します。

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/typeid.go#L31-L39

code generatorはこれらに基づいてコードを生成します

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/field.go#L52-L87

特別なgeneratorを必要としない[field data type]についてはこのようにテーブルを定義し、そこでまとめて管理するように実装しました。

## github.com/dave/jenniferによるcode generation

code generationは[github.com/dave/jennifer]を使用します。

### text/templateを使わない理由

code generationを行うとき真っ先に思いつくのは[text/template](https://pkg.go.dev/text/template)を使う方法です。実際一度は検討しました。

というか1度`text/template`で同じようなコードを書いたことがあるんですよ

https://github.com/ngicks/elastic-type/blob/879d843a3a21c963793358ca705418f9f3247ea0/generate/date.go#L197-L295

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

とりあえず使ってみましたが、使い心地がよくてAPIも一貫性があります。これ以上のものは探してもないかもしれません。

### jenniferを使ったcode generation

上記のdate生成の部分をjenniferで書きなおすと以下のようになります。

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/genestime/gen.go#L14-L170

うーんネストが深いですね。
実際に生成されるコードと記述順序を一致させようとするとネストが深くなりがちです。ただ、jenniferを利用するとGo codeのトークンと対応づいた名前の関数を順番に呼ぶだけなので、書きにくいと感じることはなかったです。分量が多くなるので書くのは大変です。コードなのでリファクタは簡単でした。

### jenniferのcode generationレシピ

プルダウンの下でjenniferの使い方にいくらか触れます。書くだけ書いて、このセクション過剰に詳細かなと思えてきましたが、記事をいじくっていられる時間が無くなってきたのでとりあえずdetailsに隠します。

:::details jenniferのcode generationレシピ

公式の[README.md](https://github.com/dave/jennifer)が丁寧なので、読めばわかると思います。

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

operatorはすべて`Op()`です。何なら`Id("[]string")`や、`Id("*time.Time")`でも問題ありません。

#### forを回しながらコードを生成する

forでsliceやmapをイテレートしながら値に基づいてコードを生成するには、`jen.Do`もしくは`jen.*Func`を呼び出します。

```go
// https://go.dev/play/p/q-zgkBQwTQC
package main

import (
	"bytes"
	"crypto/rand"
	"encoding/hex"
	"io"
	"os"

	"github.com/dave/jennifer/jen"
)

func main() {
	f := jen.NewFile("main")
	f.Func().Id("main").Params().Block(
		jen.Qual("fmt", "Println").Call(jen.Do(func(s *jen.Statement) {
			for i := 0; i < 3; i++ {
				buf := new(bytes.Buffer)
				_, err := io.CopyN(buf, rand.Reader, 16)
				if err != nil {
					panic(err)
				}
				s.Id(`"` + hex.EncodeToString(buf.Bytes()) + `"`).Op(",")
			}
		})),
	)

	if err := f.Render(os.Stdout); err != nil {
		panic(err)
	}
}

/*
package main

import "fmt"

func main() {
	fmt.Println("99efaa4504a933201846d83dce09967d", "6eec9aca8fb4c381e3ea5a96e5b4d75c", "575fa8f3d669d204071afb06575f0504")
}
*/

```

これはListFuncで書きなおしても同じコードが得られます。

```diff go
-		jen.Qual("fmt", "Println").Call(jen.Do(func(s *jen.Statement) {
+		jen.Qual("fmt", "Println").Call(jen.ListFunc(func(g *jen.Group) {
			for i := 0; i < 3; i++ {
				buf := new(bytes.Buffer)
				_, err := io.CopyN(buf, rand.Reader, 16)
				if err != nil {
					panic(err)
				}
-				s.Id(`"` + hex.EncodeToString(buf.Bytes()) + `"`).Op(",")
+				g.Id(`"` + hex.EncodeToString(buf.Bytes()) + `"`)
			}
		})),
```

こんな感じで、Doの特化版が`Custom/CustomFunc`, さらにそれぞれへの特化版が`BlockFunc`, `StructFunc`, `ValuesFunc`...といった感じのようです。

以下`mapping.json`を解析して収集した`typeId`から`type FooBar struct {...}`を生成するコードです。

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/object.go#L156-L163

#### Custom/CustomFuncをつかう

- Dictを使うと`map[Code]Code`であることからキー順序がソートされる
- Valuesは改行を自動で入れない

ことから`Custom`/`CustomFunc`を使って以下のようにすると手動で`.Line()`を呼ばなくていいぶん楽です。

多分こうするしかないかな？

```go
// https://go.dev/play/p/wkkwsrH6H-p
package main

import (
	"os"

	"github.com/dave/jennifer/jen"
)

func main() {
	f := jen.NewFile("main")

	f.Type().Id("SampleTy").Struct(
		jen.Id("Foo").String(),
		jen.Id("Bar").Int(),
	)

	f.Func().Id("main").Params().Block(
		jen.Qual("fmt", "Println").Call(
			jen.Id("SampleTy").Values(jen.Dict{
				jen.Id("Foo"): jen.Lit("foo"),
				jen.Id("Bar"): jen.Lit(123),
			}),
		),
		jen.Qual("fmt", "Println").Call(
			jen.Id("SampleTy").Values(
				jen.Id("Foo").Op(":").Lit("foo"),
				jen.Id("Bar").Op(":").Lit(123),
			),
		),
		jen.Qual("fmt", "Println").Call(
			jen.Id("SampleTy").Custom(
				jen.Options{Open: "{", Close: "}", Separator: ",", Multi: true},
				jen.Id("Foo").Op(":").Lit("foo"),
				jen.Id("Bar").Op(":").Lit(123),
			),
		),
	)

	if err := f.Render(os.Stdout); err != nil {
		panic(err)
	}
}
/*
package main

import "fmt"

type SampleTy struct {
        Foo string
        Bar int
}

func main() {
        fmt.Println(SampleTy{
                Bar: 123,
                Foo: "foo",
        })
        fmt.Println(SampleTy{Foo: "foo", Bar: 123})
        fmt.Println(SampleTy{
                Foo: "foo",
                Bar: 123,
        })
}
*/
```

#### if err != nil ...を生成する

以下のよく書くやつを生成するには

```go
if err != nil {
  return nil, err
}
```

```go
// https://go.dev/play/p/ms2qGw7Zn27
package main

import (
	"os"

	"github.com/dave/jennifer/jen"
)

func main() {
	f := jen.NewFile("main")
	f.Func().Id("foo").Params().Params(jen.Error(), jen.Error()).Block(
		jen.Var().Defs(
			jen.Err().Error(),
		),
		jen.If(jen.Err().Op("!=").Nil()).Block(
			jen.Return(jen.Nil(), jen.Err()),
		),
	)

	if err := f.Render(os.Stdout); err != nil {
		panic(err)
	}
}

/*
package main

func foo() (error, error) {
	var (
		err error
	)
	if err != nil {
		return nil, err
	}
}
*/
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

:::

### 生成されるコード

以下に置かれたデータを入力に

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/test/testdata

以下のような方が出力されます。

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/generator/test

### テスト

https://github.com/ngicks/estype/blob/cbfaf3aa60e2fb2eaf9a3c25aca2716966d521b1/test.compose.yml

以上のcomposeを使って、elasticsearch 8.4.3相手に

- mapping.jsonでindexを作れるか
- 作られたindexに生成された型のサンプル入力を格納できるか
  - plain, raw両方に対して
- `null`やmulti-valueを許容しない型に対して、許容されない値を出力しないか

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

[part1]で:

- 1. Elasticsearchの概要について説明した
- 2. Elasticsearchについて、JSONを格納したり引き出したりする場合に必要な知識を調査し、明示した

この記事で:

- 3. 以下を達成する型を作るcode generatorを作成した
  - mapping.jsonからindexに格納されたJSONを容易に生成/消費できる
  - Plain / Rawと二つに分け、アプリの決定事項を反映した型、Elasticsearchが受け入れるすべての値からUnmarshalできる型とそれぞれする
    - 相互を適切に変換するメソッドを設ける
  - `"dynamic"`値が`"strict"`以外の時にmapping.jsonに載っていない数値を格納できる
- 4. 作成中に見つけたjenniferによるcode generationのポイントを述べた

# おわりに

- これについて調べなかったら一生[github.com/elastic/elasticsearch-specification](https://github.com/elastic/elasticsearch-specification)の存在を知らなかったかもしれないので、やってよかった
- [github.com/dave/jennifer]によるcode generationがすごく快適で、code generationいつでも任せてくださいって感じになれてうれしい。

今後の課題は

- 実際に使ってみて、使い勝手が悪いかなどを確かめる。
- 似たようなことをしてる人がいないことを祈る
  - いた場合、そちらに貢献する
- いくつかオプションを追加する
  - SkipRaw
  - Omit
  - これらによって`_source`で`Elasticsearch`が返すフィールドの量を減らしとき、それ用の型をそれぞれに生成できる
- QueryDSLのヘルパーも同様に生成する
  - 今回の型生成に比べて見るべきmappingのパラメータが増えるので絶対に時間がかかる
- `Plain`に`Diff(v Plain) Raw`を実装し、[update APIのpartial update](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/docs-update.html#_update_part_of_a_document)で利用しやすくする

最近Elasticsearchをいじくる業務から離れてしまって使う機会が確保できるか微妙です。

まとめきれなくて取り留めのない感じになってしまったのが悔やまれます。誰かの役に立つ文章であることを祈ります。

[elasticsearch]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/elasticsearch-intro.html
[ingest pipelines]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/ingest.html
[field data type]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping-types.html
[field data type(s)]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping-types.html
[field data types]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/mapping-types.html
[GeoJSON]: https://www.elastic.co/guide/en/elasticsearch/reference/8.4/geo-shape.html#:~:text=GeoJSON%20or%20Well%2DKnown%20Text
[Well-Known Text]: https://docs.opengeospatial.org/is/12-063r5/12-063r5.html
[github.com/elastic/go-elasticsearch]: https://github.com/elastic/go-elasticsearch
[github.com/dave/jennifer]: https://github.com/dave/jennifer
[github.com/ngicks/und]: https://github.com/ngicks/und
[前回の記事]: https://zenn.dev/ngicks/articles/go-json-that-can-be-t-null-or-undefined
[part1]: https://zenn.dev/ngicks/articles/go-type-made-from-es-mapping_1_of_2
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
