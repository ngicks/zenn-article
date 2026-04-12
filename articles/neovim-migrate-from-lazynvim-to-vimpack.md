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

### vim.pack

#### vim.packとは

https://neovim.io/doc/user/pack/#vim.pack

[neovim 0.12]から追加された機能で、外部のpluginのインストールや有効化などの管理を行えます。

[vim.pack]はユーザーから与えられたURLを用いてgit repositoryを特定パスにcloneし、それらをsourceすることで動作します。

以前からvimに[:packadd]という機能があり、[vim.pack]これをラップすることで実装しています。ほぼ同じことを行う[mini.deps]の再実装です。

> Install, update, and delete external plugins. WARNING: It is still considered experimental, yet should be stable enough for daily use.

とあるようにまだexperimentalなのでいろいろな機能が追加されるようですが普通に使う分には確かに困りません。

#### ワークフロー

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

#### 関連issue

`vim.pack, start/opt packages, 'packpath'`関連issueは`packages` labelがつけられるようです。今後どうなっていくかなどは他のissueも見て見るとよいでしょう。

https://github.com/neovim/neovim/issues?q=state%3Aopen%20label%3Apackages

### 移行すべき？

- [lazy.nvim]の高級な`setup`自動呼出しやlazy-loadingの挙動は全くないため、それが必要である場合は移行できません。
- [mini.deps]の[再実装であることは明記](https://github.com/neovim/neovim/pull/34009)されてるので`mini.deps`を利用している方は移行できるかも

筆者が移行したのには特に深い理由はなく、単にロード順序の問題でlazy-loadingできないプラグインがあったのでいっそlazy-loading全般をやめるついでに移行もしてみた感じです。
移行前後で起動時間にほぼ差がなく、特に目に見えて困ることはありませんでした。公式にこういった機能がメンテされるのは非常に助かりますね。

### lazy.nvimにあってvim.packにないもの

筆者は[lazy.nvim]を使用してpluginの管理をしていました。おそらく多くの人もそうだったのではないかと思います。
`lazy.nvim`ではできたけど`vim.pack`単体ではできないところから移行すべきなのかがわかるのではないかと思います。

項目の名前は[lazy.nvimのspec](https://lazy.folke.io/spec)に基づいています。

|                       | `lazy.nvim` | `vim.pack`                                     |
| :-------------------- | :---------- | :--------------------------------------------- |
| tableでのpluginの管理 | ⭕          | ⭕                                             |
| 管理TUI               | ⭕          | `vim.pack.update`時に確認buffer+lspはある      |
| `setup`自動呼出し     | ⭕          |                                                |
| `init`                | ⭕          |                                                |
| `build`               | ⭕          | `PackChangedPre`/`PackChanged`イベントでcb実行 |
| `dependencies`        | ⭕          |                                                |

- 管理TUI:
  - `:Lazy`で管理TUIがが立ち上がります。
  - `vim.pack.update`時にアップデートの確認バッファが立ち上がりますが、これには専用のLSPが存在しています。
- setup自動呼出し:
  - `require("mod").setup({})`を自動的に呼び出す機能
  - プラグインが設定可能な項目を持っていたり、初期化処理を必要とする場合`setup`をトップレベルの関数にするのが慣習のようです。
  - lazy-loadingは`setup`の呼び出しの遅延でもあるので、lazy-loadingがある=`setup`自動呼出しもあるということになるっぽい
  - `setup()`は設定を行うので、設定の管理も同時に組み込まれることになります。
- `init`
  - 条件に関係なく、スタートアップ時に実行されるcallback
  - `vim.g`とか`vim.o`の設定を行う
- `build`
  - インストール時/バージョン変更時に事項されるcallback
  - [PackChangedPre/PackChanged](https://neovim.io/doc/user/pack/#PackChangedPre)イベントをフックすれば似たようなことはできます。

LSPは以下のような感じ。hoverでdiffを見るとかできますね。

![confirmation-buffer-lsp-hover](/images/neovim-migrate-from-lazynvim-to-vimpack/confirmation-buffer-lsp.png)

```
# Error ────────────────────────────────────────────────────────────────────────

## nui.nvim

 ...vim-unwrapped-0.12.1/share/nvim/runtime/lua/vim/pack.lua:245: fatal: unable to access 'https://github.com/MunifTanjim/nui.nvim/': Could not resolve host: github.com


## diffview.nvim

 ...vim-unwrapped-0.12.1/share/nvim/runtime/lua/vim/pack.lua:245: fatal: unable to access 'https://github.com/sindrets/diffview.nvim/': Could not resolve host: github.com


## nvim-bqf

 ...vim-unwrapped-0.12.1/share/nvim/runtime/lua/vim/pack.lua:245: fatal: unable to access 'https://github.com/kevinhwang91/nvim-bqf/': Could not resolve host: github.com

```

[neovim 0.12]: https://neovim.io/doc/user/news-0.12/
[vim.pack]: https://neovim.io/doc/user/pack/#vim.pack
[:packadd]: https://vimhelp.org/repeat.txt.html#%3Apackadd
[lazy.nvim]: https://github.com/folke/lazy.nvim
[mini.deps]: https://github.com/nvim-mini/mini.deps
