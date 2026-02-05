---
title: "Goのプラクティスまとめ: error handling"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## Goのプラクティスまとめ: error handling

筆者が`Go`を使い始めた時に分からなくて困ったこととか最初から知りたかったようなことを色々まとめる一連の記事です。

以前書いた記事のrevisited版です。話の粒度を細かくしてあとから記事を差し込みやすくします。

他の記事へのリンク集

- (まだ)~~[今はこうやる集](https://zenn.dev/ngicks/articles/go-basics-revisited-updated-practices)~~
- [プロジェクトを始めるまで](https://zenn.dev/ngicks/articles/go-basics-revisited-starting-projects)
- [dockerによるビルド](https://zenn.dev/ngicks/articles/go-basics-revisited-bulding-with-docker)
- `error handling`: ここ
- (まだ)~~[fileとio](https://zenn.dev/ngicks/articles/go-basics-revisited-file-and-io)~~
- (まだ)~~[jsonやxmlを読み書きする](https://zenn.dev/ngicks/articles/go-basics-revisited-data-encoding)~~
- (まだ)~~[cli](https://zenn.dev/ngicks/articles/go-basics-revisited-cli)~~
- (まだ)~~[environment variable](https://zenn.dev/ngicks/articles/go-basics-revisited-environment-variable)~~
- (まだ)~~[concurrent Go](https://zenn.dev/ngicks/articles/go-basics-revisited-concurrent-go)~~
- (まだ)~~[context.Context: long running taskとcancellation](https://zenn.dev/ngicks/articles/go-basics-revisited-context)~~
- (まだ)~~[http client / server](https://zenn.dev/ngicks/articles/go-basics-revisited-http-client-and-server)~~
- (まだ)~~[structured logging](https://zenn.dev/ngicks/articles/go-basics-revisited-structured-logging)~~
- (まだ)~~[test](https://zenn.dev/ngicks/articles/go-basics-revisited-test)~~
- (まだ)~~[filesystem abstraction](https://zenn.dev/ngicks/articles/go-basics-revisited-filesystem-abstraction)~~

(リンク集は別の記事で出そうかと思ったんですが、そういえばzennだとリンク集とか見たことがない、本使えと怒られるかもなあ・・・とちょっと不安になったので一連の記事に相互リンクを張る形にします。)

## EDIT NOTE

2025-12-24: go1.26で追加された[errors.AsType]を追記。go1.26はrc1なのでまだ使えません。[errors.As]の第二引数が任意のinterfaceをとれる野を追記。

## error handling

`Go`には`try-catch`のような構文や、`Result<T, E>`のようなtagged union typeのようなものは現状存在しません。(sum typeのproposalは長く存在するが一向に進まない。[#19412](https://github.com/golang/go/issues/19412), [#57644](https://github.com/golang/go/issues/57644)など)
代わりに、`Go`は関数が複数の値を返すことが可能で、慣習的にソース順で最後の返り値の型を`error`とすることでerrorしうる処理を表現します。
error handlingとはその値をチェックすることをさします。

この記事ではerror handling, errorの組み立て方などについて色々述べます。

## 前提知識

- [A Tour of Go](https://go.dev/tour/welcome/)
- ほか言語での開発経験

## 環境

```
# go version
go version go1.23.2 linux/amd64
```

## TL;DR

- [fmt.Errorf]でerrorはラップできる
  - 基本的にラップしてメッセージを追加したほうがよい
  - ただし[io.EOF]などはラップしてはいけない
- [errors.Is], [errors.As]/[errors.AsType]\([Go 1.26]以降\)でerrorを判別する
  - `os.Open`のerrorの判別は[fs.ErrNotExist]などと比較するとよい
- `error`を実装する型を定義する際には以下を気を付ける
  - method receiverはpointerのほうが良い
  - typed-nilに注意する
- panic-recoverで一気に抜けることもできる
  - 自分で行ったpanicが以外を拾わないように、特定の型の値でpanicするなど注意する
  - resourceの解放は常にdeferで行うとよい
    - [http.Server]がpanicを勝手にrecoverしてしまうため、panicは勝手にrecoverされるものと思っておいたほうが良い
    - また、呼び出しているライブラリの関数がふいにpanicすることも十分ありうる
- stacktraceはerrorについて回らないので付けたかったら自分でつける
  - `recover`した関数内でstacktraceを取得するとpanicのstacktraceを取得できるので、ログしたいときはこれを用いる

## errorはinterface

下記に示される通り、`error`とは`Error`メソッドのみをもつ`interface`です。

```go
// https://pkg.go.dev/builtin#error

// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface {
    Error() string
}
```

`error`型の値がnon-nilであるとき、`Error`は返り値でerrorが何に関してなのかとか、どうして起きたのかとかを説明します。

`error`はさらにほかのerrorをラップすることで木構造を構築することがあり、その木構造の中に特定の型や、特定の値を含むことで、どのようなerrorであったのか判定に使われることがあります。
値や型は後述する[errors.Is]や[errors.As]/[errors.AsType]\([Go 1.26]以降\)で探索されます。

## 基本: 失敗ならerr != nil

タイトルのような慣習(というべきなのかよくわかりませんが)があります。

- `err == nil`ならば、ほかの返り値は使っても**よい**
  - 返り値の最後の値が`error`型であるとき、それがnilならばそれ以外の値は基本的に使ってよいことになります。
    - `interface`, `*T`, `chan T`, `map[K]V`はnon-nil
    - `T`は(それがふさわしければ)non-zero
    - (ただしスライス`[]T`がnilなことは比較的普通かも)
- `err != nil`なら、ほかの返り値は基本使っては**いけない**
  - 明確にnon-nil error時に他の値を使ってもよいとドキュメントされている場合除きます。
    - e.g. `io.Reader`が`n > 0, io.EOF`を返してくることがある

慣習的にポインターを返す関数がそれの返り値のnil checkをさせることはほとんどありません。
かわりに最後の返り値のerrorのnil check, もしくは`ok bool`をチェックさせます。

```go
func failableWork(...any) (ret1 io.Reader, ret2 *UltraBigBigData, err error)

func foo() error {
    ret1, ret2, err := failableWork()
    if err != nil {
        // ret1, ret2は普通はzero value.
        // io.Reader, *Tの場合はnilなので、
        // ret1.Readを呼ぶとnil pointer derefでパニックする
        return err
    }

    // この時点ret1, ret2は普通はnon-nilな値であり、
    // メソッドよんだり、pointer derefするなりしても安全。
    n, err := ret1.Read(make([]byte, 5))
    if err != nil {
        // ...
    }
    _ = *ret2
    // ret1, ret2を使って何かする
    return nil
}
```

## 慣習的にerror型の変数名はerr

特別な事情がない限り`err`という名前の変数を使用することが多いです。

また基本的に`err`という名前の変数は複数回使いまわします。
のちにそのerrorが参照されるわけではない場合、別の変数名を当てるのはやめましょう。

```go
foo, bar, err := failableWork1()
if err != nil {
    // ...
}
baz, qux, err := failableWork2()
if err != nil {
    // ...
}
```

## errorの判別

`err != nil`でerrorなことはわかるけどどういうerrorなのかを判定したいときは多くあります。
例えばファイルを開くのがerrorしたとき、`ENOENT`なのか`EPERM`なのかぐらいは最低でも知らないとハンドルできないですよね。

- errorは基本的に、特定の値(pointerなど)で特定できるものと、型で特定できるものがある
  - e.g. 値 => [io.EOF](https://pkg.go.dev/io#EOF), [net.ErrClosed](https://pkg.go.dev/net@go1.22.3#ErrClosed), [(os/exec).ErrNotFound](https://pkg.go.dev/os/exec@go1.22.3#ErrNotFound)など
  - e.g. 型 => [\*(encoding/json).SyntaxError](https://pkg.go.dev/encoding/json@go1.22.3#SyntaxError), [\*(io/fs).PathError](https://pkg.go.dev/io/fs@go1.22.3#PathError)
- 取り合えず[errors.Is], [errors.As]/[errors.AsType]\([Go 1.26]以降\)を使っておけばよい

### errors.Is, errors.As, errors.AsType(Go 1.26以降)

[errors.Is]でerrorが特定の**値**を含むのかどうか、[errors.As]/[errors.AsType]\([Go 1.26]以降\)でerrorが特定の**型**を含むのかどうかを判定できます。

`Go`におけるerrorは木構造を持てます。木構造の構築のしかたは後述しますが、その木構造をdepth-firstで探索して特定の値を含むか、もしくは特定の型を含むかの判定を上記の二つの関数で行います。

### errors.Is: errorが特定の値を含むかどうかの判別

前述の[(os/exec).ErrNotFound](https://pkg.go.dev/os/exec@go1.22.3#ErrNotFound)の例で[errors.Is]を利用すると以下のようになります。

```go
var someCommand string = "something"
_, err := exec.LookPath(someCommand)
if errors.Is(err, exec.ErrNotFound) {
    fmt.Printf("command %q not found\n", someCommand)
} else err != nil {
    // handle error other than "not found"...
}
// continue working...
```

### errors.As/errors.AsType: errorが特定の型を含むかどうかの判別

前述の[\*(encoding/json).SyntaxError](https://pkg.go.dev/encoding/json@go1.22.3#SyntaxError)の例で、[errors.As]を利用すると以下のようになります。

[playground](https://go.dev/play/p/J1t5RJ1_3gY?v=gotip)

```go
var tgtData any
err := json.Unmarshal(brokenJsonBinary, &tgtData)
var syntaxErr *json.SyntaxError
if errors.As(err, &syntaxErr) {
    fmt.Printf("err = %#v\n", syntaxErr)
    // err = &json.SyntaxError{msg:"unexpected end of JSON input", Offset:6}
}
```

[errors.As]は第二引数で取り出したいerrorの具体的な型の**変数へのpointer**を渡します。
pointer渡しするのは、`As`が第一引数の`err`を探索しながら、第二引数に渡された値の型に代入可能なものを探し、可能ならば代入するからです。
ですので、`As`がtrueを返す時、上記の`syntaxErr`は取り出されたerrorの値となっています(Offset: 6のように、zero valueでなくなっている。)

任意のinterfaceを対象とすることもできます。

```go
type AError struct{}

func (e *AError) Error() string {
    return "a"
}

func (e *AError) A() {}

err := error(&AError{})
var aErr interface{ A() }
if errors.As(err, &aErr) {
    fmt.Printf("err = %#v\n", aErr) // err = &main.AError{}
}
var bErr interface{ B() }
if errors.As(err, &bErr) {
    fmt.Printf("err = %#v\n", bErr)
}
```

Go1.26以降では、[errors.AsType]を使うこともできます。(まだRelease Candidate 1なので使えません!)
`AsType`は`As`と違い、対象は`error`を実装している必要があります。

```go
var tgtData any
err := json.Unmarshal(brokenJsonBinary, &tgtData)
if syntaxErr, ok := errors.AsType[*json.SyntaxError](err); ok {
    fmt.Printf("err = %#v\n", syntaxErr)
    // err = &json.SyntaxError{msg:"unexpected end of JSON input", Offset:6}
}

if aErr, ok := errors.AsType[interface {
    error
    A()
}](&AError{}); ok {
    fmt.Printf("err = %#v\n", aErr) // err = &main.AError{}
}
```

### err == tgt / err.(T)

errorがラップされていない状況においてのみ、

- comparison: `err == tgt`
- [type assertion]\: `err.(T)`
- [type switch]\: `switch x := err.(type) {}`

でもerrorの判別を行えます。

基本的に[errors.Is]か[errors.As]/[errors.AsType]を使っておくほうが好ましいです。前述通りこれらはラップされているerrorに対しても正常に動作するからです。

```go
// comparison
if err == io.EOF {
    // handle eof...
}

// type-assertion
syntaxErr, ok := err.(*json.SyntaxError)
if ok {
    // handle error...
    _, err := r.Seek(syntaxErr.Offset, io.SeekStart)
}

// type-switch
switch x := err.(type) {
case nil:
    fmt.Println("no error")
case *json.SyntaxError:
    // このブランチではxは*json.SyntaxError型
    _, err := r.Seek(x.Offset, io.SeekStart)
}
```

### 例: filesystem関連errorの判別

`Go`には、[io/fs](https://pkg.go.dev/io/fs@go1.23.4)パッケージがあり、抽象的な`fs`操作が可能であることから、典型的なerrorは`fs`パッケージで再定義されています。

[syscall.Errno]が`interface { Is(error) bool }`という`errors`パッケージが特別な考慮を行うためのinterfaceを実装しているので、それをラップして返す`os`パッケージのerrorに対しての判別に使うことができます。

[snippet](https://github.com/ngicks/go-example-basics-revisited/blob/main/error-handling/is-fs-error/main.go)

```go
f, err := os.Create("path/to/file")
if err != nil {
    switch {
    case errors.Is(err, fs.ErrNotExist):
        // errors.Is(err, syscall.ENOENT)と同じ効果
        fmt.Println(fs.ErrNotExist.Error()) // file does not exist
    case errors.Is(err, fs.ErrPermission):
        // errors.Is(err, syscall.EPERM)と同じ効果
        fmt.Println(fs.ErrPermission.Error())
    case errors.Is(err, syscall.EROFS):
    }
}
```

`fs`で定義されていないerrorに関しては[syscall](https://pkg.go.dev/syscall@go1.23.4)パッケージ定義される[syscall.Errno]と[errors.Is]で比較を行います。これは[errno(3)](https://man7.org/linux/man-pages/man3/errno.3.html)の数値に`error`を実装したものですが、windows向けにも同名シンボルが定義してあるのでwindowsに対しても使うことができます。

:::details windows向けに*invent*されたerrno

https://github.com/golang/go/blob/go1.23.4/src/syscall/zerrors_windows.go#L6-L148

雑に`go doc -all syscall | sed -n '/^const (/,/^)/p'`を`GOOS=linux`, `GOOS=windows`で出し分けてみましたがどちらも133個だったのでwindows向けにも全部*invent*されているっぽいですかね。

これらの`Errno`を利用する場合には自身で`GOOS=windows`でも`go build`してみることで型チェックを通しておくことをお勧めします。(`package main`でないパッケージに対して`go build`をかけると単に型チェックだけがかかる。`go vet`でもよいです。)

:::

## errorのラッピング

前述通りGoの`error`は木構造を持つことができ、[errors.Is], [errors.As]/[errors.AsType]\([Go 1.26]以降\)を用いることでその木構造をdepth-firstに探索したマッチができます。
木構造をたどるには`interface { Unwrap() error }`もしくは`interface { Unwrap() []error }`の実装をチェックし、呼び出すわけですが、この語彙を逆にして、`error`を子ノードとしてもつ`error`を作成することを「ラップする」/「ラッピング」などと呼びます。
その方法について以下で述べます。

### fmt.Errorf

いちばん手軽なのは[fmt.Errorf]を使うことでerrorをラップする方法です。

format stringで`%w` verbを指定し、引数に`error`型を渡すことでラップを行います。
`%w`なしだと別のerrorをラップしないerrorを得られます。
`%w`以外のverbは他の`fmt.*printf`系の関数と同じように使えます。error messageを`fmt.Sprintf`で出力するのと等価です。

[snippet](https://github.com/ngicks/go-example-basics-revisited/blob/main/error-handling/wrap-error/main.go)

```go
err := fmt.Errorf("foo")

wrapped := fmt.Errorf("bar: %w, param1 = %d, param2 = %.2f", err, 10, 0.1234)

fmt.Printf("err = %v\n\n", wrapped) // err = bar: foo, param1 = 10, param2 = 0.12

fmt.Printf("not same = %t\n", err == wrapped)                // not same = false
fmt.Printf("but is wrapped = %t\n", errors.Is(wrapped, err)) // but is wrapped = true
```

複数のerrorを渡すことで複数のerrorをラップする単一のerrorを得られます。

```go
err1 := fmt.Errorf("foo")
err2 := fmt.Errorf("bar")
err3 := fmt.Errorf("baz")

wrapped := fmt.Errorf("multiple: %w, %w, %w", err1, err2, err3)

fmt.Printf(
    "wraps all = %t, %t, %t\n",
    errors.Is(wrapped, err1),
    errors.Is(wrapped, err2),
    errors.Is(wrapped, err3),
) // wraps all = true, true, true
```

複数errorを持つためここで木構造が生じます。

当然[errors.As]も機能します。

```go
type wrappeeErr struct {
    error
}

wrapped := fmt.Errorf("bar: %w", wrappeeErr{fmt.Errorf("foo")})

var tgt wrappeeErr
fmt.Printf("wrapped = %t\n", errors.As(wrapped, &tgt)) // wrapped = true
```

### 型を定義する

別のerrorを含むことができる`error` interfaceを実装する型を定義することでもerrorをラップすることができます。

この場合、`Unwrap() error`もしくは`Unwrap() []error`を実装することで[errors.Is], [errors.As]がこれらを*unwrap*することができます。

[snippet](https://github.com/ngicks/go-example-basics-revisited/blob/main/error-handling/wrap-error/main.go)

```go
type customErr struct {
    Reason string
    Param  any
    Err    error
}

func (e *customErr) Error() string {
    return fmt.Sprintf("*customErr: %s, param = %v", e.Reason, e.Param)
}

func (e *customErr) Unwrap() error {
    return e.Err
}

func wrapByCustomError() {
    err := fmt.Errorf("foo")

    wrapped := &customErr{
        Reason: "bar",
        Param:  "baz",
        Err:    err,
    }

    fmt.Printf("err = %v\n", wrapped)
    // err = *customErr: bar, param = baz
    fmt.Printf("wrapped = %t\n", errors.Is(wrapped, err))
    // wrapped = true
}
```

### 例外: io.EOFはラップしない

基本的にはerrorはラップすることでメッセージを付け足せるのでしたほうがよいのですが、例外的にラップしてはいけないerrorもあります。

それは[io.EOF]のような、

- sentinel valueとして用いられるerror
- `Go1.13`以前から利用されていたもの

です。

errorのラッピングの概念は[Go 1.13](https://go.dev/doc/go1.13#error_wrapping)([2019-09-03](https://go.dev/doc/devel/release#go1.13)以降)からstdに追加されています。
ですから、`Go1.13`以前に利用されるerrorはラッピングを考慮していないことがあります。

stdの範囲でも、`err == io.EOF`のように比較を行うことでこれらのsentinel valueとして使うツールがいくつかあります。

例えば[io.Copy](https://pkg.go.dev/io@go1.23.4#Copy)は[io.Reader]が[io.EOF]を返したときにnilを返す考慮があります。

https://github.com/golang/go/blob/go1.23.4/src/io/io.go#L444-L456

`== io.EOF`で`Go`のrepositoryを検索するとすごくたくさん出てきます。

https://github.com/search?q=repo%3Agolang%2Fgo+%3D%3D+io.EOF&type=code

このようなsentinel valueとして使われるerrorの値はラップしないようにしてください。
特に特定の`interface`(e.g. [io.Reader])を実装してそれを使う関数に渡す時、特定のerrorを返すのが`interface`の規約として決まっているとき(e.g. [io.EOF])、そのerrorはラップしないように気を付けてください。

## error valueを定義する

パッケージとして判別可能なerrorを返す時はexportされた変数を[errors.New]か[fmt.Errorf]で定義し、失敗しうる関数はこれを直接返すか、ラップして返します。

```go
var (
    ErrCauseCause = errors.New("root cause..")
    ErrFooFoo     = fmt.Errorf("foo foo")
)

// someFailableWork does ...
// When ... someFailableWork returns [ErrCauseCause]
func someFailableWork() (..., error) {
    return ..., fmt.Errorf("cause of failures...: %w", ErrCauseCause)
}
```

- 慣習的に`Err`から始まる変数名を使います
- 慣習的にerror messageはすべて小文字にします。
  - `fmt.Errrof("foo bar: %w",  fmt.Errorf("baz qux: %w", err))`みたいな感じでメッセージをつなげていくことが多いため、先頭大文字だと変に見えるからです。
- errorを返す関数はどのようなときに、どのerrorをラップして返すかをdoc commentで明確に説明します
  - `doc comment`上で`[SymbolName]`とするとリンクとして機能する挙動がgo docにはあるため、これを積極的に用います。
- error messageはわかりやすければ何でもいいですが、root causeのみを説明するとよいです
  - errorをラップして上位のコンテクストを徐々に追加形式になりがちなため、冗長なerror messageは重複を生みかねないためです。
  - i.e. `ErrNotEligible = errors.New("not eligible")`, `uid = 10 : unprivileged user: not eligible`

## error typeを定義する

[fmt.Errorf]でerrorをラッピングして回れば事足りる場面も多いですが、例えば下記のようなとき`error` interfaceを実装する型を定義することがあります。

- あとからパラメータをとり出したい
- あとからパラメータを変更したい
- ほぼ共通だが複雑なerror messageの構築処理がある
- 複数のcategoryに複数のkindがるため、categoryを型として表現したい
  - e.g. jsonにおけるio error, syntax error, semantic error

型を定義する際に気をつけるべき注意点を述べます。

すでにサンプルの中で使用していますが、定義自体は以下のように行います。

```go
type customErr struct {
    Reason string
    Param  any
    Err    error
}

func (e *customErr) Error() string {
    return fmt.Sprintf("*customErr: %s, param = %v", e.Reason, e.Param)
}

func (e *customErr) Unwrap() error {
    return e.Err
}
```

### method receiverはpointerのほうが好ましい

つまり下記のような形です。

```go
// こうではなく
func (e customErr) Error() string

// こう
func (e *customErr) Error()
```

これは二つ理由があります。

- (1) non-pointer type `T`がmethod receiverであるとき、`T`、`*T`どちらもinterfaceを満たすこと
- (2) interfaceはspec上comparableだが、比較するとき2つのinterfaceの*dynamic types*が同一でcomparableでないときruntime-panicが起きること

(1)に関しては単純に紛らわしいということです。特定の型のerrorを返す場合はドキュメントに明確に書いておくほうが良いので、どちらでもよいといえばいいのですが、method receiverがpointerであればpointerでないとinterfaceを満たせないためどちらなのかを気にする必要すらありません。
ただし、`type someErr int`のようなbuilt-inかつcomparableな型をベースとする場合はmethod receiverはnon-pointerであるほうが一般的だと思います。これはある程度のサイズ(昔ググった時はdouble型が3つ分以上、という風に言われてました)がないデータはpointerをderefするより値を渡してしまったほうがより高速であるからという理由があるからなはずです(特に出展を示せません。)

(2)に関しては以下のsnippetをご覧ください。

[snippet](https://github.com/ngicks/go-example-basics-revisited/blob/main/error-handling/panic-on-uncomparable-error-kind/main.go)

```go
package main

import (
    "fmt"
)

type uncomparableErr1 []error

func (e uncomparableErr1) Error() string {
    return "uncomparableErr1"
}

type uncomparableErr2 struct {
    errs []error
}

func (e uncomparableErr2) Error() string {
    return "uncomparableErr2"
}

func compareErr(err1, err2 error) {
    if err1 == err2 {
        fmt.Println("huh?")
    }

    func() {
        defer func() {
            if rec := recover(); rec != nil {
                fmt.Printf("comparing err1 panicked = %v\n", rec)
            }
        }()
        if err1 == err1 {
            fmt.Printf("err1 equal = %v\n", err1)
        }
    }()

    func() {
        defer func() {
            if rec := recover(); rec != nil {
                fmt.Printf("comparing err2 panicked = %v\n", rec)
            }
        }()
        if err2 == err2 {
            fmt.Printf("err2 equal = %v\n", err2)
        }
    }()
}

func main() {
    ue1 := uncomparableErr1{fmt.Errorf("foo"), fmt.Errorf("bar")}
    ue2 := uncomparableErr2{errs: []error{fmt.Errorf("foo"), fmt.Errorf("bar")}}

    // invalid operation: ue1 == ue1 (slice can only be compared to nil) compiler(UndefinedOp)
    // if ue1 == ue1 {
    // }

    compareErr(ue1, ue2)
    // comparing err1 panicked = runtime error: comparing uncomparable type main.uncomparableErr1
    // comparing err2 panicked = runtime error: comparing uncomparable type main.uncomparableErr2

    compareErr(&ue1, &ue2)
    // err1 equal = uncomparableErr1
    // err2 equal = uncomparableErr2
}
```

上記の通り、non-pointer typeをmethod receiverとしたときに、`error` typeとして`uncomparableErr1`, `uncomparableErr2`を引数に渡すと、`err == err`でcomparing uncomparable typeでrun-time panicをおこします。
この挙動はspecのcomparison operatorsの部分で明記されています。

> https://go.dev/ref/spec#Comparison_operators
>
> A comparison of two interface values with identical dynamic types causes a run-time panic if that type is not comparable.

つまり別の`error` type, 例えば`io.EOF`との比較はrun-time panicになりませんので、大きな問題にはなりにくいでしょう。

問題になるのは例えば、複数回同じ関数を実行してerrorが比較されるときなどでしょうか？
こういった比較を行うことがありうるかは筆者には想像がつきませんが、compilation errorにならずにrun-time errorになってしまうため、避けられるらならさけたほうがいい問題でしょう。

`error`のdynamic typeがpointerである場合、当然ですがpointerのアドレス同士の比較となるためrun-time panicとなりません。

もし仮に、これらのerror typeのmethod receiverをpointerに変えると

```diff go
type uncomparableErr1 []error

-func (e uncomparableErr1) Error() string {
+func (e *uncomparableErr1) Error() string {
    return "uncomparableErr1"
}
```

`compareErr(ue1, ue2)`の部分でcompilation errorとなります。pointerではないので、`error` interfaceを満たせなくなるためです。

```
cannot use ue1 (variable of type uncomparableErr1) as error value
in argument to compareErr: uncomparableErr1 does not implement error
(method Error has pointer receiver)compiler(InvalidIfaceAssign)
```

こうすると同様に、関数の返り値が`error` typeであるときにnon pointerの`uncomparableErr1`を返すこともcompilation errorとすることができます。
そのため事故的にさえnon pointer typeを`error`として返すことがなくなります。

上記より、`error` interfaceを満たす型はpointer receiverで`Error`メソッドを実装したほうが良いです。ただし例外として`int`, `float64`のような組み込み型、サイズの小さいcomparableな型をunderlying typeとする`error`は`Error` methodのreceiverをnon-pointerとしておいたほうがいいでしょう(c.f. [syscall.Errno])。

### typed nilに注意

stdを含めて、多くのライブラリが自らが定義した`error` typeを返り値の型に使うことはなく、`error` interfaceで返すことが多いです。

```go
// こうではなく
func failableWork() (any, *MyError)
// こう
func failableWork() (any, error)
// たとえ、実際には`&MyError{}`を返しているときでも。
```

:::details 自らが定義したerror typeの値を後から変更するにはどうするか

決まり切った型のerrorしか返さないunexport functionの返り値のerrorがnon-nilなときに、中身を変更したいときはstdは下記のようにしています。

https://github.com/golang/go/blob/go1.23.4/src/net/http/client.go#L716-L718

`type assertion`で特定の型として取り出し、そのうえで中身を変更します。

こうやって中身を変えるという観点からも`error` typeのmethod receiverはpointerであるべきだといえます。
non-pointerとして取り出して変更する場合、変更した変数を元の`err`に代入しなおす必要がありますが、interfaceに変換される際にbox化でallocationが起きるため避けたほうが良いです。

:::

それはなぜなのかというと

- `error` typeに変換するときのtyped nilの可能性
- `func() (..., error)`なinterfaceを満たせない
- 後方互換性のために、その関数が返す型を追加したり変えたりできなくなる

後者二つはまあそのままなので分かると思います。

問題はtyped-nilで、下記のようなことが起きます。

[snippet](https://github.com/ngicks/go-example-basics-revisited/blob/main/error-handling/typed-nil/main.go)

```go
type MyError struct {
    Msg string
}

func (e *MyError) Error() string {
    return e.Msg
}

func myTask() (someResult string, err *MyError) {
    return "ok", nil
}

func someTask() (string, error) {
    ret, err := myTask()
    return ret, err
}

func main() {
    ret, err := someTask()
    if err == nil {
        fmt.Println("success")
    } else {
        fmt.Println("failed") // failed
    }
    fmt.Printf("ret = %s, err = %#v\n", ret, err) // ret = ok, err = (*main.MyError)(nil)
}
```

`someTask`から返ってくる`err`がnon-nilとなっています。
これは、`myTask`が`*MyError(nil)`を返し、`someTask`はそれを`error`に変換しているためです。

`Go`のmethodは暗黙的にreceiverを第一引数とする関数のように取り扱われます。

:::details method呼び出しをobjdumpしてmethod receiverの扱いを確かめる

下記のソースを用意します。`*foo.Bar`の内容には今回関心がないので、`//go:noinline`をつけて、`main`にこの関数がinlineされないようにします。

```go
package main

import "fmt"

type foo int

//go:noinline
func (f *foo) Bar(baz int) {
    fmt.Println(f, baz)
}

func main() {
    f := foo(0x55)
    f.Bar(0x123)
}
```

`go build -o main ./`で適当にバイナリ出力し、`objdump -d ./main > main.exec.txt`とすると以下のように出力されます。(かなり端折ってます)

```
000000000048f1e0 <main.main>:
  48f1e0:    49 3b 66 10              cmp    0x10(%r14),%rsp
  48f1e4:    76 2b                    jbe    48f211 <main.main+0x31>
  48f1e6:    55                       push   %rbp
  48f1e7:    48 89 e5                 mov    %rsp,%rbp
  48f1ea:    48 83 ec 10              sub    $0x10,%rsp
  48f1ee:    48 8d 05 eb 94 00 00     lea    0x94eb(%rip),%rax        # 4986e0 <type:*+0x86e0>
  48f1f5:    e8 86 d6 f7 ff           call   40c880 <runtime.newobject>
  48f1fa:    48 c7 00 55 00 00 00     movq   $0x55,(%rax)
  48f201:    bb 23 01 00 00           mov    $0x123,%ebx
  48f206:    e8 35 ff ff ff           call   48f140 <main.(*foo).Bar>
  48f20b:    48 83 c4 10              add    $0x10,%rsp
  48f20f:    5d                       pop    %rbp
  48f210:    c3                       ret
  48f211:    e8 ca a9 fd ff           call   469be0 <runtime.morestack_noctxt.abi0>
  48f216:    eb c8                    jmp    48f1e0 <main.main>
```

```
  48f1e0:    49 3b 66 10              cmp    0x10(%r14),%rsp
  48f1e4:    76 2b                    jbe    48f211 <main.main+0x31>
...
  48f211:    e8 ca a9 fd ff           call   469be0 <runtime.morestack_noctxt.abi0>
  48f216:    eb c8                    jmp    48f1e0 <main.main>
```

まではstack growth preambleとかと呼ばれていて、(多分)すべての関数の先頭についています。`Go`は、というか`goroutine`はstackが固定サイズでなく成長することがあるので、まず成長が必要かのチェックが走るらしいです。さらにこの`morestack`の呼び出しの中でcooperativeな`goroutine`の切り替えが起こることがあります。つまり特定のタイミングで、stack growthが不要でも必要であるかのようにふるまうことがあります。

method receiverがpointerであるが、値はnon-pointerであるので自動的にobjectに変換されています。

```
  48f1ee:    48 8d 05 eb 94 00 00     lea    0x94eb(%rip),%rax        # 4986e0 <type:*+0x86e0>
  48f1f5:    e8 86 d6 f7 ff           call   40c880 <runtime.newobject>
```

この直後に`%rax`(=`runtime.newobject`の返り値である`unsafe.Pointer`)の指し示すアドレスにimmediate valueの`0x55`をコピーしています。
methodの引数である`0x123`は値渡しなので`bx`にコピーしています。

```
  48f1fa:    48 c7 00 55 00 00 00     movq   $0x55,(%rax)
  48f201:    bb 23 01 00 00           mov    $0x123,%ebx
  48f206:    e8 35 ff ff ff           call   48f140 <main.(*foo).Bar>
```

という感じで、レジスタにmethod receiver,methodの引数が置かれていますね。
`Go`は筆者がdeassembleしている限りにおいては関数のcalling conventionとして引数と返り値はレジスタに直接置く形をとっています。ですので、`ax`が第一引数、`bx`が第二引数となっています。

筆者はアセンブリにもamd64にも全く詳しくないのでこれ以上はよくわかりません。

:::

型を持つ`nil`に対するmethodの呼び出しは、method receiverがnon-pointerならnil pointer dereferenceでrun-time panicとなりますが、pointer receiverならばnilを渡すことにになります。関数の引数がpointerであるとき、そこにnilを渡すことが何かの意味を持つというのは普通にありうる話であり、これを禁じるのもまた、おかしな話です。

であるため、interfaceへの変換がかかる部分で、型情報のあるnilを渡すと、変換後のinterfaceはnon-nilとなります。nilをreceiverとしたmethodの呼び出しは合法であり、意図的にそれをすることも十分ありえるからでしょう。

この場合、以下のように変更すれば、`nil`がプリントされます。

[playground](https://go.dev/play/p/Vx6dsFYTtpB)

```diff go
func someTask() (string, error) {
    ret, err := myTask()
+    if err != nil {
+        return ret, err
+    }
-    return ret, err
+    return ret, nil
}
```

interfaceに値を渡す時は、typed-nilに注意しましょう。

関数呼び出しの引数や返り値には`error` interfaceのみを使うようにし、自ら定義したerror typeはそのsurfaceに一切出現させないほうが良いです。
(ただし型そのものをexportするのはよい)

### Advanced: interface { Is(error) bool }を実装する

必要になることはめったにないと思いますが、`interface { Is(error) bool }`を`error` typeに実装すると[errors.Is]とともに用いられるときに挙動がカスタマイズできるため、便利な場面があります。

[errors.Is]はそのdoc commentより第一引数が`interface { Is(error) bool }`を実装するとき、そちらの実装も使います。

> An error is considered to match a target if it is equal to that target or if > it implements a method Is(error) bool such that Is(target) returns true.
>
> An error type might provide an Is method so it can be treated as equivalent to > an existing error. For example, if MyError defines
>
> func (m MyError) Is(target error) bool { return target == fs.ErrExist }
> then Is(MyError{}, fs.ErrExist) returns true. See syscall.Errno.Is for an example in the standard library. An Is method should only shallowly compare err and the target and not call Unwrap on either.

あまりはっきり書かれていない気がしますが、[errors.Is]は`err`を順次unwrapしながら`unwrapped == target`という比較を繰り返す挙動になっています。そのため、`target`(第二引数)のdynamic typeがuncomparableであるとき基本的に何もしません。(逆に言ってuncomparable同士の比較でpanicを起こすこともありません。)
そこで、`err`(第一引数)かそれをunwrapして得られたerrorが`interface { Is(error) bool }`を実装するときにはそちらの実装による比較も行うようになっています。単なる`err1 == err2`を超えた挙動を実現できるため、例えば複数のerror値に対してマッチするようにするなどカスタマイズに幅があります。

実装サンプルを以下に挙げます。
このサンプルでは、bit flagで表現されるerrorを複数bitwise-ORすることで持つことができる`error` typeを定義し、これに対して特定のbit flagを持っているかを[errors.Is]で検査できるようにします。

[snippet](https://github.com/ngicks/go-example-basics-revisited/blob/main/error-handling/implement-is-as-format/main.go)

```go
type ErrKind int

func (e ErrKind) Error() string {
    var s strings.Builder
    s.WriteString("kind ")
    var count int
    for i := 0; i < 32; i++ {
        if e&(1<<i) > 0 {
            if count > 0 {
                s.WriteByte('&')
            }
            s.WriteString(strconv.Itoa(i + 1))
            count++
        }
    }
    return s.String()
}

const (
    ErrKind1 = ErrKind(1 << iota)
    ErrKind2
)

type errBare struct {
    Msg  string
    Kind ErrKind
}

func (e *errBare) Error() string {
    return e.Msg
}

type errIs struct {
    errBare
}

func (e *errIs) Is(err error) bool {
    if k, ok := err.(ErrKind); ok {
        return e.Kind&k > 0
    }
    return false
}
```

当然、`Is`を実装しない`errBare`では、`ErrKind1`との比較でtrueが返ってくることはありませんが、

```go
err1 := &errBare{Msg: "is", Kind: ErrKind1}
fmt.Printf("is = %t\n", errors.Is(err1, ErrKind1))                        // is = false
fmt.Printf("is = %t\n", errors.Is(fmt.Errorf("wrapped; %w", err1), err1)) // is = true
```

`errIs`ではこの`Is`の実装が利用されるため、trueとなります。

```go
err2 := &errIs{errBare{Msg: "is", Kind: ErrKind1 | ErrKind2}}
fmt.Printf("is = %t\n", errors.Is(err2, ErrKind1))                        // is = true
fmt.Printf("is = %t\n", errors.Is(err2, ErrKind2))                        // is = true
fmt.Printf("is = %t\n", errors.Is(fmt.Errorf("wrapped: %w", err2), err2)) // is = true
```

`Is`が実装されていても、`err == target`の比較はそれはそれとして[errors.Is]が行うため、`Is`の実装そのものが`receiver == input`を判定する必要はありません。したほうが良いとは思います。

(つまりこうしたほうが基本的にはいいはず)

```diff go
func (e *errIs) Is(err error) bool {
    if k, ok := err.(ErrKind); ok {
        return e.Kind == k
    }
-    return false
+    return e == err
}
```

(method receiverがpointerであるとき、必ずcomparableなのでuncomparable同士の比較によるrun-time panicを恐れる必要はない)

### interface { Is(error) bool }実装の典型: syscall.Errno

そのほかの典型例としては[syscall.Errno]があります。[errors.Is]のdoc commentでも触れられていますね。

`os`パッケージの各種関数が返すerrorで`errors.Is(err, fs.ErrNotExist)`が機能するのは`Errno`に`Is`が実装されているからです。

(unix版)
https://github.com/golang/go/blob/master/src/syscall/syscall_unix.go#L120-L132

(windows版)
https://github.com/golang/go/blob/go1.22.3/src/syscall/syscall_windows.go#L168-L194

`oserror`とは何でしょうか？

https://github.com/golang/go/blob/master/src/internal/oserror/errors.go#L5-L18

これらの値の参照をたどると以下で出てきます。

https://github.com/golang/go/blob/go1.22.3/src/io/fs/fs.go#L139-L154

わざわざ一旦関数を経由して値を定義しているのは`oserror`という文字列をgo docに乗せたくないからなのかなあと思います。[io/fsのvariable section](https://pkg.go.dev/io/fs@go1.23.4#pkg-variables)を見るとわかる通り、constやvariableセクションはソースコードがそのまま乗ってしまうのです。

`os`パッケージで使うと書いているのはどういうことかというと

https://github.com/golang/go/blob/go1.22.3/src/os/error.go#L17-L24

という感じでプラットフォーム間/API間でerrorを同一扱いするためにこういうことをしているようです。

### Advanced: interface { As(any) bool }を実装する

`Is`に更に輪をかけて必要になる場面が少ないと思いますが、`interface { As(any) bool }`を`error` typeに実装すると[errors.As]とともに用いられるときに挙動がカスタマイズできるため、便利な場面があります。

[errors.As]はそのdoc commentより第一引数が`interface { As(any) bool }`を実装するとき、そちらの実装も使います。

> An error matches target if the error's concrete value is assignable to the value pointed to by target, or if the error has a method As(any) bool such that As(target) returns true. In the latter case, the As method is responsible for setting target.
>
> An error type might provide an As method so it can be treated as if it were a different error type.
>
> As panics if target is not a non-nil pointer to either a type that implements error, or to any interface type.

こちらも同じくあまりはっきり書かれていない気がしますが、[errors.As]は単に`err`を順次unwrapながら`target`に対して`unwrapped`がassign可能かを判定し、可能ならassignしてtrueを返す挙動になっています。
こちらも同じく、`err`(第一引数)かそれをunwrapして得られたerrorが`interface { As(any) bool }`を実装するときにはそれを使用するようになっています。単なる`*target == err`を超えた挙動を実現できるため、変換しながら代入とかいろいろできるようになっています。

サンプルを以下に挙げます。
このサンプルでは、`Is`実装用いたのと同じbit flagで表現されるerrorを複数bitwise-ORすることで持つことができる`error` typeを定義し、bit flag部分だけを取り出せるように`As`を実装します。

[snippet](https://github.com/ngicks/go-example-basics-revisited/blob/main/error-handling/implement-is-as-format/main.go)

```go
type ErrKind int

func (e ErrKind) Error() string {
    var s strings.Builder
    s.WriteString("kind ")
    var count int
    for i := 0; i < 32; i++ {
        if e&(1<<i) > 0 {
            if count > 0 {
                s.WriteByte('&')
            }
            s.WriteString(strconv.Itoa(i + 1))
            count++
        }
    }
    return s.String()
}

const (
    ErrKind1 = ErrKind(1 << iota)
    ErrKind2
)

type errBare struct {
    Msg  string
    Kind ErrKind
}

func (e *errBare) Error() string {
    return e.Msg
}

type errAs struct {
    errBare
}

func (e *errAs) As(tgt any) bool {
    if k, ok := tgt.(*ErrKind); ok {
        *k = e.Kind
        return true
    }
    return false
}
```

`As`の引数に渡されるのは、[errors.As]の第二引数そのままなので、場合によって`**T`かもしれませんし, `*T`, `**T`どちらも渡されるというのもありうるので注意しましょう。

当然`As`を実装しない`errBare`に対して[errors.As]を実行してもtrueは帰ってきませんが、

```go
err1 := &errBare{Msg: "is", Kind: ErrKind1}
var kind ErrKind
fmt.Printf("as = %t\n", errors.As(err1, &kind)) // as = false
```

`errAs`では、`As`の実装が利用されるため、trueになり、さらにinterfaceの規約通り、`As`実装が`target`にassignを行うため、渡した`&kind`には値が代入されています。

```go
err2 := &errAs{errBare{Msg: "is", Kind: ErrKind1}}
kind = 0
fmt.Printf("as = %t, kind = %s\n", errors.As(err2, &kind), kind) // as = true, kind = kind 1
kind = 0
fmt.Printf("as = %t, kind = %s\n", errors.As(fmt.Errorf("wrapped: %w", err2), &kind), kind) // as = true, kind = kind 1

err2 = &errAs{errBare{Msg: "is", Kind: ErrKind1 | ErrKind2}}
kind = 0
fmt.Printf("as = %t, kind = %s\n", errors.As(err2, &kind), kind) // as = true, kind = kind 1&2
```

### interface { As(any) bool }の唯一の実装例: net/http.http2StreamError

std内で適当に検索をかけたところ、`interface { As(any) bool }`を実装するのは以下の`http2StreamError`だけのようです。

https://github.com/golang/go/blob/go1.22.3/src/net/http/h2_bundle.go#L1233-L1237

`As`の実装は以下です。

https://github.com/golang/go/blob/go1.22.3/src/net/http/h2_error.go#L13-L37

[reflect](https://pkg.go.dev/reflect@go1.22.3)を使って、`target`と自身がお互いにstructで、各fieldの名前が同一で型が`Convertible`かどうかを判定し、同じであるときに各fieldに値を`Set`しています。

すべてのfieldが代入可能かまず最初にチェックを行っているのは中途半端に代入を行って失敗を返さないようにするためでしょうね。読者の皆さんも似たような機能を実装する際には同じように中途半端な値を入れない考慮を行うとよりよいでしょう。

`ConvertibleTo`を使っていることのポイントは、ほぼ同一構造で変換可能な型で構成されるstructに代入できるようにすることでしょう。

つまり、`As`は以下のような別のstructに対しても`true`を返します。
(実際に代入可能であることはplaygroundで確認してください。)

[playground](https://go.dev/play/p/g1D9qeHGal9)

```go
type fakeHttp2Err struct {
    StreamID int // `uint32`や`http2ErrCode`の代わりに`int`を使っていることがポイントです。
    Code     int
    Cause    error // optional additional detail
}

func (e fakeHttp2Err) Error() string {
    return "fake"
}
```

ただ`func (e *fakeHttp2Err) Error() string`としまうと、`errors.As`には`**fakeHttp2Err`を渡すことになりますが、`http2StreamError`の`As`実装はそのケースを無視しているので`**fakeHttp2Err`に対しては常に`false`を返してしまいますね。

`*T`, `**T`両方に対応するためには以下のように変更します。

[playground](https://go.dev/play/p/W40XS_76S0w)

```diff
func (e http2StreamError) As(target any) bool {
-    dst := reflect.ValueOf(target).Elem()
+    dstOrig := reflect.ValueOf(target).Elem()
+    dst := dstOrig // T or *T
    dstType := dst.Type()
+
+    needSet := false
+    if dstType.Kind() == reflect.Pointer {
+        // *T
+        dstType = dstType.Elem() // T
+        if dst.IsNil() {
+            needSet = true // needs allocation but deferred until needed.
+        } else {
+            dst = dst.Elem() // T, addressable
+        }
+    }
+
    if dstType.Kind() != reflect.Struct {
        return false
    }
    src := reflect.ValueOf(e)
    srcType := src.Type()
    numField := srcType.NumField()
    if dstType.NumField() != numField {
        return false
    }
    for i := 0; i < numField; i++ {
        sf := srcType.Field(i)
        df := dstType.Field(i)
        if sf.Name != df.Name || !sf.Type.ConvertibleTo(df.Type) {
            return false
        }
    }
+
+    if needSet {
+        // newly allocated value, mutating dst is not gonna propagate to `target`.
+        dst = reflect.New(dstType).Elem()
+    }
+
    for i := 0; i < numField; i++ {
        df := dst.Field(i)
        df.Set(src.Field(i).Convert(df.Type()))
    }

+    if needSet {
+        dstOrig.Set(dst.Addr())
+    }
    return true
}
```

任意の同一構造のstructを受け付る気があるならこういう感じで`*U`/`**U`両対応が必要です。
読者がただちにこういった実装が必要になるかはわかりませんが、考慮事項としてこういうものがあると気に留めておくとよいかもしれません。

### Advanced: interface { Format(fmt.State, rune) }を実装する

[fmt.Formatter](https://pkg.go.dev/fmt@go1.23.4#Formatter)を実装すると`fmt.*printf`でどのようにprintされるかの挙動を完全にコントロールできるようになります。

```go
// Formatter is implemented by any value that has a Format method.
// The implementation controls how [State] and rune are interpreted,
// and may call [Sprint] or [Fprint](f) etc. to generate its output.
type Formatter interface {
    Format(f State, verb rune)
}
```

実装はいい例が思いつきませんでしたが、`%+v`の時は`Error` methodを無視してすべてのfieldを表示するように変更してみます。

[snippet](https://github.com/ngicks/go-example-basics-revisited/blob/main/error-handling/implement-is-as-format/main.go)

```go
type errFormat struct {
    errBare
}

func (e *errFormat) Format(state fmt.State, verb rune) {
    if verb == 'v' {
        if state.Flag('+') {
            _, _ = fmt.Fprintf(state, "msg = %s, kind = %s", e.Msg, e.Kind)
            return
        }
    }
    // plain does not inherit method from errFormat.
    type plain errFormat
    _, _ = fmt.Fprintf(state, fmt.FormatString(state, verb), (*plain)(e))
}
```

[fmt.State](https://pkg.go.dev/fmt@go1.23.4#State)自体が[io.Writer]で、結果をここに`Write`するのが`Format` methodの規約となります。

[fmt.FormatString](https://pkg.go.dev/fmt@go1.23.4#FormatString)で`state`と`verb`から`"%#v"`のようなformat stringを再建できます。
`%+v`以外のときの挙動を一切変更しないために、それ以外の場合は再建したformat stringで`Fprintf`します。
`type plain errFormat`とするとこで`Format` methodのないが構造が同じ型を定義します。そうしないと`Format` methodが再帰的に呼びされてstack overflowが起きます。
この例では`errBase`がembedされていることで`Error` methodは継承されますので都合よく動作します。基本的にはこういったdelegationを全く行わないでこの`Format` methodのなかですべてのパターンをハンドルするか、でなければ`Error`や`String`, `GoString`などは継承するようにしたほうが良いです。

全フォーマットを網羅してprintして`errBase`と`errFormat`の結果を比較します。

`fmt`の実装を洗って全パターンを網羅しました。以下です。

```go
bare := &errBare{Msg: "is", Kind: ErrKind1}
for _, v := range "vTtbcdoOqxXUeEfFgGsqp" {
    verb := string([]rune{v})
    fmt.Printf("verb %%%s, bare = %"+verb+", format = %"+verb+"\n", verb, bare, &errFormat{*bare})
    for _, f := range " +-#0" {
        verb := string([]rune{f, v})
        fmt.Printf("verb %%%s, bare = %"+verb+", format = %"+verb+"\n", verb, bare, &errFormat{*bare})
    }
}
```

多くなるので`v`の各パターンのみ結果を表示します

```
verb %v, bare = is, format = is
verb % v, bare = is, format = is
verb %+v, bare = is, format = msg = is, kind = kind 1
verb %-v, bare = is, format = is
verb %#v, bare = &main.errBare{Msg:"is", Kind:1}, format = &main.plain{errBare:main.errBare{Msg:"is", Kind:1}}
```

`%+v`のみ挙動をが変更できています。`%#v`も変わっていますが、これは型を定義した副作用です。

## panic-recover

`panic`と`recover`を活用して一気に関数を抜けるというerror handlingも存在しえます。

`panic`とは何かという話から`panic-recover`で一気に処理を中断する方法の実装例を紹介します。

### panic

組み込み関数の[panic]を呼び出したり、sliceやarrayのout of index access, nil pointer derefなどが行われると`panic`が起きます。

> quoted from https://go.dev/ref/spec#Run_time_panics
>
> Execution errors such as attempting to index an array out of bounds trigger a _run-time panic_ equivalent to a call of the built-in function panic with a value of the implementation-defined interface type runtime.Error.

> quoted from https://go.dev/ref/spec#Handling_panics
>
> Two built-in functions, panic and recover, assist in reporting and handling run-time panics and program-defined error conditions.

[panic](https://pkg.go.dev/builtin@go1.22.3#panic)が起きるとその行から通常のプログラム実行が止まり、
`defer`に登録された関数があれば登録されたのと逆順で実行していきます。

ある関数`F`のなかで`defer`された関数が`recover`を呼ぶと、`F`を呼び出す`G`から通常の関数実行の順序に戻ります。

```go
func() {
    defer func() {
        if rec := recover(); rec != nil {
            fmt.Printf("panicked = %#v\n", rec) // panicked = runtime.boundsError{x:6, y:4, signed:true, code:0x0}
        }
    }()
    var a [4]int
    var b []int = a[:]
    _ = b[6] // index out of range!
    fmt.Printf("this line can not be reached\n")
}()
fmt.Printf("back to normal execution order.\n")
```

`recover`されずに`panic`がgoroutineを終了させるとプロセス全体がエラー終了します。

```go
// つまりこうすれば確実にプロセスを殺せます
go func() { panic("die!") }()
```

`try-catch-finally`で例えると、`defer`が`finally`、`recover`が`catch`だといえます。

と、こう書くと`panic`は基本的に`recover`されない究極的なエラー終了手段かのように聞こえるかもしれません、実際上は`*http.Server`が`recover`してしまうので逆に**基本的に`recover`されるものと思ったほうが良い**です。もちろん100%挙動をコントロールできるシチュエーションでは別ですが、例えば[*http.Server]を使わずにhttp serverを実装することは少ないと思いますし、`Go`を書いていてhttp serverを実装しないことも結構珍しいと思います。

> https://pkg.go.dev/net/http@go1.22.4#Handler
>
> If ServeHTTP panics, the server (the caller of ServeHTTP) assumes that the effect of the panic was isolated to the active request. It recovers the panic, logs a stack trace to the server error log...

https://github.com/golang/go/blob/go1.23.4/src/net/http/server.go#L1937-L1950

`panic`時にプロセスが強制終了されてほしいとき、[*http.Server]を使うプログラムの場合、ユーザーが自ら特別な措置を実装する必要があります。

つまりこころもちとしては

- `panic`を意図的にする場合は
  - 意図的にrecoverし、意図しないpanicはre-panicする
  - もしくは、プロセスは異常終了すべき
- 一方で`panic`はコントロールしていないエリアで勝手に`recover`されるのは当然起こる

と思っているといいという感じです。

前述通り、その`goroutine`で`panic`した時は通常の関数実行順序をやめ、`defer`に登録されている関数を登録の逆順で実行してきます。
つまりリソース解放処理は必ず`defer`でしなければ、コントロールしていないコードによって`recover`され、静かに不正状態に陥る可能性があります。

- リソース解放処理は必ず`defer`で行おう
  - `sync.Mutex`の`Unlock`
  - `*os.File`の`Close`
  - 何かのカウントをincrementしたときのdecrement
    - `sync.WaitGroup`の`Done`

`Go`はすべての`goroutine`が眠りにつくとdeadlockであるとして、プロセスを終了してくれるガードが入っていますが、[*http.Server]がコネクション待ちに入ってる状態はdeadlockに見えないはずなので、エラーしないのに動かない状態になるはずです。

もしくはrecoverされたくないなら`go panic("panic cause")`とわざとするとよいかもしれません。
ただし、`panic`が`goroutine`を終了させたときの強制終了処理は他の`goroutine`の`defer`を呼び出しませんので、リソース解放処理をあてにしたプログラムが不正な中途状態を書き出すような場合は、それがプロセス終了後に観測されてしまいます。

できれば、あらゆる`goroutine`で起きた`panic`は`recover`で拾って`panic`しなおして、拾って・・・というのを繰り返して`main goroutine`まで伝搬させたほうがいいでしょう。
`main goroutine`で`panic`時のエラーログなどを書きだしてすべての`goroutine`を終了させるなどすると穏当にプロセスを終了させられます。
どのように`panic`を伝搬させるかは`Go`が勝手に判断できるタイプの仕事ではないのでユーザーが意図的に行う必要があります。

とはいえ、電源断(power outage / power failure, 停電など)の恐れがあるようなシステムではリソース解放が漏れなくても電断でおかしな状態になりうるので、
どちらにせよ回帰する方法がプロセス起動時に呼ばれなければなりません。

### panic-recoverの実装例

`panic`を使うと`defer`された関数以外を無視して一気に脱出ができるのでもちろんtry-catch的に使うこともできます。

https://go.dev/doc/effective_go#recover

`Effective Go`に`try-catch`的`panic-recover`の使い方が述べられています。

ポイントは以下になります

- 特定の値/型で`panic`することで処理をabortする
- パッケージをまたいでpanicが伝搬しないように公開関数/メソッドでは必ずrecoverする
- 意図した型以外でのpanicははre-panicする

前述の[*http.Server]は`http.ErrAbortHandler`をsentinel valueとして、handlerをabortするための`panic`ができるようになっています。

#### iterator-collectorを抜ける

errorによる中断をサポートしないiteratorのcollectorを中断するのにpanic-recoverは便利です

[Go 1.23]から`for-range-func`などと呼ばれる、`for-range`が特定のシグネチャを満たす関数を処理する仕様が追加されました。その関数のことをカジュアルに`iterator`と呼びます。
これによってerror発生時の処理終了をサポートしないが長くかかる関数というのが現れやすくなったと筆者は予測しており、panic-recoverも使いどころが増えたのではないかと思います。

例えば、下記の`reduce`があるとします

[snippet](https://github.com/ngicks/go-example-basics-revisited/blob/main/error-handling/panic-recover/main.go)

```go
func reduce[V, Accum any](
    reducer func(accum Accum, next V) Accum,
    initial Accum,
    seq iter.Seq[V],
) Accum {
    accum := initial
    for v := range seq {
        accum = reducer(accum, v)
    }
    return accum
}
```

これはシグネチャ上、`reducer`の失敗時に早期に中断できるようになっていません。
これをpanic-recoverによって中断可能にします。

```go
func reduceUsingFailableWork[V, Accum any](
    work func(accum Accum, next V) (Accum, error),
    initial Accum,
    seq iter.Seq[V],
) (a Accum, err error) {
    type wrapErr struct {
        err error
    }

    defer func() {
        rec := recover()
        if rec == nil {
            return
        }
        w, ok := rec.(wrapErr)
        if !ok {
            panic(rec)
        }
        err = w.err
    }()

    a = reduce[V, Accum](
        func(accum Accum, next V) Accum {
            var err error
            accum, err = work(accum, next)
            if err != nil {
                panic(wrapErr{err})
            }
            return accum
        },
        initial,
        seq,
    )
    return a, nil
}
```

ポイントは

- 独自の型を定義し、error時にそれで[panic]を呼び出す
- `defer`で`recover`を実行
- そもそも`panic`してないときにもdeferは実行されるため`recover`の返り値がnilならそのままreturn
  - どこかのバージョンから`panic(nil)`すると`recover`できなくなっているので`rec`がnilなら`panic`してないです。
- 前述の既知の型でないときre-panic
- 関数の返り値を名前付きにしておき、wrapされた`error`の値をそれに代入します。

実行してみると以下のような感じ。きちんと中断できています。

```go
sampleErr := errors.New("sample")
fmt.Println(
    reduceUsingFailableWork(
        func(accum int, next int) (int, error) {
            fmt.Printf("next = %d\n", next)
            if accum > 50 {
                return accum, sampleErr
            }
            return accum + next, nil
        },
        10,
        slices.Values([]int{5, 7, 1, 2}),
    ),
)
/*
    next = 5
    next = 7
    next = 1
    next = 2
    25 <nil>
*/
fmt.Println(
    reduceUsingFailableWork(
        func(accum int, next int) (int, error) {
            fmt.Printf("next = %d\n", next)
            if accum > 50 {
                return accum, sampleErr
            }
            return accum + next, nil
        },
        40,
        slices.Values([]int{5, 7, 1, 2}),
    ),
)
/*
    next = 5
    next = 7
    next = 1
    0 sample
*/
```

他にもやりようはいくらでもあるのでpanic-recoverでこれをクリするとは限らないですが、覚えておいて損はないはずですよ。

#### std

実際std内でも上記のように`panic`が`throw`的に使われています。

`encoding/json`内では以下のようにpanicを`throw`的に使っています。
少なくとも https://codereview.appspot.com/953041 のころからそうなので2010-04-21からずっとこんな感じでした

https://github.com/golang/go/blob/go1.23.4/src/encoding/json/encode.go#L302-L305

https://github.com/golang/go/blob/go1.23.4/src/encoding/json/encode.go#L283-L300

`any`同士の比較でpanicするのをrecoverしている部分もあります。

https://github.com/golang/go/blob/go1.23.4/src/crypto/hmac/hmac.go#L141-L149

## stacktrace

`Go`のstd libraryはstacktraceの付いたerrorを返してくることがないため、慣習的にerrorにはstacktraceがついていないのが普通です。

### stacktraceはついていない

stdのerrorが全般的にstacktrace情報を含んでくれればと思うのですが、

- 現状のerrorがstacktraceを含まず、
- `strings.Contains(err.Error(), "...")`でerrorの判別をするコードが存在し、
- 同様に`fmt.Sprintf("...", err)`で文字列をくみ上げるコードも存在する

ので、暗黙的に既存コードから帰ってくるerrorにstacktraceを持たせる変更は破壊的変更となります。

[io.EOF]のようにsentinel valueとしてふるまうerrorが普通にあり得てしまうため、そういったものに勝手にstacktraceをつけてしまうとパフォーマンスに影響することも考えられます。

そのため、既存の挙動を全く破壊せずにstacktraceを取り出す方法が実装されない限り、入れる意味がないのでstdのコードがstacktraceを含むerrorを返してくることはないでしょう。

自分向けのざっくり作ったツールほどerrorのメッセージを丁寧にラップしないので、そういうときこそstacktraceが欲しいですね。
errorメッセージをさぼって、後になってどこで起きたのかわからないerrorが吐かれて慌ててerrorのラップを整備しだすんですよね(n敗)。

いつかstdでもstacktraceがつくかもしれませんがいつになるのかわからないので必要であればライブラリを利用したほうがよろしいかと思います。

参考: [Goのerrorがスタックトレースを含まない理由](https://methane.hatenablog.jp/entry/2024/04/02/Go%E3%81%AEerror%E3%81%8C%E3%82%B9%E3%82%BF%E3%83%83%E3%82%AF%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B9%E3%82%92%E5%90%AB%E3%81%BE%E3%81%AA%E3%81%84%E7%90%86%E7%94%B1)

### ライブラリで付ける

以下のライブラリが有名です

- https://github.com/morikuni/failure
- https://github.com/cockroachdb/errors

どちらも筆者は使ったことがないので何ともです。

### 自分でつける

サクっと実装するとこんなものっていう感じのexampleをつけておきます。

`log/slog`パッケージの[この辺](https://github.com/golang/go/blob/go1.23.4/src/log/slog/logger.go#L84-L104)とか[この辺](https://github.com/golang/go/blob/go1.23.4/src/log/slog/record.go#L214-L226)を参考に。

[snippet](https://github.com/ngicks/go-example-basics-revisited/blob/main/error-handling/with-stack/main.go)

```go
const maxDepth = 100

type withStack struct {
    err error
    pc  []uintptr
}

func (e *withStack) Error() string {
    return e.err.Error()
}

func (e *withStack) Unwrap() error {
    return e.err
}

func wrapStack(err error, override bool) error {
    if !override {
        var ws *withStack
        if errors.As(err, &ws) {
            // already wrapped
            return err
        }
    }

    var pc [maxDepth]uintptr
    // skip runtime.Callers, WithStack|WithStackOverride, wrapStack
    n := runtime.Callers(3, pc[:])
    return &withStack{
        err: err,
        pc:  pc[:n],
    }
}

func WithStack(err error) error {
    return wrapStack(err, false)
}

func WithStackOverride(err error) error {
    return wrapStack(err, true)
}

func Frames(err error) iter.Seq[runtime.Frame] {
    return func(yield func(runtime.Frame) bool) {
        var ws *withStack
        if !errors.As(err, &ws) {
            return
        }

        frames := runtime.CallersFrames(ws.pc)
        for {
            f, ok := frames.Next()
            if !ok {
                return
            }
            if !yield(f) {
                return
            }
        }
    }
}

func PrintStack(w io.Writer, err error) error {
    for f := range Frames(err) {
        _, err := fmt.Fprintf(w, "%s(%s:%d)\n", f.Function, f.File, f.Line)
        if err != nil {
            return err
        }
    }
    return nil
}
```

`WithStack`を適当に深いところで呼び出して、`PrintStack`を実行すると以下のようになります。

```go
func example(err error) error {
    return deep(err)
}

func deep(err error) error {
    return calling(err)
}

func calling(err error) error {
    return frames(err)
}

func frames(err error) error {
    return WithStack(err)
}

func main() {
    sample := errors.New("sample")

    wrapped := example(sample)

    fmt.Printf("%v\n", wrapped) // sample
    err := PrintStack(os.Stdout, wrapped)
    if err != nil {
        panic(err)
    }
    /*
        main.frames(github.com/ngicks/go-example-basics-revisited/error-handling/with-stack/main.go:96)
        main.calling(github.com/ngicks/go-example-basics-revisited/error-handling/with-stack/main.go:92)
        main.deep(github.com/ngicks/go-example-basics-revisited/error-handling/with-stack/main.go:88)
        main.example(github.com/ngicks/go-example-basics-revisited/error-handling/with-stack/main.go:84)
        main.main(github.com/ngicks/go-example-basics-revisited/error-handling/with-stack/main.go:102)
        runtime.main(runtime/proc.go:272)
    */
}
```

`runtime.Frame.File`がpackage pathになっていますがこれは`go run -trimpath ./with-stack/`で実行しているためです。`-trimpath`オプションがなければソースコードの表示はローカルストレージ上のフルパスになりますので注意してください。

### panicのstacktraceをログに残す

`panic`のstacktraceをprintしてログに残したいことはあると思います。
というのが、nil pointer dereferenceがふいに起きたとき、どこでどう起きたのかログにだせないと見当がつかなくて困ります(1敗)。

- 1. panicを拾わず(=`recover`せず)プロセスを落とすことでGoにstacktraceを吐かせる
- 2. `recover`することで、panic時のstacktraceを任意の出力先に出す
- 3. 別`goroutine`で起きたpanicのstacktraceを順次main goroutineに伝搬する

についてそれぞれ述べます。

#### 1. panicを拾わず(=`recover`せず)プロセスを落とすことでGoにstacktraceを吐かせる

単に`panic`のstacktraceを表示するだけなら以下のようにします。

[playground](https://go.dev/play/p/-hstc20PnGF)

```go
package main

func main() {
    aaa()
}

func aaa() {
    bbb()
}

func bbb() {
    ccc()
}

func ccc() {
    panic("hey")
}

/*
panic: hey

goroutine 1 [running]:
main.ccc(...)
    /tmp/sandbox1615949582/prog.go:16
main.bbb(...)
    /tmp/sandbox1615949582/prog.go:12
main.aaa(...)
    /tmp/sandbox1615949582/prog.go:8
main.main()
    /tmp/sandbox1615949582/prog.go:4 +0x25
*/
```

`panic`が`recover`されなかった場合stderrに上記のフォーマットでstacktraceが表示されてプロセスが終了します。

#### 2. `recover`することで、panic時のstacktraceを任意の出力先に出す

任意のフォーマットや出力先を選択する、例えば[slog.Logger](https://pkg.go.dev/log/slog@go1.23.4#Logger)に出力したいときなどは`recover`を呼んだ関数の中で前述の[runtime.Callers](https://pkg.go.dev/runtime@go1.23.4#Callers)/[runtime.CallersFrames](https://pkg.go.dev/runtime@go1.23.4#CallersFrames)を用いればよいです。
この時点では`SP`(Stack Pointer)が巻き戻されていないので、stackを表示するとpanicを呼び出したところからのstackが表示されます。詳しいことはよくわかっていないので`recover`を呼び出したコードをコンパイルして`objdump -d`したり、[実装](https://github.com/golang/go/blob/go1.23.4/src/runtime/panic.go#L722-L806)などを参照してほしいです。`panic`の実装は`runtime.gopanic`の呼び出しに書き換えれるのが見て分かるのでstacktraceの先端が`gopanic`である理由がよくわかると思います。

```go
func main() {
    defer func() {
        rec := recover()
        if rec == nil {
            return
        }
        pc := make([]uintptr, 100)
        // skip runtime.Callers, this closure, runtime.gopanic
        n := runtime.Callers(3, pc)
        pc = pc[:n]

        fmt.Printf("panicked: %v\n", rec)
        frames := runtime.CallersFrames(pc)
        for {
            f, ok := frames.Next()
            if !ok {
                break
            }
            fmt.Printf("    %s(%s:%d)\n", f.Function, f.File, f.Line)
        }
    }()
    // work...
}
```

#### 3. 別`goroutine`で起きたpanicのstacktraceを順次main goroutineに伝搬する

ある`goroutine`で起きた`panic`を`recover`して他の`goroutine`に伝搬させるのはよくあります。(e.g. [singleflight.(\*Group).Do](https://pkg.go.dev/golang.org/x/sync/singleflight#Group.Do)・・・特にdoc commentでは触れられていないがpanicが伝搬される)

`panic`が`recover`されずに`go`キーワードをつけて呼び出された関数を終了させるとプロセス全体が強制終了します。このとき他の`goroutine`の`defer`が実行されません。
ライブラリのつくりのよっては新しく`goroutine`を作って関数を動作させるかもしれませんし、複数の`goroutine`から同じ関数の呼び出し結果を用いるかもしれません(前述の`singleflight`がまさにこれです)。これらの処理をまたぐときに`panic`が問答無用の強制終了にしかならないとなると、例えば前述したような`panic-recover`で一気に脱出をする手法が不可能になったり、`main goroutine`で`panic`のログをとっている場合などと相性が非常に悪くなってしまいます。
呼び出し側に`panic`の取り扱いをゆだねるには、`panic`の`recover`と伝搬は必須となります。

以下ではある`goroutine`で起きた`panic`を順次伝搬し、`main goroutine`でログに書き出すexampleを示します。
exampleでは省略されていますが、`main goroutine`で`panic`が起きたらすべての処理がキャンセルされる(i.e. `context.Context`をcancelする、など)ようにすれば、穏当な終了処理を適切なerrorログとともに行うことができます。

前述通り、`panic`時のstacktraceは`recover`した関数内で取得できます。
こうして各`goroutine`で取得したstacktraceを`panic()`の引数に渡す変数に収めれば、すべてのstacktraceを`main goroutine`まで伝搬することができます。

このスニペットの中で使っている`serr`パッケージは[stacktrace/自分でつける](#自分でつける)で載せているスニペットもうちょっと凝ってライブラリとして[実装](https://github.com/ngicks/go-common/blob/serr/v0.6.0/serr/withstack.go)したものです。

[snippet](https://github.com/ngicks/go-example-basics-revisited/blob/main/error-handling/log-stacktrace/main.go)

```go
package main

import (
    "context"
    "fmt"
    "sync"

    "github.com/ngicks/go-common/serr"
)

//go:noinline
func example(ctx context.Context) {
    deep(ctx)
}

func deep(ctx context.Context) {
    calling(ctx)
}

func calling(ctx context.Context) {
    frames(ctx)
}

func frames(ctx context.Context) {
    var (
        panicVal  any
        panicOnce sync.Once
        wg        sync.WaitGroup
    )
    ctx, cancel := context.WithCancelCause(ctx)
    defer cancel(nil)
    wg.Add(1)
    go func() {
        defer wg.Done()
        defer func() {
            rec := recover()
            if rec == nil {
                return
            }
            panicOnce.Do(func() {
                // In case there's many goroutines running and
                // you are going to capture only first panic value recovered.
                panicVal = serr.WithStack(fmt.Errorf("panicked: %v", rec))
            })
            cancel(panicVal.(error))
        }()
        example2(ctx)
    }()
    wg.Wait()
    if panicVal != nil {
        panic(panicVal)
    }
}

//go:noinline
func example2(ctx context.Context) {
    deep2(ctx)
}

func deep2(ctx context.Context) {
    calling2(ctx)
}

func calling2(ctx context.Context) {
    frames2(ctx)
}

func frames2(_ context.Context) {
    s := make([]int, 2)
    _ = s[4]
}

func main() {
    defer func() {
        rec := recover()
        if rec == nil {
            return
        }
        // skip runtime.Callers, inner func, WithStackOpt.
        err := serr.WithStackOpt(rec.(error), &serr.WrapStackOpt{Override: true, Skip: 3})
        fmt.Printf("panicked: %v\n", rec)
        var i int
        for seq := range serr.DeepFrames(err) {
            if i > 0 {
                fmt.Printf("caused by\n")
            }
            i++
            for f := range seq {
                fmt.Printf("    %s(%s:%d)\n", f.Function, f.File, f.Line)
            }
        }
    }()
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel() // canaled after occurrence of panic
    example(ctx)
    //nolint
    // panicked: panicked: runtime error: index out of range [4] with length 2
    //     main.main.func1(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:80)
    //     runtime.gopanic(runtime/panic.go:785)
    //     main.frames(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:51)
    //     main.calling(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:21)
    //     main.deep(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:17)
    //     main.example(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:13)
    //     main.main(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:93)
    //     runtime.main(runtime/proc.go:272)
    // caused by
    //     main.frames.func1.1.1(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:43)
    //     sync.(*Once).doSlow(sync/once.go:76)
    //     sync.(*Once).Do(sync/once.go:67)
    //     main.frames.func1.1(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:40)
    //     runtime.gopanic(runtime/panic.go:785)
    //     runtime.goPanicIndex(runtime/panic.go:115)
    //     main.frames2(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:70)
    //     main.calling2(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:65)
    //     main.deep2(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:61)
    //     main.example2(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:57)
    //     main.frames.func1(github.com/ngicks/go-example-basics-revisited/error-handling/log-stacktrace/main.go:47)
}
```

`runtime.gopanic`や`sync.(*Once).doSlow`など表示の必要がなさそうな冗長な情報が載っているので実際は必要に合わせて`Skip`の数値は増加させたほうがよいでしょう。
慣れていない人には混乱する情報になるかもしれません。

[serr.DeepFrames](https://pkg.go.dev/github.com/ngicks/go-common/serr@v0.6.0#DeepFrames)で`iter.Seq[iter.Seq[runtime.Frame]]`を得られます。
今回の実装では単にstdoutに書き出していますが、これを適当にmapして`slog.Value`に変換できれば`slog.Logger`でログに残せます。

## 小技集

### []errorをラップして一つにする(簡易)

[errors.Join]で複数errorを1つにまとめられます。返ってくる`error`の`Error` methodはそれぞれのerrorの`Error`を呼び出して結果を`"\n"`で結合します。

それが気に入らない場合は[strings.Repeat](https://pkg.go.dev/strings@go1.23.4#Repeat)で`%w`⁺sepを繰り返し、最後の余計なsepを[strings.TrimSuffix](https://pkg.go.dev/strings@go1.23.4#TrimSuffix)切り落とします。最後に[fmt.Errorf]でerrorをラップします。こうすることで任意のprefix, sepを持ったフォーマットでerrorでprint可能です。

[snippet](https://github.com/ngicks/go-example-basics-revisited/blob/main/error-handling/wrap-error-dynamic/main.go)

```go
var (
    err1 = errors.New("1")
    err2 = errors.New("2")
    err3 = errors.New("3")
)

fmt.Printf("errors.Join: %v\n", errors.Join(err1, err2, err3))
/*
    errors.Join: 1
    2
    3
*/

errs := []any{err1, err2, err3}

const sep = ", "
format := strings.TrimSuffix(strings.Repeat("%w"+sep, len(errs)), sep)
wrapped := fmt.Errorf("foobar error: "+format, errs...)

fmt.Printf("err = %v\n", wrapped) // err = foobar error: 1, 2, 3
```

### []errorをラップして一つにする(型)

基本的には上記の[errors.Join]/[fmt.Errorf]を使うパターンで事足りるんですがラップされた情報の詳細度がたりなくて困ることがあります。

`%w`でエラーをラップした場合は`Unwrap() error`もしくは`Unwrap() []error`を実装した`error`が返されます。
ただし[このあたり](https://github.com/golang/go/blob/go1.23.4/src/fmt/errors.go#L54-L78)を見るとわかる通り、返されたerrorの`Error` methodが返すstringは`%w` verbを`%v`に置き換えて`fmt.Sprintf`で出力したものをキャッシュしておき、それを返す実装となっています。
つまり、返ってきたerrorを`%#v`のようなより詳細な情報を要求するverbでprintしたとしてもこのキャッシュされたstring以上の情報は表示できません。

そこで以下のように型を定義します。

```go
type gathered struct{ errs []error }

func (e *gathered) Unwrap() []error {
    return e.errs
}

func (e *gathered) format(w io.Writer, fmtStr string) {
    for i, err := range e.errs {
        if i > 0 {
            _, _ = w.Write([]byte(`, `))
        }
        _, _ = fmt.Fprintf(w, fmtStr, err)
    }
}

func (e *gathered) Error() string {
    var s strings.Builder
    e.format(&s, "%s")
    return s.String()
}

func (e *gathered) Format(state fmt.State, verb rune) {
    e.format(state, fmt.FormatString(state, verb))
}
```

[Advanced: interface { Format(fmt.State, rune) }を実装する](<#advanced%3A-interface-%7B-format(fmt.state%2C-rune)-%7Dを実装する>)で述べた通り、`interface { Format(fmt.State, rune) }`を実装すると`fmt.*printf`で各verbが何を表示するかをコントロールできます。
この実装では受け取ったflagとverbでラップされた各errorをprintすることで、flagとverbによるprintされる情報の詳細度のコントロールを受け付けられるようになります。

このerror typeは[github.com/ngicks/go-common/serr](https://pkg.go.dev/github.com/ngicks/go-common/serr@v0.6.0)としてパッケージ化してあります。
実装サンプルとしてまとめてあるだけで筆者自身は使っていません。

## おわりに

現状筆者が知らなくて困ったerror周りの話は全部入れれたと思いますがまたなんかあったら追記します。

[Go]: https://go.dev/
[Go 1.23]: https://tip.golang.org/doc/go1.23
[Go 1.26]: https://tip.golang.org/doc/go1.26
[C++]: https://en.wikipedia.org/wiki/C%2B%2B
[Node.js]: https://nodejs.org/en
[TypeScript]: https://www.typescriptlang.org/
[python]: https://www.python.org/
[Rust]: https://www.rust-lang.org
[The Rust Programming Language 日本語]: https://doc.rust-jp.rs/book-ja/
[Visual Studio Code]: https://code.visualstudio.com/
[vscode]: https://code.visualstudio.com/
[git]: https://git-scm.com/
[errors.New]: https://pkg.go.dev/errors@go1.23.4#New
[errors.Is]: https://pkg.go.dev/errors@go1.23.4#Is
[errors.As]: https://pkg.go.dev/errors@go1.23.4#As
[errors.AsType]: https://pkg.go.dev/errors@go1.26rc1#AsType
[errors.Join]: https://pkg.go.dev/errors@go1.23.4#Join
[io.EOF]: https://pkg.go.dev/io@go1.23.4#EOF
[fs.ErrNotExist]: https://pkg.go.dev/io/fs@go1.23.4#ErrNotExist
[io.Reader]: https://pkg.go.dev/io@go1.23.4#Reader
[io.Writer]: https://pkg.go.dev/io@go1.23.4#Writer
[fmt.Errorf]: https://pkg.go.dev/fmt@go1.23.4#Errorf
[type assertion]: https://go.dev/ref/spec#Type_assertions
[type switch]: https://go.dev/ref/spec#Type_switches
[syscall.Errno]: https://pkg.go.dev/syscall@go1.23.4#Errno
[http.Server]: https://pkg.go.dev/net/http@go1.23.4#Server
[*http.Server]: https://pkg.go.dev/net/http@go1.23.4#Server
[panic]: https://pkg.go.dev/builtin@go1.22.3#panic
