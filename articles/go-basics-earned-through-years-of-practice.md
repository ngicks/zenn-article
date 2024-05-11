---
title: "Goで開発して3年のプラクティスまとめ"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

# Goで開発して3年たったのでプラクティスをまとめる

筆者は[Go]を触りだして3年ぐらいたったので、触りだしてからもっと早く知りたかったこととかをまとめて置くことで筆者の知識の整理を行います。

なるだけワンストップでたくさんの話題を扱うようにして、
読んだ人がなんとなく開発を始められるようになって、
なんとなくイディマティックなコードを書けるようにするのが目的です。

# 筆者のバックグラウンド

筆者のバックグラウンドを書くことで、アンビエントに存在する前提知識を掲示します。

- 学生時代は[C++]を使ってセンサー値を読み込んで計算を行うプログラムを記述していた。
  - つらかった
  - Windows 7
  - Visual Studio Community
- 入社してからメイン[Node.js]、ほんのちょっぴり[python]で開発を行う
- 独学で[The Rust Programming Language 日本語]を(確か2018年エディションを)読了、PDFiumやOpenSSLのbindingをrustで書いたりして使ってました
- その後[Go]を使い始める。多分一番慣れてる言語です。

つまり筆者は[Go]を使い始める前に以下をなんとなく理解しています

- [C++]・・・何年も前なので新しい文法は使っていない
- [Node.js]
- [TypeScript]
- [Rust]
- `Linux`上でファイルを読み書きしたりデータストレージとやり取りするときに起きる諸般の問題
  - [open(2)](https://man7.org/linux/man-pages/man2/open.2.html)を`O_TRUNC`付きで呼んでから[write(2)](https://man7.org/linux/man-pages/man2/write.2.html)がリターンするまでの間にファイルが0バイトの状態が観測できる、とか

筆者は`Linux`が動作する小さ目のデバイスで動くプログラムしか書かないので、
クラウドとかそういったものが視点に入っていません。
結構特殊な視点で書かれているかもしれないです。

# 対象読者

- いままで[Go]を使ってこなかった人
- ある程度コンピュータとネットワークとプログラムを理解している人
- [python]とか[Node.js]でなら開発したことある
- [git]は使える。

# 対象環境

- メインは`linux/amd64`です
- 別段`OS`/`arch`固有な要素は少ない(インストールの部分のみ)と思いますが、適宜読み替えてほしいです。

# 基本

## A Tour of Go

https://go.dev/tour/welcome/

`Go`の基本的なトピックはここに書いてあります。
インタラクティブなコードスニペットと簡単なエクササイズがあり、これさえこなせばとりあえず開発は始められます。

## Go by Example

https://gobyexample.com/

コード例とともに解説がされます。手が空いたら少しずつ読むのがいいのではないでしょうか。

## 公式読み物系

TODO: 読む

公式が出している読み物集。全部英語です。
初めから全部読む必要はないと思いますが折を見て読んでいくのがよいと思います。

- The Go Language Specification: https://go.dev/ref/spec
  - 言語仕様ですが割と短めなのでそのうち読んでおいたほうがよいでしょう。
- How to Write Go Code: https://go.dev/doc/cod
  - コードの書き方以外も含めた基本的なトピック
- Effective Go: https://go.dev/doc/effective_go
  - Goのイディオム集
- Go Wiki: Go Code Review Comments: https://go.dev/wiki/CodeReviewComments
  - よくされるCode review comment集らしいです

## Std library

TODO: いった手前自分でも読む

https://pkg.go.dev/std

standard libraryです。

HTTPなどで動作するサーバープログラムを作るのに大体必要な機能がそろっています。
できれば開発に着手する前にすべてのインターフェイスとdoc commentを読んでおくがよいと思います。

https://pkg.go.dev/golang.org/x

こっちはsub-repositoriesです。説明のとおり、Go Projectの一環ですがstd libほど厳密なバージョン管理がされていません。
std libに入ると厳密な後方互換性の約束を守る必要があるため、変更の可能性が高かったり、stdに入れるほどの重要度がないものがこちらにあるというコンセプトのはずです。

- ここで先に実装されてからstdに昇格されたり(`maps`, `slices`など)、
- stdがFrozenなので代わりにこちらのものを使うべきだったり(`syscall`の代わりに[golang.org/x/sys](https://pkg.go.dev/golang.org/x/sys#section-readme))
- 1ファイルにバンドルされてstdに組み込まれていたり([golang.org/x/net/http2](https://pkg.go.dev/golang.org/x/net/http2))

することもあります。

## golang/example

TODO: 全体をざっと眺める。

https://github.com/golang/example

公式でメンテされているexample集。新しめな話題は取り扱っていないこともある。

## 外部リソース（未読）

TODO: さっと目を通しておこう。100 Go mistakes...は面白そうなので読んでおきたい。

- https://tour.ardanlabs.com/tour/eng/list
- 100 Go Mistakes and How to Avoid Them
  Book by Teiva Harsanyi

# プロジェクトの始め方

モジュールをセットアップするまでのあれこれをまとめておきます。

公式ドキュメントに網羅されている内容ですのでそちらに当たってもらってもよいでしょう。
特に基本的文法は`The Tour of Go`で網羅的に述べられるので説明しません。

`VCS`(Version Control System)にrepositoryを一つ作り、そこに1つ`Go module`を作るところまでをここでカバーします。

`VCS`はここでは`git`しか想定されていません。

## インストール

公式の手順に従い、各OS環境に合わせて[Go]をインストールしましょう。

https://go.dev/doc/install

`linux`/`amd64`の場合はいつもの手順です

```
mkdir -p /tmp/go-download
cd /tmp/go-download
curl -LO https://go.dev/dl/go1.22.3.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.22.3.linux-amd64.tar.gz
```

一応チェックサムを確認しておいたほうがいいかもしれません。(shellを使い倒すのに慣れていないのでコマンド自体は参考程度に)

```
# checksumの値はダウンロードページから確認できる。
echo 8920ea521bad8f6b7bc377b4824982e011c19af27df88a815e3586ea895f1b36 > checksum
sha256sum go1.22.3.linux-amd64.tar.gz | awk '{print $1}' | diff - checksum
```

環境変数を設定。使っているOS/terminalに合わせた方法で設定してください。

```
PATH=$(/usr/local/go/bin/go env GOPATH)/bin:/usr/local/go/bin:$PATH
```

`/usr/local/go/bin`以下に、先ほど`.tar.gz`から解凍した`go`コマンドと`gofmt`コマンドが置かれます。

`$(go env GOPATH)/bin`以下には`go install`したバイナリがおかれます。
パスを通しておけばコマンドとして利用できるようになります。

## エディタ

エディタは個人の好みともろもろを合わせて好きに選べばいいと思います。
ただよく聞くのは以下の3通りです。

- [Visual Studio Code] + [Go extension](https://marketplace.visualstudio.com/items?itemName=golang.go)（筆者はこれ）
  - https://code.visualstudio.com/docs/languages/go
- [vim](https://www.vim.org/) or [neovim](https://neovim.io/) + gopls
  - https://github.com/golang/tools/blob/master/gopls/doc/vim.md
- JetBrainsの[GoLand](https://www.jetbrains.com/ja-jp/go/)

## VCSでのrepositoryの作成

詳しい説明はしませんが、以下の手順は[github](https://github.com/)や[gitlab](https://about.gitlab.com/)でrepositoryが存在していることを想定しますので、
先に作成しておきます。

[VCS(Version Control System)](https://en.wikipedia.org/wiki/Version_control)は, コンピュータファイルのバージョンを管理するシステムのことです。
代表的なものは[git]や[svn](https://en.wikipedia.org/wiki/Apache_Subversion),[mercurial](https://en.wikipedia.org/wiki/Mercurial)あたりだと思います。
この記事では`git`のみを取り扱います(筆者がほか二つのことをほとんど知らないからです)

`git`は、`VCS`を構築するためのサーバーおよびクライアントプログラムです。サーバーとして直接使うことはほとんどないかもしれません。
現在では[github](https://github.com/)というwebサービスを利用するか、 セルフホストすることも可能な[gitlab](https://about.gitlab.com/)、あるいは[gitbucket](https://github.com/gitbucket/gitbucket)などを使うのが一般的です。

先にローカルでrepositoryを作成してあとからremote上に作成する方法もあるはずですが、この説明ではremoteを先に作る方法しか想定されません。

## Go moduleの初期化

VCSで作成したrepositoryをローカルにクローンします。

```
# gitの場合
git clone <<uri>>
cd <<repo-name>>
```

`go mod init <<module-name>>`で`Go module`に必要なファイルを作成します

```
go mod init <<module-name>>
```

TODO: もろもろの裏どり

`<<module-name>>`は基本的に上記`<<uri>>`からプロトコルスキームを抜いたものにするとよいです。
そうすると`go get <<module-name>>`でこのモジュールを別の`Go module`へ導入できるためです。

例えば、`VCS`の`URI`が`https://github.com/ngicks/example`である場合、

```
go mod init github.com/ngicks/example
```

で作成し、`VCS`にソースをプッシュすると

```
go get github.com/ngicks/example
```

で別モジュールから導入、参照できます。

ただし、`gitlab`でサブグループを作成し、サブグループの中でソースを管理する場合、
`<<module-name>>`はvcsのsuffixを加えておかないと`go get`時に失敗するかもしれません。

TODO: 裏をとって現時点(`2024/05/10`)では確実に失敗すると断言する口調に変える。出典を明記。

つまり上記と同じ例で行くと

```
go mod init github.com/ngicks/example.git
```

とする必要があるということです。

筆者の利用する`gitlab`ではとりあえずこうすることで動作しますが、これがモノレポで複数の`Go module`が管理される場合どうなるかなどわからないので注意してください。

以前調べたときの`go tools`側の見解としては`gitlab`のバグという立場でした(TODO: issueへのリンク)。

## Private repositoryでソースをホストする場合

### GORPIVATEを設定する

https://go.dev/ref/mod#private-modules

上記の説明より、一般公開されない、つまり特別な認証が必要な`VCS`でソースを管理し、`go get`などでモジュールをインポート/ダウンロードする場合、

- `GOPRIVATE`の設定
- その`VCS`のcredentialの適切な保存

を行う必要があります。

`GOPRIVATE`の設定は以下で行います。

```
# git repositoryのURIが https://example.com
# である場合、<<host>>は`example.com`になります。
go env -w GOPRIVATE <<host>>
```

(環境変数で指定すればよいと書かれていますが、筆者はうまくいかなかったので`go env -w`で書き込んでいます。)

`git credential`の適切な保存には筆者は[Git Credential Manager](https://github.com/git-ecosystem/git-credential-manager?tab=readme-ov-file)を利用しています。
[Install instructions](https://github.com/git-ecosystem/git-credential-manager/blob/release/docs/install.md)に従いセットアップを行うと、`wsl`の場合は`wincred`への保存、`native linux`の場合は`pass`プログラムなどとの連携となります。

`GONOPROXY`, `GONOSUMDB`(`NO`であることに注意)を設定しない場合、`GOPRIVATE`がデフォルトとして使われます。
`GONOPROXY`に設定されたホストからのモジュール取得する(`direct` mode)際には相手`VCS`に合わせたコマンドが使用されます(`git`の場合`git`コマンド -> [modfetch](https://github.com/golang/go/blob/74a49188d300076d6fc6747ea7678d327c5645a1/src/cmd/go/internal/modfetch/codehost/git.go#L249))

### git-lfsを導入している場合はすべての環境でgit-lfsを使うように気を付ける

上記のような設定で`direct`モードで`Go module`が取得される場合、
`git-lfs`の導入有無で`git`からのfetch後の内容が異なることがあります。
これによってsum照合エラーで`go module download`が失敗する現象を何度か体験しています。

基本的にはすべての環境(`Dockerfile`なども含む)で`git-lfs`を導入しておくほうがよいでしょう。

[Git Large File Station](https://git-lfs.com/)は`git`で大きなファイルを取り扱うための拡張機能です。
`git-lfs`はhookとfilterを活用してコミット前後でトラック対象のファイルをテキストファイルのポインターに変換し、
トラックされた大きなファイルはremote repositoryではなく大容量ファイル用のサーバーに上げるような挙動になります。(参考: https://github.com/git-lfs/git-lfs, [Git LFS をちょっと詳しく](https://qiita.com/ikmski/items/5cc8b8832336b8d85429))

[`github`](https://docs.github.com/ja/repositories/working-with-files/managing-large-files/about-git-large-file-storage), [`gitlab`](https://docs.gitlab.com/ee/administration/lfs/index.html)双方とも`git lfs`に対応しています。

開発の経緯的に、想定された用途ははゲームなどで大きなバイナリファイルを一緒に管理することのようです。
それ以外でもテスト用の大きなファイルを管理するときなどにも使うことがあると思います。

## エントリーポイントを作成する

```

mkdir -p cmd/example
touch cmd/example/main.go

```

```go: main.go
package main

import "fmt"

func main() {
  fmt.Println("Hello")
}
```

## パッケージを分ける

ディレクトリ=パッケージです

他のパッケージをインポートするときは基本的にfully qualifiedなパスで
（確か相対パスインポートもできるけどしないほうが良いと言われてたよな…TODO: 調べる）

go.modのreplaeとかvendorとかの話は省こう
internalパッケージの話はもっと後半の細かい話し始めるところで書く？

## build / run

```
go run ./cmd/example
go build ./cmd/example
```

ポイントは

```
go build cmd/example
```

ではダメだということ。

TODO: `go`コマンドのパスの扱いがこうであることの根拠を明示する

# プロジェクトの歩き方

git cloneなりしてきたgo projectの歩き方

## cmdディレクトリ

慣習的にmainパッケージがこれ以下のサブフォルダとかに含まれる。

ライブラリとして使われるつもりがあまりないプロジェクトはトップディレクトリがmainパッケージになってることが多い（見た限り）

## func mainで検索

main関数

## go:generateで検索する

Makefileとかを使わず全部go:genrateで作業スクリプトが実行されてることがある（かも）

# 基本的プラクティス

## マルチスレッド制御

- sync: https://pkg.go.dev/sync
- sub repositoryのsync: https://pkg.go.dev/golang.org/x/sync

使い方とかを書く

## テスト関連

- Fuzzについて
- ユーティリティー
  - gotest.tools/v3/assert

### Fuzz時にはメモリ使用量に気を付ける

fuzzテスト時にはworker=cpu個数で同時多数にテストが走るので普段よりもメモリーを使う

## gotchas系

- typed nil
- sliceは値
- mapはポインター
- iterator variableのキャプチャー（Let's Encryptのミス）（Go1.22.0以降では起きない）

### defer, go statementで関数を呼び出すときの引数は記述順で評価される

例えば（以下、例となるスニペット追記）

関数の実行後の値を取りたい場合はポインターで渡すか、deferで呼び出すのを無名関数にして変数をキャプチャする

## HTTP Server framework

- stdで十分な機能があることに触れる
  - go 1.22で追加されたルーティングに触れながら
- echo, chi, ginあたりについて説明する
  - 筆者はechoしか使ったことがないと断りをいれる
- gRPCの一通りの使い方に触れる

### OpenAPIとGo

- OpenAPIに付いて説明し、基本的な書き方とコードジェネレータについて説明する
  - 筆者はoapi-codegenしかつかったことがない
- github.com/atombender/go-jsonschema による型の生成とバリデーション
  - OpenAPI v3.0.xではjsonschemaのサブセットの拡張版であることに触れる

## code generator

Goは文法が単純でマクロが存在しないので、代わりにcode generatorが用いることが多い。

- text/templateの紹介
- github.com/dave/jennifer
- go/astによるコードの解析

## logging

- slog
- zap

最近はslogだけでいいんじゃないかという気がする

slog.Handlerの作り方とか

## Filesytem abstraction

`fs.FS`はreadonly

`embed.FS`でpermission bitsが消えることを述べる

- afero: https://github.com/spf13/afero
- hackpadfs: https://github.com/hack-pad/hackpadfs
- go-billy: https://github.com/go-git/go-billy

## Cli application

- stdのflagを使うシンプルなアプリ
- https://github.com/spf13/cobra + https://github.com/spf13/viper を使ったアプリ
  - docker, docker compose, kubernetesなどで使われている。

## Enum

は存在しないが、似たようなことはiotaとtypeでてきる

```go
type Foo string

const (
  FooVariantA = "a"
  FooVariantB = "b"
)
```

体感上、変数はtype名でprefixしておくのが吉

## New関数

パッケージ内でメインとなる関心事を表すstructがあり、それのフィールドがexportされない場合は初期化のためのNew関数を定義する

## Must関数、Must prefix

ある関数がエラーしうるとき、（あとは追記）

どうまとめよう？

## io.Readerを実装する

io.Readerを実装してみたりする

ReaderFromとかWriterToについて触れる

## fmt.Formatterを実装する

カスタムエラータイプ周りの話題で

## よく使うライブラリ

- samber/lo
- clockwork
- mapstructure / copystructure

## レシピとか

- sync.OnceFunc

## Data marshaling / unmarshalng

encoding/jsonとencoding/xml、encoding/binaryぐらいの軽いチュートリアル

DisallowUnknownFieldは使わない！（オンラインでバージョンアップされるアプリでは）

## http.Clientをいじろう

- なにかのclientを実装するときは\*(http.Client)を引数で受け取ろう
  - functional option patternで渡すほうが良いこともある
    - functional option patternは真面目に実装しようとすると面倒な時があるので、code generatorを実装しようかなと思っている（したいから）。その場合この記事はそちらが完成するまでpendedのままになるかも…
- http.Roundtripper interface / http.Transport
- Example: cookie jarをセット
- Example: 送受信内容をトラップ
- Example: 名前解決部分を[github.com/miekg/dns](https://github.com/miekg/dns)に差し替え
  exampleはやったことあるやつだけにとどめてある

## パッケージ内だけで実装できるinterface

unexported methodをinterfaceにいれる

## モジュール内でのみ実装できるinterface

exported methodの引数か、返り値をinternal packageで定義される型にする
`type PrivateOnly [0]func()`などにすると名前でわかるかも

## 渡したcontext.Contextは必ずcancelしよう

```go
go func () {
  <-ctx.Done()
}()
```

みたいなコード書かれることがある（筆者は書く）。ctxがキャンセルされることがない可能性についてはctxを使う側が考慮すべきだけど、考慮が甘いことがあるので、呼び出し側もctxに用がなくなったらcancelしておいたほうがいい

## reflectionのはなし

encoding/jsonおよびencoding/json/v2を最も典型的な利用例として紹介する

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
