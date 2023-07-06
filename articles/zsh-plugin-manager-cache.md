---
title: "究極のzshプラグイン読み込み: プラグインマネージャーの限界を越える"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [zsh]
published: false
---

このような記事をご覧の皆様におかれましては、日々盆栽を丹念に育て、自身だけの最強開発環境を追求していることと存じます。 皆様であればプラグインの読み込みが起動時間に影響を及ぼさないよう、プラグインの遅延起動を設定していることでしょう。しかし、プラグインマネージャージ自体の起動時間を気にしたことはございますでしょうか。 今回は、プラグインマネージャーの読み込みを回避してプラグインを読み込むことにより、zshをさらに高速起動する手法を共有します。これにより、我々の盆栽は疾風の如く立ち昇り、開発の絶え間ない流れを切り裂く、驚異的なスピードを得ることでしょう。

## 制約
本テクニックは基本的に外部バイナリのプラグインマネージャーに適しており、`.zsh`実装のプラグインマネージャーではうまく行かないはずです。
本記事ではsheldonプラグインマネージャーを使用します。

https://github.com/rossmacarthur/sheldon

よかったsheldonの紹介記事

https://zenn.dev/ganta/articles/e1e0746136ce67

## 手法の説明
sheldonプラグインマネージャーは`sheldon source`の実行でプラグインをzshに読み込ませるためのスクリプトを吐きます。

```sh
❯ sheldon source
source "<path to plugin a>"
source "<path to plugin b>"
.
.
.
```

これを`eval`に渡すのが通常の読み込みです。
```sh
eval "$(sheldon source)"
```

しかし、これを毎回やるのはロスです。このスクリプトが変化するのは設定が変化した時のみなので、毎回プラグインの構成を変える人でもない限り、ほぼ毎回結果の変わらない無駄な処理です。
そこで、`sheldon source`の出力を取っておいて、普段はそれを`source`するだけにすれば、起動が高速化できます。そのためには次のように書きます。
```sh
if [[ -r "$HOME/.cache/sheldon/sheldon.zsh" ]]; then
  source "$HOME/.cache/sheldon/sheldon.zsh"
else
  mkdir -p $HOME/.cache/sheldon
  eval "$(sheldon source | tee $HOME/.cache/sheldon/sheldon.zsh)"
fi
```

条件分岐はキャッシュが存在していないケースで`sheldon source`を実行してプラグインの読み込みとキャッシュの作成をしてもらうためのものです。
