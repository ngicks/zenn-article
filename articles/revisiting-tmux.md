---
title: "tmuxからzellijに移行したけどやっぱtmuxに戻した話"
emoji: "🪟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tmux", "zellij"]
published: true
---

## tmuxからzellijに移行したけどやっぱtmuxに戻した話

zenn上で[zellij]の記事が少なかったので応援記事を書こうと思っていたんですが、結局[tmux]に戻したのでその経緯(何がいいと思って移行したのか、なぜ元に戻すことになったのか)とか`zellij`で便利だった機能を踏まえて変更した`tmux`の設定などについて書きます。

`zellij`をお勧めする記事でもあり、逆に何が(少なくとも筆者にとって)導入を阻害しているかなどを説明します。
`zellij`が気になるけど導入に踏み切れていない人に向けた記事になるのかなと思います。

## 経緯

### 違い

[tmux], [zellij]はいわゆるtermianl multiplexer, 1つのterminal windowを複数に分割して複数のterminalを操作したり、セッション管理を行うことでsshログイン先で実行したコマンドを中断せずにデタッチできるようにしたりします。

おおむね機能的には同じですが、両者は以下が異なります。

- 画面見たときの分かりやすさ
  - `tmux`は操作を覚える前提で画面に情報が少ない
  - `zellij`は画面にある程度操作方法が表示されている
- 開発言語
  - `tmux`はC言語でconfigureしてmakeしてビルド
  - `zellij`は[Rust]で`cargo`でビルドするので依存関係の管理がそこで完結する。
- ドキュメント
  - `tmux`は`man tmux`で完結
  - `zellij`は[ドキュメントサイト](https://zellij.dev/documentation/introduction.html)がある
- 設定
  - `tmux`はshell scriptみたいなやつ(明確な文法は筆者にはよくわかっていません)
  - `zellij`は[kdl](https://github.com/kdl-org/kdl)
    - [hcl](https://github.com/hashicorp/hcl)に近いような構文を持つ設定用の文法みたいです。
- out-of-box体験
  - `tmux`はカスタマイズ必須だと思います
  - `zellij`は割と設定なしでも使えます。(motion周りは変えたほうがいいかも)
- mode数
  - `tmux`は`prefix`(デフォルトで`Ctrl-b`)があり、これに続いて個別のキーを打つことで`tmux`固有の機能を使用します。`tmux`には通常modeと`copy mode`があり、`copy mode`でterminalをスクロールしたり、コピーしたりします。
  - `zellij`には[たくさんmodeがあります](https://zellij.dev/documentation/keybindings-modes.html)。`normal`/`pane`/`resize`/`tab`/`scroll`/`session`みたいな感じで、操作対象に合わせてmodeを切り替えて、modeによってキーに対応する機能が変わる感じです。
    - mode間で共通のバインドがデフォルトでたくさんあり、これがほかのアプリと被ることがあります。例えばデフォルトでは`Ctrl-t`で`Tab` modeに入れますが、これは`codex`のtranscript modeのbindと被っています。かぶってる場合に備えて、ほとんど何のバインドもしない`locked` modeがあります。

### どうしてzellijに移行したのか,何が優れていたか

初心者フレンドリー+コラボレーションのしやすさの観点から布教しやすいかなって思ったので導入しました。

- [zellij]は[tmux]に比べると初心者フレンドリー
  - `zellij`は画面を見ただけである程度操作が表示されている。覚えなくてもある程度使える。
- `kdl`でlayoutを記述しておき、それ呼び出せる。
  - 例えば、paneを縦横に切って4分割し、左上でzennと連携したrepositoryを`neovim`で開いて、左下で`zenn preview`を実行する場合、[こういうlayoutにします](https://github.com/ngicks/dotfiles/blob/0941c63228dd5684335977f181b6eb409ad7b0a5/.config/zellij/layouts/zenn-editor.kdl)
  - `${XDG_CONFIG_BASE:-~/.config}/zellij/layouts`などにファイルを置くと、`zellij -l zenn-editor`とかで開けます。
- plugin主体のアーキテクチャで拡張がしやすそう
- [wasm](https://webassembly.org/)でpluginが書ける
- [web client機能](https://zellij.dev/documentation/web-client.html): ブラウザから`zellij`セッションに参加できる。
  - `ssh`しなくても[tailscale](https://tailscale.com/)などをつないでおけばログインできる。
- セッション内で複数のクライアントがいたとき、クライアントごとに別々のpaneを操作できる
  - 逆に`tmux`ではできない（全員paneの切り替えなどを共有してしまう)

### どうしてtmuxに戻したのか

- 画像が表示できない:
  - [sixel](https://en.wikipedia.org/wiki/Sixel)や[kitty Terminal graphics protocol](https://sw.kovidgoyal.net/kitty/graphics-protocol/)などのプロトコルを使うとterminalでも画像の表示ができます。
  - [ここ](https://github.com/folke/snacks.nvim/blob/fe7cfe9800a182274d0f868a74b7263b8c0c020b/docs/image.md)などで言及されていますが、`tmux`はこれらのプロトコルをpassthroughする設定がありますが[zellijはまだ未対応です](https://github.com/zellij-org/zellij/issues/775)。nestした`zellij` sessionの取り扱いにために必要であるため、作者自身にも実装する動機が存在するみたいです
  - (WSLgなどのおかげで適当な画像ビューワーをterminalから起動するのでも別に構わないんですが、フローが途切れるのと、GUIの転送は`ssh`経由だと気になるほど遅くなるので避けたいんですよね。terminalで画像を表示するプロトコルはescape sequenceの後にbase64エンコードされた画像を文字列で送る形式なので多分こっちのほうがオーバーヘッドが少ないです。)
- 外部からコマンドを送るときにpaneが指定できない([#4474](https://github.com/zellij-org/zellij/issues/4474))
  - セッション内から各種コマンドを実行すると問題なく動作しますが、セッション外からコマンド実行する場合、どこで実行するかを指定できません。
  - `zellij run --floating -- ${command}`でコマンドをfloating paneで実行するときに、paneを指定できないため、最初のclientが現在focusしているtabで実行されてします。
  - [zellij action write-chars](https://zellij.dev/documentation/cli-actions.html#write-chars)というpaneに文字を打ち込むcliコマンドがpaneを指定できない
  - (`zellij -s ${session_name}`でセッションの指定はできる)

`write-chars`でpaneを指定できないのが致命的で、スクリプトから一気にpaneをいじるようなことができなくなっています。
以前の記事([zellij floating windowをpinentry-cursesのフロントエンドにする](https://zenn.dev/ngicks/articles/pinentry-in-zellij-floating-window))で説明した通り筆者は`zellij`のfloating paneを`pinentry-curses`のフロントエンドとして利用しているんですが、指定したpane(tab)にfloating paneを表示する機能がないためちょっと困っているのもあります。

## tmuxでもzellijで便利だったところを使いたい

前述の経緯から[tmux]を使う決断を下したわけですが、`zellij`にはずいぶん便利な挙動がありました。
ですので`tmux`に戻るにしろいいとこどりができるように適当に設定を変えてみました。

### Fullscreen

[zellij]では`PANE` modeで`f`を押すとfocusされたpanが画面いっぱいに広がってほかのpaneが非表示になる機能があります。
これが思いのほか便利だったため、[tmux]でも利用したいと思います。

これに当たるのは`tmux`ではzoomです。デフォルトでは`prefix >`で表示されるメニュー上でしかbindが存在しないため、適当に設定します。

```diff tmux.conf: ~/.config/tmux/tmux.conf
+unbind -T prefix f
+bind-key -T prefix f resize-pane -Z
```

`prefix f`はデフォルトだと`search window`がふられていますね。

`zellij`ではfullscreen時はtabのの名前の末尾に`(FULLSCREEN)`と表示されて分かりやすいです。この挙動をパクるためにはstatus rightにzoomかnormalなのかを表示することにしましょう。

```bash: ~/.config/tmux/set_status_right.sh
. ~/.config/tmux/color_scheme.sh

tmux set-option -g status-right-length 100

zoom_on_color="#[fg=${_tmux_color_text_dark}#,bg=${_tmux_color_5}]"
zoom_off_color="#[fg=${_tmux_color_text_dark}#,bg=${_tmux_color_4}]"

STATUS_RIGHT_ZOOM_FLAG="#{?window_zoomed_flag,${zoom_on_color}[Zoomed],${zoom_off_color}[Normal]}#[default]"
STATUS_RIGHT_TIMER="#[fg=${_tmux_color_text_light},bg=${_tmux_color_1}] %Y-%m-%dT%H:%M:%S #[default]"

STATUS_RIGHT="${STATUS_RIGHT_ZOOM_FLAG}${STATUS_RIGHT_TIMER}"

tmux set-option -g status-right "${STATUS_RIGHT}"
```

(`tmux.conf`から呼び出してます。)

```tmux.conf:~/.config/tmux/tmux.conf
run-shell "$SHELL ~/.config/tmux/set_status_right.sh"
```

別にtab名を変更してもいいんですが、状態で幅ががたがた変わるのはちょっと嫌かなと思ったのでこうしています。

### Edit Scrollback

[zellij]には`Edit Scrollback`といってpaneに表示している内容を`${EDITOR:-${VISUAL:-vim}}`もしくは設定ファイルで指定したeditorで開く機能があります。

[tmux]では`copy mode`に入ると`emacs`/`vi`の操作バインドでterminal内にカーソルを移動させてコピーを行えますが、`Edit Scrollback`はさらにこれを超えてあなたの好みのエディター(`neovim`など)でコピーを行うことができます。
これがすごく便利であったので、[tmux]でも使えるようにします。

やり方はシンプルで`tmux capture-pane`でpaneの内容を一時ファイルに吐き、これをeditorで開くというものです。
`zellij`とは違い、下記スクリプトでは`tmux display-popup`で開いたpopup paneの中でeditorを開きます。
`zellij`ではその場でpaneをeditorに差し替えます。これが`tmux`で再現できるのかわからなかったのと、使っていて案外紛らわしいのでpopupにしたほうが明快だろうという判断です。

```bash
#!/bin/bash
set -euo pipefail

pane_id="${1:-${TMUX_PANE:-}}"
if [[ -z "${pane_id}" ]]; then
  echo "No pane id provided" >&2
  exit 1
fi

c_opt=""
if [[ ! -z "${2:-}" ]]; then
  c_opt="-c ${2}"
fi

scrollback_file="$(mktemp -t tmux-scrollback.XXXXXX)"
trap 'rm -f "${scrollback_file}"' EXIT

tmux capture-pane -p -J -S -32768 -t "${pane_id}" > "${scrollback_file}"

editor="${EDITOR:-${VISUAL:-vi}}"
line_count=$(wc -l < "${scrollback_file}")
if [[ "${line_count}" -eq 0 ]]; then
  line_count=1
fi

read -r -a editor_cmd <<< "${editor}"
line_arg=""
case "$(basename "${editor_cmd[0]}")" in
  vi|vim|nvim) line_arg="+${line_count}" ;;
  nano) line_arg="+${line_count},1" ;;
esac
if [[ -n "${line_arg}" ]]; then
  editor_cmd+=("${line_arg}")
fi
editor_cmd+=("${scrollback_file}")
printf -v editor_cmd_str ' %q' "${editor_cmd[@]}"

tmux display-popup -w 90% -h 90% $c_opt -E "sh -c '${editor_cmd_str:1}' --"
```

`vi`/`vim`/`nvim`/`nano`はファイルを開くときのカーソル位置をcli optionで指定できますが他のエディターの設定の仕方はよく知らないので未対応になっています。
そもそも筆者は`vim`/`nvim`(`neovim`)しか使わないのでこれで問題ありません！

`copy-mode-vi`の`e`にしたかったんですが、すでにほかのコマンドがあるので`E`しておきました。

```diff tmux.conf: ~/.config/tmux/tmux.conf
+bind-key -T copy-mode-vi E run-shell "~/.config/tmux/edit_scrollback.sh #{pane_id}"
```

### pane/tabの作成がcwdを引き継ぐように

[zellij]ではデフォルトで新しいpane/tabを作成するとき、作成元の`cwd`を引き継ぐ挙動があります。

```diff tmux.conf: ~/.config/tmux/tmux.conf
+bind-key -T prefix v split-window -h -c "#{pane_current_path}"
+bind-key -T prefix s split-window -c "#{pane_current_path}"
+bind-key -T prefix c new-window -c "#{pane_current_path}"
```

### dragで選択部分がクリップボードにコピーされる挙動を消す。

[tmux]のデフォルトがこうなのですが、マウス操作とハイブリッドでやってると、うっかりターミナルをクリックしてしまってコピーバッファが吹き飛んだりして、気を遣うのでなくしたほうがいいなと[zellij]を使ってて思いました。

```diff tmux.conf: ~/.config/tmux/tmux.conf
+unbind -T copy-mode-vi MouseDragEnd1Pane
+unbind -T copy-mode MouseDragEnd1Pane
```

### sync-modeのbind

[zellij]だと`Tab` modeの`s`がsync modeに割り当てられていて、ずいぶん便利だなと思わされました。
[tmux]ではデフォルトだと特にbindがないため`prefix :`でプロンプトを表示して`set-window-option synchronize-panes on`と打ち込む必要があって面倒で使っていませんでした。
やっぱbindしたほうがいいですね(あたりまえ)

```diff tmux.conf: ~/.config/tmux/tmux.conf
+bind-key -T prefix S set-window-option synchronize-panes #{?pane_synchronized,off,on}
```

末尾の`#{?pane_synchronized,off,on}`はそもそもなくても動作する気がしますが、試してはいません。

## さいごに

- [zellij]も[tmux]も素晴らしいソフトウェアですが、歴史の長さの違い(`tmux`は[2007-07-09](https://github.com/tmux/tmux/blob/4b810ae4932367afc8509bc8105603a8d965b0b8/CHANGES#L3848), `zellij`は[2021-01-21](https://github.com/zellij-org/zellij/releases/tag/v0.1.0-alpha)にそれぞれ最初のリリース)から`tmux`のほうが細かい困りごとはつぶされている印象です。
- 今回は細かい挙動が決め手となって`zellij`から`tmux`に戻りましたが、操作系の違いからいろいろ思わされるところがあって今回のような設定の変更を行ったのでために別のツールに切り替えるのは相対的なものの見方ができていいかもしれないです。
- どちらかというと初心者フレンドリーなのは`zellij`なので布教するさいは`zellij`を推すといいかもなと思います。

[tmux]: https://github.com/tmux/tmux/wiki
[zellij]: https://github.com/zellij-org/zellij
[Rust]: https://rust-lang.org/
