---
title: "[gotip]generic methodsでメソッドチェーンでiteratorを処理する"
emoji: "🧪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

# \[gotip\]generic methodsでメソッドチェーンでiteratorを処理する

https://github.com/golang/go/issues/77273

iteratorをmethod chainで処理できようなヘルパーを作ってみることで上記のproposalで述べられているgeneric methodsを試してみます。

## 対象読者

- [The Go Programming Language](https://go.dev/)にある程度習熟している
- generic methodsに興味があるが自分で試すほどじゃない

## Proposalの概要

methodがtype paramを持てるようになります

つまり以下ができるようになります。

```go
type Option[V any] struct{
  some bool
  v V
}

func (o Option[V1]) Map[V2 any](mapper func(V1) V2) Option[V2] {
  if o.some {
    return Option[V2]{some: true, v: mapper(o.v)}
  }
  return Option[V2]{}
}
```

Go 1.27より前の`Go`では`[V2 any]`の部分で文法エラーでした。

generics methodsはinterfaceを満たしません。

```go
// ❌
type GenericRead interface {
  Read[V any](p []V) (int, error)
}

// ❌
var _ io.Reader = (*TypedReader)(nil)

type TypedReader struct{}
func (r *TypedReader) Read[V any](p []V) (int, error) { /* ... */ }
```

- 多くの要望がありながら、Goのmethodは新しくtype parameterを導入することができませんでした
  - 理由は
    - generic methodsはinterfaceを実装すべきだろうという前提のもと、
    - [generic methodsをinterfaceで宣言できた場合、パフォーマンスを落とさずにinstantiateする方法がわからない](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#No-parameterized-methods)からです。
      - link time(コンパイル時)にやろうとすると、すべての呼び出しを探索し、さらにreflectionをかける必要があり、コンパイル時間が激増するはずです。
      - run time(プログラム実行時)でやろうとすると、ある種のJITかreflectionをかける必要があり、複雑かつ低速となります。
  - [Go FAQ: Why does Go not support methods with type parameters?](https://go.dev/doc/faq#generic_methods)でもはっきりとgeneric methodsを実装することはないと言っていました。
- 今回のproposalではgeneric methodsはinterfaceを実装しないということでacceptedとなりました

現時点(2026-05-13)では`GOEXPERIMENT`ガード付きで実装されていますが、[コメント](https://github.com/golang/go/issues/77273#issuecomment-4360427977)を読む限り`Go 1.27`では`GOEXPERIMENT`ガードなしで導入されるようです。

## go/ast, go/typesのAPI変更はない

文法の変更となるため、linterその他が影響を受けることが確実です。

linterなど`Go`のソースコードを解析して何かを行うプログラムは`go/ast`と`go/types`を利用します。

- https://pkg.go.dev/go/ast#FuncDecl
- https://pkg.go.dev/go/types#Signature

両者見てわかる通り、型上generic methodsを表現できるのでAPIの変更自体はありません。

ないのですが、「method receiverとtype paramがどちらもnon-nil」というパターンは今までありえなかったわけですから、ツールによってはうまく動かなくなるでしょう。
プロポーザル上でも言及されていますが、メンテされているツールが追従しきるまで「経験上1～２メジャーリリース程度かかる」こととなるでしょう。

## gotipで試そう

RC(Release Candidate)になっていない最新の変更は[gotip](https://pkg.go.dev/golang.org/dl/gotip)でダウンロードしてtoolchainをビルドできます。

```bash
go install golang.org/dl/gotip@latest
gotip download
export GOEXPERIMENT=genericmethods
export PATH="$(gotip env GOROOT)/bin:${PATH}"
export GOTOOLCHAIN=path
```

`gotip mod init`するか、既存のモジュールを編集する場合は`go.mod`を変更します。

```bash
gotip mod init ...
```

```go.mod diff:go.mod
-go 1.23.0
+go 1.27
```

`go 1.27.0`だとダメで`go 1.27`を指定します。

## メソッドチェーンでiteratorを処理する

https://zenn.dev/ngicks/articles/go-make-everything-iterator
https://zenn.dev/ngicks/articles/go-make-everything-iterator-2

上記の記事で実装した

https://github.com/ngicks/go-iterator-helper

にmethod chain式のhelperを追加してみます。

ここは`claude`を叩いてぺっとやります

https://github.com/ngicks/go-iterator-helper/commit/9bb630d9f07c58175fd3def6c651188f4350a052

### `iter.Seq` / `iter.Seq2`をラップした型を作る

iteratorを処理したいので、それをラップした型を作り、そこにmethodを追加します。

元から単なる`iter.Seq[V]` / `iter.Seq2[K, V]`のラッパーとなる型を定義していました

```go
type SeqIterable[V any] iter.Seq[V]
type SeqIterable2[K, V any] iter.Seq2[K, V]
```

各種アダプタはこれに対するmethodとして実装します。

### 型を変換するmethod

型を変換するmethodが実装できてなんか変な感じしますね。

```go
func (f SeqIterable[V1]) Map[V2 any](mapper func(V1) V2) SeqIterable[V2] {
  return SeqIterable[V2](Map(mapper, iter.Seq[V1](f)))
}

func (f SeqIterable2[K1, V1]) Map[K2, V2 any](mapper func(K1, V1) (K2, V2)) SeqIterable2[K2, V2] {
  return SeqIterable2[K2, V2](Map2(mapper, iter.Seq2[K1, V1](f)))
}
```

```go
func (f SeqIterable[V]) Reduce[Sum any](reducer func(Sum, V) Sum, sum Sum) Sum {
  return Reduce(reducer, sum, iter.Seq[V](f))
}

func (f SeqIterable2[K, V]) Reduce[Sum any](reducer func(Sum, K, V) Sum, sum Sum) Sum {
  return Reduce2(reducer, sum, iter.Seq2[K, V](f))
}
```

### 引数には`iter.Seq`の代わりに`~func(yield func(V) bool)`を使う

genericsの一般的な原則としてconcreteな型を受け取るより`T2 ~T1`を受けたほうがよいのですが、iteratorを`iter.Seq`以外で処理することはないと思われるので、これを受け取るように書かれた関数がほとんどなのではないかと思います。

今回ラップした型(`SeqIterable`)と`iter.Seq`どっちも受け取りたいので、シグネチャを変えたほうがよさそうです。

```go diff
// Concat returns an iterator over the concatenation of the sequences.
-func Concat[V any](seqs ...iter.Seq[V]) iter.Seq[V] {
+func Concat[V any, Seq ~func(yield func(V) bool)](seqs ...Seq) iter.Seq[V] {
  return func(yield func(V) bool) {
    for _, seq := range seqs {
      for e := range seq {
        if !yield(e) {
          return
        }
      }
    }
  }
}

+func (f SeqIterable[V]) Concat[Seq ~func(yield func(V) bool)](seqs ...Seq) SeqIterable[V] {
+  all := make([]Seq, len(seqs)+1)
+  all[0] = Seq(f)
+  copy(all[1:], seqs)
+  return SeqIterable[V](Concat(all...))
+}
```

型推論がかかるため`Seq ~`の部分を明示的に指定することはほとんどないのでこれは破壊的変更ではないとほとんどのケースでみなせると思います。
引数から出呼び出されたときのみ明示的な指定が必要でエラーとなります。

```go diff
-return hiter.Concat[int]()
+return hiter.Concat[int, iter.Seq[int]]()
```

### `iter.Seq[V]`の`V`に制約が必要な処理はmethodで実装できない

以下は実装できません

```go
func (f SeqIterable[V]) Equal[Seq ~func(yield func(V) bool)](other Seq) bool {
  return Equal(iter.Seq[V](f), other)
}

func (f SeqIterable2[K, V]) Collect() map[K]V {
  return maps.Collect(iter.Seq2[K, V](f))
}
```

これらのケースでは`SeqIterable[V]`の`V`や`SeqIterable2[K, V]`の`K`が`comparable`である必要があるためです。
`Rust`だとこういうのはできるんですが`Go`ではできません。微妙にかゆいところに手が届かないケースもあります。

代わりに以下みたいな形でも実装できます。

```go
type From[V any] interface{
  From(seq iter.Seq[V])
}

type From2[K, V any] interface{
  From2(seq iter.Seq2[K, V])
}

func (f SeqIterable[V]) CollectInto[F From[V]]() F {
  out := *new(F)
  out.From(iter.Seq[V](f))
  return out
}

func (f SeqIterable2[K, V]) CollectInto[F From2[K, V]]() F {
  out := *new(F)
  out.From2(iter.Seq2[K, V](f))
  return out
}
```

これはこれで別の厳しさがあります。

- `From`のような自己を変更するmethodは通常method receiverがポインターであるため、`*new(F)`は`typed nil`となる
- これを回避するためには`type FromImpl[V any] struct { v *V }`みたいな形で、ポインターのフィールドを持たせて底を変更させるとかになると思います。

迂遠であるのでこれをやりたい動機はないかなあという感じです。
これに関しては今まで通りトップレベルの関数でやることになるでしょう。

## おわりに

- 文法増えると[dst](https://github.com/dave/dst)が追従できなくなるから反対派でしたが別にastのパターンが増えるわけではないので問題ありませんでした。
- やっぱあると便利です。
- `Equal`とか`Collect`とかが実装できないとトップレベルの関数を実装して使う今まで通りのスタイルとなるため、iteratorの処理としては使わないほうが一貫性はあるかも
