---
title: "Goのos.CopyFSのために構造を偽装できるFsを作る"
emoji: "🧰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## Goのos.CopyFSのために構造を偽装できるFsを作る

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

## 既存のfs.FSの作成方法

まず、既存の[fs.FS]を作成方法をまとめます。

- os上のディレクトリから: [os.DirFS](https://pkg.go.dev/os@go1.23.0#DirFS)
- ビルド時に埋め込み: [embed.FS](https://pkg.go.dev/embed@go1.23.0#FS)
- `Go`プロログラムで構造を記述: [fstest.MapFS](https://pkg.go.dev/testing/fstest@go1.23.0#MapFS)
  - ただし`io.Reader`ではなく`[]byte`でコンテンツを記述するため、メモリに乗せきれるサイズに限られる。
- zipファイルから: [zip.Reader](https://pkg.go.dev/archive/zip@go1.23.0#Reader)
- tarファイルから: [github.com/nlepage/go-tarfs](https://github.com/nlepage/go-tarfs)

この中で任意の構造を作成できるのは[fstest.MapFS](https://pkg.go.dev/testing/fstest@go1.23.0#MapFS)のみですが、これは`[]byte`でコンテンツを渡す形になっているため、メモリに乗りきらないようなサイズのコンテンツの偽装には向きません。

## 既存のfilesystem abstraction library

既存のfilesystem abstraction libraryで似たようなことができないことを示します。

GitHub starがそれなりについていて、書き込み可能なfilesystem abstraction interfaceを提供するライブラリは以下の三つなどがあります。

- https://github.com/spf13/afero
- https://github.com/go-git/go-billy
- https://github.com/hack-pad/hackpadfs

`afero`が最もstarが多く5.9kありますがほか二つは300以下程度です。

### 各ライブラリの様式

`afero`は以下のように、全部入りの単一interfaceを定義しますが

https://github.com/spf13/afero/blob/v1.11.0/afero.go#L36-L52

ほか二つはextension interfaceパターンを採用しています。

例えば`go-billy`では

https://github.com/go-git/go-billy/blob/v5.5.0/fs.go#L60-L89

という風に基本となるinterfaceを定義し

https://github.com/go-git/go-billy/blob/v5.5.0/fs.go#L131-L149

という感じで、`Basic`以上のinterfaceは必要最低限なもののみを引数の型に指定する形になります。

`hackpadfs`はもっと[fs.FS]に寄った実装になっていて

https://github.com/hack-pad/hackpadfs/blob/v0.2.4/fs.go#L11-L13

extension interfaceの粒度ももっと小さくなっています

https://github.com/hack-pad/hackpadfs/blob/v0.2.4/fs.go#L22-L33

### 各ライブラリのfsの偽装能力

各ライブラリともに今回やりたいユースケースである「複数のfsの内容の配置を変えたりしながら混ぜ込む」ということは、out-of-boxでサポートしていません。

- `afero`は[CopyOnWriteFs](https://github.com/spf13/afero/blob/v1.11.0/copyOnWriteFs.go#L13-L23)で、2つの`afero.Fs`を重ね合わせられます。
- `go-billy`は[Mount](https://github.com/go-git/go-billy/blob/v5.5.0/helper/mount/mount.go#L17-L24)によって、あるパス以下にある`Fs`をマウントすることができます。
- `hackpadfs`も同様に[mount.FS](https://github.com/hack-pad/hackpadfs/blob/v0.2.4/mount/fs.go#L21-L29)によってマウントが可能です。

## vmesh:「数のfsの内容の配置を変えたりしながら混ぜ込むことができるfilesystem

ということで作ることにしました。

ソースは以下でホストされます

https://github.com/ngicks/go-fsys-helper/tree/main/aferofs/vmesh

### 設計方針

- `afero.Fs`の実装とすることにしました
  - これが最も知名度が高いため、適合する周辺ライブラリも多いかも、という期待からです。
  - 詳細は述べませんが、3者3様に色々微妙なところがあって、ほとんどinterfaceとして使わないことになるからです。

[os.CopyFS]: https://pkg.go.dev/os@go1.23.0#CopyFS
[fs.FS]: https://pkg.go.dev/io/fs@go1.23.0#FS
[(\*tar.Writer).AddFS]: https://pkg.go.dev/archive/tar@go1.23.0#Writer.AddFS
[(\*zip.Writer).AddFS]: https://pkg.go.dev/archive/zip@go1.23.0#Writer.AddFS
