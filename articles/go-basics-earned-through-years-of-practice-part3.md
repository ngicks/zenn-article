---
title: "Goで開発して3年のプラクティスまとめ(3/4): concurrent GO編"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## Goで開発して3年のプラクティスまとめ(3/4): concurrent GO編

yet another入門記事です。

1記事にまとめようとしたら8万文字を超えて怒られたのでいくつかに分割しています。

- [part1 プロジェクトを始めるまで編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part1)
- [part2 cliアプリをつくれるところまで編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part2)
- part3 concurrent GO編: これ
- [part4 HTTP server/logger編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part4)

ご質問やご指摘がございましたらこの記事のコメントでお願いします。
(ほかの媒体やリンク先に書かれた場合、筆者は気付きません)

## Overview

- concurrentなプログラムを書くためのツールキットである`goroutine` / `chan` / `sync`/sub-repositoryの`sync`パッケージを紹介します
  - なぜそれらを使わないといけないか(`data race`)についても述べます
- signal handling

筆者は「並行」と「並列」がどっちがどっちのことを言ってるのかたびたびわからなくなってしまうので全般的に並行のことを*concurrent*, *concurrency*と記述し、並列のことを*parallel*, *parallelism*と記述します。

## 2種の想定読者

記事中では仮想的な「対象読者」と「ベテランとして取り扱われるその他の読者」が想定されています。

### 対象読者

記事中で「対象読者」と呼ばれる人々は以下のことを指します。

- 会社の同僚
- いままで[Go]を使ってこなかった人
- ある程度コンピュータとネットワークとプログラムを理解している人
- [python]とか[Node.js]で開発したことある
- [git]は使える。
- 高校生レベルの英語能力
  - 作ってるところがアメリカ企業なので英語のリンクが全般的に多い

part1以降は[A Tour of Go](https://go.dev/tour/welcome/)を完了していることと、
ポインター、メモリアロケーション、`POSIX`(もしくは`Linux`) syscallなどの基礎的概念がわかっていることが前提条件になっています。

### そのほかの読者

特に断りがない時、他の読者も聴衆として想定されます。

- 筆者と同程度かそれ以上に`Go`に長じており
- POSIX APIや通信プロトコル、他のプログラミング言語でよくやられる方法を知っている

というベテラン的な人々です。

記事中に他にいい方法があったら教えてくださいとか書いてますが、大概はこのベテランな人たちに向けて書いているのであって、対象読者は当面気にしないでください(もちろんあったら教えてください)。

## 対象環境

- 下層の仕組みに言及するとき、特に述べない限り`linux/amd64`を想定します。
- `OS`/`arch`に依存するコードは書きません。

## version

検証は`go 1.22.0`、リンクとして貼るドキュメントは`1.22.3`のものになります。

```
## go version
go version go1.22.0 linux/amd64
```

最近追加されたAPIをちょいちょい使うので`1.22.0`以降でないと動かないコードがたくさんあります。

直近の3～4 minor versionのみサポートするライブラリが多いとして、`Go 1.18`でできなくてそれ以降できるようになったことは、○○以降となるだけ書くようにします。

## サンプルコードのrepository

サンプルコードの一部は下記にアップロードされます。

https://github.com/ngicks/go-basics-example

## concurrent Go

`Go`では`net/http`などでサーバープログラムを書く場合、意識することなく _concurrent_ に処理がなされ、大抵の場合 _concurrent_ な処理は _parallel_ に実行されるため、
multi-threadなプログラムで生じうる諸般の問題が起きないようにうまくプログラムを書く必要があります。

https://go.dev/blog/waza-talk

上記の11年前のRob Pikeの講演によれば _concurrency_ とはタスクをいかに分解するかという表現方法/構造であり、分解されたタスクは _parallel_ 、つまりいくつかを同時に実行できます。
Concurrentに問題を分割しておけば、資源、つまりCPUの個数などが増加したときに全体の処理スピードが(理論上)増加量だけ(CPUコアの数が２倍になれば２倍)速くすることができるということを述べています。
このアイデアはTony Hoare 1978 Communicating Sequential Processes.という論文に書かれていたものであり、
_concurrent_ に物事を解くためのツールキットとして、channelやgoroutineが存在しています。
Rob Pikeは講演上で「この話が沁みたら家に帰ってCSPの論文を読め」と言っていますね。

ところがこのキーコンセプトを全く理解していなくても`net/http`でサーバーを書くと

https://github.com/golang/go/blob/go1.22.3/src/net/http/server.go#L3285

という感じで新しいgoroutineでハンドラが実行されます。

### goroutine

https://go.dev/ref/spec#Go_statements

Specificationによれば`"go"`キーワードの後に関数かメソッド呼び出しのExpressionを書くことで、_an independent concurrent thread of control, goroutine_ でそれが実行されます。

`go`はstatementです。つまり返り値は何もありません。`goroutine`を特定する方法は基本的にありませんし、`goroutine`を指定して終了させるような方法もありません。
これは[pthread_create(3)](https://man7.org/linux/man-pages/man3/pthread_create.3.html)が`pthread_t`でthead idを返したり、[Rustのasync](https://doc.rust-lang.org/std/keyword.async.html#)が[Future](https://doc.rust-lang.org/std/future/trait.Future.html)というステートマシンを返したりするのとは対照的です。

`pthread`が[pthread_join(3)](https://man7.org/linux/man-pages/man3/pthread_join.3.html)で終了を待てるのに対して、`goroutine`を指定して終了を待つ方法がありません。`goroutine`とそれを呼び出すコード間で`chan`や`sync.WaitGroup`(後述)などの変数を共有し、`goroutine`で動作する関数が明示的に通知することで終了を待ちます。

`goroutine`で動作する関数が終了すれば`goroutine`もexitします。きちんと終了できるようにするのはユーザーの責任です。

`goroutine`は[GOMAXPROCS](https://pkg.go.dev/runtime@go1.22.3#GOMAXPROCS)と同数(デフォルトでは(論理)CPUコアの個数)まで _concurrent_ に実行されます。
`GOMAXPROCS`かプロセスに与えられたCPUのコア数の少ないほうまで _parallel_ にコードを実行することができます。

[playground](https://go.dev/play/p/U8ERSyPrzt6)

```go
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()
	go func() {
		for {
			select {
			case <-ticker.C:
				fmt.Println("tick tok")
			case <-ctx.Done():
				return
			}
		}
	}()

	<-ctx.Done()
	/*
		tick tok
		tick tok
		tick tok
		tick tok
	*/
}()
```

`A Tour of Go`で網羅済みの内容ですので今更の紹介は必要ないかもしれませんが、このようにmain goroutineがブロックしてる間も動作しつづけていることと、
プログラム内のほかの変数(この場合`ticker`と`ctx`)にアクセスできることから同じアドレススペースで動いていることもわかると思います。

対象読者は`Node.js`の経験があるため、もしかしたら`goroutine`は[Promise](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Promise)に近い何かだと思うかもしれませんが、
`Promise`がコンストラクタに与えられたコールバック関数をeagerに実行するのに対して、`goroutine`は別段実行順序を保証しません(spec中に記載がありません)。

テストコードなどで明確にgoroutineに処理が切り替わるのを待つ必要がある場合は以下のように何かしらの方法でタイミングをそろえる必要があります。([探したらstd内でテストで似たようなことしてるところがあった](https://github.com/golang/go/blob/go1.22.3/src/cmd/cgo/internal/test/test.go#L1318-L1344))

```go
switchCh := make(chan struct{})
go func() {
	<-switchCh
	// ... do tasks ...
	close(switchCh)
}()
switchCh <- struct{}{} // block until the function above starts working
<-switchCh // wait until the goroutine above exits
```

:::details goroutineのメモリ消費量

goroutineは非常にメモリの負荷が非常に低いです。必要に応じて大量に作っても問題ありません。

goroutineは非常に軽量なスタック(見たところ[2KiBを最低値](https://github.com/golang/go/blob/go1.22.3/src/runtime/proc.go#L4901)として,[`windows`ならさらに4KiB, `plan9`なら追加で512byte, `ios`かつ`arm`ならば追加で1KiB](https://github.com/golang/go/blob/go1.22.3/src/runtime/stack.go#L72)=2~6KiB)をallocateします。その後必要に応じて拡張されていきます。

[GOMEMLIMITのデフォルトがmath.MaxInt64(=無制限)であること](https://pkg.go.dev/runtime@go1.22.3#hdr-Environment_Variables)から、例えばメモリが64GiBの`Linux`環境であれば

```
> (64 * 1024**3) / (2*1024)
33554432
```

ということで3300万個ぐらいのgoroutineを生成するとメモリを使い果たします。

実際に測ってみましょう。
以下のようなコードで100万個のgoroutineを無意味に待たせてみます。

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"os"
	"os/signal"
	"sync"
)

var (
	num = flag.Uint("n", 1_000_000, "num of goroutines")
)

func main() {
	fmt.Printf("pid = %d\n", os.Getpid())
	flag.Parse()
	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt)
	defer cancel()
	var wg sync.WaitGroup
	for range *num {
		wg.Add(1)
		go func() {
			defer wg.Done()
			<-ctx.Done()
		}()
	}
	wg.Wait()
	<-ctx.Done()
}
```

おおよそ`2.5g`使います。

```
top - 15:53:29 up  4:43,  0 users,  load average: 1.93, 0.96, 0.45
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.9 us,  0.4 sy,  0.0 ni, 98.5 id,  0.0 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem :  31661.5 total,  14527.8 free,   8854.9 used,   8278.9 buff/cache
MiB Swap:   8192.0 total,   8192.0 free,      0.0 used.  22346.2 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
61144 root      20   0 3812376   2.5g    808 S   0.0   8.0   0:14.08 main
```

計算より`0.5g`ほど多いですね。まあ計算のほうが正しくないんでしょうね。

まあそんな感じで、実際上はもっとメモリを消費したり(システムがgoroutineを作ったり、[\*g](https://github.com/golang/go/blob/go1.22.3/src/runtime/runtime2.go#L422)のallocationがあったり,runtimeがいろいろallocateしたり、そもそも書いてるプログラムが大きなsliceをallocateするなど)、そもそも制限がかかっていたり(`linux`によるプロセスごとのメモリリミット、(コンテナランタイムによる)cgroupsなど)ので、そこまでたくさんのgoroutineを作ることはできないかもしれません。実際上記のコードだとsingalのトラップ部分でも結構いろいろallocateされる動きが見えます。

ただgoroutineは大量に生成しても問題ないよっていうことを強調しておきます。実際の上限は(swapが起きて固まるのが怖いので)筆者は測るつもりはないです。

:::

### goroutineはすべて終了できるようにする

goroutineは[最低`2KiB`](https://github.com/golang/go/blob/go1.22.3/src/runtime/proc.go#L4901),[最大`1GB`](https://github.com/golang/go/blob/go1.22.3/src/runtime/proc.go#L153-L160)のstackを持ち、詳細な条件は調べてないのでわかりませんが、freeListに入れられて再利用されます。

`goroutine`がexitしなければこれらがfreeListに戻るなりしないので、leakyな`goroutine`はそれだけ無駄なメモリを消費します。

例えば[gopls](https://github.com/golang/tools/tree/master/gopls)は[Language Server Protocol](https://learn.microsoft.com/en-us/visualstudio/extensibility/language-server-protocol?view=vs-2022)に従って`stdin`/`stdout`ごしに`jsonrcp`で通信を行います。

そのための2つのファイルを１つの`duplex`なファイルであるかのように見せるために以下のような`fakeConn`を`net.Conn`として返し、

https://github.com/golang/tools/blob/b6235391adb3b7f8bcfc4df81055e8f023de2688/internal/fakenet/conn.go#L18-L29

使われ方そのものはここでは重要ではありませんが、`Listen`の代わりにこの`fakeConn`を直接使って`jsonrpc`の接続を確立するために使われます。

https://github.com/golang/tools/blob/b6235391adb3b7f8bcfc4df81055e8f023de2688/gopls/internal/cmd/serve.go#L136

`NewConn`で`go`キーワードが書かれています。
こういう感じで関数が`goroutine`を新しく作ることはよくあるわけなんですが、
この場合は以下のように`Close`で`goroutine`を終了できるようにしてあります。

https://github.com/golang/tools/blob/b6235391adb3b7f8bcfc4df81055e8f023de2688/internal/fakenet/conn.go#L59-L65

https://github.com/golang/tools/blob/b6235391adb3b7f8bcfc4df81055e8f023de2688/internal/fakenet/conn.go#L86-L93

このように、自然なリソース解放処理で作られた`goroutine`はすべてexitできるようになっているべきです(特にライブラリを作るときは)。
もし、リソース解放処理の後に実際に`goroutine`がexitするまでに時間がかかる場合は終了を待てる方法を提供するほうがよいでしょう。

例えば以下みたいに、全部の`goroutine`を終了させるメソッドと全部の`goroutine`の終了を待てるメソッドを作っておくほうがよいでしょう。

```go
type Watcher struct {
	ctx    context.Context
	cancel context.CancelFunc
	wg     sync.WaitGroup
}

func New() *Watcher {
	ctx, cancel := context.WithCancel(context.Background())
	return &Watcher{
		ctx:    ctx,
		cancel: cancel,
	}
}

type Target interface {
	Events() <-chan any
}

func (w *Watcher) Add(target Target) {
	w.wg.Add(1)
	go func() {
		defer w.wg.Done()
		for {
			select {
			case <-w.ctx.Done():
				return
			case ev := <-target.Events():
				// ... watch logic ...
			}
		}
	}()
}

func (w *Watcher) Close() {
	w.cancel()
}

func (w *Watcher) Wait() {
	w.wg.Wait()
}
```

### Multi-threadedプログラムで起こる典型的問題: data race(race condition)

#### race condition

[Wikipedia article](https://en.wikipedia.org/wiki/Race_condition)によれば、ソフトウェア文脈におけるrace conditionとは、複数のコードパスが同時に動作しており、それぞれのコードパスがそれぞれに予測とは異なった時間がかかってしまうことで、予測とは違った順序で処理を完了することで起きます。

これは、ファイルの読み書きであるとか、設定の書き換えであるとかを、複数のスレッド/複数のプログラムが行うことも含みます。状態に依存したり、状態を操作したりする処理をタイミングを合わせる方法なしで行うことで、予測しない状態を観測したりする問題のことを指していると読み取れます。

例を挙げるとプログラムの実行状態を保存したファイルがあって、プログラムは動作を完了するたびにあるフィールドの値をインクリメントするとします。プログラムAとBがあるとき、Aがファイルを読んで、書きこむまでの間にBが書き込みを行ったら、Aはその書き込みに対して上書きを行ってしまい、Bの増加分をなかったことにしてしまうので矛盾した状態になりますよね。

そういったrace conditionの典型の一つにdata raceがあります。

#### data race

[Data Race Detector](https://go.dev/doc/articles/race_detector)によれば、`Go`におけるdata raceは複数のgoroutineが同じ変数にアクセスしており、少なくとも1つ以上がwriteであると起きます。
以下があげられているdata raceを起こすコードのスニペットです

```go
func main() {
	c := make(chan bool)
	m := make(map[string]string)
	go func() {
		m["1"] = "a" // First conflicting access.
		c <- true
	}()
	m["2"] = "b" // Second conflicting access.
	<-c
	for k, v := range m {
		fmt.Println(k, v)
	}
}
```

[FAQ: Why are map operations not defined to be atomic?](https://go.dev/doc/faq#atomic_maps)によると、`map[T]U`へのconcurrentなアクセスはruntime panicが起こるように特別なチェックがかかっていますので上記のコードもうまくいけば(うまくいかなければ？)fatal panicが起きます(試した限り`recover`できない)。

`map[T]U`はruntimeにおいてはは[map構造体](https://github.com/golang/go/blob/go1.22.3/src/runtime/map.go#L116-L131)への[ポインター](https://go.dev/doc/effective_go#maps)で、mapへのwriteはあれこれ計算したうえで構造体に代入を行います(この場合[mapassign_faststr](https://github.com/golang/go/blob/go1.22.3/src/runtime/map_faststr.go#L203))。
readして計算してwriteを行いますので、複数のスレッドが同時にそのような処理を試みるとwrite中にreadして、途中のよくわからない状態を観測しまうことが考えれます。

...が、これを手元で動かしてみると特にエラーなく実行されることが多いですね。

例として試してもらえるように、もう少し不正状態の起きやすいサンプルを示しておきます。
以下のコードを筆者環境で**何度か**実行するとdata raceによる不正な状態を観測することができました。

`a []int`に、0から99の値をappendするという処理を、`GOMAXPROCS`と同数の`goroutine`の中で実行します。
data raceが起きていないとき、`a`は0-99の値のセットを順序不定で`GOMAXPROCS`と同数だけ持つことになるはずですが、
実際にはdata raceによって全く違う状態を**持つこともある**のです。(前述したとおり、数度に1回しか不正な状態は起きません)

[snippet](https://github.com/ngicks/go-basics-example/blob/main/snipet/data-race-example/main.go)

```go
package main

import (
	"fmt"
	"runtime"
	"slices"
	"sync"
)

func main() {
	var a []int

	var wg sync.WaitGroup
	for range runtime.GOMAXPROCS(0) {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for i := range 100 {
				a = append(a, i)
			}
		}()
	}
	wg.Wait()

	group := map[int]int{}
	var keys []int
	for _, num := range a {
		if !slices.Contains(keys, num) {
			keys = append(keys, num)
		}
		group[num] = group[num] + 1
	}
	slices.Sort(keys)
	for _, key := range keys {
		num, count := key, group[key]
		if count != runtime.GOMAXPROCS(0) {
			fmt.Printf(
				"data race caused invalid state at num = %d, count = %d, expected = %d\n",
				num, count, runtime.GOMAXPROCS(0),
			)
		}
	}
	fmt.Printf("result = %#v\n", group)
	/*
	   data race caused invalid state at num = 0, count = 2, expected = 24
	   data race caused invalid state at num = 1, count = 2, expected = 24
	   data race caused invalid state at num = 2, count = 2, expected = 24
	   ...省略..
	   data race caused invalid state at num = 97, count = 2, expected = 24
	   data race caused invalid state at num = 98, count = 2, expected = 24
	   data race caused invalid state at num = 99, count = 2, expected = 24
	   result = map[int]int{0:2, 1:2, 2:2, 3:2, 4:2, 5:2, 6:2, 7:2, 8:2, 9:2, 10:2, 11:2, 12:2, 13:2, 14:2, 15:2, 16:2, 17:2, 18:2, 19:2, 20:2, 21:2, 22:2, 23:2, 24:2, 25:2, 26:2, 27:2, 28:2, 29:2, 30:2, 31:2, 32:2, 33:2, 34:2, 35:2, 36:2, 37:2, 38:2, 39:2, 40:2, 41:2, 42:2, 43:2, 44:2, 45:2, 46:2, 47:2, 48:2, 49:2, 50:2, 51:2, 52:2, 53:2, 54:2, 55:2, 56:2, 57:2, 58:2, 59:2, 60:2, 61:2, 62:2, 63:2, 64:2, 65:2, 66:2, 67:2, 68:2, 69:2, 70:2, 71:2, 72:2, 73:2, 74:2, 75:2, 76:2, 77:2, 78:2, 79:2, 80:2, 81:2, 82:2, 83:2, 84:2, 85:2, 86:2, 87:2, 88:2, 89:2, 90:2, 91:2, 92:2, 93:2, 94:2, 95:2, 96:2, 97:2, 98:2, 99:2}
	*/
}
```

#### race detector

ブログポスト[Data Race Detector](https://go.dev/doc/articles/race_detector)で紹介される通り、`Go`にはrace detectorというdata raceを検知するためのツールが組み込まれています。

ブログポストで述べられる通り、ビルドを行う各種コマンドに`-race`というフラグを追加することでrace detectorが有効になります。

```
$ go test -race mypkg    // to test the package
$ go run -race mysrc.go  // to run the source file
$ go build -race mycmd   // to build the command
$ go install -race mypkg // to install the package
```

- `-race`を追加すると`CGO`が必須になります。
  - [README](https://github.com/golang/go/tree/go1.22.3/src/runtime/race)によると、`LLVM`プロジェクトの一部の`ThreadSanitizer`を使用します
    - [ここ](https://github.com/golang/build/blob/30d17922cc07a64230259027b7fcd31f57cb747b/cmd/racebuild/racebuild.go#L196)を見ると[LLVMのtsan/go](https://github.com/llvm/llvm-project/tree/main/compiler-rt/lib/tsan/go)をビルドしたものを使っていますね(=C++です)。
- [CPUとメモリ負荷が大幅に増え,defer recoverで8byteの追加のメモリーがallocateされるうえにgoroutineがexitするまで回収されない](https://go.dev/doc/articles/race_detector#Runtime_Overheads)ので、長いこと動作させるとそれで落ちるケースがあります。
- ライブラリが`race`フラグを見て動作を変える部分もあります。(例: [sync.Poolはrace enabledだとランダムな要素をPutしない](https://github.com/golang/go/blob/go1.22.3/src/sync/pool.go#L100-L107))
- race detectorはメモリが同時に読み書きされないかチェックするツールなのでたまたま衝突が起きないケースがあると検出できません。
  - 衝突が起きやすくなるように異なる関数やメソッドを同時に読んでみるテストを書くといいかもしれません。

上記のコードスニペットをrace detector付きで実行すると、以下のように警告がstderrにプリントされます。
(ソースのコードパスは一応`--redacted--`に置き換える編集をしています。実際にはソースのローカルストレージ上の絶対パスが出力されます)

```
## go run -race ./snipet/data-race-example/main.go
==================
WARNING: DATA RACE
Read at 0x00c000130000 by goroutine 7:
  main.main.func1()
      /--redacted--/snipet/data-race-example/main.go:19 +0xd0

Previous write at 0x00c000130000 by goroutine 11:
  main.main.func1()
      /--redacted--/snipet/data-race-example/main.go:19 +0x154

Goroutine 7 (running) created at:
  main.main()
      /--redacted--/snipet/data-race-example/main.go:16 +0xb4

Goroutine 11 (finished) created at:
  main.main()
      /--redacted--/snipet/data-race-example/main.go:16 +0xb4
==================
...省略...
==================
WARNING: DATA RACE
Write at 0x00c0002989b0 by goroutine 15:
  main.main.func1()
      /--redacted--/snipet/data-race-example/main.go:19 +0x131

Previous write at 0x00c0002989b0 by goroutine 23:
  main.main.func1()
      /--redacted--/snipet/data-race-example/main.go:19 +0x131

Goroutine 15 (running) created at:
  main.main()
      /--redacted--/snipet/data-race-example/main.go:16 +0xb4

Goroutine 23 (running) created at:
  main.main()
      /--redacted--/snipet/data-race-example/main.go:16 +0xb4
==================
data race caused invalid state at num = 0, count = 6, expected = 24
data race caused invalid state at num = 1, count = 7, expected = 24
data race caused invalid state at num = 2, count = 9, expected = 24
...省略...
data race caused invalid state at num = 97, count = 6, expected = 24
data race caused invalid state at num = 98, count = 6, expected = 24
data race caused invalid state at num = 99, count = 6, expected = 24
result = map[int]int{...省略...}
Found 4 data race(s)
exit status 66
```

#### テストは-raceあり/なしどっちもで実行しよう

race detectorは有効/無効でふるまいに違いが出てきます。(特にテストは)`-race`あり/なしどちらでも実行してうまく動作するか確認したほうがよいでしょう。

筆者は何度かrace detectorを**無効にすると**通過しなくなるテストを経験しています。
これはたぶんdata raceではないrace conditionが生じていたのだと思われます。
ですので、タイミングが重要なタイプのライブラリに対しては、テストはrace detectorあり/なしで2度以上実行することをお勧めします。

もちろんしっかりsynchronizationを(テストの中で)とれるように実装する必要はあります。そのためのツールキットは次の節以降で触れていきます。

### chan

https://go.dev/ref/spec#Channel_types

Specificationによれば、channelはconcurrentに実行されている関数(`goroutine`)間でコミュニケートするメカニズムを提供します。

`A Tour of Go`で網羅済みの内容なので、基本的なことは以下のスニペットのみで省略します。

```go
ch := make(chan int) // unbufferd
// ch := make(chan int, 0) // unbuffered
ch := make(chan int, 100) // buffered

// https://go.dev/ref/spec#Length_and_capacity
len(ch) // number of elements queued in channel buffer
cap(ch) // channel buffer capacity

go func () {
	for {
		// ...何かの処理...
		ch <- 4768 // send
	}
}()

go func () {
	for {
		num := <-ch // recv
	}
}()
```

[Effective Goのchannelの項目](https://go.dev/doc/effective_go#channels)で少し進んだ使い方も紹介されているので一通り読んでおくといいでしょう。
ただ1つだけ変わったこととして、

> The bug is that in a Go for loop, the loop variable is reused for each iteration

は[Go1.22](https://tip.golang.org/doc/go1.22)以降では正しくありません。`for i, v := range s {}`の`i, v`はgoroutineをまたいで参照されるのを検知するなどしたら再利用されなくなりました。

それ以外の細かい挙動としては

- [close](https://go.dev/ref/spec#Close)を呼ぶことでchennelを閉じることができます。
- closeされたchannelからの受信は即座にアンブロックします
  - 二つ目の返り値がfalseになることで検知できます
- nil channelやcloseされたchannelのcloseはパニックです
- nil channelへの送受信は永久にブロックします

以下のスニペットで挙動をしめします。

[playground](https://go.dev/play/p/2v7l_BIoYnz)

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	unbuffered := make(chan struct{})
	buffered := make(chan struct{}, 100)
	fmt.Printf("cap(unbuffred) = %d, cap(buffered) = %d\n", cap(unbuffered), cap(buffered)) // cap(unbuffred) = 0, cap(buffered) = 100

	for range 10 {
		buffered <- struct{}{}
	}
	fmt.Printf("len(unbuffred) = %d, len(buffered) = %d\n", len(unbuffered), len(buffered)) // len(unbuffred) = 0, len(buffered) = 10

	// recv from / send on closed channel
	func() {
		c := make(chan struct{})
		defer func() {
			rec := recover()
			fmt.Printf("recovered: %#v\n", rec) // recovered: "send on closed channel"
		}()
		close(c)
		_, ok := <-c
		fmt.Printf("ok = %t\n", ok) // ok = false
		c <- struct{}{}
	}()

	// close of closed channel
	func() {
		c := make(chan struct{})
		defer func() {
			rec := recover()
			fmt.Printf("recovered: %#v\n", rec) // recovered: "close of closed channel"
		}()
		close(c)
		close(c)
	}()

	// close of nil channel
	func() {
		var c chan struct{}
		defer func() {
			rec := recover()
			fmt.Printf("recovered: %#v\n", rec) // recovered: "close of nil channel
		}()
		close(c)
	}()

	// close of receive-only channel is compilation error
	// cc := make(chan struct{})
	// var c <-chan struct{} = cc
	// close(c) // invalid operation: cannot close receive-only channel c (variable of type <-chan struct{})

	// recv from nil channel
	func() {
		var c chan struct{}
		ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond)
		defer cancel()
		select {
		case <-c:
		case <-ctx.Done():
			fmt.Printf("timed out: recv on nil channel\n") // timed out: recv on nil channel
		}
	}()

	// send on nil channel
	func() {
		var c chan struct{}
		ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond)
		defer cancel()
		select {
		case c <- struct{}{}:
		case <-ctx.Done():
			fmt.Printf("timed out: send on nil channel\n") // timed out: send on nil channel
		}
	}()
}
```

#### buffer-sizeはどう決めるか？

- 0: synchronizationしたい
- 1: 送信でブロックしたくないけど受信はブロックしたい(i.e. asyncに通知を行いたい)
  - e.g. `Go 1.23`以前の`time.Timer`の内部のchannelはbuffer-size 1
    - buffer-size 1の例に`time.Timer`がちょうどいいかと思っていたんですが、`Go1.23`から改善が入るので正しくなってしまいました。
      - https://github.com/golang/go/issues/37196
      - https://github.com/golang/go/commit/966609ad9e
  - 後述のnotifier
- N: pipeline風にデータを引き渡しており、consumerとproducerの間に処理速度差があり、producerがburstyならば、バックプレッシャーをかけ始めたい任意のサイズN
  - 現実的に起こるイベントはほとんどそう(だから在庫管理が一つの学問になるん)だろうという突っ込みはあります。
  - ただし、channelがキューイングしているデータの中身を観測する方法は筆者が知る限り普通にはないため、queueを管理したいならchannelではなく別のqueueを定義したほうが良い。
    - 単なるFIFO queueならchannelを利用するだけで便利だと思いますが、queueがpriority queueであってほしいとかだと特に別の実装が必要になります。
    - ここで参考にどうぞというためだけにpriority queueを追加しました: [github.com/ngicks/eventqueue](https://github.com/ngicks/eventqueue)

だいたいこんなものでしょうか？何かしらを引用して経験的にこうと述べたかったんですが、goproxyに登録されているすべてのモジュールの`make(chan T)`を検索するぐらいしか思いつかなくて、
結局引用なしで筆者がどこかで見たことのまとめになってしまいました。

buffer-size n > 1が便利な場面もあるけど、大抵の場合は0か1だと思います。

#### channelはどのようにcloseするか

- いろんなところでcloseするのは避ける
  - `close of closed channel`, `send on closed channel`でパニックが起きるため、`close`の責務を負うのは誰なのかを明確にすべき
    - `make(chan T)`を呼んだ関数/structが責任をもってcloseを呼ぶ、とか
    - `chan<- T`を関数が返す時、closeしてもらうことで終了を通知する、とか
- 場合によっては`sync.OnceFunc`などを使って１度しかcloseが呼ばれないのを保証する。
- 関数が`<-chan T`を返す時はcloseによって終了を通知することがある。
- structのフィールドにchannelを引き渡すようなケースの場合大分ややこしいのでcloseによる終了の通知よりも、明確にdone channelを作るとか、`context.Context`を引き回すとかしたほうが良い。
- なんならcloseしなくてもよい
  - closeしなくてもGCに回収される
  - `time.Timer`などは`<-chan time.Time`を返してくるが、これらをcloseする方法はないことから、このことがわかる。

#### 特定のchannelを優先するには1段selectで包む

初期の講演でRob Pike自身も述べていますが[selectで送受信が同時に可能になっている場合ランダムにどれかが選ばれる](https://go.dev/ref/spec#Select_statements)ので、優先して送受信したいチャネルがある場合は、1段selectで包む必要があります

```go
for {
	select {
	case <-ctx.Done():
		return ctx.Err()
	default:
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-otherChannels:
		case someOther <- 547891:
		}
	}
}
```

#### Example: notifier

`<-chan struct{}`を返して、返したchannelをcloseすることでnotifyするパターン。
通知される側が複数の時に用いられるのを見たことがある。

```go
// Single-producer multi-consumer notifier
type SpmcNotifier struct{
	mu sync.Mutex
	ch chan struct{}
}

func (n *SpmcNotifier) Chan() <-chan struct{} {
	n.mu.Lock()
	defer d.mu.Unlock()
	if n.ch == nil {
		n.ch = make(chan struct{})
	}
	return n.ch
}

func (n *SpmcNotifier) Notify() {
	n.mu.Lock()
	defer n.mu.Unlock()
	if n.ch	!= nil {
		ch := n.ch
		n.ch = nil
		close(ch)
	}
}
```

buffer-size 1のchannelを使ってイベントがあったことだけを保存する。
こちらは通知される側が1つだけ、あるいは1度だけの時によく使うというイメージ。

[time.Timer](https://pkg.go.dev/time@go1.22.3#Timer)が返す`C`はbuffer-size 1でchannelが埋まっていなければ時間をsendするようになっています。
ただし`Go.1.23`から[改善が入る](https://github.com/golang/go/commit/966609ad9e)のでそれ以降はこの記述は正しくならなくなりました。

```go
// Multi-producer single-consumer notifier
type MpscNotifier struct{
	ch chan struct{}
}

func New() *MpscNotifier {
	return &MpscNotifier{
		ch: make(chan struct{}, 1)
	}
}

func (n *MpscNotifier) Chan() <-chan struct{} {
	return n.ch
}

func (n *MpscNotifier) Notify() {
	select {
		case n.ch <- struct{}{}:
		default:
	}
}
```

#### Example: send / recv one of

今まで述べてきた性質を利用して、動的な個数のchannelを使ったselectを実装します。

前述しましたが、nil channelからの送受信は永久にブロックします。逆に言えば、`[N]chan T`に対し、任意のインデックス`i`に`nil`を代入すれば`N`を上限とした動的な個数のchannelに送受信できます。
`reflect`を利用すれば現実的な上限なしの動的な個数のchannelの送受信を実装できます。

以下で与えられた`[]chan T`のうちどれか一つからsend / recvする関数を実装してみます。

```go
func Send[T ~[]C, C ~(chan<- E), E any](chans T, v E, cancel <-chan struct{}) (chosen int, sent bool) {
	var c [4]C
	_ = copy(c[:], chans)
	return Send4(c, v, cancel)
}

func Send4[T ~[4]C, C ~(chan<- E), E any](chans T, v E, cancel <-chan struct{}) (chosen int, sent bool) {
	sent = true
	select {
	case <-cancel:
		sent = false
	case chans[0] <- v:
		chosen = 0
	case chans[1] <- v:
		chosen = 1
	case chans[2] <- v:
		chosen = 2
	case chans[3] <- v:
		chosen = 3
	}
	return
}

func Recv[T ~[]C, C ~(<-chan E), E any](chans T, cancel <-chan struct{}) (v E, chosen int, received bool) {
	var c [4]C
	_ = copy(c[:], chans)
	return Recv4(c, cancel)
}

func Recv4[T ~[4]C, C ~(<-chan E), E any](chans T, cancel <-chan struct{}) (v E, chosen int, received bool) {
	received = true
	select {
	case <-cancel:
		received = false
	case v = <-chans[0]:
		chosen = 0
	case v = <-chans[1]:
		chosen = 1
	case v = <-chans[2]:
		chosen = 2
	case v = <-chans[3]:
		chosen = 3
	}
	return
}
```

この`Send4`/`Recv4`を任意数まで実装すれば何個でも対応できます

```go
func Send[T ~[]C, C ~(chan<- E), E any](chans T, v E, cancel <-chan struct{}) (chosen int, sent bool) {
	switch x := len(chans); {
	case x == 0:
		panic("zero chans")
	case x <= 4:
		var c [4]C
		_ = copy(c[:], chans)
		return Send4(c, v, cancel)
	case x <= 8:
		var c [8]C
		_ = copy(c[:], chans)
		return Send8(c, v, cancel)
	case x <= 16:
		var c [16]C
		_ = copy(c[:], chans)
		return Send16(c, v, cancel)
	// ... and so on ...
	}
}
```

32, 64, 128...と実装していくこと自体は簡単です; code generatorを整備したら簡単にいくらでも増やせます。
ただそうするとコードサイズがすさまじく大きくなりそうなので、現実的ではないですよね。ということで、`reflect`を使って本当の意味で任意数なchannelに対応します。

```go
func SendN[T ~[]C, C ~(chan<- E), E any](chans T, v E, cancel <-chan struct{}) (chosen int, sent bool) {
	cases := []reflect.SelectCase{{
		Dir:  reflect.SelectRecv,
		Chan: reflect.ValueOf(cancel),
	}}
	for _, ch := range chans {
		cases = append(cases, reflect.SelectCase{
			Dir:  reflect.SelectSend,
			Chan: reflect.ValueOf(ch),
			Send: reflect.ValueOf(v),
		})
	}
	chosen, _, _ = reflect.Select(cases)
	if chosen == 0 {
		return
	}
	return chosen - 1, true
}

func RecvN[T ~[]C, C ~(<-chan E), E any](chans T, cancel <-chan struct{}) (value E, chosen int, received bool) {
	cases := []reflect.SelectCase{{
		Dir:  reflect.SelectRecv,
		Chan: reflect.ValueOf(cancel),
	}}
	for _, ch := range chans {
		cases = append(cases, reflect.SelectCase{
			Dir:  reflect.SelectRecv,
			Chan: reflect.ValueOf(ch),
		})
	}
	chosen, recv, _ := reflect.Select(cases)
	if chosen == 0 {
		return
	}
	return recv.Interface().(E), chosen - 1, true
}

func Send[T ~[]C, C ~(chan<- E), E any](chans T, v E, cancel <-chan struct{}) (chosen int, sent bool) {
	switch x := len(chans); {
	// ... cases ...
	default:
		return SendN(chans, v, cancel)
	}
}

func Recv[T ~[]C, C ~(<-chan E), E any](chans T, cancel <-chan struct{}) (v E, chosen int, received bool) {
	switch x := len(chans); {
	// ... cases ...
	default:
		return RecvN(chans, cancel)
	}
}
```

`Send`を何度も実行し、成功するたびに送受信できたchannelをnilにフォールバックすれば「同じ値をすべてのchannelに1度ずつ送る」が実現できますね。
つまり以下のような感じです。

(サンプルはランダムな値を送りたかったので`E`の代わりに`func() E`を渡せるようにしています)

```go
func SendEach[T ~[]C, C ~(chan<- E), E any](chans T, fn func() E, cancel <-chan struct{}) (sent []int, completed bool) {
	chans = slices.Clone(chans)
	sent = make([]int, 0, len(chans))
	completed = true

	for len(chans) != len(sent) {
		chosen, ok := Send(chans, fn(), cancel)
		if !ok {
			completed = false
			break
		}
		sent = append(sent, chosen)
		chans[chosen] = nil
	}
	return
}
```

実装物は以下に置いてあります

https://github.com/ngicks/go-basics-example/tree/main/snipet/chan-one-of

一応動いています。(このrepositoryはバージョン管理する気が皆無なのでimportはしないでください)

現実的にfan-in / fan-outを実装するときにこういうのは使いますかね？`SendEach`の使い道は筆者はぱっと思いつかなかったです。なので対象読者にとっても向こう数年は不要かもしれません。

多分`reflect`を使うとオーバーヘッドがかかるので`len(chans)<=16`みたいな適当な小さい数までは固定数版に分岐する処理が妥当だと思って実装してみましたが、これが本当にいいことなのかはよくわかっていない(ベンチをとっていない)ので参考までに、という感じです。

### context.Context

https://pkg.go.dev/context@go1.22.3

[context.Context](https://pkg.go.dev/context@go1.22.3#Context)はhttp requestのrequest-scopeのようなスコープに紐づく情報を運ぶ手段を提供するinterfaceです。

- `Value`で`context.Context`に収められた任意の値を取り出すことができます
  - [context.WithValue](https://pkg.go.dev/context@go1.22.3#WithValue)で値を収めることができます。
  - ただこれらの`With*`関数は入れ子にしていくので、上位のスコープに`Value`を伝搬したい場合は収めた値自体に工夫が必要です。
    - e.g. `*sync.Map`を収めて下層で操作してもらう。
- `Done`で`<-chan struct{}`を返します。`context.Context`がcancelされたときこのchannelはcloseされます。
- cancelされた場合`Err`がnon-nil errorを返すようになります。

処理時間の長くかかる関数/メソッドはとりあえず第一引数に`context.Context`を受けとって起き、`ctx.Done()`や、`ctx.Err()`によってcancellationされた時を検知して即座にreturnできるようにするとよいでしょう。

```go
func longLoongJob(ctx context.Context) error {
	// ... do something ...

	timer := time.NewTimer(dur)
	select {
	// ctx.Done()はcancelされるとcloseされる。
	case <-ctx.Done():
		return ctx.Err()
	case <-timer.C:
	}

	// ctx.Err()はcancelされるとnon-nil
	if err := ctx.Err(); err != nil {
		return err
	}

	// ... do something ...
	return nil
}
```

そもそも大抵の長い処理を行う関数は`context.Context`を第一引数で受け取るようになっていると思うので、そのうち自然とこの様式がわかるようになると思います。

[context.WithCancel](https://pkg.go.dev/context@go1.22.3#WithCancel)のドキュメントに以下のようにありますが

> Canceling this context releases resources associated with it, so code should call cancel as soon as the operations running in this Context complete.

入れ子にするだけなのに何をリリースするねん？と思うかもしれませんが、親contextのcancelを伝搬するために親contextが既知の(=contextパッケージで実装された)ものでないとき、新しい`goroutine`の中で親contextのcancelを`Done`で監視するので、こういう感じで`cancel`は不明確に確保されたリソースをリリースされます。

### sync

https://pkg.go.dev/sync@go1.22.3
https://pkg.go.dev/sync/atomic@go1.22.3

`sync`および`sync/atomic`パッケージではsynchronization primitiveが実装されています。
`Go`には`channel`という第一級市民が存在していますが、[mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)や[atomic](https://en.wikipedia.org/wiki/Linearizability)を使うほうが問題をシンプルに解決できるときがたびたびあります。

おそらく対象読者には`mutex`や`atomic`自体がなじみない概念だと思います。

`mutex`(MUTual EXclusion=相互排他)はあるコードパス(よく*critical section*と呼ばれる)を実行している`thread of execution`が一つであることを保証する機能のことを指します。
前述のとおり、複数の`thread`が同じ変数がアクセスし、少なくとも一つが`write`である場合data raceを起こします。`mutex`はこれらのリソースへのアクセスをたった１つのプロセス、あるいはスレッドに制限することでdata raceが起きるのを防ぐことができます。
`mutex`は大抵`Lock`と`Unlock`ができるインターフェイスを備え、`Lock`によってコードパスをロックします。`Lock`を保持している状態で他の`thread`が`Lock`を呼び出すと、`Lock`を保持している`thread`が`Unlock`を呼び出すまでその`thread`はブロックしつづけます。

`atomic`はCPUに備わった命令(`x86`では`LOCK` prefix)を利用してある変数の観測と変更をほかのスレッドにわりこまれないようにするものです。`mutex`より粒度が小さくできることが限られています。`mutex`の実装にも使われています。

[Node.js]もとい`javascript`はスクリプトの実行はシングルスレッドなのでこういった概念が必要ないケースも多かったでしょうし、[python]には`GIL`があるのでこういう概念自体がなくてもすべてのコード呼び出しはシリアライズされていたので意識しなかったかもしれません。
(ただ[python]においては今後[PEP703](https://peps.python.org/pep-0703/)が徐々に実装されて`GIL`を解除できるようになったらこういう`mutex`を使わないといけなくなるかもしれません。)
single threadな両者でも時にconcurrentなリソースへアクセスする際に制限をする必要があることがあるので、[async-mutex](https://www.npmjs.com/package/async-mutex)などを利用した対象読者も多いかもしれません。

`mutex`と`atomic`変数以外のsynchronization primitiveは`mutex`や`atomic`を使って実装されています。

#### sync/atomicの概要

[sync/atomic](https://pkg.go.dev/sync/atomic@go1.22.3)で各種`atomic`な変数の操作が実装されています。各`int`/`uint` variantに対して`Add*`, `CompareAndSwap*`, `Load*`, `Store*`, `Swap*`が実装されています。
さらに[Go1.19](https://tip.golang.org/doc/go1.19#atomic_types)より便利な[atomic type](https://pkg.go.dev/sync/atomic@go1.22.3#pkg-types)が実装されています。

基本は[type](https://pkg.go.dev/sync/atomic@go1.22.3#pkg-types)を使っておくほうが良いと思います。
なお見て分かると思いますが、`Add*`はあっても`Sub*`はないので引き算が必要ならば`Uint*`ではなく`Int*`を使います。

大体は`mutex`なしでconcurrent-safeに変数を書き換えるのに使います。

```go
var working atomic.Bool
if !working.CompareAndSwap(false, true) { // oldがfalseだった場合のみtrueに置き換える。
	// swapできなかった=oldがtrueだった
	return ErrAlreadyWorking
}
defer working.Store(false)
// ... work ...
```

TODO: add progress reader example

余談ですが、C++やRustはatomic accessのorderingを複数選択可能です([The Rustonomicon::atomics](https://doc.rust-lang.org/nomicon/atomics.html))が、[Goはsequentially consistentしかありません](https://pkg.go.dev/sync/atomic@go1.22.3#pkg-overview)

#### syncの概要

[sync](https://pkg.go.dev/sync@go1.22.3)は`Go`のstdが提供するツールキットの中でもとりわけ重要できちんと理解しておく必要があります。
以下で現時点(=`Go1.22.3`)でexportされている関数や型を適当な順番でざっくり説明します。
詳細は上記ドキュメントを見てください。

- [Mutex](https://pkg.go.dev/sync@go1.22.3#Mutex)
  - [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)です
  - 見た感じ何度か[spinlock](https://en.wikipedia.org/wiki/Spinlock)を試みるので呼び出しが`FIFO`になるとは限らないっぽいです。
- [RWMutex](https://pkg.go.dev/sync@go1.22.3#RWMutex)
  - 複数の`ReadLock`、もしくは１つの`WriteLock`を同時にとることができます。
  - `Reader`が多くて`Write`する頻度が低い時はこちらを用いるとよいのでしょう。
- [Once](https://pkg.go.dev/sync@go1.22.3#Once)
  - 1度だけ実行されるのを保証するprimitiveです
  - 同時に`Do`を呼ばれると、二つ目以降の呼び出しは一つ目の呼び出しが終了するまで待ちます。
  - パッケージやstructの遅延初期化によく使います。
  - `Do`に渡す関数はコードパスごとに別のものでよいですが、`Do`が実行されるのは`Once`に対して1度きりです。
  - 下記`Once*`が[Go1.21.0](https://tip.golang.org/doc/go1.21#syncpkgsync)より追加されたため、これが直接使われるとき自動的により複雑なユースケースを実現しようとしていると思われるようになりました。
    - つまり、コードパスごとに`Do`に渡す関数が別のものじゃないと`Once`を直接使う理由があんまりないです。
    - e.g. [context.AfterFunc](https://pkg.go.dev/context@go1.22.3#AfterFunc)の[ここ](https://github.com/golang/go/blob/go1.22.3/src/context/context.go#L322-L324)と[ここ](https://github.com/golang/go/blob/go1.22.3/src/context/context.go#L347-L349)
- [OnceFunc](https://pkg.go.dev/sync@go1.22.3#OnceFunc) ([Go1.21.0](https://tip.golang.org/doc/go1.21#syncpkgsync)より)
  - [Once](https://pkg.go.dev/sync@go1.22.3#Once)のラッパーで典型的な使い道の１つである「特定の関数を１度だけ実行する」というのができます。
  - 逆に[Once](https://pkg.go.dev/sync@go1.22.3#Once)を直接使ってる場合自動的にもっと複雑なユースケースなのかと思われるようになりました。
  - 渡された`func()`が`panic`した際`OnceFunc`から返される関数の呼び出しすべてに`panic`が伝搬するように実装されています。
- [OnceValue](https://pkg.go.dev/sync@go1.22.3#OnceValue)([Go1.21.0](https://tip.golang.org/doc/go1.21#syncpkgsync)より)
  - [OnceFunc](https://pkg.go.dev/sync@go1.22.3#OnceFunc)と同様ですが渡す関数が返り値を１つもてます。
- [OnceValues](https://pkg.go.dev/sync@go1.22.3#OnceValues)([Go1.21.0](https://tip.golang.org/doc/go1.21#syncpkgsync)より)
  - [OnceFunc](https://pkg.go.dev/sync@go1.22.3#OnceFunc)と同様ですが渡す関数が返り値を2つもてます。
  - おそらくエラーしうる初期化に用いるためにあるのだと思われます。
- [WaitGroup](https://pkg.go.dev/sync@go1.22.3#WaitGroup)
  - アトミックにincrement/decrementできるカウンターで、`Wait`でカウンターが0になるまで待つことができます。
  - `Add`でn個increment、`Done`で1つdecrementします
  - `goroutine`が終了するのを待つのによく使います。
    - もちろんそれ以外のことに使ってもよいです。あくまでカウンターです。
  - 基本的に`Done`は`defer wg.Done()`で呼び出したほうがよいでしょう
    - `goroutine`で呼び出す関数がpanicしたり、`runtime.Goexit`を読んだとしても`Done`が呼び出せるからです。
    - panic時に`recover`する気がないなら`defer`じゃなくてもいいです。
  - `Wait`を呼ぶとカウントが0になるまでその`goroutine`ブロックします。すでに0ならすぐアンブロックします。
  - これまでのコードサンプルは特に何も断らずにこれを使ってきているので使い方はきっともうわかっていることでしょう。
- [Pool](https://pkg.go.dev/sync@go1.22.3#Pool)
  - 初期化にそれなりにコストを伴なったり実行中にallocationが起きる一時的なオブジェクトをためておくプールです
  - 典型的ユースケースは後述
- [Map](https://pkg.go.dev/sync@go1.22.3#Map)
  - concurrent-safeな`map[any]any`みたいなものです
  - `key`, `value`とも`any`型ですが`key`はcomparableである必要があります。
  - `map[T]U`は実装上`key`ごとのlockを行うようなことはできないため、ランダムな`key`にconcurrentにアクセスする場合は適しているとドキュメントされています。
  - 典型的なユースケースは後述
  - [Go1.20.0](https://tip.golang.org/doc/go1.20#syncpkgsync)より追加された[(\*sync.Map).CompareAndDelete](https://pkg.go.dev/sync@go1.22.3#Map.CompareAndDelete)、 [(\*sync.Map).CompareAndSwap](https://pkg.go.dev/sync@go1.22.3#Map.CompareAndSwap)を利用する際には`value`の型もcomparableである必要があります。
- [Cond](https://pkg.go.dev/sync@go1.22.3#Cond)
  - ある状態を監視する複数の`goroutine`に対して、状態の変更の通知を効率的に行う仕組み・・・というべきなんでしょうか？
  - 便利な使い道はあるんですがいつ使うべきか説明しにくい機能です。
  - もしかしたら対象読者は向こう数年使わないかもしれません。
  - [こういう概念自体はPOSIX APIにもあって](https://man7.org/linux/man-pages/man3/pthread_cond_init.3p.html)広く認知されています
  - 一応使用例を後述します

#### sync.Poolの典型的ユースケース: buf pool

[sync.Pool](https://pkg.go.dev/sync@go1.22.3#Pool)の典型的ユースケースにbuf poolがあります。

大きな`[]byte`や`*bytes.Buffer`が必要であるが、毎回allocateするとメモリー的に負荷が高いので、その関数が高い確率で繰り返し実行されると予測されるときこういったpoolが必要となります。
前述通り`goroutine`の初期スタックサイズは`2-6KiB`なので大きなbyte array`[N]byte`を宣言してしまうとおそらくスタック成長が起きてパフォーマンスが落ちるはずなので、`[]byte`のallocation自体は避けられません。

よく[io.CopyBuffer](https://pkg.go.dev/io@go1.22.3#CopyBuffer)と一緒に使います。
[io.Copy](https://pkg.go.dev/io@go1.22.3#Copy)は引数の`src`が[io.WriterTo](https://pkg.go.dev/io@go1.22.3#WriterTo)、`dst`が[io.ReaderFrom](https://pkg.go.dev/io@go1.22.3#ReaderFrom)をそれぞれどちらも実装しない場合、実行のたびに32KiB(`src`が`io.LimitedReader`である場合は`src.N`)の`[]byte`をallocateしてしまうので、普通は`io.CopyBuffer`を使うほうがよいでしょう。

[playground(full)](https://go.dev/play/p/GGFQ135j9HA)

```go
// 8KiBにしているのは単なるサンプルで、
// ファイルを読み書きするなら64KiB、
// io.CopyBuffer向けに使うなら32KiBとかの適当な値に増やしたほうがよいでしょうね。
const bufSize = 8 * 1024

var bytesPool = &sync.Pool{
	New: func() any {
		b := make([]byte, bufSize, bufSize)
		return &b
	},
}

func getBytes() *[]byte {
	return bytesPool.Get().(*[]byte)
}

func putBytes(b *[]byte) {
	if b == nil || len(*b) != bufSize || cap(*b) != bufSize {
		// reject grown / shrunk
		return
	}
	bytesPool.Put(b)
}
```

```go
var bufPool = &sync.Pool{
	New: func() any {
		return new(bytes.Buffer)
	},
}

func getBuf() *bytes.Buffer {
	return bufPool.Get().(*bytes.Buffer)
}

func putBuf(b *bytes.Buffer) {
	if b.Cap() > 64*1024 {
		// See https://golang.org/issue/23199
		return
	}
	b.Reset()
	bufPool.Put(b)
}
```

コメントでリンクされている通り、大体同じサイズの要素をプールしないと効率が悪いので大きすぎるものはPutしないようにしてあります。
`fmt`のプールも同じことをしていますね。

https://github.com/golang/go/blob/go1.22.3/src/fmt/print.go#L160-L182

`bytesPool`が返す`[]byte`はappendを呼ばれると`cap(s)`を超えてしまうので、
新しいarrayがallocateされてしまいます。
なので、appendが必要なsliceのプーリングはもう少し考慮が必要です。

具体的には以下のような感じで、

https://github.com/golang/go/blob/go1.22.3/src/crypto/tls/conn.go#L985-L995

`Put`する前にgrowしたスライスをescape済みのポインターに再代入するとallocationの頻発を防げます。

#### sync.Mapの典型的ユースケース: キャッシュ

[sync.Map](https://pkg.go.dev/sync@go1.22.3#Map)の想定される典型的ユースケースにcacheがあります

> https://pkg.go.dev/sync@go1.22.3#Map
>
> The Map type is optimized for two common use cases: (1) when the entry for a given key is only ever written once but read many times, as in caches that only grow, or (2) when multiple goroutines read, write, and overwrite entries for disjoint sets of keys. In these two cases, use of a Map may significantly reduce lock contention compared to a Go map paired with a separate Mutex or RWMutex.

これの`(1)`のほうですね。

実際std内でもcacheとして使われています。

- [encoding/jsonのmarshalerのキャッシュとして](https://github.com/golang/go/blob/go1.22.3/src/encoding/json/encode.go#L342-L369)
- [go list -export -f {{.Export}}の結果のキャッシュとして](https://github.com/golang/go/blob/go1.22.3/src/go/internal/gcimporter/gcimporter.go#L38-L74)

実装例としてimage loaderを作って置きます。`"image"` dirの下のディレクトリにある`name`の画像を`.png`と思ってロードして`image.Image`として返します。
`sync.Map`を使ってconcurrent-safeなキャッシュを実装できます。

この例ではエラーも永続化してしまいます。実用するなら`fs.ErrNotExist`以外のエラー時はキーを消すとか必要なんですがexampleなので深く考ないことにします。
実際には何かしらのLRU,LFU実装(max-ageをサポートしているとなおよい)をキャッシュとしたほうが良いかもしれません。

```go
var cache sync.Map

func loadImage(name string) (image.Image, error) {
	v, ok := cache.Load(name)
	if !ok {
		v, _ = cache.LoadOrStore(
			name,
			sync.OnceValues(func() (image.Image, error) {
				f, err := os.Open(filepath.Join("image", name))
				if err != nil {
					return nil, err
				}
				return png.Decode(f)
			}),
		)
	}
	return v.(func() (image.Image, error))()
}
```

#### sync.Condの基本的な使い方

```go
c.L.Lock()
defer c.L.Unlock()
for !condition(someVar) {
	c.Wait()
}
// do some task using someVar...
```

これが基本的な使い方になります。

`c.L.Lock`で保護された何かしらのリソースが`condition`を満たすまで待ち、その後そのリソースを使う、という感じです。

`Wait`は別のgoroutineから[Broadcast](https://pkg.go.dev/sync@go1.22.3#Cond.Broadcast)もしくは、[Signal](https://pkg.go.dev/sync@go1.22.3#Cond.Signal)が呼ばれるまでブロックします。`Broadcast`/`Signal`の言い回しからわかる通り、`Wait`でブロックしているすべての`goroutine`あるいは単一の`goroutine`をそれぞれアンブロックします。

これがなかなか筆者にはピンときませんでした。

`Wait`をインライン展開すると、

```go
c.L.Lock()
defer c.L.Unlock()
for !condition(someVar) {
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}
```

となります。

`runtime_notifyListWait`がランタイムによりブロックされる`Wait`の本体のようなロジックです。見てのとおり、`c.L.Unlock`が呼ばれるので、`Wait`中はロックは解除されています。

#### Example: Cond Wait

[sync.Cond](https://pkg.go.dev/sync@go1.22.3#Cond)の利用例を以下のように実装します。

[pthread_cond_init(3p)](https://man7.org/linux/man-pages/man3/pthread_cond_init.3p.html)のExmpalesセクションで紹介されているものと似ています。
しかしこれを典型と言い切っていいのかはよくわかりません。もっといろいろな使い方ができますからね。

状態を変数として持つ`condWorker`は`do`メソッドで特定のstate `doIf`の時のみアクション`f func()`を実行しますが、それ以外にも`waitIf`にマッチする場合は`sync.Cond`を利用してその状態になるか、もしくは別の状態になるまで待ちます。`waitIf`がマッチするstateは十分に短い時間で別のstateに遷移することがよく知られています。
こういうケースでは`chan`や`mutex`を使うよりシンプルに問題を解けているはず・・・たぶん。

実際のコードでは`changeState`メソッドではなく`*condWorker`自体がいろいろな仕事を自らするなり、メソッド呼び出しで依頼されるなりすることで内部の`state`が切り替わるはずです。
何かの仕事`do`を行う際に、それをしてもよい状態のみならず、待つこと許される状態というのを定義できると便利だと思います。

ただし`sync.Cond`で`Wait`している間`context.Context`のcancellationを受けとるなどできませんから、`sync.Mutex`と同じで場合によって長い時間ブロックすることもあり得ます。
そのためロック期間を十分認識した設計が必要です。

[playground](https://go.dev/play/p/3DP_UvkrAk6)

```go
package main

import (
	"errors"
	"fmt"
	"slices"
	"sync"
	"time"
)

var (
	ErrNotEligibleState = errors.New("not eligible state")
)

type state int

const (
	stateA state = iota + 1
	stateB
	stateC
)

type condWorker struct {
	s    state
	cond *sync.Cond
}

func newCondWorker() *condWorker {
	return &condWorker{
		s:    stateA,
		cond: sync.NewCond(&sync.Mutex{}),
	}
}

func (w *condWorker) changeState(s state) {
	w.cond.L.Lock()
	defer w.cond.L.Unlock()
	w.cond.Broadcast()
	w.s = s
}

func (w *condWorker) do(doIf state, waitIf func(state) bool, f func()) error {
	w.cond.L.Lock()
	defer w.cond.L.Unlock()
	if w.s == doIf {
		f()
		return nil
	}
	if waitIf == nil || !waitIf(w.s) {
		return ErrNotEligibleState
	}
	for {
		w.cond.Wait() // lock is freed while being blocked on Wait
		// lock is now held.
		if w.s == doIf {
			break
		}
		if !waitIf(w.s) {
			return ErrNotEligibleState
		}
	}
	f()
	return nil
}

type varSet struct {
	doIf   state
	waitIf []state
}

func main() {
	w := newCondWorker()
	sChan := make(chan state)
	doChan := make(chan varSet)

	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		defer wg.Done()
		for s := range sChan {
			fmt.Printf("changing state to %d\n", s)
			w.changeState(s)
			fmt.Printf("changed state to %d\n", s)
		}
	}()

	wg.Add(1)
	go func() {
		defer wg.Done()
		for v := range doChan {
			fmt.Printf("doing if %d, would wait if s is one of %v\n", v.doIf, v.waitIf)
			err := w.do(
				v.doIf,
				func(s state) bool { return slices.Contains(v.waitIf, s) },
				func() { fmt.Println("...working...") },
			)
			fmt.Printf("done with %v\n", err)
		}
	}()
	doChan <- varSet{doIf: stateA}
	/*
		doing if 1, would wait if s is one of []
		...working...
		done with <nil>
	*/
	doChan <- varSet{doIf: stateB}
	/*
		doing if 2, would wait if s is one of []
		done with not eligible state
	*/
	doChan <- varSet{doIf: stateB, waitIf: []state{stateA}}
	/*
		doing if 2, would wait if s is one of [1]
	*/
	fmt.Println("sleeping...")
	time.Sleep(time.Millisecond)
	fmt.Println("woke up!")
	sChan <- stateB
	/*
		changing state to 2
		changed state to 2
		...working...
		done with <nil>
	*/
	doChan <- varSet{doIf: stateC, waitIf: []state{stateB}}
	/*
		doing if 3, would wait if s is one of [2]
	*/
	fmt.Println("sleeping...")
	time.Sleep(time.Millisecond)
	fmt.Println("woke up!")
	sChan <- stateA
	/*
		changing state to 1
		changed state to 1
		done with not eligible state
	*/
	close(sChan)
	close(doChan)
	wg.Wait()
}
```

### sub-repositoryのsync

https://pkg.go.dev/golang.org/x/sync

sub-repositoryにもsyncが実装されています。

#### errgroup.Group

https://pkg.go.dev/golang.org/x/sync@v0.7.0/errgroup#Group

`errgroup.Group`はざっくりいうと以下のパターンをライブラリ化したものです。

(何度も書きますが`Go1.22.0`以降でないと動かきません)

[snippet](https://github.com/ngicks/go-basics-example/blob/main/snipet/errgroup-example/main.go)

```go
func tasksNaiveGroup(ctx context.Context, tasks []task, work func(ctx context.Context, t task) error) error {
	var wg sync.WaitGroup
	ctx, cancel := context.WithCancelCause(ctx)
	defer cancel(nil)

	sem := make(chan struct{}, 15)

	wg.Add(len(tasks))
	for _, t := range tasks {
		sem <- struct{}{}
		go func() {
			defer func() {
				wg.Done()
				<-sem
			}()
			e := work(ctx, t)
			if e != nil {
				cancel(e)
			}
		}()
	}
	wg.Wait()
	return context.Cause(ctx)
}
```

これを`errgroup.Group`を使うように変更すると以下のようになります。

```go
func tasksErrgroup(ctx context.Context, tasks []task, work func(ctx context.Context, t task) error) error {
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(15)
	for _, t := range tasks {
		g.Go(func() error {
			return work(ctx, t)
		})
	}
	return g.Wait()
}
```

- [(\*errgroup.Group).Go](https://pkg.go.dev/golang.org/x/sync@v0.7.0/errgroup#Group.Go)で任意の関数を新しいgoroutineの中で実行できます。
- [(\*errgroup.Group).SetLimit](https://pkg.go.dev/golang.org/x/sync@v0.7.0/errgroup#Group.SetLimit)で同時に実行するgoroutineの数を制限でき、
- [(\*errgroup.Group).Wait](https://pkg.go.dev/golang.org/x/sync@v0.7.0/errgroup#Group.Wait)で実行されたすべての関数が終わるまで待ち、
  - `Go`に渡された関数が最初に返したnon-nil errorを返します。
- [errgroup.WithContext](https://pkg.go.dev/golang.org/x/sync@v0.7.0/errgroup#WithContext)で`*errgroup.Group`と親contextを受け継いだ`context.Context`を返します
  - この`context.Context`は`Go`に渡された関数のエラーでcancelされます。
  - これによって`Go`に渡す関数をキャンセルできるようにするとエラーで全体を中断できます。

めちゃ便利ですね。これはすごく多用します。
しいて言えば`Go`に渡す関数が`panic`したときの手当てが特にまだ実装されていない(`v0.7.0`時点)ので、渡す関数の中で`recover`して`g.Go`を呼び出している`goroutine`で`re-panic`するように呼び出し側で工夫する必要があります。

以下のような感じでしょうか

```go

func tasksErrgroupRepanic(ctx context.Context, tasks []task, work func(ctx context.Context, t task) error) error {
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(15)
	var (
		panicOnce sync.Once
		panicked  any
	)
	for _, t := range tasks {
		g.Go(func() error {
			var err error
			defer func() {
				rec := recover()
				if rec == nil {
					return
				}
				var set bool
				panicOnce.Do(func() {
					set = true
					panicked = rec
				})
				if set {
					err = fmt.Errorf("panicked: %v", rec)
				}
			}()
			err = work(ctx, t)
			return err
		})
	}
	err := g.Wait()
	if panicked != nil {
		panic(panicked)
	}
	return err
}
```

panicよる強制終了は他のgoroutineのdeferを実行せずに終わってしまいます。
panicは伝搬させられるならなるだけしたほうが良いと思います。
とはいえ、筆者の体感上panicはほとんど遭遇しないので困る機会は少ないかもしれません。

#### semaphore.Weighted

https://pkg.go.dev/golang.org/x/sync@v0.7.0/semaphore#Weighted

token量で制限する[semaphore](<https://en.wikipedia.org/wiki/Semaphore_(programming)>)です。
対象読者には`semaphore`という概念になじみないでしょうか？`semaphore`は一般的に`mutex`と同じくリソースにアクセスできる`thread`の数を制限するものです。`mutex`がロック区間にたった一つの`thread`しかアクセスできないようにするのに対し、`counting semaphore`は任意の数`n`までに制限します。

`semaphore.Weighted`は`weighted semaphore`なので任意のtoken量`n`に対して、[(\*semaphore.Weighted).Acquire](https://pkg.go.dev/golang.org/x/sync@v0.7.0/semaphore#Weighted.Acquire)は任意の`m`を取得します。もし現在`m`個のtokenが利用可能でなければ、利用可能になるまでブロックし続けます。

#### singleflight.Group

https://pkg.go.dev/golang.org/x/sync@v0.7.0/singleflight#Group

[(\*singleflight.Group).Do](https://pkg.go.dev/golang.org/x/sync@v0.7.0/singleflight#Group.Do)にkeyと関数を受け取り実行します。
同じkeyに対して同時に複数の呼び出しがあった場合、2つ目以降の呼び出しは実際に関数を実行せずに1つ目の実行の結果を待って同じ結果をえます。

keyの型がstringだけなので、重い関数の呼び出しパラメータをkeyにしたい場合一旦パラメータをシリアライズする必要があります。
`json.Marshal`はほとんどのケースでstableな結果を返すので普通は`json.Marshal`で必要十分であると思われます。

:::details json.Marshalがstableじゃないときはいつか

筆者が思いつくのは以下の時

- `MarshalJSON() ([]byte, error)`を実装している型が含まれ、その実装がstableでない
- `nil map`と`map[T]U{}`が意味的に同じ(≒遅延初期化される)とき
  - `nil map`は`null`が出力される
  - `assignment to entry in nil map`のpanicがあるのでこれはあんまりないかもしれません。
- `nil slice`と`[]T{}`が意味的に同じ(≒遅延初期化される)とき
  - `[]T`が`nil`のとき`null`が出力される
  - `nil slice`は`append`, `len`, `cap`すべて機能するのでこのケースは多い気がします。
- `Go`バージョンをまたいだことで`json.Marshal`がunicode sequenceにescapeする文字種が変わったとき
  - 確か1度はescapeする文字種が変わっていたと思います。確認取れたらこの行は編集されます。
  - 別システムから来たjsonを未編集で使ったり、ファイルにキャッシュしている場合などで起きます。
    - つまり普通は起きません。

`for k, v := range map[T]U{} {}`したとき順序がランダムになるのは`Go`を書いているならばよくご存じでしょうが、`json.Marshal`はmapのkeyをすべて取ってsortする処理が挟まるので、ここに関してはstableな出力を得られます。

:::

使用例としては[SNMP](https://en.wikipedia.org/wiki/Simple_Network_Management_Protocol)などでUDPのbroadcastを送信してn秒待つような検索処理は`singleflight`で処理できるとよいでしょう。というか筆者は[Node.js]でそういうものを実装したことがあります。[Node.js]では`std`に近いライブラリに`singleflight`に当たるものがなくて困ったことがあります。

## signal handling

サーバープログラムは通常明示的に終了されるまで動作し続けます。
終了は大抵の場合`SIGTERM`などのシグナルによってされることが多いです。

`Go`におけるsignalはすべてruntimeによってキャッチされます。
つまり下記の「pid 1だとシグナル受け取れない問題」が起きません。

> https://man7.org/linux/man-pages/man2/kill.2.html#NOTES
>
> The only signals that can be sent to process ID 1, the init
> process, are those for which init has explicitly installed signal
> handlers.

runtimeによってキャッチされたsignalは`os/signal`パッケージによって受け取ることができます。

https://pkg.go.dev/os/signal@go1.22.3

色々注意点が述べられているので上記ドキュメントは呼んでおくほうがよいでしょう。

[(os/signal).Notify](https://pkg.go.dev/os/signal@go1.22.3#Notify)によってsignalをchannelに通知します。
[(os/signal).NotifyContext](https://pkg.go.dev/os/signal@go1.22.3#NotifyContext)で、signalを受けたときcancelされる`context.Context`を得られます。

```go
// buffer-sizeはとりあえずn > 0ならなんでもよい。
// channelのバッファが埋まるとsignalは捨てられる(Go1.22.3時点)。
// signalの使い方によっては何度signalされたかも重要なはず。その場合は適当なサイズまで増量する。
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, os.Interrupt)
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM)
```

`Notify`/`NotifyContext`のvariadicな第二引数で受け取るsignalを指定できますが、ここに何も渡さないとすべてのsignalを受け取れます。
しかし基本的にはこうしません。
なぜなら、`unix`では[preemptiveなスケジューリングのためにSIGURGを使用する](https://go.googlesource.com/proposal/+/master/design/24543-non-cooperative-preemption.md)ので、ノイズとなる[syscall.SIGURG](https://pkg.go.dev/syscall@go1.22.3#SIGURG)をたくさん受け取ってしまうためです。
とりたいsignalだけを指定するようにするとよいでしょう。

終了を通知されたいだけのケースの場合、とりあえずは以下二つだけを受けとっておけば困らないと思います

- `syscall.SIGTERM`
  - [Docker](https://docs.docker.com)などを使う場合、[特に指定しないとSIGTERMが送られてくる](https://docs.docker.com/reference/dockerfile/#stopsignal)ためです。
  - [windowsでもsyscallパッケージで定義される](https://github.com/golang/go/blob/go1.22.3/src/syscall/types_windows.go#L64)ので使っても安全です
- `os.Interrupt`
  - `unix`では`syscall.SIGINT`のエイリアス
  - CLIから実行ファイルを実行した場合は`Ctrl+C`で終了をしようとする人が多いと思います。この場合`SIGINT`が送られます。

## おわりに

- [part1 プロジェクトを始めるまで編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part1)
- [part2 cliアプリをつくれるところまで編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part2)
- part3 concurrent GO編: これ
- [part4 HTTP server/logger編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part4)

[Go]: https://go.dev/
[C++]: https://en.wikipedia.org/wiki/C%2B%2B
[Node.js]: https://nodejs.org/en
[TypeScript]: https://www.typescriptlang.org/
[python]: https://www.python.org/
[Rust]: https://www.rust-lang.org
[The Rust Programming Language 日本語]: https://doc.rust-jp.rs/book-ja/
[Visual Studio Code]: https://code.visualstudio.com/
[vscode]: https://code.visualstudio.com/
[git]: https://git-scm.com/
