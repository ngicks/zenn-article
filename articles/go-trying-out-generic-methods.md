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

- [The Go Programming Language](https://go.dev/)に対してある程度の習熟している
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

Go 1.27以前の`Go`では`[V2 any]`の部分で文法エラーでした。

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
      - run time(プログラム実行時)でやろうとすると、JITかreflectionをかける必要があり、複雑かつ低速となります。
  - [Go FAQ: Why does Go not support methods with type parameters?](https://go.dev/doc/faq#generic_methods)でもはっきりとgeneric methodsを実装することはないと言っていました。
- 今回のproposalではgeneric methodsはinterfaceを実装しないということでacceptedとなりました

コメント読む限り`Go 1.27`でメインラインに(=`GOEXPERIMENT`ガードなしで)導入されそうな感じです。

## go/ast, go/typesのAPI変更はない

- https://pkg.go.dev/go/ast#FuncDecl
- https://pkg.go.dev/go/types#Signature

両者見てわかる通り、型上generic methodsを表現できるのでAPIの変更自体はありません。

ないのですが、`go/ast`, `go/types`を利用するコードが今までなかったgeneric methodsのパターンに対応できるかというとそういうわけではないと思います。
プロポーザル上でも言及されていますが、メンテされているツールが追従しきるまで「経験上1～２メジャーリリース程度かかる」でしょう。

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

iteratorを処理したいので、それをラップ下方を作り、そこにmethodを追加します。

元から単なる`iter.Seq[V]` / `iter.Seq2[K, V]`のラッパーとなる型を定義していたのでこれに対するmethodとして実装します。

```go
type SeqIterable[V any] iter.Seq[V]
type SeqIterable2[K, V any] iter.Seq2[K, V]
```

### 型を変更するmethod

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

### `iter.Seq`の代わりに`~func(yield func(V) bool)`を受け取るようにする

genericsの一般的な原則としてconcreteな型を受け取るより`T2 ~T1`を受けたほうがよいのですが、iteratorを`iter.Seq`以外で処理することはないのでこれを受け取るように書かれた関数が多いのではないかと思います。

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

### EqualとかCollect2はmethodに実装できない

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

やる意味ないかな？

## おわりに

- 文法増えると[dst](https://github.com/dave/dst)が追従できなくなるから反対派でしたが別にastのパターンが増えるわけではないので問題ありませんでした。
- やっぱあると便利です。
- `Equal`とか`Collect2`とかが実装できないとトップレベルの関数を実装して使う今まで通りのスタイルとなるため、iteratorの処理としては使わないほうが一貫性はあるかも

[#77273]: https://github.com/golang/go/issues/77273

<!-- linux syscalls -->

[pthread(7)]: https://man7.org/linux/man-pages/man7/pthreads.7.html

<!-- other languages referenced -->

[Java]: https://www.java.com/
[TypeScript]: https://www.typescriptlang.org/
[python]: https://www.python.org/
[C]: https://www.c-language.org/
[C++]: https://isocpp.org/
[Rust]: https://www.rust-lang.org
[The Rust Programming Language 日本語]: https://doc.rust-jp.rs/book-ja/
[Lua]: https://www.lua.org/

<!-- other lib/SDKs referenced -->

[Node.js]: https://nodejs.org/en
[deno]: https://deno.com/
[tokio]: https://tokio.rs/

<!-- editors -->

[Visual Studio Code]: https://code.visualstudio.com/
[vscode]: https://code.visualstudio.com/
[neovim]: https://neovim.io/

<!-- tools -->

[git]: https://git-scm.com/
[Git Credential Manager]: https://github.com/git-ecosystem/git-credential-manager?tab=readme-ov-file
[Docker]: https://www.docker.com/
[podman]: https://podman.io/
[podman-static]: https://github.com/mgoltzsche/podman-static
[Dockerfile]: https://docs.docker.com/build/concepts/dockerfile/
[Elasticsearch]: https://www.elastic.co/docs/solutions/search

<!-- Go versions -->

[Go]: https://go.dev/
[Go 1.11]: https://go.dev/doc/go1.11
[Go 1.14]: https://go.dev/doc/go1.14
[Go 1.18]: https://go.dev/doc/go1.18
[Go 1.23]: https://go.dev/doc/go1.23
[Go 1.24]: https://go.dev/doc/go1.24
[Go 1.25]: https://go.dev/doc/go1.25

<!-- Go doc links -->

[A Tour of Go]: https://go.dev/tour/welcome/
[GOAUTH]: https://pkg.go.dev/cmd/go#hdr-GOAUTH_environment_variable

<!-- Go tools -->

[gopls]: https://github.com/golang/tools/tree/master/gopls
[github.com/go-task/task]: https://github.com/go-task/task

<!-- references to spec -->

[type assertion]: https://go.dev/ref/spec#Type_assertions
[type switch]: https://go.dev/ref/spec#Type_switches

<!-- references to sdk library -->

[panic]: https://pkg.go.dev/builtin@go1.26.2#panic
[errors.New]: https://pkg.go.dev/errors@go1.26.2#New
[errors.Is]: https://pkg.go.dev/errors@go1.26.2#Is
[errors.As]: https://pkg.go.dev/errors@go1.26.2#As
[errors.Join]: https://pkg.go.dev/errors@go1.26.2#Join
[fmt.Errorf]: https://pkg.go.dev/fmt@go1.26.2#Errorf
[fs.ErrNotExist]: https://pkg.go.dev/io/fs@go1.26.2#ErrNotExist
[http.Server]: https://pkg.go.dev/net/http@go1.26.2#Server
[*http.Server]: https://pkg.go.dev/net/http@go1.26.2#Server
[io.EOF]: https://pkg.go.dev/io@go1.26.2#EOF
[io.Reader]: https://pkg.go.dev/io@go1.26.2#Reader
[io.Writer]: https://pkg.go.dev/io@go1.26.2#Writer
[net/http]: https://pkg.go.dev/net@go1.26.2
[syscall.Errno]: https://pkg.go.dev/syscall@go1.26.2#Errno
[text/template]: https://pkg.go.dev/text/template@go1.26.2
