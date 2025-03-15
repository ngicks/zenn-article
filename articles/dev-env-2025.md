---
title: '最近の開発環境(neovim/tmux/ssh)'
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["neovim","tmux"]
published: false
---

## 最近の開発環境(neovim/tmux/ssh)

なんか設定をまとめるのに苦労したので記事にして供養します。
(書いてリンクとかまとめないと忘れちゃいそうだし。)

## wsl

普段Windowsに入れた`wsl2`からLinuxを使っています。
操作はwindows terminalからです。weztermなどの導入は検討だけして現状困ってないので放置されている状態・・・。

### installation

[How to install Linux on Windows with WSL](https://learn.microsoft.com/en-us/windows/wsl/install)

どこのバージョンからかは忘れちゃいましたが管理者権限で開いたpowershellなりで`wsl --install`を実行するだけでインストールできるようになりました。

###  install Ubuntu 24.04.05 LTS

特にこだわりはないですがUbuntu 24.04.01を入れて使っています。
Microsoft storeから入れたんだったかなあ、

[Microsoft Store: Ubuntu 22.04.1 LTS](https://apps.microsoft.com/detail/9nz3klhxdjp5)

とりあえず1つディストロを入れれば`wsl --export ${distro} ./path/to/file`でファイルに書き出し、これを`--import`することなどで複製することができるので、
適当なものを見繕えばよいです。

既存のディストロの`vhdx`(wsl instanceのvhdd)の位置を変更するにはおそらくレジストリの編集が必要です。

### 設定1: windowsの機能の有効化または無効化

windowsの機能の有効化または無効化から

- Hyper-V
- Linux用サブシステム

を有効にしておきます。何かで必要になると書いてあったけど真贋不明

### 設定2: vhdxのsparse化(optional)

ノートパソコンなどに入れる場合はsparse化も有効化しておくほうが良いと思います

```
wsl --manage ${distro} --set-sparse true
```

### 設定3: mirrored network有効化

[Accessing network applications with WSL#Mirrored mode networking](https://learn.microsoft.com/en-us/windows/wsl/networking#mirrored-mode-networking)

mirroed networkにしておくほうがいろいろ困らない。

AutoProxyは有効化しないほうがややこしくない。どうもwindowsの設定を参照して`HTTP_PROXY`を自動的に設定するだけのようですが

- `Basic-Auth`のための`user:password`ははいらない
- ときどき切ったりするときに不便

ですのでproxyは適当な`proxy.sh`などに設定しておいて`. ~/.config/proxy.sh`みたいな感じでsourceする/しないを制御できたほうが私は楽でした。
そのうちwslが改善するかもしれないので要確認。

そもそも今時ネットワーク内にProxyが存在しないことも多いかもしれないです。

### 設定4: windows terminalのCtrl+Vのpaste操作を削除

Windows terminalには`Ctrl+Shift+V`, `Ctrl+v`両方でpasteできるのがデフォルトの設定のようです。
ですのでこれを消します。

Windows Terminalの設定>操作を一番下までスクロールすると`Ctrl+v`が存在しているのでこれを消して「保存」をクリックします。

### 設定5: Nerd Fontのinstall / 設定

のちのneovimのセットアップにNerd Fontが必要なのでWindows側にこれをインストールします。

[Nerd Fonts](https://www.nerdfonts.com/)

好きなやつを選んでinstallしておきます。

ちなみにrepositoryのトップに全部のnerd fontsをinstallするスクリプトがついているので全部入れたいならcloneして実行します

https://github.com/ryanoasis/nerd-fonts/blob/v3.3.0/install.ps1

このscriptはsignされていないので実行するにはexecution policyの変更が必要です

https://learn.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.5

Windows Terminalの設定>Ubuntu 24.04.1 LTS(ディストロ)>外観の順番でナビゲート、

- テキスト>フォントフェイスを先ほどインストールしたNerd Fontにします。
- ウィンドウ>パディングを適当に下げておきます。さげないとneovimを開いたときに余白ができて少しぶさいく。 

## Ubuntuの設定

### 環境変数

適当なスクリプトで`export COLORTERM=truecolor`を設定しておきます。これさえあればtruecolorで表示できるようだ。

[#11057](https://github.com/microsoft/terminal/issues/11057)

何か副作用でうまく動かなくなったら消す予定。

### sdk類のインストール

neovimなどのプラグインには`python`や`npm`のような`sdk`/ランタイムを使うものが普通にあるので入れておいたほうが吉です。

- node.js/npm
- python
- Go
- Rust/cargo
- clang/gcc
- make

このあたりはとりあえず入れておいたほうがよいです。

- node.js/npmは[volta](https://volta.sh/)で管理しています
  - ちなみに`Basic-Auth`の必要なProxy下では動作しませんでした([volta#2005](https://github.com/volta-cli/volta/issues/2005)),
  - httpライブラリがProxy接続時に`Proxy-Authrozation`ヘッダーを入れないから起きていました。そっちの修正は取り込んでもらえたので([attochttpc#185](https://github.com/sbstp/attohttpc/pull/185))voltaのbump待ちです。
- pythonは[uv](https://github.com/astral-sh/uv)で管理しています
  - `uv venv ~/.local/uv_global`して`export PATH="$HOME/.local/uv_global/bin:$PATH"`するとglobalなpythonとして使える・・・はず。
- Go, Rust/cargoは公式通りの手順で入れてます。
  - ただしどちらも`$HOME`以下に置かれるようにしてあります
- clang/gcc/makeはシステムに付属のパッケージマネージャで導入します。(apt)

そこでオレオレパッケージマネージャ

https://github.com/ngicks/ngpkgmgr

各種パッケージにinstall scriptがついて回るのでそれを実行するのと最新版の確認方法さえまとめられれば一方通行で更新をかけるぐらいはできるわけですね。
このGo moduleでまとめたinstall sciptを順繰りに実行します。そんだけです。
これを他人に使うように勧める気は全くございませんが、installと最新の確認方法のまとめとしては機能する、かも。

### ツール類のインストール

筆者の好みと知るところにより下記をインストールをしておきます。
これらはのちにneovimプラグインなどから利用されますが普通にコマンドラインから使って便利なので入れておいて損はない。

- lazygit: https://github.com/jesseduffield/lazygit
- fzf: https://github.com/junegunn/fzf
- rigpreg: https://github.com/BurntSushi/ripgrep
- xsel: http://github.com/kfish/xsel

## tmuxの設定

[tmux](https://github.com/tmux/tmux)を導入します。

### 概要

`tmux`はterminal multiplexer, 単一のterminalを分割して複数のtemrinalを動かしたり、裏でサーバー化されるのでコマンドを裏で動かして放置して帰ったりできます。

- 1つのtermianl windowを複数のtermianlに分割できます
- session -- window -- paneの概念でterminalを分割します
- paneは1つのterminal windowの中で分割された各termianlのことをさし、
- paneを複数格納するのがwindowで
- さらにsessionは複数のwindowを格納します
- sessionごとに個別に停止したりアタッチしたりできます
- 画面に表示されるのは単一のwindowですが、他のwindow/sessionの動作は継続されます。
- 初回のtmuxの実行時にtmuxサーバーが裏で立ち上がってこれに対してコマンドを送る形で実装されます。
- そのため、sessionを継続したままdetachすることできます。
- `command | /dev/null`して`Ctrl+Z`して`bg`して`disown`するような工夫なしにコマンドを実行したまま帰るようなことができます。
- クラサバモデルなおかげかsessionにアタッチせずにコマンドを送り込めるので事前に何個かwindowを開いてpaneに分けてコマンドを実行しておく、みたいなことができます

neovimも同様にクラサバモデルをとるため、ほぼ同じようなセッション管理が可能なようですが以下の単一の理由でtmuxを使い続けています

- neovimの`:terminal`では、画面上の折り返しがそのまま改行としてコピーされています
- tmuxでは折り返しは改行されずにコピーされます

neovimだけで成立させる方法はあると思いますが、tmuxで行うほうが素直という判断に基づいて

### installation

導入は普通に`apt`で行いました。

```
sudo apt install tmux
```

### 設定

設定はここで管理しています。

https://github.com/ngicks/tmuxconf

設定は以下のいずれかが読み込まれます。

- `~/.tmux.conf`
- `$XDG_CONFIG_HOME/tmux/tmux.conf`
- `~/.confg/tmux/tmux.conf`
- `/etc/tmux.conf`

READMEに書かれている通り,`~/.config/tmux`にrepositoryをcloneすればよいようにしてあります。
`~/`直下にファイルが増えるとよくわからなくなるためです。

設定は以下を参考にします

- https://github.com/tmux/tmux/wiki/Getting-Started
- https://man7.org/linux/man-pages/man1/tmux.1.html
- https://github.com/tmux/tmux/blob/master/.github/README.md

使ってみて慣れたら上記を参照するとよいと思います

#### 基本的な使い方

とりあえず使えるとこまで持っていけばハードル下がるかと思って基本的な使い方を述べます

- `tmux`で起動
- その状態で普通にターミナルが立ち上がってます。なんでもコマンドを実行できます
- prefix(下記でかえるがデフォルトは`Ctrl-b`)+`%`で右に新しい`pane`を分割
- prefix+`"`で下に新しい`pane`を分割
- prefix+arrow keyで`pane`を移動
- prefix+`c`で新しいwindowを作成
- prefix+`w`でwindow選択画面に移動
- prefix+`s`でsession選択画面に移動
- `pane`を終了させるのは普通にterminalを終了させる(=Ctrl+DでEOFを送るなど)するとよいです
- そのほかにもprefix+`>`で開くメニューから選ぶとか、prefix+`:`でcommand promptに入り、`kill-pane`でも消せます。
- prefix+`d`でsessionを終了せずにdetach
  - `tmux`を実行しているのがリモートサーバーの場合はこのままローカルのPCは落としてもSSHを切ってもコマンドの実行は続きます。これでかえれますね。
- `tmux attach`で既存のsessionにattachできます
  - `tmux attach -t ${session_name}`で特定のsessinoにattachできます
  - デフォルトでsession nameは数字です
  - `tmux list-session`で一覧を出せます。
- prefix+`[`でcopy-modeに入ります。この状態ではterminalに表示された内容をさかのぼったりコピーしたりできます。

#### default-term

```
set -g default-terminal 'xterm-256color'
```

tmuxで開いたterminalの`$TERM`にセットされる値です。

Ubuntu 24にしてからaptで入るtmuxのバージョンが上がり、設定しなければ`tmux-256color`が設定されるようになりました。
**`tmux-256color`は古い環境のvimなどに認識されず、表示が崩れます。**
回避策はほかにもある気がしますが、その古いvimでも`xterm-256color`は認識してくれたためこのようにしておきます。

#### prefix

```
set -g prefix C-a
unbind C-b
```

通常のterminal操作をやめて`tmux`にコマンドを送るモードに入るためのキーをprefixと呼びます。

デフォルトでは`Ctrl-b`です。control characterが振られていない組み合わせなら何でもいいはずですが、`Ctrl-a`が押しやすいしよく選ばれます。

#### status bar

参考:
- https://qiita.com/nojima/items/9bc576c922da3604a72b
- https://yiskw713.hatenablog.com/entry/2022/05/17/230443

status barは以下のようになるように設定します。

![tmux-status-bar](/images/dev-ent-2025/tmux-status-bar.png)

prefixが有効時に左から二つ目のcolが明るい色になるようにします。

![tmux-status-bar-prefix-on](/images/dev-ent-2025/tmux-status-bar-prefix-on.png )

一番左のcolはmodeによって文字列が変わるようにしてあります。

![tmux-status-bar-mode-copy](/images/dev-ent-2025/tmux-status-bar-mode-copy.png)

時と場合には寄りますが、neovimを起動するwindowではpaneの分割を行わないため、モードやprefx状態を表示しないと分かりにくいのでこうしてあります。
デフォルトの設定ではtmuxがコピーモードに入ったのはpaneの境界の色がオレンジになり、pane左上にカーソル位置を示す表示が出ることでわかるるのですが、paneを分割しないとすごくわかりづらいわけですね。

この設定でも存外色の変化が視界の端でもわかるため結構気に入ってます。

具体的な設定は下記のようになります

まずstatus barの更新間隔を1秒ごとにします。時計を表示しているので秒でカウントアップするほうが良いからです。

```
set -g status-interval 1
```

status bar全体の色を白灰色に変更。これは参考先そのままパクってます。

```
set-option -g status-bg "colour238"
set-option -g status-fg "colour255"
```
status-leftの最大長を伸ばしておきます。参考リンクでの説明のとおりですがtmuxのstatus barはleft, right, そのほかで別れています。

```
set-option -g status-left-length 100
```

status barの設定は下記のようになります。

基本的に`#[fg=${color},bg=${color}]`で色を指定、それ以外の文字列リテラルはリテラルとして表示、`strftime`互換のトークンと`#H`, `#S`, `#I`などtmux固有のトークンは置き換えが発生します。
`#{?cond,if_true,if_false}`で3項演算子のような振る舞いがありますが、この場合では中に入る文字列の`,`は`#`でのエスケープが必要なようです。

ここではその他に`if`を使っています。これはconditionのmatchするとき次のコマンドを実行、そうでないときさらに次のコマンドを事項というものです。condition部分にtmux固有の表現を用いることができます。
`m`で`fnmatch(3)`を用いるマッチ、`m/r`で正規表現でマッチという意味になるようです。
`if-shell`というコマンドを用いるcondition部分がshellコマンドでよくなるのでこちらを用いたほうが分かりやすいかもしれません。

```
# display mode
STATUS_LEFT_MODE_VIEW="#[fg=colour254,bg=colour2] VIEW "
STATUS_LEFT_MODE_COPY="#[fg=colour254,bg=colour24] COPY "

# prefix key is pressed or not
prefix_on_color="#[fg=colour254#,bg=colour127]"
prefix_off_color="#[fg=colour254#,bg=colour53]"
STATUS_LEFT_PREFIX="#{?client_prefix,${prefix_on_color} ON  ,${prefix_off_color} OFF }#[default]"

# display session
STATUS_LEFT_SESSION="#[fg=colour254, bg=colour241] Session: #S #[default]"

STATUS_LEFT_VIEW=${STATUS_LEFT_MODE_VIEW}${STATUS_LEFT_PREFIX}${STATUS_LEFT_SESSION}
STATUS_LEFT_COPY=${STATUS_LEFT_MODE_COPY}${STATUS_LEFT_PREFIX}${STATUS_LEFT_SESSION}

set-option -g status-left ${STATUS_LEFT_VIEW}

set-hook -g pane-mode-changed 'if -F "#{m/r:(copy|view)-mode,#{pane_mode}}" "set-option -g status-left \"${STATUS_LEFT_COPY}\"" "set-option -g status-left \"${STATUS_LEFT_VIEW}\""'

set-option -g status-right "#[fg=colour254,bg=colour241] #H | %Y-%m-%dT%H:%M:%S #[default]"

setw -g window-status-format "| #I: #W |"
setw -g window-status-current-format "#[fg=colour254,bg=colour67]| #I: #W |"
```

マウスをonにし、copy-mode時にviみたいな挙動になるように設定します。
デフォルトではemacsスタイルらしいです。
さらに、paneの分割、移動がneovimの今の設定とかけ離れないように`s`, `v`でsplit,hjklで移動を割り振ります。

```
set-option -g mouse on
setw -g mode-keys vi

unbind \%
unbind \"
bind-key -T prefix v split-window -h
bind-key -T prefix s split-window

bind-key -r -T prefix k select-pane -U
bind-key -r -T prefix j select-pane -D
bind-key -r -T prefix h select-pane -L
bind-key -r -T prefix l select-pane -R

bind-key -r -T prefix C-k select-pane -U
bind-key -r -T prefix C-j select-pane -D
bind-key -r -T prefix C-h select-pane -L
bind-key -r -T prefix C-l select-pane -R

bind-key -T copy-mode-vi v send-keys -X begin-selection
bind-key -T copy-mode-vi C-v send-keys -X rectangle-toggle
bind-key -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "xsel -bi"
bind-key -T copy-mode-vi Enter send-keys -X copy-pipe-and-cancel "xsel -bi"
```

prefix+`R`に設定のリロードを割り当てます。設定変更中に便利です。

```
bind-key -T prefix R source-file ~/.config/tmux/tmux.conf
```

## neovimの設定

