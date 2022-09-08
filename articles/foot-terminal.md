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

alacrittyから乗り換えてfootというターミナルをしばらく使っています。zennに紹介記事がなさそうだったので、初めてですが記事を書いてみようと思い書いています。

この記事はREADME, wikiを読んで「はえ～なるほど」と思った部分、footを使ってみて気に入ったポイント、そして小学生並みの感想から成っています。ご了承ください。

# foot とは

@[card](https://codeberg.org/dnkl/foot)

GPUを使わないにも関わらず超高速のC言語製ターミナルエミュレーターです。

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

# footの良いところ、気に入っているところ

## 1.サーバー/クライアントモードがある
バックグラウンドでサーバーを起動しておくことで省メモリーと爆速起動を実現するやつです。(urxvtのデーモン/クライアントモード相当)

# 描画速度実験

footとalacrittyの描画速度が手元の環境でどれくらい違うか確かめてみました。

## 環境

CPU: Core i5-8250U
OS: Arch linux

```
❯ foot --version
foot version: 1.13.1 +pgo +ime +graphemes -assertions

❯ alacritty --version
alacritty 0.10.1 ()

❯ sway --version
sway version 1.7
```

## ザーっと文字が流れていくときの性能

```sh
exa -lR /usr > result
```

でファイルを作って、

```sh
hyperfine -r 10 --show-output -N "cat result"
```

をそれぞれ実行しました。

### とりあえずそのままで



### tmux上で



### zellij上で



## vimしてるときの性能

render-timerを使って描画にかかった時間を見ました。
vimと言いつつneovimを使ったのですがまあ見るのはrender-timerの時間なのでどんなアプリケーションを使ったかでそう違ったりはしないでしょう。

### 1文字打つ

空のバッファーに"a"と1文字打ちました。

### 1行スクロールする

先の実験で作った/usr以下の一覧を使いました。

### 1行消してみる

dd で一行消しました。

### 画面全体の変化

G で一番下に行きました。
