---
title: "[Go]*os.Rootベースのファイルシステム抽象化とライブラリ非依存ヘルパーの提案"
emoji: "📂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## \*os.Rootベースのファイルシステム抽象化とライブラリ非依存ヘルパーの提案

こんにちは。

`go1.25rc1`がリリースされましたね。

https://x.com/golang/status/1932876844849594525

DRAFT RELEASE NOTEは以下となります。

https://tip.golang.org/doc/go1.25

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
    - major versionが多すぎる！現在`v6`のpreleaseがいくつか出ています。issue内でもそれは`v6`を示唆する記述があります。
      - [Go v1のリリース日は2012-03-28](https://go.dev/doc/devel/release#go1), [Go1.16のリリースが2021-02-16](https://go.dev/doc/devel/release#go1.16)であることを踏まえると多すぎる。
  - [hackpadfs]:
    - 筆者はお試し以上に使ったことがないため特に深いことは言えないですが、
    - `go-billy`同様に[interfaceがcomposableになるように細かく分けられています](https://github.com/hack-pad/hackpadfs/blob/v0.2.4/fs.go)
    - `go-billy`以上に[fs.FS]に寄せてあって、[ベースとなるFSはfs.FSです](https://github.com/hack-pad/hackpadfs/blob/v0.2.4/fs.go#L11-L13)
      - つまりwrite operationは常に[fs.File]を`type-assertion`で書き込み可能なinterfaceに「広げる」必要があります。
- ところで`Go`は[docker], [podman]など、コンテナ基盤で盛んに使われています。
- [#20126](https://github.com/golang/go/issues/20126)でかつて`secure-join`というpath traversalを防ぎながらjoinを行うAPIの追加がproposeされましたが、ちゃんとできるのかわかんないからという理由でcloseされています。
  - [dependencies](https://pkg.go.dev/github.com/cyphar/filepath-securejoin?tab=importedby)を見れば`k8s.io`の各種パッケージからインポートされているのがわかります。
- [#67002](https://github.com/golang/go/issues/67002)で[*os.Root]が提案され、[Go 1.24]で一部のメソッド、[Go 1.25]で残りすべてのメソッドが追加されます。
  - [*os.Root]は特定のサブディレクトリの下のみを操作できる`os`のメソッドを提供するものです。
  - path traversalに加えて、symlinkによって特定のサブディレクトリの外に出るのを防ぐことができます。
  - `secure-join`とは違って`/proc`の下などで起きるようなkernel空間でのsymlink resolutionなどは考慮に入れませんが、逆にGoがサポートするすべてのプラットフォームで動作させることができます。
- [*os.Root]には[masterで確認すれば](https://pkg.go.dev/os@master#Root)わかりますが`Truncate`を除いたsymlinkやhardlink作成機能が追加されています。
- てことはこの[*os.Root]の持ってるcapabilityはベースと考えてもいいのでは・・・？
- そういうわけで、[*os.Root]の持っているmethod setを基本としたfilesystem abstraction libraryを作ってみます。
- できました: https://github.com/ngicks/go-fsys-helper/tree/main/vroot
- 逆に`type OpenFileFs[File any] interface { OpenFile(name string, flag int, perm fs.FileMode) (File, error) }`みたいに具体的なファイルの型をtype paramにすればライブラリに縛られないヘルパーが定義できますね
- つくり中です: https://github.com/ngicks/go-fsys-helper/tree/main/fsutil

## 環境

```
$ go version
go version go1.25rc1 linux/amd64
```

```
export GOTOOLCHAIN=go1.25rc1
```

の環境変数を設定することで、`go.mod`の内容にかかわらず`go1.25rc1`で実行できます！

## issue

先に述べておきますが、[*os.Root]にはまだバグがあります。

- [#73868](https://github.com/golang/go/issues/73868)
  - `OpenRoot` -> `*os.Root.OpenRoot` -> `*os.Root.OpenRoot`で開いた子root、孫rootで開いたファイルに対して`Readdir`系のメソッドを呼び出すと`ENOENT`が返ってくるというもの。
  - 書いてありますが`*os.Root.OpenRoot`が原因であるので、落としてきたsdkを手動で修正すればうまく動作します。
  - `lstat`が呼ばれるときのprefixがちゃんとわたっていないことが問題です。`lstatat`が存在していればこんなバグも起こらなかったんでしょうが、どうもPOSIX APIには存在しないようです。
  - とりあえず`rc2`を待ちましょう。
- [#69509](https://github.com/golang/go/issues/69509)
  - `wasip1`でパスの取り扱いがおかしいというもの。

以下はバグではなくenhancementですがこれは[#67002](https://github.com/golang/go/issues/67002)の中で述べられていた、各プラットフォーム向けの最適なAPIを使用することで最適な実装を行おうというものです。

- [#73076](https://github.com/golang/go/issues/73076)
  - 各プラットフォーム向けに最適な実装をしようというもの
  - 多分、ファイルに対するIO操作のほうがよほど時間がかかるのでこの最適化がされなくても十分な実行速度を持てると思いますが、std libraryはあらゆるものから使われるわけですからメンテナンス性を確保できている限り、速ければ速いほどいいですよね。

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
ディレクトリ自体は設定ファイルなりなんなりで自由に変えることができるが、どのディレクトリに書き込んでいるかはプログラムの関心から外したいという欲求が筆者にはよくありますし、実際に[fs.FS]が実装されたのはそういった欲求は広く存在するからだと思います。

[fs.FS]は単にinterfaceであるため、それさえ満たせばdata sourceは何でもよいことになります。
当然、どこかのディレクトリ以下でもいいし、samba/nfsなどのネットワークファイルシステム, tar/zipなどのアーカイブファイル、なんならin-memoryの構造でもかまいません。

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
実装にもよりますが、というのは始祖に近いdockerはデフォルトでは上記のように名前空間による隔離でコンテナを動かします([runc](https://github.com/opencontainers/runc))が、[Open Container Initiative Runtime Specification](https://github.com/opencontainers/runtime-spec)でcontainer runtimeが仕様化されているため、相互に取り換え可能な別の動作方式のものも多数存在することをさしています。例えば、guest kernelを動かしたり([gVisor](https://gvisor.dev/))、軽量KVMで分離されていたり([libkrun](https://github.com/containers/libkrun))、QEMUマシンの中で動いていたり([Kata Containers](https://katacontainers.io/)), windowsのHyper-V上で動いていたり([runhcs](https://learn.microsoft.com/ja-jp/virtualization/windowscontainers/deploy-containers/containerd#hcs))といろいろあるためです。

[#20126](https://github.com/golang/go/issues/20126)でかつてsecure-joinというpath traversalを防ぎながらjoinを行うAPIの追加がproposeされましたが、完全な実装の難しさからcloseされています。しかし、[github.com/cyphar/filepath-securejoin](https://github.com/cyphar/filepath-securejoin)の[dependencies](https://pkg.go.dev/github.com/cyphar/filepath-securejoin?tab=importedby)を見れば、`k8s.io`の各種パッケージからインポートされていることがわかります。こちらは`/proc`の下などをコンテナの名前空間を見せたりするのに使う安全策を組み込んでいるような記述があります。

[*os.Root]はこれとは違って`/proc`でのカーネル空間で起きるsymlink resolutionなどは考慮に加えません。

### \*os.Root

[#67002](https://github.com/golang/go/issues/67002)で[*os.Root]が提案され、[Go 1.24]で一部のメソッド、[Go 1.25]で残りすべてのメソッドが追加されます。

[*os.Root]は特定のサブディレクトリの下のみを操作できる`os`のメソッドを提供するものです。
path traversalに加えて、symlinkによって特定のサブディレクトリの外に出るのを防ぐことができます。

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

`Truncate`を除いたsymlinkやhardlink作成機能も含まれており、ファイルシステム操作に必要なほぼすべての機能が揃っています。

## vroot: \*os.Root-based filesystem abstraction

[*os.Root]がstdに入っちゃったらこれとうまくやれないfilesystem abstraction libraryはつらい思いをするのは目に見えています。
現状で筆者は[go.uber.org/zap](https://github.com/uber-go/zap)を使っている古いプロジェクトと[log/slog]を使用するプロジェクトを行ったり来たりしてわけわからなくなってつらい思いをしています。

[*os.Root]はinterfaceではなくstructなのでmethodの追加は破壊的変更とはみなされません。ですのでメソッドが追加されること自体は普通にあると想定すべきだと思いますが。
ですがすでに`os`パッケージはほぼ硬直していて、新しいファイル操作の関数が追加されることはめったにない・・・というか1.0から追加がありません。
追加があったのは[os.CreateTemp](https://pkg.go.dev/os@master#CreateTemp)などのショートハンド的なものです。こういったものは今後も増えるときは増えるでしょう。

ということで上記のmethod setをinterfaceとしても今後ほぼ追加はないでしょうし、あったとしてもごくまれにしか使わないのではない(のでinterfaceには加えないでいいか、もしくはextension interface patternでいいのではない)でしょうか？
ということでこれをベースとしたinterfaceを定義すると、大抵のファイルシステム操作で困らず、なおかつmajor versionを更新しないで済むような物が作れます。
せっかくなら作っちゃえばいいかな？普段なら面倒なのでやらないのですが最近claude MAX 100$プランを契約してしまったため後押しが強力で、やればいいかという気分になったので作って見ました。

https://github.com/ngicks/go-fsys-helper/tree/main/vroot

まだめちゃくちゃWIPですがここにホストて置いています。

### interface

```go
// Fs represents capablities [*os.Root] has as an interface.
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

### RootedとUnrooted

ちょっといろいろ考えて、`Rooted`と`Unrooted`でinterfaceをわけ、どちらも実装するスタイルにすることにしました。

`Rooted`は[*os.Root]と同じく、path traversalとsymlink escapeを防ぐことがinterfaceの規約となりますが、
`Unrooted`はsymlink escapeを許可します。

筆者の経験上、実際に開発をしているとsymlinkは大体の場合どこか遠くのディレクトリどうしをつなぐときに使いますし、
それを読んで内容を確かめたいときもよくあります。この時にいちいち通常の`os.Open`に立ち戻りに行くのは面倒なコードになりますし、
やってることがごちゃごちゃしすぎて認知的な負荷にもなります。
ということで、こういったケースにおいてのみ便利な`Unrooted`もあったほうが良いだろうという判断を下しました。

`Unrooted`は`OpenRoot`で`Rooted`を開けますが、`Rooted`は`Unrooted`を開けません。
これは[*os.Root]の実装に引っ張られて結果的にこうなっています。

実は[*os.Root]は[openat(2)](https://man7.org/linux/man-pages/man2/openat.2.html)などの、`fd`からの相対パス開きができるAPIに依存しています。
[*os.Root]の各methodにパスが渡されるとパスコンポーネントを分割し、(unixの場合)`O_NOFOLLOW`付きで`openat`を呼び出し、ディレクトリを1つずつ開いていきます。symlinkが見つかった場合には`readlinkat`を使って読み取り、読んだリンクのパスコンポーネントを同様に分解して`openat`で1つずつ移動していきます。

ここで、特徴となるのは[*os.Root]は`fd`によってrootを保持し続けるため、例えば、`os.OpenRoot`でrootが開かれた後に`os.Rename`などでディレクトリが移動されたとしても相変わらずそのディレクトリの下で動作し続けることです。
`fd`からパスへ、正確な逆変換を行う方法が存在しないため、`Rooted`から`Unrooted`を開くことができません。
逆に、`Unrooted`は`openat`によらない実装になっていてよく、セキュリティーも`Rooted`に比べて劣る存在であって良いので、さらにそこから`Unrooted`を開いてもよいことにしてあります。

### とりあえず実装しておいたもの

#### osfs

とりあえず[*os.Root]からinterfaceを生成できないとほとんど意味がないのでとりあえずこれは必要です。

```go
type Rooted struct {
	root *os.Root
}

type Unrooted struct {
	root string // absolute path to the root directory
}
```

`os`パッケージが`errPathEscapes`をエクスポートしないため

https://github.com/golang/go/blob/go1.25rc1/src/os/file.go#L421

文字列を見てエラーを差し替える部分を作っておきます。

https://github.com/ngicks/go-fsys-helper/blob/9e840465a3445f79c554d3757ce1b4a0d33e877c/vroot/osfs/rooted.go#L45-L58

文字列比較はやらないでいいならやりたくないですが、こうしないと`errors.Is(err, ErrPathEscapes)`でテストをかけないので仕方なくやっています。

#### WalkDir

`fs.WalkDir`と互換なものとして`vroot.WalkDir`を定義しておきます。

https://github.com/ngicks/go-fsys-helper/blob/9e840465a3445f79c554d3757ce1b4a0d33e877c/vroot/walk.go#L79

interfaceがsymlinkの存在をもとから考慮に入れているので`fs.WalkDir`と違ってsymlinkをresolveしてたどってもよいことにしてあります。

#### to/from fs.FS

`fs.FS`と`vroot`の相互変換を定義します。

https://github.com/ngicks/go-fsys-helper/blob/9e840465a3445f79c554d3757ce1b4a0d33e877c/vroot/iofs_from.go#L27-L43

https://github.com/ngicks/go-fsys-helper/blob/9e840465a3445f79c554d3757ce1b4a0d33e877c/vroot/iofs_from.go#L174-L186

https://github.com/ngicks/go-fsys-helper/blob/9e840465a3445f79c554d3757ce1b4a0d33e877c/vroot/iofs_to.go#L21-L30

https://github.com/ngicks/go-fsys-helper/blob/9e840465a3445f79c554d3757ce1b4a0d33e877c/vroot/iofs_to.go#L72-L81

#### ReadOnly

`read-only`なinterfaceも欲しいので作っておきます。これは間違って書かないようにするための安全策としてあったほうが良いですね

https://github.com/ngicks/go-fsys-helper/blob/9e840465a3445f79c554d3757ce1b4a0d33e877c/vroot/readonly.go#L12-L20

https://github.com/ngicks/go-fsys-helper/blob/9e840465a3445f79c554d3757ce1b4a0d33e877c/vroot/readonly.go#L114-L120

#### overlay

これが一番欲しかったものかもしれない。

`overlay fs`です。複数の`vroot.Rooted`を重ね合わせて一つのfsに見せかけます。

https://github.com/ngicks/go-fsys-helper/blob/9e840465a3445f79c554d3757ce1b4a0d33e877c/vroot/overlay/overlay.go#L32-L79

- 書き込みはすべて`top layer`にのみ起こるようになっています。
- 下層のレイヤー群はすべてread-onlyかつstaticという前提があります。
- 下層のレイヤーにしかないファイルに書き込もうとした場合、`top layer`にまずコピーします。
  - コピーは`chmod`などでアトリビュートを変えるか、write modeでファイルを開いたときにおこります。
  - 本当はファイルに対して初めて`Write`が呼ばれたときにコピーが起きるようにしたかったのですが、タイミングの制御があまりにも難しいので開いた瞬間になっています
- 下層にあるファイルを消さないまま消えたように見せるために、white out listを別口管理します。そのために`MetadataStore` interfaceの実装が必要です。
  - これはこのissueを参考にしています: https://github.com/opencontainers/image-spec/issues/24
  - 今`vroot`に入っている実装は単にwhite outされたファイルの名前をリストにしたテキストファイルに書き出すシンプルな実装のもののみです。
  - 実際にはtrieを保存できるオブジェクトストレージとかSQLiteとかで実装したほうがいいとは思いますが、依存するモジュールを増やしたくなかったので簡単に作れそうなこれになっています。
    - 多分ファイルが少ないうちはこの実装方法で困ることはないです。

余談ですがinterfaceとコメントで何するかの説明だけしてclaude codeにあとは実装よろしくってやったらちっともうまくいかなかったので人がほとんど書き直しています。
`MetadataStore`の実装はほぼAIが出してきたものそのままです。

#### synthfs(=in-memory fs)

以下の記事で書いたsynthetic filesystemの`vroot`版です。

https://zenn.dev/ngicks/articles/go-virtual-mesh-fs-for-os-copyfs

完全にin-memoryなfilesystem+任意のバッキングストレージという組み合わせで成り立つfilesystemで、
file pathによるtrieを構築し、各ノードには適当なバッキングストレージのファイルを追加できるようにします。
ファイルのblob以外のメタデータはin-memoryで管理します。

https://github.com/ngicks/go-fsys-helper/blob/84ed803754b44067aa0449f832b753b1b19083c1/vroot/synthfs/fs.go#L21-L55

バッキングストレージをメモリーからのみallocateするようにすると、これが[afero]でいうところの`MemMapFs`になります。

作った動機は上記の記事内で説明していますが、実はそれ以外にも理由があって、
[afero]の`MemMapFs`がパスの取り扱いが正しくなく、`fs.FS`に変換して`fs.Walk`をかけたとに在るのにパスが見つからない、というのが発生していたため、`trie`による管理に変えることで原理的にそれが起こらなくしたというのもあります。

## tarfs

`vroot`とは直接は関係ないですが、以下の記事で作った`tarfs`にsymlinkとhardlinkの取り扱いを加え、さらに、`vroot`と組み合わせるのを意識して「symlink解決時にsub-rootより上の階層に移動するか」の設定を追加しました。

https://zenn.dev/ngicks/articles/go-tar-reader-implement-reader-at

https://github.com/ngicks/go-fsys-helper/tree/84ed803754b44067aa0449f832b753b1b19083c1/tarfs

symlinkを無視するのがデフォルト挙動ですが、デフォルトでハンドルしたほうがいい気もするので(破壊的に変更になりますが)そのように変えるかもしれません。
symlinkありのtarでも普通に動いているので使えそうな感じですが、世にどんなエッジケースがあるのかよくわからないのでしばらくタグをつけずに様子見します。

ちなみに`fstest.TestFS`がsymlinkを考慮するのは`go1.25`以降であるようなので、このモジュールのテストを`go1.24`で動かすとfailします。

## fsutil: filesystem-abstraction-library-agnostic helpers

https://github.com/ngicks/go-fsys-helper/tree/main/fsutil

多分どのfilesystem abstraction libraryでも動作するようなヘルパーを書いてるけど、interfaceがそれぞれ違うせいでどれかでしか使えないっていうことがあるともったいないですよね？
ということで、逆の発想としてどのライブラリでも使えるようにヘルパーを作る仕組みを考えてみます。

「考えてみます」といっても簡単で、`File`の部分をtype parameterにしてinterfaceを再定義し、これを引数となるようなジェネリック関数を作ればよいだけです。

つまり、`Fs`, `File`のinterfaceを以下のように、method単位で細かく切り分け、

```go

// Fs files

type ChmodFs interface {
	Chmod(name string, mode fs.FileMode) error
}

type ChownFs interface {
	Chown(name string, uid int, gid int) error
}

type ChtimesFs interface {
	Chtimes(name string, atime time.Time, mtime time.Time) error
}

type LchownFs interface {
	Lchown(name string, uid int, gid int) error
}

type LinkFs interface {
	Link(oldname string, newname string) error
}

type LstatFs interface {
	Lstat(name string) (fs.FileInfo, error)
}

type MkdirFs interface {
	Mkdir(name string, perm fs.FileMode) error
}

type MkdirAllFs interface {
	MkdirAll(name string, perm fs.FileMode) error
}

type OpenFileFs[File any] interface {
	OpenFile(name string, flag int, perm fs.FileMode) (File, error)
}

type ReadLinkFs interface {
	ReadLink(name string) (string, error)
}

type RemoveFs interface {
	Remove(name string) error
}

type RemoveAllFs interface {
	RemoveAll(name string) error
}

type RenameFs interface {
	Rename(oldname string, newname string) error
}

type StatFs interface {
	Stat(name string) (fs.FileInfo, error)
}

type SymlinkFs interface {
	Symlink(oldname string, newname string) error
}

// File interfaces

type ChmodFile interface {
	Chmod(mode fs.FileMode) error
}

type ChownFile interface {
	Chown(uid int, gid int) error
}

type CloseFile interface {
	Close() error
}

type NameFile interface {
	Name() string
}

type ReadFile interface {
	Read(b []byte) (n int, err error)
}

type ReadAtFile interface {
	ReadAt(b []byte, off int64) (n int, err error)
}

type ReadDirFile interface {
	ReadDir(n int) ([]fs.DirEntry, error)
}

type ReaddirFile interface {
	Readdir(n int) ([]fs.FileInfo, error)
}

type ReaddirnamesFile interface {
	Readdirnames(n int) (names []string, err error)
}

type SeekFile interface {
	Seek(offset int64, whence int) (ret int64, err error)
}

type StatFile interface {
	Stat() (fs.FileInfo, error)
}

type SyncFile interface {
	Sync() error
}

type TruncateFile interface {
	Truncate(size int64) error
}

type WriteFile interface {
	Write(b []byte) (n int, err error)
}

type WriteAtFile interface {
	WriteAt(b []byte, off int64) (n int, err error)
}

type WriteStringFile interface {
	WriteString(s string) (n int, err error)
}
```

必要なものだけに依存するように関数側のtype paramを工夫します。
例えば、`os.CreateTemp`と似たような機能をもつヘルパーを定義するとすると、

```go
type OpenFileFs[File any] interface {
	OpenFile(name string, flag int, perm fs.FileMode) (File, error)
}

func OpenFileRandom[FS OpenFileFs[File], File any](fsys FS, dir string, pattern string, perm fs.FileMode) (File, error)
```

という感じになります。
`os.O_RDWR|os.O_CREATE|os.O_EXCL`でファイルを開ければよく、`File`自体には触りませんから、そこは`any`よい、という感じです。

逆に`File`が書けるのを期待するようなとき、例えば上記の`OpenFileRandom`でファイルを開いてからそこに内容を書き込み、最後に`Rename`することで最終的な名前にすることで、中途半端な状態が見えないようにする`SafeWrite`を考えてみます。

`OpenFileRandom`できて、writeできてcloseできて書き込み終わったらsyncできないとだめなので・・・という感じで以下のようになります。

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

大したことはないですが、こんな感じですね。
ワンポイントアドバイスとして、`File`の`Name`メソッドを信用しないというのがあります。上記スニペット中では`filepath.Base`でbase nameだけとって、他は無視しています。
これまた[afero]の話になりますが、[afero]の`BasePathFs`は文字通りbase paseにpathを結合してから下層のfsys、例えば`os.Create`などを呼び出していました。`File`の`Name`メソッドは`Open`などに渡されたパスをそのまま返す、とあります。[afero]の`BasePathFs`経由で得られた`File`の`Name`はbase pathを結合した後のパスを返してくるわけです。ということで自分が渡したパスが必ず`Name`から帰ってくるわけではないので、base name部分以外は信用しないつくりになっています。逆にbase nameが正しく帰ってこない場合はinterfaceの規約を満たしていないとみなすよりほかないでしょう。

## おわりに

- コンセプトとして[*os.Root]準拠のfilesystem abstractionを考えてみました。
  - これは[afero], [go-billy], [hackpadfs]などでのinterfaceのつらみを解消しつつ、[*os.Root]的なroot外へのsymlinkへの脱出をエラー扱いするものです。
- 逆に、filesystem-abstraction-library-agnosticなutilityについても提案しました。

今後は

- 自作ライブラリ内で使ってたたきにたたきます。
  - [この記事](https://zenn.dev/ngicks/articles/go-code-generation-from-ast-and-type-info)や、[この記事](https://zenn.dev/ngicks/articles/go-code-generation-from-ast-and-type-info-cloner)で触れている、code generatorのファイル書き込み部分に`overlay`を使用し、トップレイヤを`synthfs`のin-memory filesystemにしておき、`packages.Config`のOverlayにメモリコンテンツを渡すことで書き出し前に型チェックをかけることをひそかに構想しています。
- `rc2`を待ちます(`rc1`のバグによってGitHub Actions上のテストが通過しないため)
- `vroot-adapter`という別の名前のモジュールを作成し、[afero], [go-billy]と相互に変換がかけられるようにします。
  - ただし[afero]に関してはベストエフォートになります。

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
