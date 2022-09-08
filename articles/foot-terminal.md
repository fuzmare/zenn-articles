---
title: "速いターミナルfoot"
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

GPUを使わないにも関わらず高速なC言語製ターミナルエミュレーターです。
全体的に速いですが、状況次第でalacrittyの倍以上速いケースもあれば、alacrittyの半分以下に落ち込むケースもあります。

# footが速い理由

[wiki](https://codeberg.org/dnkl/foot/wiki/Performance)によると速さの主な理由は2つです。

## 速い理由1:VT parserが高速

VT parserはエスケープシーケンスのパーサーです。
公式ドキュメントにfootとその他ターミナルエミュレータ(alacritty,urxvt,xterm)のvtebench結果を比較する[表](https://codeberg.org/dnkl/foot/src/branch/master/doc/benchmark.md)が掲載されています。footの各項目のスコアはalacrittyを大幅に上回るか同程度となっており、大きく劣る項目はありません。
footがいくつかの項目において非常に高速である主な理由がVT parserの速さなのだそうです。
vtebenchはPTYの読み取り速度を測るベンチマークだそうですので、パーサーが高速だというのは確かに効いてくる部分なのでしょう。

## 速い理由2:差分レンダリング

footのレンダリングエンジンは画面内で変化があった領域のみ再描画する差分レンダリングを行います。 1文字、1行だけの変更やスクロール^[スクロールは画面全体が動くので素朴には全体の更新となってしまいますが、footは既存の表示を移動させて追加の行だけレンダリングすることができます。]などにおいて効果は絶大で、alacrittyよりも速く描画してくれます。
画面全体に更新が発生した場合はalacrittyに劣りますが、全体の再描画はそこまで多くないので、実用上alacrittyより高速です。(alacrittyの画面描画は小さな変更でも画面全体を再描画します。)

そもそも、画面全体を書き換えるようなイベントはそのイベント自体がそこそこ時間がかかってターミナルの描画時間は誤差になるような気がします。

# footの良いところ、気に入っているところ

## サーバー/クライアントモードがある
バックグラウンドでサーバーを起動しておくことで省メモリーと爆速起動を実現するやつです。(urxvtのデーモン/クライアントモード相当)
複数のウィンドウ(クライアント)が1つのサーバープロセスを共有するため、リソースを一本化してメモリ消費を節約できます。
また、新しいウィンドウの起動はプログラムの本体的な部分やフォント等の読み込みの必要がなく、クライアントを1つ生やすだけで済むのですごく速いです。

私はswayユーザーで、最近neovimをfloating windowで起動させるキーボードショートカットを設定したのですが、footclientの起動は一瞬、neovimの起動もプラグインの遅延読み込みにより一瞬なので良い気分です。
適当計測結果によると80ミリ秒でvim成分が摂取可能らしいです。ホントか？

## sixel対応
alacrittyは今(2022-09-08)のところsixel非対応……

# 性能検証

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

## ザーっと文字が流れていくときの全体的な性能

```sh
exa -lR /usr > result
```

でファイルを作って、

```sh
hyperfine -r 10 --show-output -N "cat result"
```

をそれぞれ実行しました。

### とりあえずそのままで

![](https://github.com/fuzmare/zenn-articles/blob/main/articles/foot-terminal/cat-raw.png?raw=true)

foot: 284.4 ms ± 16.2 ms
alacritty: 567.8 ms ± 33.5 ms

footが2倍ほど速いです。画面を流れていくような表示には滅法強いのが見て取れます。

### tmux上で

![](https://github.com/fuzmare/zenn-articles/blob/main/articles/foot-terminal/cat-tmux.png?raw=true)

foot: 1.608 s ± 0.015 s
alacritty: 1.583 s ± 0.046 s

両者大幅に速度が低下します。
footの方がalacrittyより僅差ですが遅くなりました。何度かやっても同じだったので偶然ではないようです。
foot自前のスクロールではなくtmux側がスクロールを行なっていることが差分レンダリングスクロールの性能を下げているようです。^[ステータスバーのせいかなと思ってtmux set status offをしても遅かったのでtmux自体が悪い]render-timerを表示して実行してみたところ画面全体の再描画よりは高速に描画できてるっぽかったので無効になっているわけではなさそう。

### zellij上で

![](https://github.com/fuzmare/zenn-articles/blob/main/articles/foot-terminal/cat-zellij.png?raw=true)

foot: 1.198 s ± 0.019 s
alacritty: 1.121 s ± 0.015 s

tmuxのときと同じ傾向ですが、tmuxほど性能が落ちていません。zellijの方が速いらしい。
~~でもzellijあんまり好きじゃない。~~

## vtebench

vtebenchスコアを確認しました。

|Benchmark|foot|alacritty|
|:-|-:|-:|
cursor_motion|15.52ms|31.58ms|
dense_cells|46.32ms|113.93ms|
light_cells|8.04ms|22.31ms|
scrolling|167.66ms|114.03ms|
scrolling_bottom_region|128.22ms|121.91ms|
scrolling_bottom_small_region|130.88ms|118.85ms|
scrolling_fullscreen|9.68ms|19.18ms|
scrolling_top_region|136.57ms|130.92ms|
scrolling_top_small_region|136.15ms|119.94ms|
unicode|18.57ms|26.79ms|

## 描画性能のチェック with vim

render-timerを使って描画にかかった時間を見ました。
ここで見られるのはあくまでレンダリング性能であってもう一つの心臓たるVT parserの効果は反映されません。
vimと言いつつneovimを使ったのですがまあ見るのはrender-timerの時間なのでどんなアプリケーションを使ったかでそう違ったりはしないでしょう。

### 1文字打つ

空のバッファーに"a"と1文字打ちました。
![](https://github.com/fuzmare/zenn-articles/blob/main/articles/foot-terminal/vim-ia.png?raw=true)

foot: 260.91 µs
alacritty: 1338.148 µs

footがalacrittyより5倍も速くなっています。
やはり差分レンダリングは変化量が小さいときの効果が絶大です。

### 1行スクロールする

先の実験で作った/usr以下の一覧を使いました。
![](https://github.com/fuzmare/zenn-articles/blob/main/articles/foot-terminal/vim-1C^e.png?raw=true)

foot: 734.15 µs
alacritty: 1329.285 µs

### 2行以上スクロールする

footはスクロールする行数が増えるほど不利なのでスクロール行数を増やしていったときの変化を確認します。


|スクロール行数|render-timer時間(µs)|
|-:|-:|
|1|700くらい|
|2|1200くらい|
|3|1300くらい|
|4|1400くらい|
|5|1600くらい|
|6|1800くらい|
|10|2500くらい|
|15|3700くらい|
|20|5000くらい|
|25|5500くらい|
|26|5500くらい|
|27|2500くらい|
|30|2500くらい|
|35|2500くらい|

27行スクロールから短くなり2500 µsくらいで安定しました。

### 画面全体の変化

G で一番下に行きました。
![](https://github.com/fuzmare/zenn-articles/blob/main/articles/foot-terminal/vim-G.png?raw=true)

foot: 2505.03 µs
alacritty: 1168.534 µs

2500 µsの正体は全体再描画の時間のようです。

### 1行消してみる

dd で一行消しました。
![](https://github.com/fuzmare/zenn-articles/blob/main/articles/foot-terminal/vim-dd.png?raw=true)

foot: 1946.93 µs
alacritty: 1469.813 µs

なんか遅い。実質、行を繰り上げてるだけだし差分スクロールが効いてくれそうな気がするんだけど……
もしや number 列があるから？

### set nonumberして1行消してみる

set nonu して dd で1行消してみました。

![](https://github.com/fuzmare/zenn-articles/blob/main/articles/foot-terminal/vim-nonu-dd.png?raw=true)

foot: 985.47 µs

それっぽい速さになりました。どうやら左右に領域を分割するような要素があると差分スクロールは働かないらしいです。

### 試しに左にディレクトリツリーを出してスクロールした

![](https://github.com/fuzmare/zenn-articles/blob/main/articles/foot-terminal/vim-fern+C^e.png?raw=true)

foot: 2562.01 µs

予想通り、全体再描画な時のレンダリング時間になっていました。
まあスクロール領域をうまく検出するのはそれはそれで時間がかかりそうなので単純なアルゴリズムで検出できなければ諦めて全体再描画というのは現実的な挙動なような気がします。

# まとめのようなもの

footは差分レンダリングにより表示内容の変化が小さいときはalacritty以上に速いターミナルエミュレーターです。

個人的にはサーバー/クライアントモードを持った現代的なターミナルというのが手放せないポイントとなっており、差分スクロールがちょっと微妙でたまにalacrittyより遅かろうとalacrittyに戻ろうという気持ちにはなりません。
そもそも、ターミナルを速くしてもtmuxが結構遅いみたいなのでfoot+tmuxでもalacritty+tmuxでも大差ないんじゃないかと今は思っています。(本末転倒)

だったら描画速度以外の速度……起動速度で選ぶべきではないでしょうか？
