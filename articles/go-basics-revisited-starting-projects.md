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

`Go`は後方互換性をかなり気にするためサンプルコードや述べられていることは以降のバージョンでも（何ならこれよりの前のバージョンでも）基本敵に一緒ですが、細かいところで改善が入ったりするのでそれを踏まえて読んでください。

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
- [Rust]
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
  - [spec#Keywords](https://go.dev/ref/spec#Keywords)より、keywordは24個とほかの言語と比べて少ないです。
- 任意の型をベースとする型を定義できます。
  - 型にはmethod(型に関連した関数)を定義できます。
    さらに、特定のmethod setを実装する型なら何でも代入できる`interface`をもち、これによってdynamic dispatchを実現します。
  - 例えば`type IOPort { Read(p []byte) (int, error); Write(b []byte) (int, error) }`を引数の型として指定するとすることで、`Read`と`Write`を実装するあらゆる型を代入可能な関数を定義できます。
  - `interface{}`(なんのmethodも指定しない`interface`)にはあらゆる型の変数を代入可能です。上記の*feels like a dynamically typed*の部分はこのこともさしているのだと思います。
- closure(無名の関数)を定義できます。
  - closureはスコープ内の変数をキャプチャできます。
  - 関数、method、closureはいずれも区別なく引数として渡すことができます。
- `goroutine`という言語に組み込まれたconcurrency mechanismが存在します。
  - `goroutine`は[spec#Go_statement](https://go.dev/ref/spec#Go_statements)で述べられている通り*an independent concurrent thread of control*です。いわゆる[Green Thread](https://en.wikipedia.org/wiki/Green_thread)です。
  - メモリ、スケジューリング双方で軽量であるため、OS threadと違い大量に生成するのが普通です。
    - `goroutine`ごとにstackを持っていますが、(linux/amd64では)`2KiB`から始まって必要に応じて成長します。
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

### The Go Programming Language Specification

https://go.dev/ref/spec

言語仕様ですが割と短めなのでそのうち読んでおいたほうがよいでしょう。

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
  - `Go module`は[Go 1.11]で導入されましたが、`Go module`を用いず開発することはほぼあり得ません。
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
dest=$HOME/gitrepo/$(echo $uri | sed 's/^https\:\/\///' | sed 's/^git@//')
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

例えば、`VCS`の`URI`が`https://github.com/ngicks/go-example-basics-revisited`である場合、

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

#### セルフホストのVCSかつサブグループを使用する場合`.git`などでmodule名をsuffixしておく

ただし、セルフホストの`VCS`(=URIが`go tool`に対して既知でない)でサブグループを作成し、サブグループの中でソースを管理する場合、
`<<module-name>>`はvcsのsuffixを加えておかないと`go get`時に失敗するかもしれません。

つまり上記と同じ例で行くと

```
# 架空のURLを扱うのでexample.comに変えてあります！
go mod init example.com/ngicks/subgroup/go-example-basics-revisited.git
```

とする必要があるということです。

なぜかというと

- private repositoryを参照できるgo module proxyサーバーが存在しないとき、`go tool`は`git ls-remote`などのvcsコマンドを直接実行します。
  - これは`direct` modeと呼ばれます。
- `go tool`にとって既知のホストではなく、vcs suffixがない時、`<<host>>/<<user>>/<<repository>>`がcloneに使うuriだと思われます。
  - 既知のホストとはリンク先にリストされるものです: https://github.com/golang/go/blob/go1.24.2/src/cmd/go/internal/vcs/vcs.go#L1523-L1585
  - `go tool`に渡されるのがmodule nameだとは限らないし、module nameがvcs repositoryのroot pathとも限りません
    - sub packageのパスが渡されることがあります。
      - `go install`など: module rootがmain packageでないことは多々あります。
    - 1つのvcs repositoryに対して複数のgo moduleを持つことができます。
  - サブグループへの考慮がおそらく基本的にありません。
  - `// General syntax for ...`のところからわかる通り、vcs suffix(e.g. `.git`, `.hg`,`.svn`)がついている場合はそこにマッチします。
  - ここだけ見るとuriに`.git`をつければいいだけで、module nameには不要そうに見えますが、名前不一致エラーで落ちますのでmodule nameにsuffixが必要です。
- `go tool`は、`?go-get=1` query parameterを加えたuriにアクセスしてメタデータを得ます。
  - https://go.dev/ref/mod#vcs
- `gitlab`はgroupに対して`?go-get=1`をつけたHTTP GETをしてもエラーなくresponseが返ってきます。
  - アクセスしてみてください: https://gitlab.com/gitlab-org/api?go-get=1
  - これはサブグループであってgit repositoryではなく、ましてやgo moduleではないですがエラーなく`<html><head><meta name="go-import" content="gitlab.com/gitlab-org/api git https://gitlab.com/gitlab-org/api.git"></head><body>go get gitlab.com/gitlab-org/api</body></html>`が返ってきます。

おそらく現在でもこうだと思われます。

好ましい解決方法は

- サブグループを使わない
- private repositoryを参照できるgo module proxyを運用する

だと思います。

[module proxy](https://go.dev/ref/mod#module-proxy)は`GOPROXY` protocolを実装したHTTPサーバーです。
見たところ実装自体は難しくなさそうですが、これのために1つ運用するサーバーを増やすというのもどうなのかなあという気持ちもあります。

[github.com/goproxy/goproxy](https://github.com/goproxy/goproxy)がgoでgo module proxyを実装しているのでこれを用いるか、kubernetes clusterを動作させているなら[ArtifactHUBから探して](https://artifacthub.io/packages/search?ts_query_web=go+module+proxy&sort=relevance&page=1)Helmで導入するなどするとよいかもしれないです。
筆者はやったことがないためこれ以上は書くことができません。

### source code organization

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
