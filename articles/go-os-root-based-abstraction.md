---
title: "[Go]*os.Rootベースのファイルシステム抽象化とライブラリ非依存ヘルパーの提案"
emoji: "📂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## \*os.Rootベースのファイルシステム抽象化とライブラリ非依存ヘルパーの提案

こんにちは。

`go1.25rc1`がリリースされましたね。

https://x.com/golang/status/1932876844849594525

DRAFT RELEASE NOTEは以下となります。

https://tip.golang.org/doc/go1.25

今回は[Go 1.24]で追加され、[Go 1.25]で残りのメソッドが実装されることになる[*os.Root]を基盤としたfilesystem abstraction libraryと、どのライブラリでも使用可能な汎用ヘルパー関数の設計について提案します。

## 概要

この記事では、Go 1.25で完全実装される[*os.Root]を基盤とした新しいfilesystem abstraction libraryの提案と、既存ライブラリの課題を解決するアプローチについて説明します。

### 背景

Goには[Go 1.16]で追加された[fs.FS]がありますが、これは読み込み専用のファイルシステムインターフェースです。書き込み機能を持つファイルシステム抽象化の標準化は、[プラットフォーム間の挙動の違いを統一する複雑さに対して、標準ライブラリに含める十分な動機が見つからない](https://github.com/golang/go/issues/45757#issuecomment-1675157698)という理由で見送られています。

そのため、コミュニティでは独自の書き込み可能ファイルシステム抽象化ライブラリが数多く開発されています：

- **[afero]** - 最も広く使われている（imported by: 7,666）
- **[go-billy]** - go-gitプロジェクトの一部として活発に開発
- **[hackpadfs]** - fs.FSをベースとした比較的新しいライブラリ

### セキュリティの課題

Goは[Docker]や[Podman]などのコンテナ基盤で広く使用されており、ファイルシステム操作のセキュリティは重要な課題です。特に、path traversal攻撃（`../../../etc/passwd`のような相対パスを使った不正なファイルアクセス）を防ぐ仕組みが必要とされています。

過去に[secure-joinというAPI](https://github.com/golang/go/issues/20126)の提案がありましたが、実装の複雑さから採用されませんでした。しかし、[github.com/cyphar/filepath-securejoin](https://pkg.go.dev/github.com/cyphar/filepath-securejoin?tab=importedby)が`k8s.io`の各種パッケージで使用されていることからも、この機能への需要は確実に存在します。

### \*os.Rootの登場と意義

[Go 1.24]で一部、[Go 1.25]で完全実装される[*os.Root]は、`secure-join`と似たような課題に対して取り組むために提案されました。

[*os.Root]の特徴：

- 特定のサブディレクトリ以下のみにアクセスを制限
- path traversal攻撃とsymlink escapeの両方を防止

### 本記事の提案

この[*os.Root]の登場により、既存のfilesystem abstraction libraryが直面する互換性や統一性の課題を解決する新しいアプローチが可能になります：

1. **vroot**: [*os.Root]のメソッドセットを基盤としたfilesystem abstraction library

   - `os`パッケージで定義されるファイル操作のすべてを網羅
   - 新たなAPIの基調
     - rootおよびsub-rootからsymlink escapeさせない
     - 絶対パスを受け付けない

2. **fsutil**: Genericsを活用した、filesystem-abstraction-library-agnostic helpers
   - ヘルパーが特定のfilesystem abstraction libraryにくっつかないようにgenericsでもとから剥がしとこうよという提案

あとの内容は目次を見てください。

## 環境

```
$ go version
go version go1.25rc1 linux/amd64
```

`GOTOOLCHAIN`環境変数を設定すれば問答無用で`go1.25rc1`の`sdk`を落としてそれで実行できるようになります。

```
export GOTOOLCHAIN=go1.25rc1
```

## issue

先に述べておきますが、[*os.Root]にはまだバグがあります。

- [#73868](https://github.com/golang/go/issues/73868)
  - `OpenRoot` -> `*os.Root.OpenRoot` -> `*os.Root.OpenRoot`で開いた子root、孫rootで開いたファイルに対して`Readdir`系のメソッドを呼び出すと`ENOENT`が返ってくるというもの。
  - 書いてありますが`*os.Root.OpenRoot`が新しい`*os.Root`を開くときにnameを適切に渡しそこなっているのが原因です。
  - `Go`はstdを編集してコンパイルしなおすと普通に変更が反映されるので`sdk`を直接修正すれば次回以降の`go test`などはうまく動作するようになります。
  - 実はReaddirによって[`[]fs.FileInfo`を取得される際には`os.Lstat`が用いられます](https://github.com/golang/go/blob/go1.25rc1/src/os/dir_unix.go#L151)。
  - `lstat`が呼ばれるときのprefixがちゃんとわたっていないことが問題です。`lstatat`が存在していればこんなバグも起こらなかったんでしょうが、どうもPOSIX APIには存在しないようです。
  - これを機に`Readdir`も[fstatat(3p)](https://man7.org/linux/man-pages/man3/fstatat.3p.html)を使おうみたいな話の流れになるんですかね？
  - と思ってstdを読み直すと[`os.Lstat`はすでにfstatatを使用しています](https://github.com/golang/go/blob/go1.25rc1/src/syscall/syscall_linux_amd64.go#L68-L70)のでもしかしたら使えない理由があってやっていないのかも・・・
  - windowsでは起きません。
- [#69509](https://github.com/golang/go/issues/69509)
  - `wasip1`でパスの取り扱いがおかしいというもの。

以下はバグではなくenhancementですがこれは[#67002](https://github.com/golang/go/issues/67002)の中で述べられていた、各プラットフォーム向けの最適なAPIを使用することで最適な実装を行おうというものです。

- [#73076](https://github.com/golang/go/issues/73076)
  - 各プラットフォーム向けに最適な実装をしようというもの
  - 多分、ファイルに対するIO操作のほうがよほど時間がかかるのでこの最適化がされなくても十分な実行速度を持てると思いますが、std libraryはあらゆるものから使われるわけですからメンテナンス性を確保できている限り、速ければ速いほどいいですよね。

とりあえず`rc2`を待ちましょう。

## はじめに

`Go`には[Go 1.16]で追加された[fs.FS]があります。

これはread-only filesystemで、`/`で区切られたパスによってファイルを開いて読めるだけ・・・というものです。

```go
type FS interface {
    Open(name string) (File, error)
}

type File interface {
    Read(p []byte) (n int, err error)
    Stat() (FileInfo, error)
    Close() error
}
```

特定のディレクトリの下に特定の構造があり、それを期待して読み込んだり書き込んだりするようなプログラムを書くことは、筆者としてはたびたびあります。
ディレクトリ自体は設定ファイルなりなんなりで自由に変えることができるため、どのディレクトリに読み込んでいるのかはプログラムの関心から外したいという欲求が筆者にはよくありますし、実際に[fs.FS]が実装されたのはそういった欲求は広く存在するからだと思います。

[fs.FS]は単にinterfaceであるため、それさえ満たせばdata sourceは何でもよいことになります。
当然、どこかのディレクトリ以下でもいいし、`smb`/`nfs`などのネットワークファイルシステム, `tar`/`zip`などのアーカイブファイル、なんならin-memoryの構造でもかまいません。

同様に書き込みに関しても似たように、書き込み先は何でもよいということがあります。

[fs.FS]はread-onlyですので、書き込みは行えません。
[fs.FS]に書き込めるinterfaceも実装しようというproposalは上がりましたが、[プラットフォーム間の挙動を埋めるためのコードを書き、それをstdに取り込むことはできるがそうする強い動機は見つからない](https://github.com/golang/go/issues/45757#issuecomment-1675157698)ということでcloseされています。コミュニティーの中でいろいろな形が模索されたのち、数年後にまた検討しようとのことです。

stdには取り込まれませんが、コミュニティーの中でいくつもwritableなfilesystem abstraction libaryが開発されています。
筆者が知ってる限りの例で有名なものを挙げると

- [github.com/spf13/afero]
- [github.com/go-git/go-billy]
- [github.com/hack-pad/hackpadfs]

この中では[afero]が一番有名で現在imported by: 7,666で最多となります。これはあくまでgo proxyに記録されている[afero]をimportしているgo moduleの数なので実際にはもっとたくさんのgo moduleが利用していると思われます。

筆者は[afero]を使用しており、大変便利ですが、それぞれに若干のつらさがあります。

## 既存ライブラリの辛さ

それぞれのライブラリにはそれぞれつらみがあります。

- [afero]:
  - symlinkの取り扱いが[SymlinkIfPossible(oldname, newname string) error](https://github.com/spf13/afero/blob/v1.14.0/symlink.go#L33-L39)というinterfaceになっている
  - hardlinkのサポートがない
  - [OsFs](https://github.com/spf13/afero/blob/v1.14.0/os.go#L36-L44)が単なる`os.Create`などへのショートハンドでしかないため、[BasePathFs](https://github.com/spf13/afero/blob/v1.14.0/basepath.go#L17-L28)との組み合わせが前提となっている。
    - これがあるため絶対パスを禁止することができなくなっています。
  - [MemMapFs](https://github.com/spf13/afero/blob/v1.14.0/memmap.go#L32-L36)の不備
    - pathのnormalizeに漏れがあり、`/path/to/file`と`./path/to/file`で扱いが別になってしまうため、`fs.Walk`でwalkするとき、rootを`"."`とするとパスが見つからないことがある。
      - このせいでテストでだいぶ驚くことになる
    - fileの`ReadAt`メソッドが[ReaderAtはconcurrentに呼び出されてもよいというconstraintを守っていない](https://github.com/spf13/afero/blob/v1.14.0/mem/file.go#L226-L232)
  - 実装が全般的に`/path/to/file`を受け付けてしまいます。絶対パス風ならはじいてほしいと思っています。
  - あまり活動が活発ではない。
- [go-billy]:
  - `go-git`プロジェクトの一環として開発されており、大変元気です。
  - [interfaceがcomposableになるように細かく分けられている](https://github.com/go-git/go-billy/blob/v5.6.2/fs.go)
  - `Filesytem` interfaceには[TempFile](https://github.com/go-git/go-billy/blob/v5.6.2/fs.go#L91-L101)というやや専門的すぎなメソッドが含まれていたり、
  - `File`には[Lock/Unlock](https://github.com/go-git/go-billy/blob/v5.6.2/fs.go#L173-L177)が含まれています。
    - `File`はcomposableになっていないため、この専門的なメソッドの実装は必須です
  - [osfsのOpenFile系が勝手に親フォルダを作成してしまう](https://github.com/go-git/go-billy/blob/v5.6.2/osfs/os.go#L106-L121)挙動はかなり意見が強いです。
  - major versionが多すぎる
    - `v5`が最新で、`v6`のリリースを示唆する書き込みもissue中にあります([next major versionへの言及](https://github.com/go-git/go-billy/issues/101))
    - [Go v1のリリース日は2012-03-28](https://go.dev/doc/devel/release#go1)、[Go1.16のリリースが2021-02-16](https://go.dev/doc/devel/release#go1.16)であることを考えると、多すぎる。いくらたいていはinterface的な互換があるとはいえ多すぎるmajor versionの更新は相互運用性にかかわってきます。
- [hackpadfs]:
  - 筆者はお試し以上に使ったことがないため特に深いことは言えないですが、
  - `go-billy`同様に[interfaceがcomposableになるように細かく分けられています](https://github.com/hack-pad/hackpadfs/blob/v0.2.4/fs.go)
  - `go-billy`以上に[fs.FS]に寄せてあって、[ベースとなるFSはfs.FSです](https://github.com/hack-pad/hackpadfs/blob/v0.2.4/fs.go#L11-L13)
    - つまりwrite operationは常に[fs.File]を`type-assertion`で書き込み可能なinterfaceに「広げる」必要があります。
  - `memfs`が`key-value store`ベースの実装になっており、こうなってしまうと効率的にsubfsへの分割ができなくなってしまいます。

## 根本的辛さ: interfaceとしてのかみ合わなさ

ただし、これらのライブラリは基本的にinterfaceを定めるものなので、最終的に実装に文句があるなら自前でそろえてしまえばいいということになります。
なので、根本的に回避不能な辛さはinterfaceがいいか悪いかのみで判断すべきになります。

- [afero] -> symlink周りの取り扱いが遠回り
- [go-billy] -> major versionの多さによる不安定さ、`File`に存在する`Lock`/`Unlock`メソッド
- [hackpadfs] -> [fs.File]をwritableになるようにtype-assertしなければならない

が、根本的に回避できない辛さとなります。

## \*os.Rootの登場

### secure-join

若干余談ですがコンテクストとして`secure-join`の存在を知っていたほうが[*os.Root]の立ち位置が明らかになるかも知れないので触れておきます。

`Go`は[Docker], [podman]など、コンテナ基盤で盛んに使われています。
コンテナは実装によりますが、基本的には[pivot_root(2)](https://man7.org/linux/man-pages/man2/pivot_root.2.html)、[unsahre(2)](https://man7.org/linux/man-pages/man2/unshare.2.html)その他もろもろで隔離された名前空間のなかで動作するプロセスやらroot fsやらのことをさします。

[#20126](https://github.com/golang/go/issues/20126)でかつてsecure-joinというpath traversalを防ぎながらjoinを行うAPIの追加がproposeされましたが、完全な実装の難しさからcloseされています。しかし、[github.com/cyphar/filepath-securejoin](https://github.com/cyphar/filepath-securejoin)の[dependencies](https://pkg.go.dev/github.com/cyphar/filepath-securejoin?tab=importedby)を見れば、`k8s.io`の各種パッケージからインポートされていることがわかります。こちらは`/proc`の下などをコンテナの名前空間を見せたりするのに使う安全策を組み込んでいるような記述があります。

[*os.Root]はこれとは違って`/proc`でのカーネル空間で起きるsymlink resolutionなどは考慮に加えません。

### \*os.Root

[#67002](https://github.com/golang/go/issues/67002)で[*os.Root]が提案され、[Go 1.24]で一部のメソッド、[Go 1.25]で残りすべてのメソッドが追加されます。

[*os.Root]は特定のサブディレクトリの下のみを操作できる`os`のメソッドを提供するものです。
path traversalに加えて、symlinkによって特定のサブディレクトリの外に出るのを防ぐことができます。

### \*os.Rootのmethod set

[masterで確認すれば](https://pkg.go.dev/os@master#Root)わかりますが、[*os.Root]には以下のようなメソッドが追加されています：

```go
type Root struct {
    // unexported fields
}

func (r *os.Root) Chmod(name string, mode os.FileMode) error
func (r *os.Root) Chown(name string, uid int, gid int) error
func (r *os.Root) Chtimes(name string, atime time.Time, mtime time.Time) error
func (r *os.Root) Close() error
func (r *os.Root) Create(name string) (*os.File, error)
func (r *os.Root) FS() fs.FS
func (r *os.Root) Lchown(name string, uid int, gid int) error
func (r *os.Root) Link(oldname string, newname string) error
func (r *os.Root) Lstat(name string) (os.FileInfo, error)
func (r *os.Root) Mkdir(name string, perm os.FileMode) error
func (r *os.Root) MkdirAll(name string, perm os.FileMode) error
func (r *os.Root) Name() string
func (r *os.Root) Open(name string) (*os.File, error)
func (r *os.Root) OpenFile(name string, flag int, perm os.FileMode) (*os.File, error)
func (r *os.Root) OpenRoot(name string) (*os.Root, error)
func (r *os.Root) ReadFile(name string) ([]byte, error)
func (r *os.Root) Readlink(name string) (string, error)
func (r *os.Root) Remove(name string) error
func (r *os.Root) RemoveAll(name string) error
func (r *os.Root) Rename(oldname string, newname string) error
func (r *os.Root) Stat(name string) (os.FileInfo, error)
func (r *os.Root) Symlink(oldname string, newname string) error
func (r *os.Root) WriteFile(name string, data []byte, perm os.FileMode) error
```

`Truncate`を除いたsymlinkやhardlink作成機能も含まれており、ファイルシステム操作に必要なすべての機能が揃っています。

### \*os.Rootの仕組み

[*os.Root]は[openat(2)](https://man7.org/linux/man-pages/man2/openat.2.html)などの、`fd`からの相対パス開きができるAPIに依存しています。
[*os.Root]の各methodにパスが渡されるとパスセパレータ(`/`か`\`)でパスコンポーネントに分割し、`OBJ_DONT_REPARSE`(windows)/`O_NOFOLLOW`(unix)付きで`NtCreateFile`/`openat`を呼び出し、ディレクトリを1つずつ開いていきます。
symlinkが見つかった場合には`readlinkat`を使って読み取りますが、この場合には読み込まれたリンクでパスコンポーネントを置き換え(`a/b/c`で`b -> ../d`だった場合`a/../d/c`で)、rootからパスをたどりなおします。これは`openat(dirFd, "..")`をしてしまうと、`dirFd`が開いているファイルが`rename`などで移動された際のTOCTOU(Time Of Check, Time Of Use) raceによって間違ったパスをたどってしまうため、そうならないようにするための対策のようです。

## vroot: \*os.Root-based filesystem abstraction

[*os.Root]が標準を示したことでfilesystem abstraction libraryの持つべきベーシックなinterfaceが定まりました。
・・・っていっても`os`パッケージ内での基本的なファイル操作APIは[Go 1]から特に追加も変更もなかったためずっと前から定まっていたんですが、
特定のサブディレクトリから脱出しないとか、絶対パスは使わせないというAPI constraintのベースラインがさらに追加されました。

[*os.Root]がstdに入っちゃったらこれとうまくやれないfilesystem abstraction libraryはつらい思いをするのは目に見えています。
現状[afero]は`/`から始まるパスでも動作してしまうためこのsubtleな違いが実装を入れ替えたときに微妙なエラーを引き起こすことが考えられます。(そもそも前述通り筆者は`afero`の`MemMapFs`のsubtletiesでテストが動かなかったことがあるわけですが)

どうせなら作ってしまえということで、[*os.Root]を中心にとらえたfilesystem abstraction libraryを作ってみます。

https://github.com/ngicks/go-fsys-helper/tree/main/vroot

まだめちゃくちゃWIPですがここにホストしてあります。

- major versionは基本的に上がることはないはず:
  - 前述通り、[Go 1]から特にファイル操作APIは増えたり変わったりしていないため、このinterfaceは安定しているとみなすことができます。
- intefaceのcomposabilityは一切捨てます。
  - ファイルシステムは書き込み先の事情でいきなりいろいろ変わるのでinterface上のmethodのある/なしで何かを判断し分ける必要はそもそもないと思っています。
    - 例えば、残り容量の足りなくなってきたfilesystemがremountされてread-onlyに突然なったりです。
    - `sftp`, `nfs`, `smb`などのネットワークストレージは相手サーバーの設定変更でできることが変わってきます。
  - もしかしたら`Capability` extension interfaceを通じてcapabilityのチェックができるようにするかもしれませんが現状では何も考えていません。
    - これは[statvfs(3)](https://man7.org/linux/man-pages/man3/statvfs.3.html)によるmount flagのチェックと対応するためそこまでおかしく感じないんじゃないかと思います。

### Fs

[*os.Root]のmethod setを直訳してinterfaceを作ります。

```go
// Fs represents capablities [*os.Root] has as an interface.
//
// Methods are encouraged to return [*os.LinkError] wrapping an appropriate error for Rename, Link and Symlink,
// [*fs.PathError] for others.
type Fs interface {
    Chmod(name string, mode fs.FileMode) error
    Chown(name string, uid int, gid int) error
    Chtimes(name string, atime time.Time, mtime time.Time) error
    // Close closes Fs.
    // Callers should not use Fs after return of this method but
    // it is still possible that the method is just a no-op.
    Close() error
    Create(name string) (File, error)
    Lchown(name string, uid int, gid int) error
    Link(oldname string, newname string) error
    Lstat(name string) (fs.FileInfo, error)
    Mkdir(name string, perm fs.FileMode) error
    MkdirAll(name string, perm fs.FileMode) error
    // Name returns name for the Fs.
    // For osfs, it reutnrs the name of the directory presented to OpenRoot.
    Name() string
    Open(name string) (File, error)
    OpenFile(name string, flag int, perm fs.FileMode) (File, error)
    OpenRoot(name string) (Rooted, error)
    ReadLink(name string) (string, error)
    Remove(name string) error
    RemoveAll(name string) error
    Rename(oldname string, newname string) error
    Stat(name string) (fs.FileInfo, error)
    Symlink(oldname string, newname string) error
}
```

- 1点だけ[*os.Root]と違うところ: `Readlink`ではなく`ReadLink`と名前が変えてあります。
  - これは`Go 1.25`で追加される`fs.ReadLinkFS`のinterfaceと合わせるためにこうなっています。

### File

[*os.File]を直訳して`File` interfaceを定義します。

```go
// File is basically same as [*os.File]
// but some system dependent methods are removed.
type File interface {
    // Chdir() error

    Chmod(mode fs.FileMode) error
    Chown(uid int, gid int) error
    Close() error

    // Fd returns internal detail of file handle.
    // Only os-backed File should reutrn this value.
    // Otherwise, return ^(uintptr(0)) to indicate this is invalid value.
    Fd() uintptr

    Name() string
    Read(b []byte) (n int, err error)
    ReadAt(b []byte, off int64) (n int, err error)
    ReadDir(n int) ([]fs.DirEntry, error)

    // File might implement ReaderFrom but is not necessary.
    // ReadFrom(r io.Reader) (n int64, err error)

    Readdir(n int) ([]fs.FileInfo, error)
    Readdirnames(n int) (names []string, err error)
    Seek(offset int64, whence int) (ret int64, err error)

    // SetDeadline(t time.Time) error
    // SetReadDeadline(t time.Time) error
    // SetWriteDeadline(t time.Time) error

    Stat() (fs.FileInfo, error)
    Sync() error

    // SyscallConn() (syscall.RawConn, error)

    Truncate(size int64) error
    Write(b []byte) (n int, err error)
    WriteAt(b []byte, off int64) (n int, err error)
    WriteString(s string) (n int, err error)

    // File might implement WriterTo but is not necessary.
    // WriteTo(w io.Writer) (n int64, err error)
}
```

- 実際のファイルとは限らないので`Chdir`は消します。
- `ReadFrom`, `WriteTo`は[io.Copy]向けの最適な実装を提供するextension interfaceなので強制ではなくします。
- `SetDeadline`, `SetReadDeadline`, `SetWriteDeadline`はソケットなど一部ファイル向けなので強制ではなくします。
- `SyscallConn`も同様に消します。
- `Fd`は大抵のケースで不要に思いますが、
  - `file lock`の実装に必要です。
    - `Fd`がvalidな値(=`^(uintptr(0))`以外)のとき、[fcntl(2)](https://man7.org/linux/man-pages/man2/fcntl.2.html)(unix)/[LockFile](https://learn.microsoft.com/ja-jp/windows/win32/api/fileapi/nf-fileapi-lockfile)(windows)などを用いてロックし、そうでないときは`Fs`固有の方法でロックすればよいでしょう。
    - record lockingを`vroot`上で再実装するのは骨が折れそうなので、こういう形で余白を残しつつ放置する作戦です。
  - 後述の`WalkDir`のために必須としてあります。
    - filesystemを*walk*するときはたいてい、bind mountによるループが起きていないかのチェックが必要です。
    - unix系のplatformでは[stat(2)](https://man7.org/linux/man-pages/man2/stat.2.html)などを通じて[struct stat](https://man7.org/linux/man-pages/man3/stat.3type.html)を得ることで、inodeとdev numberの組み合わせでファイル固有の値を得ることができますが、
    - windowsプラットフォームでは[GetFileInformationByHandle](https://learn.microsoft.com/ja-jp/windows/win32/api/fileapi/nf-fileapi-getfileinformationbyhandle)を用います。
      - これには`fd`・・・というか`FileHandle`の値が必要です。

### RootedとUnrooted

`vroot`は二つの中心的interfaceが存在します。

- `Rooted`: [*os.Root]と同じく、path traversalとsymlink escapeを防ぐ
- `Unrooted`: path traversalは防ぐが、symlink escapeは許す。

```go
// Unrooted is like [Rooted] but allow escaping root by sysmlink.
// Path traversals are still not allowed.
type Unrooted interface {
    Fs
    Unrooted()
    OpenUnrooted(name string) (Unrooted, error)
}

// Rooted indicates the implementation is rooted,
// which means escaping root by path traversal or symlink
// is not allowed.
type Rooted interface {
    Fs
    Rooted()
}
```

[*os.Root]との滑らかな相互運用は目指していますが実際にはroot外に向かっているsymlinkを解決したい場面はたくさんあると思います。
思いつく限りだと

- それこそ`/etc/passwd`へのsymlinkが必要な場面
- `/etc/smb.conf`が別のマウントポイントへのsymlinkになっている
- `Volta`(についてくる`npm`)や`pnpm`のようなpackage managerがsymlinkによって依存ファイルを管理している

などなどでしょうか？
そういったケースにおいてsymlinkを解決してroot外へのアクセスをさせたいことは普通にあると思うので`Unrooted`も同時に定義しておきます。

`Unrooted`はTOCTOUにも弱いつくりになっていることが想定されます(これは[afero]の`BasePathFs`と同じです)。
そのため`Unrooted`から`Rooted`を開くことは(もちろん不安全だが)できるようになっていますが、その逆はできません。
前述のとおり、[*os.Root]は`fd`からの相対的なパス操作によってpath escapeを防ぎます。この`fd`が指し示すディレクトリは開いた後に`rename`などによって移動されていることは十分にあり得ます。`fd`からパスへの正確な変換方法は筆者の知り及ぶ限りありませんし、それがTOCTOU raceに強いとも思えません。そのため`Rooted`から`Unrooted`への変換は定義上作ることができない、ということになります。

## 実装

### osfs

とりあえず[*os.Root]から`Rooted`へ変換できるようにします。

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/osfs/rooted.go#L16-L21

`os`パッケージが`errPathEscapes`をエクスポートしないため

https://github.com/golang/go/blob/go1.25rc1/src/os/file.go#L421

文字列を見てエラーを差し替える部分を作っておきます。

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/osfs/rooted.go#L45-L58

文字列比較はやらないでいいならやりたくないですが、こうしないと`errors.Is(err, ErrPathEscapes)`でテストをかけないので仕方なくやっています。

[afero]の`BasePathFs`とほぼ同じものとして`osfs`の`Unrooted`を作ります。

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/osfs/unrooted.go#L19-L26

### WalkDir

`fs.WalkDir`と互換なものとして`vroot.WalkDir`を定義しておきます。

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/walk.go#L12-L25

interfaceがsymlinkの存在をもとから考慮に入れているので`fs.WalkDir`と違ってsymlinkをresolveしてたどってもよいことにしてあります。

`fs.WalkDirFunc`とは違い、`vroot.WalkDirFunc`はsymlinkを解決した後にrealPathも受け取るようになっています。ただしこれは`ReadLink`と`Lstat`を組み合わせてパスをレキシカルに解決するだけのとても単純な仕組みであるため、TOCTOU raceには弱いです。
(現状まったくdoc commentが書けていませんが)root外のrealPathの取得は`Rooted`はもちろん`Unrooted`でもできない(root外のパスに対して`ReadLink`を呼ぶ必要があるため)ので、その場合はrealPathには`""`が渡されることになります。
なので基本的にrealPathに`""`が来てなおかつディレクトリの場合は、`SkipDir`を返してstepするのをやめたほうが良いですね。

また、`WalkDir`がbind mountによるfilesystem loopによって無限ループに陥らないようにするために、可能であればファイルからユニークな値を取り出します。

unix系では`stat`から

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/walk_unix.go#L1-L22

(`plan9`)

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/walk_plan9.go#L1-L22

windowsでは`GetFileInformationByHandle`から

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/walk_windows.go#L1-L37

それぞれユニークな値を取得します。
とりあえず`linux/amd64`, `linux/arm64`, `darwin/amd64`, `darwin/arm64`, `windows/amd64`では見た限り正しく固有な値をとれているようです。
`plan9`や`wasip1`などではとりあえずコンパイルできますが、実装的に正しいのか検証はできていません。

### to/from fs.FS

`fs.FS`と`vroot`の相互変換を定義します。

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/iofs_from.go#L32-L53

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/iofs_from.go#L192-L209

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/iofs_to.go#L18-L31

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/iofs_to.go#L69-L82

### ReadOnly

`Rooted`/`Unrooted`を`read-only`になるようにラップする仕組みも欲しいため作っておきます。
これは間違って書かないようにするための安全策としてあったほうが良いですね

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/readonly.go#L13-L21

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/readonly.go#L113-L121

### overlayfs

これが一番欲しかったものかもしれない。

`overlay filesystem`です。複数の`vroot.Rooted`を重ね合わせて一つのfsに見せかけます。

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/overlayfs/overlay.go#L35-L82

- 複数の`vroot.Rooted`を重ねて一つに見せます。
- ディレクトリの内容は統合され、
- ファイル(ディレクトリ以外)の場合は最も「上側」のレイヤーのものが選ばれます。
- Copy-On-Writeの挙動があります。
  - `chmod`などでファイルのメタデータを変更したときか、
  - write modeでファイルを開くとコピーが起きます。
    - 本当はファイルに対して初めて`Write`が呼ばれたときにコピーが起きるようにしたかったのですが、ロックの取り方やrace conditionの懸念から単純なこの挙動になっています。
    - 実用上それで困らないと思ってます。
- 書き込みはすべて`top layer`にのみ起きます。
- 下層にあるファイルを消えたように見せるために、white out listという形で消えたパスを管理します。
  - これはこのissueを参考にしています: https://github.com/opencontainers/image-spec/issues/24

ユースケースはいくつかあって

- 複数のディレクトリの内容を仮想的に重ねて見せたい
  - ビルド成果物を共有フォルダなどに格納するとき、オペレーションでミスりたくないから`WORM`(Write Once Read Many)にしておいてばらばらのディレクトリに格納しておくが実際には1つのディレクトリのように見せたい。
- code genratorなどの成果物をoverlayに書き出し、top layerのコンテンツを[packages.ConfigのOverlay](https://pkg.go.dev/golang.org/x/tools/go/packages#Config)に渡すことで書き出す前に型チェックをかける。
- cache
  - `copy-on-write`がread-onlyで開いたときにも起きるようにすればcacheとして使用できます。

layerの重ね合わせはsymlinkも考慮に加えます。
あるlayerにあるsymlinkのlink targetは別のlayerをさしていてもよく、あればそちらに向けて解決されます。
つまり、下記exampleのように動作します。

[example](https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/overlayfs/example_symlink_test.go)

```go
package overlayfs_test

import (
    "fmt"
    "io/fs"
    "os"
    "path/filepath"
    "strconv"

    "github.com/ngicks/go-fsys-helper/vroot"
    "github.com/ngicks/go-fsys-helper/vroot/osfs"
    "github.com/ngicks/go-fsys-helper/vroot/overlayfs"
)

func must1(err error) {
    // ...省略...
}

func must2[V any](v V, err error) V {
    // ...省略...
}

func tree(fsys vroot.Fs) error {
    // ...省略...
}

func Example_overlay_symlink() {
    tempDir := must2(os.MkdirTemp("", ""))

    for i := range 4 {
        layer := filepath.Join(tempDir, "layer"+strconv.FormatInt(int64(i), 10))
        must1(os.MkdirAll(filepath.Join(layer, "meta"), fs.ModePerm))
        must1(os.MkdirAll(filepath.Join(layer, "data"), fs.ModePerm))
    }

    // create leyred file system like this.
    //
    //                    +-------+
    // LAYER3:            | link2 |<-----+
    //                    +-------+      |
    //                      |            |
    //         +-------+    |        +-------+
    // LAYER2: | link3 |<---+        | link1 |
    //         +-------+             +-------+
    //             |
    //             |      +------+
    // LAYER1:     +----->| file |
    //                    +------+

    must1(os.MkdirAll(filepath.Join(tempDir, "layer3", "data", filepath.FromSlash("a/b/")), fs.ModePerm))
    must1(os.Symlink("../link3", filepath.Join(tempDir, "layer3", "data", filepath.FromSlash("a/b/link2"))))

    must1(os.MkdirAll(filepath.Join(tempDir, "layer2", "data", filepath.FromSlash("a/b/c")), fs.ModePerm))
    must1(os.Symlink("../link2", filepath.Join(tempDir, "layer2", "data", filepath.FromSlash("a/b/c/link1"))))
    must1(os.Symlink("./b/file", filepath.Join(tempDir, "layer2", "data", filepath.FromSlash("a/link3"))))

    must1(os.MkdirAll(filepath.Join(tempDir, "layer1", "data", filepath.FromSlash("a/b/")), fs.ModePerm))
    must1(os.WriteFile(filepath.Join(tempDir, "layer1", "data", filepath.FromSlash("a/b/file")), []byte("foobar"), fs.ModePerm))

    var closer []func() error
    defer func() {
        for _, c := range closer {
            err := c()
            if err != nil {
                fmt.Printf("meta fsys close error = %v\n", err)
            }
        }
    }()
    composeLayer := func(i int) overlayfs.Layer {
        metaFsys := must2(
            osfs.NewRooted(filepath.Join(tempDir, "layer"+strconv.FormatInt(int64(i), 10), "meta")),
        )
        closer = append(closer, metaFsys.Close)

        meta := overlayfs.NewMetadataStoreSimpleText(metaFsys)
        data := must2(
            osfs.NewRooted(filepath.Join(tempDir, "layer"+strconv.FormatInt(int64(i), 10), "data")),
        )
        return overlayfs.NewLayer(meta, data)
    }

    fsys := overlayfs.New(
        composeLayer(0),
        []overlayfs.Layer{composeLayer(1), composeLayer(2), composeLayer(3)},
        nil,
    )

    must1(tree(vroot.FromIoFsRooted(os.DirFS(tempDir).(fs.ReadLinkFS), tempDir)))

    fmt.Println()

    bin, err := vroot.ReadFile(fsys, filepath.FromSlash("a/b/c/link1"))
    if err != nil {
        fmt.Printf("err = %v\n", err)
    } else {
        fmt.Printf("%q: %s\n", "a/b/c/link1", string(bin))
    }

    // Output:
    // layer1/data/a/b/file
    // layer2/data/a/b/c/link1 -> ../link2
    // layer2/data/a/link3 -> ./b/file
    // layer3/data/a/b/link2 -> ../link3
    //
    // "a/b/c/link1": foobar
}
```

これ実装中にwindowsでは`ERROR_PATH_NOT_FOUND`, `ERROR_FILE_NOT_FOUND`はほぼ同じもののように扱われ、POSIXの`ENOTDIR`のような概念はないらしいということを知りました。やっぱりマルチプラットフォームなライブラリづくりは面倒ですね。

### synthfs(=in-memory fs)

以下の記事で書いたsynthetic filesystemの`vroot`版です。

https://zenn.dev/ngicks/articles/go-virtual-mesh-fs-for-os-copyfs

完全にin-memoryなfilesystem+任意のバッキングストレージという組み合わせで成り立つfilesystemで、
file pathによるtrieを構築し、各ノードには適当なバッキングストレージのファイルを追加できるようにします。

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/synthfs/fs.go#L22-L56

バッキングストレージは、ほかの`vroot.Fs`(`fs.FS`から変換してもよい)、memoryなど、何でもよいです。メモリーからのみallocateするようにすると、これが[afero]でいうところの`MemMapFs`になります。

作った動機は上記の記事内で説明していますが、

- ほかの[fs.FS]の内容から`sha256sum`などをとったメタデータを**データと同じディレクトリに**追加し、(一旦書き出しを経ずに)`CopyFS`とか`AddFS`に直接渡す。
- in-memory fsでパスを何かしらの木構造で持つものが欲しかった:
  - in-memory fsは`map[string]data`をベースに実装されること多いようである。
    - [afero]しかり、[hackpadfs]しかり
  - [afero]の`MemMapFs`でおきたパスの取り扱いが間違っている問題が起こらないようにする。
  - sub-fsysを取得するとき、全体ロックを取得しなくていいようにする。
- 権限のみを差し替えるようなラッパーが欲しかった
  - 例えば[embed.FS]は内容物のpermission bitが必ず`0o444`(ファイル)と`0o555`(ディレクトリ)になります([\[1\]](https://github.com/golang/go/blob/go1.24.4/src/embed/embed.go#L336),[\[2\]](https://github.com/golang/go/blob/go1.24.4/src/embed/embed.go#L389),[\[3\]](https://github.com/golang/go/blob/go1.24.4/src/embed/embed.go#L227-L232))。
  - そのため、`AddFS`がpermissionを広げない場合に狭いファイルが書かれることがあります。

### memfs

in-memory filesystemです。上記の`synthfs`のショートハンドです。

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/vroot/memfs/mem.go#L1-L27

## tarfs

`vroot`とは直接は関係ないですが、以下の記事で作った`tarfs`にsymlinkとhardlinkの取り扱いを加え、さらに、`vroot`と組み合わせるのを意識して「symlink解決時にsub-rootより上の階層に移動するか」の設定を追加しました。

https://zenn.dev/ngicks/articles/go-tar-reader-implement-reader-at

https://github.com/ngicks/go-fsys-helper/tree/2adb17618ef755813afd6fa49910134eb94d3ceb/tarfs

symlinkを無視するのがデフォルト挙動ですが、デフォルトでハンドルしたほうがいい気もするので(破壊的に変更になりますが)そのように変えるかもしれません。
symlinkありのtarでも普通に動いているので使えそうな感じですが、世にどんなエッジケースがあるのかよくわからないのでしばらくタグをつけずに様子見します。

ちなみに`fstest.TestFS`がsymlinkを考慮するのは`go1.25`以降であるようなのでsymlink/hardlinkありのアーカイブを使ったテストは`//go:build go1.25`で`Go 1.25`以降でないと実行されないように制限してあります。リリースされたら`CI`が常に`go1.25`を使うように変更しようかなと。

## fsutil: filesystem-abstraction-library-agnostic helpers

https://github.com/ngicks/go-fsys-helper/tree/main/fsutil

多分どのfilesystem abstraction libraryでも動作するようなヘルパーを書いてるけど、interfaceがそれぞれ違うせいでどれかでしか使えないっていうことがあるともったいないですよね？
ということで、逆の発想としてどのライブラリでも使えるようにヘルパーを作る仕組みを考えてみます。

「考えてみます」といっても特段難しいことはなく、`File`の部分をtype parameterにした`Fs` interfaceを再定義し、これを引数となるようなジェネリック関数を作ればよいだけです。
さらに、必要最低限な機能にのみ依存するようにするために、`Fs`, `File`のinterfaceはmethod単位で分割します。

### interfaceの分割

つまり、`Fs` interfaceを以下のように分割します。

```go

// Fs files

type ChmodFs interface {
    Chmod(name string, mode fs.FileMode) error
}

type ChownFs interface {
    Chown(name string, uid int, gid int) error
}

type OpenFileFs[File any] interface {
    OpenFile(name string, flag int, perm fs.FileMode) (File, error)
}
// ...
```

同様に`File`もmethodごとに切り分けます。

```go
// File interfaces

type ChmodFile interface {
    Chmod(mode fs.FileMode) error
}

type NameFile interface {
    Name() string
}

type ReadAtFile interface {
    ReadAt(b []byte, off int64) (n int, err error)
}

type ReadDirFile interface {
    ReadDir(n int) ([]fs.DirEntry, error)
}

// ...
```

### Example1: OpenFileRandom

関数は必要なinterfaceだけに依存するように前述したinterfaceを適当に組み合わせます。
例えば、`os.CreateTemp`と似たような機能をもつヘルパーを定義するとすると、

```go
type OpenFileFs[File any] interface {
    OpenFile(name string, flag int, perm fs.FileMode) (File, error)
}

var (
    ErrBadPattern = errors.New("bad pattern")
    ErrMaxRetry   = errors.New("max retry")
)

func OpenFileRandom[FS OpenFileFs[File], File any](fsys FS, dir string, pattern string, perm fs.FileMode) (File, error) {
    return openRandom(
        fsys,
        dir,
        pattern,
        perm,
        func(fsys FS, name string, perm fs.FileMode) (File, error) {
            return fsys.OpenFile(filepath.FromSlash(name), os.O_RDWR|os.O_CREATE|os.O_EXCL, perm|0o200) // at least writable
        },
    )
}

func MkdirRandom[FS interface {
    OpenFileFs[File]
    MkdirFs
}, File any](fsys FS, dir string, pattern string, perm fs.FileMode) (File, error) {
    return openRandom(
        fsys,
        dir,
        pattern,
        perm,
        func(fsys FS, name string, perm fs.FileMode) (File, error) {
            err := fsys.Mkdir(name, perm)
            if err != nil {
                return *new(File), err
            }
            return fsys.OpenFile(name, os.O_RDONLY, 0)
        },
    )
}

func openRandom[FS, File any](
    fsys FS,
    dir string,
    pattern string,
    perm fs.FileMode,
    open func(fsys FS, name string, perm fs.FileMode) (File, error),
) (File, error) {
    if dir == "" {
        dir = "." + string(filepath.Separator)
    }

    if strings.Contains(pattern, string(filepath.Separator)) {
        return *new(File), fmt.Errorf("%w: %q contains path separators", ErrBadPattern, pattern)
    }

    var prefix, suffix string
    if i := strings.LastIndex(pattern, "*"); i < 0 {
        prefix = pattern
    } else {
        prefix, suffix = pattern[:i], pattern[i+1:]
    }

    attempt := 0
    for {
        random := randomUint32Padded()
        name := filepath.Join(dir, prefix+random+suffix)
        f, err := open(fsys, name, perm.Perm())
        if err == nil {
            return f, nil
        }
        if errors.Is(err, fs.ErrExist) {
            attempt++
            if attempt < 10000 {
                continue
            } else {
                return *new(File), fmt.Errorf(
                    "%w: opening %s",
                    ErrMaxRetry, path.Join(dir, prefix+"*"+suffix),
                )
            }
        } else {
            return *new(File), err
        }
    }
}

// randomUint32Padded return math/rand/v2.Uint32 as left-0-padded string.
// The returned string always satisfies len(s) == 10 and '0' <= s[i] <= '9'.
func randomUint32Padded() string {
    // os.MkdiTemp does this thing. Just shadowing the behavior.
    // But there's no strong opinion about this;
    // It can be longer, or even shorter. We can expand this to
    // 9999999999 instead of 4294967295.
    s := strconv.FormatUint(uint64(rand.Uint32()), 10)
    var builder strings.Builder
    builder.Grow(len("4294967295"))
    r := len("4294967295") - len(s)
    for range r {
        builder.WriteByte('0')
    }
    builder.WriteString(s)
    return builder.String()
}
```

という感じになります。
`os.O_RDWR|os.O_CREATE|os.O_EXCL`でファイルを開ければよく、`File`自体には触りませんから、そこは`any`よい、という感じです。

`File`がtype paramになった都合上、`nil`を直接返せなくなってしまいますが、`*new(T)`でzero valueを作成して返せばそれでよいです。

### Example2: SafeWrite

逆に`File`が書けるのを期待するようなときについて考えてみます。例えば上記の`OpenFileRandom`でファイルを開いてからそこに内容を書き込み、最後に`Rename`することで最終的な名前にすることで、中途半端な状態が見えないようにする`SafeWrite`があったとすると、以下のようになります。

```go
type safeWriteFile interface {
    WriteFile
    CloseFile
    NameFile
    SyncFile
}

type safeWriteFsys[File safeWriteFile] interface {
    OpenFileFs[File]
    RenameFs
    RemoveFs
}

func SafeWrite[File safeWriteFile](fsys safeWriteFsys[File], name string, r io.Reader, perm fs.FileMode) error {
    dir := filepath.Dir(name)

    randomFile, err := OpenFileRandom(fsys, dir, "*.tmp", perm.Perm())
    if err != nil {
        return err
    }

    randomFileName := filepath.Join(dir, filepath.Base(randomFile.Name()))
    defer func() {
        _ = randomFile.Close()
        if err != nil {
            fsys.Remove(randomFileName)
        }
    }()

    bufP := bufpool.GetBytes()
    defer bufpool.PutBytes(bufP)

    buf := *bufP
    _, err = io.CopyBuffer(randomFile, r, buf)
    if err != nil {
        return err
    }

    err = randomFile.Sync()
    if err != nil {
        return err
    }

    err = fsys.Rename(randomFileName, filepath.Clean(name))
    if err != nil {
        return err
    }

    return nil
}
```

### 実際に複数ライブラリ相手に使ってみる

実際に複数のfilesystem abstraction libraryに対して動かしてみます。
(このsnippetは`vroot`が`go1.25rc1`指定なので`Go Playground`では動きません！`Go dev branch`モードは`go 1.25`扱いになるからです！)

```go
package main

import (
    "encoding/json"
    "fmt"
    "io/fs"
    "os"
    "path/filepath"

    billyosfs "github.com/go-git/go-billy/v5/osfs"
    "github.com/ngicks/go-fsys-helper/fsutil"
    vrootosfs "github.com/ngicks/go-fsys-helper/vroot/osfs"
    "github.com/spf13/afero"
)

func main() {
    tempDir, err := os.MkdirTemp("", "")
    if err != nil {
        panic(err)
    }
    defer func() {
        os.RemoveAll(tempDir)
    }()

    aferoBase := filepath.Join(tempDir, "afero")
    err = os.Mkdir(aferoBase, fs.ModePerm)
    if err != nil {
        panic(err)
    }
    aferoFsys := afero.NewBasePathFs(afero.NewOsFs(), aferoBase)
    {
        f, err := fsutil.OpenFileRandom(aferoFsys, "", "*.tmp", fs.ModePerm)
        if err != nil {
            panic(err)
        }
        fmt.Printf("afero file name = %q\n", f.Name())
        _ = f.Close()
    }

    goBillyBase := filepath.Join(tempDir, "go-billy")
    err = os.Mkdir(goBillyBase, fs.ModePerm)
    if err != nil {
        panic(err)
    }
    billyFsys := billyosfs.New(goBillyBase, billyosfs.WithBoundOS())
    {
        f, err := fsutil.OpenFileRandom(billyFsys, "", "*.tmp", fs.ModePerm)
        if err != nil {
            panic(err)
        }
        fmt.Printf("billy file name = %q\n", f.Name())
        _ = f.Close()
    }

    vrootBase := filepath.Join(tempDir, "vroot")
    err = os.Mkdir(vrootBase, fs.ModePerm)
    if err != nil {
        panic(err)
    }
    vrootFsys, err := vrootosfs.NewRooted(vrootBase)
    if err != nil {
        panic(err)
    }
    defer vrootFsys.Close()
    {
        f, err := fsutil.OpenFileRandom(vrootFsys, "", "*.tmp", fs.ModePerm)
        if err != nil {
            panic(err)
        }
        fmt.Printf("vroot file name = %q\n", f.Name())
        _ = f.Close()
    }

    var seen []string
    fs.WalkDir(os.DirFS(tempDir), ".", func(path string, d fs.DirEntry, err error) error {
        if err != nil || path == "." || d.IsDir() {
            return err
        }
        if d.Type().IsRegular() {
            seen = append(seen, path)
        }
        return nil
    })

    bin, _ := json.MarshalIndent(seen, "", "    ")
    fmt.Printf("seen path = %s\n", string(bin))
}
```

実行してみるとこんな感じ。動いてますね。

```
$ go run .
afero file name = "/0121404083.tmp"
billy file name = "/tmp/1301612191/go-billy/0632349466.tmp"
vroot file name = "/tmp/1301612191/vroot/3571826966.tmp"
seen path = [
    "afero/0121404083.tmp",
    "go-billy/0632349466.tmp",
    "vroot/3571826966.tmp"
]
```

こうしてみてみると[afero]の`BasePathFs`の挙動はだいぶいただけないです。`/`-prefixも削除されていたら文句なかったんですが。

### Interoperabilityへのアドバイス

思いつく限りのInteroperabilityへのアドバイスをリストしていきます。見つけ次第追記していくかも。なんか思い当たるものがあったら教えてください。

#### 全プラットフォームでgo vetしよう

下記のbashscriptで`GOOS=android`, `GOOS=ios`を除いた全OS/ARCHの組み合わせに対して`go vet ./...`がかけられます。

```bash
#!/bin/bash

supported_list=$(go tool dist list)

IFS=$'\n'
for os_arch in $supported_list; do
  IFS='/' read -r os arch <<< $os_arch
  if [[ $os == "android" ]] || [[ $os == "ios" ]]; then
    continue
  fi
  echo ${os_arch}:
  GOOS=${os} GOARCH=${arch} go vet ./...
  echo
done
```

エラーが表示されなければとりあえず全プラットフォームでコンパイルまではする、というのがわかります。

#### エラーは再定義するしかない

`syscall.ELOOP`など、大部分のエラーが`plan9`にはありません。そもそも`plan9`にsymlinkはありません。
全プラットフォームで(`GOOS=ios`, `GOOS=android`を除く)でコンパイルできる程度にするためには、この辺のエラーを再定義する必要があります。

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/fsutil/errdef/err.go#L1-L12

`plan9`側では適当にエラーを定義します。

https://github.com/ngicks/go-fsys-helper/blob/2adb17618ef755813afd6fa49910134eb94d3ceb/fsutil/errdef/err_plan9.go#L1-L30

#### FileのNameメソッドは信用しない

- `*os.File`における`Name`の挙動は`os.Open`に渡されたパスを返すことですが、ライブラリ、例えば[afero]などでは`Open`に渡したのとは別のものが返ってくることがあります。
- ライブラリによってこの挙動に差があります:
  - [afero]の`OsFs`+`BasePathFs`では、`BasePathFs`の挙動としてsubpathとして指定されたpath prefixを`Name`から削除する挙動があります。
  - [go-billy]の`osfs`(`BoundOS`)は素直に[*os.File]の`Name`の返り値を返すのでフルパスが返ります。
  - `vroot`の`osfs`も[go-billy]と同様にフルパスが返ります。(単なる[*os.Root]のラッパーなので、それの挙動が透けています。)
- これらはinterfaceですので、どのような実装になっているかは定かではありません。
  - 実装によっては下層の実装は`osfs`なんだけどパスの変換などを行っているかもしれません
- ですのでInteroperabilityを重視するならばbase name以外を信用するのは危険です。

#### 親ディレクトリの存在チェックをする

- 親ディレクトリの有無で挙動差が生まれることがよくあるようです:
  - [go-billy]\: 勝手に作る
    - 前述通り、親ディレクトリがなかったらとりあえず作ってしまいます。
  - [afero]\: 実装による
    - `MemMapFs`では親ディレクトリがなくてもファイルが作れます
    - `OsFs`では勝手に作られません。
  - `vroot`: 親ディレクトリは勝手に作られません
    - 明確に書かれていませんがinterface規約として親ディレクトリがないのにファイル作成が成功してはいけないというものがあります。
      - `acceptancetest`によって親ディレクトリがない時のファイル作成がエラーする挙動がテストされています。
- windowsでは「親ディレクトリがない」と「パスにファイルが含まれる」が一緒
  - POSIX APIにおける`ENOTDIR`の概念がないようです。
  - 試しに適当な`go module`で`os.Open(".\\go.mod\\foo)`とか呼び出してみるとわかりますが、`ERROR_PATH_NOT_FOUND`が返ってきます。
- 親ディレクトリがない時エラーになってほしいなら存在チェックをしたほうがよいです。
- パスコンポーネント上にファイルがあるときに特別な処理を行いたいときは存在チェックをしたほうが良いです。

#### Renameで上書きは常に通じるわけではない

- `sftp`でマウントしている環境だとマウント設定によってはrenameで上書きしようとするとエラーでした。

なんかfilesystem abstraction libraryの話ではなくなってしまってますが・・・

## 雑感: Claude Code使ってました

このライブラリの実装にClaude Codeを大いに活用したため雑感を記します。

- 型とdoc comment、テストケースを細かく分けて何をするかをdoc commentで書いておいて、あとよろしく！で完成するのでかなり便利です。
- といいつつ中身を見てると無駄なことをすることがあるので手動で若干直す必要があります。
- `strings.Split`を使うコードを出してくるところを`strings.SplitSeq`を使うように指示すると、一旦全部sliceに受けるようなコードにしてきたりして手で直さざるを得ないところがありました。
  - 新しめなAPIは学習されてないらしく、使われ方のパターンを把握していない感じがありますね。
- `*overlayfs.Fs`のコードはだいぶダメだったのでほぼ人が書いてます。
  - 何をどうするかをコメントで自己説明すればもっといい感じに出力してくれたかもしれませんが、それほぼ英語の自然言語でコーディングしてるだけなのでコードを手で書けばいいやって思って人間が書きました。
- 逆に`*synthfs.Fs`は`tarfs`を参考にしてっていうとほぼokなものが出ました。
  - ただしロック周りのロジックは結構手で直しました。
  - ぱっとみよさそうなんですが、もしかしたらあとで人が書き直すかもしれないです
- 似たようなコードを若干変えて特別な考慮を加える、みたいなタスクは得意そうですね。
- `gh`コマンド経由でGitHub Actionsの結果をみて修正をさせようと試みましたが急に見当違いなことを言い出しました。こういう使い方は厳しいようです。
  - というのも、claudeは`print debug`や、実験コードを一旦生成して仮説検証したり、実際に動く環境があることを活かすので、実行環境がないとその方法が通じなく、あてずっぽうなことを言うしかなくなるのかなと思います。
  - なんとなくですが、エラーの文章が発生する環境=自分が今動いている環境と思うようなバイアスがあるような感じがします。
- バグを見つけてくるのはすごい得意です。ここうまく動かないけどなんでかな？と聞いたら、人がするような、debugを仕込んで実行して状態を観測したら原因を推測して・・・というループを行って発見してきます。
  - **このループを回すのがものすごい高速なので人がデバッグする速度じゃ追いつけません**
  - 複数のモジュールにまたがって起きるバグはこの方法が通用しませんが、`Go`のコンパイルオプションにはoverlayというモジュールの一部を差し替えるものがありますから、何かしらのmcpツールで元のソースを編集してoverlayに渡してコンパイルさせられるように環境を整えたらこのデバッグ方法を行ってもらえるかもしれません。そうなってくると人間のデバッグ力じゃもうすでに追いつけないものになりますね。
  - REST APIやgRPCをまたぐとそれでも通用しないです。
  - 前述した[#73868](https://github.com/golang/go/issues/73868)はclaudeが見つけて指摘してきました。誰も報告してなければ報告しようかと思いましたがすでにされていましたね。貢献失敗！
- コードベース読んで説明するのも同様に得意そうです。
- ちなみにこの記事は一部AIに書かせましたが、丁寧でざっくりしすぎてしまい、筆者の文書ににじみ出る雑味が消えてしまったので、ほとんどの変更をrevertして人力で書き直しています。

## おわりに

- コンセプトとして[*os.Root]準拠のfilesystem abstractionを考えてみました。
  - これは[afero], [go-billy], [hackpadfs]などでのinterfaceのつらみを解消しつつ、[*os.Root]的なroot外へのsymlinkへの脱出をエラー扱いするものです。
- 逆に、filesystem-abstraction-library-agnosticなutilityについても提案しました。

今後は

- `rc2`を待ちます(`rc1`のバグによってGitHub Actions上のテストが通過しないため)
- AIが書いたテストやコードをレビューしてない部分があるのでちゃんと読んでリファクタするなりをします。
- 自作ライブラリ内で使ってたたきにたたきます。
  - [この記事](https://zenn.dev/ngicks/articles/go-code-generation-from-ast-and-type-info)や、[この記事](https://zenn.dev/ngicks/articles/go-code-generation-from-ast-and-type-info-cloner)で触れている、code generatorのファイル書き込み部分に`overlayfs`を使用し、トップレイヤを`synthfs`のin-memory filesystemにしておき、`packages.Config`のOverlayにメモリコンテンツを渡すことで書き出し前に型チェックをかけることをひそかに構想しています。だから`go1.25`をずっと待っていたんです。
- `vroot-adapter`という別の名前のモジュールを作成し、[afero], [go-billy]と相互に変換がかけられるようにします。
  - ただし[afero]に関してはベストエフォートになります。
- `vroot-adapter`下にいろんなアダプターをおいておきたいと思っています。例えば
  - `sftp`
  - `smb`
  - `nfs`
  - `s3`
  - etc, etc.
- `vroot over stream`, `stream over gRPC`で、`gRPC`経由で相互にファイルシステムを公開しあえないかなと思っています。
  - 同一マシン内でIPCするときに適切にファイルシステムを共有する方法をずっと探っていたので、それに対する答えとしてこれを考えています。
  - `tar`を送り付けあってもいいんですがそれだとあまりにオーバーヘッドが大きいので。
  - もしかしたら`NFS over gRPC`にしたほうが最適な実装は得られるかもしれないです
- [github.com/natefinch/lumberjack](https://github.com/natefinch/lumberjack)や[github.com/cavaliergopher/grab](https://github.com/cavaliergopher/grab)のfsys interfaceに書き出す版が欲しいとずっと思っていたので、そのへんを`fsutil`下に実装するかも。

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
[Go 1]: https://go.dev/doc/go1
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
[embed.FS]: https://pkg.go.dev/embed@go1.24.4#FS
[errors.New]: https://pkg.go.dev/errors@go1.24.4#New
[errors.Is]: https://pkg.go.dev/errors@go1.24.4#Is
[errors.As]: https://pkg.go.dev/errors@go1.24.4#As
[errors.Join]: https://pkg.go.dev/errors@go1.24.4#Join
[fmt.Errorf]: https://pkg.go.dev/fmt@go1.24.4#Errorf
[fs.ErrNotExist]: https://pkg.go.dev/io/fs@go1.24.4#ErrNotExist
[http.Server]: https://pkg.go.dev/net/http@go1.24.4#Server
[*http.Server]: https://pkg.go.dev/net/http@go1.24.4#Server
[io.Copy]: https://pkg.go.dev/io@go1.24.4#Copy
[io.EOF]: https://pkg.go.dev/io@go1.24.4#EOF
[io.Reader]: https://pkg.go.dev/io@go1.24.4#Reader
[io.Writer]: https://pkg.go.dev/io@go1.24.4#Writer
[fs.FS]: https://pkg.go.dev/io/fs@go1.24.4#FS
[fs.File]: https://pkg.go.dev/io/fs@go1.24.4#File
[*os.File]: https://pkg.go.dev/os@go1.24.4#File
[*os.Root]: https://pkg.go.dev/os@go1.24.4#Root
[log/slog]: https://pkg.go.dev/log/slog@go1.24.4
[syscall.Errno]: https://pkg.go.dev/syscall@go1.24.4#Errno
[text/template]: https://pkg.go.dev/text/template@go1.24.4

<!-- references to Go library -->

[github.com/spf13/afero]: https://github.com/spf13/afero/
[afero]: https://github.com/spf13/afero/
[github.com/go-git/go-billy]: https://github.com/go-git/go-billy
[go-billy]: https://github.com/go-git/go-billy
[github.com/hack-pad/hackpadfs]: https://github.com/hack-pad/hackpadfs
[hackpadfs]: https://github.com/hack-pad/hackpadfs
