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

[以前の記事]で同じテーマについていろいろ述べて解決法までを示しました。それから1年ほどたって知見が筆者の脳内で整理できたり、そもそも大がかりなこと(エンコーダーの用意とか)をしなくても`[]Option[T]`を利用すれば`encoding/json`にオミットされることが可能かつ`T | null | undefined`を表現できる型を定義できることに気付きました。

この記事は[以前の記事]の置き換えを意図しており、それをobsoleteにするものとして書いています。そのためこの記事だけを読めばいいだけになるようにします。
この目的から[以前の記事]とこの記事は大部分が重複し、一部を追加し後半の大分部分を削除するような記事になります。
(ただし前半部分も文章をリファインして`xml`の話を含めるようにしたなどのアップデートをしてあります)
必要に応じて読者には記事をスキップしてほしいと思います。

## Overview(TL;DR)

- [Elasticsearch](のupdate API)のような`JSON`における`null`と`undefined`(`JSON`にフィールドがない)状態をうまく使い分けるシステムに送る`JSON`を structをmarshalするだけでいい感じに作りたい。
- `encoding/json`の挙動を利用し、`map[K]V`もしくは`[]T`をベースとする型を工夫することで可能なことが分かった。
- some/noneを表現できる型として`Option[T]`を定義して、`[]Option[T]`を`T | null | undefined`を表現する型とした。
  - [github.com/oapi-codegen/nullable](https://github.com/oapi-codegen/nullable)は`map[bool]T`として利用するが、`[]T`のほうが動作が速いんじゃないかという仮説があった

## やること

- `Go`で`T | null | undefined`がなぜ表現しにくいかについて説明します
- `T | null | undefined`のユースケースとして`Elasticsearch`のpartial updateを説明します
- 普通、フィールドのあるなしをどうやってチェックするなどをwildに存在する広く使われるライブラリの例を引用し説明します
- 解決法を二つ紹介します。
  - `encoding/json`とすでに互換性のある方法(`[]Option[T]`)
  - `encoding/json/v2`のみで使えるもっと効率的な方法(`Option[Option[T]]`)

## 前提知識

- [Go programming language]の細かい説明はしてますが全体の説明はしないので、ある程度知っている人じゃないと意味が分からないかもしれません。
- [JSON](https://datatracker.ietf.org/doc/html/rfc8259)がどのようなフォーマットであるか。

## 環境

ドキュメントはすべて`Go1.22.5`のものを参照します。

筆者環境は以下のように`Go1.22.0`のままですが、リリースノートを見る限り記事中で言及する挙動に関する変更はなさそうなので特に影響はありません。

```
# go version
go version go1.22.0 linux/amd64
```

`Go 1.18`で追加されたgenericsを用いる以外はバージョン依存な要素はないはずなので、`Go 1.18`ではおおむね通じる話をします。

## 対象読者

- `Go`のstruct fieldで`T | null | undefined`を表現する方法がわからない人
- Go で JSON を受け取る API を組むときに validation などで悩んでいる人
- `encoding/json`のポイントを知りたい人

## 言葉の定義

本投稿では以下のように用語を定めます

| 用語              | 説明                                                                                                                |
| :---------------- | :------------------------------------------------------------------------------------------------------------------ |
| encode            | `Go` valueからバイト列(`[]byte`)への変換                                                                            |
| decode            | バイト列(`[]byte`)から`Go` valueへの変換                                                                            |
| Marshal/Unmarshal | encode/decodeとほぼ同義、ただしencode元/decode先がメソッドをもてる型であるときこの言い回しを好む                    |
| `undefined`       | encode先/decode元のJSON Objectのフィールドが存在しないこと、およびそれを出力しないような`Go` valueの状態のこと      |
| `null`            | encode先/decode元のJSON Objectのフィールドが`null` literalであること、およびそれを出力する`Go` valueの状態のこと    |
| `T`               | encode先/decode元のJSON Objectのフィールドがある型`T`の値に対応すること、およびそれを出力する`Go` valueの状態のこと |

## おさらい: 時たま困る「データがない状態」の扱い

`JSON`などのバイト列(`[]byte`)と`Go` valueの相互変換はしばしばそのフォーマットの持つ特性により、しばしば変換方法が難しいことがあります。

特に`JSON`は`JavaScript Object Notation`(javascriptのobjectの記法)という言い回しからわかる通り、`javascript`の事情を多分に含んでいます。
`javascript`には`null`とは別に`undefined`という「データがない状態」の表現が存在します。
`javascript`を書いているとき、取り扱うほとんどの値が`Object`・・・`Go`で言うと`map[string]any`のようなもの・・・ですので、フィールドがない(`undefined`)のと`null`(`Go`でいうと`nil`)が含まれていることは自然と表現できますし、そもそも値として、型として`undefined`と`null`がどちらも存在するため、「フィールドがない」だけでなく「フィールドに`undefined`がセットされている」というさらに別の状態も存在します。
[JSON の定義](https://datatracker.ietf.org/doc/html/rfc8259)上`undefined`は存在しないのでシリアライズされるときにキーが消える挙動となります。

筆者の知る限りこのような状態を自然に表現できるようにしてあるプログラミング言語はあまりないため、大抵の場合`undefined`と`null`を同一扱いするか、`optional`あるいは`nullable`と言われるようなdata containerを入れ子にした`optional<optional<T>>`にして対応していることがほとんどだと思います。

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

`encoding/json`, `encoding/xml`はstruct tagで`,omitempty`を指定すると`zero value`(厳密にいうと違うが)がオミットされます。

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
	Kwd             []string                               `json:"kwd"`
	Long            []int64                                `json:"long"`
	Text            []string                               `json:"text"`
}
```

ドキュメントは以下のように、`JSON`でpartial documentを送ると当該のフィールドだけを更新することができます。

```
curl -X POST "localhost:9200/test/_update/1?pretty"\
    -H 'Content-Type: application/json'\
    -d '{"doc": {"kwd": "new keyword"}}'
```

さらに、以下のようにフィールドに`null`を指定すると当該のフィールドを`null`に更新することができます。この場合、`null` | `undefined` | `[]`はドキュメントにフィールドがないと同じ扱いなので、フィールドを空にすることができます。

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

### wacky valueを用いればPartial JSONの受け側にはなれる

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

この挙動より、wacky valueを用いればPartial `JSON`の受け側には十分なれます。

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
	for _, input := range []string{
		`{}`,
		`{"Bar": null}`,
		`{"Foo": ""}`,
		`{"Foo": "wacky", "Bar": -9999999}`,
	} {
		s := Sample{
			Foo: &wackyStr,
			Bar: &wackyInt,
		}
		err := json.Unmarshal([]byte(input), &s)
		if err != nil {
			panic(err)
		}
		fmt.Printf("input = %s,\n", input)
		fmt.Printf("unmarshaled = %#v\n", s)
		fmt.Printf("foo was present = %t, bar was present = %t\n", s.Foo != &wackyStr, s.Bar != &wackyInt)
		fmt.Println()
		/*
			input = {},
			unmarshaled = {Foo:"wacky",Bar:-9999999}
			foo was present = false, bar was present = false

			input = {"Bar": null},
			unmarshaled = {Foo:"wacky",Bar:<nil>}
			foo was present = false, bar was present = true

			input = {"Foo": ""},
			unmarshaled = {Foo:"",Bar:-9999999}
			foo was present = false, bar was present = false

			input = {"Foo": "wacky", "Bar": -9999999},
			unmarshaled = {Foo:"wacky",Bar:-9999999}
			foo was present = false, bar was present = false
		*/
	}
}
```

ただし上記の通り、wacky valueと同値の入力を判別できないので`map[string]any`にいったんデコードする方法より筋がいいとはいい難い面があります。

こういう設計をやるとドキュメント上に「`"wacky"`ならば暗黙的に無視される」と載ることになり、すさまじくわかりにくいうえにコードジェネレーターその他と相性が悪くなるのでやらないほうがいいでしょうね。

## Genericsを利用して`T | null | undefined`を表現する型を作る

`T | null | undefined`はつまり1つの型で3つのステートを表現したいわけです。こういったdata container的な型は`Go 1.18`以前では表現しにくかったのですが、以降は[genericsが導入された](https://tip.golang.org/doc/go1.18#generics)ので容易に可能となりました。

### できないパターン: \*\*T

一般的に`present(T)| absent`は`*T`で表現されるわけですから、単純な話`T | null | undefined`には`**T`を用いればいいわけです。実際`**T`自体はOpenSSLなどのAPIで見たことがあります。

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

メソッドの代わりに第一引数に`**T`をとる関数群を定義してもいいんですが、そういう関数群がわかりにくいからメソッドセットをデータ型と結び付ける方法が提供されているはずなのだし、よくない先祖返りな感じがしますね。

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

## 課題: encoding/jsonはstructをオミットしない

stdで`JSON`とバイト列(`[]byte`)の相互変換を行うには`encoding/json`を利用します。

`encoding/json`は`Unmarshal`時、`JSON`フィールドに値がない(`undefined`)時に何も代入をおこなわず、`null`だった場合、`*T`相手には`nil`を代入し、non-pointer type `T`のフィールドには何も代入しません。
また、`json.Unmarshal`は`UnmarshalJSON`を型が実装する場合、それを呼び出すことで型レベルで挙動の変更をサポートしますが、`JSON`フィールドの値が`null`だった場合は、`UnmarshalJSON`を`[]byte("null")`を引数に呼び出します。`Unmarshal`に関しては、型が`UnmarshalJSON`さえ実装しており、`json.Unmarshal`に渡すポインタを毎回zero valueに初期化することで`T | null | undefined`を判別することが可能です。

`Marshal`時には`encoding/json`はstruct tagを参照し、`json`タグに`omitempty`オプションが設定されていると`zero value`(厳密にはempty valueであってzeroではない)であるフィールドをオミット(=出力先データにフィールドが出現しない)する挙動がありますが、これはフィールドの型がstructであるときには起きません。

つまり、`Marshal`時にいかにしてか`undefined`な値をオミットさせる方法が課題となります。

emptyの判定式は以下で行われます

https://github.com/golang/go/blob/go1.22.5/src/encoding/json/encode.go#L306-L318

ここにstructが含まれていないことから、上記がわかると思います。

`,omitempty`がオミットするのは厳密には`zero value`ではないと書いていたのはこれで、structの`zero value`はemptyでないとされるし、slice, mapの場合は`len(v) == 0`の時にemptyとされます。これは`zero value`を含みます(`len([]int(nil))`は0を返す)が、lenが0なnon-zero slice / mapもemptyと判定されます。

また、型レベルなmarshal/unmarshalの挙動の差し替え方法に、型に`MarshalJSON`([json.Marshaler](https://pkg.go.dev/encoding/json@go1.22.5#Marshaler))/`UnmarsahalJSON`([json.Unmarshaler](https://pkg.go.dev/encoding/json@go1.22.5#Unmarshaler))を実装するというものがありますが

> https://pkg.go.dev/encoding/json@go1.22.5#Marshal
>
> ...If an encountered value implements Marshaler and is not a nil pointer, Marshal calls [Marshaler.MarshalJSON] to produce JSON. ...

という記述から、少しわかりにくいですがreceiverが`nil`の時にフィールドをオミットさせるようなことを`MarshalJSON`の実装の中でコントロールさせる方法がありません。

実際上、下記の`encoding/json`のコードを参照するとわかる通り、`MarshalJSON`の呼び出しの時点ですでにフィールド名は書き込まれていますし、`MarshalJSON`は1つの有効な`JSON value`を返すことが(`appendCompact`により)期待されています。

https://github.com/golang/go/blob/go1.22.5/src/encoding/json/encode.go#L698-L704

https://github.com/golang/go/blob/go1.22.5/src/encoding/json/encode.go#L430-L450

## 関連issue

これらの問題はgolang/goのissueにも上がっており、色々な提案がされています。筆者が見たいくつかを以下で述べます。

### encoding/jsonを改善したい系

- https://github.com/golang/go/issues/5901
  - `json.Marshaler`/`json.Unmarshaler`で型レベルではどのようにバイト列(`[]byte`)と相互変換されるかを定義できますが、per-encoder / per-decoderレベルで変更できたほうが便利じゃないですかという提案。
- https://github.com/golang/go/issues/11939
  - structのzero value時にomitemptyを動作させましょうという提案
  - `time.Time`のように、`IsZero`を実装するものはこのメソッドの返り値を見て判別したらよいじゃないかという提案

ただし`encoding/json`は長い歴史があって些細な変更が大きな影響を持つ破壊的変更となってしまうので取り込むのも大変みたいです。

### encoding/json/v2

- https://github.com/golang/go/discussions/63397

`encoding/json`にもコミット履歴がある[dsnet](https://github.com/dsnet)氏(今(2024/07)はTailscale所属と書かれています)の立てたdiscussionで、`encoding/json`のもろもろの欠点と、互換性を保ったままそれらを修正するのが難しい(`JSON`に関するRFCが時間とともに厳密になっていったのに追従してデフォルトの挙動をより厳密にしたほうが良いだろう)という経緯の説明、さらに`v2`のAPIの提案とexperimental実装([github.com/go-json-experiment/json])の紹介がなされています。

この中で`time.Time`のような`IsZero`を実装する型に対してはこれが`true`を返す時オミットする`,omitzero`オプションを含むように提案されています。

これはまだproposalにもなっていない段階で、さらに変更が大きいのでレビューも時間がかかることが予測されます。
とりあえず当面、工夫なしに`T | null | undefined`をstruct fieldで表現する手段は提供されなさそうですので、後述のような方法が必要なようです。

## 没解法

先に没になった解放と没にした理由を述べます。

### structのzero valueをomitするjson encoder/decoder実装を用いる

以下のようなサードパーティのjson encoder/decoder実装はstructのzero valueをオミットする機能を有しています。

- https://github.com/clarketm/json
- https://github.com/json-iterator/go

[以前の記事]では`github.com/json-iterator/go`のほうを採用して、これの[Extension](https://pkg.go.dev/github.com/json-iterator/go#Extension)を駆使して何とかしました。
ただし、このライブラリは`encoding/json`といくつか挙動が違っていたり(筆者自身もいくつか見つけました[#657](https://github.com/json-iterator/go/issues/657))して少し不安になります。

`github.com/clarketm/json`のほうは使ったことがないので何ともですが、どちらに対しても言えるのは、`json.Marshaler`を実装する型が内部で`json.Marshal`を呼び出すとそこ以後でstructをオミットする挙動が起きなくなるので、genericsの導入よって可能になった種々のdata container系の型がネストしたときに不整合が起きます。ここが割とよろしくないわけですね。

できればstdの`encoding/json`のmarshalerの中で事足りる方法であってほしいということです。

### 特定の値をスキップするMarshalJSONを実装する

当然これは可能です。つまり以下のような感じです。

[playground](https://go.dev/play/p/u-LpI1am2Ox)

```go
type zeroStr struct {
	valid bool
	s     string
}

func (s zeroStr) IsZero() bool {
	return !s.valid
}

func (s zeroStr) S() string {
	if !s.valid {
		return ""
	}
	return s.s
}

type Sample struct {
	Foo string
	Bar zeroStr
	Baz int
}

func (s Sample) MarshalJSON() ([]byte, error) {
	var (
		b   bytes.Buffer
		bin []byte
		err error
	)
	b.WriteByte('{')
	b.WriteString("\"foo\":")
	bin, err = json.Marshal(s.Foo)
	if err != nil {
		return nil, err
	}
	b.Write(bin)
	if !s.Bar.IsZero() {
		b.WriteByte(',')
		b.WriteString("\"bar\":")
		bin, err = json.Marshal(s.Bar.S())
		if err != nil {
			return nil, err
		}
		b.Write(bin)
	}
	b.WriteByte(',')
	b.WriteString("\"baz\":")
	bin, err = json.Marshal(s.Baz)
	if err != nil {
		return nil, err
	}
	b.Write(bin)
	b.WriteByte('}')
	return b.Bytes(), nil
}

func main() {
	for _, s := range []Sample{
		{},
		{Bar: zeroStr{valid: true}},
		{Bar: zeroStr{valid: true, s: "bar"}},
	} {
		bin, err := json.Marshal(s)
		if err != nil {
			panic(err)
		}
		fmt.Printf("marshaled = %s\n", bin)
		/*
			marshaled = {"foo":"","baz":0}
			marshaled = {"foo":"","bar":"","baz":0}
			marshaled = {"foo":"","bar":"bar","baz":0}
		*/

	}
}
```

上記のようなコードはすでに煩雑ですので、必要なすべての型に対してそういった`MarshalJSON`を実装するのは手間です。
現実的には[reflect](https://pkg.go.dev/reflect@go1.22.5)パッケージによって動的にこのような処理を行うか、code generatorを実装し、このような`MarshalJSON`を生成することになると思います。

没になった理由はそこで、`reflect`でやろうにもcode generatorを実装しようにも`encoding/json`の挙動はなかなか複雑なので、そもそも実装が煩雑で難しいというのがハードルとなっています。

こういったstructをとって何かのデータ構造に変換をかけるタイプの処理を実装したことがある方はわかるかもしれませんが、`Go`は[struct fieldのembedding](https://gobyexample.com/struct-embedding)が可能で、`encoding/json`の`embed`されたフィールドの取り扱いは

> https://pkg.go.dev/encoding/json@go1.22.5#Marshal
>
> Embedded struct fields are usually marshaled as if their inner exported fields were fields in the outer struct, subject to the usual Go visibility rules amended as described in the next paragraph. An anonymous struct field with a name given in its JSON tag is treated as having that name, rather than being anonymous. An anonymous struct field of interface type is treated the same as having that type as its name, rather than being anonymous.

という感じになります。
embedされたフィールドの型がstructかつ、`json` struct tagが付いていない場合、そのstructのフィールドは親structのフィールドであるかのように出現します。
二つ以上embedされたフィールドがあってそれぞれに同名フィールドがあったときどちらが優先されるか、などなど微妙で面倒でわかりにくくて不具合になりそうな要素がたくさんあります。

さらに面倒なのが、struct fieldのembedで型的な再帰を行うことが許されているんですね。

[playground](https://go.dev/play/p/VWqBtjmi8ar)(実行してコンパイルエラーが起きないことを確かめることができる)

```go
type RecursiveEmbedding struct{
	Foo string
	Recursive
}
type Recursive struct {
	Bar string
	Recursive2
}

type Recursive2 struct {
	*RecursiveEmbedding
}
```

これは、`Tree`を定義するために以下のような型は普通にありえるので許されれているのだと思います。

```go
type Tree[T any] struct {
	node *node[T]
}

type node[T any] struct {
	left, right *node[T] // type recursion
	value T
}
```

さらに、`encoding/json`はstruct tagを使ってJSON Objectのフィールド名と`Go` structのフィールドの対応付けを定義できますので、ここでフィールド名の被りは当然起きえますし、実は`json.Unmarshal`時のフィールド名の比較はcase-insensitiveだったりしてかぶってないつもりで被ってたりもありまえます。

つまり以下のようなエッジケースが存在します。

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

どのフィールドを優先するかのルールは以下で記述されています。

https://github.com/golang/go/blob/go1.22.5/src/encoding/json/encode.go#L1184-L1244

- nameは出力先の`JSON`上でのフィールドの名前です。(つまり、`Go`のstruct field名か`json:"name"`で付けられた名前)
- indexは`[]int`で表現されるフィールドのソースコード順の出現順序です。`len`がstruct fieldのembedの深さを表現し、embedされている数だけappendされます。
- tagはstruct tagでつけられた`json:"name"`があったかどうかです。

`dominantField`は以下のように実装されます。不可思議に感じた上記のエッジケースの挙動はこれによって起きています。

https://github.com/golang/go/blob/go1.22.5/src/encoding/json/encode.go#L1248-L1262

型的な再帰が起きていた場合の挙動は以下のコードによって律せられています

https://github.com/golang/go/blob/go1.22.5/src/encoding/json/encode.go#L1075-L1092

こんなエッジケースはわざわざ探すまで体感することはなかったので、stdはさすが、よく叩かれてよく作られていると感じます。

## 解決法1: map[T]U, []Tはomitemptyでskip可能

`encoding/json`のemptyの判別は以下で行われます。

https://github.com/golang/go/blob/go1.22.5/src/encoding/json/encode.go#L306-L318

`map`, `slice`は`len(v) == 0`時にスキップされるようになっていますね。

ということは`map[T]K`あるいは`[]T`ベースで例えば以下のように

```go
type undefinedableMap map[bool]T

type undefinedableSlice []T
```

こういう型を定義すれば`encoding/json`にオミットされうる型を定義できますね(もちろん`omitempty`オプションは必要です)

> https://pkg.go.dev/builtin@go1.22.5#len
>
> ... Slice, or map: the number of elements in v; if v is nil, len(v) is zero.

とある通り、lenに`nil`を渡すとpanicするとかはないので、`zero value`をそのまま使っても大丈夫です。

[以前の記事]を書いていた時点ではこの方法に全く気付いていませんでした。なんで気付かなかったんだろう・・・

### map[bool]Tを使う実装: [github.com/oapi-codegen/nullable](https://github.com/oapi-codegen/nullable)

`OpenAPI spec`から`Go`のserver/clientを生成するライブラリのソースを管理するoapi-codegenオーがないゼーション以下で、
`map[bool]T`をベースとした`T | null | undefined`を表現できる型が実装されています。

以下がそれぞれの状態に対応します

- `undefined`: `len(m) == 0`
- `null`: `_, ok := m[false]; ok == true`
- `T`: `_, ok := m[true]; ok == true`

`bool`をkey用いれば表現できる状態の数が「キーがない」、「`true`」、「`false`」、「`true`/`false`」の4種のみになります。「`true`/`false`」は未使用とし、他3つをつかって`T | null | undefined`とすればよいわけです。

### []Option[T]も使える

同様に、`[]T`をベースとした実装もできます。こっちは`map[bool]T`と違ってとれる状態の数を制限するような方法はありません。

ただ`[]T`にしたいのは任意のmethod setを持ちながら`encoding/json`にオミットされたいからなだけなので、`undefined`を表現する以外の用途では`T`の実装に工夫をするほうが違和感がないと思います。

なので、

```go
// 前述したOption型。
// Option represents an optional value.
type Option[T any] struct {
	some bool
	v    T
}

type Undefinedable[T any] []Option[T any]
```

とします。

## 解決法2: encoding/json/v2(の候補版を使う)

[github.com/go-json-experiment/json]を使うとstruct fieldに`json:",omitzero"`オプションがついていて、なおかつ`IsZero`メソッドが`true`を返す時、エンコーダーがそのフィールドをオミットする挙動があります。これを利用すれば`[]T`や`map[bool]T`を利用せずとも任意の値をオミットすることができます。

[playground](https://go.dev/play/p/y4pgTPf6WD7)

```go
type Sample struct {
	Padding1 int      `json:",omitzero"`
	V        NonEmpty `json:",omitzero"`
	Padding2 int      `json:",omitzero"`
}

type NonEmpty struct {
	Foo string
}

func (z NonEmpty) IsZero() bool {
	return z.Foo == "foo"
}

func main() {
	var (
		bin []byte
		err error
	)
	bin, err = jsonv2.Marshal(Sample{})
	if err != nil {
		panic(err)
	}
	fmt.Printf("zero = %s\n", bin) // zero = {"V":{"Foo":""}}

	bin, err = jsonv2.Marshal(Sample{V: NonEmpty{Foo: "foo"}})
	if err != nil {
		panic(err)
	}
	fmt.Printf("foo = %s\n", bin) // foo = {}
}
```

そのため、以下のように`Option[Option[T]]`も`IsZero`さえ実装していれば同様に`undefined`時にオミットされることが可能です。

```go
type Und[T any] struct {
	opt option.Option[option.Option[T]]
}

func (u Und[T]) IsZero() bool {
	return u.IsUndefined()
}
```

`option.Option[option.Option[T]]`は`option.Option[T]`が`T`がcomparableである限りcomparableだし、slice/mapであることにかかる色々な処理をバイパスして取り扱えますからおそらくこちらのほうがメモリ的、処理時間的に効率的な実装であると思われます。

ただしこの場合でもdata container系の型が`MarshalJSON`を実装していて内部で`json.Marshal`を呼び出している場合、ここで`IsZero() == true`の時オミットされる挙動が引き継がれなくなります。
ただ`v2`と正式になれば十分な権威がありますから、メンテされているライブラリは`MarshalJSONV2`を実装してくると十分予測はできます。
`v2`に正式になった暁にはこちらの実装を使うほうが良いことになるかもしれませんね。

## 実装

実装物は以下で管理されます

https://github.com/ngicks/und/tree/main

[以前の記事]で述べたrepositoryと同じです。色々事情が変わったので破壊的変更を行い、`jsoniter`への依存などが完全になくなるようになっています。

`encoding/json/v2`が実装されるまで[github.com/go-json-experiment/json]に依存し、`v2`の実装に伴ってその依存を取り消すことで`v1.0.0`となる予定です。
今後は破壊的変更はない予定です。

### Option[T]

`Option[T]`型を実装します。と言ってもこれは前述したものと全く一緒です。

https://github.com/ngicks/und/blob/a63886fe856b790120d5c01b0e2a0613786fb3f7/option/opt.go#L40-L55

`MarshalJSON`, `UnmarshalJSON`,`MarshalJSONV2`, `UnmarshalJSONV2`が実装してあり、`None`は`null`に変換されます。

https://github.com/ngicks/und/blob/v1.0.0-alpha3/option/opt.go#L171-L223

(`// same as bytes.Clone.`っていうコメントは消し忘れなので、なんの意味もないです)

`MarshalJSONV2`向けに`IsZero`が実装してあります

https://github.com/ngicks/und/blob/a63886fe856b790120d5c01b0e2a0613786fb3f7/option/opt.go#L64-L66

この`Option[T]`実装は`Rust`の`std::Option<T>`をミミックしていますが、`Go`には借用など概念がなく、値はすべて`zero value`で初期化されるわけですから、内部の値を取り出すのはもっと単純な仕組みでよいことになります。

https://github.com/ngicks/und/blob/a63886fe856b790120d5c01b0e2a0613786fb3f7/option/opt.go#L85-L89

`T`がcomparableなら`Option[T]`もcomparableですが、`time.Time`のような一部の型は`Equal`メソッドによる比較を必要としますから、`Equal`も実装しておきます。

https://github.com/ngicks/und/blob/a63886fe856b790120d5c01b0e2a0613786fb3f7/option/opt.go#L122-L146

記事の主題とは全く関係ないですが、`Option[T]`には`Rust`の`std::Option<T>`をまねたメソッド群が実装されます

```go
func (o Option[T]) And(u Option[T]) Option[T]
func (o Option[T]) AndThen(f func(x T) Option[T]) Option[T]
func (o Option[T]) Filter(pred func(t T) bool) Option[T]
func FlattenOption[T any](o Option[Option[T]]) Option[T]
func (o Option[T]) IsNone() bool
func (o Option[T]) IsSome() bool
func (o Option[T]) IsSomeAnd(f func(T) bool) bool
func MapOption[T, U any](o Option[T], f func(T) U) Option[U]
func (o Option[T]) Map(f func(v T) T) Option[T]
func MapOrOption[T, U any](o Option[T], defaultValue U, f func(T) U) U
func (o Option[T]) MapOr(defaultValue T, f func(T) T) T
func MapOrElseOption[T, U any](o Option[T], defaultFn func() U, f func(T) U) U
func (o Option[T]) MapOrElse(defaultFn func() T, f func(T) T) T
func (o Option[T]) Or(u Option[T]) Option[T]
func (o Option[T]) OrElse(f func() Option[T]) Option[T]
func (o Option[T]) Xor(u Option[T]) Option[T]
```

これらがあると便利です。(というか複数の`*T`を相手に「このポインターが`nil`なら～」みたいな処理を何度も書いていて煩雑に思ったから`Option[T]`を実装したかったのです)

この手の型は[sql.Scanner](https://pkg.go.dev/database/sql@go1.22.5#Scanner)を実装するかが(体感上)気にされやすいです。そのためシンプルなラッパーで[sql.Scanner](https://pkg.go.dev/database/sql@go1.22.5#Scanner)および[driver.Driver](https://pkg.go.dev/database/sql/driver@go1.22.5#Driver)を実装します。

https://github.com/ngicks/und/blob/v1.0.0-alpha3/option/sql_null.go#L13-L74

そのほかにも[xml.Marshaler](https://pkg.go.dev/encoding/xml@go1.22.5#Marshaler),[xml.Unmarshaler](https://pkg.go.dev/encoding/xml@go1.22.5#Unmarshaler),[slog.LogValuer](https://pkg.go.dev/log/slog@go1.22.5#LogValuer)を実装しておきます。

https://github.com/ngicks/und/blob/v1.0.0-alpha3/option/opt.go#L15-L23

### Und[T] []Option[T]

本題である`[]Option[T]`ベースの`omitempty`でオミット可能な`T | null | undefined`を表現する型です。

https://github.com/ngicks/und/blob/v1.0.0-alpha3/sliceund/slice.go#L48-L63

[以前の記事]では`Undefinedable`という名前にしていましたが型名が長すぎると画面がうるさいので`Und[T]`まで短縮しました。

`len(u) == 0`のときを`undefined`とし、`u[0]`がnoneなら`null`, someなら`T`であるとしています。

https://github.com/ngicks/und/blob/v1.0.0-alpha3/sliceund/slice.go#L105-L119

`Option[T]`と同じく`MarshalJSON`, `UnmarshalJSON`,`MarshalJSONV2`, `UnmarshalJSONV2`を実装しています。
non-zero valueに対して`UnmarshalJSON`が呼ばれるケースもあることを考慮して`len(u) != 0`の場合、index 0に代入するような考慮がされています。

https://github.com/ngicks/und/blob/v1.0.0-alpha3/sliceund/slice.go#L130-L197

これまた記事の主題とは無関係ですが以下のように`Und[T]`も[xml.Marshaler](https://pkg.go.dev/encoding/xml@go1.22.5#Marshaler),[xml.Unmarshaler](https://pkg.go.dev/encoding/xml@go1.22.5#Unmarshaler),[slog.LogValuer](https://pkg.go.dev/log/slog@go1.22.5#LogValuer)を実装してあります。

https://github.com/ngicks/und/blob/v1.0.0-alpha3/sliceund/slice.go#L16-L28

下記の通り、実際に`json.Marshal`によって`undefined`時にフィールドがオミットされる挙動が見れます。

[playground](https://go.dev/play/p/zApld8HqF6m)(go playgroundでサードパーティのライブラリ取得を行うのは時と場合によってはタイムアウトします)

```go
type sliceUnd struct {
	Padding1 int                  `json:",omitempty"`
	V        sliceund.Und[string] `json:",omitempty"`
	Padding2 int                  `json:",omitempty"`
}

func main() {
	for _, input := range []string{
		`{"Padding1":10,"Padding2":20}`,
		`{"Padding1":10,"V":null,"Padding2":20}`,
		`{"Padding1":10,"V":"foo","Padding2":20}`,
	} {
		var s sliceUnd
		err := json.Unmarshal([]byte(input), &s)
		if err != nil {
			panic(err)
		}
		fmt.Printf("unmarshaled = %#v\n", s)
		bin, err := json.Marshal(s)
		if err != nil {
			panic(err)
		}
		fmt.Printf("marshaled = %s\n", bin)
		fmt.Println()
		/*
			unmarshaled = main.sliceUnd{Padding1:10, V:sliceund.Und[string](nil), Padding2:20}
			marshaled = {"Padding1":10,"Padding2":20}

			unmarshaled = main.sliceUnd{Padding1:10, V:sliceund.Und[string]{option.Option[string]{some:false, v:""}}, Padding2:20}
			marshaled = {"Padding1":10,"V":null,"Padding2":20}

			unmarshaled = main.sliceUnd{Padding1:10, V:sliceund.Und[string]{option.Option[string]{some:true, v:"foo"}}, Padding2:20}
			marshaled = {"Padding1":10,"V":"foo","Padding2":20}
		*/

	}
}
```

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

各テストの`V1`, `V2`サフィックスはそれぞれ以下を意味します。

- V1: `encoding/json`+`,omitempty`オプション
- V2: [github.com/go-json-experiment/json]+`,omitzero`オプション

さらに、Serdeの後に続くワードはそれぞれ以下を意味します

- Nullable: [github.com/oapi-codegen/nullable]の`Nullable[T]`型
- Map: 自家版`map[bool]T`実装(なくていいんですが`Nullable[T]`とほぼ同じ実装なので、`go get`せずにベンチで比較するために作ってありました)
- Slice: `[]Option[T]`ベースの`Und[T]`
- NonSlice: `Option[Option[T]]`ベースの`Und[T]`

実行するたび当然数値は変わりますが傾向的に速度の順序はこの通りで入れ替わることはありません。
`map[bool]T`ベース実装より`[]Option[T]`のほうが速いです。ただ現実的なアプリが気にする必要がある差にも思いません。他の重い処理をすればほとんどノイズレベルの差でしかなさそうに思います。

`Option[Option[T]]`ベースの`Und[T]`が最もパフォーマントなのはまあ想像に難くないです。各種slice向けの処理を通らないから`[]Option[T]`よりも早くて当然だといえます。
他の結果も予測どおりです。`NullableV2`と`MapV2`で差がついてるのは`Nullable[T]`が`IsZero`を実装しないからかもしれません。

## おわりに

`JSON`がたびたび持つ`T | null | undefined`を表現する必要性と、なぜ`Go`ではそれが表現しづらいかについて述べました。
その後、可能だがとらなかった方法について述べ、`[]Option[T]`をベースとする実装のしかたについて説明し、
最後にベンチマークをとって`[]Option[T]`が`map[bool]T`に比べて若干パフォーマントであることを示しました。

筆者が`Node.js`で書いていた`Elasticsearch`の前に立つサーバーアプリケーションをどうやって`Go`に移植すればよいのだろうか疑問からから始まった探索でしたが、
現実的で扱える方法が見つかったことで、一旦終わりということになります。
記事中では特に触れていなかったですが、`Elasticsearch`に格納する`JSON`向けの`undefined | null | T | [](null | T)`を表現できる型も[作成済み](https://github.com/ngicks/und/blob/v1.0.0-alpha3/sliceund/elastic/elastic.go)です。
しかし残念ながら、筆者はそういったアプリを実際に移植することはなさそうなのであまりこの成果を生かせなさそうです。

[Go programming language]: https://go.dev/
[以前の記事]: https://zenn.dev/ngicks/articles/go-json-that-can-be-t-null-or-undefined
[Elasticsearch]: https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro.html
[github.com/oapi-codegen/nullable]: https://github.com/oapi-codegen/nullable
[github.com/go-json-experiment/json]: https://github.com/go-json-experiment/json
