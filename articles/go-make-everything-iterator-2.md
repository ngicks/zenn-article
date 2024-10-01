---
title: "[Go]続・なるだけすべてをiteratorにする"
emoji: "😵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## 続・なるだけ~~すべてを~~iteratorにする

[前回の記事:\[Go\]なるだけすべてをiteratorにする](https://zenn.dev/ngicks/articles/go-make-everything-iterator)を書いた後にもいろいろ考えたので続き的な記事です。

前回から引き続きソースコードはすべて以下に上がります。

https://github.com/ngicks/go-iterator-helper

前回から引き続き、この記事ではなるだけiteratorとしてものを利用できるように考えてみたり実装してみたりします。
各種データコンテナやシーケンスデータを返すものをiteratorになるように包んだり、アダプタツールを整えてiteratorでいろんな処理ができるようにします。
なるだけiteratorにする都合上、記事上で言及しておいて「実際使うことは少ないでしょう」というようなコメントを添えているものもあります。

この記事では`Go 1.23.0`を対象バージョンとします。環境は`linux/amd64`ですが、特にOSやarchがかかわる話はしません。

また、この記事は`func(func() bool)`, `iter.Seq[V]`もしくは`func(func(V) bool)`, `iter.Seq2[K,V]`もしくは`func(func(K,V) bool)`のことをカジュアルにiteratorと呼びます。この慣習はstdのドキュメントなどでも同様です。

前回の記事からさらにエラーハンドリング、別goroutineでの処理などについて考えています。

## おさらい

[前回の記事:\[Go\]なるだけすべてをiteratorにする](https://zenn.dev/ngicks/articles/go-make-everything-iterator)のおさらいです。

### spec

[Go1.23.0](https://tip.golang.org/doc/go1.23)で言語仕様に変更がはいり、for-rangeが以下の三つの関数を受け付けるようになりました。

```
https://go.dev/ref/spec#For_statements

function, 0 values  f  func(func() bool)
function, 1 value   f  func(func(V) bool)              value    v  V
function, 2 values  f  func(func(K, V) bool)           key      k  K            v          V
```

なので下記は構文として合法であるのでコンパイルします。

[playground](https://go.dev/play/p/X2rku5_DWaX)

```go
for range func(func() bool) {} {
    // ...loop body...
}
```

仕組みとしては以下のように、loop-bodyを関数に変換してそれを引数にiteratorを呼び出すようにrewriteされれます([ここ](https://github.com/golang/go/blob/go1.23.0/src/cmd/compile/internal/rangefunc/rewrite.go))。

```go
func(func() bool) {}(func() bool {
    // ...loop body...
    return true
})
```

`func(func() bool)`以外の二つは、`iter`パッケージで[iter.Seq\[V\]](https://pkg.go.dev/iter@go1.23.1#Seq), [iter.Seq2\[K, V\]](https://pkg.go.dev/iter@go1.23.1#Seq2)という型として定義されるので、これを用いるとよいです。

```go
func someIter[K, V any]() iter.Seq2[K, V] {
    return func(yield func(K, V) bool) {
        // ...
    }
}
```

もちろん`iter.Seq[V]`, `iter.Seq2[K,V]`を用いなくてもrange-over-funcは機能するので、`Go1.23.0`以前のversionで作られたライブラリにiteratorを返すメソッドを追加したい場合は、直接`func(func(K, V) bool)`を返すとよいでしょう。

### 利点

iterator導入の目的の一つとしてデータシーケンスの繰り返し表現に統一性を持たせることというのがあります。

以下のような感じでシーケンスデータをiterateする方法はライブラリや型ごとに違ったインターフェイスが実装されています。

[playground](https://go.dev/play/p/CWIQqHyOLyQ)

```go
// *bufio.Scanner
for scanner.Scan() {
    // scanner.Text()
}
if scanner.Err() != nil {
    // err
}

// *container/list.List
for ele := s.Front(); ele != nil; ele = ele.Next() {
    // ele.Value
}

// *sync.Map
m.Range(func(k, v any) bool {
    // k, v
    return true
})
```

大体この3つのパターンが多いかな。

- `Scan`や`Next`メソッドで次のデータを準備し、データが存在するとき`true`を返すパターン
  - データそのものは構造体の`Text`, `Scan`などを呼び出して得る。
  - セットで`Err`メソッドを備えていることが多い
  - 下層に`io.Reader`を持っていたり、データベースアクセスを持っていたりする時にこのパターンになるように思います。
- `Front`や`Next`などで同じ型のポインターを返し、`nil`になるなどするまで繰り返すパターン
  - データはすべてインメモリで保持されていてポインターを差し替えることで効率的に操作が行えるタイプのデータ構造ならこのパターンになる印象です。
- `Range`や`Visit`関数にvisitorとなる関数を渡し、すべてのデータに対して関数を呼び出すパターン
  - 内部の構造がopaqueで、ポインターの参照関係で表現されるわけではないときにこうなると思われる。
  - visitorに返り値がなく、必ずすべてにvisitするものもあります。これはおそらく要素数が有限で多くないという想定があります。

この話のスコープはさらにGo1.18で追加されたgenericsによって現れたであろうdata containerライブラリにも広がります。
例えば以下の[github.com/wk8/go-ordered-map](https://pkg.go.dev/github.com/wk8/go-ordered-map/v2)は以下のようにiterateします。

```go
// *(github.com/wk8/go-ordered-map/v2).OrderedMap
for pair := om.Oldest(); pair != nil; pair = pair.Next() {
    // pair.Key, pair.Value
}
```

これは内部的に`container/list.List`を用いるため、listと同じやり口になっています。
ライブラリごとに方法が異なっていると使いづらいなーというわけですね。

さらにいうと、以下のようにfor-range可能な型を複数制約に含めてfor-rangeにかけようとすると`no core type`エラーでコンパイルできないため、`range-over-range`仕様の追加はこの面でも助かります。

[playground](https://go.dev/play/p/kejmf4uAGAo)

```go
func Foo[I []V | chan V, V any](i I) {
    // ./prog.go:4:12: cannot range over i (variable of type I constrained by []V | chan V): no core type
    for range i {
	}
}

```

### stdですでに実装されているもの

#### iteratorを返す関数

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

#### iteratorを消費する関数

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

### stdだけど未リリースのもの

- [proposal: regexp: add iterator forms of matching methods(#61902)](https://github.com/golang/go/issues/61902)
- [bytes, strings: add iterator forms of existing functions (#61901)](https://github.com/golang/go/issues/61901)

### x/exp/xiter

[proposal: x/exp/xiter: new package with iterator adapters(#61898)](https://github.com/golang/go/issues/61898)で`x/exp/xiter`が提案されています。

以下のようなadapter群が提案されています。(`2024-10-01`現在)

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

## 既存のデータソースをiteratorにする

前回の記事と重複あります。

### Range: [n, m)

ほかの言語だとよくある`n..m`の代わり。

```go
type Numeric interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
		~float32 | ~float64
}

// Range produces an iterator that yields sequential Numeric values in range [start, end).
// Values start from `start` and step toward `end`.
// At each step value is increased by 1 if start < end, otherwise decreased by 1.
func Range[T Numeric](start, end T) iter.Seq[T]
```

### Step

`Range`と近いようで微妙に違うものとして`Step`があります。

```go
// Step returns an iterator over numerics values starting from initial and added step at each step.
// The iterator iterates forever. The caller might want to limit it by [xiter.Limit].
func Step[N Numeric](initial, step N) iter.Seq[N] {
	return func(yield func(N) bool) {
		for n := initial; ; n += step {
			if !yield(n) {
				return
			}
		}
	}
}

// StepBy returns an iterator over pair of index and value associated the index.
// The index starts from initial and steps by step.
func StepBy[V any](initial, step int, v []V) iter.Seq2[int, V] {
	return func(yield func(int, V) bool) {
		if initial < 0 {
			return
		}
		for i := initial; 0 <= i && i < len(v); i += step {
			if !yield(i, v[i]) {
				return
			}
		}
	}
}
```

`Rust`の[core::iter::Iteratorにstep_byメソッドがあります](https://doc.rust-lang.org/beta/core/iter/trait.Iterator.html#method.step_by)。
筆者は使ったことはないんですが`Go`にもあるといいのかなあと思って実装しておきます。

### Once, Empty

1度だけ値を返すiteratorも欲しくなるので定義しておきましょう。これは主に`xiter.Concat`などとともに使われると思われます。
また、値を何も返さないiteratorも欲しくなります。iteratorは関数ですから単に`nil`を渡してしまうと`nil`関数の呼び出しでpanicしてしまうことがあるためですね。

```go
// Once adapts a single value as an iterator;
// the iterator yields v and stops.
func Once[V any](v V) iter.Seq[V] {
	return func(yield func(V) bool) {
		yield(v)
	}
}

// Once2 adapts a single k-v pair as an iterator;
// the iterator yields k, v and stops.
func Once2[K, V any](k K, v V) iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {
		yield(k, v)
	}
}

// Empty returns an iterator over nothing.
func Empty[V any]() iter.Seq[V] {
	return func(yield func(V) bool) {}
}

// Empty2 returns an iterator over nothing.
func Empty2[K, V any]() iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {}
}
```

### Repeat / RepeatFunc

単一要素の繰り返し、単一関数の繰り返しをiterator扱いしたいことはよくあるので実装しておきます。

```go
// Repeat returns an iterator over v repeated n times.
// If n < 0, the returned iterator repeats forever.
func Repeat[V any](v V, n int) iter.Seq[V] {
	if n < 0 {
		return func(yield func(V) bool) {
			for {
				if !yield(v) {
					return
				}
			}
		}
	}
	return func(yield func(V) bool) {
		// no state in the seq.
		for n := n; n != 0; n-- {
			if !yield(v) {
				return
			}
		}
	}
}

// RepeatFunc returns an iterator that generates result from fnV n times.
// If n < 0, the returned iterator repeats forever.
func RepeatFunc[V any](fnV func() V, n int) iter.Seq[V]  {
	if n < 0 {
		return func(yield func(V) bool) {
			for {
				if !yield(fnV()) {
					return
				}
			}
		}
	}
	return func(yield func(V) bool) {
		for n := n; n != 0; n-- {
			if !yield(fnV()) {
				return
			}
		}
	}
}
```

`n < 0`のケースでも`n--`し続けるといつかアンダーフローしてしまうため本当に無限ループにするために`n < 0`の時は別なiteratorを返すようになっています。実際の運用上ありうるのかはわかりません。

`iter.Seq2[K, V]`版も当然実装してあります。

```go
// Repeat2 returns an iterator over the pair of k and v repeated n times.
// If n < 0, the returned iterator repeats forever.
func Repeat2[K, V any](k K, v V, n int) iter.Seq2[K, V]
// RepeatFunc2 returns an iterator that generates result of fnK and fnV n times.
// If n < 0, the returned iterator repeats forever.
func RepeatFunc2[K, V any](fnK func() K, fnV func() V, n int) iter.Seq2[K, V]
```

### chan V

`chan V`そのものはすでにfor-rangeで処理可能ですが、他のアダプタに渡せるようにするために`iter.Seq[V]`に変換できたほうが何かと便利です。

```go
// Chan returns an iterator over ch.
// Either cancelling ctx or closing ch stops iteration.
func Chan[V any](ctx context.Context, ch <-chan V) iter.Seq[V] {
	return func(yield func(V) bool) {
		for {
			select {
			case <-ctx.Done():
				return
			default:
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
}
```

テストがたまに大変になるのでctxのキャンセレーションが優先的にチェックされます。
筆者が未熟なだけかもしれないですが、タイミングが絡むライブラリのテストでrace conditionを防ぎきる辛さが骨身に沁みついているのでパフォーマンスが多少落ちてもこういうガードを外す決断が下せない。

### string

[bytes, strings: add iterator forms of existing functions (#61901)](https://github.com/golang/go/issues/61901)がリリースされると`bytes`, `strings`パッケージ以下にiteratorを返す関数が追加されるので必要性はその時に大幅低下しますがとりあえず当面あると便利なので以下のように実装します。

```go
// StringsChunk returns an iterator over non overlapping sub strings of n bytes.
// Sub slicing may cut in mid of utf8 sequences.
func StringsChunk(s string, n int) iter.Seq[string] {
	return func(yield func(string) bool) {
		s := s // no state in the seq.
		if n <= 0 {
			return
		}
		var cut string
		for {
			if len(s) >= n {
				cut, s = s[:n], s[n:]
			} else {
				cut, s = s, ""
			}
			if cut == "" {
				return
			}
			if !yield(cut) {
				return
			}
		}
	}
}

// StringsRuneChunk returns an iterator over non overlapping sub strings of n utf8 characters.
func StringsRuneChunk(s string, n int) iter.Seq[string] {
	return func(yield func(string) bool) {
		s := s // no state in the seq.
		for len(s) > 0 {
			var i int
			for range n {
				_, j := utf8.DecodeRuneInString(s[i:])
				if j == 0 {
					break
				}
				i += j
			}
			if i == 0 {
				return
			}
			if !yield(s[:i]) {
				return
			}
			s = s[i:]
		}
	}
}
```

`bufio.Scanner`みたいに任意のsplitterを渡してsub stringに分割したいので以下のように定義します。

```go
// StringsCutterFunc is used with [StringsSplitFunc] to cut string from head.
// s[:tokUntil] is yielded through StringsSplitFunc.
// s[tokUntil:skipUntil] will be ignored.
type StringsCutterFunc func(s string) (tokUntil, skipUntil int)

// StringsSplitFunc returns an iterator over sub string of s cut by splitFn.
// When n > 0, StringsSplitFunc cuts only n times and
// the returned iterator yields rest of string after n sub strings, if non empty.
// The sub strings from the iterator overlaps if splitFn decides so.
// splitFn is allowed to return negative offsets.
// In that case the returned iterator immediately yields rest of s and stops iteration.
func StringsSplitFunc(s string, n int, splitFn StringsCutterFunc) iter.Seq[string] {
	return func(yield func(string) bool) {
		if splitFn == nil {
			splitFn = StringsCutNewLine
		}
		s := s
		n := n
		for len(s) > 0 {
			tokUntil, skipUntil := splitFn(s)
			if tokUntil < 0 || skipUntil < 0 {
				yield(s)
				return
			}
			if !yield(s[:tokUntil]) {
				return
			}
			s = s[skipUntil:]
			n--
			if n == 0 {
				if len(s) > 0 {
					yield(s)
				}
				return
			}
		}
	}
}
```

コードジェネレーターを実装しているときにsnake_caseとPascalCaseとcamelCaseを相互変換したいときが頻繁にあるので以下のようなsplitterはあらかじめ定義しておきます。

```go
// StringsCutNewLine is used with [StringsSplitFunc].
// The input strings will be splitted at "\n".
// It also skips "\r" preceding "\n".
func StringsCutNewLine(s string) (tokUntil int, skipUntil int)
// StringsCutWord is a split function for a [StringsSplitFunc] that returns each space-separated word of text,
// with surrounding spaces deleted. It will never return an empty string.
// The definition of space is set by unicode.IsSpace.
func StringsCutWord(s string) (tokUntil int, skipUntil int)
// StringsCutUpperCase is a split function for a [StringsSplitFunc]
// that splits "UpperCasedWords" into "Upper" "Cased" "Words"
func StringsCutUpperCase(s string) (tokUntil int, skipUntil int)
```

`iter.Seq[string]`をstringにreduceするのは頻繁にやりそうなので以下のようなものがあると便利ですね

```go
// StringsCollect reduces seq to a single string.
// sizeHint hints size of internal buffer.
// Correctly sized sizeHint may reduce allocation.
func StringsCollect(sizeHint int, seq iter.Seq[string]) string {
	var buf strings.Builder
	buf.Grow(sizeHint)
	for s := range seq {
		buf.WriteString(s)
	}
	return buf.String()
}
```

### container/heap, container/list, container/ring

stdの`container/heap`, `container/list`, `container/ring`を以下のようにするとiteratorに変換できます。
こういったものはstdで実装してほしい気もしますが、`container`のgeneric版も出てきていないのであまり期待しちゃだめなのかも。

```go
// Heap returns an iterator over heap.Interface.
// Consuming iter.Seq[V] also consumes h.
// To avoid this, callers must clone input h before passing to Heap.
func Heap[V any](h heap.Interface) iter.Seq[V] {
	return func(yield func(V) bool) {
		for h.Len() > 0 {
			popped := heap.Pop(h)
			if !yield(popped.(V)) {
				return
			}
		}
	}
}

// ListAll returns an iterator over all element of l starting from l.Front().
// ListAll assumes Values of all element are type V.
// If other than that or nil, the returned iterator may panic on invocation.
func ListAll[V any](l *list.List) iter.Seq[V] {
	return ListElementAll[V](l.Front())
}

// ListElementAll returns an iterator over from ele to end of the list.
// ListElementAll assumes Values of all element are type V.
// If other than that or nil, the returned iterator may panic on invocation.
func ListElementAll[V any](ele *list.Element) iter.Seq[V] {
	return func(yield func(V) bool) {
		// shadowing ele, no state in the seq closure as much as possible.
		for ele := ele; ele != nil; ele = ele.Next() {
			if !yield(ele.Value.(V)) {
				return
			}
		}
	}
}

// ListBackward returns an iterator over all element of l starting from l.Back().
// ListBackward assumes Values of all element are type V.
// If other than that or nil, the returned iterator may panic on invocation.
func ListBackward[V any](l *list.List) iter.Seq[V] {
	return ListElementBackward[V](l.Back())
}

// ListElementBackward returns an iterator over from ele to start of the list.
// ListElementBackward assumes Values of all element are type V.
// If other than that or nil, the returned iterator may panic on invocation.
func ListElementBackward[V any](ele *list.Element) iter.Seq[V] {
	return func(yield func(V) bool) {
		// no state in in the seq closure as much as possible.
		for ele := ele; ele != nil; ele = ele.Prev() {
			if !yield(ele.Value.(V)) {
				return
			}
		}
	}
}

// Ring returns an iterator over r.
// The returned iterator generates data assuming Values of all ring elements are type V.
// It yields r.Value traversing by consecutively calling Next, and stops when it finds r again.
// Removing r from the ring after it started iteration may make it iterate forever.
func RingAll[V any](r *ring.Ring) iter.Seq[V] {
	return func(yield func(V) bool) {
		if !yield(r.Value.(V)) {
			return
		}
		for n := r.Next(); n != r; n = n.Next() {
			if !yield(n.Value.(V)) {
				return
			}
		}
	}
}

// RingBackward returns an iterator over r.
// The returned iterator generates data assuming Values of all ring elements are type V.
// It yields r.Value traversing by consecutively calling Prev, and stops when it finds r again.
// Removing r from the ring after it started iteration may make it iterate forever.
func RingBackward[V any](r *ring.Ring) iter.Seq[V] {
	return func(yield func(V) bool) {
		if !yield(r.Value.(V)) {
			return
		}
		for n := r.Prev(); n != r; n = n.Prev() {
			if !yield(n.Value.(V)) {
				return
			}
		}
	}
}
```

### sync.Map

`sync.Map`の`Range`自体がすでに`iter.Seq2[any, any]`のシグネチャを満たしていますので不要と言えば不要ですが、以下のように`sync.Map`をiteratorに変換する関数を定義してあげると分かりやすいかもしれません。

```go
// SyncMap returns an iterator over m.
// Breaking Seq2 may stop producing more data, however it might still be O(N).
func SyncMap[K, V any](m *sync.Map) iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {
		m.Range(func(key, value any) bool {
			return yield(key.(K), value.(V))
		})
	}
}
```

やっていることはtype assertionだけなのでkey, valueにそれぞれ複数の型を使っている場合には適合しませんが、あんまりないことだと思っています。

### reflect.Value.Seq, reflect.Value.Se2

[reflect.Value.Seq](https://pkg.go.dev/reflect@go1.23.1#Value.Seq)および[reflect.Value.Seq2](https://pkg.go.dev/reflect@go1.23.1#Value.Seq2)で`iter.Seq[reflect.Value]`、`iter.Seq2[reflect.Value, reflect.Value]`をそれぞれ`reflect`を通じて得られるようになりました。これを`type assertion`を通して別の型になするadapterを定義しておきます。

```go
// AssertValue returns an iterator over seq but each value returned by [reflect.Value.Interface] is type-asserted to be type V.
func AssertValue[V any](seq iter.Seq[reflect.Value]) iter.Seq[V] {
	return mapIter(func(v reflect.Value) V { return v.Interface().(V) }, seq)
}

// Assert2 returns an iterator over seq but internal values returned by [reflect.Value.Interface] of each key-value pair
// are type-asserted to be type K and V respectively.
func AssertValue2[K, V any](seq iter.Seq2[reflect.Value, reflect.Value]) iter.Seq2[K, V] {
	return mapIter2(func(k, v reflect.Value) (K, V) { return k.Interface().(K), v.Interface().(V) }, seq)
}

// Assert returns an iterator over seq but each value is type-asserted to be type V.
func Assert[V any](seq iter.Seq[any]) iter.Seq[V] {
	return mapIter(func(v any) V { return v.(V) }, seq)
}

// Assert2 returns an iterator over seq but each key-value pair is type-asserted to be type K and V respectively.
func Assert2[K, V any](seq iter.Seq2[any, any]) iter.Seq2[K, V] {
	return mapIter2(func(k, v any) (K, V) { return k.(K), v.(V) }, seq)
}
```

やってることは特化した`xiter.Map`なので実装されている必然性のようなものは薄いですが、網羅性のために実装します。

### Third party: github.com/wk8/go-ordered-map/v2

[github.com/wk8/go-ordered-map/v2](httos://github.com/wk8/go-ordered-map): 挿入順序という意味のordered-map実装です。内部的に`map[K]*V`+`*list.List`の組み合わせで実現されています。

まだリリースされていませんが、[#41](https://github.com/wk8/go-ordered-map/pull/41)でiteratorを返す関数が実装されています。`CircleCI/cimg-go`がgo1.23.0に対応していないためにリリースできないらしいですが、[CircleCI-Public/cimg-go#300](https://github.com/CircleCI-Public/cimg-go/pull/300)がマージされたためそのうちリリースされるはず。

### Third party: github.com/gammazero/deque

[github.com/gammazero/deque](https://github.com/gammazero/deque): `[]T`ベースのdouble-ended queue実装です。`[]T`をリングバッファ的に用いることで効率性を追求していますのでqueue両端以外の要素が消されると`container/list.List`ベースな処理に比べるとややつらいはず。

こちらは動きがないため`iter.Seq`を返すメソッドの実装はありません。必要になったらPRを出してみようかと思いますが活発ではないかもしれないので出したとてマージされないかも。

なので、これをラップしてiteratorに変換できる関数を定義しましょう。

この実装では、[(\*deque.Deque\[T\]).At(i int) T](https://pkg.go.dev/github.com/gammazero/deque#Deque.At)で各インデックスにアクセス可能です。stdでも`At`メソッドを実装する型に以下の4つがあります。

- https://pkg.go.dev/encoding/asn1@go1.23.0#BitString.At
- https://pkg.go.dev/go/types@go1.23.0#MethodSet.At
- https://pkg.go.dev/go/types@go1.23.0#Tuple.At
- https://pkg.go.dev/go/types@go1.23.0#TypeList.At

そこそこ一般的なinterfaceであると判断して以下のように`At`をiteratorに変換する関数を用意します。

```go
type Atter[T any] interface {
	At(i int) T
}

// IndexAccessible returns an iterator over indices and values of a associated to the indices.
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

### Third party: github.com/ngicks/und/option

自作モジュールです。
この記事: [GoのJSONのT | null | undefinedは[]Option[T]で表現できる](https://zenn.dev/ngicks/articles/go-json-undefined-or-null-slice)で実装した`und`モジュールの各typeに`Iter`メソッドを実装しました。

```go
// Iter returns an iterator over the internal value.
// If o is some, the iterator yields the [Option.Value](), otherwise nothing.
func (o Option[T]) Iter() iter.Seq[T] {
	return func(yield func(T) bool) {
		if o.IsSome() {
			yield(o.Value())
		}
	}
}
```

これは`Rust`の[core::option::Optionのiterメソッド](https://doc.rust-lang.org/core/option/enum.Option.html#method.iter)をそのままパクったものです。

まだまだ色々いじるところがあってしばらくタグ付けるつもりはないのですが・・・。

### \*bufio.Scanner

`*bufio.Scanner`は以下のように変換できます。
`Scanner`には`Err`メソッドがあるためこれをチェックするものとするとして`iter.Seq[string]`を返してもいいのですが、`iter.Seq2[V, error]`を前提としたエラーハンドリングをいろいろ実装しているため互換性のために`iter.Seq2[V, error]`版も実装しておきます。

```go
// Scanner wraps scanner with an iterator over scanned text.
// The caller should check [bufio.Scanner.Err] after the returned iterator stops
// to see if it has been stopped for an error.
func Scan(scanner *bufio.Scanner) iter.Seq[string] {
	return func(yield func(string) bool) {
		for scanner.Scan() {
			if !yield(scanner.Text()) {
				return
			}
		}
	}
}

// ScanErr is like [Scan] but also yields scanner's error if any.
func ScanErr(scanner *bufio.Scanner) iter.Seq2[string, error] {
	return func(yield func(string, error) bool) {
		for scanner.Scan() {
			if !yield(scanner.Text(), nil) {
				return
			}
		}
		if scanner.Err() != nil {
			yield("", scanner.Err())
			return
		}
	}
}
```

### \*sql.Rows / Nexter interface { Next() bool }

`*sql.Rows`も以下のようにするとiteratorとして利用できます。
`Nexter`版を実装しておくと、[\*sqlx.Rows](https://pkg.go.dev/github.com/jmoiron/sqlx@v1.4.0#Rows)も同じように利用できます。

```go
// SqlRows returns an iterator over scanned rows from r.
// scanner will be called against [*sql.Rows] after each time [*sql.Rows.Next] returns true.
// It must either call [*sql.Rows.Scan] once per invocation or do nothing and return.
// If the scan result or [*sql.Rows.Err] returns a non-nil error,
// the iterator stops its iteration immediately after yielding the error.
func SqlRows[T any](r *sql.Rows, scanner func(*sql.Rows) (T, error)) iter.Seq2[T, error] {
	return Nexter(r, scanner)
}

// Nexter is like [SqlRows] but extends the input to arbitrary implementors, e.g. sqlx.
func Nexter[
	T any,
	Nexter interface {
		Next() bool
		Err() error
	},
](n Nexter, scanner func(Nexter) (T, error)) iter.Seq2[T, error] {
	return func(yield func(T, error) bool) {
		for n.Next() {
			t, err := scanner(n)
			if !yield(t, err) {
				return
			}
			if err != nil {
				return
			}
		}
		if n.Err() != nil {
			yield(*new(T), n.Err())
			return
		}
	}
}
```

`Nexter`版は[こちらのスクラップ](https://zenn.dev/macopy/scraps/55ae36007fc8ce)を見てなるほどと思って実装しました。皆さんの情報発信に助けられております。

### encoding/json.Decoder, ending/xml.Decoder

`*(json|xml).Decoder`も以下のようにするとiteratorへと変換できます。
基本的に、`Token`メソッドを呼び出す時は同時にdecoderそのものにアクセスして[Decode](https://pkg.go.dev/encoding/json@go1.23.1#Decoder.Decode)メソッドを呼び出したかったりするはずです。
そのため、decoderに対して特化した処理を書くことはほとんど必ずするので`iter.Seq`への変換は大した利益を及ぼさないかもしれません。

```go
// JsonDecoder returns an iterator over json tokens.
// The first non-nil error encountered stops iteration after yielding it.
// [io.EOF] is excluded from result.
func JsonDecoder(dec *json.Decoder) iter.Seq2[json.Token, error] {
	return tokener(dec)
}

// XmlDecoder returns an iterator over xml tokens.
// The first non-nil error encountered stops iteration after yielding it.
// [io.EOF] is excluded from result.
// The caller should call [xml.CopyToken] before going to next iteration if they need to retain tokens.
func XmlDecoder(dec *xml.Decoder) iter.Seq2[xml.Token, error] {
	return tokener(dec)
}

func tokener[Dec interface{ Token() (V, error) }, V any](dec Dec) iter.Seq2[V, error] {
	return func(yield func(V, error) bool) {
		for {
			t, err := dec.Token()
			if err != nil {
				if err == io.EOF {
					return
				}
				yield(*new(V), err)
				return
			}
			if !yield(t, nil) {
				return
			}
		}
	}
}
```

### Dec interface{ Decode(any) error }

上記の`JsonDecoder`と`XmlDecoder`よりももう少しジェネリックに、`Dec`インターフェイスをiteratorに変換できるものを定義します。
`Dec interface{ Decode(any) error }`は`*(json|xml).Decoder`が制約を満たすことができるので、

```go
// Decode returns an iterator over consecutive decode results of dec.
//
// The iterator stops if and only if dec returns io.EOF. Handling other errors is caller's responsibility.
// If the first error should stop the iterator, use [LimitUntil], [LimitAfter] or [*errbox.Box].
func Decode[V any, Dec interface{ Decode(any) error }](dec Dec) iter.Seq2[V, error] {
	return func(yield func(V, error) bool) {
		for {
			var v V
			err := dec.Decode(&v)
			if err == io.EOF {
				return
			}
			if !yield(v, err) {
				return
			}
		}
	}
}
```

エラーの取り扱いが少し面倒で、呼び出し側に終了かどうかを選ばせる必要があります。
なぜかというと、`Decode`に渡された値の型(`*V`)と入力の型が合わないという意味論的エラーの場合は次の`Decode`呼び出しが行えるかもしれないですが、
`Dec`内部の`io.Reader`がエラーを返した場合はそのエラーがキャッシュされて何度呼び出しても同じエラーが帰ってくることになります。
例えば、`encoding/json`は[UnmarshalTypeError](https://pkg.go.dev/encoding/json@go1.23.1#UnmarshalTypeError)というエラーを返す時は次の呼び出しで次のjson valueのデコードに移れますが、
`io.Reader`からエラーが帰ってきた場合は`Decode`を何度呼び出してもそのエラーが帰ってくる挙動になっています。

### Moving Window([]V, iter.Seq[V])

移動平均をとるときなどにmoving windowは便利です。

```go
// Window returns an iterator over overlapping sub-slices of n size (moving windows).
// n must be a positive non zero value.
// Values from the iterator are always slices of n size.
// The iterator yields nothing when it is not possible.
func Window[S ~[]E, E any](s S, n int) iter.Seq[S] {
	return func(yield func(S) bool) {
		if n <= 0 {
			return
		}
		var (
			start = 0
			end   = n
		)
		for {
			if end > len(s) {
				return
			}
			if !yield(s[start:end:end]) {
				return
			}
			start++
			end++
		}
	}
}
```

どこかにサブスライスを渡す場合、capも指定(`[start:end:cap]`)してlengthとcapを一致させると次のappend呼び出し時にlengthがcapを超えるのでsliceがコピーされます([この行](https://github.com/golang/go/blob/go1.23.1/src/cmd/compile/internal/ssagen/ssa.go#L3531))。コピーが起きないと元のsliceと同じunderlying arrayへ書き込みを行ってしまいますので、こういった気遣いをしておくと親切なケースがあります。

ついでに`iter.Seq[V]`を引数にとる`WindowSeq`も実装(これは前の記事には含まれない)

```go
// WindowSeq allocates n sized buffer and fills it with values from seq in FIFO-manner.
// Once the buffer is full, the returned iterator yields iterator over buffered values
// each time value is yielded from the input seq.
//
// n must be a positive non zero value.
// Each iterator yields exact n size of values.
// If seq yields less than n, the iterator yields nothing.
func WindowSeq[V any](n int, seq iter.Seq[V]) iter.Seq[iter.Seq[V]] {
	return func(yield func(iter.Seq[V]) bool) {
		if n <= 0 {
			return
		}
		var (
			buf    = make([]V, n)
			cursor = 0
			full   = false
		)
		for e := range seq {
			if !full {
				buf[cursor] = e
				cursor++
				if cursor == n {
					cursor = 0
					full = true
					if !yield(sliceRing(buf, cursor)) {
						return
					}
				}
				continue
			}
			buf[cursor] = e
			cursor = (cursor + 1) % n
			if !yield(sliceRing(buf, cursor)) {
				return
			}
		}
	}
}

func sliceRing[S ~[]E, E any](s S, start int) iter.Seq[E] {
	return func(yield func(E) bool) {
		if !yield(s[start]) {
			return
		}
		for i := start + 1; ; i++ {
            i = i % len(s)
			if i == start {
				break
			}
			if !yield(s[i]) {
				return
			}
		}
	}
}
```

`iter.Seq[V]`を引数にmoving windowをするということは要素をiteratorから得てFIFOでバッファする必要があります。
ring buffer的なものを効率的に実装しないと使い物にならないけどここに凝った実装したくありませんでした。かといってテスト目的以外の依存性も追加したくないですし、ちょっと悩んでしまいました。
しかしこういう感じで`iter.Seq[T]`を返せばなんとなくいい感じになるのに気づいたので解決です。

### デバッグ・テスト向け: KeyValue (K-V pair)

`map[K]V`の[iterate順序は言語仕様により未定義](https://go.dev/ref/spec#For_range)であるので順序を指定して`iter.Seq2[K, V]`を作成することができる型が欲しい。あるといろいろ便利なんですよね。

```go
type KeyValue[K, V any] struct {
	K K
	V V
}

// Values2 returns an iterator that yields the KeyValue slice elements in order.
func Values2[S ~[]KeyValue[K, V], K, V any](s S) iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {
		for _, kv := range s {
			if !yield(kv.K, kv.V) {
				return
			}
		}
	}
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

// ToKeyValue converts [iter.Seq2][K, V] into iter.Seq[KeyValue[K, V]].
// This functions is particularly useful when sending values from [iter.Seq2][K, V] through
// some data transfer mechanism that only allows data to be single value, e.g. channels.
func ToKeyValue[K, V any](seq iter.Seq2[K, V]) iter.Seq[KeyValue[K, V]] {
	return func(yield func(KeyValue[K, V]) bool) {
		for k, v := range seq {
			if !yield(KeyValue[K, V]{k, v}) {
				return
			}
		}
	}
}

// FromKeyValue unwraps iter.Seq[KeyValue[K, V]] into iter.Seq2[K, V] to counter-part,
// serving a counterpart for [ToKeyValue].
//
// In case values from seq needs to be sent through some data transfer mechanism
// that only allows data to be single value, like channels,
// some caller might decide to wrap values into KeyValue[K, V], maybe by [ToKeyValue].
// If target helpers only accept iter.Seq2[K, V], then FromKeyValues is useful.
func FromKeyValue[K, V any](seq iter.Seq[KeyValue[K, V]]) iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {
		for kv := range seq {
			if !yield(kv.K, kv.V) {
				return
			}
		}
	}
}

var _ Iterable2[any, any] = KeyValues[any, any](nil)

// KeyValues adds the Iter2 method to slice of KeyValue-s.
type KeyValues[K, V any] []KeyValue[K, V]

func (v KeyValues[K, V]) Iter2() iter.Seq2[K, V] {
	return Values2(v)
}
```

### Permutations

`[]int{1, 2, 3}`に対して、`[][]int{{1,2,3}, {1,3,2}, {2,1,3}, {2,3,1}, {3,1,2}, {3,2,1}}`をPermutations(置換)と言います。
テスト目的に使われることがたびたびあるのを目にします。
これはリンク先のHeap's algorithmを実装しただけです。

```go
// Permutations returns an iterator that yields permutations of in.
// The returned iterator reorders in in-place.
// The caller should not retain in or slices from the iterator,
// Or should explicitly clone yielded values.
func Permutations[S ~[]E, E any](in S) iter.Seq[S] {
	// implementation of Heap's algorithm
	// https://en.wikipedia.org/wiki/Heap%27s_algorithm
	return func(yield func(S) bool) {
		k := len(in)
		c := make([]int, k)

		if !yield(in) {
			return
		}

		if k < 2 {
			// no reordering
			return
		}

		i := 1

		for i < k {
			if c[i] < i {
				if i%2 == 0 {
					in[0], in[i] = in[i], in[0]
				} else {
					in[c[i]], in[i] = in[i], in[c[i]]
				}

				if !yield(in) {
					return
				}

				c[i] += 1
				i = 1
			} else {
				c[i] = 0
				i += 1
			}
		}
	}
}
```

### RunningReduce: reduceの中間結果を得られるiterator

`Reduce`だが、`reducer`実行のたびに中間の結果をyieldできるというもの。何かで使い道がありそう。

```go
// RunningReduce returns an iterator over every intermediate reducer results.
func RunningReduce[V, Sum any](
	reducer func(accumulator Sum, current V, i int) Sum,
	initial Sum,
	seq iter.Seq[V],
) iter.Seq[Sum] {
	return func(yield func(Sum) bool) {
		var i int
		for v := range seq {
			initial = reducer(initial, v, i)
			i++
			if !yield(initial) {
				return
			}
		}
	}
}
```

## Resumable / Peekable

iteratorはresumableかもしれないし、resumableではないかもしれません。
そこで`Resumable`になるようにiteratorをラップできるようになると便利なケースが多分あります。
また、次の要素を消費せずに取得して先読みしたいケースはよくあると思います。
典型的なユースケースの例はJSONのデコードで、次のトークンが`null`か`{`以外のとき`UnmarshalTypeError`を返し、`{`の時は別のデコード処理にそのままのトークンストリーム(=`{`から始まる)を渡す、みたいなときに先読みができると便利です。

### iteratorはresumableかもしれないし、そうでないかもしれない

iteratorは以下のシグネチャを持つ関数です。

```go
func(func() bool)
func(func(V) bool)
func(func(K, V) bool)
```

`Go`は仕様的にメソッドを関数シグネチャに渡してもいいですし、クロージャーを渡してもよいということになっています。
しかるにiteratorも同様にstateを持つかもしれないし、持たないかもしれません。

具体的には以下のような感じです。

[playground](https://go.dev/play/p/LhMOHbyE89V)

```go
func seq1[V any](v V, n int) iter.Seq[V] {
    return func(yield func(V) bool) {
        for m := n; m != 0; m-- {
            if !yield(v) {
                return
            }
        }
    }
}

func seq2[V any](v V, n int) iter.Seq[V] {
    return func(yield func(V) bool) {
        for ; n != 0; n-- {
            if !yield(v) {
                return
            }
        }
    }
}
```

上記二つは以下のように、breakして再度for-rangeにかけると違った挙動をします。

```go
func breakAndResume[V any](seq iter.Seq[V], n int) {
    i := n
    for v := range seq {
        fmt.Printf("%v, ", v)
        i--
        if i <= 0 {
            break
        }
    }
    fmt.Println()
    for v := range seq {
        fmt.Printf("%v, ", v)
    }
    fmt.Println()
}

func main() {
    fmt.Println("seq1:")
    breakAndResume(seq1(5, 5), 3)
    /*
        seq1:
        5, 5, 5,
        5, 5, 5, 5, 5,
    */
    fmt.Println("\nseq2:")
    breakAndResume(seq2(5, 5), 3)
    /*
        seq2:
        5, 5, 5,
        5, 5, 5,
    */
}
```

`seq1`はstatelessですが、`seq2`は`n`をシャドーイングしていないのに`n--`してしまっているのでstatefulになっています。

これらは[iter.Pull](https://pkg.go.dev/iter@go1.23.1#Pull)でラップすることで同じように扱うことができます。

```go
func wrapPull[V any](seq iter.Seq[V]) (iter.Seq[V], func()) {
    next, stop := iter.Pull(seq)
    return func(yield func(V) bool) {
        for {
            v, ok := next()
            if !ok || !yield(v) {
                return
            }
        }
    }, stop
}

func main() {
    fmt.Println("\nseq1:")
    seq, stop := wrapPull(seq1(5, 5))
    breakAndResume(seq, 3)
    stop()
    /*
        seq1:
        5, 5, 5,
        5, 5,
    */
    fmt.Println("\nseq2:")
    seq, stop = wrapPull(seq2(5, 5))
    breakAndResume(seq, 3)
    stop()
    /*
        seq2:
        5, 5, 5,
        5, 5,
    */
}
```

[iter.Pull](https://pkg.go.dev/iter@go1.23.1#Pull)および[iter.Pull2](https://pkg.go.dev/iter@go1.23.1#Pull2)は特有のコントロールを行うことでスケジューラの処理をスキップする`goroutine`をallocateし、その中で`iter.Seq`を実行するというものです。
そのため`stop`を呼ぶか、seqを最後まで消費しきるかしないと`goroutine`が終了しないため`goroutine leak`となってしまいます。

### Resumable

ここから本題です。

iteratorがresumable/statefulなのかstatelessなのかわからないのであればとりあえず問答無用で`iter.Pull`でラップしてしまえばresumableであるとして扱えますね。
ということでそういう構造体を定義します。

やってることは`iter.Pull`でiteratorをラップして返された`next`, `stop`を構造体のフィールドに代入しているだけで大したことはしていないのですが、他の構造体に埋め込む前提なのでこうしています。

```go
// Resumable converts the input [iter.Seq][V] into stateful form by calling [iter.Pull].
//
// The zero value of Resumable is not valid. Allocate one by [NewResumable].
type Resumable[V any] struct {
	next func() (V, bool)
	stop func()
}

// NewResumable wraps seq into stateful form so that the iterator can be break-ed and resumed.
// The caller must call [*Resumable.Stop] to release resources regardless of usage.
func NewResumable[V any](seq iter.Seq[V]) *Resumable[V] {
	next, stop := iter.Pull(seq)
	return &Resumable[V]{
		next: next,
		stop: stop,
	}
}

// Stop releases resources allocated by [NewResumable].
func (r *Resumable[V]) Stop() {
	r.stop()
}

// IntoIter returns an iterator over the input iterator.
// The iterator can be paused by break and later resumed without replaying data.
func (r *Resumable[V]) IntoIter() iter.Seq[V] {
	return func(yield func(V) bool) {
		for {
			v, ok := r.next()
			if !ok {
				return
			}
			if !yield(v) {
				return
			}
		}
	}
}

type Resumable2[K, V any] struct { ... }
func NewResumable2[K, V any](seq iter.Seq2[K, V]) *Resumable2[K, V]
func (r *Resumable2[K, V]) Stop()
func (r *Resumable2[K, V]) IntoIter2() iter.Seq2[K, V]
```

### Peekable

iteratorを任意にresumableにラップできるようになったことで、任意の要素数をpeekして読み込むこともできるようになりました。

```go
// First returns the first value from seq.
// ok is false if seq yields no value.
func First[V any](seq iter.Seq[V]) (k V, ok bool) {
	for v := range seq {
		return v, true
	}
	return *new(V), false
}

func main() {
	src := hiter.Range(0, 10)
	resumable := iterable.NewResumable(src)

	v0, _ := hiter.First(resumable.IntoIter())
	fmt.Printf("first:  %d\n", v0)
	v1, _ := hiter.First(resumable.IntoIter())
	fmt.Printf("second: %d\n", v1)

	fmt.Println()
	fmt.Println("reconnect them to whole iterator.")
	first = true
	for v := range xiter.Concat(hiter.Once(v0), hiter.Once(v1), resumable.IntoIter()) {
		if !first {
			fmt.Print(", ")
		}
		first = false
		fmt.Printf("%d", v)
	}
	fmt.Println()
	// first:  0
	// second: 1
	//
	// reconnect them to whole iterator.
	// 0, 1, 2, 3, 4, 5, 6, 7, 8, 9
}
```

これは少し煩雑なので、`Resumable`同様に型にまとめておきます。

```go
type Peekable[V any] struct {
	r      *Resumable[V]
	peeked []V
}

func NewPeekable[V any](seq iter.Seq[V]) *Peekable[V] {
	return &Peekable[V]{
		r: NewResumable(seq),
	}
}

func (p *Peekable[V]) Stop() {
	p.r.Stop()
}

func (p *Peekable[V]) Peek(n int) []V {
	// internal behavior might need some change to use ring buffer.
	if diff := n - len(p.peeked); diff > 0 {
		p.peeked = slices.AppendSeq(p.peeked, xiter.Limit(p.r.IntoIter(), diff))
	}
	if len(p.peeked) > n {
		return p.peeked[:n:n]
	}
	return slices.Clip(p.peeked)
}

func (p *Peekable[V]) pop() V {
	v0 := p.peeked[0]
	p.peeked[0] = *new(V)
	p.peeked = p.peeked[1:]
	return v0
}

func (p *Peekable[V]) IntoIter() iter.Seq[V] {
	return func(yield func(V) bool) {
		for len(p.peeked) > 0 {
			if !yield(p.pop()) {
				return
			}
		}
		for v := range p.r.IntoIter() {
			if !yield(v) {
				return
			}
			for len(p.peeked) > 0 {
				if !yield(p.pop()) {
					return
				}
			}
		}
	}
}

type Peekable2[K, V any] struct { ... }
func NewPeekable2[K, V any](seq iter.Seq2[K, V]) *Peekable2[K, V]
func (p *Peekable2[K, V]) Stop()
func (p *Peekable2[K, V]) Peek(n int) []hiter.KeyValue[K, V]
func (p *Peekable2[K, V]) IntoIter2() iter.Seq2[K, V]
```

ポイントとしてはiteratorを返す構造体は返されたiteratorの`yield`関数の中でその構造体を操作している可能性があることです。
このケースで言うと`yield`の中でさらに`Peek`を呼んでいるかもしれないので`yield`を抜けたらもう１度`peeked`のチェックが必要です。

先ほどのpeekのexampleを`Peekable`を使って書き直すと以下のようになります。

```go
func main() {
	fmt.Println("\nYou can achieve above also with iterable.Peekable")
	peekable := iterable.NewPeekable(src)
	fmt.Printf("%#v\n", peekable.Peek(5))
	first = true
	for v := range peekable.IntoIter() {
		if !first {
			fmt.Print(", ")
		}
		first = false
		fmt.Printf("%d", v)
	}
	fmt.Println()
	// You can achieve above also with iterable.Peekable
	// []int{0, 1, 2, 3, 4}
	// 0, 1, 2, 3, 4, 5, 6, 7, 8, 9
}
```

前述の`WindowSeq`(moving windowの`iter.Seq`を引数に受け付けるバージョン)と組み合わせることを前提に考えているので`Peek`はn個要素を先読みできるようになっています。

`Peek`の返り値はこの実装では`[]V`ですが内部的にring bufferやlistを用いるようにリファクタした場合は`iter.Seq[V]`を返すようにしたほうが`[]V`へのコピーが生じないので有利な気がします。ただ下手ないじり方をすると逆に遅くなるのはたびたび体験しているのでおとなしくコピーしておけばよい気もします。

## error handling

`iter.Seq2[V, error]`でエラーを表現するiteratorをいくつか定義しました。

```go
func Decode[V any, Dec interface{ Decode(any) error }](dec Dec) iter.Seq2[V, error]
func JsonDecoder(dec *json.Decoder) iter.Seq2[json.Token, error]
func XmlDecoder(dec *xml.Decoder) iter.Seq2[xml.Token, error]
func ScanErr(scanner *bufio.Scanner) iter.Seq2[string, error]
func SqlRows[T any](r *sql.Rows, scanner func(*sql.Rows) (T, error)) iter.Seq2[T, error]
func Nexter[T any, Nexter interface { Next() bool; Err() error }](n Nexter, scanner func(Nexter) (T, error)) iter.Seq2[T, error]
```

エラーしうるiteratorの表現にこれを使うように導かれるのはなんだか自然に思います。
これに対するエラーハンドルはどのように行うかについていくつか考えます。

### for v, err := range seq { if err != nil { return err } }

もっとも単純なのは単にfor-range-funcで要素を消費しながら、最初のエラーでreturnしてしまうことです。

```go
func handle[V any](seq iter.Seq2[V, error]) error {
	for v, err := range seq {
		if err != nil {
			// handle error
			return err
		}
		// handle v.
	}
}
```

特にいうことはないですね。大抵はこれでいいはずです。

### TryFind, TryForEach, TryReduce

`Find`, `ForEach`, `Reduce`のカウンターパーツとして`TryFind`, `TryForEach`, `TryReduce`を実装します。
上記の「最初のエラーでreturn」と同等のことをするが若干特化したヘルパーです。`slices.Collect`と同じぐらい多用するでしょうからまあ定義しておけばよいでしょう。

```go
// TryFind is like [FindFunc] but stops if seq yields non-nil error.
func TryFind[V any](f func(V) bool, seq iter.Seq2[V, error]) (v V, idx int, err error) {
	var i int
	for v, err := range seq {
		if err != nil {
			return *new(V), -1, err
		}
		if f(v) {
			return v, i, nil
		}
		i++
	}
	return *new(V), -1, nil
}

// TryForEach is like [ForEach] but returns an error if seq yields non-nil error.
func TryForEach[V any](f func(V), seq iter.Seq2[V, error]) error {
	for v, err := range seq {
		if err != nil {
			return err
		}
		f(v)
	}
	return nil
}

// TryReduce is like [xiter.Reduce] but returns an error if seq yields non-nil error.
func TryReduce[Sum, V any](f func(Sum, V) Sum, sum Sum, seq iter.Seq2[V, error]) (Sum, error) {
	for v, err := range seq {
		if err != nil {
			return sum, err
		}
		sum = f(sum, v)
	}
	return sum, nil
}
```

`TryCollect`, `TryAppendSeq`もついでに実装しておきます。

```go
// TryCollect is like [slices.Collect] but stops collecting at the first error and
// returns the extended result before the error.
func TryCollect[E any](seq iter.Seq2[E, error]) ([]E, error) {
	return TryAppendSeq[[]E](nil, seq)
}

// TryAppendSeq is like [slices.AppendSeq] but stops collecting at the first error
// and returns the extended result before the error.
func TryAppendSeq[S ~[]E, E any](s S, seq iter.Seq2[E, error]) (S, error) {
	for e, err := range seq {
		if err != nil {
			return s, err
		}
		s = append(s, e)
	}
	return s, nil
}
```

### LimitAfter2

前述した`hiter.Collect2`で`iter.Seq2[K, V]`を`[]hiter.KeyValue[K, V]`に回収できますが、最初のエラーで停止しない`iter.Seq2[V, error]`を引数に取ると無限ループにははまってしまうのでテストその他で結構面倒になります。前述の`func Decode[V any, Dec interface{ Decode(any) error }](dec Dec) iter.Seq2[V, error]`はエラーが発生しても`io.EOF`以外の時は停止しないように作ったので実際このケースで困ることが筆者自身ありました。

こういったケースには[LimitUntil](https://pkg.go.dev/github.com/ngicks/go-iterator-helper/hiter#LimitUntil)を用いることができますが、最後のerrorをyieldできないという問題が生じます。
`LimitUntil`は[Rustのcore::iterator::Iterator::take_while](https://doc.rust-lang.org/beta/core/iter/trait.Iterator.html#method.take_while)のカウンターパートとして実装しました。そっくりそのままな仕様なのでコールバック関数がfalseを返したらその値をyieldせずにiteratorは停止します。
なのでさらに以下のように`LimitAfter2`を定義します。

```go
// LimitAfter2 is like [LimitUntil2] but also yields the first pair dissatisfying f(k, v).
func LimitAfter2[K, V any](f func(K, V) bool, seq iter.Seq2[K, V]) iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {
		for k, v := range seq {
			if !f(k, v) {
				yield(k, v)
				return
			}
			if !yield(k, v) {
				return
			}
		}
	}
}
```

これは以下のように利用できます。

```go
type sample struct {
	V string
}

src := []byte(`
{"V":"foo"}
{"V":"bar"}
{"V":"baz"}
`)

dec := json.NewDecoder(io.MultiReader(bytes.NewReader(src), iotest.ErrReader(errSample)))

result := hiter.Collect2(
	hiter.LimitAfter2(
		func(_ sample, err error) bool { return err == nil },
		hiter.Decode[sample](dec),
	),
)
// []hiter.KeyValue[sample, error]{
//     {K: sample{"foo"}},
//     {K: sample{"bar"}},
//     {K: sample{"baz"}},
//     {V: errSample},
// }
```

最初のエラーで停止するがエラー値をyieldできるようになりました。

### HandleErr adapter

別口の発想としてnon-nil errorが得られたときhandleコールバック関数を実行することとして`iter.Seq2[V, error]`を`iter.Seq[V]`に変換するアダプターを実装します。

```go
// HandleErr returns an iterator over only former value of seq.
// If latter value the seq yields is non-nil then it calls handle.
// If handle returns false the iterator stops.
// Even if handle returns true, values paired to non-nil error are excluded from the returned iterator.
func HandleErr[V any](handle func(V, error) bool, seq iter.Seq2[V, error]) iter.Seq[V] {
	return hiter.OmitL(
		filter2(
			func(_ V, err error) bool { return err == nil },
			hiter.LimitUntil2(func(v V, err error) bool { return err == nil || handle(v, err) }, seq),
		),
	)
}
```

こうすれば`iter.Seq[V]`を処理するすべての関数を使うことができますし、最初のエラー=停止ではないときに便利です。
典型的なパターンにはまらないハンドリングは結局自分で定義することになるでしょうしここで定義すべきものはこんなもんでいいでしょう。

### \*errbox.Box: 最初のエラーを永続化する構造体

`*bufio.Scanner`や`*sql.Rows`は`Err`メソッドを備え、エラーが発生したときにも`Scan`、`Next`メソッドがfalseを返すようにすることで慣習的にエラーをチェックさえるものです。
このAPIスタイルに寄せるの目的で`iter.Seq2[V, error]` -> `iter.Seq[V]`に変換する構造体を定義し、最初のエラーを保存して`Err`メソッドでそれを返せるようにします。

```go
// Box boxes an input [iter.Seq2][V, error] to [iter.Seq][V] by stripping nil errors from the input iterator.
// The first non-nil error causes the boxed iterator to be stopped and Box stores the error.
// Later the error can be examined by [*Box.Err].
//
// Box converts input iterator stateful form by calling [iter.Pull2].
// [*Box.IntoIter] returns the iterator as [iter.Seq][V].
// While consuming values from the iterator, it might conditionally yield a non-nil error.
// In that case Box stores the error and stops without yielding the value V paired to the error.
// [*Box.Err] returns that error otherwise nil.
//
// The zero Box is invalid and it must be allocated by [New].
type Box[V any] struct {
	err       error
	resumable *iterable.Resumable2[V, error]
}

// New returns a newly allocated Box.
//
// When a pair from seq contains non-nil error, Box discards a former value of that pair(V),
// then the iterator returned from [Box.IntoIter] stops.
//
// [*Box.Stop] must be called to release resource regardless of usage.
func New[V any](seq iter.Seq2[V, error]) *Box[V] {
	return &Box[V]{
		resumable: iterable.NewResumable2(seq),
	}
}

// Stop releases resources allocated by [New].
// After calling Stop, iterators returned from [Box.IntoIter] produce no more data.
func (b *Box[V]) Stop() {
	b.resumable.Stop()
}

// IntoIter returns an iterator which yields values from the input iterator.
//
// As the name IntoIter suggests, the iterator is stateful;
// breaking and calling seq again continues its iteration without replaying data.
// If the iterator finds an error, it stops iteration and will no longer produce any data.
// In that case the error can be inspected by calling [*Box.Err].
func (b *Box[V]) IntoIter() iter.Seq[V] {
	return func(yield func(V) bool) {
		for v, err := range b.resumable.IntoIter2() {
			if err != nil {
				b.Stop()
				b.err = err
				return
			}
			if !yield(v) {
				return
			}
		}
	}
}

// Err returns an error the input iterator has returned.
// If the iterator has not yet encountered an error, Err returns nil.
func (b *Box[V]) Err() error {
	return b.err
}
```

前述の`*(json|xml).Decoder`/`*sql.Rows`/`Nexter interface{ Next() bool; Err() error }`を向けのラッパーはあらかじめ定義しておきます。

```go
type JsonDecoder struct { *Box[json.Token];	Dec *json.Decoder }
type XmlDecoder struct { *Box[xml.Token]; Dec *xml.Decoder }
type SqlRows[V any] struct { *Box[V] }
type Nexter[V any] struct { *Box[V] }
```

### Example

上記したものをそれぞれ使ったエラーハンドリングの例を示します。

```go
// Example error handle demonstrates various way to handle error.
func Example_error_handle() {
	var (
		errSample  = errors.New("sample")
		errSample2 = errors.New("sample2")
	)

	erroneous := hiter.Pairs(
		hiter.Range(0, 6),
		xiter.Concat(
			hiter.Repeat(error(nil), 2),
			hiter.Repeat(errSample2, 2),
			hiter.Once(errSample),
			hiter.Once(error(nil)),
		),
	)

	fmt.Println("TryFind:")
	v, idx, err := hiter.TryFind(func(i int) bool { return i > 0 }, erroneous)
	fmt.Printf("v = %d, idx = %d, err = %v\n", v, idx, err)
	v, idx, err = hiter.TryFind(func(i int) bool { return i > 5 }, erroneous)
	fmt.Printf("v = %d, idx = %d, err = %v\n", v, idx, err)
	fmt.Println()
	// TryFind:
	// v = 1, idx = 1, err = <nil>
	// v = 0, idx = -1, err = sample2

	fmt.Println("TryForEach:")
	err = hiter.TryForEach(func(i int) { fmt.Printf("i = %d\n", i) }, erroneous)
	fmt.Printf("err = %v\n", err)
	fmt.Println()
	// TryForEach:
	// i = 0
	// i = 1
	// err = sample2

	fmt.Println("TryReduce:")
	collected, err := hiter.TryReduce(func(c []int, i int) []int { return append(c, i) }, nil, erroneous)
	fmt.Printf("collected = %#v, err = %v\n", collected, err)
	fmt.Println()
	// TryReduce:
	// collected = []int{0, 1}, err = sample2

	fmt.Println("HandleErr:")
	var handled error
	collected = slices.Collect(
		sh.HandleErr(
			func(i int, err error) bool {
				handled = err
				return errors.Is(err, errSample2)
			},
			erroneous,
		),
	)
	fmt.Printf("collected = %#v, err = %v\n", collected, handled)
	fmt.Println()
	// HandleErr:
	// collected = []int{0, 1}, err = sample

	fmt.Println("*errbox.Box:")
	box := errbox.New(erroneous)
	collected = slices.Collect(box.IntoIter())
	fmt.Printf("collected = %#v, err = %v\n", collected, box.Err())
	fmt.Println()
	// *errbox.Box:
	// collected = []int{0, 1}, err = sample2
}
```

## async work

iteratorから得られたデータを別のgoroutineで処理する方法について考えます。

### ForEachGo

シンプルに[\*errgroup.Group](https://pkg.go.dev/golang.org/x/sync/errgroup#Group)で処理できるラッパーを実装します。

```go
type GoGroup interface {
	Go(f func() error)
	Wait() error
}

// ForEachGo iterates over seq and calls fn with every value from seq in G's Go method.
// After all values are consumed, the result of Wait is returned.
// You may want to use [*errgroup.Group](https://pkg.go.dev/golang.org/x/sync/errgroup#Group) as implementor.
func ForEachGo[V any, G GoGroup](ctx context.Context, g G, fn func(context.Context, V) error, seq iter.Seq[V]) error {
	for v := range seq {
		g.Go(func() error {
			return fn(ctx, v)
		})
	}
	return g.Wait()
}

func ForEachGo2[K, V any, G GoGroup](ctx context.Context, g G, fn func(context.Context, K, V) error, seq iter.Seq2[K, V]) error
```

これはわざわざ実装することはない気もしますが、網羅性のためには必要です！

実は以下のように、`fn`がパニックしたときの手当ても実装しておこうかと思っていたんですが、

```go
func ForEachGo[V any, G GoGroup](ctx context.Context, g G, fn func(context.Context, V) error, seq iter.Seq[V]) error {
	var (
		panicOnce sync.Once
		panicVal any
	)
	for v := range seq {
		g.Go(func() (err error) {
			defer func() {
				rec := recover()
				if rec != nil {
					var first bool
					panicOnce.Do(func() {
						first = true
						panicVal = rec
					})
					if first {
						err = fmt.Errorf("panicked: %v", rec)
					}
				}
			}()
			err = fn(ctx, v)
			return
		})
	}
	err := g.Wait()
	if panicVal != nil {
		panic(err)
	}
	return err
}
```

`*errgroup.Group`が改善されたときに重複になって無駄なのでとりあえず`GoGroup`の実装がなんとかしろで置いておくことにします。

### channel

`Chan`で`chan V`を`iter.Seq[V]`に変換し`ChanSend`で`iter.Seq[V]`のすべてのデータをchannelに送信します。
別のgoroutineで動作するworker上でchannelからデータを受信し、なにかしらの関数を実行して結果を送り返します。
channelは１つの値しか送れないため、エラーしうる関数の結果を送り返すには`KeyValue[V, error]`に詰めて送信します。
`FromKeyValue`で`iter.Seq[KeyValue[K, V]]`を`iter.Seq2[K, V]`に変換します。網羅性のためだけにこれを実装しました。

これは普通な感じですね。無理してiteratorで処理してる感があります。
単にchannel sendしてchannel receiveしているだけなのでworkerから帰ってくるデータの順序が送った順序と一致しているわけではないのがポイントです。

```go
// Chan returns an iterator over ch.
// Either cancelling ctx or closing ch stops iteration.
func Chan[V any](ctx context.Context, ch <-chan V) iter.Seq[V] {
	return func(yield func(V) bool) {
		for {
			select {
			case <-ctx.Done():
				return
			default:
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
}

// ChanSend sends values from seq to c.
// It unblocks after either ctx is cancelled or all values from seq are consumed.
// sentAll is true only when all values are sent to c.
// Otherwise sentAll is false and v is the value that is being blocked on sending to the channel.
func ChanSend[V any](ctx context.Context, c chan<- V, seq iter.Seq[V]) (v V, sentAll bool) {
	for v := range seq {
		select {
		case <-ctx.Done():
			return v, false
		case c <- v:
		}
	}
	return *new(V), true
}

// FromKeyValue unwraps iter.Seq[KeyValue[K, V]] into iter.Seq2[K, V]
// serving as a counterpart for [ToKeyValue].
//
// In case values from seq needs to be sent through some data transfer mechanism
// that only allows data to be single value, like channels,
// some caller might decide to wrap values into KeyValue[K, V], maybe by [ToKeyValue].
// If target helpers only accept iter.Seq2[K, V], then FromKeyValues is useful.
func FromKeyValue[K, V any](seq iter.Seq[KeyValue[K, V]]) iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {
		for kv := range seq {
			if !yield(kv.K, kv.V) {
				return
			}
		}
	}
}

func Example_async_worker_channel() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	works := []string{"foo", "bar", "baz"}

	in := make(chan string, 5)
	out := make(chan hiter.KeyValue[string, error])

	var wg sync.WaitGroup
	wg.Add(3)
	for range 3 {
		go func() {
			defer wg.Done()
			_, _ = hiter.ChanSend(
				ctx,
				out,
				xiter.Map(
					func(s string) hiter.KeyValue[string, error] {
						return hiter.KeyValue[string, error]{
							K: "✨" + s + "✨" + s + "✨",
							V: nil,
						}
					},
					hiter.Chan(ctx, in),
				),
			)
		}()
	}

	var wg2 sync.WaitGroup
	wg2.Add(1)
	go func() {
		defer wg2.Done()
		wg.Wait()
		close(out)
	}()

	_, _ = hiter.ChanSend(ctx, in, slices.Values(works))
	close(in)

	results := maps.Collect(hiter.FromKeyValue(hiter.Chan(ctx, out)))

	for result, err := range iterable.MapSorted[string, error](results).Iter2() {
		fmt.Printf("result = %s, err = %v\n", result, err)
	}

	wg2.Wait()

	// Output:
	// result = ✨bar✨bar✨, err = <nil>
	// result = ✨baz✨baz✨, err = <nil>
	// result = ✨foo✨foo✨, err = <nil>
}
```

### async.Map

前記のchannel版と違って入力のデータの順序を保ったまま別のgoroutineで処理を行うことができる`Map`です。
これも前述の[こちらのスクラップ](https://zenn.dev/macopy/scraps/55ae36007fc8ce)を見てなるほどと思って実装したものです。違いとしてはメモリ消費量がunboundではないのでたくさん要素を返す`iter.Seq[V]`+consumerのほうが遅いという状況でメモリ消費量がかさんでいかないのが違いです。

- `reservations chan chan hiter.KeyValue[V, error]`を定義し、
- `iter.Seq[V]`がデータを返すたびに`chan hiter.KeyValue[V, error]`を*reservation*としてこのチャネルに送信します。
- `Map`が返すiteratorは`for reservation := range reservations`という形でrange-over-channelすることで返すデータの順序を保証します。
- `mapper`の呼び出しは`go func() {}()`で新しい`goroutine`の中で行います。
- `mapper`の結果は*reservation*を通じて送信されます。
- `reservations`のバッファーサイズがトータルのメモリ消費量の制限、`mapper`を呼び出す`goroutine`の個数の制限が`mapper`のconcurrentな実行数の制限となります。
- さらに、channelのcloseや処理が中断された場合への少しややこしい考慮を加え、
- 新しく作られた各`goroutine`内で起きたパニックを拾って`Map`の中でre-panicする手当を実装して完成です。

`go`で新しい`goroutine`を呼び出し出すといろいろ考慮が必要でコードが一気に膨らみますね。

- 順序を保つ都合上、どこかで処理に長いことかかる要素が出てくるとそこで詰まって全体が止まります。
  - 要素ごとに`mapper`処理時間はあまりばらつかないという前提があります。
- 実装の都合上渡されたbreakされたり、ctxがcancelされたりすると`iter.Seq[V]`から受け取ったデータが宙に浮いたまま消費されずに忘れさらられることがあり得ます。
  - そのためキャンセルしたい場合`iter.Seq[V]`と`mapper`をキャンセルする穏当なキャンセルをさらに実装したほうがいいかもしれないです。

```go
type resultLike[V any] hiter.KeyValue[V, error]

// Map maps values from seq asynchronously.
//
// Map applies mapper to values from seq in separate goroutines while keeping output order.
// When the order does not need to be kept, just send all values to workers through a channel using [hiter.ChanSend]
// and receive results via other channel using [hiter.Chan].
//
// queueLimit limits max amounts of possible simultaneous resource allocated.
// queueLimit must not be less than 1, otherwise Map panics.
// workerLimit limits max possible concurrent invocation of mapper.
// workerLimit is ok to be less than 1.
// When queueLimit > workerLimit, the total number of workers is only limited by queueLimit.
//
// Cancelling ctx may stop the returned iterator early.
// mapper should respect the ctx, otherwise it delays the iterator to return.
//
// If mapper panics Map panics with the first panic value.
//
// The counter part of Map for [iter.Seq2][K, V] does not exist since mapper is allowed to return error as second ret value.
// If you need to map [iter.Seq2][K, V], convert it into [iter.Seq][V] by [hiter.ToKeyValue].
func Map[V1, V2 any](
	ctx context.Context,
	queueLimit int,
	workerLimit int,
	mapper func(context.Context, V1) (V2, error),
	seq iter.Seq[V1],
) iter.Seq2[V2, error] {
	if queueLimit <= 0 {
		panic("queueLimit must be greater than 0")
	}
	return func(yield func(V2, error) bool) {
		ctx, cancel := context.WithCancel(ctx)
		defer cancel()

		var (
			reservations = make(chan chan resultLike[V2], queueLimit-1)
			workerSem    chan struct{}
			wg           sync.WaitGroup
			panicVal     any
			panicOnce    sync.Once
		)
		if workerLimit > 0 {
			workerSem = make(chan struct{}, workerLimit)
		}

		recordPanicOnce := func() {
			rec := recover()
			if rec != nil {
				cancel()
				panicOnce.Do(func() {
					panicVal = rec
				})
			}
		}

		wg.Add(1)
		go func() {
			defer func() {
				wg.Done()
				close(reservations)
			}()
			defer recordPanicOnce()
			for v := range seq {
				rsv := make(chan resultLike[V2], 1)

				select {
				case <-ctx.Done():
					return
				case reservations <- rsv:
				}

				if workerSem != nil {
					select {
					case <-ctx.Done():
						// close rsv in all paths
						close(rsv)
						return
					case workerSem <- struct{}{}:
					}
				}

				wg.Add(1)
				go func() {
					defer func() {
						close(rsv)
						if workerSem != nil {
							<-workerSem
						}
						wg.Done()
					}()
					defer recordPanicOnce()
					v2, err := mapper(ctx, v)
					rsv <- resultLike[V2]{K: v2, V: err}
				}()
			}
		}()

		defer func() {
			cancel()
			wg.Wait()
			if panicVal != nil {
				panic(panicVal)
			}
		}()
		// record panic in case simultaneously multiple sources have panicked.
		defer recordPanicOnce()
		for rsv := range reservations {
			result, ok := <-rsv
			if !ok {
				// TODO: ignore when ok is false and return?
				continue
			}
			if !yield(result.K, result.V) {
				return
			}
		}
	}
}
```

キャンセルの例は適当に以下のような感じです。

```go
func Example_async_worker_map_graceful_cancellation() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	works := []string{"foo", "bar", "baz"}

	workerCtx, cancelWorker := context.WithCancel(context.Background())
	defer cancelWorker()

	for result, err := range async.Map(
		ctx,
		/*queueLimit*/ 1,
		/*workerLimit*/ 1,
		/*mapper*/ func(ctx context.Context, s string) (string, error) {
			combined, cancel := context.WithCancel(ctx)
			defer cancel()
			go func() {
				select {
				case <-ctx.Done():
				case <-combined.Done():
				case <-workerCtx.Done():
				}
				cancel()
			}()
			if combined.Err() != nil {
				return "", combined.Err()
			}
			return "✨" + s + "✨" + s + "✨", nil
		},
		sh.Cancellable(1, workerCtx, slices.Values(works)),
	) {
		fmt.Printf("result = %s, err = %v\n", result, err)
		cancelWorker()
	}
	// Output:
	// result = ✨foo✨foo✨, err = <nil>
	// result = ✨bar✨bar✨, err = <nil>
}
```

### async.Chunk

`slices.Chunk`で`[]T`を`iter.Seq[[]T]`に変換可能ですが、`iter.Seq[V]` -> `iter.Seq[[]V]`の変換は存在しません。
何かの都合でn要素ずつまとめて処理したいという需要はあると見越してそういったものを実装します。

ただし、前述の`chan V`、`*sql.Rows`、`Dec interface { Decode(any) error }`から変換された`iter.Seq[V]`はioを挟む都合上どこかで長時間ブロックすることがありあえます。
タイムアウトが実装されないと`n`個に到達しないが取得された要素が下流に流されず宙に浮いたままになることがあり得ます。

そこでタイムアウトを持ったChunk処理を実装します。

- 別の`goroutine`の中で`for range seq`で要素を取り出し、channelに送信します。
- `Chunk`呼び出し`goroutine`でchannelからデータを受信し、
- 受信のたびにタイマーを`timeout time.Duration`でリセットします。
- タイムアウトするかn要素取得できた時iteratorから`[]V`が得られます。

こちらも上記の`async.Map`と同様にbreakされると`iter.Seq[V]`から受け取ったデータが宙に浮いたまま消費されずに忘れさらられることがあり得ます。
こちらも同様に引数に渡す`iter.Seq[V]`のほうをキャンセルするとよいでしょう。

```go
var (
	clock = clockwork.NewRealClock()
)

// Chunk returns an iterator over consecutive values of up to n elements from seq.
//
// The returned iterator reuses the buffer it yields. Apply [sh.Clone] if the caller needs to retain slices.
//
// Chunk may yield slices where 0 < len(s) <= n.
// Values may be shorter than n if timeout > 0 and the timeout duration passed since last time seq generated a value.
//
// Chunk panics if n is less than 1.
func Chunk[V any](timeout time.Duration, n int, seq iter.Seq[V]) iter.Seq[[]V] {
	if n <= 0 {
		panic("n cannot be less than 1")
	}
	return func(yield func([]V) bool) {
		var wg sync.WaitGroup

		ctx, cancel := context.WithCancel(context.Background())

		var (
			ch        = make(chan V)
			panicVal  any
			panicOnce sync.Once
		)

		recordPanicOnce := func() {
			rec := recover()
			if rec != nil {
				cancel()
				panicOnce.Do(func() {
					panicVal = rec
				})
			}
		}

		wg.Add(1)
		go func() {
			defer func() {
				close(ch)
				wg.Done()
			}()
			defer recordPanicOnce()
			for v := range seq {
				select {
				case <-ctx.Done():
					return
				case ch <- v:
				}
			}
		}()

		timer := clock.NewTimer(timeout)
		// reset when first value is yielded.
		// As of Go 1.23, this is really safe.
		timer.Stop()

		defer func() {
			cancel()
			wg.Wait()
			if panicVal != nil {
				panic(panicVal)
			}
		}()
		// record panic in case both seq and consumer have panicked.
		defer recordPanicOnce()

		buf := make([]V, n)
		idx := 0
		for {
			select {
			case v, ok := <-ch:
				if !ok {
					if idx > 0 {
						yield(buf[:idx:idx])
					}
					return
				}

				buf[idx] = v
				idx++
				if idx != n {
					if timeout > 0 {
						timer.Reset(timeout)
					}
				} else {
					if !yield(buf) {
						return
					}
					timer.Stop()
					idx = 0
				}
			case <-timer.Chan():
				if idx > 0 {
					if !yield(buf[:idx:idx]) {
						return
					}
					idx = 0
				}
			}
		}
	}
}
```

## cancellation

皆さんは大きなファイルの`io.Copy(w, r)`がcancellableになっていないせいで`Ctrl+C`が効かなくて痛い目にあったことはありませんか？筆者はあります。頻繁にあります。

それで毎度`io.Reader`をcancellableにするラッパーを定義しています。あんまりに毎度実装しているので自作ライブラリに加えておきました。

https://github.com/ngicks/go-fsys-helper/blob/e7a9c10b017198e539f01bf9f0444420abec8cab/stream/cancellable.go#L8-L51

これがiteratorで起きるかと言われると微妙なところです。ただ以下のように、`Decoder` -> `Encoder`のラウンドトリップをiteratorで実装している場合、長い処理時間を要するiterator間の処理というのはあり得るはあり得ます。

以下の例では入力は数行の`ndjson`(newline-delimited json)ですが、大きなファイルの中身を処理していた場合は長時間この処理でブロックすることもあり得ます。

```go
// Decode returns an iterator over consecutive decode results of dec.
//
// The iterator stops if and only if dec returns io.EOF. Handling other errors is caller's responsibility.
// If the first error should stop the iterator, use [LimitUntil], [LimitAfter] or [*errbox.Box].
func Decode[V any, Dec interface{ Decode(any) error }](dec Dec) iter.Seq2[V, error] {
	return func(yield func(V, error) bool) {
		for {
			var v V
			err := dec.Decode(&v)
			if err == io.EOF {
				return
			}
			if !yield(v, err) {
				return
			}
		}
	}
}

func Encode[V any, Enc interface{ Encode(v any) error }](enc Enc, seq iter.Seq[V]) error {
	for v := range seq {
		if err := enc.Encode(v); err != nil {
			return err
		}
	}
	return nil
}

func Example_dec_enc_round_trip() {
	src := []byte(`
	{"foo":"foo"}
	{"bar":"bar"}
	{"baz":"baz"}
	`)

	rawDec := json.NewDecoder(bytes.NewReader(src))
	dec := errbox.New(hiter.Decode[map[string]string](rawDec))
	defer dec.Stop()

	enc := json.NewEncoder(os.Stdout)

	err := hiter.Encode(
		enc,
		xiter.Map(
			func(m map[string]string) map[string]string {
				return maps.Collect(
					xiter.Map2(
						func(k, v string) (string, string) { return k + k, v + v },
						maps.All(m),
					),
				)
			},
			dec.IntoIter(),
		),
	)

	fmt.Printf("dec error = %v\n", dec.Err())
	fmt.Printf("enc error = %v\n", err)
	// Output:
	// {"foofoo":"foofoo"}
	// {"barbar":"barbar"}
	// {"bazbaz":"bazbaz"}
	// dec error = <nil>
	// enc error = <nil>
}
```

ということで例によって例のごとく、必要性・実用性は置いといて網羅性のためにiteratorをcancellableにするラッパーを実装します。
１要素の処理スピードによっては`ctx.Err`でmutexを取得する部分ほうが長いとか普通にあり得るかなと思ってn要素ごとにチェックができるようにしてあります。
とりあえずn<=1にしておけば要素毎にチェックされるので大抵はそうしておけばよいでしょう！

```go
// Cancellable returns an iterator over seq but it also checks if ctx is cancelled each n elements seq yields.
func Cancellable[V any](n int, ctx context.Context, seq iter.Seq[V]) iter.Seq[V] {
	return hiter.CheckEach(n, func(V, int) bool { return ctx.Err() == nil }, seq)
}

// Cancellable2 returns an iterator over seq but it also checks if ctx is cancelled each n elements seq yields.
func Cancellable2[K, V any](n int, ctx context.Context, seq iter.Seq2[K, V]) iter.Seq2[K, V]

// CheckEach returns an iterator over seq.
// It also calls check after each n values yielded from seq.
// fn receives a value from seq and the current count of it to inspect and decide to stop.
// n is capped to 1 if it is less.
func CheckEach[V any](n int, check func(v V, i int) bool, seq iter.Seq[V]) iter.Seq[V] {
	if n <= 1 {
		// The specialized case...Does it have any effect?
		return func(yield func(V) bool) {
			i := 0
			for v := range seq {
				if !check(v, i) {
					return
				}
				i++
				if !yield(v) {
					return
				}
			}
		}
	}
	return func(yield func(V) bool) {
		i := 0
		nn := n
		for v := range seq {
			nn--
			if nn == 0 {
				if !check(v, i) {
					return
				}
				nn = n
			}
			i++
			if !yield(v) {
				return
			}
		}
	}
}

// CheckEach2 returns an iterator over seq.
// It also calls check after each n key-value pairs yielded from seq.
// fn receives a key-value pair from seq and the current count of it to inspect and decide to stop.
// n is capped to 1 if it is less.
func CheckEach2[K, V any](n int, check func(k K, v V, i int) bool, seq iter.Seq2[K, V]) iter.Seq2[K, V]
```
