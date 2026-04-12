---
title: "[neovim]lazy.nvimからvim.packへの移行"
emoji: "🚚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["neovim"]
published: false
---

## \[neovim\]vim.packへの移行

筆者は[lazy.nvim]によってpluginを管理していましたが、
2026-03-30にリリースされた[neovim 0.12]から追加された[vim.pack]を使った管理に移行したのでその感想とかを書きます。

リリースされたら即行で報告がブワーッと上がるかと思いましたがredditでいくつか見たぐらい(単に観測範囲の問題かもしれません)
zennでは[この記事](https://zenn.dev/knsh14/articles/nvim-pack-2025-07-25)を見た以外にはなかったのでせっかくなので賑わしておきます。

## vim.pack

### vim.packとは

https://neovim.io/doc/user/pack/#vim.pack

[neovim 0.12]から追加された機能で、外部のpluginのインストールや有効化などの管理を行えます。

[vim.pack]はユーザーから与えられたURLを用いてgit repositoryを特定パスにcloneし、それらをsourceすることで動作します。

以前からvimに[:packadd]という機能があり、[vim.pack]これをラップすることで実装しています。ほぼ同じことを行う[mini.deps]の再実装です。

> Install, update, and delete external plugins. WARNING: It is still considered experimental, yet should be stable enough for daily use.

とあるようにまだexperimentalなのでいろいろな機能が追加されるようですが普通に使う分には確かに困りません。

### ワークフロー

`init.lua`、もしくはインタラクティブに`vim.pack.add`を実行し、`git clone` + `:packadd`を行います

```lua
-- quoted from https://github.com/neovim/neovim/blob/6f015cdcdf0b617c9b716e833823498ce7c001c8/runtime/lua/vim/pack.lua#L28-L60
    vim.pack.add({
      -- Install "plugin1" and use default branch (usually `main` or `master`)
      'https://github.com/user/plugin1',

      -- Same as above, but using a table (allows setting other options)
      { src = 'https://github.com/user/plugin1' },

      -- Specify plugin's name (here the plugin will be called "plugin2"
      -- instead of "generic-name")
      { src = 'https://github.com/user/generic-name', name = 'plugin2' },

      -- Specify version to follow during install and update
      {
        src = 'https://github.com/user/plugin3',
        -- Version constraint, see |vim.version.range()|
        version = vim.version.range('1.0'),
      },
      {
        src = 'https://github.com/user/plugin4',
        -- Git branch, tag, or commit hash
        version = 'main',
      },
    })
    -- Plugin's code can be used directly after `add()`
    plugin1 = require('plugin1')
```

- データは`${XDG_DATA_HOME:-$HOME/.local/share}/nvim/site/pack/core/opt`以下にcloneされます。
- 一般的な`git clone`が使われるので`git`の設定をちゃんとやればprivate repositoryからでもcloneできるはずです。
- `${XDG_CONFIG_HOME:-$HOME/.config}/nvim/nvim-pack-lock.json`にlock fileが作成されます。

`vim.pack.update`でアップデートを行います。

- `vim.pack.update()`ですべてアップデート
- `vim.pack.update(nil, {target = 'lockfile'})`で`lockfile`と同じ状態になるようにアップデート
- `vim.pack.add`/`vim.pack.del`を経由せずにディスク上のプラグイン状態を変更した場合`vim.pack.update(nil, {offline = true})`で追従させます。
  - ただし場合によりでこれもうまく動きません([#38931](https://github.com/neovim/neovim/pull/38931))。
  - `:checkhealth vim.pack`で異常状態の検知が可能です。

`vim.pack.del()`でpluginを削除できます。
私は`vim.pack.del`を使わないのでこれの細かい挙動はよくわかっていません。
筆者環境は`nix`の`home-manager`で管理されていますので`${XDG_CONFIG_HOME:-$HOME/.config}/`以下がread-onlyとなります。
インタラクティブにプラグイン状態を操作するには`XDG_CONFIG_HOME`の変更が必要でそこそこ面倒なので普段はやらないわけです。

### 関連issue

`vim.pack, start/opt packages, 'packpath'`関連issueは`packages` labelがつけられるようです。今後どうなっていくかなどは他のissueも見て見るとよいでしょう。

https://github.com/neovim/neovim/issues?q=state%3Aopen%20label%3Apackages

## 移行すべき？

- [lazy.nvim]の高級な`setup`自動呼出しやlazy-loadingの挙動は全くないため、それが必要である場合は移行できません。
- [mini.deps]の[再実装であることは明記](https://github.com/neovim/neovim/pull/34009)されてるので`mini.deps`を利用している方は移行できるかも

筆者が移行したのには特に深い理由はなく、単にロード順序の問題でlazy-loadingできないプラグインがあったのでいっそlazy-loading全般をやめるついでに移行もしてみた感じです。
移行前後で起動時間の差があるのか正直よくわかりません。起動時間は割と時々でブレが大きいので100ms～500msで普通にぶれるのでそっちのブレ幅のほうが大きくでよくわからなかったです。

移行は[codex]を使って`gpt-5.4 medium`を叩いて大まかにできたので特に困らなかったです。
公式にこういった機能がメンテされるのは非常に助かりますね。

## lazy.nvimにあってvim.packにないもの

筆者は[lazy.nvim]を使用してpluginの管理をしていました。おそらく多くの人もそうだったのではないかと思います。
`lazy.nvim`ではできたけど`vim.pack`単体ではできないところから移行すべきなのかがわかるのではないかと思います。

項目の名前は[lazy.nvimのspec](https://lazy.folke.io/spec)に基づいています。

|                       | `lazy.nvim` | `vim.pack`                                                  |
| :-------------------- | :---------- | :---------------------------------------------------------- |
| tableでのpluginの管理 | ⭕          | ⭕                                                          |
| 管理TUI               | ⭕          | `vim.pack.update`時にconfirmation buffer+lspはある          |
| lazy-loading          | ⭕          | `vim.pack.add(..., {load = false})`し、手動で`:packadd`する |
| `setup`自動呼出し     | ⭕          |                                                             |
| `init`                | ⭕          |                                                             |
| `build`               | ⭕          | `PackChangedPre`/`PackChanged`イベントでcb実行              |
| `dependencies`        | ⭕          |                                                             |

- 管理TUI:
  - `:Lazy`で管理TUIがが立ち上がります。
  - `vim.pack.update`時にアップデートのconfirmation bufferが立ち上がりますが、これには専用のLSPが存在しています。
- lazy-loading:
  - 各種イベント、コマンドスタブによる遅延ロードなどを行う機能
    - コマンドスタブによる遅延ロードは`lazy.nvim`側が`:CommandName`で実行できるコマンドを登録して起き、実行をフックして対応するプラグインの`setup`を実行するものです。
  - `vim.pack`で再現する場合は`{load = false}`で`:packadd!`相当の挙動にしたのち、遅延で`:packadd`を呼び出すことで再現できます。
- setup自動呼出し:
  - `require("mod").setup({})`を自動的に呼び出す機能
  - プラグインが設定可能な項目を持っていたり、初期化処理を必要とする場合`setup`をトップレベルの関数にするのが慣習のようです。
  - lazy-loadingは`setup`の呼び出しの遅延でもあるので、lazy-loadingがある=`setup`自動呼出しもあるということになるっぽい
  - `setup()`は設定を行うので、設定の管理も同時に組み込まれることになります。
- `init`
  - 条件に関係なく、スタートアップ時に実行されるcallback
  - `vim.g`とか`vim.o`の設定を行う用に使います。
- `build`
  - インストール時/バージョン変更時に事項されるcallback
  - [PackChangedPre/PackChanged](https://neovim.io/doc/user/pack/#PackChangedPre)イベントをフックすれば似たようなことはできます。
- `dependencies`
  - 依存先を指定できます
  - 依存先は依存もとよりロード順序が先に置かれるのだと思われます。

などなど、これらを使っている場合は移行しないで`lazy.nvim`を使い続けるなり、自前で実装するなり必要です。

`vim.pack.update`のconfirmation bufferのLSPは以下のような感じ。hoverでdiffを見るとかできますね。

![confirmation-buffer-lsp-hover](/images/neovim-migrate-from-lazynvim-to-vimpack/confirmation-buffer-lsp.png)

## 移行のためにラッパーを作ろう

上記にあげた`setup`自動呼出しなんかはなくなると困るので移行するなら追加の実装が必要です。
ということで一通りやっておきました。

https://github.com/ngicks/dotfiles/tree/abe0ab9ed80ae49fc1b287a3d52e0475b6361d83/config/nvim/lua/ngpack

[codex]経由で`gpt5.4 medium`をたたいて書いたのちオーガニックコーディングで内部構造をだいぶ変えたりしています。
なのでコードの詳細には触れずにラッパー作って気づいたポイントについて書き記します。

基本機能は

- `vim.pack.add`を1度しか呼び出さないようにラップ
- setup自動呼出し
  - それに伴って`lazy.nvim`の`opts` / `config`の再現
- 簡易lazy-loading: 以下の3つplugin specの`phase`というキーで指定させ、挙動を変更
  - `"core"`: non-lazy
  - `"ui"`: `vim.schedule`でsetup実行をいったん遅延するのみ。
  - `"lazy"`: `load`関数を明示的に呼び出してsetupを実行するもの
  - 各種イベント/コマンドスタブは設定するのが大変だからドロップ
- `build`再現
  - `PackChangedPre`/`PackChanged`イベントでビルドを行えるように

行数はこんな感じ

```
❯ ls -1 ./config/nvim/lua/ngpack/ | xargs -I{} wc -l ./config/nvim/lua/ngpack/{}
386 ./config/nvim/lua/ngpack/init.lua
257 ./config/nvim/lua/ngpack/lock.lua
24 ./config/nvim/lua/ngpack/types.lua
83 ./config/nvim/lua/ngpack/util.lua
```

```
❯ 386+257+24+83
750
❯ 386+24+83
493
```

`lock.lua`はおまけだから抜くとしても大体500行あります。と考えると実装しすぎなので`lazy.nvim`のままでよくないかという気もしますね。
もちろん今後の`vim.pack`の発達によって行数が減ったり、逆に機能の追加でほしい機能が増えて行数が増えるとか、そういうこともあり得ます。

### 基本構造

tableで管理する`Plain`版ともろもろのmethodがついてる版で分けるようにしました。

https://github.com/ngicks/dotfiles/blob/abe0ab9ed80ae49fc1b287a3d52e0475b6361d83/config/nvim/lua/ngpack/types.lua#L1-L24

https://github.com/ngicks/dotfiles/blob/abe0ab9ed80ae49fc1b287a3d52e0475b6361d83/config/nvim/lua/ngpack/init.lua#L26-L40

plain lua tableを列挙することで管理、適当な関数で`NgPackSpecPlain[]` -> `NgPackSpec[]`に変換し、高級なメソッドを使っていろいろ機能の実現をするようにしています。
methodがあるといろいろ便利ですが、tableでplugin listを管理するところでいちいち`NgPackSpec:new`を呼び出してらんないですからね。

tableでプラグインを管理します。

https://github.com/ngicks/dotfiles/blob/abe0ab9ed80ae49fc1b287a3d52e0475b6361d83/config/nvim/lua/ngcfg/plugins/list.lua

`lazy.nvim`のplugin specをそのまま利用する方針にしています。とはいえ筆者は限られた機能しか使っていなません。
以下の定義を再利用しています。

- `opts`
- `config`
- `build`
- `init`
- `dependencies`

ただしそのままとはいかないので以下のように変更しています。

- `build` -> `pack_changed_pre`/`pack_changed`に分割し
- `dependencies` -> `dep`に変更し、単にtopological sortのヒントにのみ利用(未実装)

https://github.com/ngicks/dotfiles/blob/abe0ab9ed80ae49fc1b287a3d52e0475b6361d83/config/nvim/lua/ngpack/init.lua#L310-L352

メインの処理は大体以下の流れです。

- `Plain`版spec -> methodあり版specに変換
- `init`実行
- `PackChangedPre`/`PackChanged`コールバック登録
- `vim.pack.add`実行
- `"core"`(non-lazy)扱いされたspecの各種`setup`実行
- `vim.schedule`で`"ui"`(かんたんlazy)のspecの`setup`実行をスケジュール

### メインパッケージ名推定

`setup`自動実行のために、`require("mod").setup`の`"mod"`の名前を推定する必要があります。

これが案外難しいです。

- `foo-bar.nvim`は基本的に`foo-bar`
- `nvim-foo`は`foo`なときと`nvim-foo`のときがあってややこしい
  - [nvim-tree/nvim-web-devicons](https://github.com/nvim-tree/nvim-web-devicons)は`"nvim-web-devicons"`
  - [rcarriga/nvim-notify](https://github.com/rcarriga/nvim-notify)は`"notify"`

理屈上各プラグインのディレクトリに入って`./plugin`か`./lua`以下を読めばいいんですが、ここでioしたくはありません。

余計なことはせずに`.nvim`をドロップするだけにして、それ以外のケースに備えてspecに`main`を追加して指定できるようにして終わりにしておきました

https://github.com/ngicks/dotfiles/blob/abe0ab9ed80ae49fc1b287a3d52e0475b6361d83/config/nvim/lua/ngpack/init.lua#L112-L122

### setup呼び出し時、callable tableに注意

`lazy.nvim`の`opts`, `config`を再現して、結果を引数に`setup`を呼び出します。

https://github.com/ngicks/dotfiles/blob/abe0ab9ed80ae49fc1b287a3d52e0475b6361d83/config/nvim/lua/ngpack/init.lua#L159-L176

https://github.com/ngicks/dotfiles/blob/abe0ab9ed80ae49fc1b287a3d52e0475b6361d83/config/nvim/lua/ngpack/init.lua#L145-L157

ポイントは`setup`の判定です。

https://github.com/ngicks/dotfiles/blob/abe0ab9ed80ae49fc1b287a3d52e0475b6361d83/config/nvim/lua/ngpack/util.lua#L4-L19

実装するまで知らなかったんですが、`setup`がcallableな`table`なことがあるんですね。

具体的にいうと[hrsh7th/nvim-cmp](https://github.com/hrsh7th/nvim-cmp)がcallableな`table`でした。

### desync検知

lockfileと実際のrepositoryがあるので、その二つの状態がdesyncすることは普通に起こります。

`:checkhealth vim.pack`で異常状態の検知ができます。

`vim.pack.update(nil, {offline = true})`で基本的には異常状態を解決できます。
ただし[#38931](https://github.com/neovim/neovim/pull/38931)である通り、lockfileは古い内容だが、すでにrepositoryの状態が`version`で指定される最新状態であるとき、アップデートなしとみなされて、このdesync状態が解決できません。
`XDG_CONFIG_HOME`がread-onlyとなるようなセットアップ・・・つまり筆者のように`nix`の`home-manager`を使っているとこの状態が容易に起きます。

基本はlockファイルの当該エントリを消して`vim.pack.update(nil, {target = 'lockfile'})`を行うのが筋かと思います。

スクリプトの中で一気にアップデートしてるとエラーしてても気づかないことがあります。起動時に検知できたほうがいいのでそういう機能をつけました。

https://github.com/ngicks/dotfiles/blob/abe0ab9ed80ae49fc1b287a3d52e0475b6361d83/config/nvim/lua/ngpack/lock.lua

`codex`叩いたあとそんなにレビューしてないのでちゃんと動いてるのかは不明。

## そのほかの発見など

### async/awaitの使用

内部でasync/awaitが使われています。例えばここ。

https://github.com/neovim/neovim/blob/fa22a78d2a5524839437ad04a4e3ba6ff6633de6/runtime/lua/vim/pack.lua#L232-L254

そのおかげで大量に`git`コマンドを同時使用するせいか、DNSのところでエラーがしょっちゅうおきます。

内部のDNSスタブのキャッシングが機能していないのかも・・・

```
# Error ────────────────────────────────────────────────────────────────────────

## nui.nvim

 ...vim-unwrapped-0.12.1/share/nvim/runtime/lua/vim/pack.lua:245: fatal: unable to access 'https://github.com/MunifTanjim/nui.nvim/': Could not resolve host: github.com


## diffview.nvim

 ...vim-unwrapped-0.12.1/share/nvim/runtime/lua/vim/pack.lua:245: fatal: unable to access 'https://github.com/sindrets/diffview.nvim/': Could not resolve host: github.com


## nvim-bqf

 ...vim-unwrapped-0.12.1/share/nvim/runtime/lua/vim/pack.lua:245: fatal: unable to access 'https://github.com/kevinhwang91/nvim-bqf/': Could not resolve host: github.com
```

## おわりに

標準機能でこういったものがサポートされるのは非常に助かりますね。

記事のとおり、lazy-loadingにまつわるもろもろの機能が実装されていないので[lazy.nvim]や[mini.deps]などから移行するかはどのぐらいそれらの機能を使っているかで判断したらよいのではないかと思います。

[neovim 0.12]: https://neovim.io/doc/user/news-0.12/
[vim.pack]: https://neovim.io/doc/user/pack/#vim.pack
[:packadd]: https://vimhelp.org/repeat.txt.html#%3Apackadd
[lazy.nvim]: https://github.com/folke/lazy.nvim
[mini.deps]: https://github.com/nvim-mini/mini.deps
[codex]: https://openai.com/ja-JP/codex/
