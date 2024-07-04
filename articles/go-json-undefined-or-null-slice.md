---
title: "GoのT | null | undefinedは[]Option[T]でよかった"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## GoのT | null | undefinedは[]Tでよかった

- [github.com/oapi-codegen/nullable](https://github.com/oapi-codegen/nullable)が`map[bool]T`をベースに`encoding/json`にスキップされることが可能な`T | null | undefined`な型を定義していた
  - ([以前の記事]時点では全然思いついてなかったので悔しい)
- `type Option[T any] struct{valid bool; v T}`を定義して`type undefinedable []Option[T]`としたほうがパフォーマンスが出るんじゃないかと思って試した。
  - ほんのちょっぴりパフォーマンスがよかった。

## はじめに

TODO: 概要

置き換えを目指すのでコンパクト版の前回記事になることを説明。必要なら飛ばせと書く。

## Overview

TODO: 何話すか

## 対象読者

TODO: いつもの

## おさらい: 時たま困る「データがない状態」の扱い

### Goとzero valueとJSON

### T | null | undefined、あるいはpartial JSONのユースケース

TODO: elasticsearchのpartial updateの話

### 普通の方法: map[string]anyを介したvalidation

TOOD: jsonschema validator libが`map[string]any`を介してvalidationを行うのを例示。

### Partial JSONの受け側にはなれる

## 課題: encoding/jsonはstructをskipしない

## 解決法: map[T]U, []Tはomitemptyでskip可能

### map[bool]Tを使う実装: [github.com/oapi-codegen/nullable](https://github.com/oapi-codegen/nullable)

### []Option[T]も使える

## 実装

### Option[T]

### Und[T] []Option[T]

## ベンチマーク

## おわりに
