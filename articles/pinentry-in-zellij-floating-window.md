---
title: "zellij floating windowのなかでpinentry-cursesを呼び出す"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "zellij"]
published: false
---

## zellij floating windowのなかでpinentry-cursesを呼び出す

下記記事のzellij版です

https://zenn.dev/ngicks/articles/pinentry-in-tmux-popup

下の画像のようになります。

![](/images/pinentry-in-zellij-floating-window/popup.png)

## But Why?

https://zenn.dev/ngicks/articles/pinentry-in-tmux-popup

で説明したとおりですが、要するに

- いろんな環境から開発環境に入れるようにするためにzellijを使う。
  - [tailscale](https://tailscale.com/)などのVPNソリューションを使って仮想的LAN内にお互いがある状態にし、端末から開発環境にsshします。
- zellijはterminal multiplexer/session managerであり、作成したterminal sessionに複数のクライアントからログインできるようにします。
- 開発すればgit commitもします。commitには基本的に鍵を使ってsignを行います。
- signする際に[pinentry](https://www.gnupg.org/related_software/pinentry/index.html)を利用して、鍵自体にかかっているパスワード(passphrase)を入力します。
- pinentryにはx11やwaylandのGUIを利用するものとttyを利用するものがあります。
- ttyを使うものは、TUI持つアプリと衝突して表示を壊すことがあります。
- ではGUIを使うpinentryのみを使っておけばよいかと思うかもしれませんが、
  - GUIのない環境からもログインすることが増えました。
    - androidタブレットの[termux](https://github.com/termux/termux-app)などです
  - GUIをsshなどを介して転送するのは結構遅くて気になるのでterminalですむならそのほうが良いです。
- zellijのfloating windowにpinentryのプロンプトを表示すれば、terminal stateを壊さず、かつGUIなしでいい感じに入力できる。

ってことです

筆者は半年ほどtmuxをカスタマイズしながら利用していました。別にtmuxに全く不満はないのですが、zellijに乗り換えたのは、もしかしたらzellijのほうが他人にお勧めしやすいかなと思ったからです。

- 画面を見ただけで操作がわかりやすい
  - tmuxは(素では)操作をすべて覚えてなんぼのつくり
  - zellijはある程度の操作を画面上に表示する
- 開発言語が新し目で読みやすい
  - tmuxはCですが、
  - zellijは[Rust](https://www.rust-lang.org/)です。ビルド環境がばらけにくく、高級な機能もあるため、Cに慣れてなくて読みやすいです(たぶん・・・)。
- ドキュメントが読みやすい
  - tmuxはでけえマニュアル([tmux(1)](https://man7.org/linux/man-pages/man1/tmux.1.html))が出てきてグワーッてなりますが、
  - zellijは`Rust`のライブラリ/ツールでよく見るフォーマットのドキュメントサイトがあって読みやすい

ということで、zellijも使いこなせるようになっておこうかと思った次第です。
前の記事で行ったようなカスタマイズがzellijにもないと辛かったため、こうして作成しました。

## Zellijとは

https://github.com/zellij-org/zellij

[tmux](https://github.com/tmux/tmux/wiki)と同じくterminal multiplexer/session managerです。

tmuxと違って

- 設定ファイルが[kdl](https://github.com/kdl-org/kdl)
  - yamlとかtomlに似たデータフォーマットだと思えばよいです。
  - 構文的には[hcl](https://github.com/hashicorp/hcl)が近いです。
  - tmuxの設定ファイルはshell script的なものだったのでだいぶ違います。
  - (たぶん)演算や分岐を任意にできませんが、その分どこ見たら何書いてあるか把握しやすいです。
- デフォルトのコピーモードが画面の内容を一時ファイルにダンプして`$EDITOR`もしくは`$VISUAL`で編集するものである
  - 設定でエディターは決められます。
  - 設定しないと`$EDITOR` or `$VISUAL`で、それらが空の場合は`vim`にフォールバックします。
  - 普段使いのneovimの設定だと起動がちょっと遅いので軽量版設定詰めようかな・・・みたいな気持ちになってしまいますね。
- 分割されたpaneが小さくなると自動的にstackされる
- wasmでプラグインが書ける(!)
- ブラウザからsessionに接続できる(!)
- tmuxのhookにあたるものがない(っぽい)
- `zellij run`で新しいpaneでコマンドを実行するとき、`tmux popup`の`-e`オプションのような環境変数を渡すオプションがない
  - [#4031](https://github.com/zellij-org/zellij/issues/4031)

機能的な差は[#376](https://github.com/zellij-org/zellij/issues/376)で一覧になっています。
触ってると結構tmuxとは違ってて全く同じことができるわけでもないみたいですが、よくできてておもろくてすごいです。

## 設計方針(前回との重複あり)

- `zellij run --session=${session_name} --floating -- ${command}`でfloating modeのpaneを表示し、それのpty(pseudo-teletypewriter=ターミナルのデバイス)をpinentryにコントロールさせる
  - `zellij run`で新しいpaneで`${command}`を実行します。
  - floating modeはtmux popupみたいなものだと思えばよいです。実際にはいろいろ挙動が違いそうです。
  - session内から実行する場合は`--session`オプションは不要です
  - 試してみたところ表示されたterminalには(当然ではありますが)個別のptyが振られていました。
  - 前の記事で述べた通り、`pinentry-curses`はttyのコントロールをとって画面を表示します
- floating windowの中で、[tty(1)](https://man7.org/linux/man-pages/man1/tty.1.html)を実行して、その結果をpinentry呼び出し元に送る
  - floating windowのttyのデバイスパスを取得するには、その中で`tty(1)`実行するのが最も手軽です。
  - pinentryの呼び出しそのものは、floating window内で行われることはないと思ってよいです。
    - gpg-agentを呼び出すのは既存のターミナルで表示しているアプリであるからです。
- pinentryのstdinトラップして取得した`tty`をpinentryのttyとして指定する。
  - `pinentry`は[Assuan protocol](https://www.gnupg.org/documentation/manuals/assuan/Introduction.html)でpinentry呼び出し元 <--> pientryのIPCを行います。
    - これでタイトルとか、プロンプトに表示されるメッセージとか、キャンセルボタンを表示するかとか、リトライ回数とかを制御します。
  - 基本的にstdin/stdoutを介して行います。
  - [GPG_TTY](https://man.archlinux.org/man/gpg-agent.1#DESCRIPTION)を設定しておくと、pinentryは`Assuan protocol`でそのttyをコントロールする設定を行います。
    - 常に設定しておけと言われていますね。
  - `GPG_TTY`はgpg-agentを呼び出す環境のものが使われるため、ラッパースクリプトや、pinentry実装内で`GPT_TTY`の設定を上書きしても意味がありません。
  - 現状stdinをトラップして差し替えるのが最も簡単で堅牢です。
- `PINENTRY_USER_DATA`のフォーマットを`ZELLIJ_POPUP:$(which zellij):${ZELLIJ_SESSION_NAME}`とする
  - [documentに示される通り](https://github.com/gpg/gnupg/blob/gnupg-2.5.8/doc/gpg.texi#L4208-L4211)この環境変数はgpg-agentを経由しても渡されることが保証されています。
  - `ZELLIJ_POPUP`: この値を参照して呼び出すべきpinentry実装をスイッチするようにラッパースクリプトを組みます。
  - `$(which zellij)`: `zellij`が標準的なパスにない時に必要。筆者は何も考えず`cargo install`で入れているので必須です。
  - `${ZELLIJ_SESSION_NAME}`: セッション外から呼びす場合は指定が必要です。
- 最大限、前回のtmux版のソースコードを再利用する
  - ほぼ同じことしかしないです。

## 実装

### 共通部分を切り出す

[前回記事実装](https://zenn.dev/ngicks/articles/pinentry-in-tmux-popup#go%E3%81%A7tmux-popup%E5%86%85%E3%81%A7pinetnry%E3%82%92%E5%91%BC%E3%81%B3%E5%87%BA%E3%81%99%E3%83%A9%E3%83%83%E3%83%91%E3%83%BC%E3%82%92%E8%A8%98%E8%BF%B0%E3%81%99%E3%82%8B%E3%80%82)の流用可能な部分を切り出します。

- fifo(named pipe)2つ(tty, done)生成する。
  - tty: これを通じてfloating windowから`pinentry`呼び出し元に`tty(1)`の結果を送る
  - done: floating window内のコマンドを任意のタイミングまでブロックさせるために使う。
    - `pinentry`成功時などに、呼び出し元が書き込む。
- floating window(popup)表示する
  - pinentry終了までfloating windowを表示し続けるため、done fifoの読み込みでブロックしておく。
- tty取得してfifoに送る
- stdinをトラップして`OPTION ttyname=`を差し替える
- pinentry終了後、done fifoに書き込みを行って終了させる。

までは共通です。

floating windowを表示するコマンドがそれぞれ別のものになります。

```go: popup.go
package popup

import (
    "bufio"
    "context"
    "errors"
    "fmt"
    "log/slog"
    "os"
    "os/exec"
    "path/filepath"
    "strings"
    "sync"
    "syscall"
    "time"
)

func CallPinentry(
    ctx context.Context,
    logger *slog.Logger,
    tempdir string,
    buildPopUpCmd func(ttyFifo, doneFifo string) (cmd string, args []string),
    validateTtyStr func(string) (string, error),
    pinentryPath string,
    pinentryArgs []string,
) (err error) {
    ttyFifo := filepath.Join(tempdir, "tty")
    doneFifo := filepath.Join(tempdir, "done")

    for _, s := range []string{ttyFifo, doneFifo} {
        err = syscall.Mknod(s, syscall.S_IFIFO|0o600, 0)
        if err != nil {
            panic(err)
        }
    }

    logger.Debug("tty fifo created")

    popupCmdPath, popupArgs := buildPopUpCmd(ttyFifo, doneFifo)

    // Launch tmux popup in background
    popupCmd := exec.CommandContext(
        ctx,
        popupCmdPath, popupArgs...,
    )
    popupCmd.Cancel = func() error {
        return popupCmd.Process.Signal(syscall.SIGTERM)
    }
    go func() {
        <-ctx.Done()
        popupCmd.Process.Kill()
    }()

    logger.Debug("popup starting")
    err = popupCmd.Start()
    if err != nil {
        return fmt.Errorf("popup failed: %w", err)
    }
    defer func() {
        logger.Debug("waiting to done fifo")
        done, err := os.OpenFile(doneFifo, os.O_RDWR, 0)
        if err != nil {
            panic(err)
        }
        defer done.Close()
        done.SetWriteDeadline(time.Now().Add(time.Second))
        done.Write([]byte("done\n"))
    }()

    logger.Debug("opening tty fifo")
    f, err := os.OpenFile(ttyFifo, os.O_RDWR, 0)
    if err != nil {
        return fmt.Errorf("failed to open tty: %w", err)
    }
    defer f.Close()

    f.SetReadDeadline(time.Now().Add(20 * time.Second))

    scanner := bufio.NewScanner(f)

    logger.Debug("waiting tty notification")
    scanner.Scan()
    t := scanner.Text()
    if scanner.Err() != nil {
        return fmt.Errorf("scan failed: %w", scanner.Err())
    }

    targetTty, err := validateTtyStr(t)
    if err != nil {
        return err
    }

    logger.Debug("tmux popup started")

    if targetTty == "" {
        return fmt.Errorf("popup return an empty tty")
    }

    logger.Debug("got TTY from popup")

    // Run pinentry-curses with stdin interception
    cmd := exec.CommandContext(ctx, pinentryPath, pinentryArgs...)
    cmd.Cancel = func() error {
        return cmd.Process.Signal(syscall.SIGTERM)
    }

    p, err := cmd.StdinPipe()
    if err != nil {
        return err
    }

    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    err = cmd.Start()
    if err != nil {
        return fmt.Errorf("%s failed to start: %w", pinentryPath, err)
    }

    logger.Debug("pinentry-curses started")

    // Intercept stdin and replace ttyname
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
        return fmt.Errorf("pinentry-curses failed: %w", err)
    }

    return nil
}
```

### zellij run --floating部分

`zellij run --floating`呼び出し部分です。

main packageです。mainはエントリポイントの呼び出し以上のことやらないようにしないと単体テストが書きにくいからやらないほうがいいですが、そもそも単体テストとか書いてないのでこんなもんでいいです。

- `tmux popup`は何も指定しないと`$(SHELL) -c`でコマンドを実行する挙動が[ソース上](https://github.com/tmux/tmux/blob/3.5a/job.c#L169-L177)見られ、`&&`やパイプがあるコマンドもなんとなくいい感じに動きますが、
- `zellij run`では見たところうまく動かなったので、shellで実行するようにします。
  - ソースは軽く追っていますがまだどこがpane実行でコマンド動かす部分なのか読めてないです。
  - 基本は`$SHELL`で指定されたものを使います。
  - docker containerでENTRYPOINTをシェルにした場合などは、`$SHELL`が設定されていないことがあるため、その場合のために`bash`にフォールバックする挙動を加えておきます。
- `$PINENTRY_USER_DATA`を解析して`zellij`のパス、zellijのセッション名を取得します。
- [#4031](https://github.com/zellij-org/zellij/issues/4031)より、`zellij run`で環境変数を渡す方法がないためfifoは渡すコマンドで指定します。
  - こうすると`ps e`でほかのユーザーからもfifoのパスなどが観測できますが、
  - `os.MkdirTemp`で作成されたディレクトりはパーミッションが`0x600`であるので基本的には作成したユーザーにしか操作しえないので大丈夫でしょう。

```go: main.go
package main

import (
    "cmp"
    "context"
    "fmt"
    "log/slog"
    "os"
    "os/signal"
    "path/filepath"
    "strings"
    "syscall"
    "time"

    "github.com/ngicks/run-in-tmux-popup/internal/popup"
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

    shellName := cmp.Or(os.Getenv("SHELL"), "bash")
    zellijPath, sessionName, _ := strings.Cut(
        strings.TrimPrefix(
            strings.TrimSpace(
                os.Getenv("PINENTRY_USER_DATA")),
            "ZELLIJ_POPUP:",
        ),
        ":",
    )
    if len(zellijPath) == 0 || len(sessionName) == 0 {
        panic(
            fmt.Errorf(
                "enviroment variable \"PINENTRY_USER_DATA\" must be"+
                    " formated as \"ZELLIJ_POPUP:zellij_path:session_name\" but is %q",
                os.Getenv("PINENTRY_USER_DATA"),
            ),
        )
    }

    err = popup.CallPinentry(
        ctx,
        logger,
        tempdir,
        func(ttyFifo, doneFifo string) (cmd string, args []string) {
            return zellijPath, []string{
                "--session=" + sessionName,
                "run",
                "--name=pinentry-curses",
                "--floating",
                "--close-on-exit",
                "--pinned=true",
                "--",
                shellName,
                "-c",
                fmt.Sprintf("echo $(tty) >> %s && read done < %s", ttyFifo, doneFifo),
            }
        },
        func(t string) (string, error) {
            return strings.TrimSpace(t), nil
        },
        "/usr/bin/pinentry-curses",
        os.Args[1:],
    )
    if err != nil {
        panic(err)
    }
}

```

## ビルドして使えるようにする

### ビルドして配置

`~/.local/bin`に配置します。
`GOBIN`を設定して`go install`します。

```
GOBIN=$HOME/.local/bin go install github.com/ngicks/run-in-tmux-popup/cmd/zellij-popup-pinentry-curses@7bcf6df8dfbf57ae46e08876e79468ef19d6006d
```

### ラッパースクリプトを書く

`PINENTRY_USER_DATA`に基づいてこのバイナリを呼び出すようにラッパースクリプトを書きます。

```bash
#!/bin/bash

set -Ceu

case "${PINENTRY_USER_DATA-}" in
*TTY*)
  exec pinentry-curses "$@"
  ;;
*TMUX_POPUP*)
  exec $HOME/.local/bin/tmux-popup-pinentry-curses "$@"
  ;;
*ZELLIJ_POPUP*)
  exec $HOME/.local/bin/zellij-popup-pinentry-curses "$@"
  ;;
esac

exec pinentry-qt "$@"
```

### gpg-agent.confでラッパースクリプトを指定

私はこのスクリプトをdotfilesに突っ込んでおいてあるのでそのパスを指定しておきます。

```conf: ~/.gnupg/gpg-agent.conf
pinentry-program /home/ngicks/.dotfiles/scripts/pinentry.sh
```

## 完成！

敗ということで、gpg-agent経由で呼びだれるとこういう見た目になります。

![](/images/pinentry-in-zellij-floating-window/popup.png)

例によって余白がいっぱいありますが筆者は気にしません。

## Caveats

- 既に開いている非表示状態のfloating windowがあるとそれも表示されてしまう。

## おわりに

試しにclaude codeに`How can I use zellij floating windows as pinentry front end?`と聞いてもここまでのものは出てこなかったのでやって意味なかったわけでもないかなと思います。
