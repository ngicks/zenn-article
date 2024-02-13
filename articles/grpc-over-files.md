---
title: "Goでstdin/stdoutごしにgRPCする"
emoji: "🤙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

# Overview

本記事では

プログラムからほかのプログラムを呼び出す方法として

- サブプロセスとしてプログラムを呼び出し、
- より環境への依存性が少ない方法としてstdin/stdoutを介した通信し、
- ネットワーク透過な方法としてgRPCを用いる

ことを実現できることを確認します。

# 前提知識

- プログラミングに対する一定の理解を有する
- [Go]の文法を理解している

## Background

アプリケーションが複数のプログラミング言語を用いて開発されるとき(あるいは同じ言語同士でも)、それらのプログラムがお互いの機能を呼び出しあうときいくつかのことなる方法をとることができます。

例えば

- 独立した実行ファイルとして各部をビルドし、サブプロセスとしてそれらを起動したうえで何かしらの方法で通信する
- [ffi]を用いてプログラムから直接関数を呼び出す
- [WASM]を用いる

などがあるかと思います。

- [ffi](Foreign function interface)は読んで字のごとく、ほかのプログラミング言語で書かれたプログラムを呼び出すための機構のことです。
  - この記事に限ってはプラットフォームの`C ABI`を利用して`.dll`や`.so`ファイルから関数を呼び出すなどすることを指すとします。
- [WASM]はアーキテクチャ、OS非依存なバイナリインストラクションフォーマットやその他もろもろのことであり、スタックベースのVMの上で動作します。
  - [Rust]や[Go]など、複数の言語からビルド可能です。
  - VMが[Cranelift]の`JIT compilation`などで実行コードを生成することでNear-native speedで動作するとされています。
  - もとはWebブラウザ上で動作するバイナリフォーマットを目指していたのでWeb Assemblyという名前になっています。
    - `Chrome`や`Firefox`など主要なブラウザはWASMを実行可能です([参考](https://www.publickey1.jp/blog/23/firefox_120webassemblywasmgcchrome.html))。
    - ブラウザと無関係なVMの実装もいくつも出ています([wasmer], [wasmtime], [wazero]など)。

これらにも当然一長一短があり、

- サブプロセス
  - 長所: ビルドプロセスが単純。
  - 短所: プロセス起動のオーバーヘッドが高い。通信方法の整備が必要。
- [ffi]
  - 長所: 比較的高速。小さな粒度での呼び出しが可能。
  - 短所: [cross compilation](https://en.wikipedia.org/wiki/Cross_compiler)が困難になることがある。
- [WASM]
  - 長所: 安全。プログラムがホストシステムへできることを完全にコントロールできる
  - 短所: 既に存在するプログラムのすべてをビルドしてそのまま動かせるわけではない。[WASI Preview 2](https://github.com/WebAssembly/WASI/pull/577#issuecomment-1910711171)は仕様が正式化したばかりで実相が追い付いているとは限らない。

などあります。

書き出せばこれ以上に長所短所があると思いますので、最適な方法を決定する際にはもちろん読者自身の調査と判断によって行ってください。

ほかのプログラムをcliのコマンドとして直接呼び出す例として典型的なものとしてここで`git`コマンドを上げます

例えば、

- vscodeのgit extension
  - https://github.com/microsoft/vscode/blob/8edcc2d9d83926ec73932b05e13eeafbe330ca32/extensions/git/src/git.ts#L668
  - https://github.com/microsoft/vscode/blob/8edcc2d9d83926ec73932b05e13eeafbe330ca32/extensions/git/src/git.ts#L583
  - https://github.com/microsoft/vscode/blob/8edcc2d9d83926ec73932b05e13eeafbe330ca32/extensions/git/src/git.ts#L561
  - https://github.com/microsoft/vscode/blob/8edcc2d9d83926ec73932b05e13eeafbe330ca32/extensions/git/src/git.ts#L2032
- go get
  - https://github.com/golang/go/blob/e17e5308fd5a26da5702d16cc837ee77cdb30ab6/src/cmd/go/internal/modfetch/codehost/git.go#L249

`git`コマンドがそのまま呼び出されるのは環境のセットアップ、例えば`pass`プログラムや`wincred`などのとの連携がそのまま持ち込めるので都合がいいからだと思われます。

[Rust]: https://www.rust-lang.org/
[Go]: https://go.dev/
[ffi]: https://en.wikipedia.org/wiki/Foreign_function_interface
[wasm]: https://webassembly.org/
[Cranelift]: https://cranelift.dev/
[wasmer]: https://wasmer.io/
[wasmtime]: https://wasmtime.dev/
[wazero]: https://wazero.io/
