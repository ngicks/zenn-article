---
title: "libkrunをpodmanで(普通に)使えるようにする"
emoji: "⛓️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["podman", "libkrun"]
published: true
---

## libkrunをpodmanで(普通に)使えるようにする

ビルドしたり、バイナリを配置したりのセットをして[libkrun]を使うだけの記事です。

筆者はそれができるようになるまで少し戸惑いがあったので、手順や必要なことをまとめています。

ただ、ネットにはセットアップして使えるようにするまでの記事はあまりないように見受けられます。
となると多分普通は簡単にできるんだと思いますが、筆者の環境ではそうではありませんでした。

## 環境

記事の想定環境は、wsl2上で動作するUbuntuにパッケージマネージャとしての[nix]を追加したものです。

`nix`がなくても同じことはできますが、`crun`のビルドの部分が面倒になるかも。

```
$ wsl.exe --version
WSL バージョン: 2.6.1.0
カーネル バージョン: 6.6.87.2-1
WSLg バージョン: 1.0.66
MSRDC バージョン: 1.2.6353
Direct3D バージョン: 1.611.1-81528511
DXCore バージョン: 10.0.26100.1-240331-1435.ge-release
Windows バージョン: 10.0.26200.7623

$ cat /etc/issue
Ubuntu 24.04.3 LTS \n \l

$ nix --version
nix (Nix) 2.33.1
```

[podman-static]でビルドした[podman]を利用しています

```
$ podman --version
podman version 5.7.1
```

## libkrun

[libkrun]はdynamic library(`.so`ファイル)であり、それを用いるとコンテナを個別の軽量VMの中で動かすことができるようになります。

[crun]\([podman]などが用いる\)や[runc]\([Docker]などが用いる\)がデフォルトで行う[namespace(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)による隔離は、ホストとコンテナでLinux Kernelを共有します。
VMによる隔離はKernelを共有しませんのでこれより強いレベルの隔離だといえます。

似たようなものに[Kata Containers]がありますが、こちらは(筆者は詳しいことがわかりませんが)1つのVMの中で複数のコンテナを動かすため、その点で異なります。

[podman]がデフォルトで利用するLow Level Runtimeの[crun]で`libkrun`を用いるには、[annotationでrun.oci.handler=krunと指定します](https://github.com/containers/crun/blob/1.26/crun.1.md#runocihandlerhandler)

以下みたいな感じ

```
podman container run --annotation=run.oci.handler=krun --rm -it ...
```

こうすることでコンテナごとに独立した`KVM`が実行されるようです。

[libkrun]は説明されている通り、実際にはコンテナを動かす専用のツールではなく、プロセスを部分的に隔離された環境で実行するためのライブラリです。
実際、[podman desktop]は[macOS(ARM64)上ではkrunkitを用いてPodman向けVMを作る](https://podman-desktop.io/docs/podman/creating-a-podman-machine)ようです。

## キーポイント

### nix由来のcrunはlibkrunが有効になっていない

そもそも`nixpkgs`から配布されている`crun`を使えばいいだけじゃないのかと思いきや実はそうではありません。

https://search.nixos.org/packages?channel=unstable&show=crun&query=crun

```
$ nix shell nixpkgs#crun

$ which crun
/nix/store/3mapg01bapra4ckr5ncqkws2308xd23k-crun-1.26/bin/crun

$ crun --version
crun version 1.26
commit: 3241e671f92c33b0c003cd7de319e4f32add6231
rundir: /run/user/1000/crun
spec: 1.0.0
+SYSTEMD +SELINUX +APPARMOR +CAP +SECCOMP +EBPF +CRIU +YAJL
```

`+LIBKRUN`がないため有効になっていないことがわかります。

### static buildしない

[libkrunはcrunからdlopenによって読み込まれます](https://github.com/containers/crun/blob/1.26/src/libcrun/handlers/krun.c#L617-L619)が、[static binaryから`dlopen`は動作しないようです。](https://www.openwall.com/lists/musl/2012/12/08/4)

斜め読みした感じ`dlopen`は`ld`に依存してるけどstatic binaryだとそれがロードされないからって感じですかね。

下記のように検証するとなるほどたしかにstatic binaryは`ld`を要求しないヘッダーになっています。
まあいらないんだからそりゃそうかって感じですね。

```
$ ldd $(which claude)
        linux-vdso.so.1 (0x00007ffd3452a000)
        libc.so.6 => /nix/store/wb6rhpznjfczwlwx23zmdrrw74bayxw4-glibc-2.42-47/lib/libc.so.6 (0x000072bba1000000)
        /lib64/ld-linux-x86-64.so.2 => /nix/store/wb6rhpznjfczwlwx23zmdrrw74bayxw4-glibc-2.42-47/lib64/ld-linux-x86-64.so.2 (0x000072bba128b000)
        libpthread.so.0 => /nix/store/wb6rhpznjfczwlwx23zmdrrw74bayxw4-glibc-2.42-47/lib/libpthread.so.0 (0x000072bba1284000)
        libdl.so.2 => /nix/store/wb6rhpznjfczwlwx23zmdrrw74bayxw4-glibc-2.42-47/lib/libdl.so.2 (0x000072bba127f000)
        libm.so.6 => /nix/store/wb6rhpznjfczwlwx23zmdrrw74bayxw4-glibc-2.42-47/lib/libm.so.6 (0x000072bba0f08000)
```

```
$ readelf -l $(which claude) | grep interpreter
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

```
$ ldd $(which podman)
$       not a dynamic executable
```

```
$ readelf -l $(which podman) | grep interpreter
# exit code 1
```

ということで[podman-static]から配布されたバイナリを用いるのはどうやっても不可能ということになります。
自分でビルドする必要があります。

### krun -> crunなsymlinkを作って$PATHから見えるところに置く

よく見ると下記に`krun`は`crun`へのsymlinkですと明記されています。

https://github.com/containers/crun/blob/1.26/krun.1.md#description

ビルドしたときにsymlinkは出力されなかったので、見逃していました。

[`podman`ではLow Level Runtimeは`$PATH`から探索されます。](https://github.com/containers/podman/blob/v5.7.1/libpod/oci_conmon_common.go#L143)
`krun`も探索されるので`crun`と動階層とかに置いておくとよいでしょう。

よくよくソースを読むと[crunは呼び出しパスが`"krun"`であるときにふるまい変わります](https://github.com/containers/crun/blob/1.26/src/crun.c#L417-L423)。
`--annotation=run.oci.handler=krun`をつけると`podman`が勝手に`--runtime`オプションを`=krun`に切り替えるような挙動になっているように思います。

## ビルドして配置

- `libkrun`は動的ライブラリ(`.so`ファイル)で、これを読み込むためには
- `crun`は
  - static binaryにしてはいけない
  - `libkrun`オプションを有効にしないといけない
- `PATH`に`krun`が存在している必要がある

ことがわかりました。

前述通り、[podman-static]でビルドする`crun`はstatic binaryだし、`nix`配布の`crun`は`+LIBKRUN`がついていないので有効になっていません。
自前でビルドする必要があるようです。

### ビルド

```
cd /path/to/you/are/confortable/to/clone
git clone https://github.com/containers/crun
cd ./crun
# detached HEAD防止
git switch main
git pull --all --ff-only
git reset --hard tags/1.26
```

ここで適当な場所に`shell.nix`を作成し、ビルドに必要な依存物詰め合わせした環境を組み立てます。

```nix: shell.nix
{ pkgs ? import <nixpkgs> {} }:
pkgs.mkShell {
  buildInputs = with pkgs; [
    yajl
    libseccomp
    libcap
    systemd
    pkg-config
    go-md2man
    python3
    autoconf
    automake
    libtool

    libkrun
  ];
}
```

筆者の環境は元からいろいろ入っているのでこれだけで行けましたが、何もない環境なら`glibc`とか追加でいろいろ入れないとだめかもしれません。

`shell.nix`と動階層で下記を実行します。

```
nix-shell
```

初回だとダウンロード時間が結構かかるかもしれません。

あとは普通にビルドします。

```
cd /path/to/you/have/cloned/crun
./autogen.sh
./configure --with-libkrun
make
```

```
./configure --help
```

を見ると、ほかのビルドオプションも列挙されています。

```
./crun --version
crun version 1.26
commit: 3241e671f92c33b0c003cd7de319e4f32add6231
rundir: /run/user/1000/crun
spec: 1.0.0
+SYSTEMD +SELINUX +APPARMOR +CAP +SECCOMP +EBPF +LIBKRUN +YAJL
```

`+LIBKRUN`がついていれば成功です。

### 配置してsymlink

適当な位置に置きます。
とりあえず`$HOME/.local/bin`でいいんじゃないでしょうか。

```bash
mkdir -p $HOME/.local/bin
cp ./crun $HOME/.local/bin
```

`PATH`に`$HOME/.local/bin`が入っていない場合はくわえてください

```bash
case ":${PATH}:" in
    *:"$HOME/.local/bin":*)
        ;;
    *)
        export PATH="$HOME/.local/bin:$PATH"
        ;;
esac
```

symlink貼ります。

```bash
ln -s crun $HOME/.local/bin/krun
```

### /dev/kvmの権限を緩める

[libkrunは/dev/kvmへのアクセスを必要とします](https://docs.rs/kvm-ioctls/latest/src/kvm_ioctls/ioctls/system.rs.html#97-101)

デフォルトでは`/dev/kvm`は`660 root:kvm`の権限となるようです。

通常のワークロードではKVM機能を利用したいユーザーを`kvm`グループに参加させればよいようですが、
何がどう悪いのか、
筆者環境では`kvm`グループへ参加しているユーザーでも`Permission Denied`でコンテナを実行できません。

シングルユーザーの環境では`666`に権限を緩めるのもさほど危なくないはずなので、緩めてしまうことにします。

```bash
sudo chmod 666 /dev/kvm
```

設定永続化するために[/etc/udev](https://man7.org/linux/man-pages/man7/udev.7.html)にルールを追加します。

```bash
echo 'KERNEL=="kvm", MODE="0666"' | sudo tee /etc/udev/rules.d/99-kvm.rules
```

`podman container run...`の実行経路のどこかでsupplemental groupの情報が抜け落ちてるのかも。近いうちに緩めなくてもいいようにしたいです。

## 実行して効果確認

筆者環境は[home-manager](https://github.com/nix-community/home-manager)でコントロールされており、すでに`libkrun`を追加済みです。
`libkrun.so`は`~/.nix-profile/lib`以下に配置されますが、`nix`のパッケージとしてビルドしていない`crun`はここを探しに行かないので`LD_LIBRARY_PATH`の設定も必要です。

```bash
LD_LIBRARY_PATH=$HOME/.nix-profile/lib podman container run -it --rm --runtime=$HOME/.local/bin/crun --annotation=run.oci.handler=krun docker.io/library/ubuntu:noble-20260113
```

コンテナの中で`dmesg`してみます。

```
root@1eea1efa72c1:/# dmesg | grep virt
[    0.000000] Command line: reboot=k panic=-1 panic_print=0 nomodule console=hvc0 rootfstype=virtiofs rw quiet no-kvmapf init=/init.krun       virtio_mmio.device=4K@0xd0000000:5 virtio_mmio.device=4K@0xd0001000:6 virtio_mmio.device=4K@0xd0002000:7 virtio_mmio.device=4K@0xd0003000:8 virtio_mmio.device=4K@0xd0004000:9 tsi_hijack  --
[    0.064782] Booting paravirtualized kernel on KVM
[    0.072625] Kernel command line: reboot=k panic=-1 panic_print=0 nomodule console=hvc0 rootfstype=virtiofs rw quiet no-kvmapf init=/init.krun       virtio_mmio.device=4K@0xd0000000:5 virtio_mmio.device=4K@0xd0001000:6 virtio_mmio.device=4K@0xd0002000:7 virtio_mmio.device=4K@0xd0003000:8 virtio_mmio.device=4K@0xd0004000:9 tsi_hijack  --
[    0.507090] virtio-mmio: Registering device virtio-mmio.0 at 0xd0000000-0xd0000fff, IRQ 5.
[    0.507152] virtio-mmio: Registering device virtio-mmio.1 at 0xd0001000-0xd0001fff, IRQ 6.
[    0.507183] virtio-mmio: Registering device virtio-mmio.2 at 0xd0002000-0xd0002fff, IRQ 7.
[    0.507191] virtio-mmio: Registering device virtio-mmio.3 at 0xd0003000-0xd0003fff, IRQ 8.
[    0.507198] virtio-mmio: Registering device virtio-mmio.4 at 0xd0004000-0xd0004fff, IRQ 9.
[    0.529243] virtiofs virtio3: Cache len: 0x20000000 @ 0x100000000
[    0.571755] virtio-fs: tag <> not found
[    0.577382] VFS: Mounted root (virtiofs filesystem) on device 0:19.
```

あからさまに`KVM`が動作していますね。

ホスト環境からコンテナのPIDが開いているファイル情報を検索し、そこからKVMを使っているかも確認します。

`ls -q`でcontainer idを取り出し、

```
# docker.io/library/ubuntu:noble-20260113 を用いていて動作中のコンテナが一つな想定
$ container_id=$(podman container ls --filter 'ancestor=docker.io/library/ubuntu:noble-20260113' -q)
```

inspectでpidを取り出します。

```
$ pid=$(podman container inspect ${container_id} --format '{{.State.Pid}}')
```

`/proc/${pid}/fd`以下を見ます。環境によってはこれを見るのに`sudo`が必要かもしれません。

```
$ ls /proc/${pid}/fd -la | grep kvm
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 10 -> anon_inode:kvm-vm
lrwx------ 1 ngicks ngicks  64 Feb  6 01:04 22 -> anon_inode:kvm-vcpu:0
lrwx------ 1 ngicks ngicks  64 Feb  6 01:04 24 -> anon_inode:kvm-vcpu:1
lrwx------ 1 ngicks ngicks  64 Feb  6 01:04 26 -> anon_inode:kvm-vcpu:2
lrwx------ 1 ngicks ngicks  64 Feb  6 01:04 28 -> anon_inode:kvm-vcpu:3
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 30 -> anon_inode:kvm-vcpu:4
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 32 -> anon_inode:kvm-vcpu:5
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 34 -> anon_inode:kvm-vcpu:6
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 36 -> anon_inode:kvm-vcpu:7
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 38 -> anon_inode:kvm-vcpu:8
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 40 -> anon_inode:kvm-vcpu:9
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 42 -> anon_inode:kvm-vcpu:10
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 44 -> anon_inode:kvm-vcpu:11
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 46 -> anon_inode:kvm-vcpu:12
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 48 -> anon_inode:kvm-vcpu:13
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 50 -> anon_inode:kvm-vcpu:14
lrwx------ 1 ngicks ngicks  64 Feb  6 01:06 52 -> anon_inode:kvm-vcpu:15
```

どう見ても`KVM`が動いていますね。

他にも以下も確認しました。

- `--mount type=bind`
- `dmesg`のログタイムスタンプからコンテナごとに別のVMが動作していること
- 外部への通信が成立すること
  - なぜかはここを参照: https://rheb.hatenablog.com/entry/libkrun-networking
  - ただし、interfaceが存在しない(`ls /sys/class/net/ -la`に`lo`しかない)
    - 普通の(？)コンテナ環境ではドライバによって`tap`デバイスがあったりします。
    - iface情報見て何かするアプリはうまく動かなくなるかもしれないですね。
- コンテナ内では`tty`が`not a tty`を返したり、`htop`のようなTUIアプリが部分的にEnterを受け付けなかったり、interactive shellを利用するには困難さがあります。

## おわりに

[libkrun]を使って軽量VMでコンテナを隔離できることを確認しました。

ただそれをやるだけの記事があまりないところを見ると、余りに常識なのか、`Fedora`系だったら最初から有効になっているのかもしれません。

`run.oci.handler`アノテーション自体はexperimentalと述べられていたりするのでどこかが変わる可能性はあります。

LLM cli agentを動かすときのような、より強い隔離が欲しいときに`libkrun`は便利なのではないでしょうか。

<!-- links -->

[nix]: https://nixos.org/
[podman]: https://podman.io/
[podman desktop]: https://podman-desktop.io/
[Docker]: https://www.docker.com/
[podman-static]: https://github.com/mgoltzsche/podman-static
[Kata Containers]: https://katacontainers.io/
[crun]: https://github.com/containers/crun
[runc]: https://github.com/opencontainers/runc
[libkrun]: https://github.com/containers/libkrun
