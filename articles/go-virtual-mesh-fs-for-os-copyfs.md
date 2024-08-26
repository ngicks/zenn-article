---
title: "Goのos.CopyFSのために複数のfsの内容を混ぜ込むことができるFsを作る"
emoji: "🧰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## Goのos.CopyFSのために「複数のfsの内容の配置を変えたりしながら混ぜ込む」ことができるFsを作る

こんにちは。

先日[Go1.23.0](https://tip.golang.org/doc/go1.23)がリリースされました。このリリースには[os.CopyFS]などが含まれ、[Go1.22](https://tip.golang.org/doc/go1.22)で追加された[(\*tar.Writer).AddFS], [(\*zip.Writer).AddFS]などと合わせて、ますます[fs.FS]が利用しやすくなりました。

便利になってうれしい反面、以下のようなユースケースで単なる`AddFS`などではうまくいかず、結局手動でファイルをやりくりすることになります。

- 元のFSと追加先でファイルの配置のレイアウトを変えたい
  - 例えば`foo/bar`, `baz/qux`を統合して`anywhereelse/bar`, `anywhereelse/qux`にしたいとか
- 追加先には元FSのファイルを圧縮したり、展開したり、分割したり、結合したりして置きたい
- インメモリコンテンツを追加したい
  - 元のFSにあるファイルのhash値とか
  - バージョン情報とか
  - それらをまとめたjsonファイルとか

なるだけ[fs.FS]を渡したらそれで終わるように、[fs.FS]としての見せ方を工夫できたらもっと楽できそうですよね。
ということで本記事では「複数のfsの内容の配置を変えたりしながら混ぜ込む」ことを実現できるfilesystem実装を作成します。

この記事は`Go 1.23.0`を対象とします。確認環境は`linux/amd64`ですが、基本的には`os`も`arch`も無関係な話しかしないはずです。

## やること

本記事では「複数のfsの内容の配置を変えたりしながら混ぜ込む」ことができるFsを作る、を実現します

- stdの範疇ではメモリに乗りきらないコンテンツを使った仮想ファイルシステムを作ることができないことと、
- 既存の有名なfilesystem abstraction libraryのout-of-boxではこのユースケースを実現できなさそうなこと

を説明し、

- `afero.Fs`の実装であるin-memory virtual filesystemを作成し、
- ファイルとして任意のストレージ(e.g. [fs.FS])をバックとした仮想ファイルを追加できるようにする

ことで、「複数のfsの内容の配置を変えたりしながら混ぜ込む」ことができるfilesystemを作成できることを示します。

## os.CopyFS, (\*tar.Writer).AddFS, (\*zip.Writer).AddFS

まず[os.CopyFS]、[(\*tar.Writer).AddFS], [(\*zip.Writer).AddFS]とは何ぞやって言う話を一応しておきましょう。

### os.CopyFS

[os.CopyFS]は、名前の通り、[fs.FS]をosのfilesystem上にコピーします。`Go1.23.0`から追加されました。

```go
func CopyFS(dir string, fsys fs.FS) error
```

`dir`で、ディレクトリを指定してそこにコンテンツをすべてコピーします。

[embed.FS]は内容物のpermission bitを`0o444`(ファイル)と`0o555`(ディレクトリ)にしてしまうこともあり(([[1]](https://github.com/golang/go/blob/go1.23.0/src/embed/embed.go#L345),[[2]](https://github.com/golang/go/blob/go1.23.0/src/embed/embed.go#L335),[[3]](https://github.com/golang/go/blob/go1.23.0/src/embed/embed.go#L226-L231))、`CopyFS`は厳密に[fs.FS]の内容をリスペクトせず、`0o666`(ファイル)、`0o777`(ディレクトリ、ただしumaskがかかる)とbitwise orしたpermissionでファイルを作ります。

### (\*tar.Writer).AddFS, (\*zip.Writer).AddFS

[(\*tar.Writer).AddFS], [(\*zip.Writer).AddFS]双方ともにtar, zipを書き込むときに[fs.FS]のコンテンツを構造を保ったまま全部書き込めるものです。

```go
func (tw *Writer) AddFS(fsys fs.FS) error
func (w *Writer) AddFS(fsys fs.FS) error
```

ただし、[os.CopyFS]と違いこちらはpermission bitを特に広げることなく書き込むようですので、[embed.FS]の内容を書き込む際には注意が必要です。
[\*zip.Reader](https://pkg.go.dev/archive/zip@go1.23.0#NewReader)などを使ってzipやtarの展開を行うならばpermissionは好きに変えられるのでアーカイブ内でどうなっていても問題ないことが多いですが、`tar -xf`などで展開すると狭すぎるpermissionで苦しむことがあります(1敗)。

## 既存のfs.FSの作成方法

活用の仕方が整備されているのはわかりましたが、[fs.FS]はどのように作成できるのでしょうか。

- os上のディレクトリから: [os.DirFS](https://pkg.go.dev/os@go1.23.0#DirFS)
- ビルド時に埋め込み: [embed.FS]
- `Go`プロログラムで構造を記述: [fstest.MapFS](https://pkg.go.dev/testing/fstest@go1.23.0#MapFS)
  - ただし`io.Reader`ではなく`[]byte`でコンテンツを記述するため、メモリに乗せきれるサイズに限られる。
- zipファイルから: [zip.Reader](https://pkg.go.dev/archive/zip@go1.23.0#Reader)
- tarファイルから: [github.com/nlepage/go-tarfs](https://github.com/nlepage/go-tarfs)

この中で任意の構造を作成できるのは[fstest.MapFS](https://pkg.go.dev/testing/fstest@go1.23.0#MapFS)のみですが、これは`[]byte`でコンテンツを渡す形になっているため、メモリに乗りきらないようなサイズのコンテンツの偽装には向きません。動的にファイルを追加することも想定されていなさそうなので多分しないほうが良いです。

## 既存のfilesystem abstraction library

サードパーティの実装の中で、複数のソースから[fs.FS]を作り上げることができるものがありますが、そのようなfilesystem abstraction libraryで似たようなことができないことを示します。

GitHub starがそれなりについていて、書き込み可能なfilesystem abstraction interfaceを提供するライブラリは以下の三つなどがあります。

- https://github.com/spf13/afero
- https://github.com/go-git/go-billy
- https://github.com/hack-pad/hackpadfs

`afero`が最もstarが多く5.9kありますがほか二つは300以下程度です。

### 各ライブラリの様式

一応各ライブラリの違いと使い方の様式を軽く紹介しておきます。

`afero`は以下のように、全部入りの単一interfaceを定義します。

https://github.com/spf13/afero/blob/v1.11.0/afero.go#L36-L52

https://github.com/spf13/afero/blob/v1.11.0/afero.go#L54-L102

これは`OsFs`などによって実装されます。

https://github.com/spf13/afero/blob/v1.11.0/os.go#L24-L32

これは基本的に`BasePathFs`と組み合わせて特定のディレクトリ以下を`afero.Fs`として見せます。

https://github.com/spf13/afero/blob/v1.11.0/basepath.go#L17-L28

https://github.com/spf13/afero/blob/v1.11.0/basepath.go#L47-L49

後者の二つも大体似たような様式ですが、明確な違いとして、extension interfaceパターンを採用しています。

例えば`go-billy`では

https://github.com/go-git/go-billy/blob/v5.5.0/fs.go#L60-L89

という風に基本となるinterfaceを定義し

https://github.com/go-git/go-billy/blob/v5.5.0/fs.go#L131-L149

という感じで、`Basic`以上の機能はextension interfaceとして定義されます。

`hackpadfs`はもっと[fs.FS]に寄った実装になっていて

https://github.com/hack-pad/hackpadfs/blob/v0.2.4/fs.go#L11-L13

extension interfaceの粒度ももっと小さくなっています

https://github.com/hack-pad/hackpadfs/blob/v0.2.4/fs.go#L22-L33

### 各ライブラリのfsの偽装能力

各ライブラリともに今回やりたいユースケースである「複数のfsの内容の配置を変えたりしながら混ぜ込む」ということは、out-of-boxでサポートしていません。

- `afero`は[CopyOnWriteFs](https://github.com/spf13/afero/blob/v1.11.0/copyOnWriteFs.go#L13-L23)で、2つの`afero.Fs`を重ね合わせられます。
- `go-billy`は[Mount](https://github.com/go-git/go-billy/blob/v5.5.0/helper/mount/mount.go#L17-L24)によって、あるパス以下に、他の`Fs`をマウントすることができます。
- `hackpadfs`も同様に[mount.FS](https://github.com/hack-pad/hackpadfs/blob/v0.2.4/mount/fs.go#L21-L29)によってマウントが可能です。

`afero`の`CopyOnWriteFs`によるoverlayとupper layerにだけ書き込めるという特性は大変便利ですが今回やりたいことを実現できるものではありません。
後者二つのmount機能も同様に、実際にディレクトリのbind mountをfs上でするときのような挙動を実現できるので便利ですが、今回やりたいのはファイル単位の配置の偽装なので、これでユースケースを実現するのは無理そうです。

見たところこれ以上filesystem同士を組み合わせるという発想の何かはそこまでなさそうなので、「複数のfsの内容の配置を変えたりしながら混ぜ込む」を、これらのライブラリの組み合わせで実現するのは難しそうです。

## synth.Fs:「数のfsの内容の配置を変えたりしながら混ぜ込む」ことができるfilesystem

ということで作ることにしました。

ソースは以下でホストされます

https://github.com/ngicks/go-fsys-helper/tree/main/aferofs/synth

コンセプトとしてはin-memoryでlinux風な挙動をする仮想ファイルシステムを構成し、他のデータソースへのviewをファイルとして追加できるようにします。
こういう実装のしかたにすると`go-billy`や`hackpadfs`のmount的なことができなくなりますが、ファイルの配置を自由に入れかえて見せかけることができる利点があります。
mountと違い、今回の実装はほかの[fs.FS]を単なるファイルへのviewして使い、そこ書き込むつもりがありません。

### 実装

```go
package synth

// Fs constructs a synthetic filesystem that combines file-like views from different data sources,
// to synthesize them into an imitation filesystem.
//
// Fs accepts different data sources or backing storage as a virtual file.
// [Fs.AddFile] adds file-like view backed by arbitrary implementations into Fs.
// Or passing [FileViewAllocator] to [New] will allocate a new file-like view using it when [Fs.Create] or
// [Fs.OpenFile] with os.O_CREATE flag is called.
//
// Fs behaves as an in-memory filesystem if created with [MemFileAllocator].
//
// Fs tries its best to mimic ext4 on the linux.
// So it has difference when running on windows.
type Fs struct {
	// unexported fields
}

func New(umask fs.FileMode, allocator FileViewAllocator, opt ...FsOption) *Fs
```

`afero.Fs`としてふるまうように実装しました。これ単に知名度が高いことから周辺ライブラリが充実しているかも、という期待からです。

```go
var _ afero.Fs = (*Fs)(nil)

func (fsys *Fs) Chmod(name string, mode fs.FileMode) error
func (fsys *Fs) Chown(name string, uid int, gid int) error
func (fsys *Fs) Chtimes(name string, atime time.Time, mtime time.Time) error
func (fsys *Fs) Create(name string) (afero.File, error)
func (fsys *Fs) Mkdir(name string, perm fs.FileMode) error
func (fsys *Fs) MkdirAll(path string, perm fs.FileMode) error
func (fsys *Fs) Name() string
func (fsys *Fs) Open(name string) (afero.File, error)
func (fsys *Fs) OpenFile(path string, flag int, perm fs.FileMode) (afero.File, error)
func (fsys *Fs) Remove(name string) error
func (fsys *Fs) RemoveAll(name string) error
func (fsys *Fs) Rename(oldname string, newname string) error
func (fsys *Fs) Stat(name string) (fs.FileInfo, error)
```

任意のデータソースを追加できるようなメソッドを追加します。

```go
// AddFile adds a FileView to given path.
// If nonexistent, the path prefix is made as directories with permission of 0o777 before umask.
// If the path prefix contains a file, it returns syscall.ENOTDIR.
// If basename of path exists before AddFile, it will be removed.
func (f *Fs) AddFile(path string, fileData FileView) error

// FileView is a pointer to a file-like data stored in a backing storage.
//
// FileView is currently only assumed to be a regular file.
type FileView interface {
	// Open opens this FileView.
	// Implementations may or may not ignore flag.
	//
	// Open should return a newly created file handle.
	// *Fs may call Open many times and may return results as different files.
	// Therefore some attributes, e.g. file offset, should be managed separately.
	//
	// flag is same that you can use with os.OpenFile,
	// namely one of os.O_RDONLY, os.O_WRONLY or os.O_RDWR bitwise-or'ed
	// with any or none of os.O_APPEND, os.O_CREATE, os.O_EXCL, os.O_SYNC or os.O_TRUNC.
	Open(flag int) (afero.File, error)
	// Stat is a short hand for Open then Stat.
	Stat() (fs.FileInfo, error)
	// Truncate is a short hand for Open then Truncate.
	// Readonly implementations may return a bare syscall.EROFS, or similar errors.
	Truncate(size int64) error
	// Close notifies the backing storage
	// that this FileView is no longer referred by name.
	//
	// The file opened by calling Open method may still
	// exist and be used.
	//
	// The returned error might be ignored.
	Close() error
	// Rename notifies the backing storage that the FileView
	// is now referred as newname.
	Rename(newname string)
}
```

この`FileView`を、別の[fs.FS]やin-memory contentから作成できると内容を任意に混ぜ込めるというわけです。
`*synth.Fs`の`Create`などを読んで新規にファイルを作れると、ユースケースに上げた「インメモリコンテンツを追加したい、元のFSにあるファイルのhash値とか」を実現できるので、できるようにしておきます。

```go
// NewFsFileView builds FileData that points a file stored in fsys referred as path.
func NewFsFileView(fsys fs.FS, path string) (FileView, error) {
	//
}

// FileViewAllocator allocates new FileView at path.
type FileViewAllocator interface {
	Allocate(path string, perm fs.FileMode) FileView
}

var _ FileViewAllocator = (*MemFileAllocator)(nil)

type MemFileAllocator struct {
	// ...unexported fields...
}

func NewMemFileAllocator(clock clock.WallClock) *MemFileAllocator {
	//
}

func (m *MemFileAllocator) Allocate(path string, perm fs.FileMode) FileView {
	//
}
```

### 使用例: os.CopyFSとともに

このexampleは以下でもホストされます。

https://github.com/ngicks/go-fsys-helper/tree/main/aferofs/example/synth

exampleではユースケースが実現できていることを確認するために、

- ファイルを適当に分割して
- それぞれのsha256sumと、
- 元のファイルのsha256sumをとり
- 適当なjsonファイルとしてhash sumを保存する

という、筆者が仕事で本当に実装したことのあるものを選びました。
これは`S3`にファイルを上げるために100MiBずっこにファイルを分割してsha256sumをとっておき、ダウンロード後に検証できるようにするというやつです。

ですのでやりたいことは、

- 元の`fs.FS`のファイルを、`*synth.Fs`には、ここでは適当に３分割して、バラバラのファイルとして追加し、
- それぞれ分割ファイルのsha256sumをとって,
- 分割前のファイルsha256sumも同様に取り、
- それぞれのhash sumをJSON objectにして適当なパス、ここでは`foo/hashes.json`に書き込みます。

実際のos上のファイルシステムには`archive`と呼ばれる１ファイルだけがあり、分割ファイルは仮想的なfileのview、`foo/hashes.json`はメモリ上だけに存在する状態です。

この状態で[os.CopyFS]を呼び出して、分割されたファイルと`hashes.json`が書きだされることを確認します。

```go
package main

import (
	"crypto/sha256"
	"embed"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"hash"
	"io"
	"io/fs"
	"maps"
	"os"
	"strconv"

	"github.com/ngicks/go-fsys-helper/aferofs"
	"github.com/ngicks/go-fsys-helper/aferofs/clock"
	"github.com/ngicks/go-fsys-helper/aferofs/synth"
)

//go:embed data
var dataFsys embed.FS

var (
	// taken by split and sha256sum.
	// run `split -b 1024 archive` to split files.
	knownSha256Sum = map[string]string{
		"total":      "959a684d6e104edacb67511844dbba2b80673c7542d7b194d310db4dbb9b0621",
		"foo/arch_0": "ca0dbca1bee04a53a0f64c4b878fdd1cdc4ef496c813dbd63260701267357b0b",
		"foo/arch_1": "f5b57f729c4c20caa5978babd8b02aa020d9829ee5a43033b9d887b4b4d8b09b",
		"foo/arch_2": "5fd8a55d55d0581f81129d9d277f9aefe32d9b1f9a4dd2d941c36b5fa268ee72",
	}
)

func main() {
	clock := clock.RealWallClock()
	synthFs := synth.New(0, synth.NewMemFileAllocator(clock), synth.WithWallClock(clock))

	for i := range 3 {
		view, err := synth.NewRangedFsFileView(dataFsys, "data/archive", int64(i*1024), 1024)
		if err != nil {
			panic(err)
		}
		err = synthFs.AddFile("foo/arch_"+strconv.FormatInt(int64(i), 10), view)
		if err != nil {
			panic(err)
		}
	}

	hashesTaken := hashFsys(&aferofs.IoFs{Fs: synthFs}, []string{"foo/arch_0", "foo/arch_1", "foo/arch_2"})

	f, err := synthFs.Create("foo/hashes.json")
	if err != nil {
		panic(err)
	}
	enc := json.NewEncoder(f)
	enc.SetIndent("", "    ")
	err = enc.Encode(hashesTaken)
	_ = f.Close()
	if err != nil {
		panic(err)
	}

	tmpDir, err := os.MkdirTemp("", "")
	if err != nil {
		panic(err)
	}
	fmt.Printf("tmp dir: %s\n", tmpDir)
	defer os.RemoveAll(tmpDir)

	err = os.CopyFS(tmpDir, &aferofs.IoFs{Fs: synthFs})
	if err != nil {
		panic(err)
	}

	dirFs := os.DirFS(tmpDir)

	var seen []string
	err = fs.WalkDir(dirFs, ".", func(path string, d fs.DirEntry, err error) error {
		if path == "." || err != nil {
			return err
		}
		seen = append(seen, path)
		return nil
	})
	if err != nil {
		panic(err)
	}

	fmt.Printf("%s: %#v\n", tmpDir, seen)

	hashesJson := map[string]string{}

	fsFile, err := dirFs.Open("foo/hashes.json")
	if err != nil {
		panic(err)
	}
	err = json.NewDecoder(fsFile).Decode(&hashesJson)
	_ = fsFile.Close()
	if err != nil {
		panic(err)
	}

	hashesTakenCopied := hashFsys(dirFs, []string{"foo/arch_0", "foo/arch_1", "foo/arch_2"})

	fmt.Printf("sum map: %#v\n", hashesJson)
	fmt.Printf("known hashes == sum taken form copied fsys: %t\n", maps.Equal(knownSha256Sum, hashesTakenCopied))
	fmt.Printf("known hashes == copied hashes.json: %t\n", maps.Equal(knownSha256Sum, hashesJson))
}

func hashFsys(fsys fs.FS, paths []string) map[string]string {
	hashes := map[string]string{}
	totalCopied := sha256.New()
	for _, s := range paths {
		hashes[s] = sha256Sum(fsys, s, totalCopied)
	}
	hashes["total"] = hex.EncodeToString(totalCopied.Sum(nil))
	return hashes
}

func sha256Sum(fsys fs.FS, path string, total hash.Hash) string {
	f, err := fsys.Open(path)
	if err != nil {
		panic(err)
	}
	defer f.Close()
	h := sha256.New()
	w := io.MultiWriter(h, total)
	_, err = io.Copy(w, f)
	if err != nil {
		panic(err)
	}
	return hex.EncodeToString(h.Sum(nil))
}
```

なんも書かずにしれっと使っていますが、`NewRangedFsFileView`という機能を追加しています。
これは[fs.FS]上のファイルをさらに[io.SectionReader](https://pkg.go.dev/io@go1.23.0#SectionReader)で包んで特定のバイトレンジだけの仮想的なファイルにするというものです。

実行すると以下が出力されます。

```
# go run ./example/synth/
tmp dir: /tmp/4033557916
/tmp/4033557916: []string{"foo", "foo/arch_0", "foo/arch_1", "foo/arch_2", "foo/hashes.json"}
sum map: map[string]string{"foo/arch_0":"ca0dbca1bee04a53a0f64c4b878fdd1cdc4ef496c813dbd63260701267357b0b", "foo/arch_1":"f5b57f729c4c20caa5978babd8b02aa020d9829ee5a43033b9d887b4b4d8b09b", "foo/arch_2":"5fd8a55d55d0581f81129d9d277f9aefe32d9b1f9a4dd2d941c36b5fa268ee72", "total":"959a684d6e104edacb67511844dbba2b80673c7542d7b194d310db4dbb9b0621"}
known hashes == sum taken form copied fsys: true
known hashes == copied hashes.json: true
```

思った通りになりました。ちょっと便利かもです。

## おわりに

ということで、「複数のfsの内容の配置を変えたりしながら混ぜ込む」ことができるFsを作る、を実現しました。

- stdの範疇ではメモリに乗りきらないコンテンツを使った仮想ファイルシステムを作ることができないことと、
- 既存の有名なfilesystem abstraction libraryのout-of-boxではこのユースケースを実現できなさそうなこと

を説明し、

- `afero.Fs`の実装であるin-memory virtual filesystemを作成し、
- ファイルとして任意のストレージ(e.g. [fs.FS])をバックとした仮想ファイルを追加できるようにする

ことで、「複数のfsの内容の配置を変えたりしながら混ぜ込む」ことができるfilesystemを作成できることを示しました。

もちろん全然テストコードを書いていないので荒が多そうですが、手元で動かすツールに使えるぐらいにはなったかなと思います。

まだまだいろいろと整備のしがいはあります。説明中では特に何にも触れませんでしたが、`synth.Fs`はパスとして[fs.FS]と同じルールのものしか受け付けません。つまりunix風かつ、`./foo`のような`./`や、`/`へのアクセスが許されません。こうすると実装がめちゃくちゃ単純になるのでこういったルールを敷いています。`afero.Fs`はinterfaceなので、ラッパーとなる`Fs`を定義してパスの変換をかけたらいいかなと思っていました。まだまだ色々作るべきものはあります。

[fs.FS]や`afero`や`hackpadfs`の存在は`Go`を一層魅力的にするように思います。他の言語での実装、例えば[denoのFileSystem trait](https://github.com/denoland/deno/blob/e53678fd5841fe7b94f8f9e16d6521201a3d2993/ext/fs/interface.rs#L104-L338)なんかも見たことがありますが、stdに取り込まれて認知度が高くてその周辺にツールがそろってるのはあんま見たことがない気がします。(すみません、言っといてなんですが普通に他にも仮想fsをstdに取り込んでる言語ありそうですね。調べてみようかな。ご存じだったら教えてください。)

[fs.FS]は便利で楽しいですね。どんどん使い倒していきましょう。

[os.CopyFS]: https://pkg.go.dev/os@go1.23.0#CopyFS
[fs.FS]: https://pkg.go.dev/io/fs@go1.23.0#FS
[(\*tar.Writer).AddFS]: https://pkg.go.dev/archive/tar@go1.23.0#Writer.AddFS
[(\*zip.Writer).AddFS]: https://pkg.go.dev/archive/zip@go1.23.0#Writer.AddFS
[embed.FS]: https://pkg.go.dev/embed@go1.23.0#FS
