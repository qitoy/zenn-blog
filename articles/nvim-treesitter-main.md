---
title: "nvim-treesitterのmainブランチに対応します"
emoji: "🌳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "neovim", "nix", "treesitter" ]
published: true
---

# はじめに

2025-05-24にnvim-treesitterがmainブランチに移行し、大幅な変更を加えた。対応を面倒くさがっていたが、重い腰を上げて対応を完了した。

# 前提知識

自分はこれまで以下のような設定を行っていた。
https://zenn.dev/qitoy/articles/treesitter-with-nix

# どうしたか
## nix側の設定
nixpkgsのを直接mainブランチにするので簡潔な方法がわからなかったので自力で対応することにした。自分は[nvfetcher](https://github.com/berberman/nvfetcher)を使用しているので、
```toml
[nvim-treesitter-main]
src.git = "https://github.com/nvim-treesitter/nvim-treesitter.git"
src.branch = "main"
fetch.github = "nvim-treesitter/nvim-treesitter"
```
と記述した。そしたらこれをNeovimプラグインとしてビルドする。前回との差分をとると、
```diff nix:plugin.nix
-{ pkgs }:
+{ pkgs, sources }:
 {
   nvim-treesitter =
     let
-      ts = pkgs.vimPlugins.nvim-treesitter;
+      nvim-treesitter-main = pkgs.vimUtils.buildVimPlugin {
+        inherit (sources.nvim-treesitter-main) pname version src;
+        nvimSkipModules = [ "nvim-treesitter._meta.parsers" ];
+      };
     in
     pkgs.symlinkJoin {
       name = "ts-all";
       paths = [
-        ts
+        nvim-treesitter-main
       ]
-      ++ ts.withAllGrammars.dependencies;
+      ++ pkgs.vimPlugins.nvim-treesitter.withAllGrammars.dependencies;
+      postBuild = ''
+        cd $out
+        mkdir -p ./queries
+        mv ./runtime/queries/* ./queries
+      '';
     };
 }
```
という感じだろうか（`sources`はnvfetcherで追加したものだ）。ここで3点ほど追加説明が必要になる。
1. `nvimSkipModules`はテストをしないluaモジュールを指定する。`nvim-treesitter._meta.parsers`は`require`しない（するとエラーになる）モジュールなのでここで指定している。
2. パーサーを追加するところで`pkgs.vimPlugins.nvim-treesitter`を指定しているのはパーサーの情報はnixpkgsで追加されたものだからだ（つまり`nvim-treesitter-main`から参照することはできない）。
3. `postBuild`について。後述するが、クエリファイルは`nvim-treesitter/queries/{filetype}/`以下に置いてほしいのでこう記述する。

## Neovim側の設定
Neovim側の設定については以下の記事が詳しい。
https://blog.atusy.net/2025/08/10/nvim-treesitter-main-branch/
https://blog.yasunori0418.dev/p/migrate-treesitter-main

ここでは自分が特別に設定した項目について述べる。上で配置したファイルにあるパーサー・クエリファイルを読んでほしいので、`install_dir`を設定する必要がある。前回の記事の設定ですでにdpp.vimで読めるようにしたので以下の設定でよい。
```lua
require"nvim-treesitter".setup {
  install_dir = vim.fn["dpp#get"]("nvim-treesitter").path,
}
```
nvim-treesitterは`install_dir`以下に`parser`ディレクトリと`queries`ディレクトリが存在することを期待するので上の`postBuild`スクリプトが必要だった。

# おわりに
大幅に変わっていると聞いて対応を渋っていたのだが、やってみると意外とすんなりできてよかった。
