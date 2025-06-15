---
title: "[Go]tarのReaderにReadAtを実装する(ついでにfs.FSにもする)"
emoji: "📂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
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
- 元のファイル内のfile content bodyの位置さえわかれば[\*io.SectionReader]で[io.ReaderAt]を実装した形で読み込み可能です。
- ただし、sparse(疎) fileの取り扱いだけがこれだけでは済みません。
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
- (2)はsparse情報を取り出すだけなら今後も追従しなきゃいけない変更も少なさそうだしこちらに決定。
- sparse holeの情報さえあれば以前作ったライブラリで仮想的に[io.ReaderAt]を結合して1つの[io.ReaderAt]のように見せることができます。
- ということでできた

～完～

## 前提知識

最低限[A Tour of Go](https://go.dev/tour/welcome/)はこなしている
会社の同僚を想定しますので基本的な話を盛り込みたい。間違っていたりした場合はおしえていただけるとさいわいです。

## 環境

```
$ go version
go version go1.24.0 linux/amd64
```

## 前提

### Goでtarを扱うにはarchive/tarを使う

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

### io.ReaderAtはとっても便利

まずどうして[io.ReaderAt]が実装されているのかうれしいのかという話を前提としてします。

- いくつかの処理はランダムアクセスができたほうが楽
  - [\*http.Request]の`GetBody`フィールドの実装
  - `PDF`などランダムアクセスによって効率的に処理できるフォーマット
  - これらに対してランダムアクセス性がないともう一度開きなおす、事前にバッファして置くなどが必要
    - それらの処理がどの程度高価なのか事前にわからないことがある。ものすごく大きなPDFをバッファに乗せたらメモリが足りなくて[Out Of Memory](https://docs.kernel.org/admin-guide/cgroup-v1/memory.html)でkillされることもあり得ます。
- [io.Seeker]に比べて[io.ReaderAt]は規約が厳しく、使う側が楽
  - seekは[io.Reader]などが内部的に持つ現在のオフセット値を変更するという処理であるので、concurrentに呼び出すと一種のrace conditionとなる
  - その点`ReadAt`はconcurrentに複数回呼び出されてもよいという規約になっている
- [\*io.SectionReader]との組み合わせによって楽に扱える
  - [io.NewSectionReader]は[io.ReaderAt]を受けとって一定のオフセット内(=section)だけを読み込める[io.Reader]/[io.Seeker]/[io.ReaderAt]の実装を返します。

色々考えることが減るのでいいわけですね。

### sparse file

この先sparse fileとかsparse holeとか単にholeというワードが出てきますが意味は以下です

- sparseは疎という意味で、ファイルサイズに対して、実際のストレージの割り当てが少ないファイルのことを言います
- holeというのはこの「実際にはストレージが割り当てられていない」場所のことを言います。
- holeを読み込むとき、linuxでは単に`0x00`として読み込まれます
- データベースや、コアダンプ(プログラムがクラッシュしたときにそのプログラムのメモリをそのまま書き出すやつ)は、実際のデータ容量に対してなにも書き込んでいない領域がたくさんあるのが普通です。こういったものを素直に`0x00`部分にもストレージを割り当ててしまうと容量があっという間に枯渇してしまう(こともあるため)、割り当てないでおくということができます。
- [Advanced Programming in the UNIX Environment](https://www.amazon.co.jp/-/jp/W-Stevens/dp/0321637739)とか読んでおいてください

### \*tar.Readerはio.ReaderAtを実装しない

[\*tar.Reader]のAPI docを見ればわかる通り、これは[io.ReaderAt]を実装しません。
前述通りstreamとして処理できることを念頭にされているため当然ではあります。

## 基本的な話: そもそもtarとは

会社の同僚に説明するなら`tar`がどういうフォーマットかなどを軽く説明しておいてほうがいいと思うので先に説明します。
知ってる人も多いことでしょうから[tarの内容にランダムアクセス性が欲しくなる時](#tar%E3%81%AE%E5%86%85%E5%AE%B9%E3%81%AB%E3%83%A9%E3%83%B3%E3%83%80%E3%83%A0%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E6%80%A7%E3%81%8C%E6%AC%B2%E3%81%97%E3%81%8F%E3%81%AA%E3%82%8B%E6%99%82)まで飛ばしてください。

### tar

`tar`というファイルフォーマットとは何ぞやという話を一応入れておきます。

`tar`は複数のファイル/ディレクトリを1つのファイルにまとめるアーカイバです。データの圧縮は仕様に含まれませんので、圧縮が必要な場合は`tar`をさらに`gzip`や`zstd`で圧縮します。

以下の各リンクで説明されています。

`FreeBSD`と`GNU(linux)`のtarコマンドのマニュアルです

https://man.freebsd.org/cgi/man.cgi?tar(1)

https://man7.org/linux/man-pages/man1/tar.1.html

お互い微妙に違うのがわかりますね。

フォーマットや来歴の話はWikipedia(英語版)の`tar(computing)`の項目で説明されています。

https://en.wikipedia.org/wiki/Tar_(computing)

`GNU`のtar manualにヘッダーフォーマットやsparse file, dumpdirやsnapshot周りの話が説明されています。

https://www.gnu.org/software/tar/manual/html_node/Tar-Internals.html#Tar-Internals

`IBM`の`tar`のフォーマットの説明があります。`GNU tar`とは微妙に異なるフォーマットとして説明されています。

https://www.ibm.com/docs/en/aix/7.1?topic=files-tarh-file

上記各リンクで述べられているところによると、

- `tar`は`Tape ARchive`をとったもの
- `tarball`となぞらえた呼称(`tarball`はなんでもくっつけてひとまとめにしてしまうから)
- 磁気テープのようなファイルシステムのないところにファイルを保存するために必要であった

とあります。今日でも多用されていますがオリジナルの開発時期はversion 7 Unixと同時期、とあります。(≒初期リリースは1979年)
`Go`の[archive/tar]内部の実装でも`v7`という呼称が出現します。

[Punched card(wikipedia)](https://en.wikipedia.org/wiki/Punched_card)によると1980年代あたりに磁気テープが紙のパンチカードを置き換えだしたらしいです。

磁気テープは文字通り、テープ状の紙/プラスティックフィルムに磁性のある粉とか液とかを封入するなり塗布するなりしたものををリール(ボビン)に巻き付け、テープを送って磁気を読み取る/磁化することで情報の読み取り/書き込みを行います。([参考](https://www.tdk.com/ja/tech-mag/ninja/029))

これに書き込むことを意図されたデータフォーマットであるならばいくつかのことが示唆されることになります。

- シーケンシャルアクセスのみを前提とする
  - テープの送り量がデータアクセス位置となります。
  - HDDやCD/BDのようなディスク媒体のように周方向にデータセグメントを分けながら半径方向に読み取り機を動かすようなことはできません。ランダムアクセス性は低いです。
- 末尾にジャンクデータが残っていることがありうる
  - 磁気テープは磁化することで情報を書き込みますので繰り返しの書き込みができます。
  - 送りのスピードが遅いならわざわざジャンクデータを消そうとしないこともあるでしょう。

うーんまだなんかありそうだけど筆者はこの程度しか言えないなあ。

フォーマットはPOSIXに載ったりドロップしたりをしている・・・とあります。
どちらにせよ`POSIX`は買わないと読めないはずなのでここでリンクを張ることができません。そのためフォーマットの完全な説明はリンク先や、[archive/tarのソースコード](https://github.com/golang/go/tree/go1.24.1/src/archive/tar)に当たってもらうとします。
ここではいくつかの特徴を説明します。

```
block size = 512 bytes
+--------------+
| header block | 1 to many
+--------------+
|  data block  | 1 to many
+--------------+
|  data block  |
+--------------+
       .
       . variable length
       .
+--------------+
| header block | extended header
+--------------+
|  data block  | extension data only relavant to next file
+--------------+
| header block |
+--------------+
|  data block  |
+--------------+
       .
       . variable length
       .
+--------------+
|  empty block | all `0x00`
+--------------+
|  empty block |
+--------------+
|  junk data   | EOF or junk
|              |
       .
       .
       .
```

- データは512バイトのブロック単位で書き込まれます。
- ヘッダー、データ、ヘッダー、データの繰り返しで表現されます。
  - データの部分がファイルの中身です。
  - いくつかはデータ部がないものがあります
    - e.g. Symlink, Char dev, size 0のregular fileなど
- いくつかの拡張ヘッダーは次のファイルに拡張情報を与えるものがあります。
- いくつかの拡張ヘッダーは通常のヘッダーの後に現れて補足的な情報を与えます。
  - [GNU tar: an archiver tool/Basic Tar Format](https://www.gnu.org/software/tar/manual/html_node/Standard.html#Standard)参照。`old GNU`のsparse fileの情報は拡張ヘッダーとして通常のヘッダーのあとに現れます。
- ヘッダーから開始さえできていれば、どこかでちょん切ったり、逆に末尾に新しいエントリを追加してかまいません。
  - 実際インクリメンタルアップデートを行った`tar`がテストデータとして`Go`のstd内にありました。
- EOFか、`0x00`のみで構成されるブロックが二つ連続して並んでいるとアーカイブ末尾となります。
- `GNU`, `old GNU`, `PAX`, `UStar`などいくつかの仕様/拡張仕様があります。

### streamが有利な場面

`tar`がstreamとして処理可能な利点は送受するときに順次処理できることです。
`zip`などのファイルへのランダムアクセスが必要となるフォーマットであればファイルのダウンロードが終わってからようやく展開の処理ができるようになります。
`tar`ではダウンロード中に展開処理ができるため、サイズが大きければこちらのほうが速いということもあるでしょうね。

streamの利点を生かした手技には、例えば以下のように`ssh`でディレクトリを送る時に`tar`してun-`tar`するというものがあります

```shell
tar -cf - -C /path/to/src . | ssh ${target} "tar -xf - -C /path/to/dst"
```

(出展はどっかにいってしまったので書けないのですが、`scp`のプロトコルを調べている時に見つけた記事の末尾に)`scp`を使ってディレクトリをコピーするとファイルチェックを毎度するためパフォーマンスが悪い、ローカルで`tar`して結果をパイプしリモートでun-`tar`するほうが速いと聞いたことがあります。
(ただし今でもそうなのかはわかりません。[scp(1)#HISTORY](https://man.openbsd.org/scp.1#HISTORY)にある通り`OpenSSH 9.0`より`scp`は`sftp`を使用して送信するのがデフォルトになったためです。)

### c.f. zip

`Go`の[archive/zip]のトップからリンクが張られていますが、以下が[archive/zip]が参照する`zip`のフォーマットです。

https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT

記載のとおり、_4.3.16 End of central directory record_、_4.3.15 Zip64 end of central directory locator_, _4.3.14 Zip64 end of central directory record_ などをファイル末尾から読み込み、そこから _4.3.12 Central directory structure_ にseek back、central directory structureに各file headerへのオフセットが入っています。
つまり、**基本的には**読み込みにランダムアクセス性を必要とします。

zipはフォーマットとしてincremental updateが可能なようです。その中でも「ファイルを消した」をcentral directoryの追記によって表現可能なようです。
specを見る限りstream-decodeは特定の条件において可能です。この「消した」の表現を尊重する面で余計なファイルを書き出してしまうという微妙さはありますが。
(実際「すべてではないが大概のzipには可能」というstackoverflowでの回答があります: [参考](https://stackoverflow.com/a/67716044))

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
  - つまり1度の`Next`呼び出しは複数ヘッダー(とデータ)を読み込んでいることもありうるため、単に`-blockSize`では足りず、もしかしたら`-blockSize*2`かもしれないし、それ以上かもしれないということです。
  - `PAX extended header`で`GNU.sparse.`が指定されていた場合、これが無視されることになります。
    - それに関するissueが上がっていないところを見ると実用上sparse fileは使われていないのかもしれないですね。

## 実装

[github.com/nlepage/go-tarfs]は便利で利用していました。作者に感謝です。

ただやっぱり[io.ReaderAt]が欲しい。あると[io.ReaderAt]を利用する他のライブラリにそのまま渡せて便利だから・・・
なので作ることにします。

### 実装方針

- 各エントリのheader start/end offset, file content start/end offsetをとることで、[\*io.SectionReader]と組み合わせることでファイルを読めるようにします。
- `tar`のheader自体の解析は[archive/tar]に任せます。
  - それに何かの修正があるたびに追従するのはできると思いますが、気づいてから取り込むまでにラグができるのは望ましくありません。
  - `Go`のモジュールシステムでは`go.mod`に書かれたバージョンにかかわらず、コンパイルするときのtoolchainのstd libが使われます。つまり、新しいコンパイラとstdライブラリのセットでコンパイルされれば[archive/tar]を使っている部分は何もしなくても修正を受けることができます。
- [github.com/nlepage/go-tarfs]と違い、元から[io.ReaderAt]を引数とします。
  - [github.com/nlepage/go-tarfs]は[io.Reader]を受けとり、[io.ReaderAt]の実装をチェックしてそれがなかったら[\*bytes.Buffer]に一旦内容をバッファします。暗黙的な挙動は怖いです。
- 上記実装に倣い、[tar.NewReader]に読み込まれたバイト数をカウントできる[io.Reader]を渡します。
- [\*tar.Reader]の`Next`を呼び、この時点での読み込まれたバイト数が、エントリのheader end offset兼file content start offsetとなりますのでこれを記録します。
- 上記実装とは違い、header start offsetは、1つ前のエントリのfile content end offsetを512バイトのブロック単位になるようにパディングしたものを得ます。
- `file conent end offset = (file content start offset) + tar.Header.Size`で取得します。
  - ただし、これはsparse fileおよび各種LinkName系のエントリに対しては正しくありません。
- sparseのhole情報を何とかして得ます。
- 得たsparse holeの情報から、
  - `tar`の実際に格納されたデータを[\*io.SectionReader]でsparse holeがある場所で切り取り
  - (これのために作った)[ByteRepeater](https://pkg.go.dev/github.com/ngicks/go-fsys-helper/stream@v0.2.0#ByteRepeater)を使って`0x00`を無限に繰り返す[io.ReaderAt]を作り
  - (これは以前に作った)[NewMultiReadAtSeekCloser](https://pkg.go.dev/github.com/ngicks/go-fsys-helper/stream@v0.2.0#NewMultiReadAtSeekCloser)で仮想的に結合します。

### offsetの収集

前述通りあるファイルエントリへのheader, file contentの開始/終了オフセットを集めますので以下の型を定義します。

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/headers.go#L27-L32

`headerEnd - headerStart`は常に512となるとは限りません。[archive/tar]は`PAX extended header`などを読み込むと、それをユーザーに返さずに次のヘッダーを読み込んで情報をマージしてからユーザーに返すためです。

[github.com/nlepage/go-tarfs]に倣い、`Read`されたバイト数を記録するReaderを定義します。

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/headers.go#L93-L116

`Seek`が実装されていますが、これは[archive/tar]が[データを読み捨てる際に`Seek`が実装されていれば使うから](https://github.com/golang/go/blob/go1.24.1/src/archive/tar/reader.go#L852-L880)です。

offsetの収集は以下のようにできます。[github.com/nlepage/go-tarfs]とほぼ同じです。(ただし[Go 1.23]で追加されたiteratorの形でですが。)

```go
func tryMapsCollect[K comparable, V any](keyMapper func(V) K, seq iter.Seq2[V, error]) (map[K]V, error) {
    collected := make(map[K]V)
    for v, err := range seq {
        if err != nil {
            return collected, err
        }
        collected[keyMapper(v)] = v
    }
    return collected, nil
}

func tryCollectHeaderOffsets(seq iter.Seq2[*headerOffset, error]) (map[string]*headerOffset, error) {
    return tryMapsCollect(func(ho *headerOffset) string { return path.Clean(ho.h.Name) }, seq)
}

func iterHeaders(r io.ReaderAt) iter.Seq2[*headerOffset, error] {
    return func(yield func(*headerOffset, error) bool) {
        countingR := &countingReader{R: io.NewSectionReader(r, 0, math.MaxInt64-1)}
        tr := tar.NewReader(countingR)

        var (
            prev *headerOffset
            blk  block
        )
        for {
            h, err := tr.Next()
            if err != nil {
                if err == io.EOF {
                    break
                } else {
                    yield(nil, fmt.Errorf("read tar archive: %w", err))
                    return
                }
            }

            headerEnd := countingR.Count

            hh := &headerOffset{h: h, headerEnd: headerEnd, bodyStart: headerEnd}
            if prev != nil {
                // bodyEnd padded to 512 bytes block boundary
                hh.headerStart = prev.bodyEnd + (-prev.bodyEnd)&(blockSize-1)
            }

            switch hh.h.Typeflag {
            case tar.TypeLink, tar.TypeSymlink, tar.TypeChar, tar.TypeBlock, tar.TypeDir, tar.TypeFifo,
                tar.TypeCont, tar.TypeXHeader, tar.TypeXGlobalHeader,
                tar.TypeGNULongName, tar.TypeGNULongLink:
                // They may have size for name.
                hh.bodyEnd = hh.bodyStart
            default:
                hh.bodyEnd = hh.bodyStart + int(hh.h.Size)
            }

            if !yield(hh, nil) {
                return
            }
            prev = hh
        }
    }
}
```

[\*tar.Reader]が0バイトしか読まないにもかかわらず、[\*tar.Header].Sizeが0でないことがテストデータ上あったため、常に`hh.bodyEnd = hh.bodyStart + int(hh.h.Size)`としてはだめで、とりあえず上記のようにswitch-caseしています。

こうして収集した情報を用いると、以下のように[\*io.SectionReader]を用いてファイルの内容を読みだすことができます。

```go
type seekReadReaderAt interface {
    io.Reader
    io.ReaderAt
    io.Seeker
}

func makeReader(ra io.ReaderAt, h *header) seekReadReaderAt {
    return io.NewSectionReader(ra, int64(h.bodyStart), int64(h.bodyEnd)-int64(h.bodyStart))
}
```

### sparse情報の再解析

ただしこれだけではsparse fileを取り扱うことができません。

[GNU Tar Manual: Storing Sparse Files](https://www.gnu.org/software/tar/manual/html_node/Sparse-Formats.html#Sparse-Formats)によると、
`GNU tar`は3通りの記法でsparse(疎) fileの格納が可能であるとあります。実際にはほかの拡張によって別な記法があるかもしれませんが少なくとも`Go`の[archive/tar]はこの3つのみをサポートします。

https://github.com/golang/go/blob/go1.24.1/src/archive/tar/reader.go#L192-L213

https://github.com/golang/go/blob/go1.24.1/src/archive/tar/reader.go#L469-L517

https://github.com/golang/go/blob/go1.24.1/src/archive/tar/reader.go#L215-L258

https://github.com/golang/go/blob/go1.24.1/src/archive/tar/reader.go#L519-L587

https://github.com/golang/go/blob/go1.24.1/src/archive/tar/reader.go#L589-L622

問題はこれらの情報は`Go`の[archive/tar]がハンドルしきってしまい(`GNU PAX sparse format version 0.1`を除いて)ユーザーにsparseの情報が返ってこないことがです。つまり、何とかして自前でこれらの情報を解析しなお必要があります。
前述通り`tar`にはいろんな拡張仕様があり1から10まで自前で実装するのは資料に当たるのが大変そうという意味で困難ですので[archive/tar]のソースをコピーして改変するのが最も現実的な解法かと思います。

わお、stdのソースをコピーして改変。したくないですねえ。ですがよくよく[archive/tar]のソースを読んでいるとsparseの情報解析しなおすだけなら案外少量のコピーで行けそうです。

[archive/tar]の`next` methodを読むと、`handleSparse`はここで呼ばれていることがわかります。
`rawHdr`という名前で、解析された最後のheader blockを渡しています。

https://github.com/golang/go/blob/go1.24.1/src/archive/tar/reader.go#L69-L173

必要最小限だけをまねると以下のようになります

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/headers.go#L118-L146

headerの`Typeflag`が`tar.TypeXHeader`や、`tar.TypeGNULongName`のとき、[archive/tar]の`next`では情報を解析したりしています。
すでに解析済みであるので要するにこれらの時はdata sectionを読み飛ばせばいいので上記の通りになります。

`handleSparseFile`, `readOldGNUSparseMap`, `readGNUSparsePAXHeaders`は[archive/tar]では[\*tar.Reader]のmethodでしたがこれらを単なる関数になるように若干改変します。

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/headers.go#L148-L216

以下が[archive/tar]から未改変のままコピーされたコード群です。コメントアウトされたものを含んでも400行以下で少量にとどめることができました。

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/copied_go_std.go

このsparse情報の再建部分ををoffsetの収集部分に盛り込むと以下のようになります。

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/headers.go#L34-L91

### sparse file readerの作成

[github.com/ngicks/go-fsys-helper/stream](https://github.com/ngicks/go-fsys-helper/stream)で

- [NewMultiReadAtSeekCloser](https://pkg.go.dev/github.com/ngicks/go-fsys-helper/stream@v0.2.0#NewMultiReadAtSeekCloser): 複数の[io.ReaderAt]を受けとって仮想的に結合して1つの[io.ReaderAt]にする
- [ByteRepeater](https://pkg.go.dev/github.com/ngicks/go-fsys-helper/stream@v0.2.0#ByteRepeater): 特定の`byte`を無限に繰り返す

を定義しておいたので、これらを組み合わせてsparse hole部分を`0x00`で読み込む[io.ReaderAt]が完成します。

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/reader.go#L9-L47

### test

で、この考え方は正しいの？というのが気になるので、テストを行います

`$(go env GOROOT)/src/archive/tar/testdata/`以下に[archive/tar]がテストに用いている`tar`ファイル群があるのでこれをコピーしてこれを[\*tar.Reader]で読みこんだ内容と、`iterHeaders`と`makeReader`で作った[io.Reader]から読み込んだ内容が同じであるかをチェックします。

...今考えると別にファイルをコピーしておく必要はないですね。まあいいでしょう。(多分`go:embed`するつもりでこうしていたんでしょう)

実際には以下のステップを踏みます

- [setup_test.go](https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/setup_test.go):
  - `go mod edit -json`でこのモジュールの`go.mod`に書かれた内容をJSONで取得します。
  - 内容から`go.mod`の`go version`を取得します(現在`go 1.24.0`ですので以後`1.24.0`と書きます。)
  - `go install golang.org/dl/go1.24.0@latest`
  - `go1.24.0 download`でsdkをダウンロード
  - `go1.24.0 env GOROOT`でstd libraryが格納されたディレクトリパスを取得
  - `cp ${GOROOT}/src/archive/tar/testdata/* ./testdata/go1.24.0/`
- [headers_test.go](https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/headers_test.go):
  - [\*tar.Reader]で読みこんだファイル内容を収集
  - `iterHeaders`と`makeReader`で作った[io.Reader]から読み込んだ内容を収集
  - `bytes.Equal`で比較

`gnu-sparse-big.tar`, `pax-sparse-big.tar`はテストから除外しました。内容が大きいせいかテストがタイムアウトするためです。
他のすべてで`bytes.Equal`で一致したためこの実装はある程度正しいようです。

テスト実行者のgo versionによらずに同じtestdataを引っ張ってこれるからこのほうがいいかな～っていう薄い考えでこういうセットアップ実装にしましたが、`go.mod`にtoolchainを指定してしまったほうがよかったかもですね。(ciと相性悪そうだし)

### fs.FS interfaceを実装する

[fs.FS]として利用したいのでそうなるようにinterfaceを整えます。

とは言え特段ここに関しては述べることはありません。
前述の`iterHeaders`で収集した情報をもとにfile system風の[trie](https://en.wikipedia.org/wiki/Trie)を構築し、ファイルをopenする際には前述の`makeReader`を呼び出すようにします。

staticな`direntry` interfaceと`Open`で開かれたstatefulな`openDirentry` interfaceを定義します。開かれたファイルには`Read`で読み込んだカーソル位置などのstateがありますが、保存されているデータそのものはダイナミックに変化しないことを期待します。

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/ent.go#L9-L20

これらは`file`と`dir`によって実装されます。

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/entfile.go

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/entdir.go

`file`はopen時に前述の`makeReader`を呼びます。

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/entfile.go#L17-L23

`dir`は`addChild`でdirentryの追加、

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/entdir.go#L26-L56

`openChild`で子要素の読み込みを行います。これらでtrieの構築と探索を行うわけですね。

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/entdir.go#L58-L78

なので`New`はrootとなる`dir`に`direntry`を`addChild`するだけになります。

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/fs.go#L13-L82

再帰呼び出しだとstackが深くなるためfor-loopで処理できたほうが計算効率はいいと思いますがシンプルにしたかったのでこんなもんで良しとします。

こちらも同じく[\*tar.Reader]で読みこんだ内容と`*tarfs.FS`で読み込んだ内容が一致するかをテストします。
さらに、[fstest.TestFS]で[fs.FS]としての挙動が正しいのかもテストします。

regular fileとdirectory以外は無視する実装なので適当にそれらしかない`tar`を用意して実行します。

https://github.com/ngicks/go-fsys-helper/blob/dd69bbd94d5c28ce6d557906cfd3e5d454839e0d/tarfs/fs_test.go

パスしました。

## おわりに

- 前から欲しかった`tar`を[io.ReaderAt]で読める実装が用意できました。
- こういった実装はあまり見ないので、多分需要は少ないんだと思います。
- `tar`というフォーマット自体をしらべたので理解が深まってよかったです。

今後は

- readerに[io.WriterTo]を実装させる
  - 今の実装ではsparse fileのholeが単なる`0x00`として読み込まれるためコピー時に非効率です
  - linuxや他のunix系のシステムでは[lseek(2)](https://man7.org/linux/man-pages/man2/lseek.2.html)でファイルの終端を超えてseekを行うとholeが作られますが、windowsでは違うというのを聞いたことがあるため調査が大変そうなのでやめておきました
  - そもそもsparseのあるファイルを取り扱わないかも
  - 現状[archive/tar]自身もsparse fileの書き込みでholeを作る実装になっていません([#22735](https://github.com/golang/go/issues/22735))ので問題ないといえばないです。
- ~~symlink supportを加える~~加えました([69ddd3691ee](https://github.com/ngicks/go-fsys-helper/commit/69ddd3691ee1876028b8318f9c0b94c015fb9b29), [d80e27dd28a](https://github.com/ngicks/go-fsys-helper/commit/d80e27dd28a2c806372a24df6e3527e2b306750c), [60f73cb8c5c](https://github.com/ngicks/go-fsys-helper/commit/60f73cb8c5cc8ad5ac9af9c2ff7c0df46cb7840b), [981b498c87b](https://github.com/ngicks/go-fsys-helper/commit/981b498c87b8cb06c0b4ee715a16f21b2beda485))。
  - [#49580](https://github.com/golang/go/issues/49580)より~~(おそらく)~~(マージ済みです)`Go1.25`から`fs.ReadLinkFS`が追加され、さらに
  - [#67002](https://github.com/golang/go/issues/67002)より`Go1.25`から[\*os.Root]に`MkdirAll`や`Symlink`など`Go1.24`では実装が見送られたmethod群が実装されますので、これと組み合わせて使えるようにする意図もあります。

[Go]: https://go.dev/
[Go 1.23]: https://tip.golang.org/doc/go1.23
[archive/zip]: https://pkg.go.dev/archive/tar
[archive/tar]: https://pkg.go.dev/archive/zip
[tar.NewReader]: https://pkg.go.dev/archive/tar@go1.24.1#NewReader
[\*tar.Reader]: https://pkg.go.dev/archive/tar@go1.24.1#Reader
[\*tar.Header]: https://pkg.go.dev/archive/tar@go1.24.1#Header
[\*bufio.Scanner]: https://pkg.go.dev/bufio@go1.24.1#Scanner
[\*sql.Rows]: https://pkg.go.dev/database/sql@go1.24.1#Rows
[io.ReaderAt]: https://pkg.go.dev/io@go1.24.1#ReaderAt
[io.Seeker]: https://pkg.go.dev/io@go1.24.1#Seeker
[\*io.SectionReader]: https://pkg.go.dev/io@go1.24.1#SectionReader
[io.NewSectionReader]: https://pkg.go.dev/io@go1.24.1#NewSectionReader
[\*http.Request]: https://pkg.go.dev/net/http@go1.24.1#Request
[github.com/nlepage/go-tarfs]: https://github.com/nlepage/go-tarfs
[fs.FS]: https://pkg.go.dev/io/fs@go1.24.1#FS
[io.Reader]: https://pkg.go.dev/io@go1.24.1#Reader
[io.Writer]: https://pkg.go.dev/io@go1.24.1#Writer
[io.WriterTo]: https://pkg.go.dev/io@go1.24.1#WriterTo
[\*bytes.Buffer]: https://pkg.go.dev/bytes@go1.24.1#Buffer
[fstest.TestFS]: https://pkg.go.dev/testing/fstest@go1.24.1#TestFS
[\*os.Root]: https://pkg.go.dev/os@go1.24.1#Root
[\*http.Request]: https://pkg.go.dev/net/http@go1.24.1#Request
