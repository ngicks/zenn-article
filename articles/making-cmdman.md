---
title: "cmdman: バックグラウンドでアプリを動かすマネージャーをLLM叩いて作った話"
emoji: "😈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "cli", "tmux"]
published: false
---

## cmdman: バックグラウンドでアプリを動かすマネージャーをLLM叩いて作った話

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

### 各機能

[cmdman]はコマンドをバックグラウンドで実行できるアプリです。

- 基本機能:
  - life-cycle(`cmdman create`, `cmdman run`, `cmdman stop`, `cmdman rm`):
    - [podman]風なcliオプションで動くコマンドの設定や開始、終了のコントロール
  - `cmdman ls`:
    - アプリの一覧の閲覧
  - `cmdman send-keys`:
    - 任意の入力をstdinに送る
    - [tmux]のsend-keysと同じ様式
  - `cmdman attach`:
    - 手元のターミナルとコマンドのstdioを接続
- `compose`機能:
  - [docker compose]風のyamlファイルを書くことで複数のコマンドを実行
  - コマンド間の依存関係を記述可能で、順序をつけてアプリを起動可能
- `mux`機能:
  - `compose`のプロジェクト内ないしはスタンドアロンなyamlファイルを書くことで、[tmux]などのpaneを分割し、ダッシュボードを作成
  - 複数のレイアウトを記述しておき、サイクル可能
- `tui`機能
  - 上記を`tui`上から管理できます

みたいな感じ。

### 基本機能: 単コマンドをバックグラウンド実行

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

### compose機能

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

### mux機能

`mux`サポートは例えば下記のようなyamlを書き、

:::details 長いので省略

```yaml
name: devenv
commands:
  nvim:
    args:
      - nvim
    tty: true
  shell:
    args:
      - $HOME/.nix-profile/bin/zsh
      - -l
    stop_signal: SIGHUP
    tty: true
  claude:
    args:
      - $HOME/.dotfiles/devenv_run.sh
      - ""
      - -lc
      - claude
    scale: 3
    tty: true
  codex:
    args:
      - $HOME/.dotfiles/devenv_run.sh
      - ""
      - -lc
      - codex
    scale: 3
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
    - name: claude-focused-portrait
      root:
        dir: v
        splits: [50%, 50%]
        panes:
          - command: claude
            focus: true
          - dir: h
            splits: [1, 20%]
            panes:
              - nvim
              - shell
    - name: codex-focused-portrait
      root:
        dir: v
        splits: [50%, 50%]
        panes:
          - command: codex
            focus: true
          - dir: h
            splits: [1, 20%]
            panes:
              - nvim
              - shell
    - name: claude-two-right
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
                scale: 1
                focus: true
              - command: claude
                scale: 2
    - name: claude-3col
      root:
        dir: h
        splits: [1, 1, 1]
        panes:
          - command: claude
            scale: 1
            focus: true
          - command: claude
            scale: 2
          - command: claude
            scale: 3
    - name: codex-3col
      root:
        dir: h
        splits: [1, 1, 1]
        panes:
          - command: codex
            scale: 1
            focus: true
          - command: codex
            scale: 2
          - command: codex
            scale: 3
    - name: all-agents
      root:
        dir: h
        splits: [1, 1, 1]
        panes:
          - dir: v
            splits: [1, 1]
            panes:
              - command: claude
                scale: 1
                focus: true
              - command: codex
                scale: 1
          - dir: v
            splits: [1, 1]
            panes:
              - command: claude
                scale: 2
              - command: codex
                scale: 2
          - dir: v
            splits: [1, 1]
            panes:
              - command: claude
                scale: 3
              - command: codex
                scale: 3
```

:::

`tmux`セッションの中で

```
$ cmdman compose -f ./devenv.yaml up
$ cmdman compose -f ./devenv.yaml mux 0
```

を実行すると、下記のように`tmux`のwindowが分割されて、それぞれpaneがそれぞれのコマンドにattachします。

![](/images/making-cmdman/mux-tmux.png)

_(黒塗り部分隠す意味ない気がするけど一応隠してる)_

以下のコマンドで`compose` spec上の各コマンドの同時実行数を増やしたりできます。

```
$ cmdman compose -f devenv scale codex=5
```

コマンドは末尾に`-1`とか`-2`みたいなscale indexがつくのでこれで区別されます。(`docker compose`と同じような仕様)

`mux`機能は、yaml側で表示するscale indexをopt-inで指定できます。  
指定がない場合はデフォルトではscale index 1が表示され、`cycle-scale`入れ替えることができます。

```
$ cmdman compose -f ./devenv.yaml mux cycle-scale codex
```

割と便利！！！

### tui機能

`tui`は下記のような感じ

```
cmdman tui --popup
```

![](/images/making-cmdman/tui-popup.png)

`--popup`オプションで`tmux popup`の中でtuiを表示します。

とりあえず作っただけでこれでなにすべきかは考え中です。  
`mux`のレイアウトを選ぶのに使おうかなあ

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
❯ cmdman version
version:     v0.0.12
go version:  go1.26.3
```

Native Linux / Macでも動くと思うけど(特にMacで)なにも試していない。
Github Actionsでテストぐらいは走らせようかなあ

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

上記で上げたツール群は微妙に単体では私のニーズを満たしていません。

私の要求は

- (a)バックグラウンドでアプリを動作
- (b)yamlファイルで順序を記述することで、順序付きでアプリを起動
- (c)yamlファイルでレイアウトを記述することで、tmuxなどでアプリを特定のレイアウトで表示
- (d)tui管理

です。

### 既存ツールの組み合わせ

- [process-compose]は(a), (b), (d)を満たせていそうですが(c)は無理そうに見える
- [overmind]と[vde-layout]/[tmuxinator]の組み合わせが(d)以外を満たせそうに見えますが、これら、組み合わせるにも工夫がいりそうです。

記事執筆にあたり調べているだけなので実は既存のものですべて満たせている可能性があるかもと思っていましたが、ドンピシャで満たすものはないのかもしれないです。

### そもそもtmuxだけで実現できてね？

manより

```
man tmux
```

以下の2コマンドがあります。

```
     break-pane [-abdP] [-F format] [-n window-name] [-s src-pane] [-t dst-window]
                   (alias: breakp)
             Break src-pane off from its containing window to make it the only pane in dst-window.  With
             -a or -b, the window is moved to the next index after or before (existing windows are moved
             if necessary).  If -d is given, the new window does not become the current window.  The  -P
             option  prints  information about the new window after it has been created.  By default, it
             uses the format ‘#{session_name}:#{window_index}.#{pane_index}’ but a different format  may
             be specified with -F.

```

```
     join-pane [-bdfhv] [-l size] [-s src-pane] [-t dst-pane]
                   (alias: joinp)
             Like split-window, but instead of splitting dst-pane and creating a new pane, split it  and
             move  src-pane  into  the  space.   This  can be used to reverse break-pane.  The -b option
             causes src-pane to be joined to left of or above dst-pane.

             If -s is omitted and a marked pane is present (see select-pane -m), the marked pane is used
             rather than the current pane.
```

フォアグラウンド用のセッション、バックグラウンド用のセッションの二つを用意し、  
これらのコマンドを駆使してレイアウトなどの変更を行うことで所望の挙動は満たせてそうに思います。

しいて言うなら

- [tmux]/[zellij]などに強く結びつき、共通部がくくりだしにくいこと
- コマンドのログの保持がやりづらいこと
- コマンドの動作/停止状態の監視、restart policyの実装が難しいこと

などがこの方法をとらない理由といえます。

## 設計方針

作り出す前に考えてた方針です。結局これらの想定を覆すようなことは起きなかったので今もこの通りになっています。

### 概要

[podman]と同じくdaemonlessにすることにしました。

- つまり`cmdman run`などがコマンドモニターをspawnしてdaemonizeし、与えられたコマンドはモニターは子プロセスとして実行されます。
- コンテナ内を含めた様々な状況で使うことを考えたため、「daemonをあらかじめ起動しておく」というワンステップを排除したかった意図があります。
- コマンドモニターとの通信は[gRPC]で行うことにしました。
  - schema中心かつcode generator前提なので楽でいいな
  - JSONを用いるIPCも考えられますが、JSONの取り扱いはプログラミング言語によってばらばらなので避けています

[docker compose]を見習って、基本機能をラップする形で複数コマンドをまとめて管理できる機能を付けます。

- `cmdman`のcliオプションを変換したようなフォーマットのyamlを書くことでコマンドをまとめて実行できます。
- `docker compose`と同じく、`cmdman`基本機能はフラットにコマンドを管理しまし、`compose`はそれらにメタデータでコマンドをグルーピングし、プロジェクトごとにまとめて管理します。

[zellijのLayouts](https://zellij.dev/documentation/layouts.html)や[vde-layout]に似た構文で、`tmux`のpaneを分割してそれぞれのpaneでコマンドに`attach`することでダッシュボードを構成します。

- `tmux`には組み込まれたlayout機能はありますが(prefix + alt+数字キーとかで組み込まれたlayoutを呼び出せます。)、フォーマットが独自すぎて、それありきのフォーマットは人も書けなさそうだし、`tmux`以外のドライバへの拡張性がなさそうなのでやめときます。
- `zellij`のlayoutも`vde-layout`の構文もpaneの分割を何対何で分割して何を表示するかというのを入れ子構造で記述します。このフォーマットはシンプルでわかりやすいと判断しました。
- layoutの入れ替えで迅速にコマンドの表示領域を変更できるようにします。これでようやく単に`tmux`のpane分割よりもなにか便利なものといえると思ったからです。

`tui`でコマンドのログとかを見たかったので、`tui`機能も付けることにしました。

- Goで`tui`を作るということはほぼ[bubbletea]を使うということです。
  - この重い依存性をぶっこむのは気が引けるので`tui`は別モジュールにして、別バイナリにするか悩みましたが、シングルバイナリじゃないと私自身の管理が面倒なのでとりあえずシングルバイナリになってます
  - とはいえ現状で`CGO_ENABLED=0`のstatic binaryで29MiB低度なのでまあいいかって感じですね。
- `--popup`オプションをつけると`tmux`の`display-popup`内部で動作します。
  - `mux`機能はそもそもpane状態を変更していしまうのでpopupないからlayoutを変更その他できるものがないと少々使い勝手が悪いかなということでこれを実装することにしました
    - しかして`tui`からlayout変更はまだ実装していません!

### 基本機能

LLMにmanページ風の説明を書かせたらいい感じだったので詳細は下記。

https://github.com/ngicks/cmdman/blob/main/doc/man/cmdman.1.md

動作の仕組みは下図のような感じ。

![](/images/making-cmdman/architecture.webp)

- `cmdman` cliコマンドは、いわゆる`monitor`プロセスを`daemonize`し、受け取った実行したいコマンドを`monitor`の子プロセスとして実行します。
  - (読者のレベル感の設定がよくわからないので、釈迦に説法、もしくは不十分な説明でしかない気がしてならないですが)Linuxなどにおいて、コマンドはそれを呼び出しているターミナルが終了するとシグナル`SIGHUP`を受け取り、`SIGHUP`のデフォルトハンドラによって即時終了します。  
    [podman]のようなアプリ(コンテナ)をバックグラウンドで実行し、呼び出したターミナルが終了しても処理が続行できるようにするには`daemonize`とよく呼ばれる一連の処理をする必要があります。
- `monitor`プロセスはunix domain socketをlistenし、[gRPC]サーバーを立てて`cmdman stop`などが行うIPCを待ち受けます。
  - IPCに[gRPC]を用いることにします。  
    `gRPC`は[Protocol Buffers]というschema中心のRPCフォーマット/ツールのエコシステムと言えばよいでしょうか。  
    HTTP/2, /3をトランスポートとし、双方向通信のRPCをいい感じに実装可能です。元からコードジェネレーターありきで成り立っている仕組みなので、schemaを定義するとclient / server stubを各種言語向けに生成可能であるため、めんどうなclient作成の作業がスキップ可能です。LLMに使わせる場合でもコードジェネレーターがガードレールとして機能するため更新が漏れにくくなってりしていいと思います。
  - `JSON`を用いたRPCも考えられますが、`JSON`の取り扱いがプログラミング言語によってばらばらで落とし穴が割とあったので避けています。
- `--tty`オプションがついている場合は`pty`(Pseudo teleTYpewriter = 疑似ターミナル)をallocateしてターミナル付きでアプリを起動します。  
  `pty`には[github.com/creack/pty](https://github.com/creack/pty)を用います。おそらくもっとも有名な`pty`ライブラリかなと思います。`v2`系のタグも存在しますが、事故で間違えてつけてしまっただけらしいので`v1`系を使いましょう。
  - アプリはしばしばそれがターミナルで実行されているのか、スクリプトなどから呼び出されているのかなどを確認するために、stdinやstdoutがターミナルかチェックするものがあります。そういったアプリはファイルなどに出力がredirectされている際には、ファイルで読みやすいように出力内容を変えるようなことをします。  
    実際のターミナルに接続していないバックグラウンドで実行されているアプリをどうやってターミナルで実行しているかのように見せかけるかというと、`pty`としてターミナルを取得してアプリのstdin, stdout, stderrをそれに差し替えることで行います。`tmux`の`client`が`pts`などを示すのはそういう理由です。詳しくはAdvanced Programming in the Unix Environmentとか読んでください。[line discipline](https://en.wikipedia.org/wiki/Line_discipline)がどうの言う図の部分で混乱して筆者はいまいちよくわかりませんでした。
- コマンドのログはlog-driverを通じてディスクに書き出されます。  
  現状は`k8s-file` log-driverしかありません。[podman]のデフォルトロギングドライバです。誰がどういう経緯で`k8s`(=`kubernetes`)と名付けたのかいまいちわかりません。
  - ログローテーション時にrace conditionが起きないようになど、いろいろ私の事情でフォーマット改変しちゃってるから完全同じ仕様ではなくなって様な記憶がります。

- コマンドの起動、実行など、現在の状態は`commands.db`というSQLite3ファイルに書き込まれます。これも[podman]の挙動をそのまままねています。
  - `cmdman ls`などはここを参照します。`cmdman attach`, `cmdman send-keys`などがIPCを行うためにUnix domain socketのパスもここに記録されます。
- 起動、実行、終了などの状態変化のログは`event.log`に書き込まれます。
  - これも[podman]の方式をパクってます。`podman`と違ってこちらは`journald`に書き出す設定はなく、`file`ドライバオンリーです。  
    `cmdman events`はこのログファイルを[inotify(7)](https://man7.org/linux/man-pages/man7/inotify.7.html)などで監視して更新を取り込みます。

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

動作モデルは[docker compose]と大体同じ:

- 起動: yamlで書かれた`compose spec`からDAG(directed acyclic graph=向きあり循環なしグラフ)を作成し、rootノードから順繰りに実行。`after`に書かれた条件を満たせなければエラー、みたいな感じ。
- recreation: コマンドのメタデータに`compose spec`のハッシュ値を記録して起き、これを照合することで`compose spec`の更新を検知してrecreateを走らせます。

### mux機能

https://github.com/ngicks/cmdman/blob/main/doc/man/cmdman-mux.1.md

https://github.com/ngicks/cmdman/blob/main/doc/man/cmdman-mux.5.md

`mux`サポートは例えば下記のようなyamlを書き、

:::details 長いので省略

```yaml
name: devenv
commands:
  nvim:
    args:
      - nvim
    tty: true
  shell:
    args:
      - $HOME/.nix-profile/bin/zsh
      - -l
    stop_signal: SIGHUP
    tty: true
  claude:
    args:
      - $HOME/.dotfiles/devenv_run.sh
      - ""
      - -lc
      - claude
    scale: 3
    tty: true
  codex:
    args:
      - $HOME/.dotfiles/devenv_run.sh
      - ""
      - -lc
      - codex
    scale: 3
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
    - name: claude-focused-portrait
      root:
        dir: v
        splits: [50%, 50%]
        panes:
          - command: claude
            focus: true
          - dir: h
            splits: [1, 20%]
            panes:
              - nvim
              - shell
    - name: codex-focused-portrait
      root:
        dir: v
        splits: [50%, 50%]
        panes:
          - command: codex
            focus: true
          - dir: h
            splits: [1, 20%]
            panes:
              - nvim
              - shell
    - name: claude-two-right
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
                scale: 1
                focus: true
              - command: claude
                scale: 2
    - name: claude-3col
      root:
        dir: h
        splits: [1, 1, 1]
        panes:
          - command: claude
            scale: 1
            focus: true
          - command: claude
            scale: 2
          - command: claude
            scale: 3
    - name: codex-3col
      root:
        dir: h
        splits: [1, 1, 1]
        panes:
          - command: codex
            scale: 1
            focus: true
          - command: codex
            scale: 2
          - command: codex
            scale: 3
    - name: all-agents
      root:
        dir: h
        splits: [1, 1, 1]
        panes:
          - dir: v
            splits: [1, 1]
            panes:
              - command: claude
                scale: 1
                focus: true
              - command: codex
                scale: 1
          - dir: v
            splits: [1, 1]
            panes:
              - command: claude
                scale: 2
              - command: codex
                scale: 2
          - dir: v
            splits: [1, 1]
            panes:
              - command: claude
                scale: 3
              - command: codex
                scale: 3
```

:::

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

以下のコマンドで`compose` spec上の各コマンドの同時実行数を増やしたりできます。

```
$ cmdman compose -f devenv scale codex=5
```

コマンドは末尾に`-1`とか`-2`みたいなscale indexがつくのでこれで区別されます。(`docker compose`と同じような仕様)

`mux`機能は、yaml側で表示するscale indexをopt-inで指定できます。  
指定がない場合はデフォルトではscale index 1が表示され、`cycle-scale`入れ替えることができます。

```
$ cmdman compose -f ./devenv.yaml mux cycle-scale codex
```

中立的な`mux spec`に基づいてpaneを分割します。
そこで複数のterminal multiplexerをサポートできるように中立的なinterfaceを定義し、[tmux]など向けの実装を作る形にします。

...といいつつ現在は[tmux]以外向けのドライバは実装されていません。  
terminal multiplexerのwindowにドライバ固有の値や、今どのレイアウトを表示しているかなどを埋め込む必要がありますが、`zellij`や`wezterm`では難しいので現在は実装していません(後述)。

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

本記事でいうこれらはターミナルアプリです。同名のデスクトップアプリも存在します。デスクトップアプリから使ってる人も多そうに感じますのでそっちもいいかも。

これらのツールはLLMエージェントがあなたのPCを操作できるようにする橋渡し役のようなもので、よく教育されているのか、out-of-boxでだいぶいい感じにソフトウェアを作成することができます。  
ただし何をやるにしても、好みや細かな拘束条件に合わせて変えなければならないところが出てきます。いわゆるmoving partsというやつです。  
LLMエージェントは指示があいまいでも行間を保管していい感じに作ってくれますが、その補完のしかたは時々によって異なります。似たような指示で似たようなものを４～５個作ってみてください。多分何回かはぶれて作り方が変わるところが出てくると思います。

そこでそれらがある程度固定されるようにしたり、同じ改善指示を何度もしなくていいように、ある程度のカスタマイズやツールの整備が必要となります。  
以下ではそれらについて述べます。

### apm(agents向けのパッケージマネージャー)でskills/hooksを管理

https://github.com/microsoft/apm

[npm]とか[uv]のagents向け版みたいなやつで、よそのgit repositoryから[skill][Skills]や[hook][Hooks]をインポートして管理したりするものです。

> An open-source, community-driven dependency manager for AI agents.

この記事をここまで読んでいる人対して[Skills]や[Hooks]の説明は必要はないかと思いますが一応説明すると、

- [Skills]は`./.claude/skills/`などに格納される`SKILL.md`を含むディレクトリのことです。
  - `SKILL.md`に特定のタスクを行うための知識や手順を記述して起き、ディレクトリにスクリプトや関連するリソースをまとめておきます。
  - `SKILL.md`はfrontmatterでname, descriptionなどを記述可能です
  - descriptionは起動時にロードされます。
  - `SKILL.md`の本文は使用時にロードされます。
  - descriptionの内容からLLMはそれを使用すべきかを判断し、自動的に使用します。
  - もしくは`/<name>`で明示的に呼び出すこともできます。
- [Hooks]は、`JSON`で記述され、LLM agentのToolUseや各種ライフサイクルをhookしてコマンドを実行可能です。

再配布可能な割に管理の方法そのものは定義されていなかったので[apm]がそのニッチを埋めたんだと思います。

再利用可能なように下記で自前のパッケージ群を管理することにしています。

https://github.com/ngicks/agents-package

[cmdman]の`apm.yml`は下記のように

https://github.com/ngicks/cmdman/blob/897a4ac18d5b53fd4943bfeeeda5a4d744415f0e/apm.yml

これがある状態で下記で依存先をインストール可能です。

```shell
apm install --update -t claude,codex
apm compile -t codex
```

`apm install`で`.claude/`, `.codex/`, `.agents/`などが作成れます。`claude`を対象に取るとinsructionsは[rules](https://code.claude.com/docs/en/memory#organize-rules-with-claude/rules/)に変換されます。

`apm compile -t codex`でinstructionsを統合して`AGENTS.md`が出力されます。  
`claude`には`rules`がすでに出力されているので`compile`は`codex`のみを対象とします。

### AGENTS.mdでAskUserQuestionを使うように指示

https://github.com/ngicks/agents-package/blob/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/instructions/base-env.instructions.md

> You may ask back the user to resolve unclear corners, using AskUserQuestion (if available) or just a response.

これが地味によいです。

`AskUserQuestion`は[claude code]の組み込みツールで、言葉通りユーザーに質問を行うツールです。特にplanモード中に使ってくると思いますが、instructionに加えておくと質問があるときに使ってくれるようになります。

画像を取り損なっているので見た目は下記の記事などを参照。

https://zenn.dev/rakuten_tech/articles/claude-code-askuserquestion

これがかなりいいです。
[codex]にはこのツールがないので利用できません。早く実装してくれないかなあ

### go-edit-cobra: Goのcliアプリ作成/編集skill

以下です。

https://github.com/ngicks/agents-package/tree/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/skills/go-edit-cobra

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
    - graceful exit用の`[...]os.Signal`を定義するだけ
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

https://github.com/ngicks/agents-package/blob/ed8e52745ad217cfe5f1e1dbed1abc1b714dbe46/skills/go-check-outdated-patterns/SKILL.md

### プロジェクト固有review skill(go-cmdman-review-checklist)

https://github.com/ngicks/cmdman/blob/637fd046b8e87c37804f84aa41a6d0045caad0a9/.apm/skills/go-cmdman-review-checklist/SKILL.md

### カスタムlinter: オレオレvettoolで任意のlinterを作成

pkg.go.dev/golang.org/x/tools/go/analysis

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
[apm]: https://github.com/microsoft/apm
[Skills]: https://agentskills.io/home
[Hooks]: https://code.claude.com/docs/en/hooks
[agents-package]: https://github.com/ngicks/agents-package

<!-- tools -->

[mise-en-place]: https://mise.jdx.dev/
[uv]: https://docs.astral.sh/uv/
[npm]: https://www.npmjs.com/
[gRPC]: https://grpc.io/
[Protocol Buffers]: https://protobuf.dev/
[OpenAPI spec]: https://spec.openapis.org/

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
