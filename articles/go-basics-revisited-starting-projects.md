---
title: "Goのプラクティスまとめ: プロジェクトを始めるまで"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## Goのプラクティスまとめ: プロジェクトを始めるまで

筆者が`Go`を使い始めた時に分からなくて困ったこととか最初から知りたかったようなことを色々まとめる一連の記事です。

[以前書いた記事](https://zenn.dev/ngicks/articles/go-basics-earned-through-years-of-practice-part1)のrevisited版です。話の粒度を細かくしてあとから記事を差し込みやすくします。

他の記事へのリンク集

- (まだ)~~[今はこうやる集](https://zenn.dev/ngicks/articles/go-basics-revisited-updated-practices)~~
- プロジェクトを始めるまで: ここ
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

## プロジェクトを始めるまで

新しいプログラミング言語、フレームワーク、ライブラリー、ツール、etcを始めるとき筆者にとってよく障害となるのは「プロジェクトを始めるまでの方法がわからない」ということです。

そこで、この記事では

- [Go]そのものの紹介
  - [A Tour of Go]の紹介や、読み物系へのリンク
- SDK(Software Developement Kit = コンパイラとかライブラリのことをさす)のインストール
- エディターのセットアップ
  - [vscode]など
- プロジェクトの開始方法(=moduleの作成方法)
  - github / gitlabなどにrepositoryを作成してgo moduleを作って動作させるまで
- private repositoryで管理されるgo moduleをインポートできるようにする方法
- タスクランナー

などについてまとめます。

## 対象読者/前提知識

- 会社の同僚
- 今まで[Go]を使ってこなかった人
- ある程度のプログラミング経験
  - [python]や[Node.js]などを引き合いに出すことがあります。
  - [git]は使える
- 高校レベルの英語読解能力

## 環境

win11のwsl2インスタンス内で動作させます。
windowsでの手順も紹介しますが普段使う環境は下記なのでほかの環境への考慮は甘いかもしれません。

```
$ wsl.exe --version
WSL バージョン: 2.6.1.0
カーネル バージョン: 6.6.87.2-1
WSLg バージョン: 1.0.66
MSRDC バージョン: 1.2.6353
Direct3D バージョン: 1.611.1-81528511
DXCore バージョン: 10.0.26100.1-240331-1435.ge-release
Windows バージョン: 10.0.26100.6584
```

[Go]は現在(2025-10-11)の最新です。

```
$ go version
go version go1.25.2 linux/amd64
```

`Go`は後方互換性をかなり気にするためサンプルコードや述べられていることは以降のバージョンでも（何ならこれよりの前のバージョンでも）基本的に一緒ですが、細かいところで改善が入ったりするのでそれを踏まえて読んでください。

一方で、多くのライブラリ直近の2 major versionのみサポートするライブラリが多いとして、Go 1.24以降で新規にできるようになったことは、なるだけ○○以降と書くようにします。

## snippetのconvention

- shellコマンドの羅列の場合コピペしやすさを優先して特になにもprefixをつけずにコマンドを羅列します。
- ただし、コマンド実行結果をsnipet内の併記したい場合は、コマンドは`$ `でprefixされます。
- `# `から始まる行はコメントです。

## 筆者のバックグラウンド

どういう立場からいてるのかっていうのあると多分いいと思たので載せときます。

学生時代に[C++]\(Visual Studio Community 2018だったと思う\)を使ってセンサーから値を読み込んで計算を行うプログラムを作っていました。機械系の学部だったので装置含めて広く浅くだったのでした(あんまりまじめな学生じゃなかったのですしね)。そのため当時からして古い記法を使っていたと思います。何ならUIに[MFC](http://msdn.microsoft.com/ja-jp/library/d06h2x6e.aspx)を使っていました(いまなら厳密なリアルタイム性を要求するところ以外は各プラットフォームのwebviewを使うことになるでしょう)。 実験室PCと居室PCのwindowsバージョンが合わなくてビルドが通らないとか頻発してました。

社会人になってから[Node.js]\([TypeScript]\)でいろいろAPIサーバーを実装, [python]をちょっとだけ使う。[Rust]で`PDFium`とか`OpenSSL`のbindingを書いて利用していました。

[Go]は`goroutine`という[Green Thread](https://en.wikipedia.org/wiki/Green_thread)\(osが提供するthreadと違って、ランタイムが独自にコントロールするがthreadのように扱えるもの。\)を持っていることと、簡素な言語仕様で有名でした。
`goroutine`があれば`async/await`の良くある問題である、

- `async`な関数を導入するとそれの呼び出し側も基本的にはすべて`async`にしなければならない
- `async`な関数に長い時間ブロックする関数を導入すると破綻する

という問題が発生しないため注目し、使い始めました。

少し特殊な環境で動くソフトウェアを書くためデータベース周りの話にあまり明るくなかったりします。
手元でスクリプティングを行う場合はもっぱら[neovim]の[Lua]、[deno], [Go]のいずれかで行っています。

## 基本: Goとは

`Go`とはどういう言語なのかと基本的な読み物系の紹介。

### 特徴

> https://go.dev/doc/
>
> The Go programming language is an open source project to make programmers more productive.
>
> Go is expressive, concise, clean, and efficient. Its concurrency mechanisms make it easy to write programs that get the most out of multicore and networked machines, while its novel type system enables flexible and modular program construction. Go compiles quickly to machine code yet has the convenience of garbage collection and the power of run-time reflection. It's a fast, statically typed, compiled language that feels like a dynamically typed, interpreted language.

これよりもう少し説明すると、[Go]は

- C系統の文法で
- 静的型付けで
- `interface`によるdynamic dispatch(compile時ではなくruntimeにおいて呼び出される関数が決定される仕組み)があり
- GCがあり
  - (GC=garbage collector/collection=いらなくなったメモリを自動的に開放する仕組み。allocation/freeのメモリ管理を手動でやる必要がないという意味)
- 文法や構文が厳選されており、追加もめったにないため書き方がブレにくく
  - と言いつつ[Go 1.18]でgenericsの追加、[Go 1.23]でiterator(for-range-func)の追加がされているがそれでもかなり少ない
- 言語に組み込まれた`goroutine`, いわゆる[Green Thread](https://en.wikipedia.org/wiki/Green_thread)の機能があります。
  - そのため`async/await`的な記法がない
  - `goroutine`は`go`キーワードの後に関数の呼び出しを書くだけでよく、簡単。
  - 言語に組み込まれているので非同期性の実装のためにライブラリが分断されることない。
- コンパイルが非常に速く(遅くない/遅くならないというほうが正しいがここでは置いておく）
- (C-bindingを使わない限り)クロスコンパイルが簡単で
  - (クロスコンパイル/クロスビルド=linuxでwindows向けのビルドを行ったり、その逆を行ったりするような感じで、ある環境から別の環境向けの実行ファイルを出力すること)
- (C-bindingを使わない限り)staticなシングルバイナリを簡単に出力することができ
  - (static=プログラム実行時に動的ロードされるライブラリがない、つまりos/architectureが同じならどこでも動く)
- 組み込まれたモジュールとパッケージマネージャの仕組みがあり

みたいな言語です。

### A Tour of Go

https://go.dev/tour/welcome/1

公式から提供されるチュートリアル。
インタラクティブなコードスニペット/実行環境と、簡単な課題があり、これさえこなせばとりあえず開発は始められます。

筆者の記憶にある限り筆者は合計6時間ぐらいですべて終わりました(当時はGo1.16あたりだったので今はもう少し長い)。
時間のほとんどは構文エラーがよくわかんなくて費やされたのでローカルのエディターにコピーして書いてコピーしなおして実行すればもっと早く終わるかもしれません。

ということで一旦ここは飛ばしてもらってエディターのセットアップを先にしてもらったほうがいいかもしれないですね。

### Go by Example

https://gobyexample.com/

コード例とともに解説がされます。
項目数が多く、知らないstd moduleを使う部分をきちんと理解しようとすると時間がかかると思います。
手が空いたら少しずつ読むのがいいのではないでしょうか。

### 公式読み物系

公式が出している読み物集。全部英語です。
内容が古くなりつつあるため、始めたてで読むべきかは微妙ですが、古くなっても参考になる部分が大いにあります。

- How to Write Go Code: https://go.dev/doc/code
  - コードの書き方以外も含めた基本的なトピック
- Effective Go: https://go.dev/doc/effective_go
  - Goのイディオム集
  - 現在(2025-10-12)だいぶ古くなってきているので、一通り読んだらほかのドキュメントも読むようにしてください
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
部分的に古かったりする割に、`go/types`周りの話など結構アップデートされている部分もある。

## プロジェクトの始め方

`go module`を作ってプログラミングを介するまでのあれこれをまとめておきます。

`Go`の文法周りの話は`A Tour of Go`で網羅されているので説明しません。
これ以降は`A Tour of Go`をこなしことを前提とします。

`Go`をインストールしてエディタをセットアップし、`VCS`(Version Control System)にrepositoryを一つ作り、そこに1つ`go module`を作るところまでをここでカバーします。

`VCS`はここでは`git`しか想定されていません。

## Goのインストール

公式の手順に従い、各OS環境に合わせて[Go]をインストールしましょう。

https://go.dev/doc/install

### Windows

説明のとおり最新バージョンの`.msi`形式のインストーラーを実行しましょう。

インストール先のフォルダを選ぶ以外に選択する個所はありません。

```
$ go version
go version go1.25.2 windows/amd64
```

一応`PATH`に`$(go env GOPATH)/bin`以下が含まれるか確認しておきましょう。
ない場合は、かつてインストールして環境変数を手動で消したときかと思います。

```
$ go env GOPATH
C:\Users\ngicks\go
```

```
$ $env:PATH
C:\WINDOWS\system32;..(省略)..;C:\Users\ngicks\go\bin
```

### Linux

#### ダウンロード

`linux/amd64`の場合、

```
VER=1.25.2
cd $(mktemp -d)
curl -L https://go.dev/dl/go${VER}.linux-amd64.tar.gz -o ./dist.tar.gz
```

一応チェックサムを確認しておいたほうがいいかもしれません。

```
# checksumの値はダウンロードページから確認できる。
CHECKSUM=d7fa7f8fbd16263aa2501d681b11f972a5fd8e811f7b10cb9b26d031a3d7454b
ACTUAL=$(sha256sum ./dist.tar.gz | awk '{print $1}')
[[ $CHECKSUM = $ACTUAL ]]; echo $?
```

一致してれば`0`がprintされます。

#### インストール

公式は`/usr/local`以下に入れる方法を案内しています。

```
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xf ./dist.tar.gz
```

`PATH`が通っていればどこにおいても大丈夫です。
共有サーバーなどで使用する場合は`$HOME`以下に解凍するほうが無難かもしれないです。

#### PATHを通す

環境変数を設定。使っているOS/terminalに合わせた方法で設定してください。

例えば、以下のようなスクリプトを組んで`.bashrc`などから`. /path/to/script`という風に呼び出すとよいでしょう。

```shell
gobin=/usr/local/go/bin/go

export PATH=$($gobin env GOROOT)/bin:$PATH

if [[ -n $($gobin env GOBIN) ]]; then
    export PATH=$($gobin env GOBIN):$PATH
else
    export PATH=$($gobin env GOPATH)/bin:$PATH
fi
```

[Go Modules Reference#go-install](https://go.dev/ref/mod#go-install)より`GOBIN`が設定されていた場合、`go install`でコンパイルされた実行ファイルは設定されたディレクトリに格納されます。設定されない場合、`${GOPATH}/bin`以下に格納されます。

例として`lazygit`を`go install`してみましょう。

```
go install github.com/jesseduffield/lazygit@latest
which lazygit
# 筆者の環境では`$(go env GOPATH)/bin/lazygit`が表示されます。
```

ほっとくと`~/go`にgo moduleのキャッシュとかが入ってしまいます。
`$HOME`以下にディレクトリが増えてほしくないなら以下のように適当なパスに設定しておきます。

```bash
# I'm not using ~/.cache/go since it is somehow populated
export GOPATH="$HOME/.local/go"
export GOBIN="$HOME/.local/go/bin"
export PATH="$GOBIN:$PATH"
```

### mise(windows/linux/mac)

[mise-en-place](https://mise.jdx.dev/)\(以後単に`mise`と呼ぶ\)を使う方法。
もしかしたら一番おすすめかも。windowsでも動作すると書かれていますが筆者はwindowsでは試したことはないです。

`mise`はsdk・環境変数のマネジメント、タスクランナーを全部できるソフトウェアです。
サポートされるsdkは`bun`,`deno`,`elixir`,`erlang`,`Go`,`Java`,`Node.js`,`Python`,`Ruby`,`Rust`,`Swift`,`Zig`のみですが(2025-10-12時点)、そもそも`github`/`gitlab`のrelease pageからツールを落としてきて解凍して配置する機能があるため、大抵何でも管理できます。

`npx`や`go install`,`cargo install`によってインストールできるツールの管理も行えるのがお勧めなポイントです。

#### miseのインストール

公式の案内通りに行います。

https://mise.jdx.dev/installing-mise.html

`self-update`機能があるため、最新を常に使うみたいな用途だと一番上の方法がいいかもしれません。

```
curl https://mise.run | sh
```

もちろんshellに渡す前に落とされるスクリプトはよく読みましょう。

#### miseの設定

https://mise.jdx.dev/configuration.html

などを参考に、以下のような感じで

```toml: ~/.config/mise/config.toml
[settings]
experimental = true
auto_install = true

[tools]
go = "latest"

"go:golang.org/x/tools/gopls" = { version = "latest" }
"go:golang.org/x/tools/cmd/goimports" = { version = "latest" }
"go:mvdan.cc/gofumpt" = { version = "latest" }
"go:github.com/golangci/golangci-lint/cmd/golangci-lint" = { version = "latest" }
"go:github.com/golangci/golangci-lint/v2/cmd/golangci-lint" = { version = "latest" }
"go:github.com/go-delve/delve/cmd/dlv" = { version = "latest" }
```

`mise trust`で信頼できるconfigに指定しないと初回時にtrustするか聞かれます。
あらかじめ`mise trust`しておいたほうがプロンプトが出ないので便利かもしれません。

```
mise trust ~/.config/mise/config.toml
```

#### mise activate, install

`mise activate`で前述の設定をglobalに有効にします。

例えば、以下のようなスクリプトを組んで`.bashrc`などから`. /path/to/script`という風に呼び出すとよいでしょう。

```
# bashの場合
eval "$($HOME/.local/bin/mise activate bash)"
# zshの場合
eval "$($HOME/.local/bin/mise activate zsh)"
```

```
mise install
```

ですべてのツールがinstallできます。

```
$ which go
/home/ngicks/.local/share/mise/installs/go/1.25.2/bin/go
```

という感じで適当なパス以下に導入されます。`mise`を消すときは`~/.local/share/mise`を消したら全部おしまいなのでらくちんですね。

```
mise up
```

ですべてのツールの更新ができます。

最近はサプライチェーン攻撃が怖い([\[1\]](https://semgrep.dev/blog/2025/chalk-debug-and-color-on-npm-compromised-in-new-supply-chain-attack/))ので`"1.25.2"`のようなexact versionを指定して[renovate](https://docs.renovatebot.com/modules/manager/mise/)で自動更新してもらうほうがいいかもしれないですね。もしくは`mise lock`サブコマンドの実装され次第(現状されていない、2025-10-12時点)そちらを使うように移行したほうがいいでしょう([#6231](https://github.com/jdx/mise/pull/6231))。

参考までに: [筆者のdotfilesのmise config](https://github.com/ngicks/dotfiles/blob/main/.config/mise/config.toml)

## エディタのセットアップ

エディタは個人の好みともろもろを合わせて好きに選べばいいと思います。
ただよく聞く+[goのsurvey](https://go.dev/blog/survey2024-h2-results#editor-awareness-and-preferences)の上位3位は以下の3通りです。

- [Visual Studio Code]
- JetBrainsの[GoLand](https://www.jetbrains.com/ja-jp/go/)
- [vim](https://www.vim.org/) / [neovim]

`GoLand`は使ったことないのと多分入れたら終わりなので特に紹介在りません。
`vim`版も似たような設定になる気はしますが、筆者は`neovim`しか使っていないので`vim`側の設定の紹介はありません。

### Visual Studio Code

[Go extension](https://marketplace.visualstudio.com/items?itemName=golang.go)を入れたら終わりです。

初めて`Go`のプロジェクトを開いたときに依存先をインストールするかのポップアップが出るので、インストールしておきます。
もしくは`Ctrl+Shift+P`(環境によってバインドは異なるかも、特にmac)でcommand palletを開き、`Go: Install/Update Tools`と打ち込んでEnterします。

もう少しカスタマイズしたいならgoplsの設定を行います。

`vscode`を開き、`Ctrl+Shift+P`でcommand palletを開き、`Preferences: Open User Settings(JSON)`と打ち込んでEnterします。

以下を追記します。(コメント行はコピーしないでください。)

```json
{
  "editor.formatOnSave": true,
  // https://github.com/golang/tools/blob/master/gopls/doc/settings.md
  "gopls": {
    "ui.semanticTokens": true,
    "ui.diagnostic.analyses": {
      // https://staticcheck.dev/docs/checks
      "nilness": true,
      "nonewvars": true,
      "useany": true
    }
  },
  "go.lintTool": "golangci-lint-v2"
}
```

リンク先を参考に設定をいろいろ変えてみてください。
とりあえず`semanticTokens`は現状デフォルトで`false`です: [#45313](https://github.com/golang/go/issues/45313#issuecomment-2161267130)

`"go.lintTool": "golangci-lint-v2"`は`staticcheck`に不足ある場合のみ設定します。

### neovim

`v0.11`のみの話します。versionごとに設定違うかもしれないので以降のバージョンだと違うかも。

```
$ nvim --version
NVIM v0.11.4
Build type: Release
LuaJIT 2.1.1741730670
Run "nvim -V1 -v" for more info
```

#### 依存先のインストール

まず依存ツールを収集します。
下記のリストを参考に`go install`を個別にinstallするか、

https://github.com/golang/vscode-go/blob/c8ab8fbd64502aaa890e2ea3622a43ee8336d009/extension/tools/installtools/main.go#L33-L50

もちろん`mise`で管理してもよいと思います。

```toml

[tools]

go = "latest"

# gopls本体
"go:golang.org/x/tools/gopls" = { version = "latest" }
# formatter
"go:golang.org/x/tools/cmd/goimports" = { version = "latest" }
"go:mvdan.cc/gofumpt" = { version = "latest" }
# debugger
"go:github.com/go-delve/delve/cmd/dlv" = { version = "latest" }
# test generator
"go:github.com/cweill/gotests/gotests" = { version = "latest" }
# goplay client
"go:github.com/haya14busa/goplay/cmd/goplay" = { version = "latest" }
# linter
"go:honnef.co/go/tools/cmd/staticcheck" = { version = "latest" }
```

ただし依存先は変わることがあるので[allTools.ts.in](https://github.com/golang/vscode-go/blob/master/extension/tools/allTools.ts.in)の内容から生成したほうがいいかもしれないですね。

#### goplsの設定

[gopls]は`Go`の言語サーバーです。

言語サーバー(Language Server)は[LSP](https://microsoft.github.io/language-server-protocol/)\(Language Server Protocol\)を実装するサーバーです。この`LSP`というのが「定義に飛ぶ」とか「参照を検索」とかの機能を実装するための通信の取り決めです。この位置のトークンの参照先を全部教えてとjson-rpc2で問い合わせると、参照先の位置のリストが返ってくる、みたいなものをイメージしていただければ大体あっています。

- 適当なpackage managerで[neovim/nvim-lspconfig](https://github.com/neovim/nvim-lspconfig)を`rtp`に加えておく。
  - `require "lspconfig"`してエラーしなければいいということです。
- `neovim/nvim-lspconfig`の設定では不足ある場合は、`~/.config/nvim/after/lsp/gopls.lua`を作成して適当に設定を上書きする。
- どこかの`lua`スクリプトから[vim.lsp.enable](<https://neovim.io/doc/user/lsp.html#vim.lsp.enable()>)`("gopls")`を呼び出す。
- `gopls`をインストールして`$PATH`を通す。
  - [williamboman/mason](https://github.com/williamboman/mason.nvim)で管理したいなら`ensure_installed`に`"gopls"`を指定しておくとよい。

筆者は`~/.config/nvim/after/lsp/gopls.lua`を以下のようにしています。

- `autocmd`を設定しないとフォーマットがかからなかったので入れています。
- 設定はverboseだと思ったらfalseにしています。
- `completeUnimported = true`があるとまだimportしてない依存先のauto completeも動作するようになります(デフォルトはfalse)。あるとないでは体験がだいぶ違います。
- 詳しくないのでもっと簡易な設定があったらすみません。

```lua: ~/.config/nvim/after/lsp/gopls.lua
vim.api.nvim_create_autocmd("BufWritePre", {
  pattern = "*.go",
  callback = function(args)
    local clients = vim.lsp.get_clients { name = "gopls" }
    if #clients == 0 then
      return
    end

    local client = clients[1]

    local pos_encoding = vim.lsp.get_client_by_id(client.id).offset_encoding or "utf-16"

    local params = vim.lsp.util.make_range_params(vim.fn.bufwinid(args.buf), pos_encoding)
    params = vim.tbl_deep_extend("force", params, { context = { only = { "source.organizeImports" } } })

    -- buf_request_sync defaults to a 1000ms timeout. Depending on your
    -- machine and codebase, you may want longer. Add an additional
    -- argument after params if you find that you have to write the file
    -- twice for changes to be saved.
    -- E.g., vim.lsp.buf_request_sync(0, "textDocument/codeAction", params, 3000)
    local result = vim.lsp.buf_request_sync(0, "textDocument/codeAction", params)
    for cid, res in pairs(result or {}) do
      for _, r in pairs(res.result or {}) do
        if r.edit then
          local enc = (vim.lsp.get_client_by_id(cid) or {}).offset_encoding or "utf-16"
          vim.lsp.util.apply_workspace_edit(r.edit, enc)
        end
      end
    end
    vim.lsp.buf.format { async = false }
  end,
})

return {
  settings = {
    gopls = { -- https://github.com/golang/tools/blob/master/gopls/doc/settings.md
      analyses = {
        -- https://staticcheck.dev/docs/checks
        ST1003 = false,
        fieldalignment = false,
        fillreturns = true,
        nilness = true,
        nonewvars = true,
        shadow = false,
        undeclaredname = true,
        unreachable = true,
        unusedparams = true,
        unusedwrite = true,
        useany = true,
      },
      codelenses = {
        generate = true, -- show the `go generate` lens.
        regenerate_cgo = true,
        test = true,
        tidy = true,
        upgrade_dependency = true,
        vendor = true,
      },
      hints = {
        assignVariableTypes = true,
        compositeLiteralFields = true,
        compositeLiteralTypes = true,
        constantValues = true,
        functionTypeParameters = true,
        parameterNames = true,
        rangeVariableTypes = true,
      },
      buildFlags = { "-tags", "integration" },
      completeUnimported = true,
      diagnosticsDelay = "500ms",
      gofumpt = true,
      matcher = "Fuzzy",
      semanticTokens = true,
      staticcheck = true,
      symbolMatcher = "fuzzy",
      -- I've used for a while with this option, and found it annoying.
      -- Great feature tho.
      usePlaceholders = false,
    },
  },
}
```

#### golangci-lintの設定

[golangci-lint](https://golangci-lint.run/)はいろんなlintツールをまとめて実行できるようなもので、デファクトスタンダード化してるのでこの記事としては紙面を割かねばなりません。

`golangci-lint`自体はcliツールですが、[golangci-lint-langserver](https://github.com/nametake/golangci-lint-langserver)というさらなる外部コマンドによって言語サーバーとして使用できます。

基本は`nvim-lspconfig`に設定があるので`mason`で`golangci-lint`,`golangci-lint-langserver`をインストールし、configのどこかから[vim.lsp.enable](<https://neovim.io/doc/user/lsp.html#vim.lsp.enable()>)`("golangci_lint_ls")`を呼び出せばとりあえずは良いです。
下記です。

https://github.com/neovim/nvim-lspconfig/blob/master/lsp/golangci_lint_ls.lua

ただし、`golangci-lint`にはv1, v2があるんですが、それらはconfigのフォーマットに互換性がありません。v2は比較的最近([2025-03-24](https://github.com/golangci/golangci-lint/releases/tag/v2.0.0))出たので、世間に存在する設定ファイルはv1, v2混在していています。
そこで、configに合わせてv1, v2を切り替えるようにします。

めっちゃ`mise`に依存してます。v1, v2両方入れておきます。

```toml: ~/.config/mise/config.toml
[tools]
"go:github.com/nametake/golangci-lint-langserver" = { version = "latest" }
"go:github.com/golangci/golangci-lint/cmd/golangci-lint" = { version = "latest" }
"go:github.com/golangci/golangci-lint/v2/cmd/golangci-lint" = { version = "latest" }
```

`golangci-lint config verify`をv2, v1両方で実行し、passしたほうの設定を使います。

```lua: ~/.config/nvim/after/lsp/golangci_lint_ls.lua
local markers = {
  ".golangci.yml",
  ".golangci.yaml",
  ".golangci.toml",
  ".golangci.json",
}

return {
  cmd = { 'golangci-lint-langserver' },
  filetypes = { 'go', 'gomod' },
  root_dir = function(bufnr, on_dir)
    local fname = vim.api.nvim_buf_get_name(bufnr)
    local root = vim.fs.root(fname, markers)
    if root then
      on_dir(root)
    end
  end,
  root_markers = markers,
  before_init = function(_, config)
    -- switch version based on config schema.
    -- golangci-lint v2 is relatively new.
    -- So some projects still are sticking to v1,
    -- while some other has been migrated to v2.

    -- check v2 first since basically it is stricter
    -- in some aspect, e.g. required version top element.
    local v2 = vim
      .system({
        "mise",
        "exec",
        "go:github.com/golangci/golangci-lint/v2/cmd/golangci-lint",
        "--",
        "golangci-lint",
        "config",
        "verify",
      })
      :wait()

    if v2.code == 0 then
      config.init_options.command = {
        "mise",
        "exec",
        "go:github.com/golangci/golangci-lint/v2/cmd/golangci-lint",
        "--",
        "golangci-lint",
        "run",
        "--output.json.path=stdout",
        "--show-stats=false",
      }
      return
    end

    local v1 = vim
      .system({
        "mise",
        "exec",
        "go:github.com/golangci/golangci-lint/cmd/golangci-lint",
        "--",
        "golangci-lint",
        "config",
        "verify",
      })
      :wait()

    if v1.code == 0 then
      config.init_options.command = {
        "mise",
        "exec",
        "go:github.com/golangci/golangci-lint/cmd/golangci-lint",
        "--",
        "golangci-lint",
        "run",
        "--out-format",
        "json",
      }
      return
    end

    vim.notify('"golangci-lint config verify" failed for both v1 and v2', vim.log.levels.WARN)
  end,
}
```

## VCS(git)でrepositoryを作成する

手順そのものは説明しませんが、以後の手順は[github](https://github.com/)や[gitlab](https://about.gitlab.com/)でrepositoryが存在していることを想定します。

[VCS(Version Control System)](https://en.wikipedia.org/wiki/Version_control)は, コンピュータファイルのバージョンを管理するシステムのことです。
代表的なものは[git]や[svn](https://en.wikipedia.org/wiki/Apache_Subversion),[mercurial](https://en.wikipedia.org/wiki/Mercurial)あたりだと思います。
この記事では`git`のみを取り扱います(筆者がほか二つのことをほぼまったく知らないからです)

`git`は、`VCS`を構築するためのサーバーおよびクライアントプログラムです。サーバーとして直接使うことはほとんどないかもしれません。
現在では`git`サーバーは[github](https://github.com/)というwebサービスを利用するか、 セルフホストすることも可能な[gitlab](https://about.gitlab.com/)、あるいは[gitbucket](https://github.com/gitbucket/gitbucket)などを使うのが一般的だと思います。(この3つがリストされてるのは単に筆者が使ったことあるやつ3種っていうだけです)

以後の説明はremoteがすでに存在していることを前提とするため、とりあえず何か作っておいてください。別にローカルで作成してからremoteを指定しても構いません。

## Go moduleの作成

`Go`のプログラムは[Go 1.11]以降1つまたは複数の`Go module`から構成されます。

この項では`Go module`の新規作成方法と

### git clone

まず先ほど`git`(VCS)で作成したrepositoryをローカルに`git clone`しておきます。
`clone`で作成されたディレクトリに`cd`で移動しておきます。ここが`Go module`の`module root`となります。

```
git clone ${uri}
cd ${repo-name}
```

おすすめなのは`${GIT_REPOSITORY_BASE}/${domain}/${path}`のディレクトリ構成をとっておくことです。こうすれば同じ名前のrepositoryがあってもパスが被ることがなくなります。

例えば以下のような感じ

```
# uriが
# https://github.com/ngicks/go-example-basics-revisited
# であるとき
mkdir -p "${HOME}/gitrepo/github.com/ngicks/go-example-basics-revisited"
git clone https://github.com/ngicks/go-example-basics-revisited "${HOME}/gitrepo/github.com/ngicks/go-example-basics-revisited"
cd "${HOME}/gitrepo/github.com/ngicks/go-example-basics-revisited"
```

このスタイルでの管理をツール化する際には、筆者は使っていないですが以下のようなものを使うのもいいかもしれないです。

- [ghq](https://github.com/x-motemen/ghq)

### Go moduleの初期化

`Go module`を初期化します。

```
go mod init ${module_name}
```

`${module_name}`は基本的に先ほど作成したVCS repositoryの`${uri}`からプロトコルスキームを抜いたものにします。

先ほどの例で行くと、`${uri}`は`https://github.com/ngicks/go-example-basics-revisited`であるので

```
go mod init github.com/ngicks/go-example-basics-revisited
```

となります。

これを含めて`VCS`にプッシュすると

```
go get github.com/ngicks/go-example-basics-revisited
```

でほかの`Go module`から導入、参照できます。

:::details local onlyのモジュール

local only, つまりオンラインで公開する気のない場合は`${module_name}`は基本的にはなんでもよいです。

ただし、std libraryのトップレベルパッケージと同名のものと、[`"mod"`が使用できません](https://github.com/golang/go/blob/go1.25.2/src/cmd/go/internal/load/pkg.go#L849-L856)\(他にもあるかも\)。
`import "fmt"`でstd libraryがインポートできることからわかる通り、これらは特別扱いされます。`"mod"`も似たような理由です。

`Go module`の先頭はドメイン名が使われることが想定されます。そこで、

- `package.test`
- `package.invalid`

などを使用するとよいと思います([RFC6761](https://www.rfc-editor.org/rfc/rfc6761))。

:::

### (private gitかつサブグループを使用する場合)module nameに`.git`をつける

一部の`VCS`, `gitlab`などは通常の`https://${domain}/${organization}/${reponame}`階層構造を超えて、さらにサブグループを作成することができます。
つまり`https://${domain}/${organization}/${group_name1}/.../${group_nameN}/${reponame}`なるわけですね。

そのような2階層以上のグループを持ち、privateなgit出る場合、`Go module`の名前の末尾に`.git`とつけなければならないことがあります。
より正確に言うと、`git clone`しないといけないrepositoryのパスとなる部分に`.git`とつける必要があります。

```
`https://${domain}/${organization}/${group_name1}/.../${group_nameN}/${reponame}.git/path/to/submodule`
```

:::details gitlabを用いて行った検証

privateなので特に明かすことはできませんが、行った検証を記します。
**publicだと両方`go get`できました。**

現在(2025-10-13)gitlabで

- privateのgroup/subgroupを作り、その下にprojectを1つ作ります。
- projectをcloneし、サブディレクトリを二つ作ります。
- 片方で`go mod init gitlab.com/${group}/${subgroup}/prj/sub1`とし
- もう片方で`go mod init gitlab.com/${group}/${subgroup}/prj.git/sub2`とします。
- プッシュしておきます。

```
cd $(mktemp -d)
go mod init sample.test
export GOPRIVATE=gitlab.com/${group}
```

```
$ go get gitlab.com/${group}/${subgroup}/prj/sub1
go: module gitlab.com/.../proj/sub1: git ls-remote -q origin in /home/ngicks/.local/go/pkg/mod/cache/vcs/be5b4b9be0c9e04a2fc93d37972ba2206efb91f39ed469c1442b1b1d
9ee34ab3: exit status 128:
        remote: The project you were looking for could not be found or you don't have permission to view it.
        fatal: repository 'https://gitlab.com/.../subgroup.git/' not found
```

```
$ go get gitlab.com/${group}/%{subgroup}/prj.git/sub2
go: downloading gitlab.com/...
go: downloading gitlab.com/...
go: added gitlab.com/...
```

:::

なぜかというと、

前提知識としてgoではpublicではないregistryからmodule fetchを行いたい場合`GOPRIVATE`にpath prefixを指定します。
その場合、特に指定されていないと[direct access](https://go.dev/ref/mod#private-module-proxy-direct)と言って、`git`などの`VCS`に対応するコマンドを直接使います。

`go tool`はmodule pathを与えられると[module pathに?go-get=1をつけてhttp getすることでmodule metadataを得ようとします。](https://go.dev/ref/mod#vcs-find)
[Go 1.24]まで、`.netrc`を用いる以外に`go tool`にcredentialを渡す方法がありませんでした。
`.netrc`は平文で機密情報を書き出す必要があるため、通常の環境では用意されないと思います。
そのため`?go-get=1`は認証情報なしでリクエストされることがほとんどだったと思われます。
試してみれば分かりますが、現時点(2025-10-13)でgitlabの存在しないパスに`?go-get=1`をつけてリクエストしてみれば、存在しないメタデータを返してくるのがわかります(できればself-hostしているインスタンスで試して公式のgitlabに迷惑をかけないようにしましょう)
何かしらのエラーを返したり成功してしまうと、正しく認証を行っていない攻撃者に対して情報を漏らすことになってしまうので、できない、ということになります。
後方互換性のことを考えるとこの挙動が変わることはまずない気がします。

下記のソースより、`gitlab`を含めて`go tool`にとってパスパターンが既知でないものは末尾の`// General syntax`のところにマッチします。
つまり、`${domain}/${organization}/${project}`だと思って処理されます。ただし、`regexp`を見るとわかる通り`\.(?P<vcs>bzr|fossil|git|hg|svn))`にマッチすればそのパスまでが`VCS`のターゲットだとして処理されます。

https://github.com/golang/go/blob/go1.25.2/src/cmd/go/internal/vcs/vcs.go#L1565-L1627

ということで、privateかつ２階層以上のグループを持つ場合はmodule nameのrepository部分に`.git`をつけましょう。

:::details vcs suffixをつけたくない場合には？

もし仮にmodule nameに`VCS` suffixをつけたくないとしたら、好ましい解決方法は

- サブグループを使わない
- vanity import pathを返すサーバーを運用する
- private repositoryを参照できるgo module proxyを運用する

だと思います。
筆者は`VCS` suffixをつける方法に甘んじているためすべて試していないことに注意願います。

- サブグループを使わない:
  - 単純ですが、サブグループを使わないだけでも解決します。
  - 前述通り`VCS` suffixがない場合、`${domain}/${organaization}/${project}`というフォーマットとして解釈され、このパスにまず`?go-get=1`付きのHTTP GETが試みられます。前述のとおり少なくとも`gitlab`ではこれは成功するため意図通りに動作すると思われます。
- vanity import pathを返すサーバーを運用する:
  - 例えば[go.uber.org/zap](https://github.com/uber-go/zap)は実際にはgitubでホストされています。
  - `curl https://go.uber.org/zap/zapcore?go-get=1`を実行するとわかりますが、`<meta name="go-import" content="vanity/module/path VCS https://path/to/VCS">`が返ってきます。
  - このようにmodule pathと実際にソースコードがホストされるURLが違う時、module pathをvanity import pathなどと呼ぶのが通例のようです。
  - `?go-get=1`で正しい内容が返りさえすれば(=module root pathが正しく取れれば)あとは成功するでしょうから、これでもうまくいくと思われます。
  - [Go 1.25]からサブパスがmodule rootとなる場合の`go-import`のフォーマットの考慮が追加されたため、mono repo運用をする場合はこれへ追従が必要だと思います。
- go module proxyを運用する:
  - 最も正道で最も大変な方法と思われます。
  - [module proxy](https://go.dev/ref/mod#module-proxy)は`GOPROXY` protocolを実装したHTTPサーバーです。
  - 見たところ実装自体は難しくなさそうですが、これのために1つ運用するサーバーを増やすというのもどうなのかなあという気持ちもあります。
  - [github.com/goproxy/goproxy](https://github.com/goproxy/goproxy)がgoでgo module proxyを実装しているのでこれを用いるか、
  - kubernetes clusterを動作させているなら[ArtifactHUBから探して](https://artifacthub.io/packages/search?ts_query_web=go+module+proxy&sort=relevance&page=1)Helmで導入するなどするとよいかもしれないです。
  - このmodule proxy server自体にauthが必要ですので[GOAUTH]を各clientに設定してもらうか、イントラからしかアクセスできないようにするかする必要があります。これはこれで大変ですね。

module proxyを運用したい別の理由があるなら話は違いますが、`VCS` suffixをつけてしまうのが一番楽です。

:::

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
[Go 1.25]: https://go.dev/doc/go1.25

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
