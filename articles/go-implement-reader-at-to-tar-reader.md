---
title: "[Go]tarのReaderにReadAtを実装する(ついでにfs.FSにもする)"
emoji: "📂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## \[Go\]tarのReaderにReadAtを実装する(ついでにfs.FSにもする)

案外できた

実装成果物は以下で管理されます。

https://github.com/ngicks/go-fsys-helper/tree/main/tarfs

## Overview

- [archive/tar]が返してくる[\*tar.Reader]は[io.ReaderAt]を実装しません。
  - 入力/出力をstreamとして処理できるようにするため当然ではあります。
- `tar`の中身が`PDF`のようなランダムアクセスを必要とするファイルなどであると、上記より一旦展開するほかありません。
  - `gzip`はランダムアクセス可能ではないので`tar.gz`ならそもそも一旦展開は必須です。
  - ただし、`zstd`など、オプションによりランダムアクセス可能にできる圧縮方式もあります。
  - `.tar.zstd`にし、`tar`の中身を読むとき[io.ReaderAt]を実装できれば「一旦展開して書き出す」を挟まないでよくなります。
- 「一旦展開」で殆どの場面でこまならないですが、例えば`docker container`で`container fs`をread-onlyにしていたりするとき、書き込めるディレクトリのマウントが必要になったりするので避けたいです。
- まあ要するに理屈上いらん処理を書いててなんかもやもやしていたわけです。
- [tarをfsとしてmountできるOSS](https://github.com/cybernoid/archivemount)が存在していることは知っていたため、これができることだとはわかっていました。

ここまでがbackgroundです

- prior artとして[github.com/nlepage/go-tarfs]があります
  - これは`tar`を受けとって[fs.FS]を返すもの
- 上記に倣うと、`tar.NewReader`に渡す`io.Reader`実装を、`Read`で読んだbyte数を記録できるものにしておくことで、`tar header`のオフセットを所得できます。
- 実際のファイルコンテンツの`tar`ファイル内でのオフセットは、headerの終わりと[\*tar.Header]の`Size`フィールドから取得できます
- ただし、sparse file(疎)の取り扱いだけがこれだけでは済みません。
- なぜならば、[\*tar.Reader]がtar headerに存在するsparse情報を捨ててしまうため
- そこでsparse情報を取得する必要があります。
- 実装方針に以下の2案がありました。
  - 前提として`tar`の解析処理はなるだけ`Go`のstdに任せる
    - 何かの修正があるたびに追従するのは個人では現実的ではありません。
    - `Go`のモジュールシステムでは`go.mod`に書かれたバージョンにかかわらず、コンパイルするときのtoolchainのstd libが使われます。つまり、新しいコンパイラとstdライブラリのセットでコンパイルされれば[archive/tar]を使っている部分は何もしなくても修正を受けることができます。
  - (1) [\*tar.Reader]をすべて読み切って、`tar`ファイル内の実データと突き合わせることでholeを再建する
  - (2) [archive/tar]のソースの一部をコピーし、sparse情報を取り出せるようにする。
- (1)は大きなファイルで時間がかかりすぎて使い物にならない可能性が高いため却下
  - そもそもファイルを疎にしたいのはコアダンプなど容量はデカいけど実際に何かが書かれている領域は小さい時であるため。
- (2)はsparse情報を取り出すだけなら今後も追従しなきゃいけない変更もなさそうだしこちらに決定。
- 実装したらできた。

～完～

## 前提知識

最低限[A Tour of Go](https://go.dev/tour/welcome/)はこなしている
会社の同僚を想定しますので基本的な話を盛り込みたい。間違っていたりした場合はおしえていただけるとさいわいです。

## 環境

```
$ go version
go version go1.24.0 linux/amd64
```

## 前提: Goでtarを扱うにはarchive/tarを使う

[Go]のstdの範疇で`tar`を扱うには[archive/tar]を用います。

今回は`tar`を読み込む話しかしないので、読み込みのサンプルを以下に掲載します。

[playground](https://go.dev/play/p/ZTbu6wpBP0Y)

```go
package main

import (
	"archive/tar"
	"bytes"
	"compress/gzip"
	"encoding/base64"
	"fmt"
	"io"
)

var treeTarGzBase64 = `H4sIAAAAAAAAA+2YTW7DIBBGWecUPoENZAbOg622yyg/VquevkNRlLRSf1gMScT3Nkg2EkjPD1mM` +
	`k1HHCpE5jy6yvR7PGEcxeLcN7OS5s+y9GVh/a8asx1M6DIN5Taf08vTzvL/ePyjjNM+z8jdQ5Z+9` +
	`+HdepsN/A4r/ZVkUv4Eq/zH3762F/yZc/O/X9U1njSw4EP3in7/7j178W53tfKVz/9n65tabADfj` +
	`un+l/P/RP136D5z7Z9qi/xbskX/XlP7ndFBco6p/Cvn/P5JD/y0Q8+i/Y879vyuuUdN/DO6zf3mN` +
	`/hsg5tF/x4xTSume7v+Iy/0f4f6nBcX/826nuEbd/x+X859w/rdAzOP8BwAAAAAAAIAO+ABC8URH` +
	`ACgAAA==`

func main() {
	treeTarGz, err := base64.StdEncoding.DecodeString(treeTarGzBase64)
	if err != nil {
		panic(err)
	}
	gr, err := gzip.NewReader(bytes.NewReader(treeTarGz))
	if err != nil {
		panic(err)
	}
	defer gr.Close()

	tr := tar.NewReader(gr)
	for {
		hdr, err := tr.Next()
		if err != nil {
			if err == io.EOF {
				break
			} else {
				panic(err)
			}
		}
		fmt.Printf("name = %q", hdr.Name)
		switch hdr.Typeflag {
		case tar.TypeReg, tar.TypeRegA:
			content, err := io.ReadAll(tr)
			if err != nil {
				panic(err)
			}
			fmt.Printf(", regular file, size = %d, content = %q\n", hdr.Size, string(content))
		case tar.TypeDir:
			fmt.Printf(", dir\n")
		default:
			fmt.Printf(", unknown(%d)\n", hdr.Typeflag)
		}
	}
}

/*
name = "./", dir
name = "./bbb/", dir
name = "./bbb/ccc/", dir
name = "./bbb/ccc/quux", regular file, size = 5, content = "quux\n"
name = "./bbb/ccc/qux", regular file, size = 4, content = "qux\n"
name = "./bbb/bar", regular file, size = 4, content = "bar\n"
name = "./bbb/baz", regular file, size = 4, content = "baz\n"
name = "./aaa/", dir
name = "./aaa/foo", regular file, size = 4, content = "foo\n"
*/
```

- [tar.NewReader]で[\*tar.Reader]を作成
- `Reader`の`Next` methodを呼び出すと次の[\*tar.Header]まで読み込んでそれを返し,
- `Reader`の`Read` methodで`Next`が返したheaderが示すファイルエントリの中身を読み込めます。

`Next`を呼び出すと[\*tar.Reader]の中身が次のエントリの内容にセットされるような感じです。

初見だとちょっとわかりにくい気がします(筆者はちょっと混乱しました)。ですが`tar`のデータ構造とstreamで処理したいという方針を鑑みればこうなるのもわかるかと思います。
つまり、streamとして処理するのであれば、次のデータが準備できた時には前のデータにはアクセスできないということです。そのためstatefulに作る方針になりがちです。

これは[\[Go\]続・なるだけすべてをiteratorにする#利点](https://zenn.dev/ngicks/articles/go-make-everything-iterator-2#%E5%88%A9%E7%82%B9)で説明した「ScanやNextメソッドで次のデータを準備し、データが存在するときtrueを返すパターン」でのデータの繰り返し表現です。[\*bufio.Scanner]や[\*sql.Rows]と同じようなAPI方式ですね。これらのどちらかをすでに使ったことがある場合は理解しやすいかもしれません。

ちなみにソースに埋め込まれた`tar`の内容は`GNU tar(1.35)`で作成しています。

## \*tar.Readerはio.ReaderAtを実装しない

[\*tar.Reader]のAPI docを見ればわかる通り、これは[io.ReaderAt]を実装しません。
前述通りstreamとして処理できることを念頭にされているため当然ではあります。

## 基本的な話: そもそもtarとは

会社の同僚に説明するなら`tar`がどういうフォーマットかなどを軽く説明しておいてほうがいいと思うので先に説明します。
知ってる人も多いことでしょうから[aaa](aaa)まで飛ばしてください。

### tar

`tar`というファイルフォーマットとは何ぞやという話を一応入れておきます。

以下の各リンクで説明されています。

https://en.wikipedia.org/wiki/Tar_(computing)

https://man.freebsd.org/cgi/man.cgi?tar(1)

https://man7.org/linux/man-pages/man1/tar.1.html

https://www.gnu.org/software/tar/manual/html_node/Tar-Internals.html#Tar-Internals

https://www.ibm.com/docs/en/aix/7.1?topic=files-tarh-file

`tape archive`をとったものであり、`tarball`となぞらえて`tar`と呼ばれると各リンク先で説明されています。
磁気テープのようなファイルシステムのないところにファイルを保存するために必要であった、とあります。

磁気テープは文字通り、テープ状の紙/プラスティックフィルムに磁性のある粉とか液とかを封入するなり塗布するなりしたものををリール(ボビン)に巻き付け、これを送って磁気を読み取る/磁化することで情報の読み取り/書き込みを行います。([参考](https://www.tdk.com/ja/tech-mag/ninja/029))

これに書き込むことを意図されたデータフォーマットであるならばいくつかのことが示唆されることになります。

- シーケンシャルアクセスのみを前提とする
  - テープの送りがデータアクセス位置となります。
  - HDDやCD/BDのようなディスク媒体のように周方向にデータセグメントを分けながら半径方向にヘッドを動かすようなことはできません。ランダムアクセス性は低いです。
- ジャンクデータが残っていることがありうる
  - 磁化することで情報を書き込みますので繰り返しの書き込みができます。
  - 送りのスピードが遅いならわざわざジャンクデータを消そうとしないこともあるでしょう。

うーんまだなんかありそうだけど筆者はこの程度しか言えないなあ。

完全な説明はリンク先や、[archive/tarのソースコード](https://github.com/golang/go/tree/go1.24.1/src/archive/tar)に当たってもらうとして、いくつかの特徴を説明します。

- データは512バイトのブロック単位で書き込まれます。
- ヘッダー、ファイル、ヘッダー、ファイルの繰り返しで表現されます。
  - いくつかはファイル部がないものがあるかもしれません。
- いくつかの拡張ヘッダーは次のヘッダー/ファイルに拡張情報を与えるものがあります。
- `0x00`のみで構成されるブロックが二つ連続して並んでいるとアーカイブ末尾となります。
- `GNU`, `old GNU`, `PAX`, `UStar`などいくつかの仕様/拡張仕様があります。

### streamが有利な場面

`tar`がstreamとして処理可能な利点は送受するときに順次処理できることです。
`zip`などのファイルへのランダムアクセスが必要となるフォーマットであればファイルのダウンロードが終わってからようやく展開の処理ができるようになります。
`tar`ではダウンロード中に展開処理ができるため、サイズが大きければこちらのほうが速いということもあるでしょうね。

例えば以下のように`ssh`でディレクトリを送る時に`tar`してun-`tar`するというものがあります

```shell
tar -cf - -C /path/to/src . | ssh ${target} "tar -xf - -C /path/to/dst"
```

(出展はどっかに言ってしまったので書けないのですが、)`scp`のプロトコルを調べていると`scp`を使ってディレクトリをコピーするとファイルチェックを毎度するためパフォーマンスが悪い、`tar`して`untar`するほうが速いと聞いたことがあります。

(ただし今でもそうなのかはわかりません。[scp(1)#HISTORY](https://man.openbsd.org/scp.1#HISTORY)にある通り`OpenSSH 9.0`より`scp`は`sftp`を使用して送信するのがデフォルトになったためです。)

### c.f. zip

`Go`の[archive/zip]のトップからリンクが張られていますが、以下が[archive/zip]が参照する`zip`のフォーマットです。

https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT

TODO
に記載のとおり、末尾のを参照するためランダムアクセスが必要となります。

## tarの内容にランダムアクセス性が欲しくなる時

例えば以下が思いつきます。

- `tar`ファイルにランダムアクセス可能であり
  - 無圧縮、もしくは
  - `zstd`([Zstandard Seekable Format](https://github.com/facebook/zstd/blob/dev/contrib/seekable_format/zstd_seekable_compression_format.md)), `xz`(["limited random-access reading"](https://tukaani.org/xz/format.html#_features))などのような、ランダムアクセス可能オプションのある圧縮方式で圧縮されているとき
- `tar`内部のファイルがランダムアクセスを必要とし
  - e.g. `PDF`(参考:[PDF 1.7 spec](https://opensource.adobe.com/dc-acrobat-sdk-docs/pdfstandards/PDF32000_2008.pdf), 7.5.5 File Trailerより、`PDF`はファイル末尾から読み込む。更に別のFile Tailerに向けてseek backする必要があるときがある。)
- `tar`ファイルを展開したくないとき
  - e.g. `tar`が非常に大きく、ファイルは一時的に利用されるのみ、もしくはファイルシステム経由でアクセスすされることはないので展開したファイルの配置が不要である
  - e.g. アプリが`container`内で動作しており、`container fs`がread-only modeにされているとき、やらんでいいなら書き込めるディレクトリをマウントしたくない。

`tar`に内包されたファイルの特定のオフセット内を[\*io.SectionReader]\(包まれる側に[io.ReaderAt]の実装を必要とする\)で包んで[\*http.Request]の`GetBody`から返せるようにするっていうのがいちばん思いつきましたが、
代替する方法はいくらでもあるので立て付け上では上記のようなことになると思います。

## prior arts

### archivemount

下記を用いると[FUSE](https://www.kernel.org/doc/html/latest/filesystems/fuse.html)によりアーカイブファイルをmount可能です。
このアーカイブの中には`tar`も含まれます。

https://github.com/cybernoid/archivemount

これを使用したことはありませんが、seekできないのにファイスシステムとしてアクセス可能と言い張るのは無理があるので、`tar`内部のファイルへのランダムアクセスが可能なのは間違いないことでしょう。

### [github.com/nlepage/go-tarfs]

[github.com/nlepage/go-tarfs]は、`Go`のモジュールで`tar`を受け取って[fs.FS]としてアクセス可能にします。

このライブラリでも[io.ReaderAt]は実装されません。

このライブラリがどのように動作するのかというと、

まず以下のように、

https://github.com/nlepage/go-tarfs/blob/v1.2.1/fs.go#L22-L75

- [tar.NewReader]に読み込まれたバイト数をカウントできる`io.Reader`を渡します。
- [\*tar.Reader]の`Next`を順次呼び、読み込まれたバイト数-512(`blockSize`)=ヘッダー開始地点オフセットを記録しておきます。
- このようにしてファイル情報を収集します。

[fs.FS]としてファイルを開く際には以下のように、

https://github.com/nlepage/go-tarfs/blob/v1.2.1/entry.go#L58-L75

- 記録しておいたヘッダー開始地点から始まるように元の[io.ReaderAt]を[\*io.SectionReader]で包みます。
- これを[tar.NewReader]に渡し、最初のエントリを読み込ませて返します。

`tar`がフォーマットとして途中から分割しても正常に動作することを利用した巧みな実装だといえます。

ただし以下の点で正しくないと思われます。

- `r.Count()-blockSize`は正しくない
  - `Go`の[archive/tar]が一部の拡張ヘッダーの、次のヘッダーにメタデータを与える系のヘッダーを読み込んだ際、それはユーザーに返さずに次のヘッダーを読み込んでメタデータをマージしてから返します
  - つまり1度の`Next`呼び出しは複数ヘッダーを読み込んでいる可能性があります。
  - `PAX extended header`で`GNU.sparse.`が指定されていた場合、これが無視されることになります。

## 実装

[github.com/nlepage/go-tarfs]は便利で利用していました。作者に感謝です。

ただやっぱり[io.ReaderAt]が欲しい。あると他のライブラリにそのまま渡せて便利なのに・・・
なので作ることにします。

### 実装方針

- 各エントリのheader start/end offset, file content start/end offsetをとることで、[\*io.SectionReader]と組み合わせることでファイルを読めるようにします。
- `tar`のheader自体の解析は[archive/tar]に任せます。
- [github.com/nlepage/go-tarfs]と違い、元から[io.ReaderAt]を引数とします。
  - [github.com/nlepage/go-tarfs]は[io.Reader]を受けとり、[io.ReaderAt]の実装をチェックしてそれがなかったら[\*bytes.Buffer]に一旦内容をバッファします。暗黙的な挙動は怖いです。
- 上記実装に倣い、[tar.NewReader]に読み込まれたバイト数をカウントできる[io.Reader]を渡します。
- [\*tar.Reader]の`Next`を呼び、この時点での読み込まれたバイト数が、エントリのheader end offset兼file content start offsetとなりますのでこれを記録します。
- 上記実装とは違い、header start offsetは、1つ前のエントリのfile content end offsetを512バイトのブロック単位になるようにパディングしたものを得ます。
- `file conent end offset = (file content start offset) + tar.Header.Size`で取得します。
  - ただし、これはsparse fileおよび各種LinkName系のエントリに対しては正しくありまえん。
- sparseのhole情報を何とかして得ます。
- 得たsparse holeの情報から、[\*io.SectionReader]で切り取った実際に`tar`に格納されたデータと、(これのために作った)[ByteRepeater](https://pkg.go.dev/github.com/ngicks/go-fsys-helper/stream@v0.2.0#ByteRepeater)を、以前作った[NewMultiReadAtSeekCloser](https://pkg.go.dev/github.com/ngicks/go-fsys-helper/stream@v0.2.0#NewMultiReadAtSeekCloser)で仮想的に結合します。

[Go]: https://go.dev/
[archive/zip]: https://pkg.go.dev/archive/tar
[archive/tar]: https://pkg.go.dev/archive/zip
[tar.NewReader]: https://pkg.go.dev/archive/tar@go1.24.1#NewReader
[\*tar.Reader]: https://pkg.go.dev/archive/tar@go1.24.1#Reader
[\*tar.Header]: https://pkg.go.dev/archive/tar@go1.24.1#Header
[\*bufio.Scanner]: https://pkg.go.dev/bufio@go1.24.1#Scanner
[\*sql.Rows]: https://pkg.go.dev/database/sql@go1.24.1#Rows
[io.ReaderAt]: https://pkg.go.dev/io@go1.24.1#ReaderAt
[\*io.SectionReader]: https://pkg.go.dev/io@go1.24.1#SectionReader
[\*http.Request]: https://pkg.go.dev/net/http@go1.24.1#Request
[github.com/nlepage/go-tarfs]: https://github.com/nlepage/go-tarfs
[fs.FS]: https://pkg.go.dev/io/fs@go1.24.1#FS
[io.Reader]: https://pkg.go.dev/io@go1.24.1#Reader
[io.Writer]: https://pkg.go.dev/io@go1.24.1#Writer
[\*bytes.Buffer]: https://pkg.go.dev/bytes@go1.24.1#Buffer
