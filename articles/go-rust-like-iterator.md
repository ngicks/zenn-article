---
title: "GoでIterator実装"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

# はじめに

Goには長らくgenericsが存在しませんでしたが、1.18([Release Notes](https://tip.golang.org/doc/go1.18#generics))から追加されました。まだstdの公開インターフェイスには確か[`atomic.Pointer[T]`](https://pkg.go.dev/sync/atomic@go1.20.7#Pointer)しかgenericsを使うものはないです。[sync.Mapは内部的に`atomic.Pointer[T]`を使うように変更されています](https://cs.opensource.google/go/go/+/refs/tags/go1.20.7:src/sync/map.go;l=47)がそれ以外だと使われているところはあまり見つかりません。[go 1.21からはslices, maps, cmpなどgenericsを使うパッケージがstdに追加される予定](https://tip.golang.org/doc/go1.21#library)で,[sub-repositories](https://pkg.go.dev/golang.org/x)の[golang.org/x/exp/maps](https://pkg.go.dev/golang.org/x/exp@v0.0.0-20230807204917-050eac23e9de/maps),[golang.org/x/exp/slices](https://pkg.go.dev/golang.org/x/exp@v0.0.0-20230807204917-050eac23e9de/slices)でそれぞれのインターフェイスを試せるみたいです。

そんな対応状況のgenericsですが、それを使ったstd libraryはこれからどんどん増えていくでしょう。これを試すのに何がいいだろうと考えたとき、iteratorが浮かびました。そこで、この記事ではgoでrust風iteratorを実装してみて見つけたポイントについて述べます。

# なぜRust風か？

特に深い理由はないのですが、[awesome-go](https://github.com/avelino/awesome-go)に上がっているものでRust風なinterfaceを持つものがなかったからです。

`HasNext`のようなpeek methodを実装せず、`next`しか持たないことでchannelからもiteratorを作成可能にしたり、無限に要素のあるiteratorも違和感なく扱うことができます。

# RustのIterator

https://doc.rust-lang.org/std/iter/trait.Iterator.html#required-methods

Rustのiteratorは`fn next(&mut self) -> Option<Self::Item>`のみがrequired methodであり、この実装された`next`を使って種々のmethodが使用可能になるというものです。

このiteratorには、例えば[chain](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.chain)や[filter](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter)のようなmethodを呼ぶことで`next`で返ってくる値を加工することができます。

Rustはできる限りデータをstackに乗せられるようにするコンセプトがあるため、これらのmethodは

```rust
// https://doc.rust-lang.org/src/core/iter/traits/iterator.rs.html#922-925
    #[inline]
    #[stable(feature = "rust1", since = "1.0.0")]
    #[rustc_do_not_const_check]
    fn filter<P>(self, predicate: P) -> Filter<Self, P>
    where
        Self: Sized,
        P: FnMut(&Self::Item) -> bool,
    {
        Filter::new(self, predicate)
    }
```

という風に別のstructに包んでそれを返すという実装になります。[FnMut](https://doc.rust-lang.org/std/ops/trait.FnMut.html)はclosure(goで言うところの無名関数; [spec](https://go.dev/ref/spec#Function_literals)でも*closure*という単語で触れられていますね)のことです。Mutとついている通り、キャプチャーした値が可変参照として取られてもよいの意味です。割とgoの`func[T any](v T) bool`と意味が近いですね。

# GoでRust風Iterator

これをGoに翻訳すると、

```go
// Singly ended iterator.
type SeIterator[T any] interface {
	Next() (next T, ok bool)
}
```

となります。[過去の記事](https://zenn.dev/ngicks/articles/go-json-that-can-be-t-null-or-undefined)で`Option[T]`を定義していますが、ここではGoのconventionに添い、第二返り値のboolが`true`のとき`Some`, `false`のとき`None`であるとします。

これをembedしたstructを定義します

```go
type Iterator[T any] struct {
	def.SeIterator[T]
}
```

これに対するadapterを定義します。The Art of Readable Codeに倣い、filterではなくexcluderとselectorという名前にします

```go
type Excluder[T any] struct {
	inner    def.SeIterator[T]
	excluder func(T) bool
}

type Selector[T any] struct {
	inner    def.SeIterator[T]
	selector func(T) bool
}
```

Iterator構造体の中でこれらを呼び出しつつラップしていくことでRustのmethod chainを再現します。

```go
func (iter Iterator[T]) Select(selector func(T) bool) Iterator[T] {
	return Iterator[T]{adapter.NewSelector[T](iter, selector)}
}
func (iter Iterator[T]) Exclude(excluder func(T) bool) Iterator[T] {
	return Iterator[T]{adapter.NewExcluder[T](iter, excluder)}
}
```
