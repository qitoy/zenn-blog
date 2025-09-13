---
title: "nvim-treesitterのパーサーをnixで管理します"
emoji: "❄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "neovim", "nix", "treesitter" ]
published: true
---

# 2025-09-14 追記

nvim-treeのmainブランチに対応した。この記事の内容が前提になっている。
https://zenn.dev/qitoy/articles/nvim-treesitter-main

# 経緯

現在Neovim本体の管理にnixを利用しているのだが、大量にあるnvim-treesitterのパーサーもnixで管理したいと思いはじめた。しかしプラグインの設定までもnixで管理するのはやや過剰であり、また逆に不便になる。そこでnixでの管理と設定の利便性を両立させるように試行錯誤し、その解決法の1つを導いた。

:::message
以前記した方法より簡潔だと思う手法を思い付いたため記事を大幅に書き換えた。必要であれば[こちら](https://github.com/qitoy/zenn-blog/blob/cc774336af60c4d1bd739b3efb2dfec55371bb71/articles/treesitter-with-nix.md)から過去の版を参照してほしい。
:::

# TL;DR

プラグイン本体をnixで生成し適当なパスに配置、またvimrc以下にプラグインの設定を書きパッケージマネージャー側で2つを読み取りマージする。

# 環境

home-manager + dpp.vim

# どうしたか

## アイデア

2つの両立を考えた結果、nixで`~/.cache/dpp/_generated/nvim-treesitter`にパーサー入りプラグインを配置し、dpp-ext-localでローカルプラグインとして読み込むことにした。

:::details _generatedを~/.cache/dpp以下に配置した理由
自分の環境では`~/dotfiles/vim`を`~/.config/nvim`にシンボリックリンクしている。この状態で`~/.config/nvim/dpp/_generated`にhome-managerで配置しようとしてもエラーが発生する。一方これを回避するために`~/dotfiles/vim/dpp/_generated`に配置することも考えたが、これは美しくない。`_generated`は消えても再生成できるなどの理由から`~/.cache/dpp`以下に配置するのが美しいと考えた。
:::

これを実現するために、`pkgs.symlinkJoin`を用いてnvim-treesitter本体とパーサーを1つのディレクトリにまとめてそれを配置することにした。

## 実装

home-managerの設定ファイルに
```nix
  home.file =
    let
      base = ".cache/dpp/_generated";
    in
    pkgs.lib.attrsets.foldlAttrs (
      acc: name: drv:
      acc // { "${base}/${name}".source = drv; }
    ) { } (import ./plugins.nix { inherit pkgs; });
```
を書き加え、`plugins.nix`を
```nix:plugin.nix
{ pkgs }:
{
  nvim-treesitter =
    let
      ts = pkgs.vimPlugins.nvim-treesitter;
    in
    pkgs.symlinkJoin {
      name = "ts-all";
      paths = [
        ts
      ] ++ ts.withAllGrammars.dependencies;
    };
}
```
とする。こうすることにより`~/.cache/dpp/_generated/nvim-treesitter`にnvim-treesitter全部入りのパスを指定したプラグインが配置される。これをdpp.vimが読めるようにすればよい。

dpp-ext-localを用いて以下のようにして読み込む。
```ts
const [localExt, localOptions, localParams] = await args.dpp.getExt(
  args.denops,
  options,
  "local",
) as [LocalExt | undefined, ExtOptions, LocalParams];
if (localExt) {
  const local = localExt.actions.local;

  const localPlugins = await local.callback({
    denops: args.denops,
    context,
    options,
    protocols,
    extOptions: localOptions,
    extParams: localParams,
    actionParams: {
      directory: "~/.cache/dpp/_generated",
    },
  });

  for (const plugin of localPlugins) {
    if (plugin.name in recordPlugins) {
      Object.assign(recordPlugins[plugin.name], plugin);
    } else {
      recordPlugins[plugin.name] = plugin;
    }
  }
}
```
また、nvim-treesitterの設定でパーサーの自動アップデートを記述している場合は削除する。この変更で既存の設定はそのままに、プラグイン本体とパーサーはnixのものを使うことができる。

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

# 余談

拡張性のある実装をしているため、たとえばskkの辞書もnixで管理したくなったとき、
```nix:plugins.nix
skk-dict = pkgs.skkDictionaries.l;
```
の行を追記し、vim側で
```vim
const s:skk_dict = dpp#get('skk-dict').path
```
とすれば大きな手を加えることなく管理ができる。
