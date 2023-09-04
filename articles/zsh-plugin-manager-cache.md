---
title: "究極のzshプラグイン読み込み高速化: プラグインマネージャーの限界を越えろ"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [zsh]
published: false
---

このような記事をご覧の皆様におかれましては、日々盆栽を丹念に育て、自身だけの最強開発環境を追求していることと存じます。 皆様であればプラグインの読み込みが起動時間に影響を及ぼさないよう、プラグインの遅延起動を設定していることでしょう。しかし、プラグインマネージャー自体の起動時間を気にしたことはございますでしょうか。 今回は、プラグインマネージャーの読み込みを回避してプラグインを読み込むことにより、zshをさらに高速起動する手法を共有します。これにより、我々の盆栽は疾風の如く立ち昇り、開発の絶え間ない流れを切り裂く、驚異的なスピードを得ることでしょう。

## 概要
- プラグインマネージャーの結果だけ取っておいて普段は起動しないようにしよう
- 遅延はzsh-deferでかけるよ
- 全部zcompileすれば最速

## 制約
本テクニックは基本的に外部バイナリのプラグインマネージャーに適しており、`.zsh`実装のプラグインマネージャーではうまく行かないはずです。
本記事ではsheldonプラグインマネージャーを使用します。

https://github.com/rossmacarthur/sheldon

よかったsheldonの紹介記事

https://zenn.dev/ganta/articles/e1e0746136ce67

## 前提: sheldonの設定 & 遅延ロード
プラグインのインストール設定についてはよくできた先行記事がありますし、今回のテクニックの本題ではないので大雑把に行きます。この記事だけを読んでの設定はおすすめできません。

さて、高速な起動のためには遅延ロードはほぼ必須です。zinitにはプラグインマネージャー組み込みの遅延ロードがありますが、sheldonには組み込まれてはいません。代わりに、コマンドの遅延実行を提供するプラグインであるzsh-deferを使います。(sheldon docs上で紹介されている方法です。)

https://github.com/romkatv/zsh-defer

https://sheldon.cli.rs/Examples.html?search=post#deferred-loading-of-plugins-in-zsh

sheldonではプラグインの設定はtomlファイル(デフォルトで~/.config/sheldon/plugins.toml)で、次のような感じで書きます。

```toml:plugins.toml
shell = "zsh"

[templates]
defer = "{{ hooks | get: \"pre\" | nl }}{% for file in files %}zsh-defer source \"{{ file }}\"\n{% endfor %}{{ hooks | get: \"post\" | nl }}"

[plugins.zsh-defer]
github = "romkatv/zsh-defer"

[plugins.xxx]
github = 'xxx/xxx.zsh'
apply = ["defer"]
hooks.pre = 'zsh-defer source $HOME/.config/zsh/hook/xxx_pre.zsh'
hooks.post = 'zsh-defer source $HOME/.config/zsh/hook/xxx_post.zsh'
```

`templates`下`defer`で定義しているのがzsh-deferを用いた遅延ロードの設定です。
sheldonでは`templates`を用いてプラグインの読み込み処理を自由に定義することができ、ここではただ単に`source`する代わりに`zsh-defer source`でsourceを遅延実行するテンプレートを定義しています。

読み込みは設定ファイルでの列挙順に行われるので、一番最初にzsh-deferを書き、その下からは`zsh-defer`によって遅延ロードするように書いていきます。`apply = ["defer"]`がdefer templateを使用するように宣言している部分です。
ただし、全てのプラグインが遅延できるわけではないので、一部の仕方ないやつは通常の読み込みになります。

ところで、2023年5月16日のsheldon 0.7.3 ([リリースノート](https://sheldon.cli.rs/RELEASES.html#073)) から`hooks`オプションが追加されていました。これにより、プラグインの読み込み前後の処理を挿入するのがスマートに書けるようになりました。やったぜ。

# 本題: プラグインマネージャーの読み込みを回避してプラグインを読み込む方法
sheldonプラグインマネージャーは`sheldon source`の実行でプラグインをzshに読み込ませるためのスクリプトを吐きます。

```sh
❯ sheldon source
source "<path to zsh-defer>"
zsh-defer source "<path to plugin a>"
zsh-defer source "<path to plugin b>"
.
.
.
```

これを`eval`に渡すのが通常の読み込みです。

```sh
eval "$(sheldon source)"
```

しかし、これを毎回やるのはロスです。`sheldon source`で出力されるスクリプトが変化するのは設定が変化した時のみなので、毎回プラグインの構成を変える人でもない限り、ほぼ毎回結果の変わらない無駄な処理です。
そこで、`sheldon source`の出力を取っておいて、普段はそれを`source`するだけにすれば、起動が高速化できます。そのためには.zshrcに次のようなことを書きます。

```sh:.zshrc
# ファイル名を変数に入れる
cache_dir=${XDG_CACHE_HOME:-$HOME/.cache/zsh}
sheldon_cache="$cache_dir/sheldon.zsh"
sheldon_toml="$HOME/.config/sheldon/plugins.toml"
# キャッシュがない、またはキャッシュが古い場合にキャッシュを作成
if [[ ! -r "$sheldon_cache" || "$sheldon_toml" -nt "$sheldon_cache" ]]; then
  mkdir -p $cache_dir
  sheldon source > $sheldon_cache
fi
source "$sheldon_cache"
# 使い終わった変数を削除
unset cache_dir sheldon_cache sheldon_toml
```

これで、キャッシュ (~/.cache/zsh/sheldon.zsh) が存在していない、またはキャッシュが設定ファイルよりも古い場合に`sheldon source`を実行してキャッシュの作成をしてくれます。

## 補足1: sourceされる全てのファイルをzcompileしてさらに高速化
これ単体でも独立したトピックとして成立するので記事を分けさせていただきました。
sourceコマンドを乗っ取ることでプラグインなども全てzcompileしちゃうぜという記事です。

https://zenn.dev/fuzmare/articles/zsh-source-zcompile-all

## 補足2: zsh-deferはプラグイン読み込み以外でも使える
zsh-deferで遅延できないコマンドは一部ありますが、多くは遅延できます。
私は遅延させる設定を置くlazy.zshと遅延できなかった設定を置くnonlazy.zshを用意しておき、基本的に新しい設定はまずlazy.zshに投入するようにしています。
後からでもいい設定を遅延させて読み込むことで、極めて早いファーストビュー、入力受付を得ることができます。

## 合体
以上をまとめて.zshrcは次のようになります。

```sh:.zshrc
# source command override technique
function source {
  ensure_zcompiled $1
  builtin source $1
}
function ensure_zcompiled {
  local compiled="$1.zwc"
  if [[ ! -r "$compiled" || "$1" -nt "$compiled" ]]; then
    echo "\033[1;36mCompiling\033[m $1"
    zcompile $1
  fi
}
ensure_zcompiled ~/.zshrc

# sheldon cache technique
cache_dir=${XDG_CACHE_HOME:-$HOME/.cache/zsh}
sheldon_cache="$cache_dir/sheldon.zsh"
sheldon_toml="$HOME/.config/sheldon/plugins.toml"
if [[ ! -r "$sheldon_cache" || "$sheldon_toml" -nt "$sheldon_cache" ]]; then
  mkdir -p $cache_dir
  sheldon source > $sheldon_cache
fi
source "$sheldon_cache"
unset cache_dir sheldon_cache sheldon_toml

source $HOME/.config/zsh/nonlazy.zsh
zsh-defer source $HOME/.config/zsh/lazy.zsh
zsh-defer unfunction source
```

## 効果の検証

一応、効果のほどを示しておきます。ただし、当然ですが全ての環境で同等の効果を保証するものではありません。

### 両テクニック不使用
:::details .zshrcおよびベンチマーク詳細
```sh:.zshrc
eval "$(sheldon source)"

source $HOME/.config/zsh/nonlazy.zsh
zsh-defer source $HOME/.config/zsh/lazy.zsh
zsh-defer unfunction source
```
```sh
❯ hyperfine -w 5 -r 50 'zsh -i -c exit'
Benchmark 1: zsh -i -c exit
  Time (mean ± σ):      29.6 ms ±   1.6 ms    [User: 18.2 ms, System: 11.9 ms]
  Range (min … max):    27.3 ms …  38.3 ms    50 runs
```
:::
平均 29.6ms
標準偏差 1.6ms

### sheldonの出力をキャッシュするテクニックを使用
:::details .zshrcおよびベンチマーク詳細
```sh:.zshrc
# sheldon cache technique
cache_dir=${XDG_CACHE_HOME:-$HOME/.cache/zsh}
sheldon_cache="$cache_dir/sheldon.zsh"
sheldon_toml="$HOME/.config/sheldon/plugins.toml"
if [[ ! -r "$sheldon_cache" || "$sheldon_toml" -nt "$sheldon_cache" ]]; then
  mkdir -p $cache_dir
  sheldon source > $sheldon_cache
fi
source "$sheldon_cache"
unset cache_dir sheldon_cache sheldon_toml

source $HOME/.config/zsh/nonlazy.zsh
zsh-defer source $HOME/.config/zsh/lazy.zsh
zsh-defer unfunction source
```
```sh
❯ hyperfine -w 5 -r 50 'zsh -i -c exit'
Benchmark 1: zsh -i -c exit
  Time (mean ± σ):      21.1 ms ±   0.8 ms    [User: 13.6 ms, System: 7.9 ms]
  Range (min … max):    19.5 ms …  22.8 ms    50 runs
```
:::
平均 21.1ms
標準偏差 0.8ms

### sourceをオーバーライドして全てをzcompileするテクニックを使用
:::details .zshrcおよびベンチマーク詳細
```sh:.zshrc
# source command override technique
function source {
  ensure_zcompiled $1
  builtin source $1
}
function ensure_zcompiled {
  local compiled="$1.zwc"
  if [[ ! -r "$compiled" || "$1" -nt "$compiled" ]]; then
    echo "\033[1;36mCompiling\033[m $1"
    zcompile $1
  fi
}
ensure_zcompiled ~/.zshrc

eval "$(sheldon source)"

source $HOME/.config/zsh/nonlazy.zsh
zsh-defer source $HOME/.config/zsh/lazy.zsh
zsh-defer unfunction source
```
```sh
❯ hyperfine -w 5 -r 50 'zsh -i -c exit'
Benchmark 1: zsh -i -c exit
  Time (mean ± σ):      23.4 ms ±   0.9 ms    [User: 13.9 ms, System: 10.0 ms]
  Range (min … max):    21.9 ms …  25.8 ms    50 runs
```
:::
平均 23.4ms
標準偏差 0.9ms

### 両テクニックを使用
:::details .zshrcおよびベンチマーク詳細
```sh:.zshrc
# source command override technique
function source {
  ensure_zcompiled $1
  builtin source $1
}
function ensure_zcompiled {
  local compiled="$1.zwc"
  if [[ ! -r "$compiled" || "$1" -nt "$compiled" ]]; then
    echo "\033[1;36mCompiling\033[m $1"
    zcompile $1
  fi
}
ensure_zcompiled ~/.zshrc

# sheldon cache technique
cache_dir=${XDG_CACHE_HOME:-$HOME/.cache/zsh}
sheldon_cache="$cache_dir/sheldon.zsh"
sheldon_toml="$HOME/.config/sheldon/plugins.toml"
if [[ ! -r "$sheldon_cache" || "$sheldon_toml" -nt "$sheldon_cache" ]]; then
  mkdir -p $cache_dir
  sheldon source > $sheldon_cache
fi
source "$sheldon_cache"
unset cache_dir sheldon_cache sheldon_toml

source $HOME/.config/zsh/nonlazy.zsh
zsh-defer source $HOME/.config/zsh/lazy.zsh
zsh-defer unfunction source
```
```sh
❯ hyperfine -w 5 -r 50 'zsh -i -c exit'
Benchmark 1: zsh -i -c exit
  Time (mean ± σ):      16.6 ms ±   0.8 ms    [User: 10.2 ms, System: 7.0 ms]
  Range (min … max):    15.5 ms …  19.3 ms    50 runs
```
:::
平均 16.6ms
標準偏差 0.8ms

## まとめのようなもの
今回の記事を書くにあたって、私自身かなり勉強になりました。記事を書いていると記事の範囲に限らず次々と改善点が浮かび、執筆前後でzshrcはかなり変化しました。

私の環境はもともとsheldonとzsh-deferによる遅延ロードを使用しており十分に高速でした。しかし今回、両テクニックを使用することでさらに10ms以上高速化に成功しています。
ぜひ試してみてください。
