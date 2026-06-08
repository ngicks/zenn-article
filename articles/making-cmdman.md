---
title: "cmdman: バックグラウンドでアプリを動かすマネージャーをLLM叩いて作った話"
emoji: "😈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "cli", "tmux"]
published: false
---

## cmdman: command daemonを作った話

こんにちは。

ターミナルをブロックするコマンドを裏で動かすだけのマネージャーアプリを作ったので、そのアプリの紹介とか、した工夫とか、作ってく中で学んだこととかについて述べます。
[claude code]と[codex]でほぼ作っててソースは手でほとんどいじっていないので、それらをうまく動かすためにした工夫とかも書きます。

ソースコードは下記

https://github.com/ngicks/cmdman

インストール方法は下記

```shell
go install github.com/ngicks/cmdman/cmd/cmdman@latest
```

[mise][mise-en-place]を使う場合は下記

```toml: mise.toml
[tools]
"go:github.com/ngicks/cmdman/cmd/cmdman" = { version = "latest", install_env = { CGO_ENABLED = "0" } }
```

信用ならない他者のアプリなので、ソースをローカルに持ってきてあなたのLLMエージェントに「このアプリは危険に違いないからどこが危険か教えて」と聞いてみたほうがいいでしょう。言ってくる危険度が重大でないなら安全とみなせます。

## cmdman(作ったアプリ)の概要

まず作ったアプリのcli API shapeを確認してもらうことで、何を実現したかったのかとか、どういう話をしそうかについて想像がつくようにしましょう。

- 渡されたcliアプリをバックグラウンドで実行でき
  - `ls`: アプリの一覧を見たり
  - `start/stop/restart`: 停止したり、開始したり
  - `send-keys`: 任意の入力をstdinに送ったり
  - `compose`: yamlファイルを書くことで一連のコマンドを起動順序をつけて実行できたり
  - `mux`: yamlファイルを書くことで、`tmux`のようなterminal multiplexerにバックグラウンドアプリをアタッチしたり
  - `tui`: それらを`tui`上から管理できます

できます。

例として`drawio`を起動するならこう

```
cmdman run --name drawio --tty -- drawio
```

実行後、呼び出したターミナルはすぐにアンブロックされます

```
# ログを確認したり
cmdman logs drawio
# tty有効な場合はアプリにアタッチして操作することもできます。
# (drawioの場合は特に操作するところはないですが)
cmdman attach drawio
```

`podman`をまねてdaemonlessなつくりになっています。  
ですのでdaemonizeしたmonitorプロセスが渡されたコマンドを実行します。

```
$ cmdman inspect drawio --format '{{.StateJSON.MonitorPID}}'
289157
```

```
$ ps -o pid,ppid,pgid,sid,tty,stat,cmd -p 1181 -p 289157 -p 289167
    PID    PPID    PGID     SID TT       STAT CMD
   1181    1180    1180    1180 ?        S    /init
 289157    1181  289149  289149 ?        Sl   /path/to/cmdman --data-dir /... --runtime-dir /... __monitor --id ...
 289167  289157  289167  289167 pts/44   Ssl+ /opt/drawio/drawio
```

こういう感じ。`PPID`(親PID)が`init`, `TT = ?`(`tty`なし), `PID != SID`なのでdaemonizeできていますね。

monitorプロセスはcmdmanの隠しコマンドを実行する形。別のバイナリになってないのは手抜きです・・・

`compose`機能は以下みたいなyamlを書くことで機能します

```yaml
name: cmdman-compose-example
commands:
  yay1:
    args:
      - echo
      - yay
  yay2:
    env:
      - SLEEP_SECS=${INPUT_SLEEP:-3}
    args:
      - sleep
      - ${SLEEP_SECS}
  nay:
    args:
      - echo
      - nay
    after:
      yay1:
        condition: completed
      yay2:
        condition: completed
```

依存関係に基づいて起動順序が決まります。ロジックは`docker compose`とほとんど一緒のはず。

```
cmdman compose -f ./example.compose.yaml up
⠴ yay1             Starting
⠴ yay2             Waiting
◌ nay              Created
```

`mux`サポートは例えば下記のようなyamlを書き、

```yaml:devenv.yaml
name: devenv
commands:
  nvim:
    args:
      - nvim
    tty: true
  shell:
    args:
      - $HOME/.nix-profile/bin/zsh
    tty: true
  claude:
    args:
      - $HOME/.dotfiles/devenv_run.sh
      - ""
      - -lc
      - claude
    tty: true
  codex:
    args:
      - $HOME/.dotfiles/devenv_run.sh
      - ""
      - -lc
      - codex
    tty: true
mux:
  layouts:
    - name: default
      root:
        dir: h
        splits: [50%, 50%]
        panes:
          - dir: v
            splits: [1, 20%]
            panes:
              - nvim
              - shell
          - dir: v
            splits: [50%, 50%]
            panes:
              - command: claude
                focus: true
              - codex
    - name: claude-focused
      root:
        dir: h
        splits: [50%, 50%]
        panes:
          - dir: v
            splits: [1, 20%]
            panes:
              - nvim
              - shell
          - command: claude
            focus: true
    - name: codex-focused
      root:
        dir: h
        splits: [50%, 50%]
        panes:
          - dir: v
            splits: [1, 20%]
            panes:
              - nvim
              - shell
          - command: codex
            focus: true
```

`tmux`セッションの中で

```
$ cmdman compose -f ./devenv.yaml up
$ cmdman compose -f ./devenv.yaml mux 0
```

を実行すると、下記のように`tmux`のwindowが分割されて、それぞれpaneがそれぞれのコマンドにattachします。

![](/images/making-cmdman/mux-tmux.png)

_(黒塗り部分隠す意味ない気がするけど一応隠してる)_

`tui`は下記のような感じ

```
cmdman tui --popup
```

![](/images/making-cmdman/tui-popup.png)

`--popup`オプションで`tmux popup`の中でtuiを表示します。

## 環境

wsl2でUbuntuです

```
❯ wsl.exe --version
WSL バージョン: 2.6.1.0
カーネル バージョン: 6.6.87.2-1
WSLg バージョン: 1.0.66
MSRDC バージョン: 1.2.6353
Direct3D バージョン: 1.611.1-81528511
DXCore バージョン: 10.0.26100.1-240331-1435.ge-release
Windows バージョン: 10.0.26200.8457
```

```
❯ cat /etc/*-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=24.04
DISTRIB_CODENAME=noble
DISTRIB_DESCRIPTION="Ubuntu 24.04.4 LTS"
```

```
❯ tmux -V
tmux 3.6a
```

```
❯ cmdman --version
version:     v0.0.11
go version:  go1.26.3
```

Native Linux / Macでも動くと思うけど(特にMacで)なにも試していない。
Github Actionsでテストぐらいは走らせようかなあ？

## 対象読者

前提知識: 以下は所与のものとされます。(知らんくてもなんとなく読める気もする)

- Linux/POSIXシステムでコマンドを実行できること
- [tmux]のある程度の習熟
- [podman] / [docker compose]の動作モデル。
  - 軽く説明しますが基本的には既知であるとします。
- daemonizeだのptyだの言われてもなんとなくわかる
- [Go]の読解
  - ほかのプログラミング言語が読み書きできればなんとなくわかる気も
- [claude code] / [codex]の使い方
  - [Skills]だの[Hooks]だの言われてもなんとなくわかること

モチベーション面: overview(何の話をするか)も参考に

- こういうアプリを作るときのコアコンセプトとか気を付けどころを知りたい
- [Go]でLLMのプロジェクトを動かすときの苦しみともがきを見たい
  - しかしてそこまで高度なことはしていない

## Overview(TL;DR)

TODO: やったことかく

## Motivation

日々開発をしてるとルーチンワークみたいなものが発生してくるんですね

- 図をいじりたいから[drawioのローカルアプリ](https://github.com/jgraph/drawio-desktop/releases)を起動
- 踏み台サーバー経由でchromeを開きたいから`ssh-add -T ~/.ssh/...` -> `ssh -ND 1090 bastion` -> `google-chrome --proxy-server="socks5://localhost:1090"`のような、順序と依存があるコマンド列の立ち上げ。
- [go-grip](https://github.com/chrishrb/go-grip) (ローカルのmarkdownファイルをブラウザでレンダーしてくれるやつ)を特定のディレクトリで特定オプション(ディレクトリ後にlistenするポートを変えながら)起動
- [zenn preview](https://zenn.dev/ai4u_shunsuke/articles/zenn-cli-usage#%E3%83%97%E3%83%AC%E3%83%93%E3%83%A5%E3%83%BC)をzenn連携してるgithub repositoryで起動

_(例えば`go-grip`を起動すると下記ようになる。特にログは流れ続けたりしないが、Ctrl+cなどでキャンセルするまでアプリが実行される)_

```
$ go-grip -p 1122 -b=0
🚀 Starting server: http://localhost:1122/README.md
📡 Auto-reload enabled. Files will trigger browser refresh.
```

以下の方法でこういったルーチンワークはこなせますが、

- スクリプト化
  - ❌: ターミナルを占有してしまう
- systemd user service化
  - ❌: systemdが動いていないコンテナ内で動作しない
  - ❌: ユニットファイルを書くのがちょっと煩雑
- `command_name &`でバックグラウンド実行
  - ❌: ターミナルに結び付いてるからターミナルが終了したら一緒に終了する
  - ❌: ターミナルのジョブコントロール機能はちょっと煩雑
  - もちろん`command_name > /dev/null 2>&1 &`してから`disown -a`してもいいんですが、ルーチンワークをこなすためにしては長すぎるコマンドです

という感じで難点があります。

ちょっとアプリを

- 裏で動かして
- cliオプションを永続化したくて
- ログとかを確認したくて
- たまに停止とか開始をコントロールしたい

だけなんですが、その「だけ」が割と難しい

## Twisted Toughts

以前からぼんやりと下記のようなことをやりたいと思っていました

- [docker compose]みたいにyaml起動順序つきで複数のアプリの実行を定義したい
  - 前述の`ssh-add -T ...` -> `ssh -ND ... bastion`みたいなやつ
- 裏でアプリ動かしておいて、必要な時だけアタッチしてデタッチ
  - `claude code`の`/loop`を回しておいてたまに確認、とか
- アプリを実行しながらレイアウトを切り替え
  - windowの半分でneovimを開いて残りの半分にいろいろな情報を切り替えながら表示したり
- ログやアプリの実行状態をtuiから確認

役に立つとか既存のアプリでできるかどうかじゃなくてこういうのを作りたいってずっと考えてました。

昨今のLLMの性能ならこういうの、思ったように作れちゃうんじゃない？
ということで、どうせだし全部やっちゃうことにしました。

## prior arts

prior art紹介セクション

開発すること正当化するには既存のツールじゃ上記で上げた要求を満たせないことを示す必要があります。
実際には記事を書くにあたって調べてるだけで、開発してる間には知らなかったものがほとんどです。

- コンテナランタイム
  - 概説:
    - コンテナランタイムは、アプリを含めたFS Treeとそれを実行するときの設定ファイルを固めたもの(イメージ)をひな型に、[namespece(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)などで隔離された環境でアプリを動かすソフトウェア
    - `namepsce(7)`を使うため、基本はLinuxでしか動かない。Windows / MacではLinuxを動作させるためのVMを作ってその中でコンテナを動作させる
  - プロダクト:
    - [podman]
      - Redhat主体、daemonless, cli/APIはdocker互換
      - 開発言語: 本体は[Go]、[conmon](https://github.com/containers/conmon)などは[C]
    - [docker]
      - Docker Inc.主体、daemonプロセス、中央集権的configストア
      - 開発言語: [Go],CGO経由で[C]を呼び出しているところもある
- タスクキュー系
  - 概説:
    - コマンドをキューに入れて順繰りに実行する
  - プロダクト:
    - [pueue]
      - シェルスクリプトやコマンドをpushすると`pueued`が順繰り/平行に実行する
      - 開発言語: [Rust]
      - プラットフォーム: Linux/Mac/Windows
- ファイルでプロセス一覧を書く系
  - 概説:
    - ファイルにコマンドを書いてそれに基づいてプロセスが管理されるようなやつ
  - プロダクト:
    - [supervisord]
      - INI風のファイルでプロセスを管理できる
      - 開発言語: [python]
      - プラットフォーム: Linux/Mac/Solaris/FreeBSD(UNIX系全般)
    - [foreman] / [goreman]
      - `Procfile`ファイルに記述されたコマンドを一気にフォアグラウンドで実行
      - 開発言語: [foreman]は[Ruby], [goreman]は[Go]移植
      - プラットフォーム: プラットフォームへの言及無し。[Ruby]/[Go]が動く環境すべてで動くのだと思われる
    - [overmind]
      - `Procfile`ベース。各プロセスをtmuxセッション上で起動する
      - `overmind attach`は`tmux attach`のラッパー
      - 開発言語: [Go]
      - プラットフォーム: Linux/Mac/\*BSD
        - [tmux]に依存しているからなところもあるがコード自体もPOSIX系の挙動に依存しているようだ
    - [process-compose]
      - docker-compose風yamlでコマンドを起動
      - 疑似ターミナルを備えており、TUIで各コマンドの状態を閲覧可能
      - プラットフォーム: Linux/Mac/Windows

`podman`, `forman`/`goreman`以外すべてdaemonありのクラサバモデルみたいです。

セッション管理のみ

- [dtach] / [shpool]
  - どちらもGNU Screen / tmuxからターミナル機能を取り除いたセッション管理のみを意識していると記述
  - 開発言語: [C] / [Rust]

terminal multiplexerのレイアウト操作をするもの

- [vde-layout]
  - yamlで定義したレイアウトでtmux / WezTermのwindowを分割し、コマンドを実行
  - 開発言語: [TypeScript]\(というか[Node.js]互換ランタイム\)
- [tmuxinator]
  - yamlで定義したwindow / pane分割でセッションを管理できる。
  - 開発言語: [Ruby]

:::details LLMに調査させたら出てきたけど追い切れてないやつ

- 常駐スーパーバイザ系: [circus] / [immortal] / [runit] / [daemontools] / [s6] / [monit] /
  [systemd]`--user` — supervisordと同じく常駐サービスの監督向け。
- foremanクローン: [forego]\(Go\) / [node-foreman]\(Node\) / [honcho]\(Python\) /
  [prox]\(Go\)。[hivemind]はovermindのtmux非依存版。
- compose系の他実装: [mprocs]\(分割TUI\) / [devenv]\(Nix\) / [tilt] / [skaffold]\(k8s寄り\)。
- 軽量ジョブキュー: [task-spooler] / [nq] / pueueより単純なバッチ/並列実行。
- detach/reattachの仲間: [abduco]\(dtachの現代的再実装\) / [reptyr]。
- tmuxセッションマネージャ: [tmuxp] / [smug] / [sesh] — tmuxinatorと同系統。

:::

## 既存のツールじゃダメな理由

上記で上げたツール群は微妙に単体では私のニーズを満たしていないですね。

私の要求は

- (a)バックグラウンドでアプリを動作
- (b)yamlファイルで順序を記述することで、順序付きでアプリを起動
- (c)yamlファイルでレイアウトを記述することで、tmuxなどでアプリを特定のレイアウトで表示
- (d)tui管理

です。

- [process-compose]は(a), (b), (d)を満たせていそうですが(c)は無理そうに見える
- [overmind]と[vde-layout]/[tmuxinator]の組み合わせが(d)以外を満たせそうに見えますが、これら、組み合わせるにも工夫がいりそうです。

記事執筆にあたり調べているだけなので実は既存のものですべて満たせている可能性があるかもと思っていましたが、ドンピシャで満たすものはないのかもしれないです。

## 設計方針

基本的なアイデア: [podman]からコンテナ関連機能を抜かして簡易化し、[docker compose]相当の機能をその上に乗せ、レイアウト系は[vde-layout]もしくは[zellij]の記法を踏襲しています。

- 基本機能:
  - [podman]と同じくdaemonlessにします。
    - つまり事前にdaemonを起動するひと手間がいらないようにします。
    - コンテナ内で動作するllmにツールとして渡すことなどを考えると事前にdaemonを起動するようなワークフローは煩雑です。
- `compose`機能:
  - composeファイルの構文は[docker compose]とほぼ同じです。
    - やれることが少ないのでかなりサブセットになっています。
    - [docker compose]のcompose.yamlが`docker`コマンドのオプションを翻訳してファイルで書けるようになっている(厳密には違う[^1])のと同じく、`cmdman`の実行オプションをファイルで書いておけるフォーマットなので、`docker`に対して`docker compose`がこうなっているのと同じ理由で似たようなフォーマットになっています。
  - [docker compose]と同じく、基本のコマンド実行機能の上に、label（メタデータ)でプロジェクトを判別する構造にします。
    - [docker]はすべてのコンテナが単一のnamespace内でフラットに管理されます。
      - どのプロジェクトに参加しているコンテなのか、みたいな概念がないということです。
    - [docker compose]はプロジェクトの概念が導入され、
      - コンテナ名にプロジェクト名をprefixすることでプロジェクトの区別がつくようにし、
      - Label(コンテナに結びつくメタデータ)でどのプロジェクトとかを記録します。
    - namespaceの概念はあったほうがいいような気もしますが、この構造はかなり作りやすい！
    - 単一namepaceでフラットな管理+メタデータで区別のアイデアはそのまま採用します。
    - [docker compose]とは違い、特定のディレクトリに対して実行するコマンドセットという体になっています
      - 筆者のよくやるワークフローが特定のディレクトリで`neovim`, `claude code`, `codex`を開くことなので、これを作業ディレクトリごとにできると便利だなと思ってこの決断になっています。
      - `cmdman compose ps`などは`work_dir`に結び付いたプロジェクト一覧を表示することにしました。
      - コマンド名はプロジェクト名だけではなく、working directoryのハッシュ値をprefixすることとします。
    - `~/.config/cmdman/compose/`以下にあるcomposeファイルは名前のみで参照可能にします
- `mux`機能:
  - tmuxのpaneを分割するshellscriptを再帰構造のyamlにしたような構文を用いることとします。
    - この記法は[vde-layout]や[zellijのLayouts](https://zellij.dev/documentation/layouts.html)を参考にしました。
  - [tmux固有のlayout機能](https://github.com/tmux/tmux/wiki/Getting-Started#window-layouts)を使うことも考えられますが、同等機能が[zellij]/[WezTerm]に恐らくないため、拡張の際に困りそうなのでこういった構文にしました。
- `tui`機能:
  - 動いてるコマンドとかを確認できる`tui`です
  - `--popup`オプションで`tmux display-popup`内で動作するようにします。
  - `podman container ls`してから`podman container stop ${id}`するのがいつも面倒なので、`tui`で見ながら操作できると便利かなと思って追加することにしました。

[^1]: https://github.com/compose-spec/compose-spec#compose-specification より、`platform-agnostic`が明言されている。`docker`が`docker-compose`を開発していたころは`docker`コマンドをファイルで書いておけるものだったかもしれないが、のちに`compose-spec`として中立的な規格になった後からの追加は中立っぽくなっているように見える。

### 基本機能

LLMにmanページ風の説明を書かせたらいい感じだったので詳細は下記。

https://github.com/ngicks/cmdman/blob/main/doc/man/cmdman.1.md

基本機能部分は下記図のような感じ。

![](/images/making-cmdman/architecture.jpg)

`cmdman` cliコマンドは、いわゆる`monitor`プロセスを`daemonize`し、受け取った実行したいコマンドを子プロセスとして実行します。
`--tty`オプションがついている場合は`pty`(Pseudo teleTYpewriter = 疑似ターミナル)をallocateしてターミナル付きでアプリを起動します。

コマンドのログはlog-driverが`"none"`に指定されていない限り、log-driverを通じて書き込まれます。
現状は`k8s-file`しかありません。ログローテーションでraceがないように工夫しているので実装形態は異なると思いますが、発想は[podman]のパクリです。`podman`が`k8s-file`形式がデフォルトなのでそれがパクられています。

状態は`commands.db`というSQLite3ファイルに書き込まれます。これも[podman]と一緒。
コマンドが生成されたとか、開始したとか、終了したとか、削除されたとかはこのデータベースファイルが永続化します。

このイベント群のログは`event.log`にも書き込まれます。
これも[podman]の様式をそのままパクっていますが、実装形態は参考にしていないため異なります。こちらもrraceが起きないように気を遣っている。

### compose機能

https://github.com/ngicks/cmdman/blob/main/doc/man/cmdman-compose.1.md

https://github.com/ngicks/cmdman/blob/main/doc/man/cmdman-compose.5.md

[docker compose]と同じ発想でcomposeファイルをyamlで書いたら順序付きでコマンドが実行できるものです。

環境変数のinterpolation, afterでの順序付けなどができます。

```yaml
name: cmdman-compose-example
commands:
  yay1:
    args:
      - echo
      - yay
  yay2:
    env:
      - SLEEP_SECS=${INPUT_SLEEP:-3}
    args:
      - sleep
      - ${SLEEP_SECS}
  nay:
    args:
      - echo
      - nay
    after:
      yay1:
        condition: completed
      yay2:
        condition: completed
```

依存関係に基づいて起動順序が決まります。ロジックは`docker compose`とほとんど一緒のはず。

```
cmdman compose -f ./example.compose.yaml up
⠴ yay1             Starting
⠴ yay2             Waiting
◌ nay              Created
```

`yay2`コマンドが3秒ブロックするので、`nay`は3秒後に実行されます。

### mux機能

https://github.com/ngicks/cmdman/blob/main/doc/man/cmdman-mux.1.md

https://github.com/ngicks/cmdman/blob/main/doc/man/cmdman-mux.5.md

`mux`サポートは例えば下記のようなyamlを書き、

```yaml:devenv.yaml
name: devenv
commands:
  nvim:
    args:
      - nvim
    tty: true
  shell:
    args:
      - $HOME/.nix-profile/bin/zsh
    tty: true
  claude:
    args:
      - $HOME/.dotfiles/devenv_run.sh
      - ""
      - -lc
      - claude
    tty: true
  codex:
    args:
      - $HOME/.dotfiles/devenv_run.sh
      - ""
      - -lc
      - codex
    tty: true
mux:
  layouts:
    - name: default
      root:
        dir: h
        splits: [50%, 50%]
        panes:
          - dir: v
            splits: [1, 20%]
            panes:
              - nvim
              - shell
          - dir: v
            splits: [50%, 50%]
            panes:
              - command: claude
                focus: true
              - codex
    - name: claude-focused
      root:
        dir: h
        splits: [50%, 50%]
        panes:
          - dir: v
            splits: [1, 20%]
            panes:
              - nvim
              - shell
          - command: claude
            focus: true
    - name: codex-focused
      root:
        dir: h
        splits: [50%, 50%]
        panes:
          - dir: v
            splits: [1, 20%]
            panes:
              - nvim
              - shell
          - command: codex
            focus: true
```

`tmux`セッションの中で

```
$ cmdman compose -f ./devenv.yaml up
$ cmdman compose -f ./devenv.yaml mux 0
```

を実行すると、下記のように`tmux`のwindowが分割されて、それぞれpaneがそれぞれのコマンドにattachします。

![](/images/making-cmdman/mux-tmux.png)

_(黒塗り部分隠す意味ない気がするけど一応隠してる)_

```
cmdman compose -f ./devenv.yaml mux
```

を繰り返し実行すると、複数レイアウトある場合サイクルする仕様としています。

```
cmdman compose -f ./devenv.yaml mux 2
```

みたいな感じで数字インデックスか、レイアウト名を指定するとそのレイアウトが表示されます。

現在は[tmux]以外のdriverはサポートされていません。LLMに作らせるので別に作ってもいいんですが、一応自分でもチェックしてちゃんと動いてるかチェックしてるのでやろうと思うまでホールドされています。
実装上`tmux`でないと動かない部分があるのでその辺をどうするかも決める必要があります(後述)。

### tui機能

https://github.com/ngicks/cmdman/blob/main/doc/man/cmdman-tui.1.md

`tui`は[bubbletea]ベースの実装とします。

`tui`は下記のような感じ

```
cmdman tui --popup
```

![](/images/making-cmdman/tui-popup.png)

`--popup`オプションで`tmux popup`の中でtuiを表示します。

この機能は極めてexperimentalなのでのちに滅茶苦茶変えようと思っています。

コマンドログのPreviewセクションは`tty`が必要なターミナルアプリは正常に表示できず、ガチャガチャっとした表示になります。

![](/images/making-cmdman/tui-popup-scrambled-tty-output.png)

これをしようと思うと疑似ターミナルの実装が必要なので重いのでこの程度で止めてあります(後述)

## LLMの叩き方

### 概要

このアプリはほとんどLLMで作られています。

- [claude code]\: Opus 4.7 max / Opus 4.8 xhigh
  - Max5x($100/month)なのでこちらがメイン
- [codex]\: gpt-5.5 medium
  - Plus($20/month), ちょっと多めに実装タスク渡すとすぐ5hリミットが来る。

これらのツールはtuiからチャットで命令を出すとクラウド上のLLMエージェントが動作し、コードの編集、テストの実行などをおこい、自律的に実装や調査などを行うことができます。基本的な使い方の知識は所与のものとします。

これらは非常によくできたツールでout-of-boxでだいぶ良く動きますが、やはりなにかしかのカスタマイズを必要とします。

やった工夫についていろいろ書きます。

### apmによるパッケージ管理

[apm]は`claude`, `codex`その他もろもろ向けのskills/hooksなどなどをよそからインポートすることができるagent向けパッケージマネージャです。agents向けの[npm]とかそういうポジションのもの

> An open-source, community-driven dependency manager for AI agents.

基本的に信用ならないサードパーティのskillやhookをインポートするのは危険かと思いますが、自作なら問題ないだろということで自前のパッケージ群を用意してそれを利用しています。

https://github.com/ngicks/agents-package

agents向けの[npm]みたいなものという通り、`npm`でいうところの`package.json`にあたる`apm.y[a]ml`を定義して管理します。
[cmdman]の`apm.yml`は下記のように

https://github.com/ngicks/cmdman/blob/732a3fe0cc0c88803dc250fa6e6a8d4acc233f86/apm.yml

```shell
apm install --update -t claude,codex
apm compile -t codex
```

で`.claude/`, `.codex/`, `.agents/`, `AGENTS.md`などなどが作成されます。
`CLAUDE.md`は生成しないようにしています。`install`時点で`.claude/rules`が生成されるので必要ないためです。

[apm]の管理機能は

- 中央管理したskills / hooksをほうぼうで利用
- 複数のhooks / instructions(`AGENTS.md`のこと)を合成して単一の`settings.json`(hook定義)/`AGENTS.md`を生成

など便利です。
より良い代替ツールが出るまではこれを使えばいいと思います。

### AGENTS.mdでAskUserQuestionを使うように指示

https://github.com/ngicks/agents-package/blob/ed8e52745ad217cfe5f1e1dbed1abc1b714dbe46/instructions/base-env.instructions.md

> You may ask back the user to resolve unclear corners, using AskUserQuestion (if available) or just a response.

これが地味によいです。

`AskUserQuestion`は[claude code]の組み込みツールで、言葉通りユーザーに質問を行うツールです。特にplanモード中に使ってくると思いますが、instructionに加えておくと質問があるときに使ってくれるようになります。

画像を取り損なっているのでググってどういうものか見てください

https://www.google.com/search?q=askuserquestion

これがかなりいいです。
[codex]にはこのツールがないので利用できません。早く実装してくれないかなあ

### cliアプリ作成skill(go-edit-cobra)

以下です。

https://github.com/ngicks/agents-package/tree/ed8e52745ad217cfe5f1e1dbed1abc1b714dbe46/skills/go-edit-cobra

[Go]でサブコマンドありのリッチなcliアプリを作るとなると基本的に[github.com/spf13/cobra]か[github.com/urfave/cli]を用います。

筆者は[github.com/spf13/cobra]のほうを好んで使っています。基本的には多分こちらのほうがお勧め。

素の[claude code]でもいくらか`cobra`のプロジェクトを作成でききますが、いろいろカスタマイズできるところがあるので素のままだと[claude code]の機嫌次第でプロジェクトごとに構成が異なってしまいます。
ずれを抑えるためにはこういったskillが必要になってきます。

結局ノウハウや好みはすべてskillに書いてあるので、ここでは大雑把な話しにとどめます。大体以下をするように言っています。

- `cmd/<<app-name>>`以下に`main`を定義
- `package main`ではほぼ何もせずに、直下の`commands`にすべての動作を委譲
  - `main`でいろいろやると単体テストが書けなくなることへの反省
- `cmd/<<app-name>>/commands/root.go`にroot commandを定義
- `cmd/<<app-name>>/commands/<<subcommand-name>>.go`にサブコマンドを定義
- `cmd/<<app-name>>/commands/`以下では、パッケージトップスコープに変数を定義しない。
  - フラグ変数などはすべて関数内で定義。
    - こうしないとサブコマンドの単体テストが書きづらい
- `pkg/<<app-name>>`を定義し、アプリのロジックはすべてそこに委譲する
  - cliコマンドの表示に関するロジックは`pkg/<<app-name>>/cli`を定義し、そこにすべて委譲する
  - このように書いておかないと`cmd/`以下の層でロジックがいっぱい書かれて切り分けが変になる
  - `pkg`というサブパッケージを切るのは悪しき習慣と言われがちですが、ないとトップディレクトリが煩雑になる問題があってこうしています。(そのうちやめるかも)
- 常に[log/slog](https://pkg.go.dev/log/slog)でロガーを定義
  - `--log`, `--log-level`というグローバルフラグを常に定義し、これらのオプションを設定することで有効化する
  - `ctx context.Context`にロガーオブジェクトを収め、デバッグ用ロガーは必ずそこからとり出す
- 必ず`version`サブコマンドを用意すること`<<app-name>> --version`は`version`サブコマンドへのエイリアスとすること
  - `pkg/<<app-name>>/version.go`を用意し、ここにバージョンを埋め込む
  - `release`ヘルパーを定義し、これをプロジェクトにコピペする
    - `release`ヘルパーは`go run internal/cmd/release v0.0.1`みたいにすることで`version.go`の内容を`v0.0.1`に置き換てコミット -> `v0.0.1`のタグ付け -> `version.go`を`v0.0.2-devel`に置き換えてコミットという一連のバージョンダンスを行う
- 各種ヘルパーを定義してあるので必要に応じてコピーして利用する
  - ヘルパーはモジュールとして切り出してもよかったんですが、各アプリで独自に変化していく必要がある場合にモジュールだと不自然になるのでLLMにコピペさせる形式をとっています。
  - (1) `cmdsignals`:
    - `[...]os.Signal`
    - `main`パッケージ内で`signal.NotifyContext(context.Background(), cmdsignals.ExitSignals[:]...)`で、使用する
    - 定義理由:
      - graceful exit実装のために必要。ないと`Ctrl+c`(`SIGINT`)や`SIGTERM`時の挙動がいきなりプロセス終了になる。
      - 込み入ったcliアプリを作ろうとするとシグナルハンドラを取り除きたくなるケースがよくあるが、引数なしの[signal.Reset()](https://pkg.go.dev/os/signal#Reset)はすべてのハンドラをリセットしてしまうので依存先ライブラリの挙動を破壊しかねない。適当に絞るためのリストが必要だった。
        - `cmdman attach`は`Ctrl+c`をアタッチ先にフォワードするのでアタッチ開始時に`signal.Reset`必須
        - 依存先ライブラリがハンドラを使う例: [bubbletea]は`SIGWINCH`などをトラップする。
        - 書きながら思ったけどReset使わずに[Stop](https://pkg.go.dev/os/signal#Stop)使えばいいだけなんじゃ・・・(今後の課題)
  - (2) `loggerfactory`:
    - 前述の`--log`フラグのワイヤリング。
    - 環境変数(`<<APP_NAME>>_LOG_FORMAT`, `<<APP_NAME>>_LOG_LEVEL`)からでも設定できる
    - `OpenTelemetry`用のlog-level, `Trace` / `Fatal`も用意しておいた
  - (3) `versioninfo`:
    - [debug.ReadBuildInfo](https://pkg.go.dev/runtime/debug#ReadBuildInfo)から情報を読んでstructにまとめ直す
    - 前述の`version`サブコマンドから呼ぶ。
  - (4) `stdiopipe`:
    - stdin/stdout/stderrを[io.Pipe](https://pkg.go.dev/io#Pipe)経由で読めるようにしたもの。
    - `os.Stdin.Read`, `os.Stdout.Write`は、`os.Stdin`/`os.Stdout`の`Close`を呼ぶことで**アンブロックすることができません**
    - `Close`によるアンブロックを実現するにはいったんpipeをかませるしか現状方法がありません。
    - アンブロック出来なければ終了しないgoroutineが残されえます。
    - わかりにくいトラブルが起きないたようにするためにも、終了できないgoroutineは発生しないようにしましょう。

こんな感じ。
このskillは筆者の好みに合わせてガンガン変える予定なので、別に使うのを推奨しているわけですが、似たようなskillを作ることはお勧めです。
上記で上げた視点も多分いくらか役に立つのではないかと。

### Goのoutdatedな記法をチェックするskill(go-check-outdated-patterns)

### プロジェクト固有review skill(go-cmdman-review-checklist)

<!-- link section -->
<!-- MINE!!!! -->

[cmdman]: https://github.com/ngicks/cmdman

<!-- programming language -->

[Go]: https://go.dev/
[Java]: https://www.java.com/
[TypeScript]: https://www.typescriptlang.org/
[python]: https://www.python.org/
[C]: https://www.c-language.org/
[C++]: https://isocpp.org/
[Rust]: https://www.rust-lang.org
[Lua]: https://www.lua.org/
[Ruby]: https://www.ruby-lang.org/en/

<!-- runtimes -->

[Node.js]: https://nodejs.org

<!-- lib -->

[bubbletea]: https://github.com/charmbracelet/bubbletea
[github.com/urfave/cli]: https://github.com/urfave/cli
[github.com/spf13/cobra]: https://github.com/spf13/cobra

<!-- llm stuff -->

[claude code]: https://code.claude.com/docs/en/overview
[codex]: https://developers.openai.com/codex/cli
[Skills]: https://agentskills.io/home
[Hooks]: https://code.claude.com/docs/en/hooks
[agents-package]: https://github.com/ngicks/agents-package

<!-- my other articles -->

[go-os-root]: https://zenn.dev/ngicks/articles/go-os-root-based-abstraction
[pinentry]: https://zenn.dev/ngicks/articles/pinentry-in-tmux-popup

<!-- tools -->

[mise-en-place]: https://mise.jdx.dev/
[apm]: https://github.com/microsoft/apm
[npm]: https://www.npmjs.com/

<!-- prior arts -->

[tmux]: https://github.com/tmux/tmux/wiki
[zellij]: https://zellij.dev/
[WezTerm]: https://wezterm.org/index.html
[podman]: https://github.com/containers/podman
[docker]: https://www.docker.com/
[docker compose]: https://docs.docker.com/compose/
[pueue]: https://github.com/Nukesor/pueue
[pm2]: https://github.com/Unitech/pm2
[supervisord]: https://github.com/Supervisor/supervisor
[foreman]: https://github.com/ddollar/foreman
[overmind]: https://github.com/DarthSim/overmind
[process-compose]: https://github.com/F1bonacc1/process-compose
[dtach]: https://github.com/crigler/dtach
[tmuxinator]: https://github.com/tmuxinator/tmuxinator
[circus]: https://github.com/circus-tent/circus
[immortal]: https://github.com/immortal/immortal
[runit]: http://smarden.org/runit/
[daemontools]: http://cr.yp.to/daemontools.html
[s6]: https://github.com/skarnet/s6
[monit]: https://mmonit.com/monit/
[systemd]: https://systemd.io/
[forego]: https://github.com/ddollar/forego
[goreman]: https://github.com/mattn/goreman
[node-foreman]: https://github.com/strongloop/node-foreman
[honcho]: https://github.com/nickstenning/honcho
[prox]: https://github.com/fgrosse/prox
[hivemind]: https://github.com/DarthSim/hivemind
[mprocs]: https://github.com/pvolok/mprocs
[devenv]: https://github.com/cachix/devenv
[tilt]: https://github.com/tilt-dev/tilt
[skaffold]: https://github.com/GoogleContainerTools/skaffold
[task-spooler]: https://github.com/justanhduc/task-spooler
[nq]: https://github.com/leahneukirchen/nq
[abduco]: https://github.com/martanne/abduco
[shpool]: https://github.com/shell-pool/shpool
[reptyr]: https://github.com/nelhage/reptyr
[tmuxp]: https://github.com/tmux-python/tmuxp
[smug]: https://github.com/ivaaaan/smug
[sesh]: https://github.com/joshmedeski/sesh
[vde-layout]: https://github.com/yuki-yano/vde-layout
