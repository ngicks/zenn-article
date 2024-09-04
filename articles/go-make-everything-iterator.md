---
title: "[Go]すべてをiteratorにする"
emoji: "😵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## すべてをiteratorにする

できませんでした。

## iterator

[Go1.23.0](https://tip.golang.org/doc/go1.23)で言語仕様に変更がはいり、for-rangeが以下の三つの関数を受け付けるようになりました。

```
https://go.dev/ref/spec#For_statements

function, 0 values  f  func(func() bool)
function, 1 value   f  func(func(V) bool)              value    v  V
function, 2 values  f  func(func(K, V) bool)           key      k  K            v          V
```

`iter`パッケージで定義される[iter.Seq\[V\]](https://pkg.go.dev/iter@go1.23.0#Seq), [iter.Seq2\[K, V\]](https://pkg.go.dev/iter@go1.23.0#Seq2)がこの関数シグネチャを型とし定義しているため、これを用いると少しわかりやすくなります。

```go
func someIter[K, V any]() iter.Seq2[K, V] {
    return func(yield func(K, V) bool) {
        // ...
    }
}
```

## \[\]V, map\[K\]Vの代わりにiter.Seq\[V\], iter.Seq2\[K, V\]を受けとる

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
    for v := range vs {
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
また、`Go`のrange-over-mapの順序は[言語仕様により未定義](https://go.dev/ref/spec#For_range)であるので、順序が重要なケースでは`iter.Seq2[K, V]`を引数に取ると呼び出し側に順序の制御を渡すことができます。sortのためのcomparer関数を受けていた場合は、受け取る必要がなくなります。

## 既存のデータコンテナをiteratorにする

### container

### Third party: github.com/wk8/go-ordered-map/v2

https://github.com/wk8/go-ordered-map/pull/41

## iteratorにできないやつ

データを受けとった側が、データを生成する側にスキップ、終了その他を指示したい場合、iteratorにできません。
例えば以下です。

- [fs.WalkDir](https://pkg.go.dev/io/fs@go1.23.0#WalkDir)
- [io.Pipe](https://pkg.go.dev/io@go1.23.0#Pipe)

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

`io.Pipe`を丸っと模擬している例なので話のつながりが甘い感じになってしまっていますが、
要は`range-over-func`の構文や`iter.Pull`がiteratorに対してエラーなどを逆向きに伝える手段が構文上ないことを示しています。
