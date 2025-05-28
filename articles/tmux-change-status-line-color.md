---
title: "tmuxのstatus lineの色をpane modeとhost名に合わせて変える"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tmux"]
published: false
---

## tmuxのstatus lineの色をpane modeとhost名に合わせて変える

っていう話が割と苦労したのでまとめて置きます。

最近sshでサーバーに入って作業することが増えたんですが、ssh先が増えるに伴って自分がどこにいるかがよくわかなくなってしまいます。
また、[neovim]でよく作業するため、pane分割を行わないことがよくあります。その時、自分がどのpane modeにいるのかよくわからなくなってしまうんですね。
そのため表題のようなことをしました。

ちなみに[Tmux Plugin Manager](https://github.com/tmux-plugins/tpm)は使いません。

## 環境

```
$ uname -srvm
Linux 5.15.167.4-microsoft-standard-WSL2 #1 SMP Tue Nov 5 00:21:55 UTC 2024 x86_64
$ tmux -V
tmux 3.4
```

カラーコードを使用する部分が`tmux 3.3a`では動作しなかったので工夫するよりアップデートしてしまえという感じで筆者は手元で最新をビルドして`$HOME/.local/bin`に送り込んで使っています。
サーバーに既に古い`tmux`が入っている場合、それを上書きするとスクリプトが動かなくなるとかあるので自分の使用するやつだけ変えるようにしましょう。

## 前提知識

[tmux](https://github.com/tmux/tmux)の一通りの使用方法

## 完成品がこちらです

以下のような感じ。
左からpane mode, prefix-is-pressed, session name, window name, host name, clockとなっております。

![](/images/tmux-change-status-line-color/view-mode.png)

`tmux`の`prefix`を押すと左から二番目の`OFF`が明るい紫色の`ON`になり、

![](/images/tmux-change-status-line-color/prefix-pressed.png)

`COPY`モードに入ると青系の表示に変更します。

![](/images/tmux-change-status-line-color/copy-mode.png)

## 設定

dotfilesにまとめたいので`~/.config/tmux/tmux.conf`で設定しています。

`tmux`のstatus lineはstatus left, window status, status rightで構成されます。
それぞれ以下のように設定しています。

```shell: ~/.config/tmux/tmux.conf
# status line

set-option -g status-bg "colour238"
set-option -g status-fg "colour255"

run-shell "$SHELL ~/.config/tmux/set_status_left.sh"

setw -g window-status-format "| #I: #W |"
setw -g window-status-current-format "#[fg=colour254,bg=colour67]| #I: #W |"

## set status right based on $(uname -n)
## Using color code(#000000) directly seems requiring tmux 3.4 or above.
run-shell "$SHELL ~/.config/tmux/set_status_right.sh"
```

- `run-shell`を使うと`/bin/sh -c`で渡された文字列を評価します。
  - `$SHELL`を渡すとログインシェルでスクリプトを実行できるのでこうしておいています。
- `tmux.conf`の中で完結したかったですが、bashの分岐処理をがっつり使いたかったためshell scriptとして切り分けました。

```shell: ~/.config/tmux/set_status_left.sh
# display mode
STATUS_LEFT_MODE_VIEW="#[fg=#cdeccd,bg=#5fab5b] VIEW "
STATUS_LEFT_MODE_COPY="#[fg=#bdc1ee,bg=#5b62ab] COPY "

# prefix key is pressed or not
prefix_on_color="#[fg=colour254#,bg=colour127]"
prefix_off_color="#[fg=colour254#,bg=colour53]"
STATUS_LEFT_PREFIX="#{?client_prefix,${prefix_on_color} ON  ,${prefix_off_color} OFF }#[default]"

# display session
STATUS_LEFT_SESSION="#[fg=colour254, bg=colour241] Session: #S #[default]"

STATUS_LEFT_VIEW=${STATUS_LEFT_MODE_VIEW}${STATUS_LEFT_PREFIX}${STATUS_LEFT_SESSION}
STATUS_LEFT_COPY=${STATUS_LEFT_MODE_COPY}${STATUS_LEFT_PREFIX}${STATUS_LEFT_SESSION}

STATUS_LEFT="if -F \"#{==:copy-mode,#{pane_mode}}\"\
  \"set-option -p status-left \\\"${STATUS_LEFT_COPY}\\\"\"\
  \"set-option -p status-left \\\"${STATUS_LEFT_VIEW}\\\"\"\
"

tmux set-option -g status-left-length 100
tmux set-option -g status-left "${STATUS_LEFT_VIEW}"
tmux set-hook -g session-changed "${STATUS_LEFT}"
tmux set-hook -g session-window-changed "${STATUS_LEFT}"
tmux set-hook -g window-pane-changed "${STATUS_LEFT}"
tmux set-hook -g pane-mode-changed "${STATUS_LEFT}"
```

- staus leftの文字列をくみ上げ、`set-option -g status-left`と各種イベントにフックとして登録します。
- `prefix_on_color`, `prefix_off_color`の文字列を見るとわかる通り、`#{?A,B,C}`の分岐処理の中にカンマを含めるには`#`でエスケープが必要なようです。
- `STATUS_LEFT`の部分、`if -F #{==:copy-mode,#{pane_mode}}`でpane modeに対するマッチングを行います。
  - `==:`は見てわかる通り`A == B`で判定を行います。
  - `m`ならglob pattern, `m/r`ならregular expressionで判定できますが、
  - ソースコードをたどらないとわからないですが、pane modeは現状二つしかない([copy-mode](https://github.com/tmux/tmux/blob/3.5a/window-copy.c#L149), [view-mode](https://github.com/tmux/tmux/blob/3.5a/window-copy.c#L160))ため、それらはオーバーキルです。
- tmuxは各種イベント発生時にコマンドを実行する[HOOKS](https://man7.org/linux/man-pages/man1/tmux.1.html#HOOKS)機能を提供します。
- pane mode切り替え時のみならず、session切り替え時、window切り替え時、pane切り替え時をそれぞれフックすることで表示がdesyncするのを防ぎます。
  - しないと切り替えをまたいでも表示が保持されて矛盾した状態になります。

```shell: ~/.config/tmux/set_status_right.sh
HOST_COLOR="$(uname -n | sha256sum | awk '{print substr($1, 1, 6)}')"

r=0x${HOST_COLOR:0:2}
g=0x${HOST_COLOR:2:2}
b=0x${HOST_COLOR:4:2}
brightness=$(((r*299 + g*587 + b*114)/1000))

HOST_FG_COLOR=#FFFFFF
if [ $brightness -gt 128 ]; then
  HOST_FG_COLOR=#000000
fi
HOST_BG_COLOR=$HOST_COLOR

tmux set-option -g status-right-length 100
tmux set-option -g status-right "#[fg=${HOST_FG_COLOR},bg=#${HOST_BG_COLOR}] #H #[fg=colour254,bg=colour241]| %Y-%m-%dT%H:%M:%S #[default]"
```

- さて、status rightですが、host名と時計を表示しています。
  - ごくまれにssh先の時計がずれるため、時計を表示すると役に立つことがあります。
- `uname -n`でhost名を表示し、`sha256sum`の先頭６文字をとることでhost名に対して一意なカラーコードを得ます。
- host名から計算されたカラーコードの[brightnessを計算](https://www.w3.org/TR/AERT/)し、暗い色には白文字、明るい色には黒文字を表示します。
  - この方法自体はchatgptに考案させて出てきたものですが、出てきた設定そのものは動かなかったのでだいぶ手直ししています。
  - chatgptがこう提案しなかったら文字色はhost名から計算された色の補色にしようと思っていましたが、シンプルに白黒のほうが見やすい気がするのでこのほうがよかったなと思います。

## 終わりに

pane modeに対応した色の変更は視界の端でも割と見えるため便利になりました。

host名に対して色が偏るかと思いきや筆者の環境ではいい感じにばらけているため見やすくて非常によくなりました。
status line一部ではなく全体をこの色にしたり、背景色を変更したりするとより気づきやすいかもしれません。

[tmux]: https://github.com/tmux/tmux
[neovim]: https://neovim.io/
