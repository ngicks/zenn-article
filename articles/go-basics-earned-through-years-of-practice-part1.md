---
title: "Goで開発して3年のプラクティスまとめ(1/4): プロジェクトを始めるまで編"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## Goで開発して3年のプラクティスまとめ(1/4): プロジェクトを始めるまで編

yet another入門記事です。

- part1 プロジェクトを始めるまで編: これ
- [part2 cliアプリをつくれるところまで編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part2)
- [part3 concurrent GO編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part3)
- [part4 HTTP Server/logger編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part4)

## はじめに

筆者は[Go]を触りだして3年ぐらいたったので、触りだしてからもっと早く知りたかったこととかをまとめて置くことで筆者の知識の整理を行います。

元は社内向けに軽いイントロダクションのドキュメントは作っていたんですが、成果が社内に限られるのはいまいちに感じたので私的な時間を使って新規に作り直しています。
比べてみるとファイルサイズで５倍以上になっていました。多分最初に書いたものよりは役に立つと思います。

なるだけワンストップでたくさんの話題を扱うようにして、
読んだ人(会社の同僚)がなんとなく開発を始められるようになって、
なんとなくイディマティックなコードを書けるようにするのが目的です。

ご質問やご指摘がございましたらこの記事のコメントでお願いします。
(ほかの媒体やリンク先に書かれた場合、筆者は気付きません)

## Overview

筆者にとって「プロジェクトを始めるまでの方法がわからん」っていうのが新しいプログラミング言語を学ぶときの1つ目のハードルだったりします。
ついでに言うと筆者が所属するようなcorporate proxyの背後にあってprivate gitを用いる環境ではさらにもう1段ハードルがあるため大変です。大変でした。
part 1はそれらを解消しうるものになることを意図しています。

以下の順番で書きます

- 筆者のバックグラウンドとかおまけ的な話
  - 筆者は`Go`を使い始める前から開発自体はしていたので、暗黙的な前提知識がたくさんあります。それがなんとなくわかるようにしています。
- 基本的な読み物系へのリンク
- プロジェクトの始め方
  - github / gitlabなどにrepositoryを作成してgo moduleを作って動作させるまで
- Dockerfileの紹介
  - 対象読者には場合によっては高度すぎる話題かもしれないので読み飛ばすほうが良いかもしれません。

記事中にTODOコメントがそのまま残ってるかと思います。読み物を紹介しておいて、筆者が全部読んでないやつがあるんですね。後追いで読み終わったらTODOコメントを消すかもしれません。そのとき万一、筆者が知らなかったことが発覚して記事の修正が発生した場合はgithub上で差分を見れるようにしておきます。追記はあってもおおむね大幅書き直しは起こらないと思っています。

この一連の記事は基本的に突っ込んだ内容をひたすら語ります。突っ込んだ内容に付随する基本的もなるだけ書いていきます。part 1は比較的優しめな内容だと思います。

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

## 筆者のバックグラウンド

筆者のバックグラウンドを書くことで、アンビエントに存在する前提知識を掲示します。

- 学生時代は[C++]を使ってセンサー値を読み込んで計算を行うプログラムを記述していた。
  - つらかった
  - Windows 7
  - Visual Studio Community
  - 筆者は機械系の学徒であって、装置/回路の設計、実験方法の考案など広く浅くなのでこの時点で大してソフトウェアには詳しくありませんでした。
  - まるきりsingle-threadedでした
  - すごい余談ですが四元数で回転を計算するのに[Eigen](https://eigen.tuxfamily.org/index.php?title=Main_Page)を使っていました。
    - 優れたライブラリです。でもC++はほかの人が作ったパッケージを持ってくるのが大変でした。
- 入社してからメイン[Node.js]、ほんのちょっぴり[python]で開発を行う
- 独学で[The Rust Programming Language 日本語]を(確か2018年エディションを)読了、PDFiumやOpenSSLのbindingをrustで書いたりして使ってました
- 趣味レベルで`React`、仕事のヘルプでちょびっと`Vue2`
- その後[Go]を使い始める。多分一番慣れてる言語です。

つまり筆者は[Go]を使い始める前に以下をなんとなく理解しています

- [C++]・・・古い本を読みながらやったので当時の最新の書き方でもなかった。
- [Node.js]
- [TypeScript]
- [Rust]
- `Linux`上でファイルを読み書きしたりデータストレージとやり取りするときに起きる諸般の問題
  - [open(2)](https://man7.org/linux/man-pages/man2/open.2.html)を`O_TRUNC`付きで呼んでから[write(2)](https://man7.org/linux/man-pages/man2/write.2.html)がリターンするまでの間にファイルが0バイトの状態が観測できる、とか

筆者は`Linux`が動作する小さ目のデバイスで動くプログラムしか書かないので、
クラウドとかそういったものが視点に入っていません。
結構特殊な視点で書かれているかもしれないです。

## 基本

### Goとは

> https://go.dev/doc/
>
> The Go programming language is an open source project to make programmers more productive.
>
> Go is expressive, concise, clean, and efficient. Its concurrency mechanisms make it easy to write programs that get the most out of multicore and networked machines, while its novel type system enables flexible and modular program construction. Go compiles quickly to machine code yet has the convenience of garbage collection and the power of run-time reflection. It's a fast, statically typed, compiled language that feels like a dynamically typed, interpreted language.

[Go]はGoogleが開発しているオープンソースのプログラミング言語です。

- C系の文法で
- 静的型付けで
- `interface`によるダイナミックディスパッチがあり
- GCがあり
- 文法や構文が厳選されており、追加もめったにないため書き方がブレにくく
- concurrentに関数を実行する`goroutine`の仕組みがあるため、非常に容易にconcurrentな実行ができ
  - `goroutine`はruntimeがスケジューリングを行う軽量な*thread of execution*([green thread](https://en.wikipedia.org/wiki/Green_thread))
  - そのためasync/await的記法がない
- 逆にgoroutine以外ないのでライブラリ間で分断が起こることがなく
- コンパイルが非常に早く
- (C-bindingを使わない限り)クロスコンパイルが簡単で
- (C-bindingを使わない限り)staticなシングルバイナリを簡単に出力することができ
- モジュール/パッケージによるネームスペースの分割機能があり
- モジュールの取得に中央集権的なレジストリがない
  - いい点として`github`などからモジュールを取得するのがすごく簡単です

みたいな言語です。

### A Tour of Go

https://go.dev/tour/welcome/1

`Go`の基本的なトピックはここに書いてあります。
インタラクティブなコードスニペットと簡単なエクササイズがあり、これさえこなせばとりあえず開発は始められます。

慣れてない頃に、syntax highlightのかからないwebページでコードを書くのはきついと思うのでローカルのエディターにコピーして実行したほうが良いとは思います。
大体３~５時間ぐらいで全部終わると思います。

TODO: 手元のエディターでA tour of goをやれるかの検証

### Go by Example

https://gobyexample.com/

コード例とともに解説がされます。
項目数が多く、知らないstdモジュールを使う部分をきちんと理解しようとすると時間がかかると思います。
手が空いたら少しずつ読むのがいいのではないでしょうか。

### 公式読み物系

TODO: 読む

公式が出している読み物集。全部英語です。
初めから全部読む必要はないと思いますが折を見て読んでいくのがよいと思います。

- The Go Language Specification: https://go.dev/ref/spec
  - 言語仕様ですが割と短めなのでそのうち読んでおいたほうがよいでしょう。
- How to Write Go Code: https://go.dev/doc/code
  - コードの書き方以外も含めた基本的なトピック
- Effective Go: https://go.dev/doc/effective_go
  - Goのイディオム集
  - 古いところもあるので、一通り読んだらほかのドキュメントも読んでください
- Go Wiki: Go Code Review Comments: https://go.dev/wiki/CodeReviewComments
  - よくされるCode review comment集らしいです

### Std library

TODO: いった手前自分でも読む

https://pkg.go.dev/std

standard libraryです。

HTTPなどで動作するサーバープログラムを作るのに大体必要な機能がそろっています。
できれば開発に着手する前にすべてのインターフェイスとdoc commentを読んでおくがよいと思います。
`CGO`と[SWIG](https://swig.org/)への言及があるなど、読んどけばよかった系の文章が意外なほどたくさん書いてあります。

https://pkg.go.dev/golang.org/x

こっちはsub-repositoriesです。説明のとおり、Go Projectの一環ですがstd libほど厳密なバージョン管理がされていません。
std libに入ると厳密な後方互換性の約束を守る必要があります。
そのため変更の可能性が高かったり、stdに入れるほどの重要度がないものがこちらにあるというコンセプトのはずです。

- ここで先に実装されてからstdに昇格されたり(`maps`, `slices`など)、
- stdがFrozenなので代わりにこちらのものを使うべきだったり(`syscall`の代わりに[golang.org/x/sys](https://pkg.go.dev/golang.org/x/sys#section-readme))
- 1ファイルにバンドルされてstdに組み込まれていたり([golang.org/x/net/http2](https://pkg.go.dev/golang.org/x/net/http2))

することもあります。

### golang/example

TODO: 全体をざっと眺める。

https://github.com/golang/example

公式でメンテされているexample集。新しめな話題は取り扱っていないこともある。

## プロジェクトの始め方

モジュールをセットアップするまでのあれこれをまとめておきます。

公式ドキュメントに網羅されている内容ですのでそちらに当たってもらうほうがよいでしょう。
特に基本的文法は`A Tour of Go`で網羅的に述べられるので説明しません。

`VCS`(Version Control System)にrepositoryを一つ作り、そこに1つ`Go module`を作るところまでをここでカバーします。

`VCS`はここでは`git`しか想定されていません。

### インストール

公式の手順に従い、各OS環境に合わせて[Go]をインストールしましょう。

https://go.dev/doc/install

`linux`/`amd64`の場合はいつもの手順です

```
mkdir /tmp/go-download
cd /tmp/go-download
curl -LO https://go.dev/dl/go1.22.3.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.22.3.linux-amd64.tar.gz
```

一応チェックサムを確認しておいたほうがいいかもしれません。(筆者はshellを使い倒すのに慣れていないのでコマンド自体は参考程度に)

```
# checksumの値はダウンロードページから確認できる。
echo 8920ea521bad8f6b7bc377b4824982e011c19af27df88a815e3586ea895f1b36 > checksum
sha256sum go1.22.3.linux-amd64.tar.gz | awk '{print $1}' | diff - checksum
```

環境変数を設定。使っているOS/terminalに合わせた方法で設定してください。

```
export PATH=$(/usr/local/go/bin/go env GOPATH)/bin:/usr/local/go/bin:$PATH
```

`/usr/local/go/bin`以下に、先ほど`.tar.gz`から解凍した`go`コマンドと`gofmt`コマンドが置かれます。

`$(go env GOPATH)/bin`以下には`go install`したバイナリがおかれます。
パスを通しておけばコマンドとして利用できるようになります。

### エディタ

エディタは個人の好みともろもろを合わせて好きに選べばいいと思います。
ただよく聞く+[goのsurvey](https://go.dev/blog/survey2024-h1-results)の上位3位は以下の3通りです。

- [Visual Studio Code] + [Go extension](https://marketplace.visualstudio.com/items?itemName=golang.go)（筆者はこれ）
  - https://code.visualstudio.com/docs/languages/go
- [vim](https://www.vim.org/) or [neovim](https://neovim.io/) + gopls
  - https://github.com/golang/tools/blob/master/gopls/doc/vim.md
- JetBrainsの[GoLand](https://www.jetbrains.com/ja-jp/go/)

goのsurveyはself-selection, `vscode`の`Go extension`からの誘導, `GoLand`からの誘導の経路があるのでself-selection biasがかかってるはずなんですが、
それでもこの3種がよく聞くので多分本当にこの3種が多い。

### VCSでのrepositoryの作成

手順そのものは説明しませんが、以下の手順は[github](https://github.com/)や[gitlab](https://about.gitlab.com/)でrepositoryが存在していることを想定します。

[VCS(Version Control System)](https://en.wikipedia.org/wiki/Version_control)は, コンピュータファイルのバージョンを管理するシステムのことです。
代表的なものは[git]や[svn](https://en.wikipedia.org/wiki/Apache_Subversion),[mercurial](https://en.wikipedia.org/wiki/Mercurial)あたりだと思います。
この記事では`git`のみを取り扱います(筆者がほか二つのことをほぼまったく知らないからです)

`git`は、`VCS`を構築するためのサーバーおよびクライアントプログラムです。サーバーとして直接使うことはほとんどないかもしれません。
現在では`git`サーバーは[github](https://github.com/)というwebサービスを利用するか、 セルフホストすることも可能な[gitlab](https://about.gitlab.com/)、あるいは[gitbucket](https://github.com/gitbucket/gitbucket)などを使うのが一般的だと思います。(この3つがリストされてるのは単に筆者が使ったことあるやつ3種っていうだけです)

先にローカルでrepositoryを作成してあとからremote上に作成する方法もあるはずですが、この説明ではremoteを先に作る方法しか想定されません。

### Go moduleの初期化

`VCS`で作成したrepositoryをローカルにクローンします。

```
# gitの場合
git clone <<uri>>
cd <<repo-name>>
```

`go mod init <<module-name>>`で`Go module`に必要なファイルを作成します

```
go mod init <<module-name>>
```

`<<module-name>>`は基本的に上記`<<uri>>`からプロトコルスキームを抜いたものにするとよいです。
そうすると`go get <<module-name>>`でこのモジュールを別の`Go module`へ導入できるためです。

例えば、`VCS`の`URI`が`https://github.com/ngicks/go-basics-example`である場合、

```
go mod init github.com/ngicks/go-basics-example
```

で作成し、`VCS`にソースをプッシュすると

```
go get github.com/ngicks/go-basics-example
```

で別モジュールから導入、参照できます。

:::details local onlyのモジュール

ローカルオンリーなつもりならおおむね`<<module-name>>`はなんでもよいはずです。

```
go mod init whatever-whatever
```

筆者が知る限りほかのモジュールから`go get`できなくなる以外の違いはないです。

ただし、[How to write code](https://go.dev/doc/code)でも勧められる通り、なるだけ公開される前提でモジュールを作っておいたほうがよいでしょう。

:::

#### セルフホストのVCSかつサブグループを使用する場合

ただし、セルフホストの`VCS`(=URIが`go tool`に対して既知でない)でサブグループを作成し、サブグループの中でソースを管理する場合、
`<<module-name>>`はvcsのsuffixを加えておかないと`go get`時に失敗するかもしれません。

つまり上記と同じ例で行くと

```
# 架空のURLを扱うのでexample.comに変えてあります！
go mod init example.com/ngicks/subgroup/go-basics-example.git
```

とする必要があるということです。

なぜか？

`go get`が利用するロジックが以下のように、インポートパスから`vcs`がなんであるかをマッチしようとしますが、セルフホストの場合既知ではないので末尾の`// General syntax for ...`のところでマッチするはずです。

https://github.com/golang/go/blob/go1.22.3/src/cmd/go/internal/vcs/vcs.go#L1515-L1577

ここぐらいしか違いを生みそうな行がないです。

この時、`.git`がついていればregexpに正しくマッチするが、
そうでなければパスがわからないので、典型的な`<<host>>/<<user>>/<<repository>>`のパスが`.git`と思って通信しようとする、と思われます。

上記の例で行くと`go mod init example.com/ngicks/subgroup.git`だと勘違いしてしまうようです。
筆者がこの問題を観測した`gitlab`のバージョンだと`https://example.com/ngicks/subgroup?go-get=1`にアクセスされてもエラーなく、
なおかつ`subgroup`が`Go module`であるかのようにメタデータを返していまいます。
(ちなみに`.git`サフィックスのついたパスで`go get`を試みても上記の`subgroup?go-get=1`にアクセスしに行くログが出てます)
その後、`git ls-remote ... example.com/ngicks/subgroup.git`を実行して、エラーが返ってくるので、そんなモジュールないよ、という終わり方をします。

筆者の利用する`gitlab`のバージョンでは`.git`サフィックスをモジュール名につけることで解決しています。

筆者はこの現象を`go 1.20`あたりから`go 1.22.1`までの間で確認しいます。なので、

- `gitlab`のバージョンによって修正されているか？
- `go tool`のバージョンが上がることで修正されているか？
- ほかの方法はあるのか？
- モノレポで複数の`Go module`が管理される場合どうなるか？

はわかりません。もしかしたら直っていないかもしれないので、同じ現象に遭遇したらこの方法で解決できるかもしれません。

実際上は

- privateな`GOPROXY`を立てたり([ここが参考になるかもしれない](https://ifritjp.github.io/blog2/public/posts/2020/2020-06-04-go-proxy/))
- uberのようにredirect用のurlを作ったり([go.uber.org](https://go.uber.org))

することでも解決することができると思いますが、それがわかる人は自力で解決できるでしょうからここではこれ以上書きません。筆者はやったことがありません。

### プロジェクトの構成

先にまとめを述べます

- Go moduleではディレクトリ=パッケージ
- ディレクトリはパッケージを1つしか持てない
  - 例外として`_test`サフィックスを付けた(e.g. `pkg`に対して`pkg_test`)テスト用パッケージを定義することはできる
- パッケージがネームスペース
  - 複数ファイルあっても同一パッケージ間なら特にimportなどなく参照しあえる
  - 当然、パッケージスコープに同名の変数/型/関数などは複数定義できない。
- mainパッケージがexecutableとしてビルドできる
- mainパッケージのmain関数がエントリーポイントになる
- mainパッケージ以外をビルドするとパッケージの型チェックができる。
- mainパッケージは任意のパスに任意の個数用意できる。
  - `go build ./path/to/main/package/`でディレクトリ指定でビルドする
    - `./`で指定する。`path/to/main...`という感じで`./`を省略するとstd libraryを指定しているとgo toolに思われる。
- main以外のパッケージはfully-qualified pathで(`<<module-name>>/path/to/directory`)importできる。
- 外部のモジュールは`go get`で取得することができる。

#### エントリーポイントを作成する

先ほどクローンしたディレクトリで、以下のようにファイルを作成し、`Go module`を初期化しましょう

説明した目標に反してgit repositoryの直下じゃなくてサブディレクトリにモジュールを作っています。これはこの記事向けのスニペットをまとめて同じrepositoryに置きたい筆者の都合です。
なので対象読者はパスはいい感じに読み替えて都合のいいパスで実行してください。

```
mkdir mod-organization
cd mod-organization
go mod init github.com/ngicks/go-basics-example/mod-organization
```

`go mod init`実行後に以下のファイルが作成されたと思います

```mod: go.mod
module github.com/ngicks/go-basics-example/mod-organization

go 1.22.0
```

このファイルが、モジュールの名前、モジュールが作成された`go version`, モジュールが動作するのに想定する`go toolchain`、このモジュールが依存するほかの`go module`などを記録するファイルとなります。
対象読者には`pyproject.toml`、`package.json`、`deno.json`に近いものというとわかりやすいかもしれません。

このファイルは`go get`や`go mod tidy`などのコマンドに編集してもらうことになるので、手で編集することは少ないです。

エントリーポイントを作成します。

```
mkdir -p cmd/example
touch cmd/example/main.go
```

ファイルの中身を以下のようにします。

```go: cmd/example/main.go
package main

import "fmt"

func main() {
  fmt.Println("Hello world")
}
```

`main` packageの`main`関数がエントリーポイントとなります。
実行ファイルの処理内容はここからすべて呼び出されるべきという関数です。
トップレベル関数で`func init()`が定義されているとそちらが先に実行されるのでこれが最初に実行される関数というわけではありません。

以下のコマンドでビルドすることができます。

```
# go build ./cmd/example
```

`linux/amd64`で実行すると`./example`が出力されます

もしくは以下のコマンドで実行します

```
# go run ./cmd/example
Hello world
```

`go run`はOS依存のtmpディレクトリにビルドして実行するショートハンド的コマンドで、毎回ビルドしてしまうので複数回実行したい場合は`go build`したほうが良いことが多いでしょう。

ちなみに、以下ではダメです。

```
# go build cmd/example
package cmd/example is not in std (/usr/local/go/src/cmd/example)
```

```
# go help packages
...
An import path that is a rooted path or that begins with
a . or .. element is interpreted as a file system path and
denotes the package in that directory.

Otherwise, the import path P denotes the package found in
the directory DIR/src/P for some DIR listed in the GOPATH
environment variable (For more details see: 'go help gopath').
...
```

とあるように、`/`や`C:\`、`.`、`..`から始まらないパスは`$(GOPATH)`以下にあるかのように解決されてしまうからです。

以下の場合はエラーなく実行できますが、パッケージが複数のファイルを含む場合うまくビルドできないことを筆者は確認しています。

```
# go run cmd/example/main.go
Hello world
```

つまり、`cmd/example`以下にファイルを足して`cmd/example/main.go`がそれを参照するようにすると

```
# cat << EOF > cmd/example/other.go
> package main
>
> var Foo = "foo"
> EOF
```

```diff: cmd/example/main.go
package main

import "fmt"

func main() {
-  fmt.Println("Hello world")
+  fmt.Println("Hello world", Foo)
}
```

以下のような感じでエラーを吐きます。

実はファイルリストだったら実行できるんですが

```
go run ./cmd/example/main.go ./cmd/example/other.go
Hello world foo
```

ファイルが増えると困りますよね？
なのでファイルパスじゃなくてパッケージで指定するとよいでしょう。

```
# go run ./cmd/example
Hello world foo
# go run ./cmd/example/main.go
# command-line-arguments
cmd/example/main.go:6:29: undefined: Foo
# go run cmd/example/main.go
# command-line-arguments
cmd/example/main.go:6:29: undefined: Foo
```

相対パスでビルドを行う場合は`./`を必ず含めて、パッケージ名で指定するとよいでしょう。

#### パッケージを分ける

モジュールの下にはパッケージという分割単位があり、`Go module`ではこれはディレクトリ(フォルダー)と一致します。

１つのディレクトリは１つのパッケージしか持てません。
ただし例外としてテスト用のパッケージは定義可能で、パッケージ名に`_test`というサフィックスをつけて定義します。

パッケージは内部の処理の関心を強く反映しているのがよいとされます。関心によってパッケージを分けましょう。

また、パッケージ名とディレクトリ名は一致しているのが望ましいとされます。
慣習的にパッケージ名は１語で済む程短いほうが良いとされます。
複数単語を含む場合でも`_`や`-`でつながず、`some_package`の代わりに`somepackage`を用いるのがよいとされます。

同じパッケージ内のファイルはネームスペースを共有しています: つまり別のファイルに同名の関数は定義できないし、別のファイルの関数や変数を利用可能です。

```
mkdir pkg1 pkg2
touch pkg1/some.go
touch pkg2/other.go
```

ファイルを以下のようにします

```go: pkg1/some.go
package pkg1

var Foo = "foo"
```

```go: pkg2/other.go
package pkg2

import (
  "fmt"

  "github.com/ngicks/go-basics-example/mod-organization/pkg1"
)

func SayDouble() string {
  return fmt.Sprintf("%q%q", pkg1.Foo, pkg1.Foo)
}
```

上記のように、ほかのパッケージで定義した内容を利用するには、`import`宣言内で、fully qualifiedなパッケージパスを書くことで、インポートします。

`Go module`は、循環インポートを許しません。つまり

```diff: pkg1/some.go
package pkg1

+import "github.com/ngicks/go-basics-example/mod-organization/pkg2"

var Foo = "foo"
```

とすると以下のように _import cycle not allowed_ エラーによりビルドできません。

```
# go vet ./...
package github.com/ngicks/go-basics-example/mod-organization/pkg1
        imports github.com/ngicks/go-basics-example/mod-organization/pkg2
        imports github.com/ngicks/go-basics-example/mod-organization/pkg1: import cycle not allowed
```

#### モジュールを取得する(go get)

以下のコマンドドキュメントを参考にすると

https://pkg.go.dev/cmd/go

```
go get <<fully-qualified-module-path>>
```

で、モジュールを取得し、`go.mod`と`go.sum`を編集できます。

例えば

```
# go get github.com/samber/lo
```

を実行すると以下のように`go.mod`と`go.sunm`にモジュール情報が追記されます。

```diff: go.mod
module github.com/ngicks/go-basics-example/mod-organization

go 1.22.0

+require (
+	github.com/samber/lo v1.39.0 // indirect
+	golang.org/x/exp v0.0.0-20220303212507-bbda1eaf7a17 // indirect
+)
```

```diff: go.sum
+github.com/samber/lo v1.39.0 h1:4gTz1wUhNYLhFSKl6O+8peW0v2F4BCY034GRpU9WnuA=
+github.com/samber/lo v1.39.0/go.mod h1:+m/ZKRl6ClXCE2Lgf3MsQlWfh4bn1bz6CXEOxnEXnEA=
+golang.org/x/exp v0.0.0-20220303212507-bbda1eaf7a17 h1:3MTrJm4PyNL9NBqvYDSj3DHl46qQakyfqfWo4jgfaEM=
+golang.org/x/exp v0.0.0-20220303212507-bbda1eaf7a17/go.mod h1:lgLbSvA5ygNOMpwM/9anMpWVlVJ7Z+cHWq/eFuinpGE=
```

`import`で各ソースコードにモジュールを導入して使用できるようになります。

```diff: cmd/example/main.go
package main

- import "fmt"
+import (
+	"fmt"
+
+	"github.com/samber/lo"
+)

func main() {
	fmt.Println("Hello world", Foo)
+	fmt.Println(lo.Without([]string{"foo", "bar", "baz"}, "bar"))
}
```

```
# go run ./cmd/example
Hello world foo
[foo baz]
```

バージョンを指定するためには以下のいずれかで指定します。
末尾の２つは全く同じ効果をもたらすので、`git-commit-hash`で指定するほうが簡単だと思います。

```
go get <<fully-qualified-module-path>>@latest
go get <<fully-qualified-module-path>>@v1.2.3
go get <<fully-qualified-module-path>>@<<git-commit-hash>>
# v0.0.0-<<commit-date-time>>-<<commit-hash>>
go get <<fully-qualified-module-path>>@v0.0.0-20230723110635-fd0b45653fa9
```

`git-commit-hash`によるバージョン指定は`github.com`とセルフホストの`gitlab`では動作しました。
しかしどこにドキュメントされているのかよくわかりませんので100%確証はないです。

#### モジュールの構成

https://go.dev/doc/modules/layout

- 実行ファイルを作成するのが主眼となるモジュールはトップディレクトリがメインパッケージになることが多い
- ライブラリとしてインポートすることもできるが、実行ファイルも提供する場合は以下のような構成になることが多い

実際にはプロジェクトの規模や意図などによってどのようにコードをオーガナイズするとよりよくなるかは変わるので、
個別の議論は避け、ほかの記事に譲るものとします。

```
.
|-- cmd
| |-- command1
| |   |-- main.go
| |   `-- other_files.go
| `-- command2
|     |-- main.go
|     `-- other_files.go
|-- package_dir (名前はふさわしいものにする)
|   `-- package.go
|-- go.mod
|-- go.sum
`-- lib.go(名前はモジュールにふさわしい何かにする)
```

### Private repositoryでソースをホストする場合

https://go.dev/ref/mod#private-modules

上記の説明より、一般公開されない、つまり特別な認証が必要な`VCS`でソースを管理し、`go get`などでモジュールをインポート/ダウンロードする場合、

- `GOPRIVATE`の設定
- その`VCS`のcredentialの適切な保存
  - `git`の場合`git`が読み込めるなにか
  - `.netrc`

を行う必要があります。

#### GORPIVATEとcredentialを設定する

##### GOPRIVATE

`GOPRIVATE`の設定は以下で行います。

```
# git repositoryのURIが https://example.com/base_path
# である場合、<<url_wo_protocol>>は`example.com/base_path`になります。
go env -w GOPRIVATE=<<url_wo_protocol>>
```

(環境変数で指定すればよいと書かれていますが、筆者はうまくいかなかったので`go env -w`で書き込んでいます。)

`GONOPROXY`, `GONOSUMDB`(`NO`であることに注意)を設定しない場合、`GOPRIVATE`がデフォルトとして使われます。
`GONOPROXY`に設定されたホストからのモジュール取得する(`direct` mode)際には相手`VCS`に合わせたコマンドが使用されます(`git`の場合`git`コマンド -> [modfetch](https://github.com/golang/go/blob/go1.22.3/src/cmd/go/internal/modfetch/codehost/git.go#L249))。そのため、credentialの設定も多くの場合必要になります。

##### credential

credentialは以下の2パターンで利用されます

- `git`コマンドを利用するとき
  - `git`コマンドが自身の設定に基づき読み込む
- http(s)で、VCSからgo moduleのメタデータを利用するとき
  - `.netrc`(windowsでは`_netrc`)ファイルが読み込まれる
  - なくても動くかもしれないので、エラーしたら設定するぐらいでいいと思います

`git credential`の適切な保存には筆者は[Git Credential Manager](https://github.com/git-ecosystem/git-credential-manager?tab=readme-ov-file)を利用しています。

- windowsの場合、[Git for windows](https://gitforwindows.org/)に付属してきますので、インストールオプションで一緒に入れます。
- linuxの場合,[Install instructions](https://github.com/git-ecosystem/git-credential-manager/blob/release/docs/install.md)に従いセットアップを行います。

[vscode]の各種`Remote` extensionが`git credential`のヘルパーをつないでホスト環境につなげてくれるような挙動をしますので、
`wsl`などの場合はそちらを利用すれば`wincred`にcredentialの保存が簡単にできます。

`.netrc`はネットワークの認証情報を保存しておくファイルらしく、[man pageを検索するといくつかのコマンドがそれらを尊重する](https://www.google.com/search?q=netrc&sitesearch=man7.org%2Flinux%2Fman-pages&sa=Search+online+pages#ip=1)のがわかります。
フォーマットは[IBMの「.netrc ファイルの作成」](https://www.ibm.com/docs/ja/aix/7.2?topic=customization-creating-netrc-file)を参考にしてください。
[\${HOME}/.netrcあるいは${NETRC}にあるのが想定される](https://github.com/golang/go/blob/go1.22.3/src/cmd/go/internal/auth/netrc.go#L79-L92)ので適切に配置してください。

`.netrc`は`git`コマンドからも読み込まれますが、`http GET <<module-uri>>?go-get=1`でgo moduleのメタデータを取得しに行く時にも読み込まれます([auth](https://github.com/golang/go/blob/go1.22.3/src/cmd/go/internal/auth/auth.go#L18-L19))。こっちはなくても動作するかもしれませんので、エラーしない限りは設定しないほうがいいかもしれないですね(平文なので)。

#### git-lfsを導入している場合はすべての環境でgit-lfsを使うように気を付ける

`git-lfs`というよりは導入有無でfetch結果のファイルコンテンツが変わってしまうプラグイン全般なのですが。

上記のような設定で`direct`モードで`Go module`が取得される場合、
`git-lfs`の導入有無で`git`からのfetch後の内容が異なることがあります。
これによってsum照合エラーで`go mod download`が失敗する現象を何度か体験しています。

基本的にはすべての環境(`Dockerfile`なども含む)で`git-lfs`を導入しておくほうがよいでしょう。

[Git Large File Storage](https://git-lfs.com/)は`git`で大きなファイルを取り扱うための拡張機能です。
`git-lfs`はhookとfilterを活用してコミット前後でトラック対象のファイルをテキストファイルのポインターに変換し、
トラックされた大きなファイルはremote repositoryではなく大容量ファイル用のサーバーに上げるような挙動になります。(参考: https://github.com/git-lfs/git-lfs, [Git LFS をちょっと詳しく](https://qiita.com/ikmski/items/5cc8b8832336b8d85429))

[`github`](https://docs.github.com/ja/repositories/working-with-files/managing-large-files/about-git-large-file-storage), [`gitlab`](https://docs.gitlab.com/ee/administration/lfs/index.html)双方とも`git-lfs`に対応しています。

開発の経緯的に、想定された用途ははゲームなどで大きなバイナリファイルを一緒に管理することのようです。
それ以外でもテスト用の大きなファイルを管理するときなどにも使うことがあると思います。

## Dockerfile

`Dockerfile`のexample.

[docker]を使うとアプリをパッケージ化して送り込んだりするのが楽になります。
プロジェクト構成の話に近いと思うので、ここに載せておきますが実際上違った方法をとったり(e.g. [ko](https://github.com/ko-build/ko)、[Bazel](https://bazel.build/install/docker-container))、対象読者にとって早すぎる話題かもしれないのでいったん読み飛ばしていただくのもよいかもしれません。

- `docker`自体の詳細は説明しません。ドキュメントに譲ります。ガイドやマニュアルは充実しています: https://docs.docker.com/guides/
- `Dockerfile`の文法自体は紹介しません。リファレンスに譲ります: https://docs.docker.com/reference/dockerfile
- `docker image build`自体の紹介はしません。リファレンスに譲ります: https://docs.docker.com/reference/cli/docker/image/build/

また

- 暗黙的に`Ubuntu`/`Debian`系のコマンド/ファイル配置が前提になっているので定義読み替えたり書き換えてください
  - 差を考慮しきれるほど筆者はlinuxに詳しくありません。申し訳ないです。

### dockerの軽い紹介

[docker]は[Container](<https://en.wikipedia.org/wiki/Containerization_(computing)>) -- アプリケーションとその依存関係をパッケージ化したもの -- のビルダー及びランタイムおよびエコシステムです。

`docker`を使うと、アプリケーションを送り込むのが楽になります。
言ってしまえば`.tar.gz`の１ファイルを`docker`のdaemon(`dockerd`)に投げつけると、アプリと起動コマンドを送り込むことができて、その後、少しずつ設定を変えながらそのアプリケーションを何個か立ち上げる、みたいなことができます。(`tar`でも送り付けられるが)実際はコンテナを効率的に送りあうための仕組みや公開のためのレジストリなど、多岐にわたる概念の集合体が`docker`、もしくは`OCI container`です。
詳しい説明はほかの記事や[docker]自体のドキュメントに譲ります。

`Dockerfile`は、そういう`Container`のひな型となる`Image`をビルドするためのレシピを記述できるものです。

`docker`(および`containerd`)自体も`Go`で書かれているので読んでみると面白いと思います。筆者はちょっとしか読めていません。

### goをビルドするDockerfile example

以下に`Go`をstatic binaryにビルドする`Dockerfile`の例を示します。
`Dockerfile`をまず述べ、各変数とbuildkitのマウントの各パラメータの意味を述べ、ビルドコマンドなどをその後に述べます。

- 企業プロキシの裏にいてもビルドできるようにします。
- private repository管理のgo moduleがあってもビルドできるようにします。
- ほぼすべてがキャッシュに乗るので初回以降はビルド時間のほとんどがdockerのメタデータ解決時間です。
- `apt-get`を使いますが、この部分はキャッシュしません。distro/バージョンで差が大きそうな気がしてます。
  - キャッシュしたい人は[misskeyのDockerfileのここ](https://github.com/misskey-dev/misskey/blob/43cccaaee9be42fab38eaa9ca04bb5e55b5d8db7/Dockerfile#L9-L15)とかが参考になるかも

筆者はおおむねこれでうまくいっていますが、何かがあれば、static binaryに実はならないとか、そういった問題点があるかもしれないので、読者の環境に向けてカスタマイズする必要があるのは当然述べておくべきでしょう。

コードはここに置いてあります: https://github.com/ngicks/go-basics-example/tree/main/dockerfile

#### Dockerfile

```Dockerfile
# syntax=docker/dockerfile:1.4

# 上記で新しいsyntaxであることをビルダーに伝える。
# 新しい構文を使うとき、
# なぜかなくても動いたり動かなかったりする環境があってややこしいので
# とりあえず書く。

FROM golang:1.22.3-bookworm AS builder

ARG HTTP_PROXY
ARG HTTPS_PROXY
ARG GOPATH=/go
ARG CGO_ENABLED=0
ARG MAIN_PKG_PATH=.

# WORKDIRの決め方やビルドしたバイナリの置き場所はこれがいいよという自信がない。
# 必要に応じて変えてください。
WORKDIR /usr/local/container-bin/src
# git-lfsの有無でgit fetch結果が異なり、sum照合エラーになることがある。
# Private go moduleをdirect modeでgo getするならば、すべての環境に入れておくほうが安全。
# apt-getでバージョン指定をするとすぐに古いパッケージが消えるのでバージョンは固定しない。
# バージョンを固定したい場合はdebファイルを保存して
# そこからインストールしたり、ソースからビルドする。
RUN --mount=type=secret,id=certs,target=/etc/ssl/certs/ca-certificates.crt\
    apt-get update && apt-get install -yqq --no-install-recommends git-lfs
# 先にgo mod downloadを実行する
# buildkitでマウントするキャッシュ以外に変更が起きない。
# (/root/.cacheと/root/.config/goにマウントされるのでディレクトリは作成される)
# Dockerのimage layerとしてキャッシュするというより、
# コマンドの失敗する点を切り分けてエラーを見やすくする意図がある。
COPY go.mod go.sum ./
RUN --mount=type=secret,id=certs,target=/etc/ssl/certs/ca-certificates.crt\
    --mount=type=secret,id=.netrc,target=/root/.netrc\
    --mount=type=secret,id=goenv,target=/root/.config/go/env\
    --mount=type=cache,target=/go\
    --mount=type=cache,target=/root/.cache/go-build\
    go mod download
# COPY . .をしてしまうとbuildkitの遅延ファイル要求の利点がすっ飛びますが、全部送らざるを得ない
# ソース以外のコンテンツがいろいろ含まれる場合は、`.dockerignore`などをちきんと整備してください。
# https://docs.docker.com/build/building/context/#dockerignore-files
COPY . .
RUN --mount=type=secret,id=certs,target=/etc/ssl/certs/ca-certificates.crt\
    --mount=type=secret,id=.netrc,target=/root/.netrc\
    --mount=type=secret,id=goenv,target=/root/.config/go/env\
    --mount=type=cache,target=/go\
    --mount=type=cache,target=/root/.cache/go-build\
    go build -o ../bin ${MAIN_PKG_PATH}

# distrolessはtagの中身が入れ替わるので再現性を優先するならsha256で指定したほうがよい
FROM gcr.io/distroless/static-debian12@sha256:41972110a1c1a5c0b6adb283e8aa092c43c31f7c5d79b8656fbffff2c3e61f05

COPY --from=builder /usr/local/container-bin/bin /usr/local/container-bin/

ENTRYPOINT [ "/usr/local/container-bin/bin" ]
```

#### 各変数の説明

Dockerfile中の`ARG`はビルド時に`--build-arg ${NAME}=${VALUE}`で変数を引き渡せます。
各変数の名前と説明は以下に

| 変数          | 説明                          |
| ------------- | ----------------------------- |
| HTTP_PROXY    | proxyがある場合に             |
| HTTPS_PROXY   | 同上                          |
| GOPATH        | 基本は変えない                |
| CGO_ENABLED   | 0にするとスタティックバイナリ |
| MAIN_PKG_PATH | ビルド対象のパッケージパス    |

- `Go`のhttp clientはデフォルトで環境変数をよみこんでProxyにアクセスするので、`${HTTP_PROXY}`か`${HTTPS_PROXY}`を設定しておけばよいです。
  - https://github.com/golang/go/blob/go1.22.3/src/net/http/transport.go#L44

buildxのマウント機能を使って各種ファイルやキャッシュをマウントできます。
`secret`は`--secret id=${ID},src=/path/to/file`でファイルをマウントできます。
名前の通り機密情報(e.g. `.netrc`)をimageにコピーしないで利用できるようにするためのマウントなのですが、本来の用途に反して単純にファイルがマウントできる方法としても使っています。

それぞれの意味は以下に

| mount type | id                    | 説明                                                           |
| ---------- | --------------------- | -------------------------------------------------------------- |
| secret     | cert                  | PROXYがオレオレ証明書の場合root ca bundleを渡す                |
| secret     | .netrc                | `go get`とか`git ls-remote`とかのための認証情報                |
| secret     | goenv                 | `go env -w`で生成できるファイル。`GOPRIVATE`とかを入れておく。 |
| cache      | /go                   | ほかになにも設定しなかったら`go get`した内容がキャッシュされる |
| cache      | /root/.cache/go-build | ビルドキャッシュがここに入るらしい                             |

- `.netrc`は`git`やgo toolそのものから読み込まれます。private gitlabなどにアクセス必要なとき渡しますが、いらないなら空のファイルでもいいです。
  - `.netrc`ファイル自体のフォーマットは[ここ](https://www.ibm.com/docs/ja/aix/7.2?topic=customization-creating-netrc-file)などを参考に
- certはlinuxだとこのパスが問答無用で読み込まれるので、`Ubuntu`/`Debian`系以外でもこのパスでいいはずです。
  - https://github.com/golang/go/blob/go1.22.3/src/crypto/x509/root_linux.go#L9-L17
  - もちろんディストロに合わせたパスに置かないと`apk`や`yum`などのパッケージマネージャーが読み込めない可能性があります。

#### ビルドコマンド

`Dockerfile`と同階層で以下のコマンドを`./build.sh ${REPO}:${TAG}`で実行することで、`${REPO}:${TAG}`な`docker image`をビルドできます

```shell: build.sh
#! /bin/sh

docker buildx build\
    --build-arg HTTP_PROXY=${HTTP_PROXY}\
    --build-arg HTTPS_PROXY=${HTTPS_PROXY}\
    --build-arg MAIN_PKG_PATH=${MAIN_PKG_PATH:-./}\
    --secret id=certs,src=/etc/ssl/certs/ca-certificates.crt\
    --secret id=.netrc,src=${DOTNETRC_PATH}\
    --secret id=goenv,src=$(go env GOENV)\
    -t $1\
    -f Dockerfile\
    .
```

#### キャッシュの効果

上記コマンドに`--target=builder`オプションを付け足してbuilderステージまでをビルドして[dive](https://github.com/wagoodman/dive)で中身を検査してみましたが、モジュール、ビルドキャッシュともにキャッシュできていることがわかります。

![dive-checking-go-cache-effectiveness](/images/dive-checking-go-cache-effectiveness.jpg)

#### 実行

ジョークなので`./build.sh joke:joke`でイメージをビルドしました。
実行してみると正常に動作しています。

```
$ docker container run --rm joke:joke
🐤< ｺﾝﾆﾁﾊ！ ₍₍⁽⁽ 🐧₎₎⁾⁾ ₍₍⁽⁽🐔₎₎⁾⁾ ₍₍⁽⁽🐣₎₎⁾⁾ ₍₍⁽⁽🐓 ₎₎⁾⁾
```

鳥が踊ります。

## おわりに

公式が提供する読み物を紹介し、モジュールを作って実行する手続きと、追加の話題として`Go`向けの`Dockerfile`を紹介しました。

private repositoryでモジュールを作るときの手順は筆者は割と躓いたので重点的に述べておきました。

- 筆者は`gitlab`でしかprivate go moduleを作ったことがないので、別な環境では別な躓き方をするかもしれません。
  - なのでなるだけ`go get`コードの内部的な現象にフォーカスを当てました。応用が効けばいいのですが・・・。
- 企業Proxyのかかった環境で`docker image build`するのも結構大変だったので、これも述べておきました。
  - まだ足りない何かがあったらぜひ教えてほしいです。

なるだけ資料をあたり、リファレンスとソースを読み込んで情報を集めましたが、
扱う話題が広いので間違ってたり、もっといい方法がある可能性も十分あると思います。

- part1 プロジェクトを始めるまで編: これ
- [part2 cliアプリをつくれるところまで編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part2)
- [part3 concurrent GO編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part3)
- [part4 HTTP Server/logger編](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part4)

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
[docker]: https://www.docker.com/
