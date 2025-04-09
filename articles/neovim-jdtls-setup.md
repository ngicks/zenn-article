---
title: "[neovim]jdtls(javaの言語サーバー)の設定してelasticsearchのソースを読めるようにしたメモ"
emoji: "👓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["neovim", "java"]
published: false
---

## [neovim]jdtls(javaの言語サーバー)の設定してelasticsearchのソースを読めるようにしたメモ

なんか変で課題はあるんだけど読めはしてる・・・と思う状態にまでなった

できるようになったこと

- elasticsearch 8.17.4のソースを読んで定義ジャンプしたり実装検索をしたりできるようなった
  - 本当は7.x.y系を読みたかったんだけど、そちらではエラーを解決できず

未解決なこと

- elasticsearch 7.x.y系でjdtlsを動作させること
- 定義へのジャンプや実装の検索ができるけど「XXX connnat be resolved」のdiagnoseが出続ける
  - 少々鬱陶しいですが、syntax errorは実際にはないという前提に基づくと無視できるのでよしとします。

似た苦しみをする人が減るといいと思って書いてます。

## 筆者のバックグラウンド

- javaの開発はほとんどしたことありません。
  - 学生時代にSwing JAVAでUIを作ったことがあります。UI学の講義です
  - トラブル解決のためにライブラリをちょっと引っ張ってきて処理を一部分取り出したコマンドを作ったことぐらいしかありません。
  - したがって解決できていない不具合の中には筆者のjavaの知識不足によることが多くあるかもしれません。
- 筆者はほとんどneovimで開発したことがありません
  - 最近環境をセットアップしてます
  - 何気にneovim 0.11にアップデートしていますが、`nvim-lspconfig`を使った0.10までの設定のままです。
  - したがって以下略

## 環境

windows11上で動くwsl2上で動くUbuntu24.04インスタンスです。

```
$ uname -srvmpi
Linux 5.15.167.4-microsoft-standard-WSL2 #1 SMP Tue Nov 5 00:21:55 UTC 2024 x86_64 x86_64 x86_64
```

`neovim`は0.11ですが設定周りは0.10のときから刷新されていません。

```
$ nvim --version
NVIM v0.11.0
Build type: Release
LuaJIT 2.1.1741730670
Run "nvim -V1 -v" for more info
```

## Elasticsearch互換性テーブルの確認

https://www.elastic.co/support/matrix

| Elasticsearch | OpenJDK        |
| ------------- | -------------- |
| 7.3.x         | 1.8.0, 11 ~ 12 |
| 7.17.x        | 1.8.0, 17 ~ 22 |
| 8.17.x        | 17, 21, 23     |

実験のためだけで結構幅広いjdkのバージョンが必要そうです。

9.x.y系は表に乗ってないしRCにもなってないβフェーズっぽいのでとりあえず忘れます。

## 落とせるOpenJDKを全部落とす

`download.java.net`から落とせるjdkはとりあえず全部落とします。

https://jdk.java.net/archive/

ここにリストされる9～23と現行の24(2025-04-08時点)をひたすら落とします！

というのを環境毎にやるのは面倒なのでmy dotfilesで簡単なインストーラーを用意しておきました。(`deno`で動作する`typescript`です)

https://github.com/ngicks/dotfiles/blob/147aa07349e519d4d0effc2ab0c8031e576ec7e0/src/jdk/install.ts

リストされたurlからファイルをダウンロードして`~/.local/openjdk`いかに展開するだけの簡単なものです。

- windowsもwin10のどこかのバージョンから`bsdtar`が内蔵されるのでこれでそのまま動くはずです
- macは手元にないので確かめられないですが`tar`コマンドは普通入っているものと想定しています。

なにげに`osx`, `macos`で呼び方があるバージョンから変わっていたり、バイナリ配布の`aarch64`(=`arm64`)対応が割と最近のバージョンからだったりと、ダウンロードするだけにしては多めの条件分岐が発生しています。
ちなみに`linux/amd64`以外では動作させたことがないのでほかの環境で動くは正直わかりません。

落としたjdkは下記のファイルで`JAVA${VERSION}_HOME=path/to/jdk`な環境変数をそれぞれ定義するdotenvファイルを作り、

https://github.com/ngicks/dotfiles/blob/147aa07349e519d4d0effc2ab0c8031e576ec7e0/src/jdk/dotenv.ts

それを`.bashrc`などの中で

```shell
set -a
. jdk.env
set +a
```

などとすることでsourceします。

とりあえず`JAVA_HOME`と`PATH`は`JAVA24_HOME`においておきます。

```shell
export JAVA_HOME=$JAVA24_HOME
export PATH=$JAVA_HOME/bin:$PATH
```

ついでに`GRADLE_USER_HOME`を`.cache`以下になるように変更しておきます。
デフォルトでは`~/.gradle`になります。あんまり`$HOME`以下にフォルダが増えると収拾つかなくなるのでこれはcacheだと決めつけて`.cache`に入れてしまいましょう。

```shell
export GRADLE_USER_HOME=$HOME/.cache/gradle
```

時と場合によって[Eclipse TemurinのJDK](https://adoptium.net/temurin/releases/)のほうがおすすめされていることがありますがどっちがどう違うのか理解していません。
今後の課題です。

## Elasticsearchを./gradlewでビルドできるのを確認する

いきなり`jdtls`で動作させてトラブルが起きたとき切り分けられないので`./gradlew`で正常にビルドできることをまず確認します。

`7.x.y`系ではうまく動かなったので`8.17.4`で動作させることとなりました。

### Elaticsearch v7.3.1はjcenterが依存先に入っているのでビルドできない

古いelasticsearchの挙動が知りたかったのでv7.3.1でビルドできるとよいのでまずこのタグをチェックアウトして試します。

jcenterが依存先に入ってるので変更なしにはビルドできないですね。

```
$ JAVA_HOME=$HOME/.local/jdk-11.0.2 ./gradlew tasks
Starting a Gradle Daemon, 1 incompatible Daemon could not be reused, use --status for details
> Task :buildSrc:compileJava FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':buildSrc:compileJava'.
> Could not resolve all files for configuration ':buildSrc:compileClasspath'.
   > Could not find com.github.jengelman.gradle.plugins:shadow:4.0.3.
     Searched in the following locations:
       - https://jcenter.bintray.com/com/github/jengelman/gradle/plugins/shadow/4.0.3/shadow-4.0.3.pom
       - https://jcenter.bintray.com/com/github/jengelman/gradle/plugins/shadow/4.0.3/shadow-4.0.3.jar
     Required by:
         project :buildSrc
   > Could not find com.avast.gradle:gradle-docker-compose-plugin:0.8.12.
     Searched in the following locations:
       - https://jcenter.bintray.com/com/avast/gradle/gradle-docker-compose-plugin/0.8.12/gradle-docker-compose-plugin-0.8.12.pom
       - https://jcenter.bintray.com/com/avast/gradle/gradle-docker-compose-plugin/0.8.12/gradle-docker-compose-plugin-0.8.12.jar
     Required by:
         project :buildSrc

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 7s
```

参考: [5月1日のBintray、JCenter、GoCenter、ChartCenterのサービス終了について](https://jfrog.com/ja/blog/into-the-sunset-bintray-jcenter-gocenter-and-chartcenter/)

とりあえず7系の最新である`7.17.28`に変更することにします。ほかのバージョンではいろいろ修正されてて変わってはいるでしょうが、見たかった挙動はそこではないのでよいものとします。

### Elasticsearch v7.17.28はビルドできる

```
$ git describe --exact-match --tags
v7.17.28
```

`build`, `assemble`タスクはクロスコンピレーションに`docker`を使います。
環境を改めて必要性が減ったため筆者の環境には現在`docker for desktop`が導入されていませんのでそこでビルドが止まります。
とりあえず今の環境向けにのみビルドできればいいので`localDistro`を実行できることを確認します。

```
$ JAVA_HOME=$HOME/.local/jdk-17.0.2 ./gradlew localDistro
Starting a Gradle Daemon, 1 incompatible and 1 stopped Daemons could not be reused, use --status for details

> Task :build-conventions:compileJava
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
> Task :build-tools:compileJava
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
Note: Some input files use unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.

> Task :build-tools-internal:compileJava
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
Note: ~/gitrepo/github.com/elastic/elasticsearch/build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/snyk/SnykDependencyGraph.java uses unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.
=======================================
Elasticsearch Build Hamster says Hello!
  Gradle Version        : 8.10.2
  OS Info               : Linux 5.15.167.4-microsoft-standard-WSL2 (amd64)
  JDK Version           : 17.0.2+8-86 (Oracle)
  JAVA_HOME             : ~/.local/jdk-17.0.2
  Random Testing Seed   : 99D293BD13478CE5
  In FIPS 140 mode      : false
=======================================

> Task :distribution:tools:launchers:compileJava
Note: ~/gitrepo/github.com/elastic/elasticsearch/distribution/tools/launchers/src/main/java/org/elasticsearch/tools/launchers/MachineDependentHeap.java uses or overrides a deprecated API.
Note: Recompile with -Xlint:deprecation for details.

...中略...

> Task :localDistro
Elasticsearch distribution installed to ~/gitrepo/github.com/elastic/elasticsearch/build/distribution/local/elasticsearch-7.17.28-SNAPSHOT

BUILD SUCCESSFUL in 1m 56s
```

### ただしElasticsearch v7.17.28では./gradlew tasksは失敗する

`RUNTIME_JAVA_HOME`という環境変数が参照されるようになっているのでこれも設定必要です。

```
$ export JAVA_HOME=$JAVA17_HOME
$ export PATH=$JAVA_HOME/bin:$PATH
$ export RUNTIME_JAVA_HOME=$JAVA_HOME
$ ./gradlew tasks
Starting a Gradle Daemon (subsequent builds will be faster)

> Task :build-conventions:compileJava
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.

> Task :build-tools:compileJava
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
Note: Some input files use unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.

> Task :build-tools-internal:compileJava
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
Note: ~/gitrepo/github.com/elastic/elasticsearch/build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/snyk/SnykDependencyGraph.java uses unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.
=======================================
Elasticsearch Build Hamster says Hello!
  Gradle Version        : 8.10.2
  OS Info               : Linux 5.15.167.4-microsoft-standard-WSL2 (amd64)
  JDK Version           : 17.0.2+8-86 (Oracle)
  JAVA_HOME             : ~/.local/openjdk/jdk-17.0.2
  Random Testing Seed   : D8A7DA6B4F4E04E3
  In FIPS 140 mode      : false
=======================================

> Task :tasks FAILED

FAILURE: Build failed with an exception.

* Where:
Build file '~/gitrepo/github.com/elastic/elasticsearch/benchmarks/build.gradle' line: 65

* What went wrong:
Execution failed for task ':tasks'.
> Could not create task ':benchmarks:run'.
   > java.lang.UnsupportedOperationException (no error message)

* Try:
> Run with --stacktrace option to get the stack trace.
> Run with --info or --debug option to get more log output.
> Get more help at https://help.gradle.org.

BUILD FAILED in 28s
20 actionable tasks: 20 executed
```

中身を読んでないのでどうしてこのエラーが出るのかよくわかりません。

いつか解決するにしろ当面は安定して読みたいので8系の最新でtasksが動くのかとりあえず試します。

### Elasticsearch 8.17.4は正常に動作する

`8.17.4`では`tasks`もうまく動作します。前述の互換テーブルを見ても互換性が切られているのでだいぶいろいろ変わっているんでしょうね。

```
$ git describe --exact-match --tags
v8.17.4
```

```
$ export JAVA_HOME=$JAVA21_HOME
$ export PATH=$JAVA_HOME/bin:$PATH
$ export RUNTIME_JAVA_HOME=$JAVA_HOME
$ ./gradlew tasks

> Task :build-conventions:compileJava
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
Note: ~/gitrepo/github.com/elastic/elasticsearch/build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/LicensingPlugin.java uses unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.

> Task :build-tools:compileJava
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
Note: Some input files use unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.

> Task :build-tools-internal:compileJava
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
Note: Some input files use unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.
=======================================
Elasticsearch Build Hamster says Hello!
  Gradle Version        : 8.13
  OS Info               : Linux 5.15.167.4-microsoft-standard-WSL2 (amd64)
  Runtime JDK Version   : 21.0.2+13-58 (Oracle, 21.0.2+13-58)
  Runtime java.home     : ~/.local/openjdk/jdk-21.0.2
  Gradle JDK Version    : 21.0.6+7-LTS (Eclipse Temurin)
  Gradle java.home      : ~/.cache/gradle/jdks/eclipse_adoptium-21-amd64-linux.2
  Random Testing Seed   : A1B3015B45DE7320
  In FIPS 140 mode      : false
=======================================

> Task :tasks

------------------------------------------------------------
Tasks runnable from root project 'elasticsearch'
------------------------------------------------------------

Application tasks
-----------------
run - Runs this project as a JVM application

...省略...

yamlRestTest - Runs the REST tests against an external cluster
yamlRestTestV7CompatTest - Runs the REST tests against an external cluster

Rules
-----
Pattern: clean<TaskName>: Cleans the output files of a task.

To see all tasks and more detail, run gradlew tasks --all

To see more detail about a task, run gradlew help --task <task>

[Incubating] Problems report is available at: file://~/gitrepo/github.com/elastic/elasticsearch/build/reports/problems/problems-report.html

BUILD SUCCESSFUL in 32s
20 actionable tasks: 20 executed
```

すでに1度実行しているのでログ上表示されていませんが、`eclipse temurin`の`JDK`のダウンロードも行われていました。
環境差がへるのでこの取り組みはいいですね。

とりあえず`8.17.4`のほうが安定して動いてくれそうなのでこちらで言語サーバーが機能することを確かめます。

## jdtlsのセットアップ

ここからようやくneovimの設定周りに入ります。

以下のプラグインを使用します。ありがたや。

https://github.com/mfussenegger/nvim-jdtls

筆者のconfigが`NvChad`を下敷きにしてるので`lazy.nvim`で`"mfussenegger/nvim-jdtls"`を指定しておきます。

`mason`管理外で`JDK`, `jdtls`, `lombok`を使用しますのでまずダウンロードしたり、どこから設定をパクってきたり、普通にはやられない（？）設定の変更などを述べます。

### eclipse-jdtls/eclipse.jdt.lsのダウンロード

言語サーバーです。

ソースは下記でホストされています。

https://github.com/eclipse-jdtls/eclipse.jdt.ls

現時点(2025-04-09)で最新のmilestone buildの`1.46.1`をダウンロードします。

https://download.eclipse.org/jdtls/milestones/1.46.1/

```
$ mkdir -p ~/.local/jdtls/eclipse.jdt.ls/
$ pushd ~/.local/jdtls/eclipse.jdt.ls/
$ curl -LO https://www.eclipse.org/downloads/download.php?file=/jdtls/milestones/1.46.1/jdt-language-server-1.46.1-202504011455.tar.gz
$ tar -xf download.php
$ rm download.php
$ popd
```

redirectするせいなのか`-O`だと`download.php`っていう名前になっちゃうので`-o`で名前つけたほうがいいですね。
sha256の検証もしたほうがいいですが特にやってないです。署名のほうが欲しいかも。

### Eclipse PDE supportのダウンロード

下記で何気なく書かれていますが`jdtls`そのもののソースを読むためには`Eclipse PDE support`が必要です

https://github.com/eclipse-jdtls/eclipse.jdt.ls/blob/main/CONTRIBUTING.md#c-setting-up-the-jdt-language-server-in-neovim-or-emacs

多分とりあえず入れておいたほうがいいんじゃないかな？

githubのrelaeseページから適当に最新を選んでダウンロードし、zipとしてunzipします。

https://github.com/testforstephen/vscode-pde/releases

現時点(2025-04-08)では0.8.0が最新です。

```
$ mkdir -p ~/.local/jdtls/vscode-pde-0.8.0
$ pushd ~/.local/jdtls/vscode-pde-0.8.0
$ curl -LO https://github.com/testforstephen/vscode-pde/releases/download/0.8.0/vscode-pde-0.8.0.vsix
$ unzip vscode-pde-0.8.0.vsix
$ rm vscode-pde-0.8.0.vsix
$ popd
```

### lombokのダウンロード

必ず必要なのかはわかりませんが、すくなくともelasticsearch 8.x.yのソース内に`val`や`var`が出てくるので今回に限っては必須ですたぶん。

以下からダウンロードします

https://projectlombok.org/download

```
$ mkdir -p ~/.local/lombok/
$ pushd ~/.local/lombok/
$ curl -LO https://projectlombok.org/downloads/lombok.jar
$ popd
```

現時点(2025-04-08)時点での最新の`1.18.38`がダウンロードされます。

`.jar`ファイルは`zip`なので展開などは特に必要ありません。

### LunarVimやAstroVimの設定をパクろう

基本`LunarVim`や`AstroVim`がしっかりした設定を持っているのでそこからパクります

以下がそれぞれのjava向け設定です。

https://github.com/LunarVim/starter.lvim/blob/java-ide/ftplugin/java.lua

https://github.com/AstroNvim/astrocommunity/tree/main/lua/astrocommunity/pack/java

### 設定できる項目を洗う

設定可能な項目は以下で列挙されています。

https://github.com/eclipse-jdtls/eclipse.jdt.ls/wiki/Running-the-JAVA-LS-server-from-the-command-line

書いてありますが`Preferences.java`を直接見たほうがよいです

例えば下記などはwikiページには載ってません。

https://github.com/eclipse-jdtls/eclipse.jdt.ls/blob/v1.46.1/org.eclipse.jdt.ls.core/src/org/eclipse/jdt/ls/core/internal/preferences/Preferences.java#L369

### 最終的なconfig

下記は`AstroVim`のjava configやネットで見かけた様々な設定をもとに変更されています。
(パクろうといっといてなんですが基本的にjdtlsの公式からたどって設定できるものを全部設定した感じになってます)
`dap`周りはとりあえず無効化してあります(当面不要なので)
`defaults.on_attach`は`require("nvchad.configs.lspconfig").on_attach`です。

```lua
local M = {}

local function find_root()
  local source = vim.api.nvim_buf_get_name(vim.api.nvim_get_current_buf())
  local root_markers = { ".git", "mvnw", "gradlew", "pom.xml", "build.gradle", ".project" }
  local root_dir = vim.fs.root(vim.fs.dirname(source), root_markers)

  -- jdtls looks for "gradle/wrapper/gradle-wrapper.properties"
  -- some project only holds wrapper only on top directory while there's many sub-gradle projects
  local next_up = root_dir
  while next_up ~= nil and next_up ~= "" and next_up ~= "/" do
    local s = vim.uv.fs_stat(next_up .. "/gradle/wrapper/gradle-wrapper.properties")
    if s ~= nil then
      root_dir = next_up
      break
    end
    next_up = vim.fs.root(vim.fs.dirname(next_up), root_markers)
  end

  return root_dir
end

local make_opts = function(defaults)
  local root_dir = find_root()

  -- calculate workspace dir
  local project_name = vim.fn.fnamemodify(vim.fn.getcwd(), ":p:h:t")
  local workspace_dir = vim.fn.stdpath "data" .. "/site/java/workspace-root/" .. project_name
  vim.fn.mkdir(workspace_dir, "p")
  -- TODO: unique workspace for every unique paths.
  -- Base names truncated at some length and then suffix them with hash value of their absolute path.

  -- validate operating system
  local uname = vim.uv.os_uname()
  local config_suffixes = {
    Linux_x86_64 = "_linux",
    Linux_arm64 = "_linux_arm",
    Darwin_x86_64 = "_mac",
    Darwin_arm64 = "_mac_arm",
    Windows_x86_64 = "_win",
  }
  local suf = config_suffixes[uname.sysname .. "_" .. uname.machine]
  if suf == nil then
    vim.notify("jdtls: Could not detect valid OS", vim.log.levels.ERROR)
  end
  -- TODO: handle syntax only servers
  -- if ss then
  --     suf = "_ss" .. suf
  -- end
  local config_name = "config" .. suf

  local jdtls_home = vim.fn.expand "$HOME/.local/jdtls/eclipse.jdt.ls"
  local config_path = jdtls_home .. "/" .. config_name
  local launcher_path = vim.split(vim.fn.glob(jdtls_home .. "/plugins/org.eclipse.equinox.launcher_*.jar"), "\n")[1]
  return {
    cmd = {
      vim.fn.expand "$JAVA21_HOME/bin/java",
      "-Declipse.application=org.eclipse.jdt.ls.core.id1",
      "-Dosgi.bundles.defaultStartLevel=4",
      "-Declipse.product=org.eclipse.jdt.ls.core.product",
      "-Dlog.protocol=true",
      "-Dlog.level=ALL",
      "-XX:+UseParallelGC",
      "-XX:GCTimeRatio=4",
      "-XX:AdaptiveSizePolicyWeight=90",
      "-Dsun.zip.disableMemoryMapping=true",
      "-Xmx4G",
      "-Xms100m",
      "--add-modules=ALL-SYSTEM",
      "--add-opens",
      "java.base/java.util=ALL-UNNAMED",
      "--add-opens",
      "java.base/java.lang=ALL-UNNAMED",
      -- "--enable-native-access=ALL-UNNAMED",
      "-jar",
      launcher_path,
      "-configuration",
      config_path,
      "-javaagent:" .. vim.fn.expand "$HOME/.local/lombok/lombok.jar",
      "-data",
      workspace_dir,
    },
    root_dir = root_dir,
    settings = {
      java = {
        home = vim.fn.expand "$JAVA21_HOME",
        runtimes = {
          { name = "JavaSE-11", path = vim.fn.expand "$HOME/.local/openjdk/jdk-11.0.2" },
          { name = "JavaSE-12", path = vim.fn.expand "$HOME/.local/openjdk/jdk-12.0.2" },
          { name = "JavaSE-17", path = vim.fn.expand "$HOME/.local/openjdk/jdk-17.0.2" },
          { name = "JavaSE-21", path = vim.fn.expand "$HOME/.local/openjdk/jdk-21.0.2" },
          { name = "JavaSE-22", path = vim.fn.expand "$HOME/.local/openjdk/jdk-22.0.2" },
          { name = "JavaSE-23", path = vim.fn.expand "$HOME/.local/openjdk/jdk-23.0.2" },
          { name = "JavaSE-24", path = vim.fn.expand "$HOME/.local/openjdk/jdk-24" },
        },
        eclipse = { downloadSources = true },
        configuration = { updateBuildConfiguration = "interactive" },
        maven = { downloadSources = true },
        implementationsCodeLens = { enabled = true },
        referencesCodeLens = { enabled = true },
        inlayHints = { parameterNames = { enabled = "all" } },
        signatureHelp = { enabled = true },
        import = {
          gradle = {
            annotationProcessing = { enabled = true },
            -- arguments = {},
            enabled = true,
            java = {
              home = vim.fn.expand "$JAVA21_HOME",
            },
            jvmArguments = {
              "-Xmx4G",
              "-Xms100m",
            },
            wrapper = {
              enabled = true,
            },
          },
        },
        imports = {
          gradle = {
            wrapper = {
              checksums = {
                { sha256 = "41c8aa7a337a44af18d8cda0d632ebba469aef34f3041827624ef5c1a4e4419d", allowed = true },
              },
            },
          },
        },
        completion = {
          favoriteStaticMembers = {
            "org.hamcrest.MatcherAssert.assertThat",
            "org.hamcrest.Matchers.*",
            "org.hamcrest.CoreMatchers.*",
            "org.junit.jupiter.api.Assertions.*",
            "java.util.Objects.requireNonNull",
            "java.util.Objects.requireNonNullElse",
            "org.mockito.Mockito.*",
          },
        },
        sources = {
          organizeImports = {
            starThreshold = 9999,
            staticStarThreshold = 9999,
          },
        },
      },
    },
    init_options = {
      bundles = vim.split(
        vim.fn.glob(vim.fn.expand "$HOME" .. "/.local/jdtls/vscode-pde-0.8.0/extension/server/*.jar"),
        "\n"
      ),
    },
    handlers = {
      ["$/progress"] = function() end, -- disable progress updates.
      extendedClientCapabilities = {
        clientHoverProvider = false,
      },
    },
    filetypes = { "java" },
    on_attach = function(...)
      --      require("jdtls").setup_dap { hotcodereplace = "auto" }
      defaults.on_attach(...)
    end,
  }
end

M.config = function(servers, defaults)
  vim.api.nvim_create_autocmd("Filetype", {
    pattern = "java", -- autocmd to start jdtls
    callback = function()
      local opts = make_opts(defaults)
      if opts.root_dir and opts.root_dir ~= "" then
        require("jdtls").start_or_attach(vim.tbl_deep_extend("keep", opts, defaults))
      else
        vim.notify("jdtls: root_dir not found. Please specify a root marker", vim.log.levels.ERROR)
      end
    end,
  })
  -- create autocmd to load main class configs on LspAttach.
  -- This ensures that the LSP is fully attached.
  -- See https://github.com/mfussenegger/nvim-jdtls#nvim-dap-configuration
  vim.api.nvim_create_autocmd("LspAttach", {
    pattern = "*.java",
    callback = function(args)
      --      local client = vim.lsp.get_client_by_id(args.data.client_id)
      -- ensure that only the jdtls client is activated
      --     if client and client.name == "jdtls" then
      --       require("jdtls.dap").setup_dap_main_class_configs()
      --   end
    end,
  })
end

return M
```

### 変更点

`AstroVim`のjava configから変わっている部分について述べます。

#### find_root

`"mfussenegger/nvim-jdtls"`がすでに[`find_root`関数](https://github.com/mfussenegger/nvim-jdtls/blob/2f7bff9b8d2ee1918b36ca55f19547d9d335a268/lua/jdtls/setup.lua#L76-L90)を提供していますが、どうも`elasticsearch`のようにトップにしか`gradle/wrapper/gradle-wrapper.properties`を持っていなくて、サブプロジェクトとして`build.gradle`を持っているプロジェクトへの考慮は入っていないようです。

jdtlsが[`gradle/wrapper/gradle-wrapper.properties`を探す挙動を持っている](https://github.com/eclipse-jdtls/eclipse.jdt.ls/blob/v1.46.1/org.eclipse.jdt.ls.core/src/org/eclipse/jdt/ls/core/internal/managers/GradleProjectImporter.java#L412)ようなので、これと整合するようにします。
`gradle/wrapper/gradle-wrapper.properties`がある場合は`root_markers`が見つかってもさらにparent directoryに向けてさかのぼるようにします。

```lua

local function find_root()
  local source = vim.api.nvim_buf_get_name(vim.api.nvim_get_current_buf())
  local root_markers = { ".git", "mvnw", "gradlew", "pom.xml", "build.gradle", ".project" }
  local root_dir = vim.fs.root(vim.fs.dirname(source), root_markers)

  -- jdtls looks for "gradle/wrapper/gradle-wrapper.properties"
  -- some project only holds wrapper only on top directory while there's many sub-gradle projects
  local next_up = root_dir
  while next_up ~= nil and next_up ~= "" and next_up ~= "/" do
    local s = vim.uv.fs_stat(next_up .. "/gradle/wrapper/gradle-wrapper.properties")
    if s ~= nil then
      root_dir = next_up
      break
    end
    next_up = vim.fs.root(vim.fs.dirname(next_up), root_markers)
  end

  return root_dir
end
```

言語サーバーのインスタンスはroot_dirに対して1つなのでこうするのが正しいはず・・・。

#### configの分岐

`vim.fn.has("win32")`や`vim.fn.has("linux")`で分岐する例が多い気がしますが、archでも分岐しようと思ったら`vim.uv.os_uname`を用いるほうが手軽なのではと思います。というか単に筆者が`Node.js`で`libuv`に慣れているだけです。

```lua
  -- validate operating system
  local uname = vim.uv.os_uname()
  local config_suffixes = {
    Linux_x86_64 = "_linux",
    Linux_arm64 = "_linux_arm",
    Darwin_x86_64 = "_mac",
    Darwin_arm64 = "_mac_arm",
    Windows_x86_64 = "_win",
  }
  local suf = config_suffixes[uname.sysname .. "_" .. uname.machine]
  if suf == nil then
    vim.notify("jdtls: Could not detect valid OS", vim.log.levels.ERROR)
  end
  -- TODO: handle syntax only servers
  -- if ss then
  --     suf = "_ss" .. suf
  -- end
  local config_name = "config" .. suf
```

よくよく見てみると`config_*`は`arm`をサポートしているので、それ向けの分岐を加えます。例によって全く検証してないので`arm64`で動く保証は全くないですが。

```
~/.local/jdtls/eclipse.jdt.ls$ find ./ -name 'config_*'
./config_mac_arm
./config_ss_mac_arm
./config_ss_linux
./config_win
./config_linux
./config_ss_mac
./config_ss_win
./config_mac
./config_linux_arm
./config_ss_linux_arm
```

`_ss_`がついたconfigはsyntax-only modeで起動を意味するようです。
[Running the lightweight syntax language server](https://github.com/eclipse-jdtls/eclipse.jdt.ls/wiki/Running-the-lightweight-syntax-language-server)の項目で説明されていますが、フルセットのconfigだと初回起動時の処理がものすごく長いのでsyntax-only modeのconfigが用意されているようです。
現状TODOでとどめ置かれていますが何かしらの設定を読んで`config_ss_linux`に切り替わるように作ってあげると便利かもしれないですね。

#### heapの拡張

```lua
cmd = {
  vim.fn.expand "$JAVA21_HOME/bin/java",
  ...省略...
  "-XX:+UseParallelGC",
  "-XX:GCTimeRatio=4",
  "-XX:AdaptiveSizePolicyWeight=90",
  "-Dsun.zip.disableMemoryMapping=true",
  "-Xmx4G",
  "-Xms100m",
}
...省略...
settings = {
  java = {
    import = {
      gradle = {
        jvmArguments = {
          "-Xmx4G",
          "-Xms100m",
        },
      },
    }
  }
}
```

ログを見てるとちょいちょい`java.lang.OutOfMemoryError: Java heap space`が起きていたので、以下で提案されている起動オプションを追加します。

https://github.com/redhat-developer/vscode-java/issues/1548

`-XX`系は将来的に悪さするかもしれないので外しておくほうが無難かもしれませんね。

`gradle`への設定追加は機能してるのかよくわかりません。wrapperがうまく使われている場合はvmargはプロジェクト内に書かないと適応されないかもしれません。

#### PDEの追加

これがないと`jdtls`自身のソースコードがうまく読み込めなかったのでこれはとりあえず入れておいたほうが良いはず・・・。

```lua
    init_options = {
      bundles = vim.split(
        vim.fn.glob(vim.fn.expand "$HOME" .. "/.local/jdtls/vscode-pde-0.8.0/extension/server/*.jar"),
        "\n"
      ),
    },
```

## トラブルシューティングとか躓きどころとか

### 起動コマンドの例でjdtlsを指定させたりjava -jarを指定させたりしてるのはなんで？

`jdtls`コマンドはバイナリリリースに同梱されているpythonスクリプトのラッパーで中で`java -jar *.jar`を呼び出しています。

```
~/.local/jdtls/eclipse.jdt.ls/bin$ ls
jdtls  jdtls.bat  jdtls.py
```

以下でも説明されています。

https://github.com/eclipse-jdtls/eclipse.jdt.ls?tab=readme-ov-file#running-from-command-line-with-wrapper-script

つまり`jdtls`を指定している例はこのラッパースクリプトを呼び出させていて、`java -jar`で指定させる例はもっと詳細に設定を行うためにはこうするという意味合いです。

`config_${ss-only}${os}_${arch}`の分岐などやってくれるみたいですが細かく設定をいじらざるを得なかったので筆者は使用していません。

### lsp client quit with exit code 13 and signal 0

- `java`が古い
  - `jdtls`の必須バージョンの`JDK21`以降が満たされていない
- workspaceのキャッシュが古い
  - `~/.local/share/nvim/site/java/workspace-root/`以下にroot_dirごとにworkspaceディレクトリが切られていますので消すと解決することがあります。

### ログ

`:LspLog`だとログが見にくすぎます。
workspaceディレクトリの下にlogファイルが吐かれているのでそっちを見たほうがいいかもしれません。

```
$ cd ~/.local/share/nvim/site/java/workspace-root/${PROJECT_DIR}/.metadata
$ tail -f .log
```

ログがrotateされるのを`tail -f`だと多分検知できないからもうちょい工夫がいるかもですが、とりあえずこれを見ればjdtlsが出すログが見えます。

### Updated ... in ... msとstatuline下部に出続ける

多分そういうものなのだと思います。

この行で出力されていると思われます。

https://github.com/eclipse-jdtls/eclipse.jdt.ls/blob/v1.46.1/org.eclipse.jdt.ls.core/src/org/eclipse/jdt/ls/core/internal/managers/ProjectsManager.java#L498

筆者がElasticsearchプロジェクトを開いてみたところ眺めてる限り3時間ぐらい全部終わるまでかかりました。
正常なことなのかはわかりませんがsyntax-only modeのモチベーションが生じていることから時間がかかる処理なのは間違いないようです。

## おわりに

- `jdtls`のソースコードは大概うまいこと表示できています
  - `.jar`からのデコンパイル時にあるはずのクラスがなくなっていたりするのでうまく動いていない側面もありそうです
- `Elasticserach 8.17.4`ではdiagnoseに「reolveできない」とでるが定義ジャンプ、実装検索など動くようになった
  - 実態と会わないdiagnoseが表示されるのはneovim 0.11であることと各種プラグインが整合していないのかもしれません。
  - もしくはjdltsの不具合か

ためしにmavenのプロジェクトを作って書いた見た感じうまく動いてます。苦労するのはgradleですね。

今後は

- neovim 0.11の変更に追従する
  - `nvim-lspconfig`を使わなくても十分設定できるようになっているはずなので徐々に外していったほうがメインラインと齟齬がなくいいのかなと思ってまます。
- `"mfussenegger/nvim-jdtls"`にフィードバックする
  - find_rootの実装はフィードバックしたほうが良い気がします。
- jdtls自体のソースを読み、どうしてうまく動くか/動かないかを確かめる
  - もしかしたらアップストリームに何かしらのPRを送って修正が必要なところがあるのかもしれません。
