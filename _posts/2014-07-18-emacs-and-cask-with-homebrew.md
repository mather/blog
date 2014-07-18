---
layout: post
title:  Homebrew で emacs & cask
date:   2014-07-18 21:45:00
categories: [Homebrew,emacs]
---

## どうでもいい話

emacs使ってますか？

周りは割と「vimでいいんじゃね？」とか「IDE必須」なんて人も多いと思いますが、

- コマンドモードとエディタモードの切替ってめんどくさくね？日本語切り替えもあったらいちいちコマンドモードに切り替える度に日本語変換外したり…。
- IDEとかメモリバカ食いだし、クラスパス勝手に追加してたりして気持ち悪くね？

なんてことを思っていた時期が僕にもありました…(ScalaはIntellij IDEA使ってます)。

でも、`C-a`, `C-e` がターミナルの標準キーバインドと同じなのは共感できませんか？

## emacs と Emacs.app

なんて話題はさておき、言わずと知れたテキストエディタ(?)であるemacsは次のようにしてインストール可能です。

```
$ brew install emacs
```

インストール後は `emacs` コマンドが使えるようになります。この `emacs` はCUI版(`emacs -nw`)に相当するため、コンソール画面でフォアグラウンドで実行されます。

では、別ウィンドウで開くにはどうするかというと、インストール時に既に `/Applications/Emacs.app` にリンクが作成されているので、これを実行します。

```
$ open -a Emacs
```

CUI版の`emacs`は存在しないファイルを指定しても作成してくれるのに対し、app版は「そんなファイルはない」と冷静に返してきます。`open`コマンドで決められている引数の渡し方に従う必要があります。

```
$ open -a Emacs --args hoge.txt
```

## Caskでパッケージの管理

せっかくなので、caskというツールでライブラリパッケージの導入をしましょう。
ちなみに、Homebrew-Caskとは無関係です。

```
$ brew install cask
```

通常caskを導入する場合は、emacsのライブラリ読み込みパスに`cask.el`を追加して…などの設定項目が必要なのですが、Homebrewで入れた場合はパスの指定が必要ありません。

caskの依存関係でそもそもemacsは入るものなので、`/usr/local/share/emacs/site-lisp/`以下に`cask.el`のリンクが作成されています。

したがって、 `~/.emacs.d/init.el` に以下を記述すると利用可能になります。

```lisp
(require 'cask)
(cask-initialize)
```

ここまではemacsがcaskが利用できるようになるまでのプロセスですが、caskにはパッケージダウンロード機能があります。
`cask init`コマンドでCaskファイルを作成し、編集しましょう。

```
$ cd ~/.emacs.d/
$ cask init
$ emacs Cask
```

おそらく最初からすごい勢いでパッケージリストが生成されています。
これをサンプルとして、例えば`markdown-mode`を追加してみましょう。

```lisp
(depends-on "markdown-mode")
```

このように必要なパッケージリストを構成するだけです。
実際のダウンロードは`cask install`を実行します。

```
$ cask install
```

リモートサーバからのダウンロードが実行され、`.cask/<version>/elpa`などに保存されます。先ほど記述した `(cask-initialize)` をロードするときに、これらのパスが自動的に補完されるので、`load-path`の設定は不要です。

実際にemacsを起動してmarkdown-modeにしてみましょう。

```
M-x markdown-mode
```

なお、どのようなパッケージがあるかは、`M-x list-packages`で確認できます。
このパッケージリストからもインストールは可能なのですが、Caskファイルに記述してインストールするようにすることで、複数の端末間で同じ Caskファイルによるパッケージ管理が可能になります。積極的に活用しましょう。
