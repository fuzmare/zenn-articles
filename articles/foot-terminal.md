---
title: "alacrittyよりも速いターミナルfoot"
emoji: "🦶🏻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [terminal]
published: false
---
:::message
footはwayland用に作られたターミナルエミュレーターです。macの方、xorg環境の方は使えないんじゃないかと思います。
:::

alacrittyから乗り換えてfootというターミナルをしばらく使っています。zennに紹介記事がなかったので初めてですが記事を書いてみようと思い書いています。
この記事はREADME, wikiを読んで「はえ～なるほど」と思った部分、footを使ってみて気に入ったポイント、そして小学生並みの感想から成っています。ご了承ください。

# foot とは

@[card](https://codeberg.org/dnkl/foot)

高速なのにGPUレンダリングじゃないターミナルです。
C言語で書かれています。
まったくクロスプラットフォームではないです。

# footが速い理由

[wiki](https://codeberg.org/dnkl/foot/wiki/Performance)によると速さの主な理由は2つです。

## 速い理由1:VT parserが非常に高速

VT parserはエスケープシーケンスのパーサーです。
公式ドキュメントにfootとその他ターミナルエミュレータ(alacritty,urxvt,xterm)のvtebench結果を比較する[表](https://codeberg.org/dnkl/foot/src/branch/master/doc/benchmark.md)が掲載されています。footの各項目のスコアはalacrittyを大幅に上回るか同程度となっており、大きく劣る項目はありません。
footがいくつかの項目において非常に高速である主な理由がVT parserの速さなのだそうです。
vtebenchはPTYの読み取り速度を測るベンチマークなので、パーサーが高速だというのは確かに効いてくる部分なのでしょう。

## 速い理由2:差分レンダリング

footのレンダリングエンジンは画面内で変化があった領域のみ再描画する差分レンダリングを行います。 1文字、1行だけの変更やスクロール^[スクロールは画面全体が動くので素朴には全体の更新となってしまいますが、footは既存の表示を移動させて追加の行だけレンダリングすることができます。]などにおいて効果は絶大で、alacrittyよりも圧倒的に速く描画してくれます。
画面全体に更新が発生した場合はalacrittyに劣りますが、全体の再描画はそこまで多くないので、実用上alacrittyより高速です。(alacrittyの画面描画は小さな変更でも画面全体を再描画します。)

そもそも、画面全体を書き換えるようなイベントはそのイベント自体がそこそこ時間がかかってターミナルの描画時間は誤差になるような気がします。

# footのすごいところ、気に入っているところ
