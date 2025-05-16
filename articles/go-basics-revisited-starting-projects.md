---
title: "Goのプラクティスまとめ: プロジェクトを始める"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## Goのプラクティスまとめ: プロジェクトを始める

筆者が`Go`を使い始めた時に分からなくて困ったこととか最初から知りたかったようなことを色々まとめる一連の記事です。

[以前書いた記事](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part1)のrevisited版です。話の粒度を細かくしてあとから記事を差し込みやすくします。

他の記事へのリンク集

- (まだ)~~[今はこうやる集](https://zenn.dev/ngicks/articles/go-basics-revisited-updated-practices)~~
- プロジェクトを始める: ここ
- (まだ)~~[dockerによるビルド](https://zenn.dev/ngicks/articles/go-basics-revisited-bulding-with-docker)~~
- [error handling](https://zenn.dev/ngicks/articles/go-basics-revisited-error-handling)
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

## プロジェクトを始める

新しいプログラミング言語、フレームワーク、ライブラリー、ツール、etcを始めるとき筆者にとってよく障害となるのは「プロジェクトを始めるまでの方法がわからない」ということです。

そこで、この記事では[Go]そのものの紹介、SDKのインストール、エディターのセットアップ、プロジェクトの開始方法(=moduleの作成方法)、さらにprivate repositoryで管理されるgo moduleをインポートできるようにする方法、タスクランナーなどについてまとめます。

## 前提知識

- ほか言語での開発経験
  - [python]や[Node.js]などを引き合いに出すことがあります。
  - ある程度ソフトウェア開発における常識感を読者が持っているのを前提とした説明をします。
    - 誰でもわかるように書くと論文みたいになって長くなる傾向がありますし、裏取りの手間が大きくなりすぎるためです。
    - 「常識感」についても共有されていないと曖昧性が増すため筆者の知識のバックグラウンドも簡単に書きます。

## 環境

win11のwsl2インスタンス内で動作させます。ほかの環境でも動く気がしますが、以下が前提となります。

```
> wsl --version
WSL バージョン: 2.4.12.0
カーネル バージョン: 5.15.167.4-1
WSLg バージョン: 1.0.65
MSRDC バージョン: 1.2.5716
Direct3D バージョン: 1.611.1-81528511
DXCore バージョン: 10.0.26100.1-240331-1435.ge-release
Windows バージョン: 10.0.26100.3775
```

[Go]は現在(2025-05-05)の最新です。

```
$ go version
go version go1.24.2 linux/amd64
```

`Go`は後方互換性をかなり気にするためサンプルコードや述べられていることは以降のバージョンでも（何ならこれよりの前のバージョンでも）基本的に一緒ですが、細かいところで改善が入ったりするのでそれを踏まえて読んでください。

## snippetのconvention

- shellコマンドの羅列の場合コピペしやすさを優先して特になにもprefixをつけずにコマンドを羅列します。
- ただし、コマンド実行結果をsnipet内の併記したい場合は、コマンドは`$ `でprefixされます。
- `# `から始まる行はコメントです。

## 筆者のバックグラウンド

学生時代に

- [Verilog](https://en.wikipedia.org/wiki/Verilog)(HDL)でAltera製の[FPGA](https://en.wikipedia.org/wiki/Field-programmable_gate_array)の回路を記述していた
- [C++]\(Visual Studio Community 2019だったと思う\)を使ってセンサーから値を読み込んで計算を行うプログラムを作っていた
  - 四元数で回転を計算するのに[Eigen](https://eigen.tuxfamily.org/index.php?title=Main_Page)を使っていました。
  - GUIをつけるのに[MFC](http://msdn.microsoft.com/ja-jp/library/d06h2x6e.aspx)を使っていました。

`FPGA`を使ったプロジェクトはとん挫したので遊びみたいなものです。
機械や制御を専門とする学徒であったためこの時点ではソフトウェアに詳しくはありませんでした。とくに`C++`に関しては研究室に置いてあった古い本を読みながらやったので当時からしても古い書き方をしていたと思います。

社会人になってから

- [Node.js]\([TypeScript]\)\:
  - 業務アプリの一部みたいなやつ、APIサーバーとかを書いてました。
  - イベントループがあるのがいいなあと思ったんですが、仕様の過渡期とかがつらくて離れました。
    - `Node.js`の`commonjs module` -> `ECMA module`
    - `Node.js stream` -> `WebStream`
    - -> `fetch` などなど
  - もうすぐ移行が終わりそうなので楽しみにしてます。
- ちょっとだけ[python]\:
  - 業務改善のためのツールに使用してました。`Excel`をいじくるやつです。実行ファイルに固めてくれと言われてこりゃしんどいわとなってのちに[Go]で書きなおしています。
  - `python`のasyncについて教えるためにasync周りの`CPython`の実装を読んでます。おもしろいです。
- [Rust]\:
  - `C/C++`で書かれたライブラリのbindingを書いて使ってた程度であんまりメインで使ってるわけではないです。
  - `PDFium`へのbindingを書いてました。[pdfium_rs](https://github.com/asafigan/pdfium_rs)をフォークして拡張する形でやってました。楽しかったなあ。
  - [deno]のruntimeが[Rust]\([tokio]\)なので必然的に読む機会は多いです。
- [Go]\: 仕事でも趣味でも使っています。APIサーバーを書いたり、`Docker API`を通じてコンテナ状態をいじったり、画像をいじくったりいろいろやってます。

別に書かないですが

- [Java]\: 学生時代`Swing JAVA`でUIを作る講義がありました。[Elasticsearch]のソースを読んだりしてます。
- [C]\: `linux` kernelとか`busybox`とかの詳細がわからなくて困ることがあるので読まざるを得ません。プロジェクトによってはマクロでオブジェクト指向みたいなことしてて毎回ビビらされます。`QEMU/virsh`, `OpenSSL`,などなど重要なツールはどれも`C`で書かれている気がします。が、最近は[Rustに移行するもの](https://discourse.ubuntu.com/t/carefully-but-purposefully-oxidising-ubuntu/56995)も増えてきましたね。

少し特殊な環境で動くソフトウェアを書くためデータベース周りの話にあまり明るくなかったりします。
手元でスクリプティングを行う場合はもっぱら[neovim]の[Lua]、[deno], [Go]のいずれかで行っています。

## 基本: Goとは

### 特徴

> https://go.dev/doc/
>
> The Go programming language is an open source project to make programmers more productive.
>
> Go is expressive, concise, clean, and efficient. Its concurrency mechanisms make it easy to write programs that get the most out of multicore and networked machines, while its novel type system enables flexible and modular program construction. Go compiles quickly to machine code yet has the convenience of garbage collection and the power of run-time reflection. It's a fast, statically typed, compiled language that feels like a dynamically typed, interpreted language.

これよりもう少し説明すると、[Go]は

- C系統の文法で
- 静的型付けで
- 言語に組み込まれた`goroutine`, いわゆる[Green Thread](https://en.wikipedia.org/wiki/Green_thread)の機能があります。
  - これを呼び出すには`go`キーワードの後に関数やmethodの呼び出しを書くだけです。簡単です。
  - `goroutine`のランタイムがOS ThreadをCPU個数と同数（正確には違う）作っておいて、それらの上で`goroutine`をスケジュールします。
  - CPUと同数個のOS Threadを作っておいて、そのうえでタスクを動かすのは[Node.js]も似たようなコンセプトを持っていますが、`goroutine`は普通のOS Threadかのようにユーザーには見えるという違いがあります。
  - [Rust]には`async/await`構文があり、[tokio]などのランタイムを用いると似たようなことができます([deno]は[tokio]を使用しています)が、こちらは(`async fn`は)stackless statemachineになる方式、`goroutine`はそれぞれがstackを持っていますので違いがあります。
  - cooperativeなタスクの切り替えしかできないかと思いきや[Go 1.14]からpreemptiveな切り替えにも対応しています。
- GC(Garbage Collector)があり、手動でのメモリ確保・解放はほぼやることはありません。
- 言語仕様が簡潔で、言語としてサポートされた構文は多くありません。
  - [spec#Keywords](https://go.dev/ref/spec#Keywords)より、keywordは25個とほかの言語と比べて少ないです。
  - 例えばほかの言語でよくある`while`はなく、代わりにcondition部分のない`for { /* do anything */ }`を用います。
- コンパイルが*遅くならない*ように気が遣われています。
  - 機能も構文もあまりコンパイルが遅くならないように気遣って追加されるようです。proposalを読んでるとコンパイルの速度が遅くならない方式だからこの方法を採用する、みたいな言及があります。
  - `go run ./path/to/main`を実行すると、毎回ビルドしてから実行するんですが、こうしても気にならないほどにはコンパイルは速いです。
- classや継承はなく、`interface`による抽象化を行います。
  - `interface`は特定の`method set`を実装する型なら何でも代入できる型という意味になります。
  - `interface{}`(なんのmethodも指定しない`interface`)にはあらゆる型の変数を代入可能です。上記の*feels like a dynamically typed, interpreted language*の部分はこのこともさしているのだと思います。
- methodは、任意の型をベースとした型を定義し、それに関連した関数として実装します。
  - `type A struct { Foo string; Bar int }`や`type B string`のように、別の型をベースとして型を定義できます
  - もちろん、`type C A`も可能です。
  - `func (a A) MethodName() {}`という構文でmethodは定義できます。ここでいう`a`がほかの言語でいう`self`とか`this`です。
    - 同じpackage内なら複数のファイルに分散して定義したりできます。
- method、関数、closureはすべて区別なく変数や引数に代入可能です。
- 組み込み型として`array`(`[5]T`)、`slice`(`[]T`) = 動的にサイズが変更できるarray;ほかの言語のvectorに近いもの、`map`(`map[K]V`) = hash map,`chan`(`chan T`) = channel(`goroutine`-safeな通信ができる)があってこれらはfor-range構文が対応していたりと特別扱いされます。
- moduleシステムが組み込まれています。
  - moduleの公開には特別な処理は必要ありません。`github`のようなVCS(Version Control System)で公開し、`Go module`の名前を`https://`抜きのURLにすればそれで公開できます。
- pointerは存在しますが、pointer arithmeticは(`unsafe`を使わない限り)できません
- 非常に簡単に別のCPUアーキテクチャ、OS向けのビルドを行うことができます。
  - ただし`CGO`(FFI)を使わない場合
- 非常に簡単にstaticにビルドできます。
  - ただし`CGO`(FFI)を使わない場合

よくいろいろないといわれますが

- genericsは[Go 1.18]で実装されました
- iteratorは[Go 1.23]で実装されました
- `?`構文は[dicussion](https://github.com/golang/go/discussions/71460)に入ってるので1～2年以内に実装されるかも！

### A Tour of Go

https://go.dev/tour/welcome/1

`Go`の基本的な文法、機能などのトピックはここに書いてあります。
インタラクティブなコードスニペットと簡単なエクササイズがあり、これさえこなせばとりあえず開発は始められます。
**以後の文章はA Tour of Goをすべてこなしたことを前提とします。**

慣れてない頃に、syntax highlightのかからないwebページでコードを書くのはきついと思いますが、軽く調べた限り任意のエディターで実行する簡単な方法はなさそうです。

筆者の記憶にある限り筆者は合計6時間ぐらいですべて終わりました(当時はまだgenerics実装前だったので今はもう少し長い)。
時間のほとんどは構文エラーがよくわかんなくて費やされたのでローカルのエディターにコピーして書いてコピーしなおして実行すればもっと早く終わるかもしれません。

### Go by Example

https://gobyexample.com/

コード例とともに解説がされます。
項目数が多く、知らないstdmoduleを使う部分をきちんと理解しようとすると時間がかかると思います。
手が空いたら少しずつ読むのがいいのではないでしょうか。

### 公式読み物系

公式が出している読み物集。全部英語です。
内容が古くなりつつあるため、始めたてで読むべきかは微妙ですが、古くなっても参考になる部分が大いにあります。

- How to Write Go Code: https://go.dev/doc/code
  - コードの書き方以外も含めた基本的なトピック
- Effective Go: https://go.dev/doc/effective_go
  - Goのイディオム集
  - 現在(2025-05-05)だいぶ古くなってきているので、一通り読んだらほかのドキュメントも読むようにしてください
- Go Wiki: Go Code Review Comments: https://go.dev/wiki/CodeReviewComments
  - よくされるCode review comment集らしいです

そのほかにもいろいろなトピックが以下に掲載されています。

https://go.dev/doc/

### The Go Programming Language Specification

https://go.dev/ref/spec

言語仕様ですが割と短めなのでそのうち読んでおいたほうがよいでしょう。

### Std library

https://pkg.go.dev/std

standard libraryです。

HTTPなどで動作するサーバープログラムを作るのに大体必要な機能がそろっています。
できれば開発に着手する前にさっくりどういう機能がstdとして存在するかを把握しておくほうがよいでしょう。

`CGO`と[SWIG](https://swig.org/)への言及があるなど、読んどけばよかった系の文章が意外なほどたくさん書いてあります。

### Sub-repositories

https://pkg.go.dev/golang.org/x

Sub-repositoriesです。

説明のとおり、Go Projectの一環ですがstd libほど厳密なバージョン管理がされていません。
std libに入ると厳密な後方互換性の約束を守る必要があります。
そのため変更の可能性が高かったり、stdに入れるほどの重要度がないものがこちらにあるというコンセプトのはずです。

- ここで先に実装されてからstdに昇格されたり(`maps`, `slices`など)、
- stdがFrozenなので代わりにこちらのものを使うべきだったり(`syscall`の代わりに[golang.org/x/sys](https://pkg.go.dev/golang.org/x/sys#section-readme))
- 1ファイルにバンドルされてstdに組み込まれていたり([golang.org/x/net/http2](https://pkg.go.dev/golang.org/x/net/http2))
- 古い`Go`でも利用できるようにstdへの追加がこちらにも入れられる(`encoding/json/v2`など)

することもあります。

### golang/example

https://github.com/golang/example

公式でメンテされているexample集。
`go/types`周りの話など結構アップデートされている部分もある。

## SDKのインストール

公式の手順に従い、各OS環境に合わせて[Go]をインストールしましょう。

https://go.dev/doc/install

### ダウンロード

`linux/amd64`の場合、

```
VER=1.24.2
mkdir -p /tmp/go-download
curl -L https://go.dev/dl/go${VER}.linux-amd64.tar.gz -o /tmp/go-download/dist.tar.gz
```

一応チェックサムを確認しておいたほうがいいかもしれません。

```
# checksumの値はダウンロードページから確認できる。
CHECKSUM=68097bd680839cbc9d464a0edce4f7c333975e27a90246890e9f1078c7e702ad
ACTUAL=$(sha256sum /tmp/go-download/dist.tar.gz | awk '{print $1}')
[[ $CHECKSUM = $ACTUAL ]]; echo $?
```

bashじゃないと動かないかも。一致してれば`0`がprintされます。

### /usr/local以下に入れる場合

公式は`/usr/local`以下に入れる方法を案内しています。

```
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xf /tmp/go-download/dist.tar.gz
```

### ~/.local以下に入れる場合

共有サーバーでユーザーにsudo権限をつけたくないときなど、`$HOME`以下へインストールしたいこともあると思います。
`~/.local`以下にインストールしても全く問題なく動作します。

```
rm -rf ~/.local/go && tar -C ~/.local -xf /tmp/go-download/dist.tar.gz
```

以降は`~/.local/go`に入れてある前提で書かれます。

### PATHを通す

環境変数を設定。使っているOS/terminalに合わせた方法で設定してください。

筆者は以下のようなbashscriptを組んでこれを`.bashrc`から呼び出しています。

```shell
#!/bin/bash

gobin=~/.local/go/bin/go

export PATH=$($gobin env GOROOT)/bin:$PATH

if [[ -n $($gobin env GOBIN) ]]; then
    export PATH=$($gobin env GOBIN):$PATH
else
    export PATH=$($gobin env GOPATH)/bin:$PATH
fi
```

[Go Modules Reference#go-install](https://go.dev/ref/mod#go-install)より、`go install`でビルドされたバイナリは`$GOBIN`以下、もしくは`$GOPATH/bin`に保存されます。
そちらにもPATHを通しておけばコマンドとして利用できるようになります。

例として`lazygit`を`go install`してみましょう。

```
go install github.com/jesseduffield/lazygit@latest
which lazygit
# 筆者の環境では`$(go env GOPATH)/bin/lazygit`が表示されます。
```

環境変数`GOBIN`もしくは`go env -w GOBIN=/path/to/gobin`で`GOBIN`が設定されている場合はそこ以下にバイナリが保存されます。

## エディタのセットアップ

エディタは個人の好みともろもろを合わせて好きに選べばいいと思います。
ただよく聞く+[goのsurvey](https://go.dev/blog/survey2024-h2-results#editor-awareness-and-preferences)の上位3位は以下の3通りです。

- [Visual Studio Code] + [Go extension](https://marketplace.visualstudio.com/items?itemName=golang.go)
  - https://code.visualstudio.com/docs/languages/go
- JetBrainsの[GoLand](https://www.jetbrains.com/ja-jp/go/)
- [vim](https://www.vim.org/) or [neovim] + [gopls]
  - 公式に案内される方法: https://github.com/golang/tools/blob/master/gopls/doc/vim.md
  - `neovim`(v0.11.0)では
    - 適当なpackage managerで[neovim/nvim-lspconfig](https://github.com/neovim/nvim-lspconfig)を`rtp`に加えておく。
    - `neovim/nvim-lspconfig`の設定では不足ある場合は、`~/.config/nvim/after/lsp/gopls.lua`を作成して適当に設定を上書きする。
    - どこかの`lua`スクリプトから[vim.lsp.enable](<https://neovim.io/doc/user/lsp.html#vim.lsp.enable()>)`("gopls")`を呼び出す。
    - `gopls`をインストールして`$PATH`を通す。
      - [williamboman/mason](https://github.com/williamboman/mason.nvim)の`ensure_installed`に`"gopls"`を指定しておくとよい。
    - 筆者は[LunarVimの設定](https://github.com/LunarVim/starter.lvim/blob/a3b559561808cc52a2bd32b6a85da6c61a0a44a1/config.lua)や、[AstroNvimの設定](https://github.com/AstroNvim/astrocommunity/tree/baaaef19ddd6204b496dc42e465bce9e051fc95e/lua/astrocommunity/pack/go)をよくパクっています。

設定は大して難しくないので省略します。案内に従ってください。
例外として`vim`,`neovim`の言語サーバーの設定は難しいと思いますが、これらをあえて選ぶ人物は自分で調べる能力が高いでしょうからやはり省略します。

goのsurveyはself-selection, `vscode`の`Go extension`からの誘導, `GoLand`からの誘導の経路があるのでバイアスがかかっているのは間違いないと思いますが感覚的にこの3つが上記の順で多いのは間違いなさそうに感じます。

## Go moduleの作成

- [Go]ではプログラムは1つまたは複数の`Go module`で構成されます。
  - `Go module`は[Go 1.11]で導入されましたが、今日において`Go module`を用いず`Go`で開発することはほぼあり得ません。
- `go mod init <<module-name>>`でmoduleを初期化します。
  - `go.mod`が生成されます。これがあることで`go tool`に`go module`として認識されます。
  - `<<module-name>>`はVCS repositoryのURIからprotocol schemeを抜いたものにするとよいです。
    - こうすると、`go get <<module-name>>`でほかの`go module`からimport可能になります。
- とりあえず`Go module`を作る
  - 実行ファイルを作りたくても、
  - ライブラリを作りたくても
  - 実行ファイルにもなるし、ライブラリとして公開されるものでも
- moduleはさらにpackageと呼ばれる単位に分割されます。
- 1 directory = 1 packageです
  - 例外的にmain以外のpackageは`_test` suffixをつけた(e.g. `pkg`に対して`pkg_test`)テスト用packageを定義することができる
- packageはnamespaceを共有します。
  - 複数ファイルにわたって同じ変数、関数にアクセスできます
  - ほかの言語、例えば[Rust]では1つのファイルが1つのmoduleであり、ファイル内でもmoduleを定義することができますが、逆に`Go`ではこのように任意に分割する方法はありません。
- main packageが実行ファイルとしてビルドできる
  - main packageのmain関数がエントリーポイントになる
  - main packageは任意のパスに任意の個数用意できる。
- `go build ./path/to/main/package/`でディレクトリ指定でビルドする
  - 必ず`./`から始まるパスを指定する。ないとstd libraryを指定していると`go tool`に思われる。
- main以外のpackageはfully-qualified path(`<<module-name>>/path/to/directory`)でimportできる。
  - 相対パスを用いることもできるが、ほぼされることがないためしないほうが良いのではないかと思う。
- 外部のmoduleは`go get`で取得することができる。

### 事前にgit(VCS)でrepositoryを作成しておく

手順そのものは説明しませんが、以下の手順は[github](https://github.com/)や[gitlab](https://about.gitlab.com/)でrepositoryが存在していることを想定します。

[VCS(Version Control System)](https://en.wikipedia.org/wiki/Version_control)は, コンピュータファイルのバージョンを管理するシステムのことです。
代表的なものは[git]や[svn](https://en.wikipedia.org/wiki/Apache_Subversion),[mercurial](https://en.wikipedia.org/wiki/Mercurial)あたりだと思います。
この記事では`git`のみを取り扱います(筆者がほか二つのことをほぼまったく知らないからです)

`git`は、`VCS`を構築するためのサーバーおよびクライアントプログラムです。サーバーとして直接使うことはほとんどないかもしれません。
現在では`git`サーバーは[github](https://github.com/)というwebサービスを利用するか、 セルフホストすることも可能な[gitlab](https://about.gitlab.com/)、あるいは[gitbucket](https://github.com/gitbucket/gitbucket)などを使うのが一般的だと思います。(この3つがリストされてるのは単に筆者が使ったことあるやつ3種っていうだけです)

先にローカルでrepositoryを作成してあとからremote上に作成する方法もあるはずですが、この説明ではremoteがすでに存在していることを前提とします。

### Go moduleの初期化

`VCS`で作成したrepositoryをローカルにcloneします。

clone先はどこでもいいですが、個人的なおすすめは`<<uri>>`から`https://`などのprotocol schemeを外したパスで`~/`以下にディレクトリを切り、そこにcloneする管理方法です。

```
uri=<<uri>>
dest=$HOME/gitrepo/$(echo $uri | sed 's/\.git$//' | sed 's/^git@//' | sed 's/^https\:\/\///')
mkdir -p $dest
git clone $uri $dest
# clone先に移動しておきます
pushd $dest
```

`go mod init <<module-name>>`で`Go module`に必要なファイルを作成します

```
go mod init <<module-name>>
```

`<<module-name>>`は基本的に上記`<<uri>>`からprotocol schemeを抜いたものにするとよいです。
そうすると`go get <<module-name>>`でこのmoduleを別の`Go module`へ導入できるためです。

例えば、`VCS`のuriが`https://github.com/ngicks/go-example-basics-revisited`である場合、

```
go mod init github.com/ngicks/go-example-basics-revisited
```

で作成し、`VCS`にソースをプッシュすると

```
go get github.com/ngicks/go-example-basics-revisited
```

で別moduleから導入、参照できます。

:::details local onlyのmodule

ローカルオンリーなつもりならおおむね`<<module-name>>`はなんでもよいはずです。

```
go mod init whatever-whatever
```

筆者が知る限りほかのmoduleから`go get`できなくなる以外の違いはないです。

ただし、[How to write code](https://go.dev/doc/code)でも勧められる通り、なるだけ公開される前提でmoduleを作っておいたほうがよいでしょう。

:::

#### private VCSかつサブグループを使用する場合`.git`などでmodule nameをsuffixしておく

ただし、private `VCS`(=URIが`go tool`に対して既知でない)でサブグループを作成し、サブグループの中でソースを管理する場合、`<<module-name>>`はvcsのsuffixを加えておかないと`go get`時に失敗するかもしれません。

つまり上記と同じ例で行くと

```
# 架空のURLを扱うのでexample.comに変えてあります！
go mod init example.com/ngicks/subgroup/go-example-basics-revisited.git
```

とする必要があるということです。

なぜかというと

- private repositoryを参照できるgo module proxyサーバーが存在しないとき、`go tool`は`git ls-remote`などのvcsコマンドを直接実行します。
  - これは`direct` modeと呼ばれます。
- https://pkg.go.dev/cmd/go#hdr-Remote_import_paths より、
  - module nameに`VCS` suffix(e.g. `.git`, `.hg`, `.svn`) がついていると、このsuffixまでをmodule root pathとする
  - ない場合、`go tool`に渡されたパスに対して`?go-get=1`付きでHTTP GETを行い、responseからmodule root pathを得ます。
  - (余談ですがログを見る限り`VCS` suffixがついていても`?go-get=1`付きのHTTP GET自体はされます)
- `go tool`に渡されるパスはmodule root pathとは限らないため、たとえ`go get module/root/path`をしたとしても、subgroupが含まれているとパスの探索が起こります。
  - e.g.
    - `go install`: `<<module-name>>/cmd/command-name`のようにsub-packageにmain packageを作ることが多い。
    - mono-repo: 1つの`VCS` repositoryに対して複数のgo moduleをホストすることができますのでmodule root = gitなどでcloneする対象とも限りません。
- (少なくとも)`gitlab`では(おそらく)あらゆるパスに対して`?go-get=1` query paramをつけたHTTP GETが成功します
  - 返ってくる内容は`go tool`の期待に反してmodule root pathではなく、アクセスされたURLのパスをオウム返しするだけです。
  - おそらくセキュリティーのためです(後述)。そのためおそらくどのVCSでもこうなっているんじゃないでしょうか

というのが理由です。

[Go 1.24]まで、`.netrc`を用いる以外に`go tool`にcredentialを渡す方法がなく、そのため`?go-get=1`は認証情報なしでリクエストされることがほとんどだったと思われます。
少なくとも筆者の環境では`.netrc`を作成していませんがprivate repositoryから`go get`が成功していました。ということは認証なしで`?go-get=1`付きのリクエストはとりあえず成功するようにできていたんだと思います。404とか403みたいにエラー種がパスによって変わるとどういうパス構成なのかが外部からばれてしまうため、"正しい"内容を認証していない状態で返すわけにはいきませんので、こういう挙動になっていたのでしょう。
`direct` modeで`git`コマンドを直接用いる際には`git`コマンドのcredentialがそのまま利用されますから、ここは広く用いられる方法でcredentialを渡すことができます。
後方互換性のことを考えるとこの挙動が変わることはまずない気がします。

https://gitlab.com/gitlab-org/api?go-get=1 はPublicですが、これも与えられたパスをオウム返しする挙動であるのでPublicであってもうまく動かないと思われます。このURLが指し示す先はサブグループなので500番台とか400とかを返すのが正しい挙動に思えますね。

もし仮にmodule nameに`VCS` suffixをつけたくないとしたら、好ましい解決方法は

- サブグループを使わない
- vanity import pathを返すサーバーを運用する
- private repositoryを参照できるgo module proxyを運用する

だと思います。
筆者は`VCS` suffixをつける方法に甘んじているためすべて試していないことに注意願います。

- サブグループを使わない:
  - 単純ですが、サブグループを使わないだけでも解決します。
  - https://github.com/golang/go/blob/go1.24.2/src/cmd/go/internal/vcs/vcs.go#L1523-L1585 などより、`VCS` suffixがない場合、`<<host>>/<<user>>/<<repository>>`というフォーマットとして解釈され、このパスにまず`?go-get=1`付きのHTTP GETが試みられます。前述のとおり少なくとも`gitlab`ではこれは成功するため意図通りに動作すると思われます。
- vanity import pathを返すサーバーを運用する:
  - 例えば[go.uber.org/zap](https://github.com/uber-go/zap)は実際にはgitubでホストされています。
  - `curl https://go.uber.org/zap/zapcore?go-get=1`を実行するとわかりますが、`<meta name="go-import" content="vanity/module/path VCS https://path/to/VCS">`が返ってきます。
  - このようにmodule pathと実際にソースコードがホストされるURLが違う時、module pathをvanity import pathなどと呼ぶのが通例のようです。
  - `?go-get=1`で正しい内容が返りさえすれば(=module root pathが正しく取れれば)あとは成功するでしょうから、これでもうまくいくと思われます。
  - (余談ですが、mono-repoかつvanity import pathを用いたい場合module proxyを用いなければうまく動作しない気がしています。)
- go module proxyを運用する:
  - 最も正道で最も大変な方法と思われます。
  - [module proxy](https://go.dev/ref/mod#module-proxy)は`GOPROXY` protocolを実装したHTTPサーバーです。
  - 見たところ実装自体は難しくなさそうですが、これのために1つ運用するサーバーを増やすというのもどうなのかなあという気持ちもあります。
  - [github.com/goproxy/goproxy](https://github.com/goproxy/goproxy)がgoでgo module proxyを実装しているのでこれを用いるか、
  - kubernetes clusterを動作させているなら[ArtifactHUBから探して](https://artifacthub.io/packages/search?ts_query_web=go+module+proxy&sort=relevance&page=1)Helmで導入するなどするとよいかもしれないです。
  - このmodule proxy server自体にauthが必要ですので[GOAUTH]を各clientに設定してもらうか、イントラからしかアクセスできないようにするかする必要があります。これはこれで大変ですね。

module proxyを運用したい別の理由があるなら話は違いますが、`VCS` suffixをつけてしまうのが一番楽です。

### とりあえずmainを作ってビルドして実行してみる

先ほどクローンしたディレクトリで、以下のようにファイルを作成し、`Go module`を初期化しましょう

以下の手順ではgit repositoryの直下じゃなくてサブディレクトリにmoduleを作っています。これはこの記事向けのスニペットをまとめて同じrepositoryに置きたい筆者の都合です。
なので読者はパスはいい感じに読み替えて都合のいいパスで実行してください。

```
mkdir starting-projects
cd starting-projects
go mod init github.com/ngicks/go-example-basics-revisited/starting-projects
```

`go mod init`実行後に以下のファイルが作成されたと思います

```mod: go.mod
module github.com/ngicks/go-example-basics-revisited/starting-projects

go 1.24.2
```

このファイルが、`go module`の

- 名前(というかパス)
- version
- toolchain
- 依存するほかの`go module`

などを記録するファイルとなります。
`pyproject.toml`、`package.json`、`deno.json`などと近しいものです。

このファイルは`go get`や`go mod tidy`などのコマンドに編集してもらうことになるので、手で編集することは少ないです。

`go version`のfix release(1.24.2の末尾の.2)が0以外だと少々具合が悪いので編集します。

```
go mod edit -go=1.24.0
```

すると`go.mod`の内容は以下のように変更されます。

```diff: go.mod
module github.com/ngicks/go-example-basics-revisited/starting-projects

-go 1.24.2
+go 1.24.0
```

`Go`のmajor release([Go 1.23]や[Go 1.24]のような)はAPI追加、構文の追加、たまにエッジケースの挙動が破壊的に変更されますから、これは重要な観点です。他方、fix releaseはセキュリティーにかかわるfix以外では挙動の変更は起こらないことになっています。
別に動作するにもかかわらずfix releaseが古い`go module`から`go get`できなくなるため、基本的にはfix releaseは`.0`を指定しておくほうが良いのではないかと思います。
std libraryはビルドするときのtoolchainのものが使われるため、ビルドする側の設定次第で1.24.2でもビルドできますので`go.mod`では常に`.0`を指定していても問題ないはずです。

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
`func init() {}`が定義されているとそちらが先に実行されるので`main`が必ず最初に実行されるというわけではありません。

以下のコマンドでビルドすることができます。

```
go build ./cmd/example
```

`linux/amd64`で実行すると`./example`が出力されます。
`windows`だと`./example.exe`になります。

もしくは以下のコマンドで実行します

```
$ go run ./cmd/example
Hello world
```

`go run`はOS依存のtmpディレクトリにビルドして実行するショートハンド的コマンドで、毎回ビルドしてしまうので複数回実行したい場合は`go build`したほうが良いことが多いでしょう。

ちなみに、以下ではダメです。

```
$ go build cmd/example
package cmd/example is not in std (~/.local/go/src/cmd/example)
```

```
$ go help packages
...
An import path that is a rooted path or that begins with
a . or .. element is interpreted as a file system path and
denotes the package in that directory.

Otherwise, the import path P denotes the package found in
the directory DIR/src/P for some DIR listed in the GOPATH
environment variable (For more details see: 'go help gopath').
...
```

とあるように、`/`や`C:\`、`.`、`..`から始まらないパスは`$(go env GOPATH)`以下にあるかのように解決されてしまうからです。
(前述した表現では`std`であるかのように解決される、と述べていますが、これは`GOPATH`をいじることはもうないから、すなわち`GOPATH`=`std`と実質なっているからです。)

以下の場合はエラーなく実行できますが、packageが複数のファイルを含む場合うまくビルドできないことを筆者は確認しています。

```
$ go run cmd/example/main.go
Hello world
```

つまり、`cmd/example`以下にファイルを足して`cmd/example/main.go`がそれを参照するようにすると

```
cat << EOF > cmd/example/other.go
package main

var Foo = "foo"
EOF
```

```diff go: cmd/example/main.go
package main

import "fmt"

func main() {
-  fmt.Println("Hello world")
+  fmt.Println("Hello world", Foo)
}
```

以下のような感じでエラーを吐きます。

```
$ go run ./cmd/example/main.go
command-line-arguments
cmd/example/main.go:6:29: undefined: Foo
```

実はファイルリストだったら実行できるんですが

```
$ go run ./cmd/example/main.go ./cmd/example/other.go
Hello world foo
```

ファイルが増えると困りますよね？
なのでファイルパスじゃなくてpacakge pathで指定するとよいでしょう。

相対パスでビルドを行う場合は`./`を必ず含めて、directory名で指定するとよいでしょう。

### packageを分ける

前述のとおり、moduleは複数のpackageに分割されます。
`Go module`では1 directory(フォルダー) = 1 packageとなります。

- １つのdirectoryは１つのpackageしか持てません。
  - ただし例外としてmain package以外については、package nameに`_test`というsuffixをつけてテスト用の別packageを同一directory内に定義できます(e.g. `pkg`に対して`pkg_test`)。
    - `_test` packageは別のpackageなので、元のpackageとnamespaceを共有しません。そのためexportされたシンボルにしかアクセスできません。
    - 何かの都合でテスト内で循環参照が生じてしまうようなときに`_test`を定義してそれを回避するのが主な使い道になります。
- `Go`の慣習とGo teamのおすすめ的には、packageは関心をもとに分割するのが良いとされています。
  - まずはpackageは分割せず、同一package内にすべて定義し、不都合が生じ始めたら分割したらよい、と言われています。
  - もちろんもとから関心が別れる点が明確であれば先だってpackageを分割しておいても特段問題はないと思います。
- packageとdirecotryの名前は一致しているのが望ましいです。
  - これは、import時にデフォルトではpackageで指定される名前でそのpackageにアクセスできるようになるため、import path(= directoryの名前)とpackageの名前が不一致だとものすごく読みにくくなってしまうからです。
    - `gopls`(`Go`の言語サーバー)が機能していると自動的にimportが追加されたりしますが、そのさいにpackageの名前が指定されるように修正されます。(i.e. `import name "path/to/pkg"`)
  - 例外的にpackageの名前がdirectoryの名前のsuffixである場合は問題なしとされるようです
    - 上記の`gopls`の修正が起こらない。
    - `semanticTokens`の設定を有効にしていると、packageの名前部分だけ色が変わるのでわかるようになります。
- 慣習的にpackageの名前は１語で短いものが良いとされます。
  - 長い名前を与えないといけないということはpackage階層設計がうまくいっていないことの兆候だからということらしいです。
- 複数単語を含む場合でも`_`や`-`でつながず、`some_package`の代わりに`somepackage`を用いるのがよいとされます。
  - 前述通りpackageの名前がそれにアクセスするためのidentifierとなりますが、`_`が含まれるのは変数名の慣習と一致しません。
  - さらに大文字・小文字を区別しないファイルシステムが存在するためすべて小文字が好ましいという都合があります。
  - それらがあわさるとこういった慣習になっているのだと思います。

同じpackage内のファイルはnamespaceを共有しています: つまり別のファイルに同名の関数は定義できないし、別のファイルの関数や変数を利用可能です。

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

  "github.com/ngicks/go-example-basics-revisited/starting-projects/pkg1"
)

func SayDouble() string {
  return fmt.Sprintf("%q%q", pkg1.Foo, pkg1.Foo)
}
```

上記のように、ほかのpackageで定義した内容を利用するには、`import`宣言内で、fully qualifiedなpackageパスを書くことで、インポートします。

`Go module`は、循環インポートを許しません。つまり

```diff go: pkg1/some.go
package pkg1

+import "github.com/ngicks/go-example-basics-revisited/starting-projects/pkg2"

var Foo = "foo"
```

とすると以下のように _import cycle not allowed_ エラーによりビルドできません。

```
$ go build ./pkg1
package github.com/ngicks/go-example-basics-revisited/starting-projects/pkg1
        imports github.com/ngicks/go-example-basics-revisited/starting-projects/pkg2 from some.go
        imports github.com/ngicks/go-example-basics-revisited/starting-projects/pkg1 from other.go: import cycle not allowed
```

### 外部のGo moduleをimportする(go get)

以下のコマンドドキュメントを参考にすると

https://pkg.go.dev/cmd/go

```
go get <<fully-qualified-package-path>>
```

で、`Go module`を取得し、`go.mod`と`go.sum`を編集します。

例えば

```
$ go get github.com/ngicks/go-iterator-helper
go: added github.com/ngicks/go-iterator-helper v0.0.18
```

を実行すると以下のように`go.mod`と`go.sum`にmodule情報が追記されます。

```diff: go.mod
module github.com/ngicks/go-example-basics-revisited/starting-projects

go 1.24.0

+ require github.com/ngicks/go-iterator-helper v0.0.18 // indirect
```

```diff: go.sum
+github.com/ngicks/go-iterator-helper v0.0.18 h1:a9a3ndHDyYSsI9bLTV4LOUA9cg6NpwPyfL20t4HoLVw=
+github.com/ngicks/go-iterator-helper v0.0.18/go.mod h1:g++KxWVGEkOnIhXVvpNNOdn7ON57aOpfu80ccBvPVHI=
```

まだこのmoduleはこのプロジェクトのどこからも使われていないので`// indirect`がつけれています。

`import`で各ソースコードにmoduleを導入して使用できるようになります。
[gopls]の設定で[completeUnimported](https://github.com/golang/tools/blob/gopls/v0.18.1/gopls/internal/settings/settings.go#L617-L619)が有効にされていると、可能な場合`import`に書かれていない内容でも補完がかかります(e.g. `import` clauseがないファイルで`fmt.`と打つと`fmt.Println`などがサジェストされ、選ぶと`import "fmt"`が追記される)ので、見た目に反してこの内容を書くのは面倒ではありません。(初めて導入したモジュールとかは補完されないことがある。)

```diff go: cmd/example/main.go
package main

-import "fmt"
+import (
+  "fmt"
+
+  "github.com/ngicks/go-iterator-helper/hiter"
+  "github.com/ngicks/go-iterator-helper/hiter/mapper"
+  "github.com/ngicks/go-iterator-helper/hiter/mathiter"
+)

func main() {
  fmt.Println("Hello world", Foo)
+  fmt.Println(hiter.Sum(mapper.Sprintf("%x", hiter.Limit(8, mathiter.Rng(256)))))
}
```

```
$ go run ./cmd/example/
Hello world foo
9585b4cf88cdbe27
```

この時点で`go mod tidy`を実行します。

```
go mod tidy
```

```diff: go.mod
module github.com/ngicks/go-example-basics-revisited/starting-projects

go 1.24.0

-require github.com/ngicks/go-iterator-helper v0.0.18 // indirect
+require github.com/ngicks/go-iterator-helper v0.0.18
```

```diff: go.sum
+github.com/google/go-cmp v0.5.9 h1:O2Tfq5qg4qc4AmwVlvv0oLiVAGB7enBSJ2x2DqQFi38=
+github.com/google/go-cmp v0.5.9/go.mod h1:17dUlkBOakJ0+DkrSSNjCkIjxS6bF9zb3elmeNGIjoY=
github.com/ngicks/go-iterator-helper v0.0.18 h1:a9a3ndHDyYSsI9bLTV4LOUA9cg6NpwPyfL20t4HoLVw=
github.com/ngicks/go-iterator-helper v0.0.18/go.mod h1:g++KxWVGEkOnIhXVvpNNOdn7ON57aOpfu80ccBvPVHI=
+gotest.tools/v3 v3.5.1 h1:EENdUnS3pdur5nybKYIh2Vfgc8IUNBjxDPSjtiJcOzU=
+gotest.tools/v3 v3.5.1/go.mod h1:isy3WKz7GK6uNw/sbHzfKBLvlvXwUyV06n6brMxxopU=
```

実際にプロジェクト内で使われるようになったので`// indirect`が外れます。
さらに`go get`時には追加されていなかった依存先の依存先も`go.sum`に記録されています。 

`go mod tidy`を行うと`go.sum`の内容が整理されたり、使われなくなった外部moduleが削除されたりします。`VCS`にプッシュする前には`go mod tidy`を実行しておくほうがよいでしょう。

上記の例では`go get`時にversionを指定していないため、適当な最新バージョンが選ばられるようです。

下記のドキュメントにある通り、`versinon query`によってversionの指定を行うことができます。

https://go.dev/ref/mod#version-queries

大体の場合下記のいずれかの方法で行うことになると思います

```
go get <<fully-qualified-module-path>>@latest
go get <<fully-qualified-module-path>>@v1.2.3
go get <<fully-qualified-module-path>>@<<git-tag>>
go get <<fully-qualified-module-path>>@<<git-commit-hash-prefix>>
```

`git tag`は`v`でprefixされた[Semantic Versioning 2.0](https://semver.org/)形式であれば`go.mod`にそのバージョンで記載されます([参照](https://go.dev/ref/mod#vcs-version))。`sem ver`形式でなくてもよいですが、その場合は[pseudo-version](https://go.dev/ref/mod#glos-pseudo-version)というpre-release形式のversionにエンコードされて記載されます。
この`pseudo-version`を直接指定しても取得できますが、指定したいrevisionの直前の`sem ver`から1つ進んだversionのpre-releaseになる変換方式のようですので、これを直接指定するのは手間です。なのでやらないほうが良いでしょう。

### internal: 外部公開しないpackage

`internal`という名前のdirectoryを作成すると、それ以下で定義されるpackageは、`internal/`と同階層かそれ以下からしかimportできなくなります。

例として以下のように`internal`を追加してみます。

```
.
├── cmd
│   └── example
│       ├── main.go
│       └── other.go
├── go.mod
├── go.sum
├── internal
│   └── i0
│       └── i0.go
├── pkg1
│   ├── internal
│   │   └── i1
│   │       └── i1.go
│   └── some.go
└── pkg2
    ├── internal
    │   └── i2
    │       └── i2.go
    └── other.go
```

内容は何でもいいのでとりあえず以下のようにします。

```go: internal/i0/i0.go
package i0

const Yay0 = "yay0"
```

```go: pkg1/internal/i1/i1.go
package i1

const Yay1 = "yay1"
```

```go: pkg2/internal/i2/i2.go
package i2

const Yay2 = "yay2"
```

前述のとおり、`internal/`と同階層か、それより下からは参照できます。

```diff go: pkg1/some.go
package pkg1

+import (
+  "github.com/ngicks/go-example-basics-revisited/starting-projects/internal/i0"
+  "github.com/ngicks/go-example-basics-revisited/starting-projects/pkg1/internal/i1"
+)

var Foo = "foo"
+
+func SayYay0() string {
+  return i0.Yay0
+}
+
+func SayYay1() string {
+  return i1.Yay1
+}
```

```diff go: pkg2/other.go
package pkg2

import (
  "fmt"

+  "github.com/ngicks/go-example-basics-revisited/starting-projects/internal/i0"
  "github.com/ngicks/go-example-basics-revisited/starting-projects/pkg1"
+  "github.com/ngicks/go-example-basics-revisited/starting-projects/pkg2/internal/i2"
)

func SayDouble() string {
  return fmt.Sprintf("%q%q", pkg1.Foo, pkg1.Foo)
}

+func SayYay0() string {
+  return i0.Yay0
+}
+
+func SayYay2() string {
+  return i2.Yay2
+}
```

もちろんこれらはビルドが成功します。

```
$ go build ./pkg1
$ go build ./pkg2
```

試しに下ではない階層からimportしてみます

```diff go: pkg1/other.go
package pkg1

import (
  "github.com/ngicks/go-example-basics-revisited/starting-projects/internal/i0"
  "github.com/ngicks/go-example-basics-revisited/starting-projects/pkg1/internal/i1"
+  "github.com/ngicks/go-example-basics-revisited/starting-projects/pkg2/internal/i2"
)

var Foo = "foo"

func SayYay0() string {
  return i0.Yay0
}

func SayYay1() string {
  return i1.Yay1
}

+func SayYay2() string {
+  return i2.Yay2
+}
```

これは述べた通りエラーとなります。

```
$ go build ./pkg1
package github.com/ngicks/go-example-basics-revisited/starting-projects/pkg1
        pkg1/some.go:6:2: use of internal package github.com/ngicks/go-example-basics-revisited/starting-projects/pkg2/internal/i2 not allowed
```

### package構成

https://go.dev/doc/modules/layout

- 実行ファイルを作成するのが主眼となるmoduleはトップディレクトリがmainになることが多い
- ライブラリとしてインポートすることもできるが、実行ファイルも提供する場合は以下のような構成になることが多い

実際にはプロジェクトの規模や意図などによってどのようにコードをオーガナイズするとよりよくなるかは変わるので、
個別の議論は避け、ほかの記事に譲るものとします。

```
.
├── cmd
│ ├── command1
│ │   ├── main.go
│ │   └── other_files.go
│ └── command2
│     ├── main.go
│     └── other_files.go
├── package_dir (名前はふさわしいものにする)
│   └── package.go
├── go.mod
├── go.sum
└── lib.go(名前はmoduleにふさわしい何かにする)
```

## formatterの設定(goplsに任せるので特に設定はない)

[gopls]の設定で`gofmt`, `goimports`, `gofumpt`などでformatがかけられます。多分前述したeditorのセットアップをしたうえで、editorで`Format On Save`を有効にすれば問題なくformatがかかりますので特に設定はありません。

といいつつ、`vim`/`neovim`ではさらに[追加の設定](https://github.com/golang/tools/blob/gopls/v0.18.1/gopls/doc/vim.md#imports-and-formatting)が必要です。どうも試してる限りまだこのautocmdがないとimportの修正が起こらないっぽい？詳しくなくて裏が取れてません。
筆者は`lua_ls`に警告を受けるのが気に入らなかったので[若干修正](https://github.com/ngicks/dotfiles/blob/75b4a0c4db837b5fdc700b9d183ddb5d53598bb8/.config/nvim/after/lsp/gopls.lua#L1-L32)して使っています

## linterの設定

[gopls]の設定で`staticcheck`や、[golang.org/x/tools/gopls/internal/analysis/modernize](https://pkg.go.dev/golang.org/x/tools/gopls/internal/analysis/modernize)などが有効になっているはずです。

それ以外のいろいろなルールを追加したい場合は、[github.com/golangci/golangci-lint](https://github.com/golangci/golangci-lint)がよく用いられると思います。

導入方法は下記で述べられていますが、[vscode],`GoLand`に関してはextensionを入れる以外には特に設定がいらず、`vim`/`neovim`に関しては[golangci-lint-langserver](https://github.com/nametake/golangci-lint-langserver)を用いるように書かれています。
[エディタのセットアップ](#エディタのセットアップ)のところで触れたような感じで設定すればよいです([筆者の設定](https://github.com/ngicks/dotfiles/blob/75b4a0c4db837b5fdc700b9d183ddb5d53598bb8/.config/nvim/after/lsp/golangci_lint_ls.lua)。`nvim-lspconfig`の設定とマージされる前提です。)

https://golangci-lint.run/welcome/integrations/

ルールを自作し、`golangci-lint`から実行させたい場合は

- [github.com/quasilyte/go-ruleguard](https://github.com/quasilyte/go-ruleguard)で作成する
  - 参考: [【Go】コーディング規則を簡単にlinterに落としこむ！go-ruleguardを使ってみる](https://zenn.dev/hrbrain/articles/4365c28245e2d3)
- [golang.org/x/tools/go/analysis](https://pkg.go.dev/golang.org/x/tools/go/analysis)などで自作し、[Module Plugin System](https://golangci-lint.run/plugins/module-plugins)で追加

などします。

どちらも始めたての人がいきなりできるものではないと思う(`Go`のastと型システムに対する習熟がいる。これは他言語の経験では補いづらい)ため、基本は`golangci-lint`にあらかじめ統合されたもののみを使うとよいでしょう。

## Private repositoryから`go get`する

https://go.dev/ref/mod#private-modules

上記の説明より、一般公開されない、つまり特別な認証が必要な`VCS`でソースを管理し、`go get`などでmoduleをインポート/ダウンロードする場合、

- `GOPROXY`もしくは`GOPRIVATE`の設定
- (web access時にauthが必要な場合)`GOAUTH`の設定
- (`GOPROXY`を用いず、サブグループ下でソースを管理する場合)module nameを`.git`でsuffixする
- (`GOPROXY`を用いない場合）git credentialの適切な保存

を行う必要があります。

`GOPROXY`は筆者は試したことがないため、省略します。
`VCS`のcredentialは`git`のみを想定します。

### GOPRIVATE

`GOPRIVATE`の設定は以下で行います。

```
# git repositoryのURIが https://example.com/base_path
# である場合、<<url_wo_protocol>>は`example.com/base_path`になります。
go env -w GOPRIVATE=<<url_wo_protocol>>
```

(環境変数で指定すればよいと書かれていますが、筆者はうまくいかなかったので`go env -w`で書き込んでいます。)

`GONOPROXY`, `GONOSUMDB`(`NO`であることに注意)を設定しない場合、`GOPRIVATE`がデフォルトとして使われます。
`GONOPROXY`に設定されたホストからのmodule取得する(`direct` mode)際には相手`VCS`に合わせたコマンドが使用されます(`git`の場合`git`コマンド -> [modfetch](https://github.com/golang/go/blob/go1.24.2/src/cmd/go/internal/modfetch/codehost/git.go#L233))。そのため、credentialの設定も多くの場合必要になります。

`go env -w`で書き込まれた内容は`go env GOENV`で表示されるファイルに保存されます。

```
$ go env -w GOPRIVATE=example.com
$ cat $(go env GOENV)
GOPRIVATE=example.com
# -uでunset
$ go env -u GOPRIVATE
$ cat $(go env GOENV)

```

[Docker]のbuild contextに渡したい場合などはこのファイルをマウントするとよいでしょう。

### GOAUTH(筆者は試したことがない)

[Go 1.23]かそれ以前では、`go tool`がhttp accessを行う際にはcredentialを`.netrc`から読み込んでいましたが、[Go 1.24]からは[GOAUTH]を設定することで任意の方法を設定できるようになりました(デフォルトは`.netrc`)。

[private VCSかつサブグループを使用する場合.gitなどでmodule nameをsuffixしておく](#private-vcsかつサブグループを使用する場合.gitなどでmodule-nameをsuffixしておく)のところで述べましたが、[Go 1.23]まで`.netrc`以外にcredを渡す方法がなかったため`gitlab`では`?go-get=1`がついている場合credentialなしのHTTP GETを受け付けるようになっていました。そのため設定する必要があるのはこれ以外にもっときつい制限をかけた`VCS`で使用しているか、もしくはprivate go module proxyを用いる場合でしょうか。

筆者は試したことがないため参考までに、ですが、基本的には`git dir`を使用するとよいのではないかと思います。これは`GOAUTH=git /path/to/working/dir`を渡すと、[dirでgit credential fillを呼び出し](https://github.com/golang/go/blob/master/src/cmd/go/internal/auth/gitauth.go#L45)、[Basic Auth](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Authentication#basic_authentication_scheme)としてHTTP Headerにセットするものなので、git credentialの設定がしっかりされていれば追加の設定が不要であるためです。

`Basic Auth`以外の認証方法が必要な場合はカスタムコマンドを渡します。`go help goauth`を参照してください。

(試してみたかったんですが、`github`でprivate repositoryを`direct` modeで取得する際には`GOAUTH`を設定していなくてもよいみたいなので試せませんでした。)

### module nameを`.git`でsuffixする

👉[private VCSかつサブグループを使用する場合.gitなどでmodule nameをsuffixしておく](#private-vcsかつサブグループを使用する場合.gitなどでmodule-nameをsuffixしておく)

### git credentialの適切な保存

`GOPROXY`を用いない場合、`VCS`に対して直接`VCS`のコマンド(`git`なら`git ls-remote`など)が実行されます。
これは`direct` modeとドキュメント上呼ばれています。

この場合、`git`なら`git`のcredentialを適切に保存しておく必要があります。private repositoryを利用している方はすでに設定しているかもしれないのでここは読み飛ばしてもらったほうがいいのかもしれません。

#### Git Credential Manager

`git credential`の適切な保存には筆者は[Git Credential Manager]を利用しています。

- windowsの場合、[Git for windows](https://gitforwindows.org/)に付属してきますので、インストールオプションで一緒に入れます。
- linuxの場合,[Install instructions](https://github.com/git-ecosystem/git-credential-manager/blob/release/docs/install.md)に従いセットアップを行います。

[vscode]の各種`Remote` extensionが`git credential`のヘルパーをつないでホスト環境につなげてくれるような挙動をしますので、
`wsl`などの場合はそちらを利用すれば`wincred`にcredentialの保存が簡単にできます。

#### .netrc(非推奨)

`.netrc`はネットワークの認証情報を**平文で**保存しておくファイルらしく、[linuxのman pageを検索するといくつかのコマンドがそれらを尊重する](https://www.google.com/search?q=netrc&sitesearch=man7.org%2Flinux%2Fman-pages&sa=Search+online+pages#ip=1)のがわかります。
フォーマットは[IBMの「.netrc ファイルの作成」](https://www.ibm.com/docs/ja/aix/7.2?topic=customization-creating-netrc-file)を参考にしてください。
[\${HOME}/.netrcあるいは${NETRC}にあるのが想定される](https://github.com/golang/go/blob/go1.24.2/src/cmd/go/internal/auth/netrc.go#L73-L96)ので適切に配置してください。

`git`を含めた種々のコマンドが`.netrc`を読み込むようですが、`GOAUTH`が設定されていない場合は`go tool`もこれを読み込む挙動となっています。

平文(=暗号化されていない)でcredentialを保存するフォーマットを推奨するべきではないはずなので、非推奨と述べておきます。

#### (おまけ)gpg-agentの設定

上記で[Git Credential Manager]を設定し、`pass`を利用する設定にしている場合などでローカルの`GnuPG`が利用されるようになっており、環境がguiを表示可能な時

- そもそもdesktopのGUI付きlinux
- `wslg`
- `ssh config`で`X11Forwarding yes`にしている、など

にはgpg-agentをGUI付きのものにすると例えば`tmux`スクリプトなんかでgpg-agentが呼び出されたときに気づかずパスワードの入力画面で固まらないため便利です。

```
# 筆者はみための好みでpinentry-qtを選択
sudo apt install pinentry-qt
```

```
$ cat ~/.gnupg/gpg-agent.conf
pinentry-program /usr/bin/pinentry-qt
```

ターミナルで入力を求めてくる形式のものは`lazygit`の画面ステートをぶっ壊したりして大変な目にあうかもしれないです(n敗)。

### git-lfsを導入している場合はすべての環境でgit-lfsを使うように気を付ける

`git-lfs`というよりは導入有無でfetch結果のファイルコンテンツが変わってしまうプラグイン全般なのですが。

上記のような設定で`direct` modeで`Go module`が取得される場合、
`git-lfs`の導入有無で`git`からのfetch後の内容が異なることがあります。
これによってsum照合エラーで`go mod download`が失敗する現象を何度か体験しています。

基本的にはすべての環境(`Dockerfile`なども含む)で`git-lfs`を導入しておくほうがよいでしょう。

`git-lfs`は[Git Large File Storage](https://git-lfs.com/)のことで、`git`で大きなファイルを取り扱うための拡張機能です。
`git-lfs`はhookとfilterを活用してコミット前後でトラック対象のファイルをテキストファイルのポインターに変換し、
トラックされた大きなファイルはremote repositoryではなく大容量ファイル用のサーバーに上げるような挙動になります。(参考: https://github.com/git-lfs/git-lfs, [Git LFS をちょっと詳しく](https://qiita.com/ikmski/items/5cc8b8832336b8d85429))

[`github`](https://docs.github.com/ja/repositories/working-with-files/managing-large-files/about-git-large-file-storage), [`gitlab`](https://docs.gitlab.com/ee/administration/lfs/index.html)双方とも`git-lfs`に対応しています。

開発の経緯的に、想定された用途ははゲームなどで大きなバイナリファイルを一緒に管理することのようです。
それ以外でもテスト用の大きなファイルを管理するときなどにも使うことがあると思います。

## task runner

`Go`には組み込まれたtask runnerはありません。使わないか、外部のtask runnerを使います。

### 使わない

以下のケースではtask runnerを用いなくても十分通用します。

- 複雑なビルド過程やテストマッチャーがない
- `github actions`,`gitlab ci`, [Dockerfile]などでビルドを記述できる
- `//go:generate`で事足りる
- `.sh`と`.bat`を両対応できる程度の量しかスクリプトがいらない
- 管理用コマンドも全部`Go`で実装するつもり

[Generate Go files by processing source](https://pkg.go.dev/cmd/go#hdr-Generate_Go_files_by_processing_source)より、`//go:generate command`というマジックコメントを`.go`ファイルの中に書き込むと、`go generate ./path/to/file_or_package`で`command`を実行できます。主眼は`code generator`を実行することですが実際には任意のコマンドを実行可能です。

依存moduleの管理は`go.mod`でできますし、moduleをvendorする機能もサポートされていますので、task runnerが必要ないケースも多いのではないでしょうか。

### Make

[make](https://www.gnu.org/software/make/)を使っているプロジェクトをいくつか見たことがあります。
筆者はこの方法をとったことがないため何とも言えませんが、

- pros:
  - `linux`ならもとからインストール済みのことが多い
- cons:
  - `make`はtask runnerではないという批判
  - syntaxの覚え方が苦しんで覚える以外の方法があるのかわからない
  - windowsで動かそうとするとハマる

という感じでしょうか(筆者の完全主観ですが)

チームがすでに`make`に慣れている場合はよい思います。

### github.com/go-task/task

[github.com/go-task/task]を用いるという方法もあります。

https://taskfile.dev/

> Task is a task runner / build tool that aims to be simpler and easier to use than, for example, GNU Make.

とある通り、`make`などの代替を目指すものです。

- pros:
  - `Go`で開発されているため、toolchainが入っていれば簡単にインストール可能
  - cross-platform
  - yamlで書ける
- cons:
  - 管理すべきツールが増える

```
go install github.com/go-task/task/v3/cmd/task@latest
```

`--init`で初期化を行います。

```
$ task --init
Taskfile created: Taskfile.yml
```

デフォルトで以下が作成されます。

```yaml: Taskfile.yml
# https://taskfile.dev

version: '3'

vars:
  GREETING: Hello, World!

tasks:
  default:
    cmds:
      - echo "{{.GREETING}}"
    silent: true
```

[Templating Reference](https://taskfile.dev/reference/templating/)にあるように[text/template]の構文でtemplateを書けるようですね

```diff yaml: Taskfile.yml
# https://taskfile.dev

version: '3'

vars:
  GREETING: Hello, World!

tasks:
  default:
    cmds:
-      - echo "{{.GREETING}}"
+      - echo "{{index .GREETING 0}}"
    silent: true
```

```
$ task default
72
```

72は`H`のascii codeです。

[Usage#task-dependencies](https://taskfile.dev/usage/#task-dependencies)より、task間に依存関係を記述可能です

```diff yaml: Taskfile.yml
# https://taskfile.dev

version: '3'

vars:
  GREETING: Hello, World!

tasks:
  default:
    cmds:
      - echo "{{index .GREETING 0}}"
    silent: true
+  quack:
+    cmds:
+      - echo quack
+  run:
+    deps: [quack]
+    cmds:
+      - go run ./cmd/example
```

```
$ task run
task: [quack] echo quack
quack
task: [run] go run ./cmd/example
Hello world foo
ede693e1e85a2f70
```

最近のpowershell(というかwindowsに？)にはunix風コマンドがいくつかあります(`tar`とか`curl`とか)。echoは存在するみたいなのでこれはwindowsでも動作するようです。

### deno task

[dax]: https://github.com/dsherret/dax

[deno]のtask runnerを用いる方法もあると思います

https://docs.deno.com/runtime/reference/cli/task/

- pros:
  - cross-platform
  - jsonで書ける
  - [dax]を用いた容易なスクリプト開発
- cons:
  - 別言語の知識が必要
  - 管理すべきツールが増える

[deno]は[Rust]の[tokio](https://github.com/tokio-rs/tokio)をバックエンドに、`javascript` engineの[V8](https://v8.dev/)で動作する`javascript`/`typescript`ランタイムです。

[dax]を用いるとshellscriptのようなノリで`typescript`がかけるためいい感じです。
なんとshellコマンド間や`javascript object`にpipeが行えるのです。shellscriptで書くには億劫な高度な演算を`typescript`でかいたり、コマンドの結果を`WebStream`に受けていじくったりできるので便利だと思います。

ただ半面導入するツールが増えのが難点です。`Go`で開発されているわけでもないためぱっと導入できるわけでもないですし、書き込まれるキャッシュ領域も増えるので、必ずしもこれが最適な選択というわけでもありません。
チームがすでに`typescript`に慣れており、何かの事情ですでに[deno]を導入している場合はよいかもしれません。

`task`であげた例と似たようなものは以下のようになります。

```json: dneo.json
{
  "tasks": {
    "quack": "deno eval 'console.log(\"quack\")'",
    "run": {
      "command": "go run ./cmd/example",
      "dependencies": ["quack"]
    }
  },
  "imports": {
    "#/": "./script/"
  }
}
```

```
$ deno task run
Task quack deno eval 'console.log("quack")'
quack
Task run go run ./cmd/example
Hello world foo
ed35803c9ff5b841
```

[dax]を用いるとshellのように`typescript`がかけます。
少し極端な例として`sha256sum`をとるのをshellだけでやるような形と、`WebStream`を混在させたバージョンの二つを挙げてスタイルの自由さの例とします。
[builtInCommands](https://github.com/dsherret/dax/blob/0.43.0/src/command.ts#L90-L107)は[dax]が抽象化しているのでwindowsでも動きます。`sha256sum`や`awk`はないので実際には`WebStream`版のようなことをすることになるでしょう。

```ts: script/dax_example.ts
import { crypto } from "jsr:@std/crypto";
import { encodeHex } from "jsr:@std/encoding/hex";

import $ from "@david/dax";

const sum1 = await $`cat ./go.mod | sha256sum | awk '{print $1}'`.text();

const pipe = new TransformStream<Uint8Array, Uint8Array>();
const sumP = crypto.subtle.digest("SHA-256", pipe.readable);
await $`cat ./go.mod > ${pipe.writable}`;
const sum2 = encodeHex(await sumP);

console.log(sum1);
console.log(sum2);
console.log("same? =", sum1 === sum2);
```

```
$ deno run -A ./script/dax_example.ts
a141062eb619fb89a183d62b8896d192170b1dd6fc479611f6c5a427038447f0
a141062eb619fb89a183d62b8896d192170b1dd6fc479611f6c5a427038447f0
same? = true
```

[dax]・・・すばらしい・・・

## おわりに

private repositoryから`go get`するのは特に躓いたのでまとめておきました。

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
[Dockerfile]: https://docs.docker.com/build/concepts/dockerfile/
[Elasticsearch]: https://www.elastic.co/docs/solutions/search

<!-- Go versions -->

[Go]: https://go.dev/
[Go 1.11]: https://go.dev/doc/go1.11
[Go 1.14]: https://go.dev/doc/go1.14
[Go 1.18]: https://go.dev/doc/go1.18
[Go 1.23]: https://go.dev/doc/go1.23
[Go 1.24]: https://go.dev/doc/go1.24

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

[panic]: https://pkg.go.dev/builtin@go1.24.2#panic
[errors.New]: https://pkg.go.dev/errors@go1.24.2#New
[errors.Is]: https://pkg.go.dev/errors@go1.24.2#Is
[errors.As]: https://pkg.go.dev/errors@go1.24.2#As
[errors.Join]: https://pkg.go.dev/errors@go1.24.2#Join
[fmt.Errorf]: https://pkg.go.dev/fmt@go1.24.2#Errorf
[fs.ErrNotExist]: https://pkg.go.dev/io/fs@go1.24.2#ErrNotExist
[http.Server]: https://pkg.go.dev/net/http@go1.24.2#Server
[*http.Server]: https://pkg.go.dev/net/http@go1.24.2#Server
[io.EOF]: https://pkg.go.dev/io@go1.24.2#EOF
[io.Reader]: https://pkg.go.dev/io@go1.24.2#Reader
[io.Writer]: https://pkg.go.dev/io@go1.24.2#Writer
[syscall.Errno]: https://pkg.go.dev/syscall@go1.24.2#Errno
[text/template]: https://pkg.go.dev/text/template@go1.24.2
