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
- `daemonize`だの`pty`だの言われてもなんとなくわかる
- [Go]の読解
  - ほかのプログラミング言語が読み書きできればなんとなくわかる気も
- [claude code] / [codex]の使い方
  - [Skills]だの[Hooks]だの言われてもなんとなくわかること

モチベーション面: overview(何の話をするか)も参考に

- こういうアプリを作るときのコアコンセプトとか気を付けどころを知りたい
- [Go]でLLMのプロジェクトを動かすときの苦しみともがきを見たい
  - しかしてそこまで高度なことはしていない

## 用語・前提解説

以降で所与のものとされる知識なりなんなりの説明。わかってる人は飛ばしていい。

### daemonize

普通ssh先とかでコマンドを呼び出したとき実行中にsshの接続が切れたらコマンドって止まりますよね。これは呼び出したコマンドが接続しているsshの子プロセスだからです。  
`daemonize`は親プロセスを`/init`にしたり、stdin, stdout/stderrの接続を呼び出しターミナルから切ることでバックグラウンドにプロセスを回すことを言います。

`libc`に[daemon(3)](https://man7.org/linux/man-pages/man3/daemon.3.html)としてこの機能が実装されています。

普通は以下の手順を踏みます。

- [fork(2)]
- [setsid(2)]
- [fork(2)]
- stdin/stdout/stderrを含めてすべての`fd`を`close`
- stdin/stdout/stderrに`/dev/null`をセット

２回`fork(2)`を呼ぶのでこれをdouble-forkとよく呼びます。

Advanced Programming in the UNIX Environment (Addison-Wesley Professional Computing Series), W. Richard Stevensによると２回目の`fork(2)`はSVR4(System V Release 4)準拠システムではしたほうがよい（とアドバイスされることもあるだろう、という言い回し）。  
現代でいうとSolarisがSVR4にあたる。  
基本はいらないけどポータビリティを意識してるコードではdouble-forkをする、って感じ、多分。

[fork(2)]: https://man7.org/linux/man-pages/man2/fork.2.html
[setsid(2)]: https://man7.org/linux/man-pages/man2/setsid.2.html

### pty

PTY = Pseudo teleTYpewriter = 疑似端末

プログラムを動作させるとき、stdin, stdout, stderrっていう入出力のチャネルが存在すると思いますが、ここにptyを接続すると、プルグラムにターミナルにつながっていると思わせることができます。  
あらゆるターミナルエミュレーター(Windows Terminal、WezTerm, etc)はptyを使ってターミナルとしてふるまっています。

プログラムがさらにptyを使用してターミナル風にふるまうこともできます。VS Codeの組み込みターミナルや、[tmux]がpane分割ができるのもptyをallocateしているからです。

stdin, stdout, stderrはターミナルでないときがあります。`cat foo | bar > baz`とすれば`bar`プログラムのstdinは`foo`, stdoutは`baz`となります。
ターミナルアプリは、例えば`vim`のようにターミナル画面全体をコントロールしたり、文字に色を付けたりすることがあります。  
単にファイルである`stdout`には「画面」はないわけですから、全体をコントロールするという概念そのものがありません。  
ptyが必要なのはこういう画面という概念があって特殊なシーケンスを書き込むと操作できたりするから、だと思う。  
筆者の不勉強によりこれ以上は書けません。

ptyはmaster - slaveのペアです。プログラムのstdin,stdout,stderrに渡して読み書きしてもらう側と、実際にターミナルとして画面を描画する側で相互にやり取りするわけですから、ペアになる必要があります。ペアにならないといけない理由は[pipe(2)](https://man7.org/linux/man-pages/man2/pipe.2.html)と似ていますね。

[pty(7)](https://man7.org/linux/man-pages/man7/pty.7.html)によると、歴史的経緯とシステム間(BSD系、Solaris系、Linux、etc)でいろいろ異なるインターフェイスの形態があるみたいですが、  
[pts(4)](https://man7.org/linux/man-pages/man4/pts.4.html)によるとLinuxでは`/dev/ptmx`を開いてmasterを得て、library function[ptsnane(3)](https://man7.org/linux/man-pages/man3/ptsname.3.html)でslaveの名前を得るようです。

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
  `pty`には[github.com/creack/pty]を用います。おそらくもっとも有名な`pty`ライブラリかなと思います。`v2`系のタグも存在しますが、事故で間違えてつけてしまっただけらしいので`v1`系を使いましょう。
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

### パッケージ管理

[apm]を使って管理する

#### apm(agents向けのパッケージマネージャー)でskills/hooksを管理

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

- `Skills`は`./.claude/skills`や`.agents/skills/`にそのままコピーされます
- `Hooks`は合成されて１つの`.claude/settings.json`や`.codex/hooks.json`が出力されます
  - `hooks`が相対パスでスクリプトを参照している場合、`./.claude/hooks`以下にスクリプトをコピーしたうえでそこを使うようにパスを差し替えるようです。

再利用可能なように下記で自前のパッケージ群を管理することにしています。

https://github.com/ngicks/agents-package

`package.json`や`pyproject.toml`に当たるものは`apm.yaml`となります。  
[cmdman]の`apm.yml`は下記

https://github.com/ngicks/cmdman/blob/897a4ac18d5b53fd4943bfeeeda5a4d744415f0e/apm.yml

これがある状態で下記で依存先をインストール可能です。

```shell
apm install --update -t claude,codex
apm compile -t codex
```

`apm install`で`.claude/`, `.codex/`, `.agents/`などが作成れます。`claude`を対象に取るとinsructionsは[rules](https://code.claude.com/docs/en/memory#organize-rules-with-claude/rules/)に変換されます。

`apm compile -t codex`でinstructionsを統合して`AGENTS.md`が出力されます。  
`claude`には`rules`がすでに出力されているので`compile`は`codex`のみを対象とします。

### Instructions

Instructionsは`AGENTS.md`とか`CLAUDE.md`とかのことをさします。

- 基本的に何も書かない
  - コンテクストがある程度長くなると忘れる傾向にあるので、セッション開始時に知っておくとお得なごく限られた情報のみ書いたほうがいいと思われる
- [apm]\: ファイル名に`.instructions.md`のサフィックスをつける

#### AskUserQuestion

https://github.com/ngicks/agents-package/blob/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/instructions/base-env.instructions.md

> You may ask back the user to resolve unclear corners, using AskUserQuestion (if available) or just a response.

これが地味によいです。

`AskUserQuestion`は[claude code]の組み込みツールで、言葉通りユーザーに質問を行うツールです。特にplanモード中に使ってくると思いますが、instructionに加えておくと質問があるときに使ってくれるようになります。

画像を取り損なっているので見た目は下記の記事などを参照。

https://zenn.dev/rakuten_tech/articles/claude-code-askuserquestion

これがかなりいいです。
[codex]にはこのツールがないので利用できません。`codex`に対しても同じ`instructions`を用いたい場合、`AskUserQuestion`がなければ単にchat messageにフォールバックしなさいと書いておいたほうがいいでしょう。早く実装してくれないかなあ

### Skill

#### go-edit-cobra: Goのcliアプリ作成/編集skill

ある意味このskill自体がこのblogpostの核心かもしれない。  
LLMにcliアプリを何度も作らせて、必ずほしい機能は何なのかとか、どういう構成をとってほしいのかとかの個人的な好みと失敗が凝集して出来上がった文章です。

以下です。

https://github.com/ngicks/agents-package/tree/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/skills/go-edit-cobra

[Go]でサブコマンドありのリッチなcliアプリを作るとなると基本的に[github.com/spf13/cobra]か[github.com/urfave/cli]を用います。

筆者は[github.com/spf13/cobra]のほうを好んで使っています。基本的には多分こちらのほうがお勧め。

素の[claude code]でもいくらか`cobra`のプロジェクトを作成でききますが、いろいろカスタマイズできるところがあるので指示が不明瞭だと`claud`の機嫌次第でプロジェクトごとに構成が異なってしまいます。
ずれを抑えるためにはこういったskillが必要になってきます。

結局ノウハウや好みはすべてskillに書いてあるので、ここでは大雑把な話しにとどめます。大体以下をするように言っています。

以下のファイル配置をとるように指示します。

```
<module-root>/
├── go.mod
├── cmd/
│   └── <name>/
│       ├── main.go
│       └── commands/
│           ├── root.go                  # rootCmd() + Execute(ctx) + runRoot
│           ├── version.go               # always present; "version" subcommand + --version alias
│           ├── config.go                # always present; "config" subcommand (prints resolved config)
│           ├── zz_<helper>.go           # any non-subcommand helper file (prefix zz_)
│           ├── <subcmd>.go              # one per flat leaf
│           ├── <parent>.go              # one per group (no RunE)
│           └── <parent>_<child>.go      # one per nested leaf (zz-prefix a trailing GOOS/GOARCH/test leaf: foo_windows.go → foo_zzwindows.go)
├── internal/
│   ├── cmdsignals/
│   │   └── signals.go                   # always present
│   ├── stdiopipe/                       # only when a subcommand needs cancellable stdio
│   │   └── stdiopipe.go
│   ├── cmd/
│   │   └── release/
│   │       └── main.go                  # always present; cross-platform release helper
│   ├── loggerfactory/
│   │   └── loggerfactory.go             # always present; --log / --log-level wiring
│   ├── templateutil/
│   │   └── templateutil.go              # always present; shared text/template FuncMap + FuncDocs (json, + project helpers)
│   └── versioninfo/
│       └── versioninfo.go               # always present; ReadVersionInfo (Version + VCS info)
└── pkg/
    └── <name>/
        ├── version.go                   # always present; release-controlled `const Version`
        ├── config.go                    # always present; Config + DefaultConfig, PartialConfig + Apply, LoadConfig
        ├── <service>.go                 # internal service implementation
        └── cli/                         # CLI-presentation code (printing, prompts, tables, colors)
            ├── config.go                # always present; RenderConfig (config subcommand JSON / --format)
            └── <ui>.go
```

- `cmd/<name>`に`main`を定義
  - => これは割と普通の慣習。よく見る。
- `cmd/<name>/commands`にすべて委譲。
  - `func main`はほぼ`commands.Execute(ctx)`を呼ぶだけ
  - => mainが膨らんでテストしにくくなることへの反省
    - e2eありきでmainにいろいろ書くのもありだとは思う。
- `cmd/<<name>>/commands/`以下で
  - `root.go`でrootコマンドを定義
  - `<<subcommand-name>>.go`でそれぞれサブコマンドを定義
    - [github.com/spf13/cobra-cli](https://github.com/spf13/cobra-cli)と異なり、すべてのコマンド定義は関数内で行う。
      - => `var (...)`でモジュールトップにコマンド定義を書くと、サブコマンド定義部分のみの単体テストが書きにくくなる。
  - 現状はすべての子コマンド、孫コマンドはフラット配置。別ディレクトリに分けてもいいという風に文言を変えるかも。
- 実際のアプリケーションロジックは`./pkg/<name>`にすべて委譲
  - => `./cmd`以下が膨らんでくるとテストを書きにくくなることへの反省
  - => `./pkg`という意味のないパスを一段噛ませるのは、モジュールトップディレクトリにモノがあふれるのを防ぎたいだけです。
    - インポートする際にモジュールパスに不要なものが入ってくることになるので、良くない慣習かも
    - `./pkg`以下にいろいろ定義する慣習自体は`docker`, `docker compose`などで見られます。
  - `./pkg/<name>`のname部分はGoらしくなるように`-`や`_`を削除する
    - `package <name>`で宣言した部分がインポートされた際にパッケージにアクセスするidentifierになります。
    - Goのidentのルール上`-`は持てません。
    - `_`はidentに含まれ得ますが、慣習上いれません
- ターミナルへ結果やログをプリントする部分は`./pkg/<name>/cli`にすべて委譲
  - => `./cmd`以下が膨らんでくるとテストを書きにくくなることへの反省
    - とにかく`./cmd`以下を薄くしたい
  - わざわざこう書いていることからわかる通り、なにも指示をしないと`./cmd`以下にログやプリントのロジックも書かれてしまいます。
- アプリのconfing loadロジックを定義
  - `${XDG_CONFIG_HOME:-$HOME/.config}/<name>/config.{json,yaml,yml}`を読み込むように設定
  - フラグ > 環境変数 > configファイル > デフォルト値の優先順位で合成するように指示。
  - 環境変数は[github.com/caarlos0/env](https://github.com/caarlos0/env)で読み込むように指示。信頼と実績👍
  - `cobra`に組み込まれた[github.com/spf13/viper](https://github.com/spf13/viper)連携は利用しない
    - `viper`は仕組み上、外部リソースをgenericな`map[string]any`に変換 -> [mapstructure](https://github.com/go-viper/mapstructure/)で任意のstructに変換という過程をたどり、わりとややこしい。
    - LLMにコードを生成させる前提だと「書くには煩雑で面倒なだけだが簡単なコード」を避ける理由がない。`viper`のような仕組みでコードを軽量化する意図が薄い。
- 必須サブコマンドとして以下を定義
  - `version`: アプリのバージョンをprint
    - `./pkg/<name>/version.go`にソースコードとして埋められたバージョン情報を返却
    - debuginfoも返せるようになっています。
    - => 動的ロードされるライブラリがちゃんとシステムに存在しているか、のチェックに`version`コマンドをよく用いるので必須にしています。
  - `config`: アプリのconfigをprint
    - => アプリのデータの保存先とかを参照する外部スクリプトとかよく書くので必須にしています。
    - `--format`でGoの[text/template]表現を実行
      - => json出力して`jq`で処理、でもいいんですがが、`text/template`表現なら外部コマンドが一切いらないのでそこがいいです。
        - [dockerがtext/template表現を受け付ける](https://docs.docker.com/engine/cli/formatting/)ので`docker`よく触る人なら慣れているはず。

あとはいくつかヘルパーを定義して、必要なものはコピーするように指示しています。

ヘルパー群は別Go moduleとして定義してimportするだけでもいいんですが、そうなると依存先がさらにヘルパーモジュールに依存している場合にバージョン衝突が起きます。  
Goのモジュール解決のルールにより、`go.mod`はメジャーバージョンが同じである限り、複数のバージョンに対して同時に依存することができませんから、基本的にはより新しいほうが採用されます。  
後方互換性を気にする普通のGo moduleではこの使用は簡素で分かりやすいですが、このヘルパーに限っては困る挙動となります。なぜならヘルパーの挙動はガンガン破壊するつもりだからです。  
つまりコピペ方式をとることで重複を許容して変更の自由さをとっています。

Skillはヘルパーのカタログだけ持っており、実際のヘルパーはGoのソースコードで直接書かれています。確かカタログだけ記述するように明確に指示したような気がするので、きちんとそのように言わないとmarkdown内で全部書かれてしまう可能性あり。気を付けよう。

以下がカタログです。

https://github.com/ngicks/agents-package/blob/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/skills/go-edit-cobra/reference/workflows.md#helper-catalog

##### cmdsignals

https://github.com/ngicks/agents-package/tree/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/skills/go-edit-cobra/helpers/internal/cmdsignals

[signal.NotifyContext](https://pkg.go.dev/os/signal#NotifyContext)の代わりになるパッケージ

```go
// ExitSignals are the signals that should cancel top-level CLI execution.
var ExitSignals = [...]os.Signal{
	os.Interrupt,
	syscall.SIGTERM,
}

func NotifyContext(
	inCtx context.Context,
) (blockOn func(), ctx context.Context, cancel func(error))

func Pause(ctx context.Context, installHandler func()) (ok bool)

func Resume(ctx context.Context, removeHandler func()) (ok bool)
```

以下がメインの目的です

- graceful exitのためにシグナルハンドラを設定するのは普通だと思うが、どのシグナルをとるかを毎回定義するのが面倒だった
- `cmdman attach`のような、signalをアタッチ先にforwardしないといけないとき、一旦デフォルトのgraceful exit用のハンドラを外す必要がある。つけなおすためにはどこかでシグナルのセットを定義しておく必要がある。

さらに、重要な機能に`Pause`/`Resume`があります。

伝統的なPOSIX風のシグナルハンドラの操作では[sigaction(2)](https://man7.org/linux/man-pages/man2/sigaction.2.html)でカーネルレベルでatomicにシグナルハンドラを取り替えられます。`sigaction(2)`の`sa_mask`とか[pthread_sigmask(3)](https://man7.org/linux/man-pages/man3/pthread_sigmask.3.html)とかなどでプロセスにシグナルを渡さずにカーネルのところで貯めといてほしいシグナルセットとかを設定できるので、これでブロック状態入れ替えながら準備したりすることもあります。  
Goのsignal周りには全くそういうものがないので、どうもGoコード側で工夫をするのが求められているようです。

`Pause`/`Resume`はその「工夫」をあらかじめするものです。

- `Pause`:
  - `installHandler`実行
  - ctxに紐づいたchannelを[signal.Stop](https://pkg.go.dev/os/signal#Stop)で停止
  - `NotifyContext`から返る`blockOn`ループの中でシグナルを受けてcancelを呼ぶ部分を停止
- `Resume`: => `Pause`の逆
  - stateを変更してblockOnループ内でのキャンセレーションを復活
  - ctxに紐づいたchannelで[signal.Notify](https://pkg.go.dev/os/signal#Notify)を呼び、サブスクリプションを再開
  - `removeHandler`を実行。

柔軟性はないですがこれでLLMがふいに[signal.Reset](https://pkg.go.dev/os/signal#Reset)を呼んですべてのシグナルハンドラが外れるということはなくなる・・・はず。

##### loggerfactory

https://github.com/ngicks/agents-package/tree/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/skills/go-edit-cobra/helpers/internal/loggerfactory

[slog.Logger](https://pkg.go.dev/log/slog#Logger)をセットアップ一連の機能

`--log`, `--log-format`というフラグを追加し、さらに環境変数を呼んでログレベルやフォーマットを設定できるようにします。

- [pflag.BoolFunc](https://pkg.go.dev/github.com/spf13/pflag#BoolFunc)でフラグを設定することでフラグなしならログなしとかそういう感じにします。
- OpenTelemetry互換の`Trace`レベルと`Fatal`レベルも定義してあります。
  - 以下から`Trace = -8`, `Fatal = 12`でいいのがわかります。
    - https://github.com/open-telemetry/opentelemetry-go-contrib/blob/e2ed263954cb5fd04dd8cec838dee68d2599f402/bridges/otelslog/handler.go#L203
    - https://github.com/open-telemetry/opentelemetry-go/blob/76be4f71f5765042e31d544c1df7715d803e3461/log/severity.go#L21

##### stdiopipe

https://github.com/ngicks/agents-package/blob/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/skills/go-edit-cobra/helpers/internal/stdiopipe/stdiopipe.go

Goにおけるstdin, stdout, stderrは(unix系システムにおいては)ファイルディスクリプタの0, 1, 2をGoの[\*os.File](https://pkg.go.dev/os#File)でラップしたものです。

https://pkg.go.dev/os#pkg-variables

これが何が困るかというと、`Read` / `Write`でブロックしているとき、これらのファイルを`Close`してもアンブロックされません。
`goroutine`の中で`for-loop`で`Read`をするようなプログラムを書いているとき、`Read`がアンブロックされないとその`goroutine`が回収できず、リソースリークとなります。  
一旦[io.Pipe](https://pkg.go.dev/io#Pipe)をかますことでアンブロックできるようにしているのがこのヘルパーです。

以下をコピーしてローカルで実行してみればこの挙動がわかります。

```go
package main

import (
	"errors"
	"fmt"
	"os"
	"time"
)

func main() {
	go func() {
		// Nothing is ever written to p[1], so this Read blocks forever.
		var buf [1]byte
		n, err := os.Stdin.Read(buf[:])
		fmt.Printf("Read returned: n=%d err=%v\n", n, err)
	}()

	time.Sleep(100 * time.Millisecond)

	err := os.Stdin.SetReadDeadline(time.Now())
	fmt.Printf("SetReadDeadline: %v (ErrNoDeadline=%v)\n",
		err, errors.Is(err, os.ErrNoDeadline))

	err = os.Stdin.Close()
	fmt.Printf("Close: %v\n", err)
	time.Sleep(100 * time.Millisecond)
	fmt.Println("=> Read is still blocked: a blocking, non-pollable fd cannot be unblocked by a deadline")
}
// SetReadDeadline: file type does not support deadline (ErrNoDeadline=true)
// Close: <nil>
// => Read is still blocked: a blocking, non-pollable fd cannot be unblocked by a deadline
```

ちなみに`/dev/stdin`を開いても同じくdeadlineはつけられない。

pipeして`/proc/self/fd/p[0]`を開くようにするとdaedlineをつけられる模様。

```go
package main

import (
	"errors"
	"fmt"
	"os"
	"syscall"
	"time"
)

func main() {
	var p [2]int
	if err := syscall.Pipe(p[:]); err != nil {
		panic(err)
	}
	// For real stdin you would just write os.Open("/dev/stdin").
	r, err := os.Open(fmt.Sprintf("/proc/self/fd/%d", p[0]))
	if err != nil {
		panic(err)
	}

	done := make(chan error, 1)
	go func() {
		var buf [1]byte
		_, err := r.Read(buf[:]) // blocks (nothing written to p[1])
		done <- err
	}()

	time.Sleep(200 * time.Millisecond)
	fmt.Printf("SetReadDeadline: %v\n", r.SetReadDeadline(time.Now()))

	err = <-done
	fmt.Printf("Read returned: err=%v (ErrDeadlineExceeded=%v)\n",
		err, errors.Is(err, os.ErrDeadlineExceeded))
}
// SetReadDeadline: <nil>
// Read returned: err=read /proc/self/fd/4: i/o timeout (ErrDeadlineExceeded=true)
```

ただしどうもこのハックはLinuxオンリーかもしれない。少なくともBSD系ではドキュメントを読む限り動きません。

FreeBSD fd(4) — https://man.freebsd.org/cgi/man.cgi?query=fd&sektion=4

>       The files /dev/fd/0 through /dev/fd/# refer to file descriptors which
>       can be accessed through the file system.  If the file descriptor is
>       open and the mode the file is being opened with is a subset of the mode
>       of the existing descriptor, the call:
>
>           fd = open("/dev/fd/0", mode);
>
>       and the call:
>
>           fd = fcntl(0, F_DUPFD, 0);
>
>       are equivalent.

他のBSD派生でも同じなはずなので、同様にmacでもこの挙動でしょう。

`io.Pipe`をかませる方法はそれどころかWindowsでも同様に動くので、多少効率が悪かろうとこの方法をとったほうが良いといえます。

##### versioninfo

https://github.com/ngicks/agents-package/blob/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/skills/go-edit-cobra/helpers/internal/versioninfo/versioninfo.go

[debug.ReadBuildInfo](https://pkg.go.dev/runtime/debug#ReadBuildInfo)の`Settings`の中身が`"vcs.revision"`などを照会しないといけないのをstructに詰めなおすヘルパーです
いるんかいらんかは微妙ですが・・・まあいいでしょう！

#### go-check-outdated-patterns: Goの古いイディオムを指摘するreview skill

https://github.com/ngicks/agents-package/blob/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/skills/go-check-outdated-patterns/SKILL.md

`claude`を好きにさせると普通に古い書きまわしで`Go`を書いてくることがあります。

例えば、

- `new("expr")`の代わりに`func ptr[T any](t T) *T { return &t }`を使ってきたり(Go1.26で追加)
- `errors.AsType[T]`の代わりに`erros.As`を使ってきたり(Go 1.26で追加)
- `sync.WaitGroup.Go()`の代わりに`wg.Add(1), go func() { defer wg.Done(); ... }()`を使ってきたり(Go1.25で追加)
- `omitzero`の代わりに`omitempty`を使ってきたり(Go1.24で追加)

みたいな感じ。間違ってないんだけど古いんだよな～って書き方がよくされるのでそれを戒めるスキルです。  
結構大部分は公式のlinterである`modernize`がカバーしてくれるのでlinterをhookで実行するだけでもいいんですが、確実に間違っているといえない範囲のものは注意してこない(上記の`omitempty`は`modernize`のwarning対象ではない)のでこのスキルはいくらか有用です。

> Read Go's release notes from https://go.dev/doc/go1.24 to go1.26
> and collect DOs(recoomendations) and DONTs

みたいなプロンプトかまして作らせました。

このスキルだけだとロードされないことがあって、確実性が欲しい場面ではlinterに落とし込んだほうがいいなと思ってlinterの自作にも進んでいます(後述)

#### go-cmdman-review-checklist: プロジェクト固有のルールを守らせるreview skill

https://github.com/ngicks/cmdman/blob/637fd046b8e87c37804f84aa41a6d0045caad0a9/.apm/skills/go-cmdman-review-checklist/SKILL.md

`claude code`鼻にも言わなくても周辺のコードと雰囲気を合わせるので、特に何も言わなくても規則が守られることも多いんですが、例えば新しくディレクトリを作ってそこ以下で新しい機能を開発してるときとかは、周辺のコードを読んでいないのでその「周辺の雰囲気」がなくなることで急にプロジェクト全体の雰囲気から逸脱したことをやりだすことがあります。

なのでプロジェクト全体ルールはinstructionsやskillで呼び出せたほうが規模が大きくなってきたら楽かなあと思います。

### Hooks

#### crabswarm hook exec: claude / codex両対応のhook runner

https://github.com/ngicks/crabswarm/blob/7b3e9ea74420ac2d5dcfb85186852e42179df05a/cmd/crabswarm/commands/hook_exec.go

hook呼び出すのに若干苦労があるのでラップできるようなツールを作りました

具体的な苦労っていうのは以下です。

- warningをclaude返すには特別なJSONフォーマットに成形しなおすか、stdoutに出力しないといけない
  - `golanci-lint`のエラー内容はstderrに出力されるためstdoutに出力しなおすスクリプトが必要です。
- ファイルタイプでフィルターする機能がないっぽい？
  - つまり`golangci-lint`は`.go`をいじったときだけ実行したいのに、`.ts`なファイルをいじっても実行してしまう
  - リファレンスによるとそんな機能はない: https://code.claude.com/docs/en/hooks
  - この記事のようなことをしないといけない => https://zenn.dev/budougumi617/articles/claudecode-hooks-format-for-go
- `claude`コマンドを呼び出したパスで実行される
  - web uiとバックエンドを同時に開発しているmono-repoなんかでプロジェクト横断して`claude`に触らせてるとき、web ui向けのhookは`package.json`のあるディレクトリ以下で実行しないと行けないんだけど、どこがそのディレクトリなのかの検知が煩雑
- `codex`のtool_inputが特殊フォーマットすぎて`jq`では編集されたファイルがどれなのか全くわからない

かなり致命的！

毎度これらを実装するスクリプトを組むわけにもいきませんからヘルパーを作っておきました。

- `--ft`(filetype)による絞り込み + root detection:
  - [neovimのlsp](https://neovim.io/doc/user/lsp/)のroot周りの設定を参考ににしてます。
  - root detection = `js`なら`package.json`を探して、それと同階層に`cwd`が設定されます。
  - ft fitler = `--ft=rust`のとき、`.rs`や`Cargo.toml`ファイルが編集されたときだけhookが実行されます
  - 組み込みの設定ほか、ユーザーの設定で拡張可能にしてあります
    - [組み込み設定](https://github.com/ngicks/crabswarm/blob/7b3e9ea74420ac2d5dcfb85186852e42179df05a/pkg/crabswarm/hook/exec/default_config.json)は`go`, `rust`, `python`, `javascript`, `typescript`, `shell`, `yaml`, `json`, `markdown`, `make`, `moonbit`に対応。
    - これ以上は私が困らない限りデフォルトでは追加されない
- `crabswarm hook exec`に渡されるコマンドはGo text/template形式で生成可能
  - templateに渡される値に編集されたファイルなどが入っているので、そのファイルに対してのみフォーマッターを実行したり、親ディレクトリに対してlintをかけたりできる
- `codex`対応！
  - `codex`も`claude`と動形式のhookに対応しています: https://developers.openai.com/codex/hooks
  - ただしtool_inputの中身が`claude`と全然違って`*** Begin Patch\n*** Update File: path/to/file...`みたいなテキストフォーマットになっているため、`jq`だけでは編集されたファイルのパスを取り出せません。
  - そこで`codex`のソースを洗ってパーザーを組んでいます。

今んところallowかblockかを解凍する以外できていないので他のtoolに対応したいところ・・・

#### linterとformatter

https://github.com/ngicks/agents-package/blob/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/hooks/golangci-lint-run-after-edit/hooks/hook.json

https://github.com/ngicks/agents-package/blob/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/hooks/golangci-lint-fmt-after-edit/hooks/hook.json

上記の`crabswarm hook exec`を使って簡素に設定できてます。

使っているのは[golangci-lint](https://golangci-lint.run/)です。デファクトスタンダードな雰囲気があってとくになにもいうことがない。

`Go`では特にlinterをhook出かけるのは効果が高いです。

- もとから公式ツールに暑いlinterがある
- LLMは古かったり非効率が書き方をしてくる

非効率な書き方の例はfor-loopの中でstringの足し算をしちゃうやつとかですね

```go
s := "foo bar"

for i := range 10 {
    s += "hmm..."
}
```

これは`+=`のところで毎回allocationが走るため非常に非効率です。プログラミング言語によっては裏でしれっとallocationが１回になるようにコードをrewriteしてくることもあるかもしれませんが、少なくとも筆者の知っているバージョンの`Go`ではそうなっていません。  
`golangci-lint`を用いなくても`gopls`(Goの言語サーバー)に組み込まれたanalyzerで[strings.Builder](https://pkg.go.dev/strings#Builder)を使って書きなおすようにwarningが出ます。  
**claudeが実際にこういうコードを出してきました**

linterを入れないのは余りに危険です。何の設定もしていない公式toolchainについてくるチェッカーで十分高級なことも多いと思うのでhookでかかけるのがお勧め。

#### カスタムlinter: オレオレvettoolで任意のlinterを作成

`golangci-lint`のあらかじめ定義するルールセットに存在しないruleが欲しいとき、自作する必要があります。

https://golangci-lint.run/docs/contributing/new-linters/

上記より、Goでカスタムlinterを作るためには`go/analysis`という公式から提供される機能を用いればよいです。

https://pkg.go.dev/golang.org/x/tools/go/analysis

これ自体は簡単ですが、これを`golangci-lint`に組み込むのは面倒です

https://golangci-lint.run/docs/plugins/module-plugins/

その場でビルドしてカスタムバイナリを実行しろということらしい。`worktree`でワークスペースをコピーしまくるのでこの過程若干というかなり面倒ですね。

ただ幸運なことに、`golangci-lint`に組み込まれることにこだわりがない限りはもっと簡単な方法があります。  
上記の`go/analysis`で実装されたツールを`go vet -vettool`で呼び出す方法です。

```
❯ go tool vet help | grep '\-.*tool'
For analyzers of the first kind, use "go vet -vettool=PROGRAM"
For analyzers of the second kind, use "go fix -fixtool=PROGRAM"
```

ということで作っておきました。

vettoolは以下

https://github.com/ngicks/go-ngcheckers

呼び出しのapm packageは以下

https://github.com/ngicks/agents-package/blob/632e76121d26524d67c4e2d17f79ef65e0d2dd5a/hooks/go-vet-ngcheckers/hooks/hook.json

ルールは今のところ一つしかありません。

- Go1.24以降では`json:",omitempty"`の全面禁止: https://github.com/ngicks/go-ngcheckers/blob/main/rules/noomitempty

`claude`に作らせたら一撃かと思いきや意外と熟達が必要です。
中身はast(abstract syntax tree)の解析などでなってるわけですから、それに対する熟達がないと抜けに気づけないんですね。

例えば初期実装では下記のテストケースが抜けてました。

https://github.com/ngicks/go-ngcheckers/blob/fad48429027334e755459422925d8ccc22e7a488/rules/noomitempty/testdata/go124/a/a.go#L45-L50

Goのspecを眺めたりひたすらastいじりまわしてた人ならすぐ気づくんでしょうが、Goのstruct tagはstring literalなので「\`\`」でも「""」でも書けます。  
普通は「\`\`」のほうしか使われないのでケースとして抜けるらしい。

「言語仕様を網羅してるか？」という問いかけ一つでこの抜けに気づかせることができたかもしれない。次回からそういう質問もしてみようと思います。

## 実装していく中での発見

### ptyは１つのファイル

当たり前なんだけどptyって１つのファイルなんですよね。

`pty`のallocationには[github.com/creack/pty]を用います。

`pty`の外向けのinterfaceはかなりシンプルで、以下の`Start`を呼ぶのがもっとも簡単。カスタマイズしたかったら`StartWith...`とか`Open`とか読んでねって感じ

https://github.com/creack/pty/blob/v1.1.24/run.go

`Open`は[pty(7)](https://man7.org/linux/man-pages/man7/pty.7.html)に基づいて`/dev/ptmx`を開いてmasterを得て、それに対して`ioctl`することでslave(の番号)を得ます。  
子プロセスのstdin,stdout,stderrにslave側を指定してmaster側を返します。

なんとなく勝手に`pty`ってstdoutとstderrに区別があるのかと思ってたんですが、ないんですね。
考えて見れば当たり前で手元のターミナルの画面にstdoutとstderrを区別するものはありませんから、ptyにもなくて当然ですね。

勝手に思い込んで勝手に驚いてました。

### daemonizeはsingle-forkでいいらしいけどconmonがdouble-forkだからdouble-forkにした

`cmdman run`などがコマンドをバックグラウンド実行するために、`__monitor`隠しサブコマンドをサブプロセスとして、それが自らを`daemonize`します。

基本はdouble-forkするものと思っていたんですが、どうしてなんだろうと思って調べていたら以下の記事に行きつきました。

https://sleepy-yoshi.hatenablog.com/entry/20100228/p1

曰く、Solaris以外ではsingle-forkでいいらしい。  
たしかに(記事中でも引用されている)Advanced Programming in the UNIX Environmentを読み直すと、double-forkの２回目のforkはoptionalな言い回しでした。

じゃあいいかと思って当初はsingle-forkで実装していたんですが、気になって[conmon](https://github.com/containers/conmon)(podman/CRI-Oのcontainer monitor)はdouble-forkをしていた・・・

https://github.com/containers/conmon/blob/535294085c5603dfa2923a4f0ccddda28d64424c/src/conmon.c#L88-L106

> ```c
> /* In the non-sync case, we double-fork in
>  * order to disconnect from the parent, as we want to
>  * continue in a daemon-like way */
> ```

readmeにもがっつりdouble-forkと載ってます。

cmdmanは全くSolarisを意識していないですがdouble-forkを採用します！

### term.Restoreだけではterminal stateは不十分

`cmdman attach`は`--tty`フラグのついたアプリにアタッチして、呼び出しているターミナルにコマンドを表示します。
`vim`にアタッチしたら画面全体が`vim`になるみたいな感じです。

これをするには[termios(3)](https://man7.org/linux/man-pages/man3/termios.3.html)より、`cfmakeraw(3)`で`struct termios`をraw modeに設定し, `tcsetattr(3)`で適用します。元に戻すには`cfmakeraw(3)`を呼び出す前の`termios(3)`をコピーしておいて`tcsetattr(3)`で適用しなおすことで行います。

`Go`では[term.MakeRaw](https://pkg.go.dev/golang.org/x/term#MakeRaw)/[term.Restore](https://pkg.go.dev/golang.org/x/term#Restore)を用います。

当初は単にこの`MakeRaw` / `Restore`しか実装していなかったんですが、`cmdman attach`してから`ctrl-p,ctrl-q`でdetachするとカーソルが消えたり、表示状態が壊れることがわかりました。

ここで`podman`/`docker`のソースを洗ってどうやってattachから復帰すべきかの参考にしようとしましたが・・・全く何もない！

では[bubbletea]はどうしているのでしょう？

https://github.com/charmbracelet/bubbletea/blob/v2.0.7/cursed_renderer.go#L143-L246

https://github.com/charmbracelet/bubbletea/blob/v2.0.7/tty.go#L33-L53

ansiコードを使ってターミナル状態をリセットしていました。見てのとおり、ターミナルは特定のエスケープシーケンスを書き出すと、対応した機能が動作します。ちょうどLLMのtool useが特殊なトークンでかこまれたテキストを出力すると読む側がそれを解釈して実行しているのと似たようなものです。

`cmdman`は任意のコマンドを実行するため状態がどうなるかわかりません。ということでベストエフォートで戻せるもんは戻します。

https://github.com/ngicks/cmdman/blob/897a4ac18d5b53fd4943bfeeeda5a4d744415f0e/pkg/cmdman/cli/attach.go#L556-L568

シーケンスの一覧は[XTerm Control Sequences](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html)で確認できます。一部はデファクトスタンダードなようです。

> CSI ? Pm h
> Ps = 2 5 ⇒ Show cursor (DECTCEM), VT220.

ちょっと読みにくいですがesacpe(`\033`) + `[?` + `25`(Pm) + `h`ってことみたいですね。

で結局`podman`にそういうコードがなかったのはどういう仕組み何だろうと思い、attachとdetachを試してみました。

```shell
container_id=$(podman container run -td --rm --init docker.io/library/ubuntu:noble-20260113 watch ls -la /tmp)
podman container attach $container_id
```

でアタッチしたのちに`ctrl-p,ctrl-q`でデタッチすると・・・２度以降のアタッチで`watch`が表示されない！
どういう現象かわかりませんがこの辺そもそもちゃんと実装されてないから`podman`にそういうコードがなかったっぽい。
確かに今回調べるまで`attach`があるの知らなかった・・・
[私の前の記事](https://zenn.dev/ngicks/articles/using-krun-in-podman)で説明した`libkrun`によるコンテナごとのマイクロVM文理を行っている場合、`libkrun`の制限によって`podman container exec`でコンテナのnamespace内で新しくコマンドを実行したりできないので、そういうケースなどで`attach`が便利なのかなと思います。

### attach時に特定のシーケンスはリプレイしないとだめそう

ある日、`cmdman compose mux up`と`cmdman compose mux down`何度か繰り返していると、`neovim`を表示しているターミナルで`tmux`マウスサポートが動作していました。  
この時点でようやく気付いたんですが、よく考えると`tmux`の組み込まれたマウスサポート機能って`neovim`みたいなマウスサポートのあるアプリ上では機能しないようになっているんですね。

調べてみるとこのマウスサポートのON/OFFは`tmux`の以下の行でコントロールされているらしく、

https://github.com/tmux/tmux/blob/3.6/input.c#L1968-L1979

調べて突き合わせると[console_codes(4)](https://man7.org/linux/man-pages/man4/console_codes.4.html)dで述べられる` ESC [ ? 1000 h`で有効化されるらしい

>       DEC Private Mode (DECSET/DECRST) sequences
>
>       These are not described in ECMA-48.  We list the Set Mode
>       sequences; the Reset Mode sequences are obtained by replacing the
>       final 'h' by 'l'.

>       ESC [ ? 1000 h
>              X11 Mouse Reporting (default off): Set reporting mode to 2
>              (or reset to 0) —see below—.

`cmdman attach`は、というかmonitorはスクロールバックを保存するためにリングバッファを備えており、`cmdman attach`時にはこのバッファ内容がリプレイすることでスクロールバックを再建します。  
挙動から見るに`neovim`は起動時に１度しか`ESC [ ? 1000 h`を発火しないため、しばらく操作してから`mux up`を行うとバッファからこのコントロールシーケンスがあふれてしまうのですね。

ということで以下でこの手の状態の変更を行うシーケンスを貯めてバッファあふれて消えないように対策することにしました

https://github.com/ngicks/cmdman/blob/897a4ac18d5b53fd4943bfeeeda5a4d744415f0e/pkg/cmdman/terminal_state.go

### zellij / weztermに実装を広げるのが大変な理由

`mux`機能のコア要件は、cmdmanが作ったwindowに**識別子(identity)をstampして、後からそのstampでwindowを列挙できる**こと。レイアウトのサイクルや`mux down`(後片付け)、`mux ls`に要る。しかもこの列挙は`$TMUX`や今いるpaneに依存せず、サーバー全体に対してどこからでも効かないといけない(`down`はpaneの外からでも叩くので)。

これがmultiplexerごとに事情が全然違う。

- **[tmux]**: window単位のuser option`@cmdman_window`がネイティブにある。`set-option -w`でstampして`list-windows -a -F '#{@cmdman_window}'`で列挙。きれい。
- **[WezTerm](将来)**: per-paneのuser varsをpaneの中からescape sequenceで吐いて`wezterm cli list`で読み戻す、という手はある。でもこれを配線するのにユーザー側のluaスクリプト設定が要って、セットアップ負担がデカすぎる。
- **[zellij](将来)**: そもそも今は任意のper-window/per-paneメタデータを格納する手段が無い。プラグインを書いてそこにデータを持たせるしかない。完全にvisionの段階。

このへんの設計判断は[`pkg/muxctl`のpackage doc](https://github.com/ngicks/cmdman/blob/897a4ac18d5b53fd4943bfeeeda5a4d744415f0e/pkg/muxctl/doc.go)に書き残してあります。

ここで効いてくる教訓が「**title(window/pane/tabのタイトル)をidentityの保管場所にしてはいけない**」。タイトルはプログラムやシェルやユーザーが平気で上書きするので、title-baseのidentityは通常運用で静かに失われる。これは実際、paneタイトルの末尾にレイアウトindexを埋めてた初期実装が壊れやすかったので`@cmdman_marker`オプションに移した、というのと同じ教訓です。

「じゃあサイドカーのレジストリファイル(identity→window idのJSON)を置けばいいのでは？」も検討して却下しました。サイドカーはsource of truthにはなれず、せいぜいキャッシュ。読むたびにliveness(手動でwindowを閉じた / サーバー再起動でidが再利用された)とownershipをライブのmultiplexerに対して再検証しないといけなくて、それには結局multiplexer内のマークが要る。「ネイティブstamp + サイドカー + ロック + クラッシュ時掃除」に膨らむだけなので、ネイティブに格納手段があるtmuxではネイティブが厳密に上位互換でした。

### tuiでターミナルアプリを表示するには

前述の通り、`tui`のログPreviewはtty必須のアプリ(neovimとかtopとか)を流すとガチャガチャに崩れます。原因はシンプルで、cmdmanのtuiはコマンドの出力バイト列をそのままPreview領域に流し込んでるだけだから。出力には生のescape sequence(カーソル移動の`ESC[H`、色の`ESC[31m`、alt screen切り替え...)が混ざっていて、それを解釈せずに描くとああなる。

ちゃんと表示しようと思うと、結局**自前のターミナルエミュレータが要る**。tty必須のアプリは「向こう側に本物のターミナルがいる」前提でescape sequenceを吐いてくるので、それを自分のtuiの中の小さな矩形領域にレンダリングするには、

1. ptyを割り当ててアプリにttyを掴ませ(=アプリにtty向けの出力を吐かせ)、
2. ptyから返ってくるバイト列をパースして**カーソル位置 + styled cellの2次元グリッド**(=画面状態)を自前で持ち、
3. そのグリッドをtuiの矩形にcell単位で描き直す、

という三段構えが要る。ptyは「アプリにtty向けの出力を吐かせるtransport」、ターミナルエミュレータは「そのescape streamを画面グリッドに解釈するinterpreter/renderer」で役割が別もの。どっちが欠けてもダメで、ptyだけだとアプリは喋るけど生のescape codeが画面に見えるし、エミュレータだけだとそもそもアプリがtty出力を吐いてくれない。

実例として[process-compose]がこれをやっています。`tview`ベースのtuiの中で管理プロセスの出力を表示するために、サードパーティのVTエミュレータ(vt10xとか)に乗っかるんじゃなくて**自前のANSI/VTパーサ + cellグリッド**をin-treeで実装している。型にそのままコメントが付いてる:

https://github.com/F1bonacc1/process-compose/blob/1057510943cfa642cf34cb001bb356350e2da87b/src/tui/ansi_terminal.go#L17-L24

> ```go
> // AnsiTerminal is a simple ANSI escape sequence parser and terminal emulator
> ```

ptyから読んだバイト列をこのエミュレータに食わせてグリッドを更新し、それをtuiの矩形に描き出す、という流れ。要するにtuiの中にミニ・ターミナルエミュレータを丸ごと抱えることになる。

別にprocess-composeみたいに手書きする必要があるわけでもなくて、cmdmanのtuiが乗ってる[bubbletea]側のエコシステムには出来合いの[charmbracelet/x/vt]があります。"a virtual terminal emulator that can be used to emulate a modern terminal application"と銘打たれた、まさにこれ用のVTエミュレータ:

https://github.com/charmbracelet/x/blob/25656177ba8e62179605964fc9075a5226d93b0b/vt/vt.go#L1-L3

> ```go
> // Package vt is a virtual terminal emulator that can be used to emulate a
> // modern terminal application.
> ```

ただ中身はフルのVTパーサ + screen cellグリッド + scrollback(デフォルト10000行) + DEC private modes(マウス・focus・bracketed paste・alt screen...)一式で、前述のリプレイの節で自前で苦労してたモード追跡なんかも全部入った、process-composeが手書きしたのと同じレイヤーをそっくり持つ重量級です。乗っかれば楽ではあるんですが、ログをチラ見するだけのPreviewにこの machinery を引きずり込むのはcmdmanにはオーバースペックなので入れていません。ちゃんと操作したいときは`attach`でネイティブのターミナルにstdioを繋ぐ方に倒す、という割り切りにしています。

### Windows移植

完全に移植性を捨てて書いてるので今のところWindowsでは動きませんが、もし移植するなら何が大変で何が楽かだけ整理しておきます。

**本丸はpty**。cmdmanが使ってる[github.com/creack/pty]はそもそもWindows非対応です。`start_windows.go`は`ErrUnsupported`を返すだけのスタブで、中身がない:

https://github.com/creack/pty/blob/master/start_windows.go

ConPTY対応を入れようとしたPRは何度か出てるんですが、どれもマージされていません([#155](https://github.com/creack/pty/pull/155)はopenのまま放置、[#109](https://github.com/creack/pty/pull/109)・[#208](https://github.com/creack/pty/pull/208)はcloseされた)。[Windows対応のトラッキングissue](https://github.com/creack/pty/issues/95)も2020年から開きっぱなし。なので移植するなら自前でConPTYを叩くか、ConPTYラッパーの別ライブラリを使うことになる。

そのConPTY(`CreatePseudoConsole` API)自体が割と最近の代物で、Windows 10 version 1809(build 17763, 2018年10月)でようやく入ったものです。

https://learn.microsoft.com/en-us/windows/console/createpseudoconsole

逆に言うと、それ以前のWindowsには「向こうにptyがいる」前提のescape streamを供給する仕組みが無かった。じゃあConPTY以前はどうしてたかというと、有名なのが[winpty]の力技で、**子プロセスを隠しコンソール(hidden console)に繋いで起動し、そのコンソールのscreen bufferをポーリングして変化を拾い、escape sequenceのストリームに変換し直す**というもの。READMEにそのまま書いてある:

https://github.com/rprichard/winpty/blob/master/README.md

> The software works by starting the `winpty-agent.exe` process with a new, hidden console window, which bridges between the console API and terminal input/output escape codes. It polls the hidden console's screen buffer for changes and generates a corresponding stream of output.

VSCodeの統合ターミナルが使ってる[node-pty]も、ConPTYが来るまではこのwinptyをバックエンドにしていて、build 18309以降でConPTYをデフォルトに切り替えています。前のセクションで「tuiでアプリを表示するには自前のターミナルエミュレータが要る」と書きましたが、Windowsの旧来のやり方はそれを**OS側のコンソールに肩代わりさせてscreen bufferから読み戻す**という、ちょうど裏返しのアプローチなのがちょっとおもしろい。

一方で**daemonizeはむしろ楽**。Windowsにはforkが無くてCreateProcessでプロセスを作るので、double-forkみたいな小細工が要らない。detached生成のフラグを渡すだけでバックグラウンド化できるので、前述のfork談義はWindowsでは丸ごと不要になる。要するに、Windows移植の重さはほぼ全部pty側に乗っかってる、という話でした。

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

<!-- Go std reference -->

[text/template]: https://pkg.go.dev/text/template

<!-- runtimes -->

[Node.js]: https://nodejs.org

<!-- lib -->

[bubbletea]: https://github.com/charmbracelet/bubbletea
[charmbracelet/x/vt]: https://github.com/charmbracelet/x/tree/main/vt
[github.com/urfave/cli]: https://github.com/urfave/cli
[github.com/spf13/cobra]: https://github.com/spf13/cobra
[github.com/creack/pty]: https://github.com/creack/pty
[process-compose]: https://github.com/F1bonacc1/process-compose
[winpty]: https://github.com/rprichard/winpty
[node-pty]: https://github.com/microsoft/node-pty

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
