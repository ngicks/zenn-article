---
title: "Goのrange over func proposalを試してみる"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

# Goのrange over func proposal

https://github.com/golang/go/issues/61405

上記のproposalで述べられているrange over funcを試し見てみます。

# 対象読者

- [The Go Programming Language](https://go.dev/)に対してある程度の習熟している
- 上記のproposalのinterfaceの感触が気になってるけど自分で実装してみるほどじゃない

# この記事のすること

- 上記issueの特記すべき(というか筆者が気になった)ところを取り上げます
- sliceやmapからこのiterator funcを作る関数を書いてみます
- iterator funcから得られる値を加工するadapter funcを書きます
  - e.g. filter, map

# Proposalの概要

以下のように、`func(func()bool)`, `func(func(V)bool)`, `func(yield func(k K, v V) bool)`を`for-range`で消費できるようにするというproposalです

```go
package main

import (
	"fmt"

	"github.com/ngicks/go-iterator-helper/iteratorhelper"
)

func main() {
	var iter func(yield func(k int, v string) bool)
	iter = iteratorhelper.SliceIter([]string{"foo", "bar", "baz", "qux", "quux"})
	var adaptedIter func(yield func(k []int, v []string) bool)
	adaptedIter = iteratorhelper.Chunk(iter, 2)

	for k, v := range adaptedIter {
		fmt.Printf("k = %#v, v = %#v\n", k, v)
        /*
        k = []int{0, 1}, v = []string{"foo", "bar"}
        k = []int{2, 3}, v = []string{"baz", "qux"}
        k = []int{4}, v = []string{"quux"}
        */
	}
}
```

## signature

上記リンクトップのコメントではspec textに以下を追加すると述べています。
実際にはリンク先には`func(func()bool) bool`という感じで`bool`の返り値がついているんですが、gotipで動かせる現状版だと下記が正しいです。コメントでも、proposalからこのboolの返り値が取り除かれていないのは誤りであると述べられています。

```
Range expression                                   1st value          2nd value

array or slice      a  [n]E, *[n]E, or []E         index    i  int    a[i]       E
string              s  string type                 index    i  int    see below  rune
map                 m  map[K]V                     key      k  K      m[k]       V
channel             c  chan E, <-chan E            element  e  E
integer             n  integer type                index    i int
function, 0 values  f  func(func()bool)
function, 1 value   f  func(func(V)bool)           value    v  V
function, 2 values  f  func(func(K, V)bool)        key      k  K      v          V
```

integer以下が追加行です。`range over integer`はこの記事ではスコープから外します。

3つのシグネチャがあります。おそらく下二つはslice, mapをrange overすることを念頭に置いたsignatureでしょう。

これらのiterator funcは、 `for range`で消費することができます。

```go
for x, y := range f { ... }
for x, _ := range f { ... }
for _, y := range f { ... }
for x := range f { ... }
for range f { ... }
```

実際にはこれらは

```go
for x, y := range f { ... }
```

を

```go
f(func(x T1, y T2) bool { ... })
```

にrewriteすることで動作を実現します。

## プロトタイプ実装

gotipを使うと実行できます。

```
go install golang.org/dl/gotip@latest
gotip download 510541
GOEXPERIMENT=range gotip run ...
```

## Discussion Summary / FAQ の気になるところ(2023/07/23時点)

https://github.com/golang/go/issues/61405#issuecomment-1638896606

このissueコメントでモチベーション、その他もろもろが述べられていますが、そのうち筆者が気になったものだけピックアップします

### What if the iterator function ignores yield returning false? / What if the iterator function saves yield and calls it after returning?

- 現在のプロトタイプ実装では特にチェックされない
  - チェックするとインライン化がうまくいかないため
- 標準化の際にはパフォーマンスに過度な影響が出ないようにチェックする必要がある
  - 間違った使用法でpanicするように

### What if the iterator function calls yield on a different goroutine?

- まだ決まってない
  - Goはgoroutineを特定できないようにしてきたので、yieldだけ「iteratorが呼びだれたgoroutineから呼ばれなければならない(_must_ be called)」としてしまうのは変(_strange_)
  - iteratorを呼び出したgoroutine以外でyieldを呼ぶとpanicするというのはさらに変だが、だとしてもそうする価値があるかもしれない。

### What happens if the iterator function recovers a panic in the loop body?

- まだ決まってない
  - 現在のプロトタイプはrecover後もう一度yieldを呼んで処理を続けることを許す
    - これはGoroutineの挙動とは似通っていない
    - もしかしたらこれは間違いかもしれないが、だとしても、効率的にただすことが難しい。
- 継続的に検討する

### Can range over a function perform as well as hand-written loops?

yes.

このループは

```go
for i, x := range slices.Backward(x) {
	fmt.Println(i, x)
}
```

いかにダイレクトに変換され、

```go
slices.Backward(s)(func(i int, x string) bool {
	fmt.Println(i, x)
	return true
})
```

slices.Backwardが小さいため以下のようにインライン化され

```go
func(yield func(int, E) bool) bool {
	for i := len(s)-1; i >= 0; i-- {
		if !yield(i, s[i]) {
			return false
		}
	}
	return true
}(func(i int, x string) bool {
	fmt.Println(i, x)
	return true
})
```

関数リテラルが即座に呼ばれていることがわかるのでインライン化され、

```go
{
	yield := func(i int, x string) bool {
		fmt.Println(i, x)
		return true
	}
	for i := len(s)-1; i >= 0; i-- {
		if !yield(i, s[i]) {
			goto End
		}
	}
End:
}
```

この場合yieldが*devirtualize*できる

```go
{
	for i := len(s)-1; i >= 0; i-- {
		if !(func(i int, x string) bool {
			fmt.Println(i, x)
			return true
		})(i, s[i]) {
			goto End
		}
	}
End:
}
```

さらに関数リテラルがインライン化できる

```go
{
	for i := len(s)-1; i >= 0; i-- {
		var ret bool
		{
			i := i
			x := s[i]
			fmt.Println(i, x)
			ret = true
		}
		if !ret {
			goto End
		}
	}
End:
}
```

この時点で、[SSAバックエンド](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/README.md)が不必要な変数などを判定し、上記のコードを以下と同等に扱う

```go
for i := len(s)-1; i >= 0; i-- {
	fmt.Println(i, s[i])
}
```

この最適化は閾値以下の単純なloop bodyと単純なiteratorに対してのみおこなわれる。逆に複雑なloop bodyやiteratorに対しては関数呼び出しのオーバーヘッドは重大ではない・・・そうです。

(多分コンパイルの速さは損なわれないということを言いたいのかな)

# 実装

実際に実装してみてgotipで実行してみましょう。

書いたコードはこのrepositoryに入っています

https://github.com/ngicks/go-iterator-helper

## base

- `slice`, `map`ベースのfuncは`slices`, `maps`のパッケージで実装されるでしょうから不要ですが、実験としては必要なので作ります。
- channelは元から`for rage`できますが、filterやmapのような加工関数に適合できるようにfuncへ変換する関数を作っておきました。

```go
func SliceIter[T ~[]E, E any](sl T) func(yield func(k int, v E) bool) {
	return func(yield func(k int, v E) bool) {
		for idx, v := range sl {
			if !yield(idx, v) {
				return
			}
		}
	}
}

func SliceIterSingle[T ~[]E, E any](sl T) func(yield func(v E) bool) {
	return func(yield func(v E) bool) {
		for _, v := range sl {
			if !yield(v) {
				return
			}
		}
	}
}

func MapIter[K comparable, V any](m map[K]V) func(yield func(k K, v V) bool) {
	return func(yield func(k K, v V) bool) {
		for k, v := range m {
			if !yield(k, v) {
				return
			}
		}
	}
}

func ChanIter[V any](ch <-chan V) func(yield func(V) bool) {
	return func(yield func(V) bool) {
		for v := range ch {
			if !yield(v) {
				return
			}
		}
	}
}
```

## base for custom-container

- 前述のproposalに含まれる`range over int`で、0からnの`loop`はコンパイラサポートによってできるようになることになっていますが、nからmのループは頻出パターンではないものとして考慮から外すようなことがプロポーザルで述べられていますね。なのでnからmのfuncを作っときます。
- このproposalの目的の一つに、「genericsの追加によってもたらされたcustom containerに統一的なiterate-overのinterfaceを与えること」のようなことが書いてあります。custom containerの例としてordered mapが上がっていますので、例として作っておきました。
- 同じく、`strings.Line()`のようなものが追加できると述べています。`bufio.Scanner`をfunction iteratorに適合させたアダプタも作ってみます。

```go
package iteratorhelper

import (
	"bufio"
	"io"

	orderedmap "github.com/wk8/go-ordered-map/v2"
)

func RangeIter(start, end int) func(yield func(int) bool) {
	return func(yield func(int) bool) {
		for i := start; i < end; i++ {
			if !yield(i) {
				return
			}
		}
	}
}

func OrderedMapIter[K comparable, V any](o *orderedmap.OrderedMap[K, V]) func(yield func(k K, v V) bool) {
	return func(yield func(k K, v V) bool) {
		for pair := o.Oldest(); pair != nil; pair = pair.Next() {
			if !yield(pair.Key, pair.Value) {
				return
			}
		}
	}
}

func Scan(r io.Reader, split bufio.SplitFunc) func(yield func(text string, err error) bool) {
	scanner := bufio.NewScanner(r)
	if split != nil {
		scanner.Split(split)
	}

	return func(yield func(text string, err error) bool) {
		for scanner.Scan() {
			if scanner.Err() != nil {
				yield("", scanner.Err())
				return
			}
			if !yield(scanner.Text(), nil) {
				return
			}
		}
	}
}

```

## adapter

baseで作られたfuncを加工するadapter funcを作ります。ほかの言語で言うところのiterator methodのfilterとかmapとかですね。

筆者がよく使うのは

- Chain
- Chunk
- Enumerate
- Filter
- Map
- Skip / Take
- Window
- Zip

あたりなので、実装してみました。

- iteratorに渡されるyield funcを即時関数として、自身に渡されるyieldの呼び出しの前後でデータの加工などを行うことになります。
  - [echoなどのmiddleware](https://github.com/labstack/echo/blob/a2e7085094bda23a674c887f0e93f4a15245c439/middleware/slash.go#L44-L72)と大体似たようなものですね。
- とりあえず作ってみたっていうだけなので、`func(func(K, V)bool) bool`向けのみのadapterの未実装しています。
  - 例外として`func(func(V)bool) bool`を`func(func(K, V)bool) bool`に変換するためにEnumerateを作ってあります。
- ただしzipだけは特別な考慮が必要ですので、後述します。

```go
func Chain[K, V any](
	iter1, iter2 func(yield func(k K, v V) bool),
) func(yield func(k K, v V) bool) {
	return func(yield func(k K, v V) bool) {
		stopped := false
		iter1(func(k K, v V) bool {
			if !yield(k, v) {
				stopped = true
				return false
			}
			return true
		})
		if stopped {
			return
		}
		iter2(yield)
	}
}

func Chunk[K, V any](iter func(yield func(k K, v V) bool), size uint) func(yield func(k []K, v []V) bool) {
	if size == 0 {
		panic("Chunk: size must not be zero.")
	}
	return func(yield func(k []K, v []V) bool) {
		stopped := false
		chunkedK, chunkedV := make([]K, size), make([]V, size)
		idx := uint(0)
		iter(func(k K, v V) bool {
			if idx < size {
				chunkedK[idx] = k
				chunkedV[idx] = v
				idx++
			}
			if idx >= size {
				idx = 0
				if !yield(append([]K{}, chunkedK...), append([]V{}, chunkedV...)) {
					stopped = true
					return false
				}
			}
			return true
		})
		if !stopped && idx != 0 {
			yield(append([]K{}, chunkedK[:idx]...), append([]V{}, chunkedV[:idx]...))
		}
	}
}

func Enumerate[V any](iter func(yield func(v V) bool)) func(yield func(idx int, v V) bool) {
	return func(yield func(idx int, v V) bool) {
		var idx int
		iter(func(v V) bool {
			if !yield(idx, v) {
				return false
			}
			idx++
			return true
		})
	}
}

func FilterSelect[K, V any](
	iter func(yield func(k K, v V) bool),
	selector func(k K, v V) bool,
) func(yield func(k K, v V) bool) {
	return func(yield func(k K, v V) bool) {
		iter(func(k K, v V) bool {
			if selector(k, v) && !yield(k, v) {
				return false
			}
			return true
		})
	}
}

func FilterExclude[K, V any](
	iter func(yield func(k K, v V) bool),
	excluder func(k K, v V) bool,
) func(yield func(k K, v V) bool) {
	return func(yield func(k K, v V) bool) {
		iter(func(k K, v V) bool {
			if !excluder(k, v) && !yield(k, v) {
				return false
			}
			return true
		})
	}
}

func Map[K1, V1, K2, V2 any](
	iter func(yield func(k K1, v V1) bool),
	mapper func(k K1, v V1) (K2, V2),
) func(yield func(k K2, v V2) bool) {
	return func(yield func(k K2, v V2) bool) {
		iter(func(k K1, v V1) bool {
			if !yield(mapper(k, v)) {
				return false
			}
			return true
		})
	}
}

func SkipWhile[K, V any](
	iter func(yield func(k K, v V) bool),
	predicate func(k K, v V) bool,
) func(yield func(k K, v V) bool) {
	return func(yield func(k K, v V) bool) {
		taking := false
		iter(func(k K, v V) bool {
			if !taking && !predicate(k, v) {
				taking = true
			}
			if taking && !yield(k, v) {
				return false
			}
			return true
		})
	}
}

func TakeWhile[K, V any](
	iter func(yield func(k K, v V) bool),
	predicate func(k K, v V) bool,
) func(yield func(k K, v V) bool) {
	return func(yield func(k K, v V) bool) {
		taking := true
		iter(func(k K, v V) bool {
			if taking && !predicate(k, v) {
				return false
			}
			if taking && !yield(k, v) {
				return false
			}
			return true
		})
	}
}

func Window[K, V any](iter func(yield func(k K, v V) bool), size uint) func(yield func(k []K, v []V) bool) {
	if size == 0 {
		panic("Window: size must not be zero.")
	}
	return func(yield func(k []K, v []V) bool) {
		bufK, bufV := make([]K, size), make([]V, size)
		idx := uint(0)
		iter(func(k K, v V) bool {
			if idx < size {
				bufK[idx] = k
				bufV[idx] = v
				idx++
			} else {
				copy(bufK, bufK[1:])
				copy(bufV, bufV[1:])
				bufK[len(bufK)-1] = k
				bufV[len(bufV)-1] = v
			}

			if idx == size {
				if !yield(append([]K{}, bufK...), append([]V{}, bufV...)) {
					return false
				}
			}

			return true
		})

		if idx != size {
			_ = yield(append([]K{}, bufK[:idx]...), append([]V{}, bufV[:idx]...))
		}
	}
}
```

### zip

zipは、二つのiteratorを受けとり、両方を同時にiterator overするというiteratorです。

今回のproposalでは単なる関数を連続して呼ぶという仕組みであるため、コントロールをループの呼び出し側に戻すタイミングがありません。そのため二つのiteratorを一緒に進めるというのはcoroutineベースの実装にするか、明示的にNextメソッドを呼ぶような方式にしなければ、実際上おそらく不可能でしょう

そこで、受け取った二つのiteratorをgoroutineの中で消費し、1つ要素を受けとるたびにチャネルを通して元の実行コンテキストに送信します。

```go
func Zip[V1, V2 any](
	left func(yield func(v V1) bool),
	right func(yield func(v V2) bool),
) func(yield func(l V1, r V2) bool) {
	return func(yield func(l V1, r V2) bool) {
		iterResult := make(chan bool)
		leftCh := make(chan V1)
		rightCh := make(chan V2)

		go func() {
			left(func(v V1) bool {
				leftCh <- v
				return <-iterResult
			})
			close(leftCh)
		}()
		go func() {
			right(func(v V2) bool {
				rightCh <- v
				return <-iterResult
			})
			close(rightCh)
		}()

		defer func() {
			close(iterResult)
			for _ = range leftCh {
			}
			for _ = range rightCh {
			}
		}()

		for {
			l, okL := <-leftCh
			r, okR := <-rightCh
			if !okL || !okR {
				break
			}
			if !yield(l, r) {
				break
			}
			iterResult <- true
			iterResult <- true
		}
	}
}
```

# 感想

- 3つシグネチャがあるので、少なくとも2つずつadapterを作る必要があるのでそこが少し手間かなという感じです。
- どうやってFilterのようなアダプタ関数を定義するか一瞬わかりませんでした。
- 全体を書くのに3から4時間ぐらいかかりました。
  - VS CodeのGo extensionのサポートを正常に得られなくなりますので、エラーなどがすぐにわからなくなったためです。
    - 今思えばコンソールで`watch GOEXPERIMENT=range gotip vet ./...`を実行しておけばよかったです。
  - やっぱり普段の環境はすごく助かっているんだなっていうことがよくわかりました。
- 前からiterator protocolをユーザーのコードに向けて公開してほしいと思っていたので、この変更はすごく歓迎です。
- Goにとってあまりない構文の追加でありますので、for range loopを見て何かしらのcode generationを行っているようなコードは影響を受けますね。
- [github.com/golang/go/commit/390763ae](https://github.com/golang/go/commit/390763aed84189cb360e7ceb94aef56125fb140d), [github.com/golang/go/commit/e82cb142](https://github.com/golang/go/commit/e82cb14255cc63099e5c728676506cb4d0d97378) などでメインラインのコンパイラに`GOEXPERIMENT`ガード付きで取り込まれていますね。これが順調に進んでるとみなしていいのかよくわかりませんが。

どんどん使いやすくなって助かりますね。このまま取り込まれるのかはわかりませんが楽しみです。

# 参考

参考にさせていただいた記事です。

- https://zenn.dev/team_soda/articles/0b57bba3a3665f
