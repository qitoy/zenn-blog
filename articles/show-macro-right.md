---
title: "マクロを右端に表示します"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "neovim" ]
published: true
---

# 動機

neovim上で`cmdheight = 0`をしているとき、マクロが記録されているか気になります。多くの人々が独自にこの問題に対処してきました。Shougo氏は記録中のときのみ`cmdheight`を1にします。
https://github.com/Shougo/shougo-s-github/blob/2f1c9acacd3a341a1fa40823761d9593266c65d4/vim/rc/vimrc#L47-L49
ryoppippi氏はColorSchemeを変更する手段を考案しました。
https://zenn.dev/vim_jp/articles/68eb77d2f2a37a

一方自分は最初はnoice.nvimを利用し表示していました。しかしながら自分のneovimにはnoiceは不要だと感じ、削除しました。そこでryoppippi氏の方法を見ていたところ、「結局なんらかの方法で表示するのなら、virtual-textのような方法で表示させればよいのではないか」と至り、これの実装に着手しました。

# アイデア

`vim.on_key()`を使用し記録中のマクロレジスタを`vim.api.nvim_buf_set_extmark()`で表示します。

# 実装

短いのでこちらを参照してください。neovim luaは初めてなので間違いがあるかもしれません。
https://github.com/qitoy/dotfiles/blob/9bfcfa3e6307d1a4554f3e9e8b727db249d0d306/vim/lua/extmode/init.lua
