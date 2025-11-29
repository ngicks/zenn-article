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
  - おまけで`multi-arch build`に対応

について述べます。

[Docker] / [podman]はいわゆるコンテナランタイムです。
コンテナは、アプリとその依存関係をパッケージ化し、それを隔離した環境で動作させる一連の仕組みだと思っておけばよいです。

コンテナは、一般的なアプリをPCにインストールしたときに起こる以下のような問題を回避することができます

- アプリが保存するデータや設定ファイルの位置がほかのアプリと衝突する
- アプリが必要なライブラリがいろいろあって全部入れないといけない
- アプリ同士で必要なライブラリのバージョンが異なっており衝突する
- 同じアプリを複数動かそうとすると一時ファイルの位置や、ネットワークのポートが衝突していまう

こうしたコンテナを動作させるのがコンテナランタイムです。

今回の記事は別に`Go`にばかり関係するというわけでもないですね。

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

### 概説

![concept-of-container](/images/go-basics-revisited-bulding-with-docker/concept-of-container.webp)

コンテナは「そのアプリだけ入ったLinuxシステムみたいなやつ」を隔離した環境で動かすアプリの動作方式などのことを言います。

コンテナ内ではファイルシステムを好きにいじっていいですし、そのアプリ専用のライブラリとかが全部あるべき場所にある状態にできます。`samba`に対する`/etc/smb.conf`みたいに、決まりきったパスの設定ファイルを読み込むアプリがある場合、設定ファイルのパスを変更できなかったらそのアプリは同時に1つしか動かすことができませんが、コンテナではお互い隔離されるため、同じパス(`/etc/smb.conf`)でもコンテナ同士では異なったファイルを読み込ませることができます。
また、ネットワークのポートについても同様で、コンテナ内のポートは、ホスト側に公開されるときに別のポートに割り当てることができます。

![objects-relation](/images/go-basics-revisited-bulding-with-docker/objects-relation.webp)

コンテナはイメージというコンテナのテンプレートのようなものから作成されます。イメージは`Dockerfile`(`Containerfile`)という、簡単なsyntaxで構成されたテキストファイルをもとに作成することができます。

`Dockerfile`(`Containerfile`)で、`apt-get`でライブラリやアプリを導入したり、`go build`などでアプリケーションをビルドしたりして、「アプリ」と「アプリが動作する環境」を作るためのビルドスクリプトを記述します。
ちなみに`Dockerfile`と`Containerfile`は仕様上全く同じものです。コンテナエコシステムが広がるにしたがって`Docker`以外のものが増えて来たので固有名詞である`docker-`というのを消そうという動きがあります。

`podman image build`/`docker image build`などのコマンドで`Dockerfile`(`Containerfile`)をビルドしてイメージを作成します。
イメージはそのような方法で作られた、「アプリを含んだファイルシステム」と、環境変数・エントリーポイント(=コンテナ実行時に実行されるアプリ)などを含んだ設定ファイルからなります。

コンテナは、イメージをもとに作成される、「実行可能なインスタンス」です。
コンテナの隔離環境は、中で動いているアプリから見ると普通のLinuxみたいに見えて、ファイルシステムに書き込みを行ったりできます。一方で、イメージはリードオンリーで不変ですので、イメージからコンテナを作るには書き込みができる領域を確保する必要があります。また、イメージでは決められない「どのホストのポートをコンテナのどのポートに公開するか」とか、「どのパスにどのボリュームをマウントするか」などのコンテナ固有の情報が含まれます。

### 利点

コンテナシステムの利点は以下などがあると考えられます。

- スケール性:
  - アプリ単位の隔離環境を用意するため、同じアプリを容易に複数動作させられる。
  - 依存関係をすべて含むため、複数のホストマシンで分散して動作させられる。
  - アプリ単位、つまり1プロセス-1コンテナの粒度で分割するのがふつうであるため、必要な部分だけを稼働数を増減することが容易
- 配布の容易性:
  - イメージをファイルとして保存するためのフォーマットが規定されたことで、容易に共有が可能
- 管理の容易性:
  - 隔離されていることでホスト環境を汚さない(=アンインストール時に消し忘れそうなものがない)

コンテナを調べていると、よく「KVM/Hypervisor仮想化と違ってゲストOSがないので軽量」という言い回しがされるように思います

参考:

- [コンテナ型の仮想化を基礎から学ぶ！従来の技術との違いやメリットを解説 ](https://www.ctc-g.co.jp/keys/blog/detail/containerized-virtualization)
- [コンテナ技術を他の仮想化技術と比較しながら整理](https://qiita.com/n0mura/items/b57800356eb6c59be7d9)
- [サーバ仮想化技術とコンテナ技術の違い](https://jpn.nec.com/cloud/service/container/comparison.html)
- [コンテナ型仮想化とは、クラウド展開に便利な進化中の仮想化技術](https://insights-jp.arcserve.com/container-virtualization)
- [コンテナ化と仮想化：7つの技術的違い](https://www.trianz.com/ja/insights/containerization-vs-virtualization)

実際[dockerはlinux kernel機能のnamespaceを使用して隔離環境を作成する](https://docs.docker.com/get-started/docker-overview/#the-underlying-technology)ため`docker`に関しては普通はこの言説のとおりだと思います。
ただし別段その方式に限らなければならないわけではなく、実際に`docker`から利用可能なコンポーネントである[kata-container](https://katacontainers.io/)(`QEMU/KVM`や`Firecracker`など)や[windows containerのrunhcs](https://learn.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/containerd#runhcs)(`Hyper-V`)などはVMを使ってコンテナの隔離環境を作成します。
[OCI Runtime Spec](https://github.com/opencontainers/runtime-spec/blob/main/spec.md)上でも隔離環境の作成方法に指定はありません。

この事実からゲストOSがないことがコンテナの本質ではなく、前述のアプリ配布エコシステムの成立と、1プロセス-1コンテナの粒度で隔離することが目指したいものだと言えます。

...という説明から前述の「そのアプリだけ入ったLinuxシステムみたいなやつ」という説明も正確でないことがわかります(windowsやfreebsdのcontainerがありますから)。ですが、筆者の観測する限り大抵コンテナと言ったらLinuxです。

### Docker / podman ?

コンテナランタイムです。

- イメージをpullしたり、buildしたり、
- コンテナを作成/実行したり
- イメージ/コンテナの作成/停止/実行をしたり

するものです。

[Docker]はこの分野の草分け的存在です。`Docker, Inc.`開発。開発者ツールとしてはかなりポピュラーだと思います。
[podman]は`Docker`より後発のランタイム。`Red Hat`開発。rootless by default, daemonlessなどいろいろ進んだ機能が多い。手元で動かすなら`docker`より扱いが楽なこともしばしば。

全く違うところが作っているので中身の作りは違いますが、コマンドとしては互換性があります。

version 1.0.0のリリース時期は`docker`のほうが速い:

- `Docker`: [2014-06](https://docs.docker.com/engine/release-notes/prior-releases/#100-2014-06-09)
- `Podman`: [2019-01](https://github.com/containers/podman/releases/tag/v1.0.0)

## Reference

これ以上の詳細な情報は公式的なドキュメントへお進みください。

- Docker Guide: https://docs.docker.com/guides
- Dockerfile reference: https://docs.docker.com/reference/dockerfile
- Docker Cli Reference: https://docs.docker.com/reference/cli/docker

(`podman`はcliは`Docker`互換なのでreferenceも`Docker`のものを見たらいい。違いが出るところまで踏み込みません)

## Docker / podman(-static)のインストール方法

`Docker`と`podman`のインストール方法について述べます。`podman`のほうは[podman-static]を用いるため、やや変則的な方法になります。
[podman-static]のビルドには`Docker`が必要なことに注意してください。

### Docker

以下の2つが代表的なインストール方法かと思います

- `Docker Desktop`や[Rancher Desktop](https://rancherdesktop.io/)を利用する方法
- [Install Docker Engine](https://docs.docker.com/engine/install/): 公式の手続きに基づいてインストールする方法

ただし`Docker Desktop`は[従業員250人以上もしくは年間売り上げ$10 million(≒15.6億円)で有料ライセンスが必要となる](https://docs.docker.com/subscription/desktop-license/)ことに注意です。
`Rancher Desktop`はほぼ`Docker Desktop`と同じようなことをするOSSでこちらは`Apache-2.0` licenseです。起動が遅かったりするのでwindows側から`docker`コマンドをたたきたいとかでない限りはおすすめしません。

[Install Docker Engine](https://docs.docker.com/engine/install/)についてのみ説明します。

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

[podman-static]は`static`(=動的にロードされるライブラリがない=ホスト環境に対する依存性が低い)に`podman`をビルドするためのスクリプト集です。ちなみに`docker`コマンドが必要です。古いバージョンの`podman`を使っても(`alias docker=podman`)行けると思います。

repositoryをcloneして下記を実行し、`./build/asset/podman-linux-amd64`以下のファイルを適当なところにコピーしたら完了です。

実行する前に、スクリプトの内容はよく読んでおきましょう: https://github.com/mgoltzsche/podman-static/blob/master/Dockerfile

```
  sudo make
  sudo make singlearch-tar
```

`./build/asset/podman-linux-amd64`以下を`/`以下に構造を保ったままコピーしていけば標準的なパスに`podman`が入るような状態になるようです。
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

## GoをビルドするDockerfile example

以下に`Go`をstatic binaryにビルドする`Dockerfile`の例を示します。
`Dockerfile`をまず述べ、各変数とbuildkitのマウントの各パラメータの意味を述べ、ビルドコマンドなどをその後に述べます。

- 通常版・企業プロキシ下版の２バージョンを説明します。
- 双方でprivate repository管理のgo moduleがあってもビルドできるようにします。
- ほぼすべてがキャッシュに乗るので初回以降はほとんど時間がかかりません。
- `apt`以外のパッケージマネージャには対応できていません。筆者がそれら(`apk`や`pacman`など)を使うことがあったら調べて追記します。
- 現在のプロジェクトの`go.mod`を解析して得られたGo versionの最新のパッチバージョンでビルドできるような半自動的な仕組みを考えます。
  - つまり、`go.mod`の記載が`go1.24.1`の場合、`go1.24.10`でビルドする、みたいな感じです。

コードはここに置いてあります: https://github.com/ngicks/go-example-basics-revisited/tree/main/building-with-docker

通常版・企業プロキシ下版両方を示してから各パートとポイントについて説明し、最新のパッチバージョンを取得する方法を含んだビルドスクリプトを示します。
最後におまけとしてmulti-archビルド(`amd64`(普通のPCなど)で`arm64`(Raspberry Piなど)むけのimageをビルドすること)をする方法を示します。

### Dockerfile(Containerfile)の例

#### 通常版

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

FROM gcr.io/distroless/static-debian12@sha256:6ceafbc2a9c566d66448fb1d5381dede2b29200d1916e03f5238a1c437e7d9ea

COPY --from=builder /app/bin /app/bin

ENTRYPOINT [ "/app/bin" ]
```

#### 企業プロキシ下版

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

FROM gcr.io/distroless/static-debian12@sha256:6ceafbc2a9c566d66448fb1d5381dede2b29200d1916e03f5238a1c437e7d9ea

COPY --from=builder /app/bin /app/bin

ENTRYPOINT [ "/app/bin" ]
```

### Go固有のポイント

- `git-lfs`を入れよう: [以前の記事のこの部分](https://zenn.dev/ngicks/articles/go-basics-revisited-starting-projects#git-lfs%E3%82%92%E5%B0%8E%E5%85%A5%E3%81%97%E3%81%A6%E3%81%84%E3%82%8B%E5%A0%B4%E5%90%88%E3%81%AF%E3%81%99%E3%81%B9%E3%81%A6%E3%81%AE%E7%92%B0%E5%A2%83%E3%81%A7git-lfs%E3%82%92%E4%BD%BF%E3%81%86%E3%82%88%E3%81%86%E3%81%AB%E6%B0%97%E3%82%92%E4%BB%98%E3%81%91%E3%82%8B)でも書きましたが、`git-lfs`はとりあえずすべての環境に入れておいたほうがいいです。
  - private gitからGo moduleを落としてくるとき、`git-lfs`の有無で`git fetch`した結果が変わるためmodule sumが食い違ってしまうためです。
- `CGO`使わない場合は`CGO_ENABLED=0`に: static binaryを出力すると、`glibc`などのバージョン差を気にしなくてよくなる
  - コンテナ内で使うならば特にバージョンずれるとかない気がするので常に`1`でも問題ない気はします。
  - `CGO`必要になる場合、例えば[github.com/mattn/go-sqlite3](https://github.com/mattn/go-sqlite3)を使う場合は明示的に`1`にします。

### Goに関係するパラメータ群とその意味

| 種           | 名前          | 変更必要？ | 説明                                                                             |
| :----------- | :------------ | :--------- | :------------------------------------------------------------------------------- |
| ARG          | TAG_GOVER     |            | ビルドに使う`Go`のバージョン。後述するスクリプトで自動的に決定する               |
| ARG          | TAG_DISTRO    | yes        | ビルドに使う`Go`イメージのディストロ。debianのバージョンが進むたび名前が変わる   |
| ARG          | CGO_ENABLED   | yes        | `CGO`が含まれないとき`0`に設定するとstaticなバイナリになる                       |
| ARG          | GOCACHE       |            | ビルドキャッシュの位置                                                           |
| ARG          | GOENV         |            | `go env`で読み書きできるファイルの位置                                           |
| ARG          | GOPATH        |            | モジュールキャッシュなどの位置                                                   |
| ARG          | GOPRIVATE     | yes        | private registryのpath prefix. host以降(e.g. `github.com/ngicks/go-playground`). |
| ARG          | MAIN_PKG_PATH | yes        | build context内のビルド対象へのパス                                              |
| secret mount | goenv         |            | ホスト側の`go env`をマウントするためのid.                                        |

`GOPRIVATE`は`go env`に書き込むほどじゃない実験用のレジストリとか入れるために`ARG`にしてありますが、普通はいらないので消してしまったほうがいいかも。

### ポイント1: `# syntax=docker/dockerfile:1`を先頭につけとく

https://docs.docker.com/build/buildkit/frontend/

追加の文法(`<<EOF`のヒアドキュメントなど)を使えるようにするためのインストラクションです。

`docker build --build-arg BUILDKIT_SYNTAX=${syntax}`によって設定できるとも書いてあります。
環境によってこの変数が自動的に設定されていたりされていなかったりすることがあってややこしかったのでとりあえず`# syntax=docker/dockerfile:1`をファイル先頭に書いておくことを推奨します。

- `syntax=docker/dockerfile:1`なら`1.x.y`の範囲
- `syntax=docker/dockerfile:1.20`なら`1.20.x`の範囲

のような感じで、バージョン範囲の指定も行えるとあります。
ただ、筆者の体験する限りバージョンが上がることで変更された挙動によってうまく動かなくなったことはなかったためとりあえず`:1`の指定の仕方を推奨しておきます。トラブルが起きたらより狭い固定をしましょう。

### ポイント2: `docker.io/library`を省略しない

例では以下のように書いています。

```dockerfile
FROM docker.io/library/golang:${TAG_GOVER}-${TAG_DISTRO} AS builder
```

しかし実際には`docker.io/library`の部分を省略した例のほうがよく見るかもしれません。

例えば以下のコマンドを実行してみてください

```
$ docker image pull golang:1.25.4-bookworm
```

別に成功しますよね

このイメージは[docker hub](https://hub.docker.com)でホストされています。以下のページです。

https://hub.docker.com/_/golang

`Docker`の場合、imageの名前は以下のようになっています

https://docs.docker.com/reference/cli/docker/image/tag/#description

> [HOST[:PORT]/]NAMESPACE/REPOSITORY[:TAG]

記述のとおり、`HOST`,`NAMESPACE`部分は省略可能でデフォルトはそれぞれ`docker.io`、`library`となります。

これの何が困るかというと、`Docker`以外のランタイムが大量に出てきた結果、この補完の処理がうまく動かなったり([github.com/containerd/nerdctl#4468](https://github.com/containerd/nerdctl/issues/4468))、そもそも補完しないツールが出てきていることです(`podman`はどのレジストリに補完するかの選択が出る)。

`docker.io/library`は省略しないようにしましょう。`a`なら`docker.io/library/a`、 `a/b`なら`docker.io/a/b`、もとから`a/b/c`ならそのままで。

- `node` -> `docker.io/library/node`
- `astral/uv` -> `docker.io/astral/uv`
- `gcr.io/distroless/static-debian12` -> (そのまま)

### ポイント3: multi-stage buildを使う

https://docs.docker.com/build/building/multi-stage/

multi-stage buildは`Dockerfile`に`FROM`が複数あることをさします。

`FROM`から始まる一連のビルドステップをステージと呼ぶようです。(多分build-contextでcontextという語が使われちゃってるからstageと呼んでいるのだと思う。根拠なし。)
`FROM`の後に、さらに`FROM`が書かれるとそれらはステージとして分かれることになります。

- 各ステージは無関係: ステージはアーティファクトや中間イメージを共有しません。
- `COPY --from=<stage-name>`で成果物のコピーが可能
- 最終的なビルドステージのみがイメージに保存されます。

ビルド環境と実行環境を別のステージとして用意し、最終的なイメージには実行環境を残す野が典型的な使い方になると思います。

つまりメリットとして以下があります。

- イメージサイズの減少: ビルドに必要なツールなどを含まない形でビルドが行えます。
- attack surfaceの減少: 余計なツールがなくなる。
- ビルド高速化(並列ビルド): 各ステージは無関係なのでそれぞれを同時にビルドすることができます

この記事で見せた例でもmulti-stage buildを行っています。

```dockerfile
FROM docker.io/library/golang:${TAG_GOVER}-${TAG_DISTRO} AS builder

# ...

RUN <<EOF
    # ...
    go build -o ../bin ${MAIN_PKG_PATH}
EOF

# ...

FROM gcr.io/distroless/static-debian12@sha256:6ceafbc2a9c566d66448fb1d5381dede2b29200d1916e03f5238a1c437e7d9ea

COPY --from=builder /app/bin /app/bin

ENTRYPOINT [ "/app/bin" ]
```

`AS`でステージに名前を付け、`go build`でバイナリを出力し、`COPY --from=builder`で最終ステージにコピーしています。最後のステージには`gcr.io/distroless/static-debian12`の内容と、ビルドした`/app/bin`だけが残り、`go`コマンドなどは入りません。

#### (補足)そもそも削除はイメージの容量を減らさない

そもそもファイル削除ではイメージサイズが減りません。
[OCI Image Spec](https://github.com/opencontainers/image-spec/blob/main/spec.md)でも書かれていますが、削除は[whiteout](https://github.com/opencontainers/image-spec/blob/main/layer.md#whiteouts)という仕様にしたがってマーカーファイルを置くことで表現されます。実際にはデータは消えません。
ファイルの削除によって本当にイメージ上からファイルが消えるのは、そのファイルがそのレイヤー上で作成されていたときのみです。

実際に挙動を観測してみましょう。
例えば`build-essential`を導入して`gcc`などを含めたビルドツールを導入してみます。

```dockerfile: build-essential.Containerfile
FROM docker.io/library/ubuntu:noble-20251013

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
<<EOF
    rm -f /etc/apt/apt.conf.d/docker-clean
    echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
    apt-get update
    apt-get install -yqq --no-install-recommends build-essential
EOF
```

こうするとimage (virtual) sizeが300MiB以下程度跳ね上がっていることがわかります。

```
$ podman image ls
REPOSITORY                          TAG                IMAGE ID      CREATED         SIZE
localhost/build-essential           0.0.0              6a2afdf0228d  17 seconds ago  378 MB
docker.io/library/ubuntu            noble-20251013     c3a134f2ace4  6 weeks ago     80.6 MB
```

これに対してさらに、導入したパッケージを消すようなコードを加えてみます。

```dockerfile: build-essential-rm.Containerfile
FROM docker.io/library/ubuntu:noble-20251013

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
<<EOF
    rm -f /etc/apt/apt.conf.d/docker-clean
    echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
    apt-get update
    apt-get install -yqq --no-install-recommends build-essential
EOF

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
<<EOF
    rm /usr -rf
EOF
```

ファイルは消えているようですが

```
$ podman container run -it --rm localhost/build-essential-rm:0.0.0
Error: crun: executable file `/bin/bash` not found: No such file or directory: OCI runtime attempted to invoke a command that was not found
```

容量は変わりません

```
REPOSITORY                          TAG                IMAGE ID      CREATED         SIZE
localhost/build-essential-rm        0.0.0              f1d0c07bcbb4  2 minutes ago   378 MB
```

### ポイント4: RUN --mount=type=cacheでキャッシュする

https://docs.docker.com/reference/dockerfile/#run---mounttypecache

ビルド間でキャッシュを永続化する機能です。

実際に以下のように利用しています。

```dockerfile
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
<<EOF
    rm -f /etc/apt/apt.conf.d/docker-clean
    echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
    apt-get update
    apt-get install -yqq --no-install-recommends git-lfs openssh-client
EOF
```

```dockerfile
RUN \
    --mount=type=cache,target=/go \
    --mount=type=cache,target=/root/.cache/go-build \
<<EOF
    go mod download
    # go generate ./...
    go build -o ../bin ${MAIN_PKG_PATH}
EOF
```

- 同時アクセス不可なものは`sharing=locked`をつける。
  - `apt`は同時に１つしか実行できない。
  - `Go`のキャッシュはconcurrenct-safe
    - `GOPATH`(`/go`): [26794#issuecomment-442953703](https://github.com/golang/go/issues/26794#issuecomment-442953703)
    - `GOCACHE`(`~/.cache/go-build`): `go help cache`参照: `The cache is safe for concurrent invocations of the go command.`
- どっちかわかんない場合は`sharing=private`にしておくと(効率は落ちるが)安全。
- debian系のベースイメージは`apt`キャッシュを消す設定になっている。
  - 実際にコンテナに入って`/etc/apt/apt.conf.d/docker-clean`の中身を見たらわかりますが、ダウンロードしたキャッシュなどを消す設定が入っています。
  - これはdockerユーザー向けの気遣いです。
    - キャッシュをかけないケースではこの設定のほうがイメージが軽くなるので良いです。

### ポイント5: RUN --mount=type=bindでソースコードをマウントする

https://docs.docker.com/reference/dockerfile/#run---mounttypebind

unix系のシステムでディレクトリを別のディレクトリにつなげることを`bind mount`と言います。[mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html)とか読んでおいてください。

ビルドコンテクストとかほかのステージとかをマウントすることができる機能です。

```dockerfile
WORKDIR /app/src
RUN \
    --mount=type=bind,target=/app/src \
<<EOF
    go build -o ../bin ${MAIN_PKG_PATH}
EOF
```

こうすることでソースコードのコピーを行うことなくビルドを行うことができます。

#### (補足)--mount=type=bindを使わないほうがいい時

プログラミング言語やビルドの仕方によっては`--mount=type=bind`を使わないほうがよさそうな時があります。

代わりの下記のように`COPY`でbuild-contextをbuild backendに送ります。

```dockerfile
WORKDIR /app/src
COPY . .
RUN <<EOF
    go build -o ../bin ${MAIN_PKG_PATH}
EOF
```

`COPY . .`の行でビルドコンテクストがすべてbuild backendに送信されてしまうはずなので、不要なファイルは`.dockerignore`などで無視できるようにしたほうが良いです。

https://docs.docker.com/build/concepts/context/#dockerignore-files

どういう時に使うべきじゃないかというと

ビルドシステムがbuild-contextに固有なものを書き出してしまうとき

です。
以下が具体例です。

- `Node.js`
  - `node-gyp`が使われてるとき
    - `npm`モジュールがnative-moduleをビルドしてしまう時、プラットフォーム差が発生
  - `devDependencies`が存在する
    - `./node_modules`の中身がビルドに影響されて入れ替わるため、ホスト側に影響が波及してしまう
  - 例外: `esbuild`などでバンドルするため出力がディレクトリ外に指定できるとき。
- `python`
  - `venv`(`uv`/`poetry`など)がディレクトリ内に作成される場合
    - デフォルト設定で`pyroject.toml`と同じ階層の`.venv`ディレクトリに書き込みます。
    - `venv`はsymlinkで`python`実行ファイルにリンクされるためコンテナ内外でリンク先の違いにより齟齬が発生
  - インストール時にnative moduleをビルドするとき
    - 結構何でもビルドしてる感じします。
- `Rust`
  - なにも設定を加えないとディレクトリ内の`target`ディレクトリにビルド成果物を出力します。
  - 例外: [build.target-dir](https://doc.rust-lang.org/cargo/reference/config.html#buildtarget-dir)の指定でディレクトリ外に出力されるとき

どっちかというと`Go`がコンテナ上でビルドするときに都合が良すぎるだけで、基本はコピーしたほうが安全かもしんないです。
もちろん`Go`でもビルドスクリプト中で環境に固有なものを吐き出してしまう場合は`bind`を使わないほうがいいです。めったにないとは思うんですが。

### ポイント6: private gitを使うための設定(RUN --mount=type=ssh)

[以前の記事のこの部分](<https://zenn.dev/ngicks/articles/go-basics-revisited-starting-projects#(private-git%E3%81%8B%E3%81%A4%E3%82%B5%E3%83%96%E3%82%B0%E3%83%AB%E3%83%BC%E3%83%97%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%99%E3%82%8B%E5%A0%B4%E5%90%88)module-name%E3%81%AB.git%E3%82%92%E3%81%A4%E3%81%91%E3%82%8B>)で説明しましたが、`Go`でprivate VCSでホストされるモジュールを取得する際、特別な設定がされていなければ`VCS`に対応するコマンド、つまり`git`コマンドが使用されます。

そこでprivateなGo moduleが含まれる場合、以下が必要となります。

- `Dockerfile`側:
  - ssh clientの追加
  - ssh-keyscanによるknown_hostsのリスト
  - `insteadOf`設定によって`git`コマンドが`ssh`を使うように変更
  - `RUN --mount=type=ssh`によって`ssh-agent`のソケットのマウントを要求
- ホスト側:
  - `ssh-agent`の準備
  - `github.com`/`gitlab.com`などのホスティングサービスにssh keyの登録

https://docs.docker.com/reference/dockerfile/#run---mounttypessh

#### Dockerfile側

`Dockerfile`は以下のように各種設定が必要です。

```dockerfile
# openssh-clientが必要なので入れておく
RUN \
<<EOF
    apt-get update
    apt-get install -yqq --no-install-recommends openssh-client
EOF

ARG SSH_HOSTS="github.com,gitlab.com"

# `~/.ssh/known_hosts`を埋めておくことで、ssh login時にプロンプトが出ないようにする
# ホストの`~/.ssh/known_hosts`をマウントしたほうがいいかも
# ssh-keyscanはHTTP_PROXYを無視するのでproxy下環境では使えない。
RUN <<EOF
    mkdir -p -m 0700 ~/.ssh
    for item in $(echo $SSH_HOSTS | tr ',' '\n' ); do
      if [ ! -z ${item} ]; then
        git config --global url."ssh://git@${item}".insteadOf https://${item}
        ssh-keyscan ${item} >> ~/.ssh/known_hosts
      fi
    done
EOF

RUN --mount=type=ssh \
<<EOF
    go mod download
    # go generate ./...
    go build -o ../bin ${MAIN_PKG_PATH}
EOF
```

#### ホスト側

ホスト側ではssh keyの作成、ssh-agentの準備、ホスティングサービスへのpubkeyの登録などが必要です。下記を参考に行ってください。

https://docs.github.com/en/authentication/connecting-to-github-with-ssh

https://docs.gitlab.com/user/ssh/

筆者はgpg-agentにssh-agentの役割を担ってもらうことにしています。
下記のスクリプトを`.bashrc`などから読み込ませると

```bash
if [ -t 0 ]; then
        # Set GPG_TTY so gpg-agent knows where to prompt.  See gpg-agent(1)
        export GPG_TTY="$(tty)"
fi

# https://wiki.archlinux.org/title/GnuPG#SSH_agent

# Start gpg-agent if not already running
if ! pgrep -x -u "${USER}" gpg-agent &> /dev/null; then
  gpg-connect-agent /bye &> /dev/null
fi

# Additionally add:
# Set SSH to use gpg-agent (see 'man gpg-agent', section EXAMPLES)
unset SSH_AGENT_PID
if [ "${gnupg_SSH_AUTH_SOCK_by:-0}" -ne $$ ]; then
  # export SSH_AUTH_SOCK="/run/user/$UID/gnupg/S.gpg-agent.ssh"
  export SSH_AUTH_SOCK="$(gpgconf --list-dirs agent-ssh-socket)"
fi

# Refresh gpg-agent tty in case user switches into an X session
gpg-connect-agent updatestartuptty /bye > /dev/null

dbus-update-activation-environment --systemd SSH_AUTH_SOCK
```

#### (podman)ビルド直前にgpg-agentをunlockしておく

下記より、`buildah`にハードコードされた２秒というタイムアウトがあるためssh-agent経由で鍵にかかったpassphraseのプロンプトを入力するのはほぼ不可能です。

https://github.com/containers/buildah/blob/v1.42.1/pkg/sshagent/sshagent.go#L126-L131

下記のような感じで、`ssh-add -T`をビルド直前に読んで鍵のアンロックを行っておく必要があります。

```
ssh-add -T ~/.ssh/id_ecdsa.pub

podman buildx build \
    --ssh default=${SSH_AUTH_SOCK} \
...
```

`Docker`の実装はだいぶ違ったのでこの問題は起きないかも。特に検証していないのでわかりませんが。

### ポイント7: distrolessを使うならsha256sumでイメージを指定しよう

最終ステージをこうしています。

```dockerfile
FROM gcr.io/distroless/static-debian12@sha256:6ceafbc2a9c566d66448fb1d5381dede2b29200d1916e03f5238a1c437e7d9ea
```

これは[distroless](https://github.com/GoogleContainerTools/distroless)が基本的にタグをつけないスタイルだからです。

このsha256sumは以下の手順でえらえます。

```
$ podman image pull gcr.io/distroless/static-debian12:latest
$ podman image inspect gcr.io/distroless/static-debian12:latest --format '{{.Digest}}'
sha256:ed92139a33080a51ac2e0607c781a67fb3facf2e6b3b04a2238703d8bcf39c40
```

`:latest`の中身はビルドされるたびに変わるので時々pullしなおします。

### 企業プロキシ下版の考慮点

- `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`とそれらの小文字版を導入
  - ステージ内で`ARG`を宣言すると`RUN`コンテクスト内で環境変数として挿入されます。`FROM`より前の行で宣言しないように注意！
- 企業プロキシは`ssh`を通さないことが多いみたいなので`ssh`関連のものは全部削除
- `SSL_CERT_FILE`, `NODE_EXTRA_CA_CERTS`, `DENO_CERT`を宣言することで`curl`など、`Node.js`、`deno`がそれぞれが企業プロキシのオレオレ証明書を含んだca bundleを使うように指定します。
  - imageに`ca-certificates`パッケージを導入する場合はパスかぶりを避けるため`/etc/ssl/certs/ca-certificates.crt`以外の位置(`/ca-certificates.crt`など)を指定してマウント位置も買えたらいいです。
- `.netrc`ファイルを作成し、secret mountでマウントする
  - 平文で機密情報を書かないといけないフォーマットなのでさけられるなら避けたほうがいです。
  - フォーマットは[IBM: .netrc ファイルの作成](https://www.ibm.com/docs/ja/aix/7.2.0?topic=customization-creating-netrc-file)などをご覧ください

```diff dockerfile
 ARG GOPATH="/go"
 ARG GOPRIVATE=""

-ARG GIT_SSH_HOSTS="github.com,"
 ARG MAIN_PKG_PATH="."

+ARG HTTP_PROXY
+ARG HTTPS_PROXY=${HTTP_PROXY}
+ARG NO_PROXY
+ARG http_proxy=${HTTP_PROXY}
+ARG https_proxy=${HTTP_PROXY}
+ARG no_proxy=${NO_PROXY}
+
+# for curl, etc.
+ARG SSL_CERT_FILE="/etc/ssl/certs/ca-certificates.crt"
+ARG NODE_EXTRA_CA_CERTS=${SSL_CERT_FILE}
+ARG DENO_CERT=${SSL_CERT_FILE}
+
 RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
     --mount=type=cache,target=/var/lib/apt,sharing=locked \
+    --mount=type=secret,id=certs,target=/etc/ssl/certs/ca-certificates.crt \
 <<EOF
     rm -f /etc/apt/apt.conf.d/docker-clean
     echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
     apt-get update
-    apt-get install -yqq --no-install-recommends git-lfs openssh-client
-EOF
-
-RUN <<EOF
-    mkdir -p -m 0700 ~/.ssh
-    for item in $(echo $GIT_SSH_HOSTS | tr ',' '\n' ); do
-      if [ ! -z ${item} ]; then
-        git config --global url."ssh://git@${item}".insteadOf https://${item}
-        ssh-keyscan ${item} >> ~/.ssh/known_hosts
-      fi
-    done
+    apt-get install -yqq --no-install-recommends git-lfs
 EOF

 WORKDIR /app/src

-RUN --mount=type=ssh \
+RUN --mount=type=secret,id=netrc,target=/root/.netrc \
     --mount=type=secret,id=goenv,target=/root/.config/go/env \
+    --mount=type=secret,id=certs,target=/etc/ssl/certs/ca-certificates.crt \
     --mount=type=cache,target=/go \
     --mount=type=cache,target=/root/.cache/go-build \
     --mount=type=bind,target=/app/src \
```

### go.mod記載のgo versionから最新のpatch versionを取得

ということができるツールを作りました。

こういう`go.mod`があるディレクトリで

```gomod
module github.com/ngicks/go-example-basics-revisited/building-with-docker

go 1.25.0

require (
	github.com/ngicks/go-iterator-helper v0.0.23
	github.com/ngicks/go-playground v0.0.1
)
```

で実行すると

```
$ go run github.com/ngicks/go-common/tools/golatestpatchver@latest
1.25.4
```

これをビルドスクリプトから読み込ませれば常に最新のパッチバージョンでビルドできます！

一応どのように作ったかは:

- `Go`のすべてのバージョンの一覧は以下の二通りの方法で得ることができます
  - `curl https://proxy.golang.org/golang.org/toolchain/@v/list`
  - `git ls-remote --tags https://go.googlesource.com/go 'go*'`
- 多分リリースされてから反映されるであろうから`goproxy`に問い合わせるほうがリリース直後のタイミング問題が起きにくいから、そちらのほうがいい気がしますが、すべてのツールチェイン向けのデータが出てきてちょっと冗長なので`git`のほうを採用することにしました。
- 出力結果を解析し、ソートします。
  - バージョンタグはsemverにしたがっておらず、`go1.9beta1`のような感じで数字部分が`x.y.z`の３桁じゃなかったり、`beta1`, `rc1`のようなsuffixが付きます。
  - そこで, `beta1` -> `-beta.1`に変換するなどしてsemverにしてから解析しています。
- `x.y.z`の部分のうち、`y`が一致しているもので最新をprintします。

となります。ちょっとめんどくさいですね。

### ビルドして実行

:::message

**実際にはこのままではビルドは成功しません!**

private repositoryの挙動のチェックのために、筆者しかアクセスできないprivate gitへのアクセス権が必要になっています。

:::

以下のようなスクリプトでビルドできます。

```
#! /bin/sh

set -Cue

TAG_GOVER=1.25.0
if [ -f ./ver ]; then
  TAG_GOVER=$(cat ./ver)
fi

arch=${TARGET_ARCH:-$(go env GOARCH)}

echo $arch

# let gpg key unlocked for ssh login.
# As you can see in https://github.com/containers/buildah/blob/v1.42.1/pkg/sshagent/sshagent.go#L126-L131
# buildah sets 2 sec timeout for ssh-agent so you have low chance to successfully enter passphrase.
ssh-add -T ~/.ssh/id_ecdsa.pub

podman buildx build \
    --platform linux/${arch} \
    --build-arg TAG_GOVER=${TAG_GOVER} \
    --build-arg HTTP_PROXY=${HTTP_PROXY} \
    --build-arg HTTPS_PROXY=${HTTPS_PROXY} \
    --build-arg MAIN_PKG_PATH=${MAIN_PKG_PATH:-./} \
    --build-arg GOPRIVATE=${GOPRIVATE} \
    --secret id=certs,src=/etc/ssl/certs/ca-certificates.crt \
    --secret id=goenv,src=$(go env GOENV) \
    --ssh default=${SSH_AUTH_SOCK} \
    -t ${1}-${arch} \
    -f Containerfile \
    .
```

最新のパッチバージョンは`go run github.com/ngicks/go-common/tools/golatestpatchver@latest > ver`でファイルに保存してそこを経由させています。
こうしないと最新の`Go`がリリースされてから`dockerhub`にイメージが上がるまでの隙間時間に対応しにくいですからね。
のちの`multi-arch build`に備えてarchでtagをsuffixすることにしています。

jokeなのでjokeという名前でビルドしてみます

```
$ ./build.sh joke:0.0.3
```

できます

```
$ podman image ls
REPOSITORY                          TAG                IMAGE ID      CREATED            SIZE
localhost/joke                      0.0.3-amd64        b2c977b38cfb  3 days ago         5.45 MB
```

実行してみます。

```
$ podman container run -t --rm localhost/joke:0.0.3-amd64

yay
🐤< ｺﾝﾆﾁﾊ！ ₍₍⁽⁽ 🐓₎₎⁾⁾ ₍₍⁽⁽🐔₎₎⁾⁾ ₍₍⁽⁽🐣₎₎⁾⁾ ₍₍⁽⁽🐧 ₎₎⁾⁾

```

鳥が踊ります。

## (おまけ)multi-arch build

手元のPCは`amd64`だけど実行したい環境は`arm64`だ、とかそういうケースが最近は増えてきているんじゃないかと思います。
というのもmacのラップトップは`arm64`だったりしますし、Raspberry Piも`arm64`です。

そもそも対象読書はCPU Architectureというのがわからないでしょうか？
CPU Architectureとはプログラムから見ると命令セットの仕様のことです。
昨今の高レベルなプログラミング言語を書いていると違いは意識されにくいかもしれませんが、アセンブリを直接書くと大幅に違います。
`Go`もランタイムのところにアーキテクチャ依存のアセンブリがおいてあるので違いを見比べてみるといいと思います。

`arm64`(`aarch64`)は`amd64`(`x86_64`)より安価なので利用される場面が多いようです。

ビルドシステムが動作しているマシンとは異なるOS/アーキテクチャ向けにプログラムをビルドすることをcross-compilationなどと呼びます([Wikipedia: Cross-Compiler](https://en.wikipedia.org/wiki/Cross_compiler))。
`Go`は容易に別システム向けのバイナリをビルドできますが、コンテナは`Go`だけでは済まないことがあるため、`qemu`などのVMを使ってcross-compilationを行います。

`buildah`のドキュメント曰く`multi-arch build`には`qemu-user-static`が必要です([\[1\]](https://github.com/containers/buildah/blob/v1.42.1/docs/buildah-build.1.md), [\[2\]](https://github.com/containers/buildah/blob/v1.42.1/docs/buildah-from.1.md))

ソースを見る限り`binfmt_misc`も必要ですが、そもそも`wsl2`のインスタンスなら元から有効になっているようです。

```
$ sudo apt install qemu-user-static
# wsl2なら既に有効になっているはず
$ sudo systemctl enable --now systemd-binfmt
$ modprobe binfmt_misc
# 何も表示されなければok
```

さて、`Containerfile`についてなのですが、

```dockerfile
FROM docker.io/library/golang:${TAG_GOVER}-${TAG_DISTRO}
```

の部分は、dockerの仕様に基づきmultiarchに対応したイメージである場合は自動的にビルド対象のarchのイメージがpullされるのでこのままでよいです。

問題はdistrolessの部分で、こちらはhash sumで指定しているので自動的なフォールバックがかからないんですね。

```diff dockerfile
-FROM gcr.io/distroless/static-debian12@sha256:6ceafbc2a9c566d66448fb1d5381dede2b29200d1916e03f5238a1c437e7d9ea
+# arm64
+FROM gcr.io/distroless/static-debian12@sha256:ed92139a33080a51ac2e0607c781a67fb3facf2e6b3b04a2238703d8bcf39c40
```

(この部分答えが出ていない。どうしたらいんだろう？)

ではこの状態でビルドしてみます。
`build.sh`ではarchで分岐できるように調節してあるので、環境変数を設定してスクリプトを呼び出すだけです。
[Red Hat Documentation: コンテナーの構築、実行、および管理: 4.8.マルチアーキテクチャーイメージのビルド](https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/building-multi-architecture-images_assembly_working-with-container-images)を見る限り、`podman build`で`--platform linux/amd64,linux/arm64`を指定すればまとめてビルドしてくれるみたいですが、安定してイメージにarchに基づいたタグをつける方法がぱっと見つからなかったので細かく分けています。privateに使えるcontainer registryを動作させているならそちらの方法でよいかなと思います。

```
TARGET_ARCH=arm64 ./build.sh joke:0.0.3
```

特に問題なくビルドできました。

本当に動くのかどうかを`Raspberry Pi`で試しましょう。
筆者は`k3s`でクラスターを組んでいるので、そこに投入します。レジストリ経由ではないイメージの受け渡しの方法がすぐに見つからなかったのでローカルにイメージを保存してローカル経由で`k3s`組み込みの`containerd`にロードさせます。

```
$ podman image save localhost/joke:0.0.3-arm64 | gzip > joke:0.0.3-arm64.tar.gz
$ scp ./joke:0.0.3-arm64.tar.gz ${remote-machine}:/tmp
$ ssh ${remote-machine}
$$ sudo k3s ctr images import ./tmp/joke\:0.0.3-arm64.tar.gz
```

動かしてみます

```
$ kubectl run joke --image=localhost/joke:0.0.3-arm64 --image-pull-policy=Never
pod/joke created
$ kubectl logs pod/joke
yay
🐤< ｺﾝﾆﾁﾊ！ ₍₍⁽⁽ 🐧₎₎⁾⁾ ₍₍⁽⁽🐓₎₎⁾⁾ ₍₍⁽⁽🐔₎₎⁾⁾ ₍₍⁽⁽🐣 ₎₎⁾⁾
 %
```

動いてますね。

## おわりに

同僚の書いた`Dockerfile`が`build-essential`やどでかい依存を丸ごとイメージに残しているからビルドや配布物のアップロードに時間がかかって辛い思いをしたあの夜や、どうやって`Dockerfile`を書くのか微妙に思い出せなくて過去に書いた`Dockerfile`を引っ張り出してコピペしなおしたあの時の自分を救うためにいろいろ書きました。

private gitを使うさいのビルド方法は結構難儀したしあまりまとまって書かれていることもないように思うのでまとめました。

資料や挙動は確認できるものはしていますが、間違っている場合にはコメントで教えていただけると幸いです。

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
