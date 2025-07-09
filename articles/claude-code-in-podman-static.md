---
title: "podman-staticのsandboxでclaude-codeを動かす"
emoji: "⛓️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claude-code", "podman"]
published: true
---

## podman-staticのsandboxでclaude-codeを動かす

podman-staticの設定でちょっとてこずったのでまとめておきます。

社内で共有できたらいいかなあと思って書いているのであんまり凝った内容じゃないです。ご注意ください。

## やること

- dockerのインストール
- podman-staticのビルド
- podmanの設定
- podmanで動かす作業用コンテナイメージのビルド
- コンテナでclaude-codeをrun

試してないけど全部スクリプト化してあります。([ここ](https://github.com/ngicks/dotfiles/tree/main/build_podman_static))

## やらないこと

- dockerがなにかとかpodmanが何かとかの説明
- より高度な制限(ネットワークの接続先の限定など)
  - これは今後の課題としておきます。

## 前提

- linux環境があること
- それが比較的新しめのubuntuか、適宜自分で読み替えられること
- linuxにおけるterminal操作に対する多少の習熟

## 環境

```
$ uname -srvmpi
Linux 5.15.167.4-microsoft-standard-WSL2 #1 SMP Tue Nov 5 00:21:55 UTC 2024 x86_64 x86_64 x86_64
```

## なぜ？

claude-codeなどを含め、LLMに直接terminalの操作権を与えてコードを生成させたり読ませて解説させたりをすることが最近増えてきました。

通常claude-codeを動かすときはユーザー権限（terminalにログインしているあなた）しか与えないと思うので、システムに重要なファイルを消すようなことはできないと思いますが、
例えばホームディレクトリ以下(`/home/yourname`)を全部消すとかそういうことは可能です。

claude-codeは安全策としてコマンド実行時(editやbashコマンド)にはユーザーに判断を仰ぐようになっています。
でもすべての変更をいちいち確認するのは面倒ですよね？ということで破壊してもいい環境を作って全部好きにしてもらえばそこそこ安全かつ楽にできます。

sandbox化には[devcontainer](https://containers.dev/)を使う方法など別の方法も考えられますが、よく知らんのでいったん忘れます。

## dockerのインストール

podman-staticがdockerでビルドするのを前提としているので入れます。
じゃあdockerでclaude-code動かせばよくないですか？と思ったそこのあなた。その通りです。
ただ、諸般の事情によりstaticなバイナリでdaemonlessでroot権限を一切必要とせずにコンテナを動かしたい場合はpodmanのほうが多分いいです。

以下をやるだけです。

https://docs.docker.com/engine/install/

```shell
#!/bin/bash

for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove $pkg;
done
```

```shell
#!/bin/bash

# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

というようなbashscriptにできます。

そのまま普段使いのdocker環境をセットアップするなら[rootless mode](https://docs.docker.com/engine/security/rootless/)もやったほうがいいと思いますが今回は不要なので省略です。

有効化しておきます。

```
sudo systemctl start docker
```

## podman-staticのビルド

podman-staticは[podman](https://podman.io/)がstatic(動的ロードされるライブラリがない)なバイナリになるようにいろいろ調節したビルドスクリプトのようです。

中身をちゃんと読みましょう。

https://github.com/mgoltzsche/podman-static/blob/9ccf992536ecf0a4a3b82ac27fc708f00ab141ab/Dockerfile

問題なさそうに見えます。

見てる感じ

```
sudo make singlearch-tar
```

で、全部ビルドできそうです。

`tar-all`でほかのアーキテクチャ、プラットフォーム向けのビルドができるようです。

ビルド成果物を全部`~/.local/containers`以下に突っ込みます。

```
uid=$(id -u)
gid=$(id -g)
sudo chown ${uid}:${gid} -R ./build
cp -r ./build/asset/podman-linux-amd64/usr/local/ ~/.local/containers/
```

## podmanの設定

設定をします。

以下より、`$HOME/.config/containers`以下に設定ファイルを置いたらデフォルトで読み込まれますのでそこに置けばよいです。

https://docs.podman.io/en/stable/markdown/podman.1.html#configuration-files

以下の名前を持つファイルを`$HOME/.config/containers`に置きます。

- `containers.conf`: https://github.com/containers/common/blob/main/docs/containers.conf.5.md
- `policy.json`: https://github.com/containers/image/blob/main/docs/containers-policy.json.5.md
- `registries.conf`: https://github.com/containers/image/blob/main/docs/containers-registries.conf.5.md
- `seccomp.json`: https://matsuand.github.io/docs.docker.jp.onthefly/engine/security/seccomp/
- `storage.conf`: https://github.com/containers/storage/blob/main/docs/containers-storage.conf.5.md

基本的には先の[podman-staticのconf](https://github.com/mgoltzsche/podman-static/tree/v5.5.2/conf/containers)をそのまま持ってくればいいと思います。
`secomp.json`はpodman-staticビルド時に出力されるものをそのままでもいい気がしますがセキュリティー意識が強い人は中身見ていじったほうがいいと思います。

ただしいくつかは編集が必要です。
基本的には上記のリンクを順次読んで設定していただくほうが良いと思いますが、下記のようにしてもとりあえず動きます。
`/home/ngicks`の部分を適宜自分のユーザー名のホームディレクトリに変更してください。

### containers.conf

```conf
[engine]
cgroup_manager = "cgroupfs"
conmon_path=["$HOME/.local/containers/lib/podman/conmon"]
events_logger="file"
helper_binaries_dir=["/home/ngicks/.local/containers/bin", "/home/ngicks/.local/containers/lib/podman"]
```

- `cgroup_manager`: `cgroupfs`にしているとsystemdのない環境でも動作するんだと思います。ある環境では基本的に`systemd`(デフォルト値)にしておいたほうがいいと思います(が私が想定している環境ではないことも多々あるため基本`cgroupfs`にしてあります)
- `conmon_path`: 普通は`$PATH`から探されるみたいですが普通の探索の範囲に入れないので設定が必要です。ドキュメントされていませんがソース読む限り`$HOME`は解決してくれるみたいです。
- `helper_binaries_dir`: 同じくです。ただこちらは環境変数の展開が起きるとドキュメントされておらず、実際にも展開されないため固定の設定が必要です。
  - 仕方ないので`$HOME`を絶対パスに置き換えるスクリプトを書くことにしました。

### storage.conf

```
[storage]

driver = "overlay"
runroot = "$XDG_RUNTIME_DIR/containers/storage"
graphroot = "$HOME/.local/share/containers/graphroot"
rootless_storage_path = "$HOME/.local/share/containers/storage"

[storage.options]

additionalimagestores = [
]
ignore_chown_errors = "true"
mount_program = "/home/ngicks/.local/containers/bin/fuse-overlayfs"
mountopt = "nodev,fsync=0"

[storage.options.thinpool]
```

- `runroot`: `$XDG_RUNTIME_DIR`は見たところこの環境では設定されていて、これが`/run/user/$(id -u)`に展開されるためこのように設定します。
  - ない場合はログインスクリプトなどで設定するようにしてください。
- `graphroot` : `$HOME`以下に設定できれば何でもいいです。この項目も環境変数を展開します。
- `rootless_storage_path`: みたところ使われませんが設定しておきます。
- `mount_program`: これは環境変数を展開しません。`$PATH`から解決もしないようです。

## podmanで動かす作業用コンテナイメージのビルド

rootlessでpodmanを動作させるために下記が必要となるので入れます。

```
sudo apt install -y uidmap
```

### 最小版

claude-codeが入っているだけの最小版は下記のようになります。

```dockerfile
# syntax=docker/dockerfile:1.4

FROM ubuntu:noble-20250619

RUN <<EOF
apt-get update
apt-get install -y --no-install-recommends \
    ca-certificates \
    git \
    curl \
    unzip \
    jq
curl -o- https://fnm.vercel.app/install | bash
/root/.local/share/fnm/fnm install 24
eval "`/root/.local/share/fnm/fnm env`"
npm install -g @anthropic-ai/claude-code
EOF

ENV CLAUDE_CONFIG_DIR=/root/.config/claude

WORKDIR /root
```

ポイント

- `npm install`したいので`Node.js`を入れます。
- [公式に推奨される方法の一つであるfnmを使った方法](https://nodejs.org/ja/download/current)をここでは採用しています。
- 個人的には[Volta](https://docs.volta.sh/guide/)でいいんじゃないかと思いますが、これは[#2005](https://github.com/volta-cli/volta/issues/2005)より、Basic Authの必要なproxy影響下にいる場合使用が不可能なためここでは使わないこととしています。
  - かといってfnmがそういった環境で使えるかを筆者は確かめていないことに注意してください。
- `CLAUDE_CONFIG_DIR`を設定しないとコンテナ破棄するたびにログインしなおしが必要になります(語弊あり)。
  - 設定しないと`~/.claude`に設定が入り、`~/.claude.json`にログイントークンなんかが入ります。
  - 設定するとそれらが全部まとめてこのディレクトリに入るので管理が簡単になります。
  - `XDG_CONFIG_DIRS`が設定されているとそれが尊重されるかもしれません。

### 自家版

私も多分に漏れず自分の[dotfiles](https://github.com/ngicks/dotfiles)をメンテしています。
dotfilesに含んでいるのが自然に感じるのかはわかりませんが、いろいろなsdkとツールのインストーラーが入っているのでおおむねいつもの環境を作ることができます。

ということで、dotfilesのインストーラーを使っていつもの開発環境を作り、これをclaudeに使わせることにします。

- Go
- Rust
- Deno
- Volta - Node.js
- uv - CPython
- rbenv - ruby

が現状だとインストールされます。

```dockerfile
# syntax=docker/dockerfile:1.4

FROM ubuntu:noble-20250619

WORKDIR /root

RUN <<EOF
  apt-get update
  apt-get install -y --no-install-recommends\
      ca-certificates \
      git \
      curl \
      make \
      build-essential \
      gcc \
      clang \
      xsel \
      p7zip-full \
      unzip \
      jq \
      tmux \
      libyaml-dev \
      zlib1g-dev
EOF

WORKDIR /root/bin

RUN <<EOF
  curl -L https://github.com/TomWright/dasel/releases/download/v2.8.1/dasel_linux_amd64.gz -o dasel.gz
  gzip -d ./dasel.gz
  chmod +x dasel
EOF

WORKDIR /root/.dotfiles
RUN <<EOF
  git clone https://github.com/ngicks/dotfiles.git .

  git submodule update --init --recursive
  cp ./ngpkgmgr/prebuilt/linux-amd64/* ~/bin

  # ruby installation stuck. Do it twice
  export PATH=$HOME/bin:$PATH
  ./install_sdk.sh
  . ~/.config/env/00_path.sh
  export PATH=$HOME/.local/rbenv/shims/ruby:$HOME/bin:$PATH
  ./install_sdk.sh

  ~/.deno/bin/deno task install

  . ~/.config/env/00_path.sh
  deno task basetool:install
  deno task gotools:install
EOF

WORKDIR /root/.dotfiles
RUN <<EOF
  export PATH=$HOME/bin:$PATH
  . ~/.config/env/00_path.sh
  # if an update includes deno's (almost impossible),
  # update may fails because a deno executable got swapped
  deno task update:all || deno task update:all
EOF

WORKDIR /root
# claude code knows it.
RUN $HOME/.cargo/bin/cargo install ripgrep

WORKDIR /root/.dotfiles
RUN <<EOF
  export PATH=$HOME/bin:$PATH
  . ~/.config/env/00_path.sh
  npm install -g @anthropic-ai/claude-code
EOF

ENV CLAUDE_CONFIG_DIR=/root/.config/claude

WORKDIR /root
```

But why?:

- 言語サーバー設定を普段使ってるものを使いまわしたいから
  - lspをmcpサーバー経由で公開するようなアダプターがあるとする
  - 現状で言語サーバーの起動設定を持っているのは普段使っているエディター(vscodeやneovimのような)となる。
  - そのため、エディター経由でlsp-mcpアダプターを動かすんじゃね？と思ったからその辺がそのまま入ってるといい
  - ※実際そうなっていくかは全然不明。もしかしたら一足飛びにLLM専用の言語サーバープロトコルが出来上がっていくかもしれないし。
- claudeが一部のツールを活用するようなので用意しておいてあげると便利かなと思う
  - pythonで実験コードや機械的な変換をかけるスクリプトを吐いてきたことがある
  - 作業スクリプトをdenoで作らせるとかもあっていいなと思う。
    - LLMがやる作業をスクリプト化させて仮にLLMを保有する国から通信を遮断された時にも使える資産にして置くという考え方がある。
  - etc, etc
  - ここには載ってないがghコマンドを渡してあげるとGitHub Actionsのworkflowの内容とかも見せられるので入れると便利かも。
- いちいちイメージを分けずに使いまわしたい
  - 普段使い環境は全部入りなのでなんでもできるはず≒できないことが見つかり次第dotfilesにフィードバックがかかるので便利。

### build

ビルドは以下のように行います。タグは特に意味なく`devenv`としていますが適当に変えてください。

```bash
#!/bin/bash

podman image build . -f ./devenv.Dockerfile -t devenv
```

ビルド時に以下のようなwarningが出るかもしれません。

```
WARN[0000] Using cgroups-v1 which is deprecated in favor of cgroups-v2 with Podman v5 and will be removed in a future version. Set environment variable `PODMAN_IGNORE_CGROUPSV1_WARNING` to hide this warning.
```

見たところwslの私が使っているディストロではcgroups v1 v2どっちもあるモードで動いている(`/sys/fs/cgroup/unified/`が存在する。)ため、今の設定ではv1側しかうまくpodmanに見せられていないようです。おそらく`cgroup_manager`を`systemd`にすれば問題なく動く気がします。その辺は今後、環境を見て書き換えるようにインストーラースクリプトを詰めようかと思います。

## コンテナでclaude-codeをrun

下記bashscriptを実行して起動したshellでclaudeコマンドを実行します。
初回はログインが求められますが、2回目以降はcredentialが使いまわされるはずです。

```bash
#!/bin/bash

if ! podman volume exists claude-config; then
  podman volume create claude-config
fi

podman run -it --rm --init\
  --mount type=volume,src=claude-config,dst=/root/.config/claude\
  --mount type=bind,src=.,dst=$(pwd)\
  --workdir $(pwd)\
  devenv:latest
```

- `claude-config`という名前のボリュームを作ってこれを先ほどの`CLAUDE_CONFIG_DIR`にマウントします。
  - 別にホストで使っているものをそのままマウントしてもいいんですが、せっかくなので分けます。
  - ここにcredentialとか入るので危ないっちゃ危ないですが、結局元から平文でcredentialとかを保存しちゃうのであんまり気にするこたないかと思ってます。
  - 前述の設定より、`storage.conf`の`graphroot`で設定したパス以下の`volume/<volume-name>`以下に保存されます。気になるなら暗号化したボリュームを作ったほうがいいと思います。
  - claude-codeがgpgなどで暗号化する対応を待ったほうがいいんじゃないかと思います。
- パス構成を保ったままcwdをマウントします。
  - `claude-config`下に保存されるセッション情報にパスが含まれるためそのままパス構成を保っておくほうが望ましいです。
  - cwdをマウントしておけばホスト側からclaudeの変更を監視できるので都合が良いです。
    - claudeに好きにremoteにプッシュさせる勇気は筆者には現状ありません。
    - プッシュさせるならGitHub Actions上で動かしたらいいかなあと思ってます。ブランチプロテクションもかけられるしちょうどいいんじゃないかと。
    - コンテナにはgitconfigなどは言っていないため、もう少し工夫しないとcommitにsignできなかったりいろいろ困ります。
  - workdirも同様に$(pwd)にすることでcdしてマウントしたところに移動する手間をなくします。

## おわりに

- podman-staticをビルドしてpodmanでclaudeを動かせるところまでの手順を示しました。
- ちょびっと使ってみていますが、特に支障はないように思います。

今後は

- 各種設定をもっと詳しく見ていきます。
- インストーラースクリプト([これのこと](https://github.com/ngicks/dotfiles/tree/main/build_podman_static))をリファクタします。
  - 前述した`cgroup_manager`の自動的な切り替えなどを行います。
- ネットワークの接続先に制限をかける方法を調べます
  - netavarkのプラグインでやるか、
  - ホスト側のiptablesをいじって禁止するかのどちらかでやることになるでしょう。
- `secomp.json`をもうちょい詰めます。
- [libkrun](https://github.com/containers/libkrun)を用いてkernelごと分離して動作させてみます。
