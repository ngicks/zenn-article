---
title: "[Go]なるだけすべてをiteratorにする"
emoji: "😵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## なるだけ~~すべてを~~iteratorにする

すべては言い過ぎですね。

ソースはすべてここに上がります。

https://github.com/ngicks/go-iterator-helper

この記事ではなるだけiteratorとしてものを利用できるように考えてみたり実装してみたりします。
各種データコンテナやシーケンスデータを返すものをiteratorになるように包んだり、アダプタツールを整えてiteratorでいろんな処理ができるようにします。
なるだけiteratorにする都合上、記事上で言及しておいて「実際使うことは少ないでしょう」というようなコメントを添えているものもあります。

この記事では`Go 1.23.0`を対象バージョンとします。環境は`linux/amd64`ですが、特にOSやarchがかかわる話はしません。

また、この記事は`func(func() bool)`, `iter.Seq[V]`もしくは`func(func(V) bool)`, `iter.Seq2[K,V]`もしくは`func(func(K,V) bool)`のことをカジュアルにiteratorと呼びます。この慣習はstdのドキュメントなどでも同様です。

## EDIT NOTE

- 2024-09-14:
  - `iter.Seq2[V, error]` -> `iter.Seq[V]`に変換することができる`Err`メソッドのあるstructを新規追加しました。
  - `Window`の仕様をRustのそれに合わせて、`len(input slice) < window-size`の時何もyieldしないようにしました。
  - `*(json|xml).Decoder`のラッパーが正しくなかった(多分)のと共通化できるのに気づいたので修正。
- 2024-09-15:
  - MergeSortのベンチをとったので追記。
- 2024-09-18:
  - MergeSortのベンチをいろいろ改良。筆者自身が結果を誤解していたので少し文言を変更。デカいstructかつ結果を1要素ずつ消費するときのみ`iter.Seq[V]`やや有利という感じの文言に。
  - collectionのところに`Compact`を追加。これは`deno`のstdと全く関係がない。
  - `SumOf`のところに`Sum`も追加
- 2024-09-19(final):
  - まずそうな記述に気付いたとき以外はこれ以上編集しないことにする。
  - for-range-funcはrewriteであることを追記
  - 全般的にdoc commentを修正したのを反映
  - `xiter.Zipped[T, error]`を使えばエラーをchannel経由で伝搬できると書いたが、`KeyValue[T, error]`のほうがシンプルなのでそう変更。
  - `Chan`のctxがnilでもよいという仕様をなくした。なぜnilでもいいという風に書いた・・・？
  - `Chan`がctx cancellationを優先して確認するように変更。
  - `ChanSend`のexampleを追記。
  - `*Box`(`iter.Seq2[V, error]` -> `iter.Seq[V]`に変換することができる`Err`メソッドのあるstruct)はstatefulなiteratorを返すのでIntoIterを実装すべきだった。
  - `*Box`で`*sql.Rows`をスキャンするexampleを追記。このAPIスタイル好きかも。
  - `Compact`をadapterの項目以下に移動。
  - `SumOf`, `ReduceGroup`, `RunningReduce`がseq-lastじゃなかったので修正
- 2024-09-24:
  - `*errbox.Box`がドキュメントに反してresumableでなかったはさすがにまずいので修正

## iterator

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

`func(func() bool)`以外の二つは、`iter`パッケージで[iter.Seq\[V\]](https://pkg.go.dev/iter@go1.23.0#Seq), [iter.Seq2\[K, V\]](https://pkg.go.dev/iter@go1.23.0#Seq2)という型として定義されるので、これを用いると少しわかりやすくなります。

```go
func someIter[K, V any]() iter.Seq2[K, V] {
    return func(yield func(K, V) bool) {
        // ...
    }
}
```

もちろん`iter.Seq[V]`, `iter.Seq2[K,V]`を用いなくてもrange-over-funcは機能するので、`Go1.23.0`以前のversionで作られたライブラリにiteratorを返すメソッドを追加したい場合は、直接`func(func(K, V) bool)`を返すとよいと思います。

## stdですでに実装されているもの

現時点でiteratorを引数に取ったり、返り値として返す関数はstd内にあまりありません。

特記すべきこととして`slices.Chunk`でsliceを引数に、範囲の被らないn個要素のsub sliceを生成するiteratorを返します。

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

## stdだけど未リリースのもの

- [proposal: regexp: add iterator forms of matching methods(#61902)](https://github.com/golang/go/issues/61902)
- [bytes, strings: add iterator forms of existing functions (#61901)](https://github.com/golang/go/issues/61901)
  - `Go1.24`でリリースされます

## x/exp/xiter

[proposal: x/exp/xiter: new package with iterator adapters(#61898)](https://github.com/golang/go/issues/61898)で`x/exp/xiter`が提案されています。

以下のようなadapter群が提案されています。(`2024-09-08`現在)

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

`[]V`や`map[K]V`を引数に受けていると、別のデータ構造を使用したい場合変換しなければならないことや、複数の`[]V`,`map[K]V`を用いたい場合に結合する処理が必要でした。
結合の際に`len(s1)+len(s2)`の長さを持つsliceなどをallocateする必要があるはずなので、そこでメモリとコピーの負荷があったはずです。

- `heap.Interface`だろうが`*list.List`だろうが`*ring.Ring`だろうが、iteratorにする関数を定義すれば同じ`iter.Seq[V]`というシグネチャで受け取れます。
- `xiter`の実装で`Concat`, `Concat2`が提案されているので`[]V`や`map[K]V`を結合する処理は書かなくてよくなります。
  - 新しいsliceのallocationはなくなるので、入力のsliceのサイズが大きければメモリ負荷的に有利になっていくはずです。

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

iterator生成元となる型にすでにiteratorを返すメソッドが定義されていてなおかつ上記と異なるのは普通にありうると思います。
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

## 実装の注意点

iteratorやadapterなどを実装するときの注意点を述べます。

### range-over-funcはbreakしたら二度と呼ばない

`for range seq`は１度しか呼び出さないようにします。

例えば、`v`をn回yieldするiteratorを実装するとします。以下二通りの実装ができます。

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

これは単に`seq1`はステートをクロージャーの中に持たないのに対し、`seq2`は(`n`を引き算してしまうことで)ステートのあるクロージャーを返してしまっているからです。
これは例のために露骨にしてありますが、実際上`heap.Interface`や`*bufio.Scanner`をiteratorにラップしたものは内部でステートを変更してしまうので、`seq2`のように**iteratorがstatefulなこともある**ということです。

これは`iter.Pull`でラップすることで同じように扱うことができます。

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

ただ`iter.Pull`は新しい`goroutine`を取得してしまうため、stopが呼ばれるか、seqを最後までiterateするかをしないと`goroutine leak`となってしまいます。
`iter.Pull`を不用意に呼ぶとうっかり`goroutine leak`をしてしまうかもしれないのでやらんでいいならやらないほうがいいですね。

### 基本はstatelessにする

上記の議論を踏まえて、statelessにできるiteratorはすべてstatelessにしたほうがいいかなあと思います。
statefulなiteratorのほうが少ないと思われるので、ドキュメントにわざわざ書くならstatefulのほうです。デフォルトではstatelessにしておくことでドキュメントでは`All iterators (returned iter.Seq[V] and iter.Seq2[K, V]) are stateless otherwise noted`とREADMEに一度載せるだけで済むので、そこがいいと思います。

### リソースの確保はiter.Seq内で行う

```go
func puller[V any](seq iter.Seq[V], n int) iter.Seq[V] {
    // ここではなく
    /*
        next, stop := iter.Pull(seq)
    */
    return func(yield func(V) bool) {
        // ここでする
        next, stop := iter.Pull(seq)
        defer stop()
        n := n // statelessにするためにnをshadowingしておく
        n--
        // ...
    }
}
```

前述のとおり、`iter.Pull`の呼び出し方を間違うと`goroutine leak`となります。
iterator外でリソースの確保を行ってしまうと、iteratorが呼びされなかった場合にクリーンナップ処理が実行されないため`leak`となります。

これは当然と言えば当然なんですがうっかりしちゃいそうです。

## K-V pair

[maps.Collect](https://pkg.go.dev/maps@go1.23.0#Collect)で`iter.Seq2[K, V]`を`map[K]V`に受け取れますが、
`map[K]V`にしてしまうとiteratorが生成した順序が保存できなかったりして困るときがある・・・ないかも、ありそう。

またデバッグ・テストコードのために順序付きで`iter.Seq2[K, V]`を生成したいため以下のような型を定義すると便利です。

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

## Repeat

特定の要素を繰り返したいというケースはありますよね。以下のように`iter.Seq`に適合するようにしておきます。

```go
// Repeat returns an iterator that generates v n times.
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
func RepeatFunc[V any](fnV func() V, n int) iter.Seq[V] {
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

あげてないですが`Repeat2`, `RepeatFunc2`も実装しています。

いちばん思いつくこれのユースケースは`math/rand/v2`の`N`を何度も呼べるようにすることです。

```go
rng := hiter.RepeatFunc(func() int { return rand.N(20) }, -1)
```

特定の値以下の乱数を何度も得たいときが本当にたくさんあって・・・ほとんどテストですが。

## 既存のデータシーケンスをiteratorにする

:::details #56413のに載ってるけど実装しなかったやつとその理由

https://github.com/golang/go/discussions/56413

- archive/tar.Reader.Next: Nextを呼ぶたび`tar.Reader`の中身が変わるステートフルなのがiteratorに合わないと感じた
  - ~~tarの`io.ReaderAt`を受けて`*io.SectionReader`を返すライブラリを実装してもいいなと考えていたので、そっち版にiteratorを実装しようかなという検討による。~~
    - `archive/tar`の実装とPAXとUSTarの仕様をチラチラ見てたんですが心おれました。tarをディレクトリとしてfsにマウントできるものがあるのは知っているので、不可能ではないとわかっているんですが大変ではありそうです。
- bufio.Reader.ReadByte: 力尽き
- expvar.Do: 力尽き
- flag.Visit: 力尽き
- go/token.FileSet.Iterate: 力尽き
- path/filepath.Walk: 後述
- runtime.Frames.Next: 力尽き

:::

### Range: [n, m)

[range-over-intがGo1.22.0で実装された](https://tip.golang.org/doc/go1.22#language)ことで、以降の`Go`では`for i := range n {}`が`for i := 0; i < n ; i++ {}`のショートハンドとして機能します。

が、一方で任意な`[n, m)`な範囲をiterateする方法(`for i := n; i < m; i++ {}`のショートハンド)は特に定義されていないので、以下の`Range`でそれを実現します。

```go
type Numeric interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
        ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
        ~float32 | ~float64
}

// Range produces an iterator that yields sequential Numeric values in range [start, end).
// Values start from `start` and step toward `end`.
// At each step value is increased by 1 if start < end, otherwise decreased by 1.
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

`start > end`する都合上type constraintにcomplexを入れられません。

### Window(moving window)

`slices.Chunk`がある一方でwindowはないので以下のように実装します。

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
            if !yield(s[start:end]) {
                return
            }
            start++
            end++
        }
    }
}
```

### Chan

channel `<-chan V`をiteratorに変換できると他のアダプタをそのまま使えて良いので以下のように定義します。

あんまり使う機会はないかもですね。

- channelはsynchronizationを目的として使うことが多いですから、そういった目的ではiteratorにできません。
- channelからデータを受けとって加工して別のchannelに送信する目的では一旦`iter.Seq`を介すといろんな処理が共通化ができていいかも。
  - `iter.Seq[KeyValue[T, error]]`を利用すればタスクのエラーも容易につたえることができますしね。

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
```

第二返り値がboolなとき、望ましい状態の時にtrueになる慣習(`ok semantics`と個人的に呼んでいる)があるので全部送信できたらtrueが返るようにしてあります。

EDIT 2024-09-19: 前述した「channelからデータを受けとって加工して別のchannelに送信する」パターンでの活用例としては以下のような形になるでしょうか。
この処理では`iter.Seq[KeyValue[T, error]]`を使っていない(エラーしないので)ですし、加工の処理は単に`"✨"`を付け足すだけなので微妙に見えるかもしれないですが、実際には別の`goroutine`に分割していくつかconcurrentに処理したくなるレベルの重い/io boundなタスクだと思ってほしいです。

```go
func ExampleChanSend() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    var (
        in, out = make(chan string), make(chan string)
        wg      sync.WaitGroup
    )

    wg.Add(1)
    go func() {
        defer wg.Done()
        hiter.ChanSend(ctx, in, hiter.Repeat("hey", 3))
    }()

    for range 3 {
        wg.Add(1)
        go func() {
            defer wg.Done()
            // super duper heavy tasks
            _, _ = hiter.ChanSend(
                ctx,
                out,
                hiter.Tap(
                    func(_ string) {
                        // sleep for random duration.
                        // Ensuring moderate distribution among workers(goroutines.)
                        time.Sleep(rand.N[time.Duration](100))
                    },
                    xiter.Map(
                        func(s string) string { return "✨" + s + "✨" },
                        hiter.Chan(ctx, in),
                    ),
                ),
            )
        }()
    }

    for i, decorated := range hiter.Enumerate(hiter.Chan(ctx, out)) {
        fmt.Printf("%s\n", decorated)
        if i == 2 {
            cancel()
        }
    }
    wg.Wait()
    // Output:
    // ✨hey✨
    // ✨hey✨
    // ✨hey✨
}
```

相当`iter.Seq`周りのツールがそろって来ない限りこういう書き方をしている自分を想像できませんが、できるはできますね。
処理が分割しやすくなるのでいろいろ取り扱いやすくなるのかもしれません。

### string

stringを処理するiteratorを実装します。

stringの中身はutf-8 encodingの`[]byte`なので、stringを`[]byte`として既存のiterator-adapterで処理すること自体はできます。
一方で、`strings`パッケージも存在する通り、stringの操作はプログラムを作るときにおいてとりわけ特別扱いされます。
これを踏まえて、stringを特別扱いするiteratorがあったほうがいいと判断しています。

[bytes, strings: add iterator forms of existing functions (#61901)](https://github.com/golang/go/issues/61901)がリリースされればstdでもstringsパッケージ以下にiteratorを返す関数群が実装されますが、それまでのつなぎや遊び用に作っている感じです。

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

`*bufio.Scanner`に当たるようなsplitterがあったほうが便利だと思って以下のように実装しています。

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

splitterはとりあえず以下の３つを実装しておきました。

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

### \*bufio.Scanner

`*bufio.Scanner`は以下のように変換できます。

```go
// Scanner wraps scanner with an iterator over scanned text.
// The caller should check [bufio.Scanner.Err] after the returned iterator stops
// to see if it has been stopped for an error.
func Scan(scanner *bufio.Scanner) iter.Seq[string] {
    return func(yield func(text string) bool) {
        for scanner.Scan() {
            if !yield(scanner.Text()) {
                return
            }
        }
    }
}
```

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

javascriptにおける([whatwgのstream spec](https://streams.spec.whatwg.org/#rs-class)の)[ReadableStreamもAsyncIterator](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream#async_iteration)だったりしますね。なのでとてつもなく珍しいスタイルのAPIということはないと思います。

ただ`Go`は`io.Reader`周りのツールがしっかりそろっているのでわざわざこういう形にしないかなと思います。

### json/xml Decoder

[\*json.Decoder](https://pkg.go.dev/encoding/json@go1.23.0#Decoder)および[\*xml.Decoder](https://pkg.go.dev/encoding/xml@go1.23.0#Decoder)も以下のようにするとiteratorに変換可能です。

ただし、あまりうれしさはありませんね。
decoderでトークンを読み進めてから`dec.Decode`を呼ぶことで大きなjson arrayを1要素ずつ読み進めるとか、そういったユースケースが普通にあります。
そのためdecoderへのアクセスが常に必要なことが多いからです。

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

~~`*bufio.Scanner`のように`Err`メソッドのあるstructを定義すれば`iter.Seq[*.Token]`とできます。そうしたほうがいいかも。
今回はこのぐらい素直で薄いラッパーにとどめておきます。~~

EDIT 2024-09-14: `Err`メソッドのあるstructを新規追加しました

### \*sql.Rows

`*sql.Rows`も以下のようにするとiteratorとして利用できます。

```go
// SqlRows returns an iterator over scanned rows from r.
// scanner will be called against [*sql.Rows] after each time [*sql.Rows.Next] returns true.
// It must either call [*sql.Rows.Scan] once per invocation or do nothing and return.
// If the scan result or [*sql.Rows.Err] returns a non-nil error,
// the iterator stops its iteration immediately after yielding the error.
func SqlRows[T any](r *sql.Rows, scanner func(*sql.Rows) (T, error)) iter.Seq2[T, error] {
    return func(yield func(T, error) bool) {
        for r.Next() {
            t, err := scanner(r)
            if !yield(t, err) {
                return
            }
            if err != nil {
                return
            }
        }
        if r.Err() != nil {
            yield(*new(T), r.Err())
            return
        }
    }
}
```

~~non-nil error = stopになるようなiteratorはなんとなくぎこちなさがありますね。~~
~~`encoding`で述べたのと同様に`Err`メソッドのあるstructを定義すれば`iter.Seq[T]`とできますが今回はこちらもやめておきました。~~

EDIT 2024-09-14: `Err`メソッドのあるstructを新規追加しました。

### iter.Seq2[V, error]をiter.Seq[V]にする

上記の`*(json|xml).Decoder`/`*sql.Rows`は`iter.Seq2[V, error]`となっており、non-nil errorが帰ってくるとその直後にiteratorが止まるというものでしたが、これはなんとなくぎこちなさを感じるものでした。
ということでラッパーとなるstructを定義して`iter.Seq2[V, error]`を`iter.Seq[V]`に変換しましょう。

まず`iter.Pull2`を使って`iter.Seq[V, error]`をresumableに変換します。

```go
// Resumable2 converts the input [iter.Seq2][K, V] into stateful form by calling [iter.Pull2].
//
// The zero value of Resumable2 is not valid. Allocate one by [NewResumable2].
type Resumable2[K, V any] struct {
    next func() (K, V, bool)
    stop func()
}

// NewResumable2 wraps seq into stateful form so that the iterator can be break-ed and resumed.
// The caller must call [*Resumable2.Stop] to release resources regardless of usage.
func NewResumable2[K, V any](seq iter.Seq2[K, V]) *Resumable2[K, V] {
    next, stop := iter.Pull2(seq)
    return &Resumable2[K, V]{
        next: next,
        stop: stop,
    }
}

// Stop releases resources allocated by [NewResumable2].
func (r *Resumable2[K, V]) Stop() {
    r.stop()
}

// IntoIter2 returns an iterator over the input iterator.
// The iterator can be paused by break and later resumed without replaying data.
func (r *Resumable2[K, V]) IntoIter2() iter.Seq2[K, V] {
    return func(yield func(K, V) bool) {
        for {
            k, v, ok := r.next()
            if !ok {
                return
            }
            if !yield(k, v) {
                return
            }
        }
    }
}
```

これをさらに包み、`iter.Seq[V, error]`がnon-nil errorを返したときにこれを記録し、iterationが止まるようにします。
記録されたerrorは`Err`メソッドで確認可能にします。これは`*bufio.Scanner`と同じ様式です。

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

前述の`*(json|xml).Decoder`/`*sql.Rows`を向けのラッパーはあらかじめ定義しておきましょう。

```go
type JsonDecoder struct {
    *Box[json.Token]
    Dec *json.Decoder
}

func NewJsonDecoder(dec *json.Decoder) *JsonDecoder {
    return &JsonDecoder{
        Box: New(hiter.JsonDecoder(dec)),
        Dec: dec,
    }
}

type XmlDecoder struct {
    *Box[xml.Token]
    Dec *xml.Decoder
}

func NewXmlDecoder(dec *xml.Decoder) *XmlDecoder {
    return &XmlDecoder{
        Box: New(hiter.XmlDecoder(dec)),
        Dec: dec,
    }
}

type SqlRows[V any] struct {
    *Box[V]
}

func NewSqlRows[V any](rows *sql.Rows, scanner func(*sql.Rows) (V, error)) *SqlRows[V] {
    return &SqlRows[V]{
        Box: New(hiter.SqlRows(rows, scanner)),
    }
}
```

EDIT 2024-09-19: 使ってみると以下のような感じ。

```go
func ExampleNewSqlRows_successful() {
    type TestRow struct {
        Id    int
        Title string
        Body  string
    }

    mock := testhelper.OpenMockDB(false)
    defer mock.Close()

    rows, err := mock.Query("SELECT id, title, body FROM posts")
    if err != nil {
        panic(err)
    }

    scanner := func(r *sql.Rows) (TestRow, error) {
        var t TestRow
        err := r.Scan(&t.Id, &t.Title, &t.Body)
        return t, err
    }

    boxed := errbox.NewSqlRows(rows, scanner)
    for row := range boxed.IntoIter() {
        fmt.Printf("row = %#v\n", row)
    }
    fmt.Printf("stored err: %v\n", boxed.Err())
    // Output:
    // row = errbox_test.TestRow{Id:1, Title:"post 1", Body:"hello"}
    // row = errbox_test.TestRow{Id:2, Title:"post 2", Body:"world"}
    // row = errbox_test.TestRow{Id:3, Title:"post 3", Body:"iter"}
    // stored err: <nil>
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

EDIT 2024-09-18: これを書いているときに筆者自身が😵になっていたので気が付きませんでしたが`(*sync.Map).Range`はすでに`iter.Seq2[any, any]`のシグネチャを満たしていますね。この関数を使うとkeyとvalueに`K`, `V`のtype paramを与えられるでそこだけ違います。

### Third party: github.com/wk8/go-ordered-map/v2

insertion-ordered mapの実装に筆者は[github.com/wk8/go-ordered-map/v2](httos://github.com/wk8/go-ordered-map)を使ったことがあります。
`map[K]*V`+`*list.List`の組み合わせです。

https://github.com/wk8/go-ordered-map/pull/41

まだリリースされていませんが、`iter.Seq`, `iter.Seq2`を返すAPIが追加されました。

### Third party: github.com/gammazero/deque

`[]T`ベースのdeque実装に筆者は[github.com/gammazero/deque](https://github.com/gammazero/deque)を用いたことがあります。

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

実装を見ると明言されているのでわかりやすいですが、ordered mapと言いつつ内部のデータ構造は[treap](https://en.wikipedia.org/wiki/Treap)ですので、`K`によるアクセスは`O(1)`ではなく`O(log n)`となります。かわりに特定のkey値範囲(e.g. `"aaa"`以上`"ccc"`以下のような)の探索などを行えます。

データ構造は色々覚えておくと便利ですねえ。

## iteratorにできないやつ

逆に、シーケンスではあるが、iteratorにしがたいものがあります。

具体的には以下二つが見つかりました。

- [fs.WalkDir](https://pkg.go.dev/io/fs@go1.23.0#WalkDir)
  - [filepath.WalkDir](https://pkg.go.dev/path/filepath@go1.23.0#WalkDir)も同様
- [io.Pipe](https://pkg.go.dev/io@go1.23.0#Pipe)

どちらも何かの方法で、データを受けた側が、生成する側に、データのスキップなどを指示する方式をとっています。現状のiteratorの仕組みはこの逆向きのシグナルの伝搬を定義していないため、別口の仕組みを実装する必要があります。これは直感的にiteratorにフィットしないのでiteratorにする意味は薄いだろうということです。

[fs.WalkDir](https://pkg.go.dev/io/fs@go1.23.0#WalkDir)は`fs.FS`とコールバック関数を引数に取り、`fs.FS`を深さ優先でwalkしながら見つかったパスごとにコールバック関数を実行します。コールバック関数が[fs.SkipDir](https://pkg.go.dev/io/fs@go1.23.0#SkipDir)を返すとディレクトリのwalkがスキップされます。[fs.SkipAll](https://pkg.go.dev/io/fs@go1.23.0#SkipAll)を返すと探索をやめることができます。

[io.Pipe](https://pkg.go.dev/io@go1.23.0#Pipe)はin-memory pipeを作成して両端のreaderとwriterを返し、writerに書き込まれた内容をreaderから読むことができます。reader/writerどちらも[CloseWithError](https://pkg.go.dev/io@go1.23.0#PipeReader.CloseWithError)を備え、エラーを片方からもう片方に伝搬できます。

例えば以下のようにすることで、`iter.Pull`を`io.Pipe+goroutine`の代わりに使うことができるのですが、
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

`io.Pipe`を丸っと模擬している例なので話のつながりが甘い感じになってしまっている気がしますが、
readerを読む側からエラーを逆方向に伝える仕組みがありませんから同等にならないことを表現しています。

## collection操作のヘルパをいくつか定義する

README.mdで述べていますが

> The idea is stolen from https://jsr.io/@std/collections/doc.

`deno`の`jsr:@std/collection`からいくつかアイデアを盗んで`iter.Seq`がかかわって便利そうなものを実装します。

### Permutations

`[]int{1, 2, 3}`に対して、`[][]int{{1,2,3}, {1,3,2}, {2,1,3}, {2,3,1}, {3,1,2}, {3,2,1}}`をPermutations(置換)と言います。
テスト目的に使われることがたびたびあるのを目にします。

```go
// Permutations returns an iterator that yields permutations of in.
// The returned iterator reorders in in-place.
// Callers should not retain in or slices from the iterator,
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

in-placeで並べかえをするので、`slices.Collect`とともに用いる場合は、明示的なクローンが必要です。

```go
slices.Collect(
    xiter.Map(
        slices.Clone,
        Permutations([]int{1, 2, 3, 4, 5}),
    ),
)
```

`slices.Clone`を呼ぶだけのマッパーは高頻度で使いそうなので関数として定義しておくと便利かもしれませんね。

### ReduceGroup

`iter.Seq2[K, V]`を`maps.Collect`のように`map[K]V`に格納しますが、格納前にすでに格納された値をとって`reducer`を実行します。
`maps.Collect`と違って`iter.Seq2`が同値の`K`を返す時に単に上書きしたくないときに有効です。

```go
func ReduceGroup[K comparable, V, Sum any](reducer func(accumulator Sum, current V) Sum, initial Sum, seq iter.Seq2[K, V]) map[K]Sum {
    m := make(map[K]Sum)
    for k, v := range seq {
        if _, ok := m[k]; !ok {
            m[k] = initial
        }
        m[k] = reducer(m[k], v)
    }
    return m
}
```

### RunningReduce

`Reduce`だが、`reducer`実行のたびに中間の結果をyieldできるというもの。何かで使い道がありそう。

```go
func RunningReduce[V, Sum any](reducer func(accumulator Sum, current V, i int) Sum, initial Sum, seq iter.Seq[V]) iter.Seq[Sum] {
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

### Sum, SumOf

`Sum`と`SumOf`。
`SumOf`は`iter.Seq[T]`の各要素をselectorで何かしらの`sum`可能な型に変換するバージョンの`Sum`です。

```go
// as per https://go.dev/ref/spec#arithmetic_operators
type Summable interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
        ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
        ~float32 | ~float64 |
        ~complex64 | ~complex128 |
        ~string
}

func Sum[S Summable](seq iter.Seq[S]) S {
    return reduce(
        seq,
        func(e S, t S) S { return e + t },
        *new(S),
    )
}

func SumOf[V any, S Summable](selector func(ele V) S, seq iter.Seq[V]) S {
    return reduce(
        seq,
        func(e S, t V) S { return e + selector(t) },
        *new(S),
    )
}
```

~~まず`Sum`を実装したほうがいいかもですね。~~

EDIT 2024-09-18: `Sum`も実装しました。

## adapterでiteratorを加工する。

とりあえずいりそうなadapterは実装しておきます。

[以前の記事: Goの1.22にGOEXPERIMENTガード下で導入されるrange over func proposalを試してみる](https://zenn.dev/ngicks/articles/go-trying-out-iter-proposal)では、

- Chain
- Chunk
- Enumerate
- Filter
- Map
- Skip / Take
- Window
- Zip

を、筆者がよく使うものとして実装しました。

このうち、stdで`Chunk`、`xiter`(のproposal)で`Concat`(=`Chain`),`Filter`, `Limit`(=`Take`), `Map`, `Skip`, `Zip`は実装されているので下記を実装しました。

```go
func Alternate[V any](seqs ...iter.Seq[V]) iter.Seq[V]
func Alternate2[K, V any](seqs ...iter.Seq2[K, V]) iter.Seq2[K, V]
func Compact[V comparable](seq iter.Seq[V]) iter.Seq[V]
func Compact2[K, V comparable](seq iter.Seq2[K, V]) iter.Seq2[K, V]
func CompactFunc[V any](eq func(i, j V) bool, seq iter.Seq[V]) iter.Seq[V]
func CompactFunc2[K, V any](eq func(k1 K, v1 V, k2 K, v2 V) bool, seq iter.Seq2[K, V]) iter.Seq2[K, V]
func Decorate[V any](prepend, append Iterable[V], seq iter.Seq[V]) iter.Seq[V]
func Decorate2[K, V any](prepend, append Iterable2[K, V], seq iter.Seq2[K, V]) iter.Seq2[K, V]
func Enumerate[T any](seq iter.Seq[T]) iter.Seq2[int, T]
func Flatten[S ~[]E, E any](seq iter.Seq[S]) iter.Seq[E]
func FlattenF[S1 ~[]E1, E1 any, E2 any](seq iter.Seq2[S1, E2]) iter.Seq2[E1, E2]
func FlattenL[S2 ~[]E2, E1 any, E2 any](seq iter.Seq2[E1, S2]) iter.Seq2[E1, E2]
func LimitUntil[V any](f func(V) bool, seq iter.Seq[V]) iter.Seq[V]
func LimitUntil2[K, V any](f func(K, V) bool, seq iter.Seq2[K, V]) iter.Seq2[K, V]
func OmitF[K, V any](seq iter.Seq2[K, V]) iter.Seq[V]
func OmitL[K, V any](seq iter.Seq2[K, V]) iter.Seq[K]
func Pairs[K, V any](seq1 iter.Seq[K], seq2 iter.Seq[V]) iter.Seq2[K, V]
func Skip[V any](n int, seq iter.Seq[V]) iter.Seq[V]
func Skip2[K, V any](n int, seq iter.Seq2[K, V]) iter.Seq2[K, V]
func SkipLast[V any](n int, seq iter.Seq[V]) iter.Seq[V]
func SkipLast2[K, V any](n int, seq iter.Seq2[K, V]) iter.Seq2[K, V]
func SkipWhile[V any](f func(V) bool, seq iter.Seq[V]) iter.Seq[V]
func SkipWhile2[K, V any](f func(K, V) bool, seq iter.Seq2[K, V]) iter.Seq2[K, V]
func Tap[V any](tap func(V), seq iter.Seq[V]) iter.Seq[V]
func Tap2[K, V any](tap func(K, V), seq iter.Seq2[K, V]) iter.Seq2[K, V]
func Transpose[K, V any](seq iter.Seq2[K, V]) iter.Seq2[V, K]
```

`xiter`が、`seq iter.Seq[V]`を引数に受けるときに末尾で受けるという`callback-first style`ないしは`seq-last style`になっているのでそれに追従しています。組み合わせて使っても違和感がないはずです。

以下でいくつか使ってみます。

### Compact(ADDED 2024-09-18)

`slices.Compact`のiterator版です

```go
// Compact skips consecutive runs of equal elements from seq.
// The returned iterator is pure and stateless as long as seq is so.
func Compact[V comparable](seq iter.Seq[V]) iter.Seq[V] {
    return func(yield func(V) bool) {
        var (
            first bool = true
            prev  V
        )
        for t := range seq {
            if first {
                first = false
                if !yield(t) {
                    return
                }
            } else if prev != t {
                if !yield(t) {
                    return
                }
            }
            prev = t
        }
    }
}
```

`xiter`にMergeがあるのでCompactもあると便利だなと思って実装しました。下記が典型的な使用例です。

```go
func Example_compact() {
    m := xiter.Merge(
        xiter.Map(func(i int) int { return 2 * i }, hiter.Range(1, 11)),
        xiter.Map(func(i int) int { return 1 << i }, hiter.Range(1, 11)),
    )

    first := true
    for i := range Compact(m) {
        if !first {
            fmt.Printf(", ")
        }
        fmt.Printf("%d", i)
        first = false
    }
    fmt.Println()
    // Output:
    // 2, 4, 6, 8, 10, 12, 14, 16, 18, 20, 32, 64, 128, 256, 512, 1024
}
```

ベンチをとるときに要素数ごとにベンチをとりたい、32以下とかまでの数字は2の倍数全部をテストして、以降は`2^n`だけをテストしたいとかそういうときに使いやすいです。
なくても困ってなかったんですがあると便利ですね。なんとなく手続きの塊から論理の塊になるみたいで面白いです。

### Merge sort

真鯵ソート。

iteratorなしで普通に実装するならこう書きますが、

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

これをiteratorありにすると・・・

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
マージのたびに新しいsliceをallocateしていたのがなくなるので、要素数が増えていくにつれてお得になるかもしれません。ベンチとってみてください！

iteratorにするついでに引数が`[]T`なのをやめてもっとジェネリックに

```go
type SliceLike[T any] interface {
    At(i int) T
    Len() int
}

func mergeSortIterFunc[M SliceLike[T], T any](m M, cmp func(l, r T) int) iter.Seq[T] {
    // ...
}
```

とすると、`*(github.com/gammazero/deque).Deque`や`*(github.com/rsc/omap).Map`を`[]T`に変換せずにmerge sortできます。

以下のような感じでしょうか。
exampleなので深く考えていないですがとりあえずテストは通っています。

```go
type SliceLike[T any] interface {
    hiter.Atter[T]
    Len() int
}

var _ SliceLike[any] = sliceAdapter[any]{}

type sliceAdapter[T any] []T

func (s sliceAdapter[T]) At(i int) T {
    return s[i]
}

func (s sliceAdapter[T]) Len() int {
    return len(s)
}

type subbable[S SliceLike[T], T any] struct {
    S    S
    i, j int
}

func (s subbable[S, T]) At(i int) T {
    i = s.i + i
    if i < s.i || i >= s.j {
        panic("index out of range")
    }
    return s.S.At(i)
}

func (s subbable[S, T]) Len() int {
    return s.j - s.i
}

func (s subbable[S, T]) Sub(i, j int) subbable[S, T] {
    i = i + s.i
    j = j + s.i
    if i < 0 || i < s.i ||j > s.j || i > j {
        panic(fmt.Errorf("index out of range: i=%d, j=%d,len=%d", i, j, s.Len()))
    }
    return subbable[S, T]{
        S: s.S,
        i: i,
        j: j,
    }
}

func mergeSortAtterFunc[S SliceLike[T], T any](s S, cmp func(l, r T) int) iter.Seq[T] {
    sub := subbable[S, T]{
        S: s,
        i: 0,
        j: s.Len(),
    }
    return mergeSortSubbableFunc(sub, cmp)
}

func mergeSortSubbableFunc[S SliceLike[T], T any](s subbable[S, T], cmp func(l, r T) int) iter.Seq[T] {
    if s.Len() <= 1 {
        return hiter.OmitF(hiter.IndexAccessible(s, hiter.Range(0, s.Len())))
    }
    return func(yield func(T) bool) {
        left, right := s.Sub(0, s.Len()/2), s.Sub(s.Len()/2, s.Len())
        leftIter := mergeSortSubbableFunc(left, cmp)
        rightIter := mergeSortSubbableFunc(right, cmp)
        for t := range xiter.MergeFunc(leftIter, rightIter, cmp) {
            if !yield(t) {
                return
            }
        }
    }
}
```

~~要素数がいくつかまでは`[]T`を一度allocateしたほうが多分処理が速いです。今までの経験からくる勘だと大体、32～64個の間のどこかあたりに「これ以下なら`[]T`へ変換したほうがよい」という分水嶺がありそう。~~

ベンチとってみました

- `SliceLike[T]`を引数に取る版は`[]T`を引数に取る版に比べると`ns/op`も`B/op`、`alloc/op`すべて劣るため、どうも一旦`[]T`にしたほうがよいようです。
- `iter.Seq[V]`版は`T`が大きなstructで結果を`[]T`に受けずに1要素ずつ消費する場合に限って`B/op`が大分落ちているので最も大きな利点はそこでしょうか
  - `alloc/op`はすごく増えているのでメモリフラグメンテーションが起きるのでパフォーマンスが落ちるでしょうか？その辺の感覚がなくてよくわかりません。

多分公平なベンチになっている。もっと凝った実装にしないかぎり`[]T`をどうこうしたほうがいいですね。

:::details ベンチの内容

コード:

```go
func Benchmark_deque_merge_sort_slice_conversion(b *testing.B) {
    rng := hiter.RepeatFunc(func() int { return mathRand.N[int](1000) }, -1)
    randNumArray := slices.Collect(xiter.Limit(rng, 2048))

    b.Run("[]int", func(b *testing.B) {
        b.ResetTimer()
        deque_merge_sort(b, randNumArray[:], cmp.Compare)
    })

    type bigStruct struct {
        Key string
        Mah [2048]byte
    }
    bigStructs := make([]bigStruct, 2048)
    for i := range 2048 {
        bigStructs[i] = bigStruct{
            Key: hiter.StringsCollect(4*8*2, xiter.Limit(randStr(), 8)),
        }
    }

    b.Run("[]bigStruct", func(b *testing.B) {
        b.ResetTimer()
        deque_merge_sort(b, bigStructs, func(i, j bigStruct) int { return cmp.Compare(i.Key, j.Key) })
    })

    starBigStructs := make([]*bigStruct, 2048)
    for i := range 2048 {
        starBigStructs[i] = &bigStruct{
            Key: hiter.StringsCollect(4*8*2, xiter.Limit(randStr(), 8)),
        }
    }

    b.Run("[]*bigStruct", func(b *testing.B) {
        b.ResetTimer()
        deque_merge_sort(
            b,
            starBigStructs,
            func(i, j *bigStruct) int {
                if i == nil {
                    return -1
                }
                return cmp.Compare(i.Key, j.Key)
            },
        )
    })
}

func sizes() iter.Seq[int] {
    return xiter.Merge(
        xiter.Map(func(i int) int { return i * 5 }, hiter.Range(1, 5)),
        xiter.Map(func(i int) int { return 1 << i }, hiter.Range(1, 12)),
    )
}

func randStr() iter.Seq[string] {
    return hiter.RepeatFunc(func() string {
        var buf [4]byte
        _, err := rand.Read(buf[:])
        if err != nil {
            panic(err)
        }
        return hex.EncodeToString(buf[:])
    }, -1)
}

func deque_merge_sort[T any](b *testing.B, input []T, cmp func(l T, r T) int) {
    for i := range sizes() {
        d := deque.New[T]()
        for ele := range xiter.Limit(slices.Values(input), i) {
            d.PushBack(ele)
        }
        b.Run(fmt.Sprintf("%02d", i), func(b *testing.B) {
            b.Run("slice_version", func(b *testing.B) {
                b.ResetTimer()
                for range b.N {
                    ok := slices.IsSortedFunc(mergeSortFunc(input[:i], cmp), cmp)
                    if !ok {
                        panic("eh")
                    }
                }
            })
            b.Run("converted_to_slice", func(b *testing.B) {
                b.ResetTimer()
                s := make([]T, d.Len())
                for i := range d.Len() {
                    s[i] = d.At(i)
                }
                for range b.N {
                    ok := slices.IsSortedFunc(slices.Collect(collection.MergeSortFunc(s, cmp)), cmp)
                    if !ok {
                        panic("eh")
                    }
                }
            })
            b.Run("converted_to_slice_no_collect", func(b *testing.B) {
                b.ResetTimer()
                s := make([]T, d.Len())
                for i := range d.Len() {
                    s[i] = d.At(i)
                }
                for range b.N {
                    var prev T
                    for n := range collection.MergeSortFunc(s, cmp) {
                        if cmp(prev, n) > 0 {
                            panic("oh?")
                        }
                        prev = n
                    }
                }
            })
            b.Run("no_conversion", func(b *testing.B) {
                b.ResetTimer()
                for range b.N {
                    ok := slices.IsSortedFunc(slices.Collect(collection.MergeSortSliceLikeFunc(d, cmp)), cmp)
                    if !ok {
                        panic("eh")
                    }
                }
            })
        })
    }
}

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

結果:

長いので抜粋

```
Benchmark_deque_merge_sort_slice_conversion/[]int/2048/slice_version-24                             7818            142249 ns/op          180224 B/op       2047 allocs/op
Benchmark_deque_merge_sort_slice_conversion/[]int/2048/converted_to_slice-24                         331           3651633 ns/op         1337616 B/op      45052 allocs/op
Benchmark_deque_merge_sort_slice_conversion/[]int/2048/converted_to_slice_no_collect-24              331           3683872 ns/op         1277589 B/op      45038 allocs/op
Benchmark_deque_merge_sort_slice_conversion/[]int/2048/no_conversion-24                              325           3627108 ns/op         1452198 B/op      49148 allocs/op
Benchmark_deque_merge_sort_slice_conversion/[]bigStruct/2048/slice_version-24                        130           9431584 ns/op        50233485 B/op       2048 allocs/op
Benchmark_deque_merge_sort_slice_conversion/[]bigStruct/2048/converted_to_slice-24                    46          25529435 ns/op        27181490 B/op      45053 allocs/op
Benchmark_deque_merge_sort_slice_conversion/[]bigStruct/2048/converted_to_slice_no_collect-24         44          22871505 ns/op        10759264 B/op      45038 allocs/op
Benchmark_deque_merge_sort_slice_conversion/[]bigStruct/2048/no_conversion-24                         40          28564271 ns/op        27202875 B/op      49148 allocs/op
Benchmark_deque_merge_sort_slice_conversion/[]*bigStruct/2048/slice_version-24                      4479            254424 ns/op          192000 B/op       2047 allocs/op
Benchmark_deque_merge_sort_slice_conversion/[]*bigStruct/2048/converted_to_slice-24                  313           3657275 ns/op         1309389 B/op      45051 allocs/op
Benchmark_deque_merge_sort_slice_conversion/[]*bigStruct/2048/converted_to_slice_no_collect-24       337           3620493 ns/op         1261134 B/op      45038 allocs/op
Benchmark_deque_merge_sort_slice_conversion/[]*bigStruct/2048/no_conversion-24                       337           3588790 ns/op         1424020 B/op      49147 allocs/op
```

:::

### string decorate

`"foo bar baz"`のような文章を分割して`1. foo 2. bar 3. baz`という文章にdecorateします。
メール文書のテンプレートを書くときとかこういったユースケースがなくはないと思います。

以下のようになります。

```go
func ExampleDecorate() {
    src := "foo bar baz"
    var num atomic.Int32
    numListTitle := iterable.RepeatableFunc[string]{
        FnV: func() string { return fmt.Sprintf("%d. ", num.Add(1)) },
        N:   1,
    }
    m := hiter.StringsCollect(
        9+((2 /*num*/ +2 /*. */ +1 /* */)*3),
        hiter.SkipLast(
            1,
            hiter.Decorate(
                numListTitle,
                iterable.Repeatable[string]{V: " ", N: 1},
                hiter.StringsSplitFunc(src, -1, hiter.StringsCutWord),
            ),
        ),
    )
    fmt.Printf("%s\n", m)
    // Output:
    // 1. foo 2. bar 3. baz
}
```

`StringsCollect`、`StringsSplitFunc`はすでに述べました。

今回のような、`iter.Seq`の前後に固定の要素を付け足したい場合があったので以下のように`Decorate`を実装しました。
`Iterable[T]`はここでしか使っていないのでここ専用のinterfaceみたいになってます。

```go
// Decorate decorates seq by prepend and append,
// by yielding additional elements before and after seq yields.
// Both prepend and append are allowed to be nil; only non-nil [Iterable] is used as decoration.
func Decorate[V any](prepend, append Iterable[V], seq iter.Seq[V]) iter.Seq[V] {
    return func(yield func(V) bool) {
        for v := range seq {
            if prepend != nil {
                for vp := range prepend.Iter() {
                    if !yield(vp) {
                        return
                    }
                }
            }
            if !yield(v) {
                return
            }
            if append != nil {
                for va := range append.Iter() {
                    if !yield(va) {
                        return
                    }
                }
            }
        }
    }
}
```

`hiter/iterable`以下で、各種コンテナに`Iterable`, `IntoIterable`などを実装したラッパを定義しています。
`iterable.RepeatableFunc`もその一つです。

```go
// Repeatable generates an iterator that generates V N times.
type Repeatable[V any] struct {
    V V
    N int
}

func (r Repeatable[V]) Iter() iter.Seq[V] {
    return hiter.Repeat(r.V, r.N)
}

// RepeatableFunc generates an iterator that generates value returned from FnV N times.
type RepeatableFunc[V any] struct {
    FnV func() V
    N   int
}

func (r RepeatableFunc[V]) Iter() iter.Seq[V] {
    return hiter.RepeatFunc(r.FnV, r.N)
}
```

Decorateの作り的に末尾にも空白が入ってしまいますが、今回のユースケース的にはあってはいけません。
そのため、それを消すために最後のN要素を消すadapterを実装します。

```go
// SkipLast returns an iterator over seq that skips last n elements of seq.
func SkipLast[V any](n int, seq iter.Seq[V]) iter.Seq[V] {
    return func(yield func(V) bool) {
        var ( // easy implementation for ring buffer.
            buf    = make([]V, n)
            cursor int
            full   bool
        )
        for v := range seq {
            if !full {
                buf[cursor] = v
                cursor++
                if cursor == n {
                    cursor = 0
                    full = true
                }
                continue
            }
            vOld := buf[cursor]
            if !yield(vOld) {
                return
            }
            buf[cursor] = v
            cursor++
            if cursor == n {
                cursor = 0
            }
        }
    }
}
```

`iter.Seq`が関数であることから、以下のように`for v := range seq`を二つに分割できないことに注意してください。
分割すると`seq`の実行が頭からもう１度行われるのでseqがstatelessなら2度目の呼び出し時に先頭にn要素スキップを挟む必要があります。statelessでない場合は何が起きるかわかりません。単に前のiterationの続きから始まる場合はn要素スキップを挟んではいけないことになり、**呼び出し時に乱数を使っていたりする場合は不整合な状態になる**ことになります。

```go
// 以下はうまく動かない
    return func(yield func(V) bool) {
        var ( // easy implementation for ring buffer.
            buf    = make([]V, n)
            cursor int
        )
        for v := range seq {
            buf[cursor] = v
            cursor++
            if cursor == n {
                cursor = 0
                break
            }
        }
        for v := range seq {
            vOld := buf[cursor]
            if !yield(vOld) {
                return
            }
            buf[cursor] = v
            cursor++
            if cursor == n {
                cursor = 0
            }
        }
    }
```

### moving average

`Window`のユースケースに移動平均があるので、これができることをしめします。

```go
func ExampleWindow_moving_average() {
    src := []int{1, 0, 1, 0, 1, 0, 5, 3, 2, 3, 4, 6, 5, 3, 6, 7, 7, 8, 9, 5, 7, 7, 8}
    movingAverage := slices.Collect(
        xiter.Map(
            func(s []int) float64 {
                return float64(hiter.Sum(slices.Values(s))) / float64(len(s))
            },
            hiter.Window(src, 5),
        ),
    )
    fmt.Printf("%#v\n", movingAverage)
    // Output:
    // []float64{0.6, 0.4, 1.4, 1.8, 2.2, 2.6, 3.4, 3.6, 4, 4.2, 4.8, 5.4, 5.6, 6.2, 7.4, 7.2, 7.2, 7.2, 7.2}
}
```

### range map

`for i := initial; underLimit(i); i += step {}`ではたびたびoff-by-one errorが起きるといわれています。
これを防ぐために`Range`、`Map`、`LimitUntil`を組み合わせて任意な数値をiterateします。

以下で7の倍数を50より下の範囲でiterateする例を示します。

```go
func ExampleRange_prevent_off_by_one() {
    for i := range hiter.LimitUntil(
        func(i int) bool { return i < 50 },
        xiter.Map(
            func(i int) int { return i * 7 },
            hiter.Range(0, 10),
        ),
    ) {
        if i > 0 {
            fmt.Printf(" ")
        }
        fmt.Printf("%d", i)
    }
    // Output:
    // 0 7 14 21 28 35 42 49
}
```

`LimitUntil`は`xiter`にはないので実装する必要があります。

```go
// LimitUntil returns an iterator over seq that yields until f returns false.
func LimitUntil[V any](f func(V) bool, seq iter.Seq[V]) iter.Seq[V] {
    return func(yield func(V) bool) {
        for v := range seq {
            if !f(v) || !yield(v) {
                return
            }
        }
    }
}
```

## おわりに

- 現状はiteratorまわりにデファクトとなるツールがそろっていないので筆者はまだほとんど全く使っていません。
  - `xiter`もまだマージされていませんし
- データコンテナ/シーケンス系ライブラリをもメンテしている場合は`iter.Seq`を返すメソッドを追加したほうが使用者には便利になります
- 1つ1つのアダプタはなかなかtrivialなので、このように自前で作ったライブラリをメンテし続けてもいいかもしれないです。
- 無理になんでもiteratorにすると逆に読みにくくなりそうですが、`filter`や`map`みたいなものは(筆者は)何度もforで実装していましたし、筆者は結構実装ミスをしていたりしたので`xiter`で実装されるものぐらいは活用したほうが無難でしょうね。
