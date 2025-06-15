---
title: "[Go]*os.Rootベースのファイルシステム抽象化とライブラリ非依存ヘルパーの提案"
emoji: "📂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## \*os.Rootベースのファイルシステム抽象化とライブラリ非依存ヘルパーの提案

こんにちは。

今回は[Go 1.24]で追加され、[Go 1.25]で残りのメソッドが実装されることになる[*os.Root]を意識したfilesystem abstraction libraryと、逆にfilesystem abstraction libraryどれに対しても使えるようなヘルパーの作り方の提案を行います。

## Overview

- `Go`には[Go 1.16]で追加された[fs.FS]があります。
  - これはread-only filesystemで、`/`で区切られたパスによってファイルを開いて読めるだけ・・・というものです。
  - 当然write interfaceのあるものも実装しようというproposalは上がりましたが、[プラットフォーム間の挙動を埋めるためのコードを書き、それをstdに取り込むことはできるがそうする強い動機は見つからない](https://github.com/golang/go/issues/45757#issuecomment-1675157698)ということでcloseされています。
    - この頃はRuss CoxがGoのTechnical Leaderでした。退任からそんなに経ってないですが少し懐かしく感じますね。
- 有名なwritableなfilesystem abstraction libraryはいくつかあります。筆者が知ってる限りの例を挙げると
  - [github.com/spf13/afero]
  - [github.com/go-git/go-billy]
  - [github.com/hack-pad/hackpadfs]
- これらにはそれぞれちょっとつらいところがあります。
  - [afero]:
    - symlink周りが[SymlinkIfPossible(oldname, newname string) error](https://github.com/spf13/afero/blob/v1.14.0/symlink.go#L33-L39)というinterfaceに分かれている
    - [OsFsが単なるos.Createなどへのショートハンドでしかない](https://github.com/spf13/afero/blob/v1.14.0/os.go#L36-L44)ので[BasePathFs](https://github.com/spf13/afero/blob/v1.14.0/basepath.go#L17-L28)との組み合わせが前提
    - [MemMapFs](https://github.com/spf13/afero/blob/v1.14.0/memmap.go#L32-L36)がpathの取り扱いが少し間違っていて`/`で開いてしまうと`fs.Walk`を組み合わせたとき`.`をrootとするとパスが見つけられなかったり、[ReaderAtのconcurrentに呼び出されてもよいというconstraintが守られていなかった](https://github.com/spf13/afero/blob/v1.14.0/mem/file.go#L226-L232)りする
    - 更新が元気ではない
  - [go-billy]:
    - `go-git`プロジェクトの一環なのでずっと元気に更新し続けてるのがすごいいいところです。
    - [interfaceがcomposableになるように細かく分けられています](https://github.com/go-git/go-billy/blob/v5.6.2/fs.go)
    - 標準とするのはややとがりすぎな[TempFile](https://github.com/go-git/go-billy/blob/v5.6.2/fs.go#L91-L101), [Chroot](https://github.com/go-git/go-billy/blob/v5.6.2/fs.go#L151-L160), [LockとUnlock](https://github.com/go-git/go-billy/blob/v5.6.2/fs.go#L173-L177)とかがinterfaceに込められてて実装するのがつらい。
    - [osfsのOpenFile系が勝手に親フォルダを作成してしまう](https://github.com/go-git/go-billy/blob/v5.6.2/osfs/os.go#L106-L121)のが非常にネック
    - major versionが多すぎる！現在`v6`のpreleaseがいくつか出ています。issue内でもそれはv6で入れないと(破壊的変更になる)みたいな言及があります。
      - [Go v1のリリース日は2012-03-28](https://go.dev/doc/devel/release#go1), [Go1.16のリリースが2021-02-16](https://go.dev/doc/devel/release#go1.16)であることを踏まえると多すぎる。
  - [hackpadfs]:
    - 筆者はお試し以上に使ったことがないため特に深いことは言えないですが、
    - `go-billy`同様に[interfaceがcomposableになるように細かく分けられています](https://github.com/hack-pad/hackpadfs/blob/v0.2.4/fs.go)
    - `go-billy`以上に[fs.FS]に寄せてあって、[ベースとなるFSはfs.FSです](https://github.com/hack-pad/hackpadfs/blob/v0.2.4/fs.go#L11-L13)
      - つまりwrite operationは常に[fs.File]を`type-assertion`で書き込み可能なinterfaceに「広げる」必要があります。
- ところで`Go`は[docker], [podman]など、コンテナ基盤で盛んに使われています。
- [#20126](https://github.com/golang/go/issues/20126)でかつてsecure-joinというpath traversalを防ぎながらjoinを行うAPIの追加がproposeされましたが、ちゃんとできるのかわかんないからという理由でcloseされています。
  - [dependencies](https://pkg.go.dev/github.com/cyphar/filepath-securejoin?tab=importedby)を見れば`k8s.io`の各種パッケージからインポートされているのがわかります。
- [#67002](https://github.com/golang/go/issues/67002)で[*os.Root]が提案され、[Go 1.24]で一部のメソッド、[Go 1.25]で残りすべてのメソッドが追加されます。
  - [*os.Root]は`secure-join`の別解ともいえるもので、特定のsub directory以下にのみアクセスできる`os`のinterfaceを提供します。
- [*os.Root]には[masterで確認すれば](https://pkg.go.dev/os@master#Root)わかりますが`Truncate`を除いたsymlinkやhardlink作成機能が追加されています。
- てことはこの[*os.Root]の持ってるcapabilityはベースと考えてもいいのでは・・・？
- そういうわけで、[*os.Root]の持っているmethod setを基本としたfilesystem abstraction libraryを作ってみます。
- できました: https://github.com/ngicks/go-fsys-helper/tree/main/vroot
- 逆に`type OpenFileFs\[File any\] interface { 	OpenFile(name string, flag int, perm fs.FileMode) (File, error) }`みたいに具体的なファイルの型をtype paramにすればライブラリに縛られないヘルパーが定義できますね
- つくり中です: https://github.com/ngicks/go-fsys-helper/tree/main/fsutil

`Go1.24`で

https://github.com/golang/go/issues/67002

すでに`go1.25rc1`がリリースされています。

https://x.com/golang/status/1932876844849594525

Release Noteは以下です。

https://tip.golang.org/doc/go1.25

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
[Dockerfile]: https://docs.docker.com/build/concepts/dockerfile/
[podman]: https://podman.io/
[Elasticsearch]: https://www.elastic.co/docs/solutions/search

<!-- Go versions -->

[Go]: https://go.dev/
[Go 1.11]: https://go.dev/doc/go1.11
[Go 1.14]: https://go.dev/doc/go1.14
[Go 1.16]: https://go.dev/doc/go1.16
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

[panic]: https://pkg.go.dev/builtin@go1.24.4#panic
[errors.New]: https://pkg.go.dev/errors@go1.24.4#New
[errors.Is]: https://pkg.go.dev/errors@go1.24.4#Is
[errors.As]: https://pkg.go.dev/errors@go1.24.4#As
[errors.Join]: https://pkg.go.dev/errors@go1.24.4#Join
[fmt.Errorf]: https://pkg.go.dev/fmt@go1.24.4#Errorf
[fs.ErrNotExist]: https://pkg.go.dev/io/fs@go1.24.4#ErrNotExist
[http.Server]: https://pkg.go.dev/net/http@go1.24.4#Server
[*http.Server]: https://pkg.go.dev/net/http@go1.24.4#Server
[io.EOF]: https://pkg.go.dev/io@go1.24.4#EOF
[io.Reader]: https://pkg.go.dev/io@go1.24.4#Reader
[io.Writer]: https://pkg.go.dev/io@go1.24.4#Writer
[fs.FS]: https://pkg.go.dev/io/fs@go1.24.4#FS
[fs.File]: https://pkg.go.dev/io/fs@go1.24.4#File
[*os.Root]: https://pkg.go.dev/os@go1.24.4#Root
[syscall.Errno]: https://pkg.go.dev/syscall@go1.24.4#Errno
[text/template]: https://pkg.go.dev/text/template@go1.24.4

<!-- references to Go library -->

[github.com/spf13/afero]: https://github.com/spf13/afero/
[afero]: https://github.com/spf13/afero/
[github.com/go-git/go-billy]: https://github.com/go-git/go-billy
[go-billy]: https://github.com/go-git/go-billy
[github.com/hack-pad/hackpadfs]: https://github.com/hack-pad/hackpadfs
[hackpadfs]: https://github.com/hack-pad/hackpadfs
