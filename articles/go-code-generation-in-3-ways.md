---
title: "Goのcode generation: text/template,jennifer,ast(dst)-rewrite"
emoji: "🧰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## Goのcode generationについてまとめる

Goは、RustやCにあるようなマクロを言語機能として持っていないため、たびたびcode generationによって。。。

TODO: もうちょい膨らませる。

## やること

## 前提知識

## 環境

## 3つの(おそらく)代表的な方法

## text/template

## github.com/dave/jennifer

## ast(dst)-rewrite

１からastをくみ上げることでコードを生成することもできますが、それをやるならば上記の`text/template`か`github.com/dave/jennifer`を用いるほうが楽なはずなので、ここでは深く紹介しません。
その代わり、astや型情報

### astutilを使った書き換え

### dstを使った書き換え

https://stackoverflow.com/questions/31628613/comments-out-of-order-after-adding-item-to-go-ast

## まとめ

## おわりに
