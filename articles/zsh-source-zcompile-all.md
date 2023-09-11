---
title: "souceコマンドをオーバーライドして全てをzcompileする"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [zsh]
published: true
---
## ダイレクトマーケティング
この記事は同時公開の記事

https://zenn.dev/fuzmare/articles/zsh-plugin-manager-cache

の一部として作成しようとしていたものですが、独立したトピックとして成立しているので記事を分割したものです。
本記事で紹介したテクニックともう一つのテクニックを組み合わせることでzshの起動時間を大幅に高速化できます。是非こちらも読んでください。

## 概要
- sourceコマンドをオーバーライドしてsourceする時になんでもかんでもzcompileしてしまおう

## zcompile is 何
`zcompile`はzshの組み込みコマンドで、関数またはスクリプト全体をコンパイルします。manに「テキストのパースを避けることで、関数のautoloadやスクリプトのsourceを高速化する」というようなことが書いてあるので、ソースコードのテキストを解析した中間表現が出力されているのだと思います。

これを使って.zshrcをコンパイルしている例はよく見ます。しかし、ファイルを分割していたり、プラグインがあったりすると.zshrcをコンパイルしただけでは足りません。可能であれば読み込まれる全てを`zcompile`したいものです。

# souceコマンドをオーバーライドして全てをzcompileする
souceコマンドをオーバーライドして全てをzcompileするには次のようにsourceコマンドを再定義します。

```sh:.zshrc
function source {
  zcompile $1
  builtin source $1
}
```

基本のコードはこれだけです。
本来の`source`を乗っ取って、`zcompile`と本来の(ビルトインの)`source`を行っています。
ただし、いつもこれが機能していては寧ろ遅くなってしまうので、必要のないときは機能しないようにする必要があります。

## 実用的にしよう
というわけで、ファイルが更新されていたら(元ファイルがzwcファイルより新しくなっていたら)それをzcompileするという関数`ensure_zcompiled`を用意してこれを毎回呼ぶようにしました。

```sh:.zshrc
function source {
  ensure_zcompiled $1
  builtin source $1
}
function ensure_zcompiled {
  local compiled="$1.zwc"
  if [[ ! -r "$compiled" || "$1" -nt "$compiled" ]]; then
    echo "Compiling $1"
    zcompile $1
  fi
}
ensure_zcompiled ~/.zshrc
```

処理時間の増加は誤差に埋もれる程度で、測定しても増えているのかよく分からない程度でした。
これでsourceされるあれこれを自動的にzcompileすることができるようになりました。
.zshrcはオーバーライドしたsourceで読み込まれる前に読まれているので、わざわざ `ensure_zcompiled` に投げています。

zshが起動した後はこのsource乗っ取りをやめたいという場合は.zshrc末尾に

```sh:.zshrc
unfunction source
```

をつければOKです。

## まとめのようなもの
zcompileを使用することによりzshの起動を高速化することができます。
なので全部をzcompileすればさらに高速化することができます。
全部がzcompileされたzshは速いです。
以上です。

## こちらも読んで

https://zenn.dev/fuzmare/articles/zsh-plugin-manager-cache

