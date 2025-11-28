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

## Docker / podmanなどによるビルド

[プロジェクトを始めるまで](https://zenn.dev/ngicks/articles/go-basics-revisited-starting-projects)で基本的なビルド方法については述べました。
しかし現代においてはアプリケーションは何かしらのコンテナとしてデプロイすることが普通であると思われるので、この記事では

- そもそも[Docker] / コンテナ / イメージ / `Dockerfile`(`Containerfile`)とは何ぞやという簡単な説明
- [Docker] / [podman]\([podman-static]\)のインストール手順
- [Docker] / [podman]を用いた`Go`アプリケーションのビルド方法
  - cooporate proxy下対応
  - 最大限キャッシュを効かせる
  - 半自動的に`Go`の最新パッチバージョンを使用

について述べます。

[Docker] / [podman]はいわゆるコンテナランタイムです。
コンテナは、アプリとその依存関係をパッケージ化し、それを隔離した環境で動作させる一連の仕組みだと思っておけばよいです。

コンテナは、一般的なアプリをPCにインストールしたときに起こる以下のような問題を回避することができます

- アプリが保存するデータや設定ファイルの位置がほかのアプリと衝突する
- アプリが必要なライブラリがいろいろあって全部入れないといけない
- アプリ同士で必要なライブラリのバージョンが異なっており衝突する
- 同じアプリを複数動かそうとすると一時ファイルの位置や、ネットワークのポートが衝突していまう

こうしたコンテナを動作させるのがコンテナランタイムです。

## 対象読者/前提知識

- 会社の同僚
- 今まで[Go]を使ってこなかった
- dockerはいくらか触ったことがある
- 高校レベルの英語読解能力

## 環境

win11のwsl2インスタンス内で動作させます。[Docker Desktop](https://www.docker.com/products/docker-desktop/)をインストールした場合/PCに直接Linuxをインストールした場合でも基本的に同様になると思います。

```
$ wsl --version
WSL バージョン: 2.6.1.0
カーネル バージョン: 6.6.87.2-1
WSLg バージョン: 1.0.66
...
```

distroはUbuntu 24.04LTSです。説明は暗黙的に`Ubuntu`を前提とします。

```
$ cat /etc/*-release
...
DISTRIB_DESCRIPTION="Ubuntu 24.04.3 LTS"
...
```

ランタイムにはpodmanを用います

```
$ podman --version
podman version 5.7.0
```

## そもそも[Docker] / コンテナ / イメージ / `Dockerfile`ってなに？

コンテナにまつわる用語として[Docker] / コンテナ / イメージ / `Dockerfile`などがよく聞かれると思います。
これらが一体何なのかという話をざっくりこことでしておきます。
全体の構図が見えることのみを目指すので、厳密でなかったり詳細でなかったりしますが、記事の趣旨からして詳細に立ち入ることはしません。

### コンテナ？

コンテナとは、アプリケーションとその依存関係と、アプリの起動方法などををひとまとめにして、隔離環境で実行できるようにしたものを、実行したプロセスなどをさしてコンテナと言います。アプリだと思ってくれてもいいかもしれません。

例えばPCに直接[PlantUML](https://plantuml.com/)を入れてサーバーとして利用できるようにしたいとします。
その場合、下記のようなことを行う必要があります。

- `Java`実行環境のインストール
- `tomcat`などのセットアップ
- `Graphviz`などの依存先をインストール
- `plantuml`のソースをコピーしビルドする
- `systemd`のunitファイルなどを書いてサービス永続化
- ポート/一時ファイルの位置などが他と被らないようにマネージ

参考:

- [PlantUMLクイックスタートガイド](https://plantuml.com/starting)
- [PlantUML Serverを仮想マシン上にセットアップする (Tomcatサーバー編)](https://zenn.dev/h1d3mun3/articles/9ab6f0e3d10195)

まあまあいろいろやりますね。
厄介なのがほかのサービスに必要な`Java`とかのバージョンが異なる場合です。工夫すれば異なる複数のバージョンを同時に動かすこともできますが、工夫が必要であることは覚えておいてください。

これが[Docker] / [podman]を用いると

```
podman container run -d -p 8881:8080 docker.io/plantuml/plantuml-server:tomcat-v1.2025.10
```

で済みます。(dockerの場合、`podman`の部分を`docker`に取り換えてください)
`docker.io/plantuml/plantuml-server:tomcat-v1.2025.10`の部分が、こういったサーバー用途のアプリとして配布されたもの(イメージと呼ぶ)をさします。
コンテナエコシステムだとHTTPなどを通じてアクセスされるサーバーとしてアプリを配布することが多いため、ありもののパッケージとしてすでにサーバーとして動くように調節されていることが多いわけです。

同じアプリを設定を変えながら複数建てたい場合、

```
podman container run -d -p 8882:8080 --env PLANTUML_LIMIT_SIZE=8192 docker.io/plantuml/plantuml-server:tomcat-v1.2025.10
```

のように、同じようにコマンドを実行すればよいです。

このアプリがシステム再起動を挟んでも起動してほしい場合は`--restart=always`を付け足します。

```
podman container run -d -p 8883:8080 --env PLANTUML_LIMIT_SIZE=8192 --restart=always docker.io/plantuml/plantuml-server:tomcat-v1.2025.10
```

つまり、コンテナの特記すべき性質は

- スケール性: アプリ単位の隔離であることで必要に応じて起動させるインスタンスの数を増減できる
  - 基本的に1プロセス-1コンテナの粒度で隔離します。
- イメージ配布システム: (VM仮想化に比べて)簡単に利用できるありもののパッケージ(イメージと呼ばれる)が広く配布されていること
- 管理容易: アプリとその依存性は隔離されているのでホストやほかのアプリを汚すことがないこと

などがあげられます。

### よく言われる「VM仮想化と違ってゲストカーネルがない」というのは厳密には誤り

コンテナを調べていると、よく「KVM/Hypervisor仮想化と違ってゲストOSがないので軽量」という言い回しがされるように思います

参考:

- [コンテナ型の仮想化を基礎から学ぶ！従来の技術との違いやメリットを解説 ](https://www.ctc-g.co.jp/keys/blog/detail/containerized-virtualization)
- [コンテナ技術を他の仮想化技術と比較しながら整理](https://qiita.com/n0mura/items/b57800356eb6c59be7d9)
- [サーバ仮想化技術とコンテナ技術の違い](https://jpn.nec.com/cloud/service/container/comparison.html)
- [コンテナ型仮想化とは、クラウド展開に便利な進化中の仮想化技術](https://insights-jp.arcserve.com/container-virtualization)
- [コンテナ化と仮想化：7つの技術的違い](https://www.trianz.com/ja/insights/containerization-vs-virtualization)

実際[dockerはlinux kernel機能のnamespaceを使用して隔離環境を作成する](https://docs.docker.com/get-started/docker-overview/#the-underlying-technology)ため`docker`に関しては普通はこの言説のとおりだと思います。
しかし[OCI Runtime Space](https://github.com/opencontainers/runtime-spec/blob/main/spec.md)にはどのような方法で隔離環境を作成するかには規定がなく、実際に[kata-container](https://katacontainers.io/)や[windows containerのrunhcs](https://learn.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/containerd#runhcs)はVMを使ってコンテナを動作させます。

`kata-container`は設定をいじれば`docker`からも利用可能なコンポーネントです。

この事実からゲストOSがないことがコンテナの本質ではなく、前節で上げたようなアプリ配布エコシステムの成立と、1プロセス-1コンテナの粒度で隔離することが重要な性質であることを述べておきます。

### イメージ？

コンテナのひな型のことをイメージと呼びます。
アプリとその依存関係と、アプリの起動設定などをまとめたもので、これをコンテナランタイムが読み込んで隔離環境を作成し、実行するとコンテナとなります。

参考:

- [What is Docker? > Docker architecture > Docker objects > images](https://docs.docker.com/get-started/docker-overview/#images)
- [OCI Image Spec](https://github.com/opencontainers/image-spec/blob/main/spec.md)

### Docker / podman ?

コンテナランタイムです。

- イメージをpullしたり、buildしたり、
- コンテナを作成/実行したり
- イメージ/コンテナの作成/停止/実行をしたり

するものです。

[Docker]はこの分野の草分けです。`Docker, Inc.`開発。開発者ツールとしてはかなりポピュラーだと思います。
[podman]は`Docker`より後発のランタイム。`Red Hat`開発。rootless by default, daemonlessなどいろいろ進んだ機能が多い。手元で動かすなら`docker`より扱いが楽なこともしばしば。

version 1.0.0のリリース時期は`docker`のほうが速い:

- `Docker`: [2014-06](https://docs.docker.com/engine/release-notes/prior-releases/#100-2014-06-09)
- `Podman`: [2019-01](https://github.com/containers/podman/releases/tag/v1.0.0)

### Dockerfile(Containerfile)?

`Dockerfile`はイメージをビルドする際のレシピとなるファイルです。リファレンスは下記。

https://docs.docker.com/reference/dockerfile/

`docker buildx build -f Dockerfile -t ${repo}:${tag} .`で指定すると`${repo}:${tag}`でタグ付けされた(名づけられた)イメージがビルドされます。

`Containerfile`は`Dockerfile`と全く同じものですがコンテナランタイムがdocker外にもたくさん現れたため`Docker`という固有名詞を排したい意図で`podman`のデフォルト参照先となっているようです([参考](https://github.com/containers/buildah/discussions/3170))

## Docker / podman(-static)のインストール方法

### Docker

以下の2つが代表的なインストール方法かと思います

- `Docker Desktop`や[Rancher Desktop](https://rancherdesktop.io/)を利用する方法
- [Install Docker Engine](https://docs.docker.com/engine/install/)\: 公式の手続きに基づいてインストールする方法

`Docker Desktop`は[従業員250人以上もしくは年間売り上げ＄10 million(≒15.6億円)で支払い義務が発生する](https://docs.docker.com/subscription/desktop-license/)ライセンス形態であることに注意です。

後者についてのみ説明します。

```bash
#!/bin/bash

set -Cue

# Add Docker's official GPG key:
sudo -E apt-get update
sudo -E apt-get install -y ca-certificates curl

sudo -E install -m 0755 -d /etc/apt/keyrings
sudo -E curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo -E chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo -E apt-get update
sudo -E apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

`rootless`で実行したい場合は追加で下記も実施します。開発環境のdockerとしては`rootless`にしておくほうがお勧めです。

https://docs.docker.com/engine/security/rootless/

### podman

[podman-static]を利用してビルドします。

[podman-static]は`static`(=動的にロードされるライブラリがない=ホスト環境に対する依存性が低い)に`podman`をビルドするためのスクリプト集です。

repositoryをcloneして下記を実行し、`./build/asset/podman-linux-amd64`以下のファイルを適当なところにコピーしたら完了です。

実行する前に、スクリプトの内容はよく読んでおきましょう: https://github.com/mgoltzsche/podman-static/blob/master/Dockerfile

```
  sudo make
  sudo make singlearch-tar
```

(`sudo`がつくのは`docker`コマンドの実行のために必要なだけで、`rootless`にしている場合などは`sudo`なしで実行する)

筆者は`~/.local/share/podman`以下にビルド成果物をまとめておきたかったためさらに追加のビルドスクリプトを組んでいます。

以下をcloneして下記のスクリプトを実行します。[deno]が必要です。

https://github.com/ngicks/dotfiles

```
build/podman-static/build.sh
build/podman-static/install.sh
```

(もちろん実行する前に中身はよく読んでください)

`install.sh`完了後、下記を`.bashrc`などから呼び出すと`podman`コマンドに`PATH`が通ります。

```
. ~/.config/containers/path.sh
```

## goをビルドするDockerfile example

以下に`Go`をstatic binaryにビルドする`Dockerfile`の例を示します。
`Dockerfile`をまず述べ、各変数とbuildkitのマウントの各パラメータの意味を述べ、ビルドコマンドなどをその後に述べます。

- 企業プロキシの裏にいてもビルドできるようにします。
- private repository管理のgo moduleがあってもビルドできるようにします。
- ほぼすべてがキャッシュに乗るので初回以降はビルド時間のほとんどがdockerのメタデータ解決時間です。
- `apt-get`を使いますが、この部分はキャッシュしません。distro/バージョンで差が大きそうな気がしてます。
  - キャッシュしたい人は[misskeyのDockerfileのここ](https://github.com/misskey-dev/misskey/blob/43cccaaee9be42fab38eaa9ca04bb5e55b5d8db7/Dockerfile#L9-L15)とかが参考になるかも

筆者はおおむねこれでうまくいっていますが、何かがあれば、static binaryに実はならないとか、そういった問題点があるかもしれないので、読者の環境に向けてカスタマイズする必要があるのは当然述べておくべきでしょう。

コードはここに置いてあります: https://github.com/ngicks/go-basics-example/tree/main/dockerfile

### 通常版Dockerfile(Containerfile)

```dockerfile
# syntax=docker/dockerfile:1

ARG TAG_GOVER="1.25.0"
ARG TAG_DISTRO="bookworm"

FROM docker.io/library/golang:${TAG_GOVER}-${TAG_DISTRO} AS builder

ARG CGO_ENABLED="0"
ARG GOCACHE="/root/.cache/go-build"
ARG GOENV="/root/.config/go/env"
ARG GOPATH="/go"
ARG GOPRIVATE=""

ARG SSH_HOSTS="github.com,"
ARG MAIN_PKG_PATH="."

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
<<EOF
    rm -f /etc/apt/apt.conf.d/docker-clean
    echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
    apt-get update
    apt-get install -yqq --no-install-recommends git-lfs openssh-client
EOF

RUN <<EOF
    mkdir -p -m 0700 ~/.ssh
    for item in $(echo $SSH_HOSTS | tr ',' '\n' ); do
      if [ ! -z ${item} ]; then
        git config --global url."ssh://git@${item}".insteadOf https://${item}
        ssh-keyscan ${item} >> ~/.ssh/known_hosts
      fi
    done
EOF

WORKDIR /app/src

# https://docs.docker.com/build/building/context/#dockerignore-files
# COPY . .

RUN --mount=type=ssh \
    --mount=type=secret,id=goenv,target=/root/.config/go/env \
    --mount=type=cache,target=/go \
    --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=bind,target=/app/src \
<<EOF
    go mod download
    # go generate ./...
    go build -o ../bin ${MAIN_PKG_PATH}
EOF

WORKDIR /app

# arm64
FROM gcr.io/distroless/static-debian12@sha256:ed92139a33080a51ac2e0607c781a67fb3facf2e6b3b04a2238703d8bcf39c40
# amd64
# FROM gcr.io/distroless/static-debian12@sha256:6ceafbc2a9c566d66448fb1d5381dede2b29200d1916e03f5238a1c437e7d9ea

COPY --from=builder /app/bin /app/bin

ENTRYPOINT [ "/app/bin" ]
```

```dockerfile
# syntax=docker/dockerfile:1

ARG TAG_GOVER="1.25.0"
ARG TAG_DISTRO="bookworm"

FROM docker.io/library/golang:${TAG_GOVER}-${TAG_DISTRO} AS builder

ARG HTTP_PROXY
ARG HTTPS_PROXY

ARG SSL_CERT_FILE="/etc/ssl/certs/ca-certificates.crt"
ARG NODE_EXTRA_CA_CERTS="/etc/ssl/certs/ca-certificates.crt"
ARG DENO_CERT="/etc/ssl/certs/ca-certificates.crt"

ARG GOPATH="/go"
ARG GOCACHE="/root/.cache/go-build"
ARG GOPRIVATE=""
ARG CGO_ENABLED="0"
ARG MAIN_PKG_PATH="."

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    --mount=type=secret,id=certs,target=/etc/ssl/certs/ca-certificates.crt \
<<EOF
    rm -f /etc/apt/apt.conf.d/docker-clean
    echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
    apt-get update
    apt-get install -yqq --no-install-recommends git-lfs openssh-client
EOF

RUN <<EOF
    git config --global url."ssh://git@github.com".insteadOf https://github.com
    mkdir -p -m 0700 ~/.ssh
    ssh-keyscan github.com >> ~/.ssh/known_hosts
EOF

WORKDIR /app/src

# https://docs.docker.com/build/building/context/#dockerignore-files
# COPY . .

RUN --mount=type=secret,id=certs,target=/etc/ssl/certs/ca-certificates.crt\
    --mount=type=ssh \
    --mount=type=secret,id=goenv,target=/root/.config/go/env \
    --mount=type=cache,target=/go \
    --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=bind,target=/app/src \
<<EOF
    go mod download
    # go generate ./...
    go build -o ../bin ${MAIN_PKG_PATH}
EOF

WORKDIR /app

FROM gcr.io/distroless/static-debian12@sha256:ed92139a33080a51ac2e0607c781a67fb3facf2e6b3b04a2238703d8bcf39c40
# FROM gcr.io/distroless/static-debian12@sha256:6ceafbc2a9c566d66448fb1d5381dede2b29200d1916e03f5238a1c437e7d9ea

COPY --from=builder /app/bin /app/bin

ENTRYPOINT [ "/app/bin" ]

```

### 企業Proxy下版

```dockerfile
# syntax=docker/dockerfile:1

ARG TAG_GOVER="1.25.0"
ARG TAG_DISTRO="bookworm"

FROM docker.io/library/golang:${TAG_GOVER}-${TAG_DISTRO} AS builder

ARG CGO_ENABLED="0"
ARG GOCACHE="/root/.cache/go-build"
ARG GOENV="/root/.config/go/env"
ARG GOPATH="/go"
ARG GOPRIVATE=""

ARG MAIN_PKG_PATH="."

ARG HTTP_PROXY
ARG HTTPS_PROXY=${HTTP_PROXY}
ARG NO_PROXY
ARG http_proxy=${HTTP_PROXY}
ARG https_proxy=${HTTP_PROXY}
ARG no_proxy=${NO_PROXY}

# for curl, etc.
ARG SSL_CERT_FILE="/etc/ssl/certs/ca-certificates.crt"
ARG NODE_EXTRA_CA_CERTS="/etc/ssl/certs/ca-certificates.crt"
ARG DENO_CERT="/etc/ssl/certs/ca-certificates.crt"

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    --mount=type=secret,id=certs,target=/etc/ssl/certs/ca-certificates.crt \
<<EOF
    rm -f /etc/apt/apt.conf.d/docker-clean
    echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
    apt-get update
    apt-get install -yqq --no-install-recommends git-lfs
EOF

WORKDIR /app/src

RUN --mount=type=secret,id=netrc,target=/root/.netrc \
    --mount=type=secret,id=goenv,target=/root/.config/go/env \
    --mount=type=secret,id=certs,target=/etc/ssl/certs/ca-certificates.crt \
    --mount=type=cache,target=/go \
    --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=bind,target=/app/src \
<<EOF
    go mod download
    # go generate ./...
    go build -o ../bin ${MAIN_PKG_PATH}
EOF

WORKDIR /app

# arm64
FROM gcr.io/distroless/static-debian12@sha256:ed92139a33080a51ac2e0607c781a67fb3facf2e6b3b04a2238703d8bcf39c40
# amd64
# FROM gcr.io/distroless/static-debian12@sha256:6ceafbc2a9c566d66448fb1d5381dede2b29200d1916e03f5238a1c437e7d9ea

COPY --from=builder /app/bin /app/bin

ENTRYPOINT [ "/app/bin" ]
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
[podman]: https://podman.io/
[podman-static]: https://github.com/mgoltzsche/podman-static
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
