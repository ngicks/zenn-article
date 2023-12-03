---
title: "Goの1.22にGOEXPERIMENTガード下で導入されるrange over func proposalを試してみる"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

# Goのrange over func proposal

https://github.com/golang/go/issues/61405

上記のproposalで述べられているrange over funcを試し見てみます。

# 対象読者

- [The Go Programming Language](https://go.dev/)に対してある程度の習熟している
- ユーザーがカスタマイズできる`for range loop`に興味があるけどこういうproposalが出てたのは知らなかった。もしくは
- 上記のproposalのinterfaceの感触が気になってるけど自分で実装してみるほどじゃない

# この記事のすること

- 上記issueの筆者が気になったところを取り上げます
- このproposalで述べられている新しい構文を試します。
  - sliceやmapからこのiterator funcを作る関数を書いてみます
  - iterator funcから得られる値を加工するadapter funcを書きます
    - e.g. filter, map

# Proposalの概要

以下のように、

- `func(func()bool)`
- `func(func(V)bool)`
- `func(yield func(k K, v V) bool)`

を`for-range`で消費できるようにするというproposalです

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
	}
}
/*
k = []int{0, 1}, v = []string{"foo", "bar"}
k = []int{2, 3}, v = []string{"baz", "qux"}
k = []int{4}, v = []string{"quux"}
*/
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

このコメントは2023-07-23から現在(2023-12-04)まで更新されていません。

### What if the iterator function ignores yield returning false? / What if the iterator function saves yield and calls it after returning?

- 現在のプロトタイプ実装では特にチェックされない
  - チェックするとインライン化がうまくいかないため
- 標準化の際にはパフォーマンスに過度な影響が出ないようにチェックする必要がある
  - 間違った使用法でpanicするように

### What if the iterator function calls yield on a different goroutine?

- まだ決まってない
  - Goはgoroutineを特定できないようにしてきたので、yieldだけ「iteratorが呼びだれたgoroutineから呼ばれなければならない(_must_ be called)」としてしまうのは変
  - iteratorを呼び出したgoroutine以外でyieldを呼ぶとpanicするというのはさらに変だが、だとしてもそうする価値があるかもしれない。

### What happens if the iterator function recovers a panic in the loop body?

- まだ決まってない
  - 現在のプロトタイプはrecover後もう一度yieldを呼んで処理を続けることを許す
    - これはGoroutineの挙動とは似通っていない
    - もしかしたらこれは間違いかもしれないが、だとしても、効率的にただすことが難しい。
- 継続的に検討する

### Can range over a function perform as well as hand-written loops?

はい。

このループは

```go
for i, x := range slices.Backward(x) {
	fmt.Println(i, x)
}
```

以下にダイレクトに変換され、

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

この最適化は閾値以下の単純なloop bodyと単純なiteratorに対してのみおこなわれる。逆に複雑なloop bodyやiteratorに対しては関数呼び出しのオーバーヘッドは重大ではない。

# 現状

- https://github.com/golang/go/commit/e82cb14255cc63099e5c728676506cb4d0d97378
  - プロポーザルの準備。mainブランチへの変更。
- https://github.com/golang/go/issues/61897
  - 新しいstd package `iter`を追加しようというproposal
- https://github.com/golang/go/issues/61898
  - 新しいsub repositoryのpackage `x/exp/xiter`を追加しようというproposal
- https://github.com/golang/go/issues/64277
  - `GOEXPERIMENT=rangefunc-limited`付きでメインラインに取り込まれることになりそうです
- [このコメントによれば](https://github.com/golang/go/issues/61897#issuecomment-1790799275)、`range-over-func`は`1.22`で`GOEXPERIMENT`ガード下で利用化になります。`1.23`で正式に実装される可能性があります。

# 実装

実際にproposalの構文でiteratorやadapterを実装してみてgotipで実行してみましょう。

上記`iter`および`x/exp/xiter`のproposalで基本的なアダプター類の実装例は述べられています。以下の実装はそれらと重複するところがあります。

書いたコードはこのrepositoryに入っています

https://github.com/ngicks/go-iterator-helper

## base

元となるデータソースからiteratorを生成します。

proposalが通過し、周辺ライブラリが追従するころには`slices`、`maps`パッケージでsliceやmapからiteratorを生成する関数は提供されるでしょうから自分で実装する必要はないと思いますが、実験として必要なので実装します。

ここは特にいうことはないですね。

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

sliceやmapは元から`for range loop`でiterate可能でしたが、このproposalではほかのデータソースからもiteratorを作成できます。そのためのサンプルを以下に示します。

- `RangeIter`: nからmの数値型を列挙する
  - 前述のproposalには`range over int`が含まれ、この構文で、0から`n`を列挙できます。0 to nは頻出で一般的であるとされる一方で、`n to m`は比較的一般的でないとみなされたようです。
- `OrderedMapIter`: `github.com/wk8/go-ordered-map/v2`のOrderedMapの要素を古いものから新しいものに向けて列挙くする
  - このproposalの目的の一つに、「genericsの追加によってもたらされたcustom containerに統一的なiterate-overのinterfaceを与えること」のようなことが書いてあります。custom containerの例としてordered mapが上がっていますので、例として作っておきました。
- `Scan`: `io.Reader`を読んでテキストとして解釈し、`bufio.SplitFunc`に応じて列挙する
  - 同じく、proposalには`strings.Line()`のようなものが追加できると述べています。`bufio.Scanner`をfunction iteratorに適合させたアダプタも作ってみます。

```go
package iteratorhelper

import (
	"bufio"
	"io"

	orderedmap "github.com/wk8/go-ordered-map/v2"
)

type Numeric interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
		~float32 | ~float64
}

func RangeIter[T Numeric](start, end T) func(yield func(T) bool) {
	return func(yield func(T) bool) {
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
			if !yield(scanner.Text(), nil) {
				return
			}
		}
		if scanner.Err() != nil {
			_ = yield("", scanner.Err())
			return
		}
	}
}
```

## adapter

baseで作られたfuncを加工するadapter funcを作ります。ほかの言語で言うところのiterator methodのfilterとかmapとかですね。

iteratorに渡されるyield funcを即時関数として、自身に渡されるyieldの呼び出しの前後でデータの加工などを行うことになります。[echoなどのmiddleware](https://github.com/labstack/echo/blob/a2e7085094bda23a674c887f0e93f4a15245c439/middleware/slash.go#L44-L72)と大体似たような形式で行いますね。

筆者がよく使うのは

- Chain
- Chunk
- Enumerate
- Filter
- Map
- Skip / Take
- Window
- Zip

あたりなので、これらを実装してみます。

### Chain

渡されたiteratorを順番に一つずつiterateしていきます。yieldがfalseを返した後にyieldを呼ぶのはプロトコル違反なので適当なフラグが必要です。

```go
func Chain[K, V any](
	iters ...(func(yield func(k K, v V) bool)),
) func(yield func(k K, v V) bool) {
	return func(yield func(k K, v V) bool) {
		stopped := false
		for _, iter := range iters {
			iter(func(k K, v V) bool {
				if !yield(k, v) {
					stopped = true
					return false
				}
				return true
			})
			if stopped {
				return
			}
		}
	}
}
```

### Chunk

渡されたiterの中身をsize個ずつまとめて返します。yieldが`false`を返した後にさらにyieldを呼ぶのはプロトコル違反なので適当なフラグが必要です。

```go
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
```

yieldに渡される`[]K`および`[]V`はクローンされたものが渡されていますが、これはiterate結果が保存されるケースは十分一般的だと予想したためです。例えば、以下のように

```go
func CollectSlice[V any](iter func(yield func(v V) bool)) []V {
	var out []V
	for v := range iter {
		out = append(out, v)
	}
	return out
}

func CollectMap[K comparable, V any](iter func(yield func(k K, v V) bool)) map[K]V {
	out := make(map[K]V)
	for k, v := range iter {
		out[k] = v
	}
	return out
}
```

実際には`iter`か`x/exp/xiter`に定義されたものを使うことになると思うので深く考えてないです。

### Enumerate

ほかの言語では筆者はまあまあな頻度で使います。１要素しか返さないiteratorを受け取って、何個目の要素かを表す`int`要素を足して２要素のiteratorに変換します。

```go
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
```

### Filter

Select, Excludeです。

```go
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
```

### Map

mapperによってiterの返す値と型を変換します。

```go
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
```

### SkipWhile / TakeWhile

特定の条件が満たされる(間は|まで)iteratorの返す値を無視するようなadapterです。

```go
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
		iter(func(k K, v V) bool {
			if !predicate(k, v) {
				return false
			}
			if !yield(k, v) {
				return false
			}
			return true
		})
	}
}
```

### Window

iteratorの返す値をバッファーしてsize個のwindowを１つずつ移動しているかのような値を返すadapterです。例えば`[1,2,3,4,5]`のsliceを一要素ずつ返すiteratorがあり、size=3である場合、Windowを適用後は`[1,2,3], [2,3,4], [3,4,5]`が返されるようになります。

移動平均取るときとか、筆者はごくたまにしか使わないですが便利です。

サンプルなのでものすごくコピーが生じる実装です。実用的なものを作るならコピーを少なくするためにバッファーサイズをsizeの数倍にしてコピーの頻度を下げるとか、sliceから直接生成させるようにするとか、たぶんそういった工夫が必要ですね。

```go
func Window[K, V any](iter func(yield func(k K, v V) bool), size uint) func(yield func(k []K, v []V) bool) {
	if size == 0 {
		panic("Window: size must not be zero.")
	}
	return func(yield func(k []K, v []V) bool) {
		bufK, bufV := make([]K, size), make([]V, size)
		idx := uint(0)
		ended := false
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
					ended = true
					return false
				}
			}

			return true
		})

		if !ended && idx != size {
			_ = yield(append([]K{}, bufK[:idx]...), append([]V{}, bufV[:idx]...))
		}
	}
}
```

### Swap

2要素を返すiteratorの第1要素と第2要素を入れ替えるadapterです。たまに必要になりそうな気がする。

```go
func Swap[K, V any](iter func(yield func(k K, v V) bool)) func(yield func(v V, k K) bool) {
	return func(yield func(v V, k K) bool) {
		iter(func(k K, v V) bool {
			if !yield(v, k) {
				return false
			}
			return true
		})
	}
}
```

### Zip

zipは、二つのiteratorを受けとり、両方を同時にiterate overするというiteratorです。

今回述べられるproposalはいわゆるPush型のiteratorであり、明示的にPullするタイミングを制御できないため、１個要素取得したらコントロールを呼び出し側に戻す、というようなことはできません。
そのため二つのiteratorを一緒に進めるというのは不可能であり、pushをいかにしてかpullに変換する必要があります。

そこで、受け取った二つのiteratorをgoroutineの中で消費し、1つ要素を受けとるたびにチャネルを通して元の実行コンテキストに送信します。channelをunbufferedにしておけば、Pull型への変換が可能となるわけです。

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

### Zip(iter.Pull版)

https://github.com/golang/go/issues/61897#issuecomment-1818871435

新しいCLで`iter` packageが使用できるようになったため`iter.Pull`を利用したZipを実装します。

`iter.Pull`および`iter.Pull2`は新しいcoroutineの中でiteratorを実行することでPush型iteratorをPull型に変換すると述べています。

```
go install golang.org/dl/gotip@latest
gotip download 543319
GOEXPERIMENT=rangefunc gotip test ./...
```

```go
import (
	"iter"
)

func ZipPull[V1, V2 any](
	left func(yield func(v V1) bool),
	right func(yield func(v V2) bool),
) func(yield func(l V1, r V2) bool) {
	return func(yield func(l V1, r V2) bool) {
		nextL, stopL := iter.Pull(left)
		nextR, stopR := iter.Pull(right)
		defer stopL()
		defer stopR()

		for {
			l, lOk := nextL()
			r, rOk := nextR()

			if !lOk || !rOk {
				return
			}
			if !yield(l, r) {
				return
			}
		}
	}
}
```

:::details coroの実装

えっcoroutine？となると思います。私はなりました。
`Go`はcoroutineをサポートしていませんし、追加したともテキスト中に書かれていませんので急にしれっと出てきました。

[Wikipedia](https://en.wikipedia.org/wiki/Coroutine)によればcoroutineは実行を一時中断、再開させれるコンピュータプログラムコンポーネントです。

どういうことなのかわからなかったので[CL](https://go-review.googlesource.com/c/go/+/543319)を読んでみることにします。

[CL](https://go-review.googlesource.com/c/go/+/543319)によれば、coroutineは[/src/runtime/coro.go](https://go-review.googlesource.com/c/go/+/543319/1/src/runtime/coro.go)で定義されています。

やはりこのCLでcoroutineを実装しているということで間違いないようです。

> // A coro represents extra concurrency without extra parallelism,
> // as would be needed for a coroutine implementation.
> // The coro does not represent a specific coroutine, only the ability
> // to do coroutine-style control transfers.
> // It can be thought of as like a special channel that always has
> // a goroutine blocked on it. If another goroutine calls coroswitch(c),
> // the caller becomes the goroutine blocked in c, and the goroutine
> // formerly blocked in c starts running.
> // These switches continue until a call to coroexit(c),
> // which ends the use of the coro by releasing the blocked
> // goroutine in c and exiting the current goroutine.
> //
> // Coros are heap allocated and garbage collected, so that user code
> // can hold a pointer to a coro without causing potential dangling
> // pointer errors.

いきなりcoroutineじゃないって言ってますね。`coro`はcoroutineそのものではないが、既存のgoroutineの仕組みを利用して、上記Zip実装の中でやっていたようなことをするより効率的にcoroutine的なコントロールを実現するものと述べられていますね。

`coro`を定義することで、coroutineとして動作させるgoroutineと、coroutineの中で動作させる関数を記録します。

```go
type coro struct {
	gp guintptr
	f  func(*coro)
}
```

`g`(goroutineのことですね)を拡張して`coro`をトラックできるようにします。

```diff go
type g struct {
	// 中略
+	coroexit     bool // argument to coroswitch_m
	// 中略
+	coroarg *coro // argument during coroutine transfers
	// 中略
}
```

`getg()`という関数(実際にはコンパイラがTLSから`*g`を取得するようにリライトするらしい)があるので、現在の`*g`は割とどこからでも取得可能です。
実装の中で`func newproc1(fn *funcval, callergp *g, callerpc uintptr) *g`(新しいgoroutineを作成する)や`func mcall(fn func(*g))`(`*m`の`g0`のスタックで`fn`を実行する)を利用するため、自由に引数を渡せたりできません。そのため`*g`にcoroutine関連の情報をストアしておく必要があるようですね。

`c := newcoro(f func(*coro))`で、`f`を新しいgoroutineの中で実行、`f`は引数に渡される`*coro`を引数に`coroswitch(*coro)`を呼び出すことが期待されているようです。`newcoro`を呼び出した側のgoroutineで`coroswitch(c)`を呼ぶことで、`f`とその外側が交互に実行・ブロッキングが切り替えられます。

`corostart()`, `coroswitch()`によって`newproc1`で作られた新しいgoroutineのスケジュール状態を手動で切り替えてしまうことで、重いスケジューラ部分をスキップする、それよによってcoroutine-likeなコントロールを効率的に実現するということのようです。

`iter.Pull`はこの挙動を利用することで、goroutineからparallelismをなくしてconcurrencyだけ得ることでPush型iteratorを効率的にPull型に変換するということらしいですね。

:::

## 複数adapterの適用

```go
func multipleAdapter() {
	base1 := iteratorhelper.SliceIter([]string{"foo", "bar", "baz", "qux", "quux"})
	base2 := iteratorhelper.SliceIter([]string{"corge", "grault", "garply"})
	adapted := iteratorhelper.FilterExclude(
		iteratorhelper.Map(
			iteratorhelper.Chain(base1, base2),
			func(_ int, v string) (time.Time, string) {
				return time.Now(), v + v
			},
		),
		func(k time.Time, v string) bool {
			return k.UnixNano()%3 == 0
		},
	)

	for k, v := range adapted {
		fmt.Printf("k = %#v, v = %#v\n", k, v)
	}
}
/*
k = time.Date(2023, time.November, 30, 14, 9, 58, 979296385, time.Local), v = "foofoo"
k = time.Date(2023, time.November, 30, 14, 9, 58, 979335361, time.Local), v = "barbar"
k = time.Date(2023, time.November, 30, 14, 9, 58, 979336663, time.Local), v = "quxqux"
k = time.Date(2023, time.November, 30, 14, 9, 58, 979337755, time.Local), v = "corgecorge"
k = time.Date(2023, time.November, 30, 14, 9, 58, 979343827, time.Local), v = "graultgrault"
k = time.Date(2023, time.November, 30, 14, 9, 58, 979344979, time.Local), v = "garplygarply"
*/
```

このように入れ子にiteratorにadapterを適用していきます。

method chainによるadapterの適用はできません。
[#49085](https://github.com/golang/go/issues/49085)がないため、methodが新しいtype parameterを追加できないためです。このためMapのような型の変換を行うadapterはmethod chainで実装できません。
このproposal上のコメントでもある通り[type parameterのあるmethodがinterfaceを実装すべきなのかという問題](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#No-parameterized-methods)があるため、実装される可能性も低いです。

# 感想

- 3つシグネチャがあるので、少なくとも2つずつadapterを作る必要があるのでそこが少し手間に感じます。
- どうやってFilterのようなアダプタ関数を定義するか一瞬わかりませんでした。
- 書くのにずいぶん時間がかかりました。あとからちょこちょこいじっていますが、最初に大体のコードを書き上げるのに3，4時間ぐらいかかりました
  - 構文追加によってVS CodeのGo extensionなどエディター上での支援が得られなくなるためです。
    - 今思えばコンソールで`watch GOEXPERIMENT=range gotip vet ./...`を実行しておけばよかったです。
  - やっぱり言語サーバーの支援は偉大です。
- 前からiterator protocolをユーザーのコードに向けて公開してほしいと思っていたので、この変更はすごく歓迎です。
- Goにとってあまりない構文の追加でありますので、for range loopを見て何かしらのcode generationを行っているようなコードは影響を受けるかもしれませんね。
- いろいろ呼んでるとこの構文追加による利益は「シーケンスデータの生成および加工の標準的な方法が決まること」であって、シーケンスデータの加工さえしていれば`for-range`を使わないコードベースであっても同様に利益を得ますね。
- `iter`および`x/exp/xiter`で普段つかうようなものは実装されると思いますので、基本的にはそれらのアダプタを組み合わせるアダプタを書くことになるかな・・・未来は明るいですね。

どんどん使いやすくなって助かりますね。このまま取り込まれるのかはわかりませんが楽しみです。

# 参考

参考にさせていただいた記事です。

- https://zenn.dev/team_soda/articles/0b57bba3a3665f

[#61897]: https://github.com/golang/go/issues/61897
