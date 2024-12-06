---
title: "nvim-treesitterのパーサーをnixで管理します"
emoji: "❄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "neovim", "nix" ]
published: true
---

# 経緯

現在neovimの管理にnixを利用しているのだが、大量にあるnvim-treesitterのパーサーもnixで管理したいと思いはじめた。しかし安易に導入するとnix式の評価に比較的時間がかかることなどから設定の利便性が下がる。そこでnixでの管理と設定の利便性を両立させるように試行錯誤し、その解決法の1つを導いた。

# TL;DR

プラグイン本体をnixで生成し適当なパスに配置、またvimrc以下にプラグインの設定を書きパッケージマネージャー側で2つを読み取りマージする。

# 環境

home-manager + dpp.vim

# どうしたか

## 導入前

```ts:dpp.ts
...
const tomlLoad = tomlExt.actions["load"];

const tomls = await Promise.all(
  [
    { path: "/path/to/plugins.toml", lazy: false },
    { path: "/path/to/plugins_lazy.toml", lazy: true },
  ].map((toml) =>.map((toml) =>
    tomlLoad.callback({
      denops: args.denops,
      context,
      options,
      protocols,
      extOptions: tomlOptions,
      extParams: tomlParams,
      actionParams: {
        path: toml.path,
        options: {
          lazy: toml.lazy,
        },
      },
    })
  ),
) as (Toml | undefined)[];

// merge result
for (const toml of tomls) {
  for (const plugin of toml.plugins ?? []) {
    recordPlugins[plugin.name] = plugin;
  }
}
...
```

```toml:plugins_lazy.toml
[[plugins]]
repo = 'nvim-treesitter/nvim-treesitter'
on_event = ['BufRead', 'CursorHold']
hook_post_update = 'TSUpdate'
lua_source = '''
require'nvim-treesitter.configs'.setup {
  ensure_installed = 'all',
  ...
}
'''
```

## アイデア

2つの両立を考えた結果、nixで`~/.cache/dpp/_generated.toml`にプラグインとパーサーを吐き出し、一方`plugins_lazy.toml`内で設定を記述し、`dpp.ts`側でマージすることにした。

:::details _generated.tomlを~/.cache/dpp以下に配置した理由
自分の環境では`~/dotfiles/vim`を`~/.config/nvim`にシンボリックリンクしている。この状態で`~/.config/nvim/dpp/_generated.toml`にhome-managerで配置しようとしてもエラーが発生する。一方これを回避するために`~/dotfiles/vim/dpp/_generated.toml`に配置することも考えたが、これは美しくない。`_generated.toml`は消えても再生成できるなどの理由から`~/.cache/dpp`以下に配置するのが美しいと考えた。
:::

まずnatsukium氏のdotfilesの[nix部分](https://github.com/natsukium/dotfiles/blob/6566016ed187957b770561d78fb8b57c432220d9/applications/nvim/default.nix#L77-L80)、[nvim-treesitter部分](https://github.com/natsukium/dotfiles/blob/6566016ed187957b770561d78fb8b57c432220d9/applications/nvim/lua/plugins/misc.lua#L83-L90)を参考にして書いてみたが、runtimepathが巨大になってしまう。dpp.vimがruntimepathの圧縮をしているのにこれではもったいないので何らかの方法で1つにまとめたい。

そこで`stdenv.mkDerivation`を用いてnvim-treesitter本体とパーサーを1つのディレクトリにまとめてそれを配置することにした。

## 実装

まずnix側に手を加える。`home.nix`に
```nix:home.nix
home.file.".cache/dpp/_generated.toml".source =
    let tomlFormat = pkgs.formats.toml { };
    in tomlFormat.generate "_generated.toml" (import ./plugins.nix { inherit pkgs; });
```
を書き加え、`plugins.nix`を
```nix:plugin.nix
{ pkgs }: {
  plugins = [
    {
      name = "nvim-treesitter";
      path =
        let
          ts = pkgs.vimPlugins.nvim-treesitter;
          ts-all = pkgs.symlinkJoin {
            name = "ts-all";
            paths = [ ts ] ++ ts.withAllGrammars.dependencies;
          };
        in
        "${ts-all}";
    }
  ];
}
```
とする。こうすることにより`~/.cache/dpp/_generated.toml`にnvim-treesitter全部入りのパスを指定したプラグインを含むtomlが配置される。これをdpp.vimが読めるようにすればよい。

`dpp.ts`及び`plugins_lazy.toml`に以下の変更を加える。
```diff ts:dpp.ts
   
     { path: "/path/to/plugins.toml", lazy: false },
     { path: "/path/to/plugins_lazy.toml", lazy: true },
+    { path: "~/.cache/dpp/_generated.toml", lazy: false },
   ].map((toml) =>.map((toml) =>
...
   for (const plugin of toml.plugins ?? []) {
-    recordPlugins[plugin.name] = plugin;
+    recordPlugins[plugin.name] = {
+      ...plugin,
+      ...recordPlugins[plugin.name],
+    };
   }
```
```diff toml:plugins_lazy.toml
 [[plugins]]
-repo = 'nvim-treesitter/nvim-treesitter'
+name = 'nvim-treesitter'
 on_event = ['BufRead', 'CursorHold']
-hook_post_update = 'TSUpdate'
 lua_source = '''
 require'nvim-treesitter.configs'.setup {
-  ensure_installed = 'all',
   ...
 }
 '''
```
この変更で`_generated.toml`を読み、既存設定を優先させるようにマージさせ、プラグイン本体とパーサーはnixのものを使うことができる。

これでbuiltinのパーサーについては完了した。次に外部のパーサーを追加する方法を記す。

## 外部のパーサーを追加する

ここでは[tree-sitter-satysfi](https://github.com/monaqa/tree-sitter-satysfi)を追加することにしよう。まずsatysfiのパーサーを作成する。適当な場所に
```nix
{ tree-sitter, fetchFromGitHub }: tree-sitter.buildGrammar {
  language = "satysfi";
  version = "5519c547418ecb31ac7d63e64653aed726b5d1c3";
  src = fetchFromGitHub {
    owner = "monaqa";
    repo = "tree-sitter-satysfi";
    rev = "5519c547418ecb31ac7d63e64653aed726b5d1c3";
    fetchSubmodules = false;
    sha256 = "sha256-yei8UHiVChYpx2UyPsDyOd3usItZN68rwu0+VoBtPi0=";
  };
}
```
（一例、nvfetcherを利用してもよい）を配置し、`callPackage`をする。これを`tree-sitter-satysfi`とする。これを`plugins.nix`に渡し、以下のようにすればよい。
```diff nix:plugin.nix
-            paths = [ ts ] ++ ts.withAllGrammars.dependencies;
+            paths = [ ts (pkgs.neovimUtils.grammarToPlugin tree-sitter-satysfi) ] ++ ts.withAllGrammars.dependencies;
```
これで新たなパーサーが追加された。どうやらREADMEにある`queries/`以下をコピーする工程は必要ないらしい。

# おわりに

nvim-treesitterの設定自体はそこまで頻繁にいじるものでもないので完全にnix側に倒してもよかった。だがこの機会に導入してみたことが他の様々なプラグイン本体をnixで管理したくなった時に役立つのだと思う。

# 変更履歴

- 12/06 文言を一部修正。`ts-all`の際に`pkgs.symlinkJoin`を使うとよいとのことなのでそれを使用するように。
