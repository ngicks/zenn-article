---
title: "バージョン情報を数値に変換するときは4桁ずつにしてという嘆きと実装"
emoji: "🆕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## バージョン情報を数値に変換するときは4桁ずつにしてという嘆きと実装

データベース上でversionを比較するために単純な数値型に変換するという仕組みが敷かれているとき、`a.b.c.d`の各セグメント(`a`, `b`のような)が3桁までという想定の変換処理が組まれていると、セグメントが4桁なversioningで管理されている何かを突っ込む必要があるとき破綻し、各セグメントを3桁までになるように切り落とす処理と、拡張フィールドに切り落としてないversinonを格納する必要があり、これが運用上ミスしやすいポイントになって苦しいという嘆きです。

各セグメントは4桁までを想定しても特に問題ないはずだということと、そのためのクライアント側のリファレンス実装をぱっと書いて終わります。

## 前提

### 広く使われるversinoning: Sementic Versionning 2.0

現在広く使われているversioningといえば[Sementic Versionning 2.0]\(以後、semverなどと呼ばれる\)があると思います

これは`a.b.c-pre-release+buildmeta`という形式です。
`a.b.c`の`a`,`b`,`c`は数値、`pre-release`と`buildmeta`は`a-z`,`A-Z`,`0-9`,`-`, `.`のみで構成されます。

`buildmeta`はsort orderに影響しないが、`pre-release`はします。

- `a.b.c`の各セグメントは個別に比較され、`a > b > c`の順で重大です
- `pre-release`はdot-separatedなテキストであり、dotで区切られた各セグメントが個別に比較されます
  - 左ほど重大です
  - 数値のみで構成されている場合は数値として比較され
  - テキストが含まれる場合はascii code pointで比較されます
  - `数値のみ < テキストを含む`です
- `pre-release`がないほうが小さいです

つまりspecに乗せられているように下記のような順序になります。

```
https://semver.org/#spec-item-11

Example: 1.0.0-alpha < 1.0.0-alpha.1 < 1.0.0-alpha.beta < 1.0.0-beta < 1.0.0-beta.2 < 1.0.0-beta.11 < 1.0.0-rc.1 < 1.0.0.
```

### 厳密にsemverでないversinoningもよく使われる

- 厳密にsementic versioningでなくてもこの`a.b.c`という形式を採用していることは多いと思います
- `a.b`に`-pre-release`がつくようなバージョンも見たことがあります
  - [synology NASのAssistantアプリ](https://www.synology.com/ja-jp/releaseNote/Assistant)とか
- `a.b.c.d`という4つ型のバージョン情報もあります
  - [QTS OS](https://www.qnap.com/en/download?model=ts-432x&category=firmware)とか

### データベースはsemverを直接扱えることがある

データベースによってはsemverを直接扱えることがあります

- 例えばElasticsearchです([Version field type](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/version.html))
- PostgreSQLには[semver](https://pgxn.org/dist/semver/)というpluginがありますね。使ったことがないので詳細はわかりません。

もちろん`a.b.c.d`というsemverではないversion textはこれらを用いることができません。

### 扱えない時、数値に変換するなどの運用をよく行う

扱えないとき、何かしらの方法でversionを比較可能なものに変えて、それによってindexを張っておくとかしておくことになります。

そもそもデータベースごとにsemverライクなversion textを比較するための拡張関数を作ったらええやんという話はありますがここでは無視します。
そんなのがすっとできる人はそう多くないとしましょう。
(そもそもデータベースの運用やシステム設計をしているのが自分であるとは限りません)
(ただしいつかそういう続編を作る可能性はあります。)

## 問題: a.b.c.dの各セグメントを何桁までと想定するか

問題は比較可能な形にversionを変換するときの処理です。

- 1)`semver`の`x.y.z`の各セグメントを3桁の数字にして、`1000^n`かけていき足し算をする。
  - つまり、`1.2.3`なら`001_002_003`とします。
- 2)`x << 16 + y << 8 + z`のように結合する。

(1)は(2)よりも目で見てわかりやすい良さがあります。

ここで問題になるのがこの`x`、`y`などを何桁までと想定するかです

一般性を加味すると`255`までの制限で事足りるケースが多いと思います。`semver`そのものには制限は課されていませんが、`255`が一般的というような記述があります。

ところが先に挙げたものが([QTS OS](https://www.qnap.com/en/download?model=ts-432x&category=firmware))ちょうどよい典型例となっていますが、

**どこかが4桁になるversinoningが採用されているとこの制限では破綻してしまい**,
結局は3桁まで各セグメントを切り詰めたうえで、拡張フィールドを設けてそこに切り詰めていないversionを格納する必要が生じてしまいます。
これは運用上ミスを起こしやすいポイントとなってしまいますのでできればやりたくありません。

(1)の方法で数値に変換すると仮定するとき、この数値を

- signed integerの64bit型として取り扱うのならば、最大値は`1<<63 - 1 = 9223372036854775807`
- unsigned integer 64bit型として取り扱うのならば、最大値は`1<<64 - 1 = 18446744073709551615`

となります。

それぞれ19桁と20桁となります。int64を採用する場合は`a.b.c.d`の各セグメントは4桁まで、uint64を採用するならば5桁までそれぞれ最大で取れることになります。
もし何かしらの事情でこのversionを加算・減算する必要があるとし、さらに加算しかできないようなケースがある場合([Goの(\*atomic.Int64).Add](https://pkg.go.dev/sync/atomic@go1.24.2#Int64.Add)など)があるとすると、それを考慮したうえでもint64で各セグメントは4桁まで取ることができます。

そもそも各セグメント3桁までとしても`a.b.c.d`で12桁必要となります。
これはusigned integer 32bit型が`1<<32 - 1 = 4294967295` = 10桁となるので12桁格納できる時点で64bit型なんですよね。
なら4桁ずつでいいじゃんという話です。
嘆きの内容は**値域の制限はデータ型の中で取れる最大にしておいてよ**ということになります。

今のところ4桁ずつだったとして問題は思いつかないです。なんかあるでしょうか。

## ということでリファンレス実装: exver

`a.b.c.d`はあまり一般的ではないので既存のプラグインが各種データベース向けに存在するかわかりません。
そこで各種データベース向けの拡張関数を書いいけばこの記事の厚みと有用度合いが増すと思いますが、さすがにそれは骨が折れるのでいったんわきに置いておきます。
ここでは単に運用上さっくりできるクライアント側でデータを加工して投入する方式でリファンレス実装を作ってみます。

実装は[Go]でやりますが、別段[Go]固有な要素は多くないためほかの言語への移植も難しくないと思います。

実装されたものは下記となります。

https://pkg.go.dev/github.com/ngicks/go-common/exver@v1.1.0

### フォーマット

このpackageが想定するversionのテキスト表現のフォーマットを下記のようにします。

```
[v]MAJOR[.MINOR[.PATCH[.EXTRA][-PRERELEASE][+BUILD]]]
```

これは[golang.org/x/mod/semver](https://pkg.go.dev/golang.org/x/mod/semver)に`.EXTRA`を付け足して`v`をoptionalとしたのみでほぼ一緒です。

### VersionとCoreで分ける

`exver`はsemverのように`pre-release`と`buildmeta`とをとれるフォーマットとした一方で
やりたいことはversionの`pre-release`なしの`a.b.c.d`の部分のみを数値に変換することです。

これを呼び出し側に明確に意識してもらうために`a.b.c.d`の部分のみの型と、`pre-release`と`buildmeta`のついたものを型で分けます。

semverの`<version core>`の言い回しになぞらえて`a.b.c.d`の部分を`Core`と呼び、`pre-release`と`buildmeta`をつけたものを`Version`と呼びます。

https://github.com/ngicks/go-common/blob/exver/v1.1.0/exver/ver.go#L17-L29

https://github.com/ngicks/go-common/blob/exver/v1.1.0/exver/ver.go#L257-L270

各セグメントは4桁までであると想定するので`9999`以上であるときエラーにしないといけません。ということで`Core`を初期化するには専用の関数を用いなければならないようにします。

https://github.com/ngicks/go-common/blob/exver/v1.1.0/exver/ver.go#L31-L51

versionのtext表現との相互変換がないと運用上厳しためparserも実装する必要がありますがこれはsemverのspecを愚直に実装するのみです

https://github.com/ngicks/go-common/blob/exver/v1.1.0/exver/ver.go#L572-L904

### Core.Int64でint64に変換できるようにする

これは特にいうことはない気がします。

https://github.com/ngicks/go-common/blob/exver/v1.1.0/exver/ver.go#L167-L179

### pre-releaseもsort可能なstringに変換する

`Core.Int64`で目的は達成できましたが、`pre-release`も同様にsortできるフォーマットに変換できれば、
既存の`semver`で管理されるcomponentも同じ仕組みの上で管理できてよいのでこちらも変換をかけてみます。

https://github.com/ngicks/go-common/blob/exver/v1.1.0/exver/ver.go#L380-L442

前述通り、`pre-release`のsort orderのルールは

- `pre-release`はdot-separatedなテキストであり、dotで区切られた各セグメントが個別に比較されます
  - 手前ほど重大です
  - 数値のみで構成されている場合は数値として比較され
  - テキストが含まれる場合はascii code pointで比較されます
  - `数値のみ < テキストを含む`です
- `pre-release`がないほうが小さいです

です。

これを単純なascii orderでソートできるように変換をかけるためには適当な長さになるように想定を置き、その長さまでpaddingを加えます。

ですので下記のように制限を課します。

- テキストは必ず256 byteになる
  - 256 byteでもasciiなので256文字です。こんな長い`pre-release`は手で書くのは不便なのでここまでやられないだろうという想定です。
- dotで区切られたセグメントは8個まで
  - 現実的にはそんなたくさんかかないだろう想定です
- dotで区切られた各セグメントは31文字まで
  - `alpha123`とか`beta123`とかで8文字以下なのでやはり31文字でも十分余裕があるだろうという想定です。

`<31-letters> "." ...`という繰り返しになるように各セグメントをpaddingします。
こういう方式にすればテキスト表現を目でみたときわかりやすいほか、
最後の一文字はかならず`.`であり、これを切り落とすのは安全となります。
こうすればよくある`VARCHAR(255)`なフィールドにテキストを格納可能です。(固定長なので`CHAR(256)`にいれたらよいですが)

paddingは数値のみなら左に、テキストを含む場合は右に、それぞれ`0`をpadします。

さらに、`pre-release`がついていないversionのほうがsort orderでは後に来るため、この関数はそのとき`~`の繰り返しを返します。`ascii table`上`~`は`DEL`に次いで大きい値であるのでascii textとして比較するとき大きいとして判定されるためです。

## おわりに

versionをデータベース上でsort可能にするために単純なnumeric valueに変換する際に`a.b.c.d`の各セグメントの桁数が3桁だとされているとき破綻するパターンを経験したことがあり、さらに各3桁の場合12桁必要でその場合64bitのintegerを用いることになるため各4桁でも全く問題がないという嘆きについて説明し、
これに対して各セグメントが4桁まで想定される変換処理を含んだ[Version型]()を[Go]で実装しました。

もろもろの都合により`a.b.c.d`なversionを比較したり、生成したりする処理をすることがよくあるため、こういうことができるきれいに作られていた実装が欲しかったため個人的には価値ある活動ができました。
(会社で作る場合はもっとざっくりしたもので十分なのでそこまできれいに作らなかった。)

それが処理を低速にしたり、データをいたずらに肥大化させない限り、想定される値域はデータフォーマットの中で最大になるようにしてくれたらなあという祈りをここに残します。

[Sementic Versionning 2.0]: https://semver.org/
[Go]: https://go.dev/
