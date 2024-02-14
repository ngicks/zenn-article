---
title: "Goでstdin/stdoutごしにgRPCする"
emoji: "🤙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

# Overview

本記事では

プログラムからほかのプログラムを呼び出す方法として

- サブプロセスとしてプログラムを呼び出し、
- より環境への依存性が少ない方法としてstdin/stdoutを介した通信し、
- ネットワーク透過な方法としてgRPCを用いる

ことを実現できることを確認します。

# 前提知識

- プログラミングに対する一定の理解を有する
- [Go]の文法を理解している

# Background

アプリケーションを開発しているとき、そのアプリケーションとは別の言語で作られた資産を活用したくなる時がしばしばあります。

そもそも普通にプログラムをlinuxなどでビルドしてリンクされる[libc](https://en.wikipedia.org/wiki/C_standard_library)はC言語で書かれたライブラリです。

以下の標準出力に`"foobar"`と出力するだけのプログラムを`cargo`でビルドすると、

```rust
fn main() {
    println!("foobar")
}
```

```
# cargo build --release
# ldd target/release/main
        linux-vdso.so.1 (0x00007ffc1a952000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f1ff6b3e000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f1ff6916000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f1ff6bc0000)
```

という感じで`libc`がリンクされます。

それ以外の例でいえば、[PDFium](https://pdfium.googlesource.com/pdfium/), [Libre Office](https://github.com/LibreOffice)などはほかの言語からよく利用されていると思います。

- [pdfium-render](https://github.com/ajrcarey/pdfium-render)
- [Pdfium.NET SDK](https://pdfium.patagames.com/help/html/Welcome-to-the-Pdfium-NET-SDK.htm)
- [pypdfium2](https://pypi.org/project/pypdfium2/1.0.0/)
- [libreoffice-rs](https://github.com/undeflife/libreoffice-rs)
- [github.com/dveselov/go-libreofficekit](https://github.com/dveselov/go-libreofficekit)

こういったほか言語で書かれたプログラムを呼び出すのを[ffi]などと呼び、呼び出しのラッパーライブラリのことを`binding`などと呼んだりします。

静的にコンパイルされる言語は

例えば

- 独立した実行ファイルとして各部をビルドし、サブプロセスとしてそれらを起動したうえで何かしらの方法で通信する
- [ffi]を用いてプログラムから直接関数を呼び出す
- [WASM]を用いる

などがあるかと思います。

- [ffi](Foreign function interface)は読んで字のごとく、ほかのプログラミング言語で書かれたプログラムを呼び出すための機構のことです。
  - この記事に限ってはプラットフォームの`C ABI`を利用して`.dll`や`.so`ファイルから関数を呼び出すなどすることを指すとします。
- [WASM]はアーキテクチャ、OS非依存なバイナリインストラクションフォーマットやその他もろもろのことであり、スタックベースのVMの上で動作します。
  - [Rust]や[Go]など、複数の言語からビルド可能です。
  - VMが[Cranelift]の`JIT compilation`などで実行コードを生成することでNear-native speedで動作するとされています。
  - もとはWebブラウザ上で動作するバイナリフォーマットを目指していたのでWeb Assemblyという名前になっています。
    - `Chrome`や`Firefox`など主要なブラウザはWASMを実行可能です([参考](https://www.publickey1.jp/blog/23/firefox_120webassemblywasmgcchrome.html))。
    - ブラウザと無関係なVMの実装もいくつも出ています([wasmer], [wasmtime], [wazero]など)。

これらにも当然一長一短があり、

- サブプロセス
  - 長所: ビルドプロセスが単純。
  - 短所: プロセス起動のオーバーヘッドが高い。通信方法の整備が必要。
- [ffi]
  - 長所: 比較的高速。小さな粒度での呼び出しが可能。
  - 短所: [cross compilation](https://en.wikipedia.org/wiki/Cross_compiler)が困難になることがある。
- [WASM]
  - 長所: 安全。プログラムがホストシステムへできることを完全にコントロールできる
  - 短所: 既に存在するプログラムのすべてをビルドしてそのまま動かせるわけではない。[WASI Preview 2](https://github.com/WebAssembly/WASI/pull/577#issuecomment-1910711171)は仕様が正式化したばかりで実相が追い付いているとは限らない。

などあります。

書き出せばこれ以上に長所短所があると思いますので、最適な方法を決定する際にはもちろん読者自身の調査と判断によって行ってください。

ほかのプログラムをcliのコマンドとして直接呼び出す例として典型的なものとしてここで`git`コマンドを上げます

例えば、

- vscodeのgit extension
  - https://github.com/microsoft/vscode/blob/8edcc2d9d83926ec73932b05e13eeafbe330ca32/extensions/git/src/git.ts#L668
  - https://github.com/microsoft/vscode/blob/8edcc2d9d83926ec73932b05e13eeafbe330ca32/extensions/git/src/git.ts#L583
  - https://github.com/microsoft/vscode/blob/8edcc2d9d83926ec73932b05e13eeafbe330ca32/extensions/git/src/git.ts#L561
  - https://github.com/microsoft/vscode/blob/8edcc2d9d83926ec73932b05e13eeafbe330ca32/extensions/git/src/git.ts#L2032
- go get
  - https://github.com/golang/go/blob/e17e5308fd5a26da5702d16cc837ee77cdb30ab6/src/cmd/go/internal/modfetch/codehost/git.go#L249

`git`コマンドがそのまま呼び出されるのは環境のセットアップ、例えば`pass`プログラムや`wincred`などのとの連携がそのまま持ち込めるので都合がいいからだと思われます。

# 実装

## fakeconn

```go
package fakeconn

import (
	"context"
	"io"
	"net"
	"sync"
	"time"
)

var _ net.Conn = (*FakeConn)(nil)

// FakeConn is a faked connection which reads from a reader and writes to a writer.
type FakeConn struct {
	name    string
	rFeeder *feeder
	wFeeder *feeder
	cancel  func()
}

func New(name string, r io.Reader, w io.Writer) *FakeConn {
	ctx, cancel := context.WithCancel(context.Background())
	return &FakeConn{
		name:    name,
		rFeeder: newFeeder(ctx, r.Read),
		wFeeder: newFeeder(ctx, w.Write),
		cancel:  cancel,
	}
}

// Run runs FakeConn and blocks the current goroutine until c is closed.
// Calling Run twice or more causes undefined behavior.
//
// Run starts 2 additional goroutines.
func (c *FakeConn) Run() {
	var wg sync.WaitGroup
	wg.Add(2)
	go func() {
		defer wg.Done()
		c.rFeeder.run()
	}()
	go func() {
		defer wg.Done()
		c.wFeeder.run()
	}()
	wg.Wait()
}

func (c *FakeConn) Read(b []byte) (n int, err error)  { return c.rFeeder.do(b) }
func (c *FakeConn) Write(b []byte) (n int, err error) { return c.wFeeder.do(b) }
func (c *FakeConn) Close() error {
	c.cancel()
	return nil
}
func (c *FakeConn) LocalAddr() net.Addr                { return fakeAddr(c.name) }
func (c *FakeConn) RemoteAddr() net.Addr               { return fakeAddr(c.name) }
func (c *FakeConn) SetDeadline(t time.Time) error      { return nil } // TODO: maybe implement?
func (c *FakeConn) SetReadDeadline(t time.Time) error  { return nil } // TODO: maybe implement?
func (c *FakeConn) SetWriteDeadline(t time.Time) error { return nil } // TODO: maybe implement?

// feeder serializes io operations.
type feeder struct {
	ctx      context.Context
	fn       func(b []byte) (int, error)
	bufCh    chan []byte
	resultCh chan feederResult
}

type feederResult struct {
	n   int
	err error
}

func newFeeder(ctx context.Context, fn func(b []byte) (int, error)) *feeder {
	return &feeder{
		ctx:      ctx,
		fn:       fn,
		bufCh:    make(chan []byte),
		resultCh: make(chan feederResult),
	}
}

func (f *feeder) do(b []byte) (n int, err error) {
	select {
	case f.bufCh <- b:
	case <-f.ctx.Done():
		return 0, io.EOF
	}
	select {
	case result := <-f.resultCh:
		return result.n, result.err
	case <-f.ctx.Done():
		return 0, io.EOF
	}
}

func (f *feeder) run() {
	for {
		var buf []byte
		select {
		case <-f.ctx.Done():
			return
		case buf = <-f.bufCh:
		}
		n, err := f.fn(buf)
		select {
		case <-f.ctx.Done():
			return
		case f.resultCh <- feederResult{n, err}:
		}
	}
}

var _ net.Addr = fakeAddr("")

type fakeAddr string

func (a fakeAddr) Network() string { return "fake" }
func (a fakeAddr) String() string  { return string(a) }

```

## fakelistener

```go
package fakeconn

import (
	"net"
	"slices"
	"sync"
)

var _ net.Listener = (*FakeListener)(nil)

type FakeListener struct {
	addr      net.Addr
	mu        sync.Mutex
	done      chan struct{}
	closeOnce func()
	updateCh  chan struct{}
	queue     []net.Conn
}

// NewFakeListener returns a newly allocated *FakeListener.
func NewFakeListener(addr net.Addr) *FakeListener {
	done := make(chan struct{})
	return &FakeListener{
		addr:      addr,
		done:      done,
		closeOnce: sync.OnceFunc(func() { close(done) }),
		updateCh:  make(chan struct{}),
	}
}

// AddConn adds
func (l *FakeListener) AddConn(conn net.Conn) {
	l.mu.Lock()
	defer l.mu.Unlock()
	l.queue = append(l.queue, conn)
	l.notify()
}

func (l *FakeListener) notifier() chan struct{} {
	l.mu.Lock()
	defer l.mu.Unlock()
	return l.updateCh
}

func (l *FakeListener) notify() {
	ch := l.updateCh
	l.updateCh = make(chan struct{})
	close(ch)
}

func (l *FakeListener) Accept() (net.Conn, error) {
	l.mu.Lock()
	select {
	case <-l.done:
		l.mu.Unlock()
		// TODO: use a more appropriate error
		return nil, net.ErrClosed
	default:
	}
	if len(l.queue) > 0 {
		defer l.mu.Unlock()
		return l.consumeOne()
	}
	l.mu.Unlock()

	for {
		select {
		case <-l.done:
			return nil, net.ErrClosed
		case <-l.notifier():
			l.mu.Lock()
			if len(l.queue) > 0 {
				defer l.mu.Unlock()
				return l.consumeOne()
			}
			l.mu.Unlock()
		}
	}
}

func (l *FakeListener) consumeOne() (net.Conn, error) {
	conn := l.queue[0]
	l.queue = slices.Delete(l.queue, 0, 1)
	return conn, nil
}

// Close closes the listener.
// Any blocked Accept operations will be unblocked and return errors.
func (l *FakeListener) Close() error {
	l.mu.Lock()
	defer l.mu.Unlock()
	l.closeOnce()
	return nil
}

// Addr returns the listener's network address.
func (l *FakeListener) Addr() net.Addr {
	return l.addr
}
```

## client

```go
package main

import (
	"bufio"
	"context"
	"errors"
	"flag"
	"fmt"
	"io"
	"net"
	"os"
	"os/exec"
	"os/signal"
	"runtime"
	"sync"
	"syscall"
	"time"

	"github.com/golang/protobuf/ptypes/wrappers"
	"github.com/ngicks/example-grpc-over-file/api/echoer"
	"github.com/ngicks/example-grpc-over-file/internal/fakeconn"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/protobuf/types/known/anypb"
)

var (
	p          = flag.Bool("p", false, "whether to use stdio or os.Pipe")
	subCmdPath = flag.String("c", "./sub", "path to built sub command")
)

func main() {
	flag.Parse()

	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()

	cmdCtx, cmdCancel := context.WithCancel(ctx)
	defer cmdCancel()
	cmd := subCmd(cmdCtx, *subCmdPath, *p)
	stderr, _ := cmd.StderrPipe()

	var (
		r io.ReadCloser
		w io.WriteCloser
	)
	if *p {
		pr1, pw1, err := os.Pipe()
		if err != nil {
			panic(err)
		}
		pr2, pw2, err := os.Pipe()
		if err != nil {
			panic(err)
		}
		defer func() {
			for _, f := range []*os.File{pr1, pw1, pr2, pw2} {
				_ = f.Close()
			}
		}()
		cmd.ExtraFiles = []*os.File{pr1, pw2}
		r, w = pr2, pw1
	} else {
		r, _ = cmd.StdoutPipe()
		w, _ = cmd.StdinPipe()
	}

	go func() {
		scanner := bufio.NewScanner(stderr)
		for ctx.Err() == nil && scanner.Scan() {
			fmt.Printf("cmd stderr: %s\n", scanner.Text())
		}
		err := scanner.Err()
		if err != nil && !errors.Is(err, ctx.Err()) {
			fmt.Printf("cmd err: %s\n", err)
		}
	}()

	err := cmd.Start()
	if err != nil {
		panic(err)
	}
	defer func() { _ = cmd.Wait() }()

	fakeConn := fakeconn.New("fake", r, w)
	go fakeConn.Run()
	defer func() { _ = fakeConn.Close() }()

	conn, err := grpc.DialContext(
		ctx,
		"",
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		grpc.WithContextDialer(func(ctx context.Context, s string) (net.Conn, error) {
			return fakeConn, nil
		}),
	)
	if err != nil {
		panic(err)
	}
	defer conn.Close()

	c := echoer.NewEchoerClient(conn)

	runCtx, runCancel := context.WithTimeout(ctx, 30*time.Second)
	defer runCancel()

	stream, err := c.Echo(runCtx)
	if err != nil {
		panic(err)
	}

	closeOnce := sync.OnceValue(func() error { return stream.CloseSend() })
	defer func() { _ = closeOnce() }()

	for _, msg := range []string{"foo", "bar", "baz"} {
		fmt.Printf("sending %s\n", msg)
		err = stream.Send(&echoer.EchoRequest{Payload: must(anypb.New(&wrappers.StringValue{Value: msg}))})
		if err != nil {
			panic(err)
		}
		fmt.Printf("sent %s\n", msg)
		fmt.Printf("receiving %s\n", msg)
		resp, err := stream.Recv()
		if err != nil {
			panic(err)
		}
		fmt.Printf("received %s\n", msg)
		fmt.Printf("seq = %d, payload = %s\n", resp.GetSeq(), resp.GetPayload().String())
	}

	if err := closeOnce(); err != nil {
		panic(err)
	}
	fmt.Printf("closed\n")

	if runtime.GOOS == "windows" {
		// I'm not sure why but sending SIGTERM on the process blocks long on Windows.
		err = cmd.Process.Kill()
	} else {
		err = cmd.Process.Signal(syscall.SIGTERM)
	}
	if err != nil {
		panic(err)
	}
	err = cmd.Wait()
	fmt.Printf("wait error: %v\n", err)
	fmt.Printf("exit code = %d\n", cmd.ProcessState.ExitCode())
}

func must[V any](v V, err error) V {
	if err != nil {
		panic(err)
	}
	return v
}

func subCmd(ctx context.Context, cmdPath string, usePipe bool) *exec.Cmd {
	args := []string{}
	if usePipe {
		args = append(args, []string{"-r", "3", "-w", "4"}...)
	}
	return exec.CommandContext(ctx, cmdPath, args...)
}
```

## server

```go
package main

import (
	"context"
	"errors"
	"flag"
	"fmt"
	"os"
	"os/signal"
	"sync/atomic"
	"syscall"

	"github.com/ngicks/example-grpc-over-file/api/echoer"
	"github.com/ngicks/example-grpc-over-file/internal/fakeconn"
	"google.golang.org/grpc"
)

var (
	r = flag.Int("r", -1, "fd for read. defaults to stdin")
	w = flag.Int("w", -1, "fd for write. defaults to stdout")
)

type server struct {
	printf func(format string, args ...any)
	seq    atomic.Int64
	echoer.UnimplementedEchoerServer
}

func (s *server) Echo(req echoer.Echoer_EchoServer) error {
	s.printf("receiving on echo method\n")
	defer func() {
		s.printf("echo exiting\n")
	}()

	for req.Context().Err() == nil {
		s.printf("receiving on msg\n")
		msg, err := req.Recv()
		if err != nil {
			if errors.Is(err, req.Context().Err()) {
				err = nil
			}
			return err
		}
		s.printf("received on msg\n")
		newSeq := s.seq.Add(1)
		payload := msg.GetPayload()
		err = req.Send(&echoer.EchoResponse{Seq: newSeq, Payload: payload})
		if err != nil {
			if errors.Is(err, req.Context().Err()) {
				err = nil
			}
			return err
		}
	}
	return nil
}

func main() {
	flag.Parse()

	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()

	// logFile, err := os.OpenFile("./log", os.O_APPEND|os.O_CREATE|os.O_RDWR, fs.ModePerm)
	// if err != nil {
	// 	panic(err)
	// }
	// defer func() {
	// 	_ = logFile.Sync()
	// 	_ = logFile.Close()
	// }()

	log := func(s string, args ...any) {
		// _, _ = fmt.Fprintf(logFile, s, args...)
		_, _ = fmt.Fprintf(os.Stderr, s, args...)
	}

	var rFile, wFile *os.File
	if *r < 0 {
		rFile = os.Stdin
	} else {
		rFile = os.NewFile(uintptr(*r), "in")
	}

	if *w < 0 {
		wFile = os.Stdout
	} else {
		wFile = os.NewFile(uintptr(*w), "out")
	}

	defer func() {
		_ = rFile.Close()
		_ = wFile.Close()
	}()

	log("cmd staring, r = %v, w = %v\n", rFile.Fd(), wFile.Fd())

	fakeConn := fakeconn.New("sub-fake", rFile, wFile)
	go fakeConn.Run()
	defer func() { _ = fakeConn.Close() }()

	fakeListener := fakeconn.NewFakeListener(fakeConn.LocalAddr())
	fakeListener.AddConn(fakeConn)

	s := grpc.NewServer()
	echoer.RegisterEchoerServer(s, &server{printf: log})

	go func() {
		<-ctx.Done()
		log("context signaled\n")
		_ = fakeListener.Close()
		s.Stop()
	}()

	if err := s.Serve(fakeListener); err != nil {
		log("serve error = %v\n", err)
	}
	log("done\n")
}
```

[Rust]: https://www.rust-lang.org/
[Go]: https://go.dev/
[ffi]: https://en.wikipedia.org/wiki/Foreign_function_interface
[wasm]: https://webassembly.org/
[Cranelift]: https://cranelift.dev/
[wasmer]: https://wasmer.io/
[wasmtime]: https://wasmtime.dev/
[wazero]: https://wazero.io/
