---
title: "[Go]すべてをiteratorにする"
emoji: "😵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## すべてをiteratorにする

すべては言い過ぎですね。

ソースはすべてここに上がります。

https://github.com/ngicks/go-iterator-helper

この記事ではなるだけiteratorとしてものを利用できるように考えてみたり実装してみたりします。
各種データコンテナやシーケンスデータを返すものをiteratorになるように包んだり、アダプタツールを整えてiteratorでいろんな処理ができるようにします。
なるだけiteratorにする都合上、記事上で言及しておいて「実際使うことは少ないでしょう」というようなコメントを添えているものもあります。

この記事では`Go 1.23.0`を対象バージョンとします。環境は`linux/amd64`ですが、特にOSやarchがかかわる話はしません。

また、この記事は`func(func() bool)`, `iter.Seq[V]`もしくは`func(func(V) bool)`, `iter.Seq2[K,V]`もしくは`func(func(K,V) bool)`のことをカジュアルにiteratorと呼びます。この慣習はstdのドキュメントなどでも同様です。

## iterator

[Go1.23.0](https://tip.golang.org/doc/go1.23)で言語仕様に変更がはいり、for-rangeが以下の三つの関数を受け付けるようになりました。

```
https://go.dev/ref/spec#For_statements

function, 0 values  f  func(func() bool)
function, 1 value   f  func(func(V) bool)              value    v  V
function, 2 values  f  func(func(K, V) bool)           key      k  K            v          V
```

[playground](https://go.dev/play/p/X2rku5_DWaX)

```go
for range func(func() bool) {} {
	// ...
}
```

`func(func() bool)`以外の二つは、`iter`パッケージで定義される[iter.Seq\[V\]](https://pkg.go.dev/iter@go1.23.0#Seq), [iter.Seq2\[K, V\]](https://pkg.go.dev/iter@go1.23.0#Seq2)という型として定義されるので、これを用いると少しわかりやすくなります。

```go
func someIter[K, V any]() iter.Seq2[K, V] {
    return func(yield func(K, V) bool) {
        // ...
    }
}
```

もちろん`iter.Seq[V]`, `iter.Seq2[K,V]`を用いなくてもrange-over-funcは機能するので、`Go1.23.0`以前のversionで作られたライブラリにiteratorを返すメソッドを追加したい場合は、直接`func(func(K, V) bool)`を返したり、受け取ったりするとよいのではないでしょうか。

## stdですでに実装されているもの

### iteratorを返す関数

`go/ast`

```go
// https://pkg.go.dev/go/ast@go1.23.0#Preorder
func Preorder(root Node) iter.Seq[Node]
```

`maps`

```go
// https://pkg.go.dev/maps@go1.23.0#All
func All[Map ~map[K]V, K comparable, V any](m Map) iter.Seq2[K, V]
// https://pkg.go.dev/maps@go1.23.0#Keys
func Keys[Map ~map[K]V, K comparable, V any](m Map) iter.Seq[K]
// https://pkg.go.dev/maps@go1.23.0#Values
func Values[Map ~map[K]V, K comparable, V any](m Map) iter.Seq[V]
```

`reflect`

```go
// https://pkg.go.dev/reflect@go1.23.0#Value.Seq
func (v Value) Seq() iter.Seq[Value]
// https://pkg.go.dev/reflect@go1.23.0#Value.Seq2
func (v Value) Seq2() iter.Seq2[Value, Value]
```

`slices`

```go
// https://pkg.go.dev/slices@go1.23.0#All
func All[Slice ~[]E, E any](s Slice) iter.Seq2[int, E]
// https://pkg.go.dev/slices@go1.23.0#Backward
func Backward[Slice ~[]E, E any](s Slice) iter.Seq2[int, E]
// https://pkg.go.dev/slices@go1.23.0#Chunk
func Chunk[Slice ~[]E, E any](s Slice, n int) iter.Seq[Slice]
// https://pkg.go.dev/slices@go1.23.0#Values
func Values[Slice ~[]E, E any](s Slice) iter.Seq[E]
```

### iteratorを消費する関数

`maps`

```go
// https://pkg.go.dev/maps@go1.23.0#Collect
func Collect[K comparable, V any](seq iter.Seq2[K, V]) map[K]V
// https://pkg.go.dev/maps@go1.23.0#Insert
func Insert[Map ~map[K]V, K comparable, V any](m Map, seq iter.Seq2[K, V])
```

`slices`

```go
// https://pkg.go.dev/slices@go1.23.0#AppendSeq
func AppendSeq[Slice ~[]E, E any](s Slice, seq iter.Seq[E]) Slice
// https://pkg.go.dev/slices@go1.23.0#Collect
func Collect[E any](seq iter.Seq[E]) []E
// https://pkg.go.dev/slices@go1.23.0#Sorted
func Sorted[E cmp.Ordered](seq iter.Seq[E]) []E
// https://pkg.go.dev/slices@go1.23.0#SortedFunc
func SortedFunc[E any](seq iter.Seq[E], cmp func(E, E) int) []E
// https://pkg.go.dev/slices@go1.23.0#SortedStableFunc
func SortedStableFunc[E any](seq iter.Seq[E], cmp func(E, E) int) []E
```

## x/exp/xiter

[proposal: x/exp/xiter: new package with iterator adapters(#61898)](https://github.com/golang/go/issues/61898)で`x/exp/xiter`が提案されています。

以下のようなadapter群が提案されています。

```go
func Concat[V any](seqs ...iter.Seq[V]) iter.Seq[V]
func Concat2[K, V any](seqs ...iter.Seq2[K, V]) iter.Seq2[K, V]
func Equal[V comparable](x, y iter.Seq[V]) bool
func Equal2[K, V comparable](x, y iter.Seq2[K, V]) bool
func EqualFunc[V1, V2 any](x iter.Seq[V1], y iter.Seq[V2], f func(V1, V2) bool) bool
func EqualFunc2[K1, V1, K2, V2 any](x iter.Seq2[K1, V1], y iter.Seq2[K2, V2], f func(K1, V1, K2, V2) bool) bool
func Filter[V any](f func(V) bool, seq iter.Seq[V]) iter.Seq[V]
func Filter2[K, V any](f func(K, V) bool, seq iter.Seq2[K, V]) iter.Seq2[K, V]
func Limit[V any](seq iter.Seq[V], n int) iter.Seq[V]
func Limit2[K, V any](seq iter.Seq2[K, V], n int) iter.Seq2[K, V]
func Map[In, Out any](f func(In) Out, seq iter.Seq[In]) iter.Seq[Out]
func Map2[KIn, VIn, KOut, VOut any](f func(KIn, VIn) (KOut, VOut), seq iter.Seq2[KIn, VIn]) iter.Seq2[KOut, VOut]
func Merge[V cmp.Ordered](x, y iter.Seq[V]) iter.Seq[V]
func Merge2[K cmp.Ordered, V any](x, y iter.Seq2[K, V]) iter.Seq2[K, V]
func MergeFunc[V any](x, y iter.Seq[V], f func(V, V) int) iter.Seq[V]
func MergeFunc2[K, V any](x, y iter.Seq2[K, V], f func(K, K) int) iter.Seq2[K, V]
func Reduce[Sum, V any](f func(Sum, V) Sum, sum Sum, seq iter.Seq[V]) Sum
func Reduce2[Sum, K, V any](f func(Sum, K, V) Sum, sum Sum, seq iter.Seq2[K, V]) Sum
func Zip[V1, V2 any](x iter.Seq[V1], y iter.Seq[V2]) iter.Seq[Zipped[V1, V2]]
func Zip2[K1, V1, K2, V2 any](x iter.Seq2[K1, V1], y iter.Seq2[K2, V2]) iter.Seq[Zipped2[K1, V1, K2, V2]]
type Zipped[V1, V2 any] struct{ ... }
type Zipped2[K1, V1, K2, V2 any] struct{ ... }
```

とりあえず、これらはそのうち実装されるものとして自前実装はいらないということになります。

CLがでてmergeされるまで以下にvendorしておきます。

https://github.com/ngicks/go-iterator-helper/blob/main/x/exp/xiter/xiter.go

シグネチャが変わることもあり得るでしょうし現時点で使うのは時期尚早かなという気もします。

## リファクタ: \[\]V, map\[K\]Vの代わりにiter.Seq\[V\], iter.Seq2\[K, V\]を受けとる

単なるデータシーケンスとして`[]V`と`map[K]V`を受けとっていたとこを`iter.Seq[V]`、`iter.Seq2[K, V]`を受け取るようにリファクタします。

```go
// before

func foo[V any](vs []V) {
    for _, v := range vs {
        // ...
    }
}

func bar[K, V any](mapper map[K]V) {
    for k, v := range mapper {
        // ...
    }
}

// after

func foo[V any](seq iter.Seq[V]) {
    for v := range seq {
        // ...
    }
}

func bar[K, V any](seq iter.Seq2[K, V]) {
    for k, v := range seq {
        // ...
    }
}
```

`[]V`や`map[K]V`を引数に受けていると、別のデータソースを使用したい場合変換しなければならないことや、複数の`[]V`,`map[K]V`を用いたい場合に結合する処理が必要でした。

- `heap.Interface`だろうが`*list.List`だろうが`*ring.Ring`だろうが同じ`iter.Seq[V]`というシグネチャで受け取れます。
- `xiter`の実装で`Concat`, `Concat2`が提案されているので`[]V`や`map[K]V`を結合する処理は書かなくてよくなります。

また、`Go`のrange-over-mapの順序は[言語仕様により未定義](https://go.dev/ref/spec#For_range)であるので、順序が重要なケースでは`map[K]V`をソートするためのcompare funcを受けとることがあると思います。`iter.Seq2[K, V]`を引数に取ると呼び出し側に順序の制御を渡すことができますのでそういったものが不要になります。

## iteratorを返すinterfaceを定義する

- 何度か同じiteratorをデータソースから得たいとか、
- iteratorの初期化のコストが高いので必要になるまで遅延したいとか

そういう時に備えてiteratorを返すinterfaceを定義します。

```go
// Iterable wraps basic Iter method.
//
// Iter should always return iterators that yield same set of data.
type Iterable[V any] interface {
	Iter() iter.Seq[V]
}

// Iterable2 wraps basic Iter2 method.
//
// Iter2 should always return iterators that yield same set of data.
type Iterable2[K, V any] interface {
	Iter2() iter.Seq2[K, V]
}

// IntoIterable wraps basic IntoIter2 method.
//
// Calling IntoIter may mutate underlying state.
// Therefore calling the method again may also not yield same result.
type IntoIterable[V any] interface {
	IntoIter() iter.Seq[V]
}

// IntoIterable2 wraps basic IntoIter2 method.
//
// Calling IntoIter2 may mutate underlying state.
// Therefore calling the method again may also not yield same result.
type IntoIterable2[K, V any] interface {
	IntoIter2() iter.Seq2[K, V]
}
```

データソースとなりうる型にすでにiteratorを返すメソッドが定義されていてなおかつ上記と異なるのは普通にありうると思います。
そこで、関数がinterfaceを満たせるような型を以下に定義します。

```go
var (
	_ Iterable[any]           = FuncIterable[any](nil)
	_ IntoIterable[any]       = FuncIterable[any](nil)
	_ Iterable2[any, any]     = FuncIterable2[any, any](nil)
	_ IntoIterable2[any, any] = FuncIterable2[any, any](nil)
)

type FuncIterable[V any] func() iter.Seq[V]

func (f FuncIterable[V]) Iter() iter.Seq[V] {
	return f()
}

func (f FuncIterable[V]) IntoIter() iter.Seq[V] {
	return f()
}

type FuncIterable2[K, V any] func() iter.Seq2[K, V]

func (f FuncIterable2[K, V]) Iter2() iter.Seq2[K, V] {
	return f()
}

func (f FuncIterable2[K, V]) IntoIter2() iter.Seq2[K, V] {
	return f()
}
```

## K-V pair

[maps.Collect](https://pkg.go.dev/maps@go1.23.0#Collect)で`iter.Seq2[K, V]`を`map[K]V`に受け取れますが、
`map[K]V`にしてしまうとiteratorが生成した順序が保存できなかったりして困るときがある・・・ないかも、ありそう。

またデバッグ目的で順序付きで`iter.Seq2[K, V]`を生成したいため以下のような型を定義すると便利です。

```go
import "iter"

type KeyValue[K, V any] struct {
	K K
	V V
}

// AppendSeq2 appends the values from seq to the KeyValue slice and
// returns the extended slice.
func AppendSeq2[S ~[]KeyValue[K, V], K, V any](s S, seq iter.Seq2[K, V]) S {
	for k, v := range seq {
		s = append(s, KeyValue[K, V]{k, v})
	}
	return s
}

// Collect2 collects values from seq into a new KeyValue slice and returns it.
func Collect2[K, V any](seq iter.Seq2[K, V]) []KeyValue[K, V] {
	return AppendSeq2[[]KeyValue[K, V]](nil, seq)
}

var _ Iterable2[any, any] = KeyValues[any, any](nil)

// KeyValues adds the Iter2 method to slice of KeyValue-s.
type KeyValues[K, V any] []KeyValue[K, V]

func (v KeyValues[K, V]) Iter2() iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {
		for _, kv := range v {
			if !yield(kv.K, kv.V) {
				return
			}
		}
	}
}
```

## 既存のデータシーケンスをiteratorにする

### range

range-over-intが`Go1.22.0`で実装されたことで、`for i := range n {}`という構文で`[0, n)`をiterateすることができます。

が、一方で任意な`[n, m)`な範囲をiterateする方法は特に定義されていないので、以下の`Range`でそれを実現します。

```go
type Numeric interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
		~float32 | ~float64
}

// Range produces an iterator that yields sequential Numeric values in range [start, end).
// Values start from `start` and steps toward `end` 1 by 1,
// increased or decreased depending on start < end or not.
func Range[T Numeric](start, end T) iter.Seq[T] {
	return func(yield func(T) bool) {
		switch {
		default:
			return
		case start < end:
			for i := start; i < end; i++ {
				if !yield(i) {
					return
				}
			}
		case start > end:
			for i := start; i > end; i-- {
				if !yield(i) {
					return
				}
			}
		}
	}
}
```

### Chan

channel `<-chan V`をiteratorに変換できると他のアダプタをそのまま使えるので良いので以下のように定義します。

あんまり使う機会はないかもですね。channelはsynchronizationを目的として使うことが多いですから、そういった目的ではiteratorにする意味が薄いです。

```go
// Chan returns an iterator over ch.
// Either cancelling ctx or closing ch stops iteration.
// ctx is allowed to be nil.
func Chan[V any](ctx context.Context, ch <-chan V) iter.Seq[V] {
	return func(yield func(V) bool) {
		if ctx == nil {
			ctx = context.Background()
		}
		for {
			select {
			case <-ctx.Done():
				return
			case v, ok := <-ch:
				if !ok || !yield(v) {
					return
				}
			}
		}
	}
}
```

### container

stdの`container/heap`, `container/list`, `container/ring`を以下のようにするとiteratorに変換できます。

```go
// Heap returns an iterator over heap.Interface.
// Consuming iter.Seq[T] also consumes h.
// To avoid this, the caller must clone input h before passing to Heap.
func Heap[T any](h heap.Interface) iter.Seq[T] {
	return func(yield func(T) bool) {
		for h.Len() > 0 {
			popped := heap.Pop(h)
			if !yield(popped.(T)) {
				return
			}
		}
	}
}

// ListAll returns an iterator over l.
func ListAll[T any](l *list.List) iter.Seq[T] {
	return ListElementAll[T](l.Front())
}

// ListElementAll returns an iterator over from ele to end of the list.
func ListElementAll[T any](ele *list.Element) iter.Seq[T] {
	return func(yield func(T) bool) {
		for ; ele != nil; ele = ele.Next() {
			if !yield(ele.Value.(T)) {
				return
			}
		}
	}
}

// ListBackward returns an iterator over l,
// traversing it backward by calling Back and Prev.
func ListBackward[T any](l *list.List) iter.Seq[T] {
	return ListElementBackward[T](l.Back())
}

// ListElementBackward returns an iterator over from ele to start of the list.
func ListElementBackward[T any](ele *list.Element) iter.Seq[T] {
	return func(yield func(T) bool) {
		for ; ele != nil; ele = ele.Prev() {
			if !yield(ele.Value.(T)) {
				return
			}
		}
	}
}

// Ring returns an iterator over r.
// by traversing from r and consecutively calling Next.
func RingAll[T any](r *ring.Ring) iter.Seq[T] {
	return func(yield func(T) bool) {
		if !yield(r.Value.(T)) {
			return
		}
		for n := r.Next(); n != r; n = n.Next() {
			if !yield(n.Value.(T)) {
				return
			}
		}
	}
}

// RingBackward returns an iterator over r,
// traversing it backward starting from r and consecutively calling Prev.
func RingBackward[T any](r *ring.Ring) iter.Seq[T] {
	return func(yield func(T) bool) {
		if !yield(r.Value.(T)) {
			return
		}
		for n := r.Prev(); n != r; n = n.Prev() {
			if !yield(n.Value.(T)) {
				return
			}
		}
	}
}
```

### \*sync.Map

`*sync.Map`も同様にiteratorに変換できます。

```go
// SyncMap returns an iterator over m.
// Breaking Seq2 may stop producing more data, however it may still be O(N).
func SyncMap[K, V any](m *sync.Map) iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {
		m.Range(func(key, value any) bool {
			return yield(key.(K), value.(V))
		})
	}
}
```

### Third party: github.com/wk8/go-ordered-map/v2

insertion-ordered mapの実装に筆者は[github.com/wk8/go-ordered-map/v2](httos://github.com/wk8/go-ordered-map)を使ったことがあります。
`map[K]*V`+`*list.List`の組み合わせで実現しています。

https://github.com/wk8/go-ordered-map/pull/41

まだリリースされていませんが、`iter.Seq`, `iter.Seq2`を返すAPIが追加されました。

### Third party: github.com/gammazero/deque

`[]T`ベースのdeque実装に筆者は[github.com/gammazero/deque](https://github.com/gammazero/deque)を用いたことがあります。

こちらは動きがないため`iter.Seq`を返すメソッドの実装はありません。必要になったらPRを出してみようかと思いますが活発ではないかもしれません。

[(\*deque.Deque\[T\]).At(i int) T](https://pkg.go.dev/github.com/gammazero/deque#Deque.At)で各インデックスにアクセス可能です。stdでも`At`メソッドを実装する型に以下の4つがあります。

- https://pkg.go.dev/encoding/asn1@go1.23.0#BitString.At
- https://pkg.go.dev/go/types@go1.23.0#MethodSet.At
- https://pkg.go.dev/go/types@go1.23.0#Tuple.At
- https://pkg.go.dev/go/types@go1.23.0#TypeList.At

そこそこ一般的なメソッドであると判断して以下のように`At`をiteratorに変換する関数を用意します。

```go
type Atter[T any] interface {
	At(i int) T
}

// IndexAccessible returns an iterator over indices and values associated to it in the index-accessible object a.
// If indices generates an out-of-range index, the behavior is not defined and may differs among Atter implementations.
func IndexAccessible[A Atter[T], T any](a A, indices iter.Seq[int]) iter.Seq2[int, T] {
	return func(yield func(int, T) bool) {
		for i := range indices {
			if !yield(i, a.At(i)) {
				return
			}
		}
	}
}
```

これを使って以下のようにiteratorに変換できます。

[playground](https://go.dev/play/p/baXpOo0pGZI)

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"

	"github.com/gammazero/deque"
	"github.com/ngicks/go-iterator-helper/hiter"
)

func main() {
	d := deque.New[int]()
	for i := range 15 {
		d.PushBack(rand.N(10)*100 + i)
	}

	fmt.Printf(
		"%#v\n",
		slices.Collect(hiter.OmitF(hiter.IndexAccessible(d, hiter.Range(3, 12)))),
	)
	// []int{303, 304, 505, 606, 607, 908, 309, 310, 11}
}

```

### Third party: github.com/rsc/omap

[Russ Coxのordered map実装。](https://github.com/rsc/omap)
これは上記の`github.com/wk8/go-ordered-map/v2`と違ってkey type `K`によるorderです。

時期的にrange-over-funcの提案のに合わせて作ったようなので`iter.Seq2[K, V]`などを返すメソッドがあります。

実装を見ると明言されているのでわかりやすいですが、ordered mapと言いつつ[treap](https://en.wikipedia.org/wiki/Treap)ですので`K`によるアクセスは`O(1)`ではなく`O(log n)`となります。かわりに特定のkey値範囲(e.g. `"aaa"`以上`"ccc"`以下のような)の探索などを行えます。

データ構造は色々覚えておくと便利ですねえ。

### io.Reader

`io.Reader`もiteratorにできますね。

[playground](https://go.dev/play/p/t5CGQNtlvf6)

```go
package main

import (
	"crypto/rand"
	"fmt"
	"io"
	"iter"
	mathRand "math/rand/v2"
)

func reader(r io.Reader, buf []byte) iter.Seq2[[]byte, error] {
	return func(yield func([]byte, error) bool) {
		for {
			n, err := r.Read(buf)
			if !yield(buf[:n], err) {
				return
			}
			if err != nil {
				return
			}
		}
	}
}

type randomSizedReader struct {
	r io.Reader
}

func (r *randomSizedReader) Read(p []byte) (int, error) {
	n := mathRand.N(len(p))
	return r.r.Read(p[:n])
}

func main() {
	var size int
	buf := make([]byte, 64)
	for p, err := range reader(&randomSizedReader{rand.Reader}, buf) {
		if err != nil {
			if err == io.EOF {
				break
			}
			panic(err)
		}
		size += len(p)
		fmt.Printf("%x\n", p)
		if size >= 1024 {
			break
		}

	}
}
/*
40151943879a1012bab43d0d27c48cc56859f62399
c3236c5ab41d29cc0823278a96dbc716d25c91270a93d0
b89b0638ec23eff5547b635a96f2aa8397341829a95ac42f33bad138def09fbc4e7b1bedfdb2d0474d7441dd9c70993e9e5e7e
...中略...
af286643dd0bc692addd
a3be5929a084ddd733f28a8d8b11e10bd6029278390eb198e4707b6c70cc2d
b0be743571677a078b5eaf08a1a19f5c783adb372faa8901a13557df09022e15ac522d71d18d662c36f2cd70e9d60d9574f23bd94a
*/
```

javascriptにおける([whatwgのstream spec](https://streams.spec.whatwg.org/#rs-class)の)[ReadableStreamもAsyncIterator](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream#async_iteration)だったりしますね。

ただ`Go`は`io.Reader`周りのツールがしっかりそろっているのでわざわざこういう形にしないかなと思います。

### json/xml Decoder

[\*json.Decoder](https://pkg.go.dev/encoding/json@go1.23.0#Decoder)および[\*xml.Decoder](https://pkg.go.dev/encoding/xml@go1.23.0#Decoder)も以下のようにするとiteratorに変換可能です。

ただし、あまりうれしさはありませんね。
decoderでトークンを読み進めてから`dec.Decode`を呼ぶことで大きなjson arrayを1要素ずつ読み進めるとか、そういったユースケースが普通にあります。
そのためdecoderへのアクセスが常に必要なことが多いからです。

```go
// JsonDecoder returns an iterator over json tokens.
func JsonDecoder(dec *json.Decoder) iter.Seq2[json.Token, error] {
	return func(yield func(json.Token, error) bool) {
		for dec.More() {
			if !yield(dec.Token()) {
				return
			}
		}
		if !yield(dec.Token()) {
			return
		}
	}
}

// XmlDecoder returns an iterator over xml tokens.
// The first non-nil error encountered stops iteration.
// Callers should call [xml.CopyToken] before going to next iteration if they need to retain tokens.
func XmlDecoder(dec *xml.Decoder) iter.Seq2[xml.Token, error] {
	return func(yield func(xml.Token, error) bool) {
		for {
			tok, err := dec.Token()
			if !yield(tok, err) {
				return
			}
			if err != nil {
				return
			}
		}
	}
}
```

## iteratorにできないやつ

逆に、シーケンスではあるが、iteratorにしがたいものがあります。

具体的には以下二つが見つかりました。

- [fs.WalkDir](https://pkg.go.dev/io/fs@go1.23.0#WalkDir)
  - [filepath.WalkDir](https://pkg.go.dev/path/filepath@go1.23.0#WalkDir)も同様
- [io.Pipe](https://pkg.go.dev/io@go1.23.0#Pipe)

どちらも何かの方法で、データを受けた側が、生成する側に、データのスキップなどを指示する方式をとっています。現状のiteratorの仕組みはこの逆向きのシグナルの伝搬を定義していないため、別口の仕組みを実装する必要があります。これは直感的にiteratorにフィットしないのでiteratorにする意味は薄いだろうということです。

[fs.WalkDir](https://pkg.go.dev/io/fs@go1.23.0#WalkDir)は、コールバック関数を渡すと、`fs.FS`の中で見つかった各パスごとにそれを呼び出します。
このコールバック関数は、[fs.SkipDir](https://pkg.go.dev/io/fs@go1.23.0#SkipDir)を返すことでディレクトリをスキップをwalker側に指示することができます。

[io.Pipe](https://pkg.go.dev/io@go1.23.0#Pipe)はin-memory pipeしてreaderとwriterを返し、writerに書き込まれた内容がreaderから読み込雌が、reader/writerどちらからも[CloseWithError](https://pkg.go.dev/io@go1.23.0#PipeReader.CloseWithError)を備え、エラーを片方から片方に伝搬できます。

例えば以下のようにすることで、`iter.Pull`を`io.Pipe`の代わりに使うことができるのですが、
実際にはreader側に`CloseWithError`を実装できなかったため、同等とはいきませんでした。
`Pull`で動いている側にエラーを伝搬する仕組みが構文上備わっていないので、別口で仕組みを作る必要があります。
それをするなら`io.Pipe`などを使ったほうがいいんじゃないかという話です。

```go
type CloserWithError interface {
	io.Writer
	CloseWithError(err error) error
}

func Pipe(fn func(w CloserWithError)) io.ReadCloser {
	next, stop := iter.Pull2(func(yield func(b []byte, err error) bool) {
		fn(yieldWriter(yield))
	})
	return &iterReader{nil, next, stop}
}

type iterReader struct {
	buf []byte
	next func() ([]byte, error, bool)
	stop func()
}

func (r *iterReader) Read(p []byte) (n int, err error) {
	if len(r.buf) == 0 {
		var ok bool
		r.buf, err, ok = r.next()
		if !ok {
			return 0, io.EOF
		}
		if err != nil {
			r.stop()
			return 0, err
		}
	}
	n = copy(p, r.buf)
	if n > 0 {
		r.buf = r.buf[n:]
	}
	return n, nil
}

func (r *iterReader) Close() error {
	r.stop()
	return nil
}

type yieldWriter func(b []byte, err error) bool

func (w yieldWriter) Write(p []byte) (n int, err error) {
	if !w(p, nil) {
		return 0, io.ErrUnexpectedEOF
	}
	return len(p), nil
}

func (w yieldWriter) CloseWithError(err error) error {
	w(nil, err)
	return nil
}
```

`io.Pipe`を丸っと模擬している例なので話のつながりが甘い感じになってしまっている気がします。
reader側に`CloseWithError`を実装してwriter側に伝えるのが自然に行えませんでした。

## adapterでiteratorを加工する。

### Merge sort

真鯵ソート。

普通はこういう風に書きますが、

```go
// implementation of merge sort
// https://en.wikipedia.org/wiki/Merge_sort
func mergeSortFunc[S ~[]T, T any](m S, cmp func(l, r T) int) S {
	if len(m) <= 1 {
		return m
	}
	left, right := m[:len(m)/2], m[len(m)/2:]
	left = mergeSortFunc(left, cmp)
	right = mergeSortFunc(right, cmp)
	return mergeFunc(left, right, cmp)
}

func mergeFunc[S ~[]T, T any](l, r S, cmp func(l, r T) int) S {
	m := make(S, len(l)+len(r))
	var i int
	for i = 0; len(l) > 0 && len(r) > 0; i++ {
		if cmp(l[0], r[0]) < 0 {
			m[i] = l[0]
			l = l[1:]
		} else {
			m[i] = r[0]
			r = r[1:]
		}
	}
	for _, t := range l {
		m[i] = t
		i++
	}
	for _, t := range r {
		m[i] = t
		i++
	}
	return m
}
```

これをiteratorにすると・・・

```go
func mergeSortIterFunc[S ~[]T, T any](m S, cmp func(l, r T) int) iter.Seq[T] {
	if len(m) <= 1 {
		return slices.Values(m)
	}
	return func(yield func(T) bool) {
		left, right := m[:len(m)/2], m[len(m)/2:]
		leftIter := mergeSortIterFunc(left, cmp)
		rightIter := mergeSortIterFunc(right, cmp)
		for t := range xiter.MergeFunc(leftIter, rightIter, cmp) {
			if !yield(t) {
				return
			}
		}
	}
}
```

こうなります！
ただし`xiter.MergeFunc`の中で`iter.Pull`が呼ばれるので効率的なのかは疑問が残ります！
(`iter.Pull`はスケジューラをスキップする特有のコントロールを受ける`goroutine`を作ります。)
新しいsliceをallocateしないので要素数が増えていくにつれてお得になるかもしれません。ベンチとってみてください！

iteratorにするついでに引数が`[]T`なのをやめてもっとジェネリックに

```go
type SliceLike[T any] interface {
	At(i int) T
	Len() int
	Sub(i, j int) SliceLike[T]
}

func mergeSortIterFunc[M SliceLike[T], T any](m M, cmp func(l, r T) int) iter.Seq[T] {
	// ...
}
```

としてもいいんですが、手が込みすぎている気がしたのでやめておきました。
(ただこうすると前記の`github.com/gammazero/deque`や`github.com/rsc/omap`を`[]T`に変換せずにmerge sortできます。)
