---
title: "tmux popupのなかでpinentry-cursesを呼び出す"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "tmux"]
published: true
---

## tmux popupのなかでpinentry-cursesを呼び出す

出来上がったものがこちらになります。

![](/images/pinentry-in-tmux-popup/popup.png)

コードはこちら

https://github.com/ngicks/run-in-tmux-popup

## Why?

- [claude code](https://docs.anthropic.com/ja/docs/claude-code/overview)の様子を見にandroid端末からPCにsshして止まってたらたたくような機会が増えました。
- ポートをインターネットにあけるようなことをしなくても[tailscale](https://tailscale.com/)でVPNをつないで容易に仮想的にLAN内にいるかのように見せかけることができます。
- androidでは[termux](https://github.com/termux/termux-app)を使ってterminal環境を整えています。
- せっかくなのでsshした先で`git push`を行ったりしたいと思います。
- git commitはgpgで署名を行います([ここなど参考に](https://docs.github.com/ja/authentication/managing-commit-signature-verification/signing-commits))。OSSの貢献受付条件に署名が必須なことがあるのでつけておいたほうが良いです。
- 署名には証明書を使います。証明書には`passphrase`と言ってパスワードをかけておくことが多いです。
- `passphrase`は基本的に[pinentry](https://www.gnupg.org/related_software/pinentry/index.html)で入力します。これはパスフレーズ専用のUIを提供するプログラムです。
- [ソースを見ればわかりますが](https://github.com/gpg/pinentry)はダイアログの表示方式でいくつか実装が分かれます。
  - TUI(Terminal UI)で表示する`pinentry-tty`と`pinentry-curses`
  - GUIで表示する`pinentry-qt`, `pinentry-gnome`, `pinentry-gtk`などがあります。
- 筆者は普段GUIで表示する`pinentry-qt`を使っています。
- linuxは古くからX11(X Window System)というネットワーク透過な(ローカルでもネットワーク経由でも同じように扱える)方式でGUIを表示していました。
- sshはX11をforwardできるのでネットワーク経由でもGUI付きアプリを表示できます。
- ただし`termux`でGUI(X11アプリケーション)を動かすのはまだnightly buildが必要らしく、熱心にこれを追いかけられる自信はないので安定するまで避けておこうと思っています。
- そのため、androidの`termux`からログインしているときに限りTUIで`pinentry`を表示したいわけですが
- ssh > tmux > nvim(toggle term) > [lazygit](https://github.com/jesseduffield/lazygit)で表示しているときに`pinentry-curses`がterminal stateを壊すので快適ではありません。
  - `GPG_TTY`環境変数を別のpaneに指定しておくとそのpaneでpinentry-cursesの入力が行えるので、破壊を免れることができます。
- ところで、[tmux](https://github.com/tmux/tmux/wiki)には[3.2](https://raw.githubusercontent.com/tmux/tmux/3.2/CHANGES)から追加された`display-popup`という機能があり、モーダルのようなものを表示できるようになっています。
- これで`pinentry-curses`を表示できたらいかなるterminalアプリのterminal stateを破壊しないでいられるのでは？と考えるに至ります。
- 案外面倒な課題があったため記事として書き留めておきます。

## 失敗した試み

### `$DISPLAY`や`$TMUX`環境変数で分岐する

#### 失敗

単純な発想として、`~/.gnupg/gpg-agent.conf`の`pinentry-program`の項目にラッパースクリプトを指定し、その中で工夫することが考えられます。

```conf: ~/.gnupg/gpg-anget.conf
pinentry-program /path/to/wrapper.sh
```

ラッパースクリプトの中で`$DISPLAY`などを判定することで、GUIを使うものとそうでないものを分岐します。

```bash: wrapper.sh
if [ -n $DISPLAY ]; then
  exec pinentry-qt "$@"
fi

if [ -n $TMUX ]; then
  exec wrapper_for_tmux_popup.sh
fi

pinentry
```

これが失敗しているのは以下で

- `$DISPLAY`が設定されていることがX11アプリを表示できる環境にいることを示すわけではない
  - `tmux`の環境は同じ環境に複数のクライアントから接続できるため、環境変数は割と当てになりません。
- `$TMUX`環境変数は`tmux`セッション中では設定されていますが、`gpg-agent`を呼び出す[pass](https://www.passwordstore.org/)を経由してこのスクリプトが実行されたときには設定されていません。

#### 改善策: `PINENTRY_USER_DATA`を利用して分岐する方式へ

以下のリンクより、`PINENTRY_USER_DATA`という環境変数はpinentryプログラムにわたってくるため、これによって分岐することができます。
自動的な分岐は諦め、任意の情報をここに納めて分岐することにします。

https://github.com/gpg/gnupg/blob/gnupg-2.5.8/doc/gpg.texi#L4208-L4211

例えば以下のような感じです。

```bash: wrapper.sh
#!/bin/bash

set -Ceu

case "${PINENTRY_USER_DATA-}" in
*TTY*)
  exec pinentry-curses "$@"
  ;;
*TMUX_POPUP*)
  exec wrapper_for_tmux_popup.sh
  ;;
esac

exec pinentry-qt "$@"
```

呼び出す側が環境変数を設定して分岐させます。

```
$ pass git/...
# pinentry-qtのGUIが起動
$ PINENTRY_USER_DATA=TTY pass git/...
# pinentry-cursesが起動
```

### bashscriptで頑張る

#### 失敗

詳細はソースをコミットせず消してしまったため散逸してしまっていますが以下を参考に、bashとしていろいろやるように工夫しました

- https://qiita.com/ko1nksm/items/897ba32ea07949d1d0e4
- https://qiita.com/ko1nksm/items/b33d0bfc426cf7f1b5f7

```bash
#!/bin/bash

set -Ceu

case "${PINENTRY_USER_DATA-}" in
*TTY*)
  exec pinentry-curses "$@"
  ;;
*TMUX_POPUP*)
  # mdtemp -dで一時ディレクトリを作って、
  tempdir=''
  cleanup() {
    [ "$tempdir" ] || return 0
    rm -rf "$tempdir"
  }
  trap "cleanup" EXIT
  tempdir=$(mktemp -d)
  # mkfifoでnamed pipeを作って、tmux popupで動作させたshellのttyを送信させて
  fifo="$tempdir/fifo"
  mkfifo $fifo
  tmux popup -e parent_fiio=$fifo -E "echo \$(tty) >> \${parent_fiio} && $SHELL" &
  read -r popup_tty < $fifo
  # 取得したttyを指定してpinentry-cursesを起動
  export GPG_TTY=$(popup_tty)
  pinentry-curses -T $popup_tty "$@"
  exit
  ;;
esac

exec pinentry-qt "$@"
```

これが失敗しているのは以下で

- 実は`GPG_TTY`が設定されていると[この行](https://github.com/gpg/gnupg/blob/gnupg-2.5.8/agent/call-pinentry.c#L473)より[Assuan protocol](https://www.gnupg.org/documentation/manuals/assuan/Introduction.html)で直接ttyが指定されるため、`-T`オプションが意味がない
- `GPG_TTY`環境変数は`pass`などを呼び出した環境のものが使われるため、ラッパースクリプトないから上書きできない。
- bashscriptで複雑なstdin/stdoutのパイプと介入は難しい。
  - 筆者が不慣れなだけではあります。

このスクリプトをclaude codeに改善させようとしたんですが点でダメなものが出て困りました。
TUIをclaudeに見せて評価する方法がはっきりわからなかったのでフィードバックかかりにくくてダメだったみたいです。こういうタスクの任せたかは要研究ですね。

#### 改善策: プログラムを書いてしまう・popup内でpinentryを動作させる

- bashscript諦めます。プラグラム書いてコンパイルします。
  - 複数のストリームに介入して内容をフィルターしたりするのはbashより表現が簡単だと思います。
- popup内でpinentryを動作させる
  - shellとしてpinentryの内容を表示してしまうと復号されたパス情報が表示されるため、pinentry自体の内容が表示されないように工夫が必要です。

## Goでtmux popup内でpinetnryを呼び出すラッパーを記述する。

ということで実装します。

やらないといけないこと:

- tmuxのpopupは個別にttyが割り当てられるみたいなのでこれを取得し、そのttyをターゲットにpinentryを起動します。
- 前述通り、Assuan protocolで直接`OPTION ttyname=`が指定されるため、プログラムはこの行をトラップして内容を書き換える必要があります。
- なんとかしてtmux popupをpinentryが終了するまで生き残らせる必要があります。

方針:

- fifo(named pipe)を作成してこれを環境変数経由でファイル名をtmux popup内のプログラムに渡します。
  - tmux popup内のプログラムにfdを渡す手段はないようです(fork直後にstdin/stdout/stderr以外をすべてcloseするお約束をしています。実装を確認してください。)
  - 環境変数は`-e`オプションで渡したものが最優先されます。(実装を確認してください。)
  - tmux popupに渡すコマンドに変数を入れてしまうとおそらく`ps e`などで閲覧されてしまいます。
    - `$SHELL -c $command`でコマンドが実行されるためです。(実装を確認してください)
  - プログラムクラッシュ時に環境変数がプリントされるとはよく言われますが、すべての環境変数はttyの名前を送信するまで秘されていればよいだけなのでここは問題になりません。
- fifoは`os.MkdirTemp`で作成したディレクトリの下に作成します。
  - デフォルトで`/tmp`(unix系の場合)以下に`0o700`のpermissionでディレクトリが作成されます。
  - sticky bitがついているので`/tmp`以下はだれでもフォルダが作れますが、作成したユーザー以外はディレクトリのrenameができませんのでhijackの心配は基本ないと思います。
- fifoからttyを取得するときランダムなprefix, suffixを加えます。
  - 一応のfifo hijack対策です。

bashで頑張るのをやめると簡単ですね！そのうちbashでもすいすいやれるようになりたいです。

```go: tmux-popup-pinentry-curses
package main

import (
	"bufio"
	"context"
	"crypto/rand"
	"encoding/hex"
	"errors"
	"fmt"
	"io"
	"log/slog"
	"os"
	"os/exec"
	"os/signal"
	"path/filepath"
	"strings"
	"sync"
	"syscall"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
	defer cancel()
	ctx, stop := signal.NotifyContext(ctx, syscall.SIGINT, syscall.SIGTERM, syscall.SIGABRT)
	defer stop()

	tempdir, err := os.MkdirTemp("", "")
	if err != nil {
		panic(err)
	}
	defer func() {
		_ = os.RemoveAll(tempdir)
	}()

	logger := slog.New(slog.NewTextHandler(os.Stderr, nil))
	if os.Getenv("TMUX_POPUP_DEBUG") == "1" {
		logFile, err := os.OpenFile(
			filepath.Join(tempdir, "log.txt"),
			os.O_APPEND|os.O_CREATE|os.O_RDWR, 0o700,
		)
		if err != nil {
			panic(err)
		}
		defer logFile.Close()

		logger = slog.New(slog.NewTextHandler(logFile, &slog.HandlerOptions{Level: slog.LevelDebug}))
	}

	ttyFifo := filepath.Join(tempdir, "tty")
	doneFifo := filepath.Join(tempdir, "done")

	for _, s := range []string{ttyFifo, doneFifo} {
		err = syscall.Mknod(s, syscall.S_IFIFO|0o600, 0)
		if err != nil {
			panic(err)
		}
	}

	logger.Debug("tty fifo created")

	// Just little counter measurement for fifo hijack.
	// Adding random generated prefix and suffix to info
	// to detect suspicious sender
	var prefBytes, sufBytes [16]byte
	_, err = io.ReadFull(rand.Reader, prefBytes[:])
	if err != nil {
		panic(err)
	}
	_, err = io.ReadFull(rand.Reader, sufBytes[:])
	if err != nil {
		panic(err)
	}

	pref := hex.EncodeToString(prefBytes[:])
	suf := hex.EncodeToString(sufBytes[:])

	popupCmd := exec.CommandContext(
		ctx,
		"tmux", "popup",
		"-e", "TTY_FIFO_FILE="+ttyFifo,
		"-e", "DONE_FIFO_FILE="+doneFifo,
		"-e", "SEC_PREFIX="+pref,
		"-e", "SEC_SUFFIX="+suf,
		"-E", "echo ${SEC_PREFIX}$(tty)${SEC_SUFFIX} >> ${TTY_FIFO_FILE} && read done < ${DONE_FIFO_FILE}",
	)
	popupCmd.Cancel = func() error {
		return popupCmd.Process.Signal(syscall.SIGTERM)
	}

	err = popupCmd.Start()
	if err != nil {
		panic(err)
	}
	defer func() {
		done, err := os.OpenFile(doneFifo, os.O_RDWR, 0)
		if err != nil {
			panic(err)
		}
		defer done.Close()
		done.Write([]byte("done\n"))
	}()

	f, err := os.Open(ttyFifo)
	if err != nil {
		panic(err)
	}
	defer f.Close()

	scanner := bufio.NewScanner(f)

	scanner.Scan()
	t := scanner.Text()
	t, ok := strings.CutPrefix(t, pref)
	if !ok {
		panic(fmt.Errorf("suspicious sender: incorrect prefix"))
	}
	targetTty, ok := strings.CutSuffix(t, suf)
	if !ok {
		panic(fmt.Errorf("suspicious sender: incorrect suffix"))
	}

	logger.Debug("tmux popup started")

	if targetTty == "" {
		panic("empty tty")
	}

	logger.Debug("got TTY from popup")

	cmd := exec.CommandContext(ctx, "/usr/bin/pinentry-curses", os.Args[1:]...)
	cmd.Cancel = func() error {
		return cmd.Process.Signal(syscall.SIGTERM)
	}

	p, err := cmd.StdinPipe()
	if err != nil {
		panic(err)
	}

	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	err = cmd.Start()
	if err != nil {
		panic(err)
	}

	logger.Debug("pinentry-curses started")

	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		defer wg.Done()
		defer p.Close()

		scanner := bufio.NewScanner(os.Stdin)
		for scanner.Scan() {
			line := scanner.Text()

			// Replace ttyname option with the popup's TTY
			if strings.HasPrefix(line, "OPTION ttyname=") {
				original := line
				line = "OPTION ttyname=" + targetTty
				logger.Debug("replaced ttyname", slog.String("old", original), slog.String("new", line))
			}

			logger.Debug("forwarding input", slog.String("line", line))

			_, err := p.Write([]byte(line + "\n"))
			if err != nil {
				logger.Warn("write error", slog.Any("err", err))
				break
			}
		}

		if err := scanner.Err(); err != nil {
			logger.Warn("scanner error", slog.Any("err", err))
		}
	}()

	err = cmd.Wait()
	if err != nil {
		var execErr *exec.ExitError
		if errors.As(err, &execErr) {
			err = fmt.Errorf("%v: stderr = %s", execErr, string(execErr.Stderr))
		}
	}

	logger.Debug("pinentry-curses finished", slog.Any("err", err))

	os.Stdin.Close()

	wg.Wait()

	if err != nil {
		panic(err)
	}
}
```

ちょっとしたポイント

- `os.Open(fifo)`で開く(=O_RDONLYで開く)と誰かが書き込みモードで開くまでブロックします([fifo(7)](https://man7.org/linux/man-pages/man7/fifo.7.html))
- pinentry-cursesがttyの内容をコントロールするため別にpopup内で特に何かを実行する必要はありません。
- `tmux popup -E "read done < ${DONE_FIFO}"`のようにfifoの読み込みでpopupをブロックさせます。
  - `read done`の`done`を省略すると終了時、popupに一瞬エラーメッセージが表示されてしまうので受けておいたほうがよさそうです。
- `pinentry-qt`はほっとくとそのうちタイムアウト終了するのでそれに倣って一応2分のタイムアウトを入れておきます。

## ビルドして配置、ラッパースクリプトを書く

ビルドして適当に`$PATH`経由で見つかるところに置きます。
`~/.local/bin`と`~/bin`はスタートアップ時に`.profile`などで存在確認がされて、あると`$PATH`に加えられるようになっているのでここに置いておきます。

```
$ go build ./cmd/tmux-popup-pinentry-curses
# copy to somewhere included in $PATH
$ mv tmux-popup-pinentry-curses ~/.local/bin
```

ラッパースクリプトを書きます。

```bash: wrapper.sh
#!/bin/bash

set -Ceu

case "${PINENTRY_USER_DATA-}" in
*TTY*)
  exec pinentry-curses "$@"
  ;;
*TMUX_POPUP*)
  exec $HOME/.local/bin/tmux-popup-pinentry-curses "$@"
  ;;
esac

exec pinentry-qt "$@"
```

`gpg-agent.conf`でこのスクリプトを指定します。

```conf: ~/.gnupg/gpg-agent.conf
pinentry-program /path/to/wrapper.sh
```

## 完成！

再び同じ画像ですが以下のように完成しました。

![](/images/pinentry-in-tmux-popup/popup.png)

余白いっぱいありますけど筆者は気にしません！

## おわりに

これでX11アプリをサポートしない環境からSSHで開発環境に入ってneovimを起動してtoggle term経由のlazygitでpinentry-cursesが呼び出された場合に起こるterminal stateの破壊を防ぐできるようになって快適になりました。

ググった限りではtmux popupをpinentryのフロントエンドに使うなにかを見たことがなかったので作ってみました。
多分見つけてないだけでだれかしているとは思いますが、まあここまでくると頑張って探すより自分で作ったほうが早そうだったので作ってしまいました。

セキュリティー的に安全なのかはよくわかんないので使われる場合は自己責任でお願いします。(問題ないとは思いますが)
