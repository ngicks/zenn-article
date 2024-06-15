---
title: "Goで開発して3年のプラクティスまとめ(4/4): HTTP Server/logger編"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## Goで開発して3年のプラクティスまとめ(4/4): HTTP Server/logger編

yet another入門記事です。

1記事にまとめようとしたら8万文字を超えて怒られたのでいくつかに分割しています。

- [part1 プロジェクトを始めるまで編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part1)
- [part2 cliアプリをつくれるところまで編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part2)
- [part3 concurrent GO編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part3)
- part4 HTTP Server/logger編: これ
- part5以降・・・未定(_unlikely_)

ご質問やご指摘がございましたらこの記事のコメントでお願いします。
(ほかの媒体やリンク先に書かれた場合、筆者は気付きません)

## Overview

- `net/http`のClient / Serverの書き方を紹介します
  - client
    - サンプルとして`multipart/form-data`を送信します
    - Clientをカスタマイズする方法に触れます
  - server
    - stdのみでhttp serverを実装してみます
    - [github.com/labstack/echo](https://github.com/labstack/echo)を使ってstdノミバージョンを書きなおします
- ロギングライブラリについて紹介します。
  - structured loggingのみに絞ります

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

## 対象環境

- メインは`linux/amd64`です
- 別段`OS`/`arch`固有な要素は少ないと思いますが、適宜読み替えてほしいです。

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

## HTTP client / server

現実的なソフトウェアを作るうえでHTTPを通じて通信を行うことは非常に多いと思います。

最近では[gRPC](https://grpc.io/)を使うことも多いかもしれません(と言いつつ筆者は製品で使ったことはありませんが)

### client

[net/http]: https://pkg.go.dev/net/http@go1.22.3

https://pkg.go.dev/net/http@go1.22.3#pkg-overview

上記のoverviewにさっそく書いてありますが、http requestを送るためには

- [Get](https://pkg.go.dev/net/http@go1.22.3#Get)
- [Head](https://pkg.go.dev/net/http@go1.22.3#Head)
- [Post](https://pkg.go.dev/net/http@go1.22.3#Post)
- [PostForm](https://pkg.go.dev/net/http@go1.22.3#PostForm)

のいずれかを使うことができます。

ソースを見れば一発で分かりますが、これらは[http.DefaultClient](https://pkg.go.dev/net/http@go1.22.3#DefaultClient)の同名メソッド呼び出しのショートハンドです。
簡単なcliアプリなら上記を使うのが簡単でいいですが、実際にはいろいろ困るケースがあります。

実際にある困ったケースを先に上げます

- cookieの保存が必要なった
- ctxにセットされているRequest-idをリクエストに引き渡す機能を付け足したくなった(後述)

そのため、なにかのclientを作る(e.g. ダウンローダーなど)ときはなるだけ[\*http.Client](https://pkg.go.dev/net/http@go1.22.3#Client)を引数にとるようにしたほうが良いといえるでしょう。

cookieの保存は`Client`の`Jar`フィールドに[\*(net/http/cookiejar).Jar](https://pkg.go.dev/net/http/cookiejar@go1.22.3#Jar)インスタンスを渡せば大体のケースでokです。`Jar`フィールドは[CookieJar](https://pkg.go.dev/net/http@go1.22.3#CookieJar)というinterfaceなので挙動を変えたい場合は上記`net/http/cookiejar`実装をラップするなりします。

#### \*http.Clientを引数として受け取る

例えばその構造体が以下であるとして

```go
type Downloader struct {
	// ... fields ...
}

// Download downloads anything from given url into specified storage.
// If the target server url points to accepts Range requests
// Downloader tries to ...(and anything like that)...
func (d *Downloader) Download(ctx context.Context, url *urlpkg.URL) error {
	// ...implementation...
}
```

おそらくフィールドを隠したいので`New`関数を作ると思います

```go
func New() *Downloader {
	return &Downloader{
		// ... initialization ...
	}
}
```

ここで`*http.Client`を受けとっておくとよいという話です。

```go
func New(client *http.Client) *Downloader {
	return &Downloader{
		client: client,
		// ... rest of initialization ...
	}
}
```

実際には[Functional options pattern](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html)を使うほうが良いかもしれません。
`http.DefaultClient`で十分っていうケースもありうるので、デフォルト値を持たせるためです。

```go
type option interface {
	apply(*Downloader)
}

type funcOpt func(*Downloader)

func(o funcOpt) apply(d *Downloader) {
	o(d)
}

func WithClient(client *http.Client) option {
	return funcOpt(func(d *Downloader) {
		d.client = client
	})
}

func New(opts ...option) *Downloader {
	d := &Downloader{
		client: http.DefaultClient,
		// ... rest of initialization ...
	}
	for _, o := range opts {
		o(d)
	}
	return d
}
```

`*http.Client`の[Do](https://pkg.go.dev/net/http@go1.22.3#Client.Do)メソッド以外を呼び出すつもりもないので、`HTTPRequestDoer`を定義してそれを受けとるほうが良いかもしれません。

(筆者はacronymも頭文字以外は小文字にしたほうがいいよ派ですが、この記事中では全部大文字にしています)

```go
type HTTPRequestDoer interface {
	Do(req *http.Request) (*http.Response, error)
}

func WithHTTPRequestDoer(doer HTTPRequestDoer) option {
	return funcOpt(func(d *Downloader) {
		d.doer = doer
	})
}

c := &http.Client{/*...*/}
New(WithHTTPRequestDoer(c))
```

一般にdata raceを避けるために引数に受け取った`*http.Client`のフィールドに対してwriteを行うのはお勧めできない行為なので型の詳細はいりませんし、`Doer`のほうがtest doubleを用意しやすいのでこちらのほうが良いケースもたくさんあるでしょう。

#### Requestを送る(multipart/form-data)

普通のrequestの送り方はドキュメントを見ていたらすぐわかると思うので、基本から少し飛び出して`multipart/form-data`の送信のexampleを示します。

`multipart`のhttp requestを送るには[mime/multipart](https://pkg.go.dev/mime/multipart@go1.22.3)を使います。

コード例を先に示し、説明を後に乗せます。

[snippet](https://github.com/ngicks/go-basics-example/blob/main/snipet/http-request-multipart-form-data/main.go)

```go
import (
	"bytes"
	"context"
	"fmt"
	"io"
	"mime/multipart"
	"net/http"
	"strings"
)

func sendMultipart(ctx context.Context, url string, client *http.Client) error {
	randomMsg1 := strings.NewReader("foobarbaz")
	randomMgs2 := strings.NewReader("quxquuxcorge")

	/*
		1.
		multipart.NewWriter(w)でio.Writerをラップした*multipart.Writerを得る。
		各種メソッドの呼び出しの結果はwに逐次書き込まれる。
	*/
	var buf bytes.Buffer
	mw := multipart.NewWriter(&buf)

	/*
		2.
		CreateFormField, CreateFormFile, CreateFormFieldなど各種メソッドを呼び出すと
		--<<boundary>>
		content-disposition: form-data; name="foo"

		のようなヘッダーを書いたうえで、そのセクションの内容を書き込むためのio.Writerを返す。
	*/
	w, err := mw.CreateFormField("foo")
	if err != nil {
		return err
	}
	_, err = io.Copy(w, randomMsg1)
	if err != nil {
		return err
	}

	w, err = mw.CreateFormFile("bar", "bar.tar.gz")
	if err != nil {
		return err
	}
	_, err = io.Copy(w, randomMgs2)
	if err != nil {
		return err
	}

	/*
		3．
		*multipart.Writerは必ず閉じる。
		閉じなければ最後の--<<boundary>>--が書かれない。
	*/
	err = mw.Close()
	if err != nil {
		return err
	}

	/*
		4.
		http.NewRequestWithContextにデータを受けたbytes.Bufferを渡す。
		(*Writer).FormDataContentType()でboundary込みの"Content-Type"を得られるのせセットする。
	*/
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, url, &buf)
	if err != nil {
		return err
	}
	req.Header.Add("Content-Type", mw.FormDataContentType())
	resp, err := client.Do(req)
	if err != nil {
		return err
	}
	_ = resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("not 200: code =%d, status = %s", resp.StatusCode, resp.Status)
	}
	return nil
}
/*
	header: User-Agent = Go-http-client/1.1
	header: Content-Length = 374
	header: Content-Type = multipart/form-data; boundary=3f297ee5803297f68c78b73c870aa54e7e083628eef2d2fb42560336909b
	header: Accept-Encoding = gzip
	bytes read: 374
	body=
	--3f297ee5803297f68c78b73c870aa54e7e083628eef2d2fb42560336909b
	Content-Disposition: form-data; name="foo"

	foobarbaz
	--3f297ee5803297f68c78b73c870aa54e7e083628eef2d2fb42560336909b
	Content-Disposition: form-data; name="bar"; filename="bar.tar.gz"
	Content-Type: application/octet-stream

	quxquuxcorge
	--3f297ee5803297f68c78b73c870aa54e7e083628eef2d2fb42560336909b--

*/
```

- 1. `multipart.NewWriter(w)`で`io.Writer`をラップした[\*multipart.Writer](https://pkg.go.dev/mime/multipart@go1.22.3#Writer)を得ます。`*multipart.Writer`から得られる`io.Writer`への書き込みはこの`w`に書き込まれます。
- 2. [(\*Writer).CreateFormField](https://pkg.go.dev/mime/multipart@go1.22.3#Writer.CreateFormField)、[(\*Writer).CreateFormFile](https://pkg.go.dev/mime/multipart@go1.22.3#Writer.CreateFormFile)、[(\*Writer).CreatePart](https://pkg.go.dev/mime/multipart@go1.22.3#Writer.CreatePart)などでセクションのヘッダーを書き込んだうえで、セクション内容を書き込むための`io.Writer`をえます。
  - 入力データソースがファイルなどの場合は[io.Copy](https://pkg.go.dev/io@go1.22.3#Copy)で書き込むとよいです。
  - `io.Copy`は`src`が[io.WriterTo](https://pkg.go.dev/io@go1.22.3#WriterTo)あるいは`dst`が[io.ReaderFrom](https://pkg.go.dev/io@go1.22.3#ReaderFrom)を実装しているときはそれらの実相を使いますが、それ以外は中間バッファをallocateしてしまいます。[io.CopyBuffer](https://pkg.go.dev/io@go1.22.3#CopyBuffer)は中間バッファを渡せるので、前述のbuf poolと組み合わせてこちらを使うほうが良いかなと思います。今回に限っては[\*strings.Reader](https://pkg.go.dev/strings@go1.22.3#Reader)は[io.WriterToを実装している](https://pkg.go.dev/strings@go1.22.3#Reader.WriteTo)ので、`io.Copy`で問題ありません。
- 3. `*multipart.Writer`は必ず閉じます。閉じなければ最後の`--<<boundary>>--`が書かれません。
- 4. [(\*Writer).FormDataContentType](https://pkg.go.dev/mime/multipart@go1.22.3#Writer.FormDataContentType)でboundary込みの`Content-Type`を得られます。これは手動でセットする必要があります。

ただこの方法だと一旦[bytes.Buffer](https://pkg.go.dev/bytes@go1.22.3#Buffer)などにすべてのデータを受けてしまうことになり、各セクションのデータが大きい場合メモリ的負荷が高くなってしまいます。そこで、[io.Pipe](https://pkg.go.dev/io@go1.22.3#Pipe)を使ってストリーム化します。

多分以下の方法がidiomaticだと思います。ご意見などお待ちしてます。

```go
func sendMultipartStream(ctx context.Context, url string, client *http.Client) error {
	randomMsg1 := strings.NewReader("foobarbaz")
	randomMgs2 := strings.NewReader("quxquuxcorge")

	writeMultipart := func(mw *multipart.Writer, buf []byte, content1, content2 io.Reader) error {
		var (
			w   io.Writer
			err error
		)

		w, err = mw.CreateFormField("foo")
		if err != nil {
			return err
		}

		_, err = io.CopyBuffer(w, content1, buf)
		if err != nil {
			return err
		}

		w, err = mw.CreateFormFile("bar", "bar.tar.gz")
		if err != nil {
			return err
		}

		_, err = io.CopyBuffer(w, content2, buf)
		if err != nil {
			return err
		}

		err = mw.Close()
		if err != nil {
			return err
		}

		return err
	}

	/*
		1.
		一旦でセクション内容以外を書き出してデータサイズをえる。
		各セクションの内容がファイルなどであれば、
		サイズは既知であるのでこれでContent-Lengthを適切に設定できる。
	*/
	metaData := bytes.Buffer{}
	err := writeMultipart(multipart.NewWriter(&metaData), nil, bytes.NewBuffer(nil), bytes.NewBuffer(nil))
	if err != nil {
		return err
	}
	fmt.Printf("meta data size = %d\n\n", metaData.Len())

	/*
		2.
		io.Pipeでin-memory pipeを取得し、
		別goroutineの中でpw(*io.PipeWriter)にストリーム書き込みする。
		pwに書かれた内容はpr(*io.PipeReader)から読み出せるので、
		これをhttp.NewRequestWithContextに渡す。
	*/
	pr, pw := io.Pipe()
	mw := multipart.NewWriter(pw)
	defer pr.Close()
	go func() {
		/*
			3.
			goroutineの中で書き込む処理そのものは
			non-stream版とさほど変わらない
		*/
		var err error
		defer func() {
			_ = pw.CloseWithError(err)
		}()
		err = writeMultipart(mw, nil, randomMsg1, randomMgs2)
	}()

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, url, pr)
	if err != nil {
		return err
	}
	req.Header.Add("Content-Type", mw.FormDataContentType())
	/*
		4.
		ContentLengthを設定する。
		req.Header.Set("Content-Length", num)は無視されるので、必ずこちらを設定する。
	*/
	req.ContentLength = int64(metaData.Len()) + randomMsg1.Size() + randomMgs2.Size()

	// Doして終わり。
	resp, err := client.Do(req)
	if err != nil {
		return err
	}
	_ = resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("not 200: code =%d, status = %s", resp.StatusCode, resp.Status)
	}
	return nil
}
/*
	meta data size = 353

	header: User-Agent = Go-http-client/1.1
	header: Content-Length = 374
	header: Content-Type = multipart/form-data; boundary=83964749f5fbc8d761fef1243f6b8cdf2a55269c70b33cea5c838c1da7b3
	header: Accept-Encoding = gzip
	bytes read: 374
	body=
	--83964749f5fbc8d761fef1243f6b8cdf2a55269c70b33cea5c838c1da7b3
	Content-Disposition: form-data; name="foo"

	foobarbaz
	--83964749f5fbc8d761fef1243f6b8cdf2a55269c70b33cea5c838c1da7b3
	Content-Disposition: form-data; name="bar"; filename="bar.tar.gz"
	Content-Type: application/octet-stream

	quxquuxcorge
	--83964749f5fbc8d761fef1243f6b8cdf2a55269c70b33cea5c838c1da7b3--

*/
```

- 1. まず、`bytes.Buffer`にセクション内容以外のヘッダーデータを書きだしてサイズを得ておきます。
  - もちろんサイズはバウンダリのサイズから計算可能ですが、こうすれば間違えようがないのでこういうやり方をします。
  - Content-Lengthを適切に設定しないとリクエストを拒絶するサーバーが存在するため、事前にサイズを計算する必要があります。
- 2. 次に[io.Pipe](https://pkg.go.dev/io@go1.22.3#Pipe)でin-memory pipeを作り、別`goroutine`の中でstream書き込みを行います。
  - `io.Pipe`は`io.Writer`を`io.Reader`に変換するようなもので、pw([\*io.PipeWriter](https://pkg.go.dev/io@go1.22.3#PipeWriter))に書かれた内容はpr([\*io.PipeReader](https://pkg.go.dev/io@go1.22.3#PipeReader))から読み込めます。
  - 中間バッファを持たないので、別の`goroutine`で動作させないとdeadlockを起こします。
  - [pipe(2)](https://man7.org/linux/man-pages/man2/pipe.2.html)に似ていますが、こちらはin-memoryです。`pipe(2)`に対応するのは[os.Pipe](https://pkg.go.dev/os@go1.22.3#Pipe)です。
- 3. 書き込みの処理自体はnon-stream版と大して変わりません。
- 4. [http.NewRequestWithContext](https://pkg.go.dev/net/http@go1.22.3#NewRequestWithContext)にこの`io.PipeReader`を渡し、`req.ContentLength`に各セクションのヘッダーとトレイラーのサイズ+各セクションのサイズ(既知であるとする)をセットします。
  - 前述のnon-stream版ではこれを設定していませんでした。
  - これは[特定のreaderの場合自動的にContentLengthが設定されるから](https://github.com/golang/go/blob/go1.22.3/src/net/http/request.go#L917-L938)です。
  - ちなみに、`req.Header.Set("Content-Length", size)`としても意味がなく、`req.ContentLength`のみが尊重されます([[1]](https://github.com/golang/go/blob/master/src/net/http/transfer.go#L291-L296), [[2]](https://github.com/golang/go/blob/go1.22.3/src/net/http/request.go#L708), [[3]](https://github.com/golang/go/blob/go1.22.3/src/net/http/request.go#L97-L103))。

とりあえずこのようにしておけば、筆者が体験する限りrequestがはじかれるようなことはありませんでした。
おそらくこれで行けてると思うんですが、ダメなケースがあるなどの場合コメントで教えていただけると幸いです。

#### \*http.Clientを色々差し替える

`Go`のstdは全般的にいい感じの粒度でinterfaceに切り分けられているため、内部の挙動を実装をラップする形で差し替えたり変更するのが容易です。

[http.Client](https://pkg.go.dev/net/http@go1.22.3#Client)はドキュメントのとおり、redirectやcookieの挙動を制御するハイレベルなclientであり、1つのrequest-responseのトランザクションは[http.RoundTripper](https://pkg.go.dev/net/http@go1.22.3#RoundTripper)で取り扱います。

これがinterfaceであるので、これの実装である[http.Transport](https://pkg.go.dev/net/http@go1.22.3#Transport)をラップしたり、全く別な実装を与えることで挙動を変更することができます。

##### Example: Request-idをつける

Headerに`X-Request-Id`がない場合付け足す`RoundTripper`実装です。

> https://pkg.go.dev/net/http@go1.22.3#RoundTripper
>
> // RoundTrip should not modify the request, except for
> // consuming and closing the Request's Body.

と書かれています。以下実装ではmutationを防ぐために`req.Clone`を呼んでdata raceを避けます。
`context.Context`経由でrequest-idが渡されるケースも想定してあります。

[snippet](https://github.com/ngicks/go-basics-example/blob/main/snipet/http-client-request-id/main.go)

```go
type keyTy string

const (
	RequestIdKey keyTy = "request-id"
)

type RequestIdTrapper struct {
	RoundTripper http.RoundTripper
}

func (t *RequestIdTrapper) RoundTrip(req *http.Request) (*http.Response, error) {
	req, err := addRequestID(req)
	if err != nil {
		return nil, err
	}
	return t.RoundTripper.RoundTrip(req)
}

func addRequestID(req *http.Request) (*http.Request, error) {
	if reqId := req.Header.Get("X-Request-Id"); reqId != "" {
		return req, nil
	}

	// Clone req to avoid data race.
	req = req.Clone(req.Context())

	reqId, _ := req.Context().Value(RequestIdKey).(string)

	if reqId == "" {
		var buf [16]byte
		_, err := io.ReadFull(rand.Reader, buf[:])
		if err != nil {
			return nil, fmt.Errorf("generating random X-Request-Id: %w", err)
		}
		reqId = fmt.Sprintf("%x", buf)
	}

	req.Header.Set("X-Request-Id", reqId)

	return req, nil
}

client := &http.Client{
	Transport: &RequestIdTrapper{RoundTripper: http.DefaultTransport},
}
```

これを[echo](https://echo.labstack.com/)などのmiddlewareと`log/slog`を組み合わせると一貫したロギングができるので調査がしやすくなります(後述)。
`RoundTripper`でやるのは邪道な感はあるので、できれば`*http.Client`を呼び出す前の段階で`addReuqestID`を使ったほうがいいですね。

##### Example: 名前解決を[github.com/miekg/dns](https://github.com/miekg/dns)に差し替える

※例示であってセキュリティー的に安全なのかいまいちわかっていません。

[http.RoundTripper](https://pkg.go.dev/net/http@go1.22.3#RoundTripper)の実装にはデフォルトで[http.Transport](https://pkg.go.dev/net/http@go1.22.3#Transport)が使われます。
`Transport`も内部の挙動を関数を差し替えるなりすることである程度手を入れさせてくれます。ここで、`Transport`の`DialContext`を差し替えることで名前解決の実装を変更できることを示します。

`Go`のstdで実装されるDNS Client部分は`glibc`などのそれよりもforgivingな実装になっていないのか、[github.com/miekg/dns](https://github.com/miekg/dns)では解決できるけどstdではできないということがあります。`github.com/glang/go`のissueを軽くさらったくらいじゃこの話題は出てこないので、多分ものすごく珍しい環境だとは思います。

そこで以下のようにDNS部分を[paepcke.de/dnsresolver](https://github.com/paepckehh/dnsresolver)経由で[github.com/miekg/dns](https://github.com/miekg/dns)実装のDNSクライアントに差し替えます。

```go
import (
	"context"
	"fmt"
	"net"
	"net/http"
	"net/netip"
	"os"
	"strings"
	"sync"
	"time"

	"github.com/miekg/dns"
	"paepcke.de/dnsresolver"
)

var (
	resolverOnce = sync.OnceValue(func() *dnsresolver.Resolver {
		// ResolverAutoが`/etc/resolve.conf`をLstatして存在チェックする。
		// wslやdockerなど一部環境はこれがsymlinkなので、その環境ではうまく動かない
		// そのため手動であらかじめStatしておいて、存在していれば明示的にそこを読み込ませる。
		if _, err := os.Stat("/etc/resolv.conf"); err == nil {
			return dnsresolver.ResolverResolvConf()
		}
		return dnsresolver.ResolverAuto()
	})
)

func dialer(fallbackOnly bool) func(ctx context.Context, network, addr string) (net.Conn, error) {
	d := &net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
	}
	return func(ctx context.Context, network, addr string) (net.Conn, error) {
		var (
			host = addr
			port = ""
			err  error
		)
		if strings.Contains(addr, ":") {
			host, port, err = net.SplitHostPort(addr)
			if err != nil {
				return nil, err
			}
		}

		raw, _ := strings.CutPrefix(host, "[")
		raw, _ = strings.CutSuffix(raw, "]")
		_, err = netip.ParseAddr(raw)
		if err == nil {
			// turns out it is already a raw IP addr.
			return d.DialContext(ctx, network, addr)
		}

		if fallbackOnly {
			_, err := net.LookupHost(host)
			if err == nil {
				return d.DialContext(ctx, network, addr)
			}
		}

		r := resolverOnce()
		resolved, err := r.LookupAddrs(host, []uint16{dns.TypeA, dns.TypeAAAA})
		if err != nil {
			return nil, err
		}
		if resolved[0].Is4() {
			addr = resolved[0].String()
		} else {
			addr = "[" + resolved[0].String() + "]"
		}
		if port != "" {
			addr += ":" + port
		}
		return d.DialContext(ctx, network, addr)
	}
}

func main() {
	t := &http.Transport{
		Proxy:                 http.ProxyFromEnvironment,
		ForceAttemptHTTP2:     true,
		MaxIdleConns:          100,
		IdleConnTimeout:       90 * time.Second,
		TLSHandshakeTimeout:   10 * time.Second,
		ExpectContinueTimeout: 1 * time.Second,
	}
	t.DialContext = dialer(false)

	client := &http.Client{
		Transport: t,
	}
	var err error
	_, err = client.Get("https://www.google.com")
	// err = <nil>
	fmt.Printf("err = %v\n", err)
	_, err = client.Get("https://142.250.206.228:443")
	// err = Get "https://142.250.206.228:443": tls: failed to verify certificate: x509: cannot validate certificate for 142.250.206.228 because it doesn't contain any IP SANs
	fmt.Printf("err = %v\n", err)
	_, err = http.DefaultClient.Get("https://142.250.206.228:443")
	// err = Get "https://142.250.206.228:443": tls: failed to verify certificate: x509: cannot validate certificate for 142.250.206.228 because it doesn't contain any IP SANs
	fmt.Printf("err = %v\n", err)
	_, err = client.Get("https://[2404:6800:400a:804::2004]:443")
	// err = Get "https://[2404:6800:400a:804::2004]:443": dial tcp [2404:6800:400a:804::2004]:443: connect: cannot assign requested address
	fmt.Printf("err = %v\n", err)
}
```

とりあえず動いている。
実際上は`func DialContext(ctx context.Context, network, addr string) (net.Conn, error)`をラップするように実装したほうがいいと思いますし、stdをもうちょい掘ったら賢い`addr`のハンドリング方法もある気がしますね。exampleとしてはこんなものかと思います。
`SAN`がないのでIP直指定だとうまいこと動かないですが、それはそれとしてIPv4 addrを渡されたときのパターンとしては動いていますね。
IPv6 addrが直接渡された時も(おそらく)筆者の環境がIPv6契約していないからうまくいかないですがそれはそれとしてうまいこと下層のDialerに渡せているようですね。

[paepcke.de/dnsresolver](https://github.com/paepckehh/dnsresolver)のソースは一通り読みましたが問題ないと判断して使っています。やはり結構面倒な都合がたくさん転がってるので、それをうまく処理してくれているように見えます。windowsで動くのかがよくわからないです。

DNS clientって何があるんですかね？こんな困りかたするの稀な気しますし、DNS server実装はいっぱいあってもclientはあんまりないように見えます。
筆者はDNSのメッセージフォーマットそのものをしっかり理解して[github.com/miekg/dns](https://github.com/miekg/dns)を直接使う方向に進もうかと思っています。
お勧めのDNS Clientあったら教えてください。

### server

`Go`のhttp serverはstdの時点で非常に強力で、std以外を一切使わないで開発するというのも十分可能です。

stdの`net/http`だけでサーバーを実装するときにかかわる概念をざっくり切り分けると以下のようになります。

```
        +---------------+
        | addr / socket |
        +---------------+
               | socket
        +--------------+
        | net.Listener |
        +--------------+
               | conn (connection)
        +--------------+
        | *http.Server | for { conn := lis.Accept(); go server.Serve(conn) }
        +--------------+
               |
       +----------------+
       | *http.ServeMux | multiplexes http.Handler-s
       +----------------+
         | | | | | | | |  ...
+--------------+ +--------------+
| http.Handler | | http.Handler | ... Handle each request
+--------------+ +--------------+
      |
+-----------------------+
| worker struct, etc... | ... running in separate goroutines, etc...
+-----------------------+
```

`addr / socket` - `net.Listener` - `*http.Server`までの関係はPOSIXで言うところの[socket(2)](https://man7.org/linux/man-pages/man2/socket.2.html)して[bind(2)](https://man7.org/linux/man-pages/man2/bind.2.html)して[listen(2)](https://man7.org/linux/man-pages/man2/listen.2.html)してfor-loopの中で[accept(2)](https://man7.org/linux/man-pages/man2/accept.2.html)するのと一致します。典型的なPOSIXプログラムは`pthread`でハンドラを実行することになると思いますが、`Go`は代わりに`goroutine`でハンドラを実行します

https://github.com/golang/go/blob/go1.22.3/src/net/http/server.go#L3254-L3286

`*http.Server`は`(net.Listener).Accept`した後さらに`context.Context`をあれこれマネージしたり新しい`goroutine`で`conn`からヘッダーを読んで[\*http.Request](https://pkg.go.dev/net/http@go1.22.3#Request), [http.ResponseWriter](https://pkg.go.dev/net/http@go1.22.3#ResponseWriter)を用意したりして`Hnadler`を呼び出します。

[\*http.ServeMux](https://pkg.go.dev/net/http@go1.22.3#ServeMux)はそれ自体が`http.Handler`です。`*http.ServeMux`はpath patternとそれ対応した`http.Handler`を登録しておくことでroutingを実現してくれるものです。

筆者は初見の時`mux`という言い回しにピンとこずに困りました。`mux`は`multiplexer`のことで、この略し方は別段`Go`に限らずされるときはされるみたいです。`multiplexer`という語自体ははデジタル回路を勉強したことがある対象読者は聞いたことがあることでしょう(アナログマルチプレクサも存在します)。回路などで言う`multiplexer`は複数の情報ストリームを１つの信号線で流す多重化の仕組みことです。ソフトウェアの文脈で言えば`HTTP/2`/`QUIC`が１つの`TCP`/`UDP`コネクションの上で`stream`を複数流すことを`multiplex`と言いますね。

#### stdのみ

前述のとおり、こういう感じでサーバーを書くことができます。
下記の状態では、どのパスにアクセスされても`mux.Handle`の行に到達するようになっています。

```go
package main

import (
	"fmt"
	"net"
	"net/http"
)


func main() {
	// ... initialization of application ...

	handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte(`{"result":"foobarbaz"}` + "\n"))
	})

	mux := http.NewServeMux()

	mux.Handle("/", handler)

	server := &http.Server{
		Handler: mux,
	}

	listener, err := net.Listen("tcp", "127.0.0.1:8080")
	if err != nil {
		panic(err)
	}

	fmt.Printf("listening = %s\n", listener.Addr())
	fmt.Printf("server closed = %#v\n", server.Serve(listener))
}
```

このサンプルではオミットされていますが実際には`http.Handler`から呼び出されるアプリケーションがあって、これらの初期化やリクエスト可能になるまでの準備を先に行うことになるでしょう。

[mux.Handle](https://pkg.go.dev/net/http@go1.22.3#ServeMux.Handle)はpath pattern, [http.Handler](https://pkg.go.dev/net/http@go1.22.3#Handler)を登録しておくことができ、incoming requestのPathがもっとも一致するpath patternに登録される`http.Handler`にルーティングします。
[http.HandlerFunc](https://pkg.go.dev/net/http@go1.22.3#HandlerFunc)は関数を`http.Handler`に変換するための型です。`Go`を書いているとこういう感じで関数や単なる変数がinterfaceを満たすことができるような型を作ることがよくあると思うので覚えておくほうが良いと思います。

[Go 1.22](https://tip.golang.org/doc/go1.22#enhanced_routing_patterns)より[mux.Handle](https://pkg.go.dev/net/http@go1.22.3#ServeMux.Handle)のpath patternの扱いに変更がいろいろはいりMETHODを持つことができるようになりました。

```go
	// GETはGET/HEADでマッチ
	mux.Handle("GET /foo", handler)
	// {name}でpath paramを設定できる。muxを通過した後にハンドラ内でr.PathValue("name")でこのパスを取得できる。
	mux.Handle("POST /foo/{parma}", handler)
	// 上記のmethod付きにマッチしないときこっちにルーティングされるので、http.StatusMethodNotAllowedを返したいならこういうルートを作る。
	mux.Handle("/foo", handler)
	// 他にマッチしないときここに必ずルートされる。
	mux.Handle("/", handler)
```

`r`はclient側の実装で取り扱った`*http.Request`と全く同じものなので`URL`フィールドや`Header`フィールドを持っています。それらを判定に使うことができます。
`w`は`io.Writer`を実装するので`io`パッケージの各種関数群や`Write`メソッドを呼び出すことでresponseを書くことができます。

client側としてrequestを送る際は`Content-Length`ヘッダーは無視されていましたが、serverがResponseを送る際は[headerの値が使われます](https://github.com/golang/go/blob/go1.22.3/src/net/http/server.go#L1191-L1195)。セットしなくても1度きりの`Write`しかしなかったりなどする場合は自動的に`Content-Length`が自動的につけられたりしますが、サイズが既知の場合はなるだけつけたほうがいいでしょうね。
コードを見る限り`Content-Type`もsniffingによって自動的にセットされる挙動があるようですが、以下のコードのようなjsonはsniffingでは判別不可能なようです。手動で`w.Header().Set("Content-Type", "application/json")`しなければなりません。

[snippet](https://github.com/ngicks/go-basics-example/blob/main/snipet/http-server-std-only/main.go)

```go
mux.Handle("/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	if r.URL.Path != "" && r.URL.Path != "/" {
		w.WriteHeader(http.StatusNotFound)
		_, _ = w.Write([]byte(`{"err":"path not found"}` + "\n"))
		return
	}
	if r.Method != http.MethodHead && r.Method != http.MethodGet {
		w.WriteHeader(http.StatusMethodNotAllowed)
		_, _ = w.Write([]byte(`{"err":"method not allowed"}` + "\n"))
		return
	}
	if !strings.HasPrefix(r.Header.Get("Content-Type"), "application/json") {
		w.WriteHeader(http.StatusBadRequest)
		_, _ = w.Write([]byte(`{"err":"non json content type"}` + "\n"))
		return
	}
	w.WriteHeader(http.StatusOK)
	_, _ = w.Write([]byte(`{"result":"ok"}` + "\n"))
}))
```

適当な例として`sync.Map`に`type Sample struct {	Foo string;	Bar int }`なJSONを収めて取得できるハンドラを書いてみます。

```go
type Sample struct {
	Foo string
	Bar int
}

func (s Sample) Validate() error {
	var builder strings.Builder
	if s.Foo == "" {
		builder.WriteString("missing Foo")
	}
	switch n := s.Bar; {
	case n == 0:
		if builder.Len() > 0 {
			builder.WriteString(", ")
		}
		builder.WriteString("missing Bar")
	case n < 0:
		if builder.Len() > 0 {
			builder.WriteString(", ")
		}
		builder.WriteString("negative Bar")
	case n > 250:
		if builder.Len() > 0 {
			builder.WriteString(", ")
		}
		builder.WriteString("too large > 250 Bar")
	}
	if builder.Len() > 0 {
		return fmt.Errorf("validation error: %s", builder.String())
	}
	return nil
}

type getResult struct {
	Key   string
	Value any    `json:",omitempty"`
	Err   string `json:",omitempty"`
}

type postResult struct {
	Key     string
	Prev    any    `json:",omitempty"`
	Swapped bool   `json:",omitempty"`
	Result  string `json:",omitempty"`
	Err     string `json:",omitempty"`
}

func main() {
	mux := http.NewServeMux()
	var store sync.Map
		mux.Handle("POST /pp/{key}", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")

		key := r.PathValue("key")

		enc := json.NewEncoder(w)
		enc.SetEscapeHTML(false)

		if !strings.HasPrefix(r.Header.Get("Content-Type"), "application/json") {
			w.WriteHeader(http.StatusBadRequest)
			_ = enc.Encode(postResult{Key: key, Err: "non json content type"})
			return
		}
		dec := json.NewDecoder(io.LimitReader(r.Body, 64<<20)) // hard limit on 64MiB
		dec.DisallowUnknownFields()
		var s Sample
		if err := dec.Decode(&s); err != nil {
			w.WriteHeader(http.StatusBadRequest)
			_ = enc.Encode(postResult{Key: key, Err: "bad request shape"})
			return
		}
		if dec.More() {
			w.WriteHeader(http.StatusBadRequest)
			_ = enc.Encode(postResult{Key: key, Err: "junk data after json value"})
			return
		}
		if err := s.Validate(); err != nil {
			w.WriteHeader(http.StatusBadRequest)
			_ = enc.Encode(postResult{Key: key, Err: err.Error()})
			return
		}

		prev, loaded := store.Swap(key, s)
		w.WriteHeader(http.StatusOK)
		_ = enc.Encode(postResult{Key: key, Prev: prev, Swapped: loaded, Result: "ok"})
	}))
	mux.Handle("GET /pp/{key}", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")

		key := r.PathValue("key")
		val, loaded := store.Load(key)
		if !loaded {
			w.WriteHeader(http.StatusNotFound)
			enc := json.NewEncoder(w)
			_ = enc.Encode(getResult{
				Key: key,
				Err: "not found",
			})
			return
		}
		w.WriteHeader(http.StatusOK)
		enc := json.NewEncoder(w)
		enc.SetEscapeHTML(false)
		_ = enc.Encode(getResult{
			Key:   key,
			Value: val,
		})
	}))
	mux.Handle("/pp/{key}", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusMethodNotAllowed)
		_, _ = w.Write([]byte(`{"err":"method not allowed"}` + "\n"))
	}))
	// ... listen and serve ...
}
```

[snippet](https://github.com/ngicks/go-basics-example/blob/main/snipet/http-server-std-only/main.go)で実行して試せるようにしています。気になる場合は実行してみて試してみてください。

middlewareが欲しいし、結構ボイラープレートな処理がすでに発生していますね。
後述の[github.com/labstack/echo](https://github.com/labstack/echo)を使う版ではこれを解決したいと思います。

`*http.Server`の`BaseContext`および`ConnContext`フィールドにコールバック関数を渡すことで、サーバー全体/コネクション(=request)レベルで`context.Context`をトラップして変更できます。
これらの`context.Context`は[(\*http.Request).Context](https://pkg.go.dev/net/http@go1.22.3#Request.Context)で取得できますので、`http.Handler`内で呼ばれる関数にはこれが渡されることが多いでしょう。
ただし、[(\*http.Server).Close](https://pkg.go.dev/net/http@go1.22.3#Server.Close)を呼べば、この`context.Context`はcancelされるので、context cancellationのために絶対にセットしないといけないということはありません。

`*http.Server`のgraceful shutdownには[(\*http.Server).Shutdown](https://pkg.go.dev/net/http@go1.22.3#Server.Shutdown), 強制的な終了には[(\*http.Server).Close](https://pkg.go.dev/net/http@go1.22.3#Server.Close)を用います。
`Close`が呼ばれると、requestについてくるcontextはcancelされます。
新しいrequestが来た時点で、connは別goroutineでRead待ちの状態になっており[[1]](https://github.com/golang/go/blob/go1.22.3/src/net/http/server.go#L2028)[[2]](https://github.com/golang/go/blob/go1.22.3/src/net/http/server.go#L677), `Close`がこれらのconnを閉じる[[3]](https://github.com/golang/go/blob/go1.22.3/src/net/http/server.go#L2950)ことでエラーが起きます[[4]](https://github.com/golang/go/blob/go1.22.3/src/net/http/server.go#L712),それによってrequestの親contextがcancelされます[[5]](https://github.com/golang/go/blob/go1.22.3/src/net/http/server.go#L749)

#### [github.com/labstack/echo](https://github.com/labstack/echo)

`std`の`net/http`をラップする形でmiddlewareの追加などを提供するライブラリがあります。
`net/http`を使わない別の方向に進んだライブラリももちろんあるんですがここではそれらは取り扱いません。

- [gin](https://github.com/gin-gonic/gin)
- [chi](https://github.com/go-chi/chi)
- [echo](https://github.com/labstack/echo)

あたりが有名だと思います。

筆者は[github.com/labstack/echo](https://github.com/labstack/echo)しか使ったことがないので他の詳細はわかりませんがおそらくおおむねに多様な形になっていると思います。

以下で、stdのみでつくったAPIを`echo`を使って再実装してみます。

基本的には公式のドキュメントをあたってください: https://echo.labstack.com/

```go
package main

import (
	"context"
	"fmt"
	"net"
	"net/http"
	"strings"
	"sync"

	"github.com/labstack/echo/v4"
)

func main() {
	e := echo.New()
	e.HideBanner = true
	e.HidePort = true

	var store sync.Map
	g := e.Group("/pp")
	g.POST("/:key", func(c echo.Context) error {
		c.Logger().Infof("request id associated with ctx: %s", c.Request().Context().Value(RequestIdKey))
		key := c.Param("key")

		if !strings.HasPrefix(c.Request().Header.Get("Content-Type"), "application/json") {
			return c.JSON(http.StatusBadRequest, postResult{Key: key, Err: "non json content type"})
		}

		var s Sample
		err := c.Bind(&s)
		if err != nil {
			return c.JSON(http.StatusBadRequest, postResult{Key: key, Err: "binding: " + err.Error()})
		}
		if err := s.Validate(); err != nil {
			return c.JSON(http.StatusBadRequest, postResult{Key: key, Err: err.Error()})
		}

		prev, loaded := store.Swap(key, s)
		return c.JSON(http.StatusOK, postResult{Key: key, Prev: prev, Swapped: loaded, Result: "ok"})
	})
	g.GET("/:key", func(c echo.Context) error {
		key := c.Param("key")
		val, loaded := store.Load(key)
		if !loaded {
			return c.JSON(http.StatusNotFound, getResult{Key: key, Err: "not found"})
		}
		return c.JSON(http.StatusOK, getResult{Key: key, Value: val})
	})

	// e.Start("127.0.0.1:8080") // You can use echo's Start to let it handle listener and server!

	server := &http.Server{
		Handler: e,
	}

	listener, err := net.Listen("tcp", "127.0.0.1:8080")
	if err != nil {
		panic(err)
	}
	fmt.Printf("listening = %s\n", listener.Addr())
	fmt.Printf("server closed = %v\n", server.Serve(listener))
}
```

ボイラープレートが減りましたね

- `(*echo.Echo).Group`サブパス以下のrouter groupを作成できます。
- `*echo.Echo`あるいは`*echo.Group`の`GET`, `POST`, `DELETE`, `PUT`などのメソッドで、http methodを絞ったルーティングができます。
- `echo.Context.Bind`でパスパラメータやボディーのペイロードなどを構造体などに代入できます。
  - この場合、`Content-Type`を見た判別が行われるのでrequestに`Content-Type`が入っていないと、`s`には何も束縛されないままエラーしません。
  - `json`出ないといけないとかの場合、middlewareなどではじいておくほうが良いです。
- `(echo.Context).JSON`などで、特定のデータフォーマットでresponseを書くことができます。
  - responseに`Content-Type`も付きます。
- `(*echo.Echo).Start`でサーバーを開始できますが、listenerやserverの細かい制御がしたいので、`http.Handler`としてのみここでは使っています。

`(*echo.Echo).Use`,`(*echo.Echo).Pre`あるいは`(*echo.Group).Use`でmiddlewareを追加できます。
[Node.js]での開発経験がある対象読者には[express.js](https://expressjs.com/)や[koa](https://koajs.com/)でなじみ深い概念だと思います。

例えばhandlerの処理前+後にそれぞれログを残すには

```diff
import (
	"context"
	"fmt"
	"net"
	"net/http"
+	"os"
	"strings"
	"sync"

	"github.com/labstack/echo/v4"
+	"github.com/labstack/gommon/log"
)

// ...

func main() {
	// ...
+	e.Logger.SetOutput(os.Stdout)
+	e.Logger.SetLevel(log.DEBUG)
+	e.Use(func(next echo.HandlerFunc) echo.HandlerFunc {
+		return func(c echo.Context) error {
+			c.Logger().Infof("received path = %s", c.Request().URL.Path)
+			err := next(c)
+			if err != nil {
+				c.Logger().Errorf("err: %#v", err)
+			} else {
+				c.Logger().Infof("no error")
+			}
+			return err
+		}
+	})
	// ...
}
```

例えば、requestのヘッダーについている`X-Request-Id`か、ない場合はランダム生成した文字列をresponseの`X-Request-Id`ヘッダーに追加しながら、
`(*http.Request).Context()`にそのrequest-idをセットするには以下のようにします。

```diff
import (
	"context"
	"fmt"
	"net"
	"net/http"
	"os"
	"strings"
	"sync"

	"github.com/labstack/echo/v4"
+	"github.com/labstack/echo/v4/middleware"
	"github.com/labstack/gommon/log"
)

// ...

+type keyTy string
+
+const (
+	RequestIdKey keyTy = "request-id"
+)
+
+func modifyConfig[T any](base T, fn func(c T) T) T {
+	return fn(base)
+}

func main() {
	// ...
+	e.Pre(middleware.RequestIDWithConfig(modifyConfig(
+		middleware.DefaultRequestIDConfig,
+		func(conf middleware.RequestIDConfig) middleware.RequestIDConfig {
+			conf.RequestIDHandler = func(ctx echo.Context, s string) {
+				ctx.SetRequest(ctx.Request().WithContext(context.WithValue(ctx.Request().Context(), RequestIdKey, s)))
+			}
+			return conf
+		},
+	)))
	// ...
}
```

### OpenAPIとコードジェネレーターやgRPCについて

## log/slog: structured logging

ログはプログラムを作って運用する場合に重要なトピックとなります。

ログを書くことでプログラムの内部状態を部分的に時系列的に保存することができます。
大抵のソフトウェアが手元で動かないため、
このような手掛かりがなければ不審な動きや不具合が起きたときに原因を調べるのが非常に難しくなります。

[Go 1.21](https://tip.golang.org/doc/go1.21#slog)よりstdで`log/slog`というstructured logging(構造化ロギング)のためのライブラリを提供されるようになりました。

### structured loggingとは

structured loggingと言えば対象読者的には[structlog](https://www.structlog.org/en/24.2.0/)とか[winston](https://www.npmjs.com/package/winston)が想像されるでしょうか？
これらを使いこなしていた対象読者には下の説明は不要なので飛ばしてください。

structured loggingというのは言葉の通り構造化された情報をログとして出力することを指しています。

これは、ものすごい乱暴に言うと`JSON`(or `yaml`, `xml`,`csv`,`MessagePack` etc etcのような任意のデータ構造を表現できるフォーマット)でログを出力できるということです。
`JSON`なので、処理に合わせて情報を付け足したり任意の構造に構築できます。

```json
// 普通ログは出力時間、ログレベル、メッセージを含みますよね
{"time":"2024-06-13T15:20:39.178198009Z","level":"DEBUG","msg":"yay"}
// JSONなので任意に情報を追加しても容易にパーズできます。
{"time":"2024-06-13T15:20:39.178198009Z","level":"DEBUG","msg":"happening of some event", "additional_info":"message for a","nested":{"path":"/foo","method":"POST"}}
```

例えば、http serverで動くプログラムを作るとき、http requestをきっかけとして起きる一連のイベントを追跡したいことはよくあると思います。
この時、追跡のための情報として`X-Request-Id`ヘッダーから取り出したrequest-idや、リクエストを受けとった時間、clientが指定したパラメータなどをログに出力しようと考えることになるでしょう。
この時、それらの情報をログのコンテキスト(以後`zap`の言い回しに倣って*structured context*もしくは*logging context*と呼ばれる)として引きまわせば一連のログにそれらの情報が出力することができます。出力されたログに対して`grep`や`jq`を駆使すれば簡単に任意のコンテキストを追跡することができます。

前述のとおり、structured loggingの重要な要素として以下があります。

- logging contextをlogger objectに結び付けられること
  - 情報は任意に追加して累積できる
- logging contextは任意の構造であること
  - `winston`の例で行くと`Object`
  - `structlog`で言うと`dict[str, Any]`
  - `Go`の場合は`[]slog.Attr`
- ログ出力時にはそれらの情報の構造を任意のフォーマットに変換して出力できること

一般に、ログ出力時間、ログレベル、ロガーメソッドを呼び出したソースコード上の短い名前などを出力したいという要求があるため、特に設定を行わなくてもロガーメソッドを呼ぶだけでこれらの情報がstructured contextに追加されて出力されることが多いです。

上記の話を踏まえるとものすごくnaiveな実装は以下のようになります。

[playground](https://go.dev/play/p/-uieTLG2OS6)

```go: 仮想的なstructured_logger.go
type Logger struct {
	arbitraryLoggingContext map[string]any
}

func (l *Logger) Log(msg string) {
	lc := maps.Clone(l.arbitraryLoggingContext)

	// 出力時間は自動的に追加されることが多い
	lc["time"] = time.Now().String()

	// 呼び出し関数、ソースの名前と行番号は自動で追加さることが多い
	var pcs [1]uintptr
	runtime.Callers(2, pcs[:])
	fs := runtime.CallersFrames([]uintptr{pcs[0]})
	f, _ := fs.Next()
	lc["caller"] = fmt.Sprintf("%s:%d:%s", f.File, f.Line, f.Function)

	lc["msg"] = msg

	// log levelはこの例では出てこない

	bin, _ := json.Marshal(lc)
	bin = append(bin, []byte("\n")...)
	os.Stdout.Write(bin)
}

func main() {
	logger := Logger{arbitraryLoggingContext: make(map[string]any)}

	// loggerには情報が紐づく
	logger.arbitraryLoggingContext["request-id"] = r.Header.Get("X-Request-Id")
	logger.arbitraryLoggingContext["param"] = r.Header.Get("Parameter-A")
	logger.arbitraryLoggingContext["request-received-at"] = time.Now().String()

	logger.Log("foobar")
	// {"caller":"/tmp/sandbox4050927908/prog.go:53:main.main","msg":"foobar","param":"param-a","request-id":"1111111111","request-received-at":"2009-11-10 23:00:00 +0000 UTC m=+0.000000001","time":"2009-11-10 23:00:00 +0000 UTC m=+0.000000001"}

	// 情報は追加できる
	logger.arbitraryLoggingContext["added"] = "added massage"

	logger.Log("bazbazbaz")
	// {"added":"added massage","caller":"/tmp/sandbox4050927908/prog.go:58:main.main","msg":"bazbazbaz","param":"param-a","request-id":"1111111111","request-received-at":"2009-11-10 23:00:00 +0000 UTC m=+0.000000001","time":"2009-11-10 23:00:00 +0000 UTC m=+0.000000001"}
}
```

### third party structured logging library

サードパーティにもstructured loggerのライブラリがたくさんあります。

- [github.com/sirupsen/logrus](https://github.com/sirupsen/logrus)(in maintainance mode)
- [github.com/rs/zerolog](https://github.com/rs/zerolog)
- [go.uber.org/zap](https://github.com/uber-go/zap)

`log/slog`のstd入りが最近([2023-08-08](https://go.dev/doc/devel/release#go1.21.0))なので、上記のどれかや似たようなサードパーティロギングライブラリを使っていることも多いのではないでしょうか。
パフォーマンスその他でこれらのライブラリを使うこともあると思いますが、stdに入っていて安定しているという点で`log/slog`に強みがあり、おそらく大概のケースで`log/slog`を使うほうが良いと思われます。

### log/slogのAPI

[playground](https://go.dev/play/p/CuyfOcRx814)

```go
slog.SetDefault(slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug})))

// Top level function = uses default logger
slog.Debug("foo", "a", "a", "b", 2, "c", map[string]any{"foo": "bar"})
// time=2009-11-10T23:00:00.000Z level=DEBUG msg=foo a=a b=2 c=map[foo:bar]

// Same as top level function
slog.Default().Debug("foo", "a", "a", "b", 2, "c", map[string]any{"foo": "bar"})
// time=2009-11-10T23:00:00.000Z level=DEBUG msg=foo a=a b=2 c=map[foo:bar]

// Newly allocated instance methods
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug}))
logger.Info("foo", "a", "a", "b", 2, "c", map[string]any{"foo": "bar"})
// {"time":"2009-11-10T23:00:00Z","level":"INFO","msg":"foo","a":"a","b":2,"c":{"foo":"bar"}}

// Add key-value pairs to logging context
logger = logger.With("foo", "bar")
logger.Error("baz", "a", "b", "c", 123)
// {"time":"2009-11-10T23:00:00Z","level":"ERROR","msg":"baz","foo":"bar","a":"b","c":123}
```

- [slog.Debug](https://pkg.go.dev/log/slog@go1.22.3#Debug)のようなトップレベル関数は`DefaultLogger`の同名メソッドへのショートハンドです
  - [slog.SetDefault](https://pkg.go.dev/log/slog@go1.22.3#SetDefault)でセットできます
  - [slog.Default](https://pkg.go.dev/log/slog@go1.22.3#Default)で、セットされたloggerを取り出せます。
- [slog.New](https://pkg.go.dev/log/slog@go1.22.3#New)で新しいインスタンスを作ることもできます。
- `Debug`, `Info`の第二引数はvariadicで、緩い型付けのkey-value pairないしは`slog.Attr`を任意の数渡すことができます。
- `With`でstructured logging contextに情報を追加したloggerをえられます。

[python]は[keyword argument](https://docs.python.org/3/glossary.html#term-argument)があったり、[winston](https://www.npmjs.com/package/winston)ではObjectをそのまま渡すことが多かったりします。
`Go`にはそういったものはないですし、Objectみたいに`map[string]any`を書くのは煩雑だったりするので、variadic argと`With`のようなメソッドで解決する形になります。

### log/slogの関係図

```
+--------------------+
| consumer of logger |
+--------------------+
          |  key-value(loosely typed pair), []slog.Attr
   +--------------+
   | *slog.Logger | convert variadic args into Attr or Record, add convenient methods(Info, Debug, etc)
   +--------------+
          |  slog.Record, []slog.Attr
   +--------------+
   | slog.Handler | manage logging context and structure, write log to output.
   +--------------+
          | `[]byte`, etc...
      +--------+
      | output | os.Stdout, remote logging service, etc...
      +--------+
```

[\*slog.Logger]: https://pkg.go.dev/log/slog@go1.22.3#Logger
[slog.Handler]: https://pkg.go.dev/log/slog@go1.22.3#Handler

[\*slog.Logger]は[slog.Handler]を使いやすいinterfaceに整えるものです。

- interface間で変換を行います。
  - [InfoContext](https://pkg.go.dev/log/slog@go1.22.3#Logger.InfoContext)や[DebugContext](https://pkg.go.dev/log/slog@go1.22.3#Logger.DebugContext)などのleveled method -> `slog.Handler.Handle`
  - `(args ...any)`のloosely typed pair -> [][slog.Attr](https://pkg.go.dev/log/slog@go1.22.3#Attr)
    - `string, any`のペアと`slog.Attr`の混在が許されます。
- [With](https://pkg.go.dev/log/slog@go1.22.3#With)が、例えば[Node.jsのwistonで言うところのchild](https://github.com/winstonjs/winston?tab=readme-ov-file#creating-child-loggers)で、structured logging contextに情報を追加したロガーを返すものです。
- [WithGroup](https://pkg.go.dev/log/slog@go1.22.3#Logger.WithGroup)が少し特殊で筆者的にピンときにくい機能です(後述)

[slog.Handler]はstructured logging contextを実際の出力フォーマットに変換して出力するものです。

- Handler実装は現状stdには
  - [TextHandler](https://pkg.go.dev/log/slog@go1.22.3#TextHandler)
  - [JSONHandler](https://pkg.go.dev/log/slog@go1.22.3#JSONHandler)
  - の二つしかありません。現実的にはこの二つ以外を使うことはまれでしょう。
- [\*slog.HandlerOptions](https://pkg.go.dev/log/slog@go1.22.3#HandlerOptions)でログ出力するレベルの制限や`Attr`の書き換えなどを行います。
- `WithAttrs`,`WithGroup`でstructured logging contextを構築できます。
  - これらのメソッドは呼ばれた時点でログを部分的に書きだしてバッファーしておくなどの最適化のためにinterface上のメソッドになっているようです。
  - 実際これらを全く使わずに、渡す`slog.Attr`を工夫でも全く同じ構造のログを書きだせます。

`WithGroup`というのが少し特殊というかピンときにくい機能で、`log/slog`上のdoc commentでも述べられていますが、以下二つは同じログを出力します

```go
logger.WithGroup("s").LogAttrs(ctx, level, msg, slog.Int("a", 1), slog.Int("b", 2))
logger.LogAttrs(ctx, level, msg, slog.Group("s", slog.Int("a", 1), slog.Int("b", 2)))
```

`WithGroup`はlogging contextの中を進んでいってしまい、戻ってくることはできないんですね。つまり、

[playground](https://go.dev/play/p/Hlb6RVgFNcC)

```go
// {/*ここにいる*/}
logger = logger.WithGroup("s")
// {"s":{/*ここにいる*/}}
logger = logger.WithAttr("a", 1)
// {"s":{"a":1/*ここにいる*/}}
logger = logger.WithGroup("t")
// {"s":{"a":1,"t"{/*ここにいる*/}}}
logger.Info("foo", "b", 2, "c", 3)
// {"time":"2009-11-10T23:00:00Z","level":"INFO","msg":"foo","s":{"a":1,"t":{"b":2,"c":3}}
```

筆者的には結構ピンときませんでした。読者の皆さんはどうだったでしょうか？

前述のとおり、stdの`Handler`は`WithAttrs`や`WithGroup`を読んだ時点で部分的にlogging contextを書き出してバッファするのでこういう風になってるみたいです。

### structured contextのストレージ

structured contextを引きまわすことですべてのログにトレースIDを乗せることができますが、引き回す方法がそのままコードの複雑さに跳ね返ります。

コンテキスト起因で振る舞いを変えるAPIは

- [Node.js]ならば[AsyncStorage](https://nodejs.org/docs/latest/api/async_context.html#class-asynclocalstorage)
- `Rust`の`Tokio`の場合[task_local](https://docs.rs/tokio/0.2.22/tokio/task/struct.LocalKey.html)

などがほかの言語やフレームワークならばあります。これらをstructured logging contextのストレージとして使うことができました。

いわゆる`TLS(Thread Local Storage)`のような物はgoroutineにはないため[pthread_getspecific(3p)](https://man7.org/linux/man-pages/man3/pthread_getspecific.3p.html)のようなAPIも当然存在しません。

`Go`はもっと明示的な方法でlogging contextを引き回します。

- `*slog.Logger`
  - というか`slog.Handler`
  - `With`, `WithAttrs`, `WithGroup`で任意の情報をstructured contextに追加したlogger objectが作られる
- `context.Context`
  - 大抵の長く動作する関数は第一引数でこれを受け取るので、これにログを格納できれば
  - `WithValue`によって任意の情報を任意のキーに関連付けた形で格納した`context.Context`を得られることができる
  - `Value`メソッドによって、キーに関連づいた情報を引き出せる
  - ただし、`slog.TextHandler`、`slog.JSONHandler`は`context.Context`から情報を引き出してログに乗せる機能を提供しない(`Go1.22.4`時点)。
    - なのでサンプルとして`context.Context`をとって格納された値をログに乗せる`slog.Handler`のexampleを実装します(後述)

### loggerの引き回し方

```go
// defaultを使う
logger := slog.Default()

// 引数で受け取る
func New(logger *slog.Logger) *Something {
	// ...
}
// 引数で受け取る2
func (*Something) someWork(ctx context.Context, logger *slog.Logger) error {
	// ...
}

// ctxつけてひきまわす
type keyTy string
const (
	SlogLoggerKey keyTy = "*slog.Logger"
)

logger, ok := ctx.Value(SlogLoggerKey).(*slog.Logger)
if !ok || logger == nil {
	logger = slog.Default()
}
```

筆者がぱっと思いつくのは上記三つくらいです。

- 1. defaultを使う方法はもっとも簡単です
  - `slog.Default`を使うので設定してくださいとドキュメントに書いても読まれない可能性があるのでやや堅くない印象を受けます。
  - ライブラリごとに出力先を変えたいなどの要求があるとき、呼び出し側にコントロールがないのでそこがいまいちに感じるときがあります。
- 2. 引数で受け取る方法はこの中で最も明示的です
  - functional options patternを併用すると、interface上気付きやすく、なおかつデフォルトも持たせられるのでバランスがいいかもしれません
  - 代わりにライブラリの使用者は個別にloggerの渡し方を考える必要があって手間ではあります。
- 3. ctxにつけてひきます
  - おそらく最も非明示的です
  - どのcontext keyに`*slog.Logger`をつけるかの同意をとるのが最も難しいです
    - std内での使い方などを見ると、ctxにValueを持たせるのは公開interface上に情報を出さず、なおかつ特定の条件のみ与えられるoptionalな値など(=context-scopeの値)を引き渡すのに有用な手段という雰囲気を感じます。
    - [http.ServerContextKey](https://pkg.go.dev/net/http@go1.22.3#ServerContextKey)みたいに公開された値としてctxに値を載せてくるstdライブラリももちろんあります。
  - その代わりに、httpのrequest-scopeのような狭い情報を付帯したloggerを引き回せるのでログの取りやすさは圧倒的に楽だと思います。

ライブラリを作るなら`1.`か`2.`。アプリ内なら`3.`もありという感じではないでしょうか。

筆者は`2.`,`3.`を併用してます。

### example: context-handler: ctxからkey-valueを取り出すslog.Handler

前述の「ライブラリに引数として`*slog.Logger`を渡すことで明示的コントロールを行う」と「request-scopeな情報を引き出せる」のいいとこどりをするための`slog.Handler`実装を通して、このinterfaceの実装のコツというか癖みたいなのを述べたいと思います。

[(\*log/slog.Logger).Log](https://pkg.go.dev/log/slog@go1.22.3#Logger.Log)および`foobarContext`のような名前のメソッドは第一引数に`context.Context`を受けとることができ、これから情報を引き出してログに乗せることができます。

他の関数が`context.Context`を第一引数で受けるときと違い、これはcancellationをとるためではなく、値を取り出すためだけに渡されます。

ということで、`context.Context`から事前に定義したkeyの情報取り出してログに乗せる`slog.Handler`を作成してみます。

設計は適当に以下とします

- `(context.Context).Value`に渡すkeyと、logging context上のkeyをマップするための`map[any]string`を定義できる
- `(context.Context).Value`から取り出されたvalueをlogging contextに追加する値に変換できる関数を定義できる
  - data raceを避けつつ値を格納するのに`*sync.Map`を入れるケースを想定するためです
    - 大抵のケースではいらない気づかいです。
  - `fmt.Stringer`などを実装しない型を変換するのに便利だと思います。
- `context.Context`から取り出された情報は
  - group nameが設定された場合は: トップレベルに1つgroupをつけてlogging contextに追加します
  - group nameが設定れない場合は: トップレベルにフラットに追加します。
  - 必ずログの先頭に追加されます
    - timeなどのbuilt-in fieldのほうが勿論優先されます。

このぐらいやれば驚きのない挙動が作れますかね？

ポイントとしては

- `WithGroup`が`slog.Handler`のもつ内部状態を変更してしまうので、単に内部`slog.Handler`の同名メソッドを呼ぶだけでは足りない
- [slogtest.Run](https://pkg.go.dev/testing/slogtest@go1.22.4#Run), [slogtest.TestHandler](https://pkg.go.dev/testing/slogtest@go1.22.4#TestHandler)で大まかな挙動のテストができる
  - ただしこのテストは`slog.Handler`が末尾のgroupに情報を付け足すパターンを想定しないので、そういう実装にすると通過しなくなる。
- 各種メソッドはconcurrentに呼び出されることになるのでdata raceを避ける気づかいが必要になる

ぐらいですかね

[snippet](https://github.com/ngicks/go-basics-example/blob/main/snipet/logger-slog-handler-context/main.go)

```go
package main

import (
	"bytes"
	"context"
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log/slog"
	"os"
	"slices"
	"sync"
	"testing/slogtest"
	"time"
)

type keyConverter struct {
	key     any
	convert func(v any) any
}

type groupAttr struct {
	name string
	attr []slog.Attr
}

type ctxHandler struct {
	inner        slog.Handler
	ctxGroupName string
	keyMapping   map[any]string
	keyOrder     []keyConverter
	groups       []groupAttr
}

var _ slog.Handler = (*ctxHandler)(nil)

func newCtxHandler(inner slog.Handler, ctxGroupName string, keyMapping map[any]string, keyOrder []keyConverter) (slog.Handler, error) {
	if len(keyMapping) != len(keyOrder) {
		return nil, errors.New("ctxHandler: mismatching len(keyMapping) and len(keyOrder)")
	}
	for _, k := range keyOrder {
		_, ok := keyMapping[k.key]
		if !ok {
			return nil, errors.New("ctxHandler: keyOrder contains unknown key")
		}
	}
	return &ctxHandler{
		inner:        inner,
		ctxGroupName: ctxGroupName,
		keyMapping:   keyMapping,
		keyOrder:     keyOrder,
		groups:       []groupAttr{{name: "top"}},
	}, nil
}

func (h *ctxHandler) Enabled(ctx context.Context, level slog.Level) bool {
	return h.inner.Enabled(ctx, level)
}

func (h *ctxHandler) Handle(ctx context.Context, record slog.Record) error {
	var ctxAttrs []slog.Attr
	for _, k := range h.keyOrder {
		v := ctx.Value(k.key)
		if k.convert != nil {
			v = k.convert(v)
		}
		ctxAttrs = append(ctxAttrs, slog.Attr{Key: h.keyMapping[k.key], Value: slog.AnyValue(v)})
	}

	topGrAttrs := h.groups[0].attr
	if h.ctxGroupName == "" {
		topGrAttrs = append(ctxAttrs, slices.Clone(topGrAttrs)...)
	}

	// reordering attached attrs.
	//
	// ctx attrs alway come first.
	// Groups come latter.
	// Attrs attached to record will be added to last group if any,
	// otherwise will be attached back to the record.

	var attachedAttrs []slog.Attr
	record.Attrs(func(a slog.Attr) bool {
		attachedAttrs = append(attachedAttrs, a)
		return true
	})
	if len(attachedAttrs) > 0 {
		record = slog.NewRecord(record.Time, record.Level, record.Message, record.PC)
	}
	if h.ctxGroupName != "" {
		record.AddAttrs(slog.Attr{Key: h.ctxGroupName, Value: slog.GroupValue(ctxAttrs...)})
	}
	record.AddAttrs(topGrAttrs...)
	if len(h.groups) == 1 {
		record.AddAttrs(attachedAttrs...)
	}

	groups := h.groups[1:]
	if len(groups) > 0 {
		g := groups[len(groups)-1]
		groupAttr := slog.Attr{Key: g.name, Value: slog.GroupValue(append(attachedAttrs, g.attr...)...)}
		if len(groups) > 1 {
			groups = groups[:len(groups)-1]
			for i := len(groups) - 1; i >= 0; i-- {
				g := groups[i]
				groupAttr = slog.Attr{Key: g.name, Value: slog.GroupValue(append(slices.Clone(g.attr), groupAttr)...)}
			}
		}
		record.AddAttrs(groupAttr)
	}
	return h.inner.Handle(ctx, record)
}

func (h *ctxHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	if len(attrs) == 0 {
		return h
	}
	groups := slices.Clone(h.groups)
	g := groups[len(groups)-1]
	g.attr = append(slices.Clone(g.attr), attrs...)
	groups[len(groups)-1] = g
	return &ctxHandler{
		inner:        h.inner,
		ctxGroupName: h.ctxGroupName,
		keyMapping:   h.keyMapping,
		keyOrder:     h.keyOrder,
		groups:       groups,
	}
}

func (h *ctxHandler) WithGroup(name string) slog.Handler {
	if name == "" {
		return h
	}
	return &ctxHandler{
		inner:        h.inner,
		ctxGroupName: h.ctxGroupName,
		keyMapping:   h.keyMapping,
		keyOrder:     h.keyOrder,
		groups:       append(slices.Clone(h.groups), groupAttr{name: name}),
	}
}

type keyTy string

const (
	RequestIdKey keyTy = "request-id"
	SyncMapKey   keyTy = "sync-map"
)

func must[V any](v V, err error) V {
	if err != nil {
		panic(err)
	}
	return v
}

func main() {
	wrapHandler := func(h slog.Handler, ctxName string) (slog.Handler, error) {
		return newCtxHandler(
			h,
			ctxName,
			map[any]string{
				RequestIdKey:       "request-key",
				SyncMapKey:         "values",
				"request-received": "request-received",
			},
			[]keyConverter{
				{key: RequestIdKey},
				{key: SyncMapKey, convert: func(v any) any {
					m, ok := v.(*sync.Map)
					if !ok {
						return nil
					}
					values := map[string]any{}
					m.Range(func(key, value any) bool {
						values[key.(string)] = value
						return true
					})
					return values
				}},
				{key: "request-received", convert: func(_ any) any {
					return time.Now()
				}},
			},
		)
	}

	randomId := hex.EncodeToString(must(io.ReadAll(io.LimitReader(rand.Reader, 16))))
	store := &sync.Map{}
	store.Store("foo", "foo")
	store.Store("bar", 123)
	store.Store("baz", struct {
		Key   string
		Value string
	}{"baz", "bazbaz"})

	ctx := context.Background()
	ctx = context.WithValue(ctx, RequestIdKey, randomId)
	ctx = context.WithValue(ctx, SyncMapKey, store)

	for _, ctxGroupName := range []string{"", "ctx"} {
		logger := slog.New(
			must(
				wrapHandler(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug}),
					ctxGroupName,
				),
			),
		)
		logger.DebugContext(ctx, "yay", slog.String("yay", "yayay"))
		logger.With("foo", "bar").WithGroup("nah").With("why", "why not").DebugContext(ctx, "nay")
	}
	/*
	   {"time":"2024-06-13T15:20:39.178198009Z","level":"DEBUG","msg":"yay","request-key":"d2a4f0e9bc80f229a2e91abc46f4752e","values":{"bar":123,"baz":{"Key":"baz","Value":"bazbaz"},"foo":"foo"},"request-received":"2024-06-13T15:20:39.17820893Z","yay":"yayay"}
	   {"time":"2024-06-13T15:20:39.17829812Z","level":"DEBUG","msg":"nay","request-key":"d2a4f0e9bc80f229a2e91abc46f4752e","values":{"bar":123,"baz":{"Key":"baz","Value":"bazbaz"},"foo":"foo"},"request-received":"2024-06-13T15:20:39.178298952Z","foo":"bar","nah":{"why":"why not"}}
	   {"time":"2024-06-13T15:20:39.178314551Z","level":"DEBUG","msg":"yay","ctx":{"request-key":"d2a4f0e9bc80f229a2e91abc46f4752e","values":{"bar":123,"baz":{"Key":"baz","Value":"bazbaz"},"foo":"foo"},"request-received":"2024-06-13T15:20:39.178315423Z"},"yay":"yayay"}
	   {"time":"2024-06-13T15:20:39.178328608Z","level":"DEBUG","msg":"nay","ctx":{"request-key":"d2a4f0e9bc80f229a2e91abc46f4752e","values":{"bar":123,"baz":{"Key":"baz","Value":"bazbaz"},"foo":"foo"},"request-received":"2024-06-13T15:20:39.17832927Z"},"foo":"bar","nah":{"why":"why not"}}
	*/
	var buf bytes.Buffer
	handler := must(wrapHandler(slog.NewJSONHandler(&buf, &slog.HandlerOptions{Level: slog.LevelDebug}), ""))

	err := slogtest.TestHandler(
		handler,
		func() []map[string]any {
			// fmt.Println(buf.String())
			var ms []map[string]any
			for _, line := range bytes.Split(buf.Bytes(), []byte{'\n'}) {
				if len(line) == 0 {
					continue
				}
				var m map[string]any
				if err := json.Unmarshal(line, &m); err != nil {
					panic(err)
				}
				ms = append(ms, m)
			}
			return ms
		},
	)
	fmt.Printf("slogtest.TestHandler = %v\n", err)
	// slogtest.TestHandler = <nil>
}
```

前述のとおり、以下の二つは同じログを出力し、

```go
logger.WithGroup("s").LogAttrs(ctx, level, msg, slog.Int("a", 1), slog.Int("b", 2))
logger.LogAttrs(ctx, level, msg, slog.Group("s", slog.Int("a", 1), slog.Int("b", 2)))
```

なおかつ、`WithGroup`や`WithAttrs`はhandler内でlogging contextをすべてクローンしたうえで部分的なlogを書き出してバッファしておく挙動があります。

このケースのように、なるだけトップレベルに情報を後付けしたいよっていうケースではそれらの挙動がどうしても邪魔になってしまうため、`Handle`までなるだけ下層の`slog.Handler`を触らないようにしています。

思いのほか面倒ですね。パフォーマンスと使いやすさと間違いにくさの折り合いがつくのがこの辺だったのでしょう。
実際[zapのパフォーマンステスト](https://github.com/uber-go/zap?tab=readme-ov-file#performance)を見ても、そこまでめちゃくちゃ遅い感じはしないです。特に`WithAttrs`された場合、その時点でログが部分的に書きだされるのが機能しているのか、`already has 10 fields of context`のテストでは結構パフォーマンスがいいですから、そういうことなんでしょうね。
ここら辺がもうちょい簡単だったら`XmlHandler`も例示してみようかなとか考えていたんですが、難しそうなのでやめておきました。

## slice

おまけにsliceの話をしておきます。

先にこの説明を書いてしまっていてて、あとから章立てを変えたときに不要になってしまったのですが、消すのももったいないので入れてあります。
これまた`Go`をよく書く人には知られた話なんですが、慣れてないうちはびっくりするポイントかもしれません。

### sliceは値 / sliceを共有するときの挙動

`Go`では`slice`をやり取りすることがよくあると思います。

`slice`は以下のようなデータ構造として表現されます。

https://github.com/golang/go/blob/go1.22.3/src/runtime/slice.go#L15-L26

[spec](https://go.dev/ref/spec#Slice_types)上では不明確ですが、`slice`は引数などで渡すと、値としてコピーして渡されます。
この時、`array`のポインタがコピーされるのでallocationが起こるまで同じ`array`を操作してしまうことがあります。

([spec上の不明確さを修正しようというissue](https://github.com/golang/go/issues/5083)は存在するが、長らくマージされない)

このことは、[slices.Sort](https://pkg.go.dev/slices@go1.22.3#Sort)が`slice`を引数として受け取ってin-placeでデータを編集することから明らかではあると思いますが、
わからず書いてしまうと変な動きをするプログラムになってしまうので注意してください。

[Effective Go#slices](https://go.dev/doc/effective_go#slices)や、[Go Slices: usage and internals](https://go.dev/blog/slices-intro)、[Arrays, slices (and strings): The mechanics of 'append'](https://go.dev/blog/slices)で説明がなされているのでご一読ください。

以下が挙動を示したサンプルコードです。`Go 1.22.3`時点ではコメントされた通りの結果がprintされます。
`cap(s)`を超えるように`append`されると別のarrayがallocationが起きているような動作が見えます。
厳密な現象がどうあれ、渡された`slice`を`append`するときは慎重な検討を行いましょう。
場合によっては[slices.Clone](https://pkg.go.dev/slices@go1.22.3#Clone)を用いて明確にallocationを行いましょう。

[playground](https://go.dev/play/p/NnnvE1LYvFb)

```go
package main

import (
	"fmt"
	"slices"
)

func appendUnsharedSlice(s []int, i int) []int {
	s = append(s, i)
	return s
}

func main() {
	var s, ss []int
	printS := func() {
		fmt.Printf("s : %#v, len = %d, cap = %d\n", s, len(s), cap(s)) // `[]int(nil)`
	}
	printSS := func() {
		fmt.Printf("ss: %#v, len = %d, cap = %d\n", ss, len(ss), cap(ss)) // `[]int(nil)`
	}
	printS() // s : []int(nil), len = 0, cap = 0
	s = append(s, 123)
	printS() // s : []int{123}, len = 1, cap = 1
	appendUnsharedSlice(s, 442)
	printS() // s : []int{123}, len = 1, cap = 1
	s = slices.Grow(s, 3)
	printS() // s : []int{123}, len = 1, cap = 4
	appendUnsharedSlice(s, 442)
	s = s[:cap(s)]
	printS() // s : []int{123, 442, 0, 0}, len = 4, cap = 4

	s = s[:1]
	for i := 1; i < cap(s); i++ {
		ss = appendUnsharedSlice(s[:i], i)
	}
	s = s[:cap(s)]
	printS() // s : []int{123, 1, 2, 3}, len = 4, cap = 4
	fmt.Println("mutation to ss propagates back to s")
	ss[2] = 999
	printS()  // s : []int{123, 1, 999, 3}, len = 4, cap = 4
	printSS() // ss: []int{123, 1, 999, 3}, len = 4, cap = 4

	ss = appendUnsharedSlice(s, 965)
	fmt.Println("new, grown slice holds a different array")
	// Grown slice holds a newly allocated array.
	printSS() // ss: []int{123, 1, 999, 3, 965}, len = 5, cap = 8

	fmt.Println("mutation to ss does not affect original slice since it has been grown / newly allocated.")
	ss[2] = 111
	printS()  // s : []int{123, 0, 999, 2}, len = 4, cap = 4
	printSS() // ss: []int{123, 1, 111, 3, 965}, len = 5, cap = 8
}
```

## さいごに

軽い気持ちで社内向けに書いていたドキュメントを強化してやるかと思って始めたんですが、ファイルサイズが５倍以上に膨れ上がっても全くまだまだカバーすべき内容はカバーしきれていないと来ています。
さすがにそろそろ疲れたのでここまでにしておきます。

- [part1 プロジェクトを始めるまで編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part1)
- [part2 cliアプリをつくれるところまで編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part2)
- [part3 concurrent GO / HTTP Server編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part3)
- part4 HTTP Server/logger編: これ

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
