---
title: "Goのプラクティスまとめ: プロジェクトを始める"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## Goのプラクティスまとめ: プロジェクトを始める

筆者が`Go`を使い始めた時に分からなくて困ったこととか最初から知りたかったようなことを色々まとめる一連の記事です。

[以前書いた記事](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part1)のrevisited版です。話の粒度を細かくしてあとから記事を差し込みやすくします。

他の記事へのリンク集

- (まだ)~~[今はこうやる集](https://zenn.dev/ngicks/articles/go-basics-revisited-updated-practices)~~
- プロジェクトを始める: ここ
- (まだ)~~[dockerによるビルド](https://zenn.dev/ngicks/articles/go-basics-revisited-bulding-with-docker)~~
- [error handling](go-basics-revisited-error-handling.md)
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

そこで、この記事では[Go]そのものの紹介、SDKのインストール、エディターのセットアップ、プロジェクトの開始方法(=モジュールの作成方法)、さらにcooporate proxy下でgo moduleをインポートできるようにする方法などについてまとめます。

## 前提知識

- ほか言語での開発経験
  - [python]や[Node.js]などを引き合いに出すことがあります。
  - ある程度ソフトウェア開発における常識感を読者が持っているのを前提とした説明をします。
    - 誰でもわかるように書くと論文みたいになって長くなる傾向がありますし、裏取りの手間が大きくなりすぎるためです。
    - 「常識感」についても共有されていないと曖昧性が増すため筆者の知識のバックグラウンドも簡単に書きます。

## 環境

```
$ go version
go version go1.24.2 linux/amd64
```

`Go`は後方互換性をかなり気にするためサンプルコードや述べられていることは以降のバージョンでも（何ならこれよりの前のバージョンでも）基本的に一緒ですが、細かいところで改善が入ったりするのでそれを踏まえて読んでください。

## snipetのconvention

- shellコマンドの羅列の場合コピペしやすさを優先して特になにもprefixをつけずにコマンドを羅列します。
- ただし、コマンド実行結果をsnipet内の併記したい場合は、コマンドは`$ `でprefixされます。
- `# `から始まる行はコメントです。

## 筆者のバックグラウンド

学生時代に

- [Verilog](https://en.wikipedia.org/wiki/Verilog)(HDL)でAltera製の[FPGA](https://en.wikipedia.org/wiki/Field-programmable_gate_array)の回路を記述していた
- [C++]\(Visual Studio Community 2019だったと思う\)を使ってセンサーから値を読み込んで計算を行うプログラムを作っていた
  - 四元数で回転を計算するのに[Eigen](https://eigen.tuxfamily.org/index.php?title=Main_Page)を使っていました。

`FPGA`を使ったプロジェクトはとん挫したので遊びみたいなものです。
機械や制御を専門とする学徒であったためこの時点ではソフトウェアに詳しくはありませんでした。とくにC++に関しては研究室に置いてあった古い本を読みながらやったので当時からしても古い書き方をしていたと思います。

社会人になってから

- [Node.js]\([TypeScript]\)
- ちょっとだけ[python]
- [Rust]\: C/C++で書かれたライブラリのbindingを書いて使ってた程度であんまりメインで使ってるわけではないです。
- [Go]

を扱っています。

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

- C系の文法
- 強い静的型付け
  - 暗黙的な型変換は起きません。筆者の知る限り最近の言語はそれを行わないので意外でもないでしょうか。
- GC(Garbage Collector)があり、手動でのメモリ確保・解放はほぼやることはありません。
- 言語仕様が簡潔で、言語としてサポートされた構文は多くありません。
  - [spec#Keywords](https://go.dev/ref/spec#Keywords)より、keywordは25個とほかの言語と比べて少ないです。
- 任意の型をベースとする型を定義できます。
  - 型にはmethod(型に関連した関数)を定義できます。
- 特定のmethod setを実装する型なら何でも代入できる`interface`をもち、これによってdynamic dispatchを実現します。
  - 例えば`type IOPort { Read(p []byte) (int, error); Write(b []byte) (int, error) }`を引数の型として指定するとすることで、`Read`と`Write`を実装するあらゆる型を代入可能な関数を定義できます。
  - `interface{}`(なんのmethodも指定しない`interface`)にはあらゆる型の変数を代入可能です。上記の*feels like a dynamically typed, interpreted language*の部分はこのこともさしているのだと思います。
- closure(無名の関数)を定義できます。
  - closureはスコープ内の変数をキャプチャできます。
  - 関数、method、closureはいずれも区別なく引数として渡すことができます。
- `goroutine`という言語に組み込まれたconcurrency mechanismが存在します。
  - `goroutine`は[spec#Go_statement](https://go.dev/ref/spec#Go_statements)で述べられている通り*an independent concurrent thread of control*です。いわゆる[Green Thread](https://en.wikipedia.org/wiki/Green_thread)です。
  - メモリ、スケジューリング双方で軽量であるため、OS threadと違い大量に生成するのが普通です。
    - `goroutine`ごとにstackを持っていますが、2～6KiB(プラットフォームによる)から始まって必要に応じて成長します。
  - `go`キーワードを前につけて関数やmethodを実行するだけです。とっても簡単。
  - 言語に組み込まれているため、エコシステム内でどのランタイムを使うかなどの分断が起きません。
  - async/awaitのような構文を持ち込まずに非同期性を実現します。
    - あとから長いIO処理入れたくなった、となったとしてもsignatureの変更が起こらないという良さがあります。
- `goroutine`間で通信するためのprimitiveとして組み込まれた`chan`(channel)型が存在します。
- 組み込み型として`slice`(動的にサイズが変更できるarray;ほかの言語のvectorに近いもの)、`map`(hash map),`chan`(channel)があってこれらはfor-range構文が対応していたりと特別扱いされます。
- コンパイルが*遅くならない*ように気が遣われています。
- モジュールシステムとパッケージマネージャが組み込まれています。
  - モジュールの公開には特別な処理は必要ありません。`github`のようなVCS(Version Control System)で公開し、go moduleの名前を`https://`抜きのURLにすればそれで公開できます。
- pointerは存在しますが、pointer arithmeticは(`unsafe`を使わない限り)できません
- 非常に簡単に別のCPUアーキテクチャ、OS向けのビルドを行うことができます。
  - ただし`CGO`(C-binding)を使わない場合
- 非常に簡単にstaticにビルドできます。
  - ただし`CGO`(C-binding)を使わない場合

### A Tour of Go

https://go.dev/tour/welcome/1

`Go`の基本的な文法、機能などのトピックはここに書いてあります。
インタラクティブなコードスニペットと簡単なエクササイズがあり、これさえこなせばとりあえず開発は始められます。
**以後の文章はA Tour of Goをすべてこなしたことを前提とします。**

慣れてない頃に、syntax highlightのかからないwebページでコードを書くのはきついと思うのでローカルのエディターにコピーして実行したほうが良いとは思います。
大体３~５時間ぐらいで全部終わると思います。

### Go by Example

https://gobyexample.com/

コード例とともに解説がされます。
項目数が多く、知らないstdモジュールを使う部分をきちんと理解しようとすると時間がかかると思います。
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
  - `neovim`(v0.11.0)では適当なpackage managerで[neovim/nvim-lspconfig](https://github.com/neovim/nvim-lspconfig)をロードし、`~/.config/nvim/after/lsp/gopls.lua`を作成して適当に設定を上書きするとよいです。
    - `gopls`(Goの言語サーバー)などのインストールは[williamboman/mason](https://github.com/williamboman/mason.nvim)を用いると楽です。もちろん自らの道を行ってもよいでしょう(全部`lazy.nvim`で管理するなど)。
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
  - 例外的にmain以外のpackageは`_test` suffixをつけた(e.g. `pkg`に対して`pkg_test`)テスト用パッケージを定義することができる
- packageはname spaceを共有します。
  - 複数ファイルにわたって同じ変数、関数にアクセスできます
  - ほかの言語、例えば[Rust]では1つのファイルが1つのmoduleであり、ファイル内でもmoduleを定義することができますが、逆に`Go`ではこのように任意に分割する方法はありません。
- main packageが実行ファイルとしてビルドできる
  - main packageのmain関数がエントリーポイントになる
  - main packageは任意のパスに任意の個数用意できる。
- `go build ./path/to/main/package/`でディレクトリ指定でビルドする
  - 必ず`./`から始まるパスを指定する。ないとstd libraryを指定していると`go tool`に思われる。
- main以外のpackageはfully-qualified pathで(`<<module-name>>/path/to/directory`)importできる。
  - 相対パスを用いることもできるが、ほぼされることがないためしないほうが良いのではないかと思う。
- 外部のモジュールは`go get`で取得することができる。

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

`<<module-name>>`は基本的に上記`<<uri>>`からプロトコルスキームを抜いたものにするとよいです。
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

:::details local onlyのモジュール

ローカルオンリーなつもりならおおむね`<<module-name>>`はなんでもよいはずです。

```
go mod init whatever-whatever
```

筆者が知る限りほかのモジュールから`go get`できなくなる以外の違いはないです。

ただし、[How to write code](https://go.dev/doc/code)でも勧められる通り、なるだけ公開される前提でモジュールを作っておいたほうがよいでしょう。

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
  - 返ってくる内容は`go tool`の期待に反してmodule root pathではなく、渡されたパスをオウム返しするだけです。
  - おそらくセキュリティーのためです(後述)。そのためおそらくどのVCSでもこうなっているんじゃないでしょうか

というのが理由です。

[Go 1.24]まで、`.netrc`を用いる以外に`go tool`にcredentialを渡す方法がなく、そのため`?go-get=1`は認証情報なしでリクエストされることがほとんどだったと思われます。
少なくとも筆者の環境では`.netrc`を作成していませんがprivate repositoryから`go get`が成功していました。ということは認証なしで`?go-get=1`付きのリクエストはとりあえず成功するようにできていたんだと思います。セキュリティー的な懸念から"正しい"内容を認証していない状態で返すわけにはいきませんので、こういう挙動になっていたのでしょう。
`direct` modeで`git`コマンドを直接用いる際には`git`コマンドのcredentialがそのまま利用されますから、ここは広く用いられる方法でcredentialを渡すことができます。
後方互換性のことを考えるとこの挙動が変わることはまずない気がします。

https://gitlab.com/gitlab-org/api?go-get=1 はPublicですが、これも与えられたパスをオウム返しする挙動であるのでPublicであってもうまく動かないと思われます。このURLが指し示す先はサブグループなので500番台とか404とかを返すのが正しい挙動に思えますね。

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
  - このmodule proxy server自体にauthが必要ですので[GOAUTH](https://pkg.go.dev/cmd/go#hdr-GOAUTH_environment_variable)を各clientに設定してもらうか、イントラからしかアクセスできないようにするかする必要があります。これはこれで大変ですね。

module proxyを運用したい別の理由があるなら話は違いますが、`VCS` suffixをつけてしまうのが一番楽です。

### とりあえずmainを作ってビルドして実行してみる

先ほどクローンしたディレクトリで、以下のようにファイルを作成し、`Go module`を初期化しましょう

以下の手順ではgit repositoryの直下じゃなくてサブディレクトリにモジュールを作っています。これはこの記事向けのスニペットをまとめて同じrepositoryに置きたい筆者の都合です。
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

```mod: go.mod
module github.com/ngicks/go-example-basics-revisited/starting-projects

go 1.24.0
```

`Go`のmajor release([Go 1.23]や[Go 1.24]のような)はAPI追加やたまにエッジケースの挙動が破壊的に変更されますから、これは重要な観点ですが、fix releaseはセキュリティーにかかわるfix以外では挙動の変更は起こらないことになっています。
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
package cmd/example is not in std (/usr/local/go/src/cmd/example)
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

以下の場合はエラーなく実行できますが、パッケージが複数のファイルを含む場合うまくビルドできないことを筆者は確認しています。

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

```diff: cmd/example/main.go
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

相対パスでビルドを行う場合は`./`を必ず含めて、パッケージ名で指定するとよいでしょう。

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

  "github.com/ngicks/go-example-basics-revisited/starting-projects/pkg1"
)

func SayDouble() string {
  return fmt.Sprintf("%q%q", pkg1.Foo, pkg1.Foo)
}
```

上記のように、ほかのパッケージで定義した内容を利用するには、`import`宣言内で、fully qualifiedなパッケージパスを書くことで、インポートします。

`Go module`は、循環インポートを許しません。つまり

```diff: pkg1/some.go
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
go get <<fully-qualified-module-path>>
```

で、`Go module`を取得し、`go.mod`と`go.sum`を編集します。

例えば

```
$ go get github.com/ngicks/go-iterator-helper
go: added github.com/ngicks/go-iterator-helper v0.0.18
```

を実行すると以下のように`go.mod`と`go.sunm`にmodule情報が追記されます。

```diff: go.mod
module github.com/ngicks/go-example-basics-revisited/starting-projects

go 1.24.0

+ require github.com/ngicks/go-iterator-helper v0.0.18 // indirect
```

```diff: go.sum
+github.com/ngicks/go-iterator-helper v0.0.18 h1:a9a3ndHDyYSsI9bLTV4LOUA9cg6NpwPyfL20t4HoLVw=
+github.com/ngicks/go-iterator-helper v0.0.18/go.mod h1:g++KxWVGEkOnIhXVvpNNOdn7ON57aOpfu80ccBvPVHI=
```

`import`で各ソースコードにモジュールを導入して使用できるようになります。

```diff: cmd/example/main.go
package main

-import "fmt"
+import (
+	"fmt"
+
+	"github.com/ngicks/go-iterator-helper/hiter"
+	"github.com/ngicks/go-iterator-helper/hiter/mapper"
+	"github.com/ngicks/go-iterator-helper/hiter/mathiter"
+)

func main() {
	fmt.Println("Hello world", Foo)
+	fmt.Println(hiter.Sum(mapper.Sprintf("%x", hiter.Limit(8, mathiter.Rng(256)))))
}
```

```
$ go run ./cmd/example/
Hello world foo
9585b4cf88cdbe27
```

上記の例では`go get`時にversionを指定していないため、すでに1度`go get`済みでキャッシュがある場合はその中の最新、そうでなければ`proxy.golang.org`によって既知の中の最新がダウンロードされるようです。
明確にversionを指定したい場合は、package pathの末尾に`@version`で指定します。
**gitの場合は`v`-prefixのついた[sem ver](https://semver.org/)形式のgit tagがversionとなります。**

下記のいずれかの方法で指定できます。
末尾の２つは全く同じ効果をもたらすので、`git-commit-hash`で指定するほうが簡単だと思います。

```
go get <<fully-qualified-module-path>>@latest
go get <<fully-qualified-module-path>>@v1.2.3
go get <<fully-qualified-module-path>>@<<git-commit-hash>>
# v0.0.0-<<commit-date-time>>-<<commit-hash>>
go get <<fully-qualified-module-path>>@v0.0.0-20230723110635-fd0b45653fa9
```

タグのついていない特定のcommit hashを指定可能です。
`git-commit-hash`によるバージョン指定は`github.com`とセルフホストの`gitlab`では動作しました。
しかしどこにドキュメントされているのかよくわかりませんので100%確証はないです。

### package構成

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

## おわりに

<!-- other languages referenced -->

[TypeScript]: https://www.typescriptlang.org/
[python]: https://www.python.org/
[C++]: https://en.wikipedia.org/wiki/C%2B%2B
[Rust]: https://www.rust-lang.org
[The Rust Programming Language 日本語]: https://doc.rust-jp.rs/book-ja/
[Lua]: https://www.lua.org/

<!-- other SDKs referenced -->

[Node.js]: https://nodejs.org/en
[deno]: https://deno.com/

<!-- editors -->

[Visual Studio Code]: https://code.visualstudio.com/
[vscode]: https://code.visualstudio.com/
[neovim]: https://neovim.io/

<!-- tools -->

[git]: https://git-scm.com/

<!-- Go versions -->

[Go]: https://go.dev/
[Go 1.11]: https://go.dev/doc/go1.11
[Go 1.23]: https://go.dev/doc/go1.23
[Go 1.24]: https://go.dev/doc/go1.24

<!-- Go doc links -->

[A Tour of Go]: https://go.dev/tour/welcome/

<!-- Go tools -->

[gopls]: https://github.com/golang/tools/tree/master/gopls

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

```

```
