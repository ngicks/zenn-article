---
title: "Goで開発して3年のプラクティスまとめ"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

# Goで開発して3年たったのでプラクティスをまとめる

筆者はGoを触りだして3年ぐらいたったので、触りだしてからもっと早く知りたかったこととかをまとめて置くことで筆者の知識の整理を行います。

なるだけワンストップでたくさんの話題を扱うようにして、なんとなくイディマティックなコードを書けるようにするのが目的です。

誰かの役に立つことを願います。

# 筆者のバックグラウンド

筆者のバックグラウンドを書くことで、アンビエントに存在する前提知識を掲示します。

- 学生時代は[C++]を使ってセンサー値を読み込んで計算を行うプログラムを記述していた。
  - つらかった
  - Windows 7
  - Visual Studio Community
- 入社してからメイン[Node.js]、ほんのちょっぴり[python]で開発を行う
- 独学で[The Rust Programming Language 日本語]を(確か2018年エディションを)読了、PDFiumやOpenSSLのbindingをrustで書いたりして使ってました
- その後[Go]を使い始める。多分一番慣れてる言語です。

つまり筆者は[Go]を使い始める前に以下をなんとなく理解しています

- [C++]・・・何年も前なので新しい文法は使っていない
- [Node.js]
- [TypeScript]
- [Rust]
- [Linux]上でファイルを読み書きしたりデータストレージとやり取りするときに起きる諸般の問題
  - [open(2)](https://man7.org/linux/man-pages/man2/open.2.html)を`O_TRUNC`付きで呼んでから[write(2)](https://man7.org/linux/man-pages/man2/write.2.html)がリターンするまでの間にファイルが0バイトの状態が観測できる、とか

筆者は[linux]が動作する小さ目のデバイスで動くプログラムしか書かないので、クラウドとかそういったものが視点に入っていません。
結構特殊な視点で書かれているかもしれないです。

# 対象読者

- メインは会社の同僚です。
- いままで[Go]を使ってこなかった人
- ある程度コンピュータとネットワークとプログラムを理解している人

# 基本

## A Tour of Go

https://go.dev/tour/welcome/

`Go`の基本的なトピックはここに書いてあります。
インタラクティブなコードスニペットと簡単なエクササイズがあり、これさえこなせばとりあえず開発は始められます。

## Go by Example

https://gobyexample.com/

コード例とともに解説がされます。手が空いたら少しずつ読むのがいいのではないでしょうか。

## 公式読み物系

TODO: 読む

公式が出している読み物集。全部英語です。
初めから全部読む必要はないと思いますが折を見て読んでいくのがよいと思います。

- The Go Language Specification: https://go.dev/ref/spec
  - 言語仕様ですが割と短めなのでそのうち読んでおいたほうがよいでしょう。
- How to Write Go Code: https://go.dev/doc/cod
  - コードの書き方以外も含めた基本的なトピック
- Effective Go: https://go.dev/doc/effective_go
  - Goのイディオム集
- Go Wiki: Go Code Review Comments: https://go.dev/wiki/CodeReviewComments#mixed-caps
  - よくされるCode review comment集らしいです

## Std library

TODO: いった手前自分でも読む

https://pkg.go.dev/std

standard libraryです。

HTTP(1.1 or 2)で動作するサーバープログラムを作るのに大体必要な機能がそろっています。
できれば開発に着手する前にすべてのインターフェイスとdoc commentを読んでおくがよいと思います。

## golang/example

TODO: 全体をざっと眺める。

https://github.com/golang/example

公式でメンテされているexample集。新しめな話題は取り扱っていないこともある。

## 外部リソース（未読）

TODO: さっと目を通しておこう。100 Go mistakes ~は面白そうなので読んでおきたい。

- https://tour.ardanlabs.com/tour/eng/list
- 100 Go Mistakes and How to Avoid Them
  Book by Teiva Harsanyi

[C++]: https://en.wikipedia.org/wiki/C%2B%2B
[Node.js]: https://nodejs.org/en
[TypeScript]: https://www.typescriptlang.org/
[python]: https://www.python.org/
[Rust]: https://www.rust-lang.org
[The Rust Programming Language 日本語]: https://doc.rust-jp.rs/book-ja/
