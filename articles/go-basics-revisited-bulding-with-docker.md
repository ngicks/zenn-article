---
title: "Goのプラクティスまとめ: dockerによるビルド"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## Goのプラクティスまとめ: dockerによるビルド

筆者が`Go`を使い始めた時に分からなくて困ったこととか最初から知りたかったようなことを色々まとめる一連の記事です。

以前書いた記事のrevisited版です。話の粒度を細かくしてあとから記事を差し込みやすくします。

他の記事へのリンク集

- (まだ)~~[今はこうやる集](https://zenn.dev/ngicks/articles/go-basics-revisited-updated-practices)~~
- [プロジェクトを始めるまで](https://zenn.dev/ngicks/articles/go-basics-revisited-starting-projects)
- `dockerによるビルド`: ここ
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

## Dockerfile

`Dockerfile`のexample.

[Docker]を使うとアプリをパッケージ化して送り込んだりするのが楽になります。
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
