---
layout: post
title:  "型クラスってなんだろう"
date:   2014-07-05 18:54:00
categories: [Java,Scala,Haskell]
---

Scalaにおける型クラスの実装例とその意義について [猫型さんのブログ記事その1](http://nekogata.hatenablog.com/entry/2014/06/30/062342), [その2](http://nekogata.hatenablog.com/entry/2014/07/01/184246) を読んで「なるほどなあ」と思ったものの、なんかぼんやりと自分でも思うところがあるので少しつぶやいてたら自分のまとめを書きたくなったので書いてみます。

## 概要

自分が思っている以下の要素をざっくり説明できればな、と思っています。

- データ型の持つ側面や責務(responsibility)を表現するために型クラスを使う。（インターフェースとの類似）
- 型クラスは宣言的。(<=>命令的)
- 型クラスの参照透明性と数学的構造

## 型クラスと型クラスのインスタンス

僕の中で「型クラス」ってワードが出てきたのがHaskellを勉強している時のこと。
ここでは一旦、JavaやScalaとは離れて考えます。

まず、型(type)と言うのは、あるまとまったデータの構造と種類に名前を付けるものです。
オブジェクト指向の「クラス(class)」などと比較されるものですが、Haskellでは「データ構造と種類」のみを表現します。正確には代数データ型でしょうか。

その型がどのような構造や側面を持っているか、というものを「ある関数を適用できる（適用した場合の挙動を規定する）」と規定するときに型クラス(type class)を定義します。Haskellだと例えば次のようなものがあります。

- `Eq` : 2つの値が等しいか判定できる。`==` 関数を適用可能。
- `Ord`: 2つの値の大小を比較できる。`>`, `>=` 関数を適用可能。
- `Functor` : 「ファンクタ則」を持つコンテナ型のデータ構造。`fmap` 関数を適用可能。
- `Monad` : 「モナド則」を持つコンテナ型のデータ構造。 `return`, `>>=` 関数を適用可能。

 `instance Eq Integer` と宣言することで、 `Integer` は「 `Eq` 型クラスのインスタンス」と宣言することが出来ます。つまり、「`Integer`型のデータには `==` が適用できる」と記述したことになります。

型クラスにも階層構造があり、例えば `Ord` 型クラスは `Eq` 型クラスのサブクラスです。

## インターフェースとしての型クラス

型クラスは「ある関数が型クラスのインスタンスに定義されている」という意味を持つので、Javaの視点ではインターフェース(interface)とみなすことが出来ます。

先ほどインスタンスとして宣言することができる、と書きましたが、インスタンス化するには実装を補う必要があるので、型クラスのインスタンス定義では必要な関数に対する実装を行います。
 `Eq` の例では `==` という関数は実装されていないのでインスタンス宣言時に実装を記述しますが、 `/=`(not equal, ≠)は「 `==` の否定」と定義されていますので、実装は必要ありません（というか出来ません）。

Javaでは`Object#equals` が具体的な`Eq`の実装のように見えますが、これはクラスを定義するときに決めなければいけないため、宣言的ではありません。任意のJavaインターフェースは先に定義する必要があり、サブクラスでしか実装を定義することが出来ないのです。

余談ですが、Javaでは最上位クラスのObjectに簡易な実装が定義されており、具象クラスでそれぞれ再実装することになっていますが、どのクラスが `equals` をそれぞれのクラスに合わせて正しく再実装したかどうかわからないので安易に`equals`を使うことが出来ず混乱の元になります。
一方、`Eq`型クラスのインスタンスであれば、等号が実装されていることが明確になります。

### 型クラスは宣言的

Javaのインターフェースとの違いは宣言的であることです。クラス定義の際に`A implements B` と記述するのではなく、定義済みの型に対して「この型はある関数が適用可能である」と追加するのです。

### Scalaでの型クラスの実装例

では、Scalaで改めて`Eq`を`Int`に対して定義してみましょう。（注:これは通常の`==`よりかなり貧弱です）

```scala
trait Eq[T] {
  def eql(that: T): Boolean // abstract
  def neq(that: T) = ! this.eql(that) // implemented
}

implicit class IntEq(i: Int) extends Eq[Int] {
  def eql(that: Int) = i == that
}

1 eql 1 //=> true
1 eql "1" //=> false
```

一足飛びにいきなり `implicit class` が出てきましたが、「`Int`型だったら`IntEq`型とみなすことが出来て、`Eq[Int]`を実装しています」という意味になります。

`implicit class` 自体はScala2.10から定義されているシンタックスシュガーで、実際にはラッパークラス(`IntEq`)と、ラッパークラスへの変換を行う関数(`Int => IntEq`)の暗黙的な定義が行われています。

```scala
class IntEq(i: Int) extends Eq[Int] {
  def eql(that: Int) = i == that
}
implicit val toIntEq = (i: Int) => new IntEq(i)
```

これでIntを拡張出来ました。他のクラスも同様に実装すればよいです。

```scala
implicit class StringEq(s: String) extends Eq[String] {
  def eql(that: String) = s == that
}
```

この`eql`の適用範囲は同じ型同士です。通常JavaやScalaの等号は異なるクラス間でも定義されていますが、この`eql`を使うときは異なる型を比較しようとするとコンパイル時エラーになります。

### 型クラスを利用する関数の定義

型クラスがインターフェースなら、インターフェースを利用する関数もかけるはずです。
Scalaでは次のように定義できます。

```scala
def isSame[T <% Eq[_]](a: T, b: T) = a eql b
```

この関数で`T <% Eq[_]`とは `Eq[_]` とみなせること、つまり`Eq`型クラスのインスタンスであることを条件にしていて、型クラスを実装していなければコンパイル時にエラーとなります。

実際にはこの記法はシンタックスシュガーで、REPLで上記関数を定義すると以下の様に関数の型が表示されます。

```
isSame: [T](a: T, b: T)(implicit evidence$1: T => Eq[_])Boolean
```

ここで implicit parameter(evidence)として、このような関数の存在が暗黙的に定義されていることを要求しています。

つまり次のように実装を書き直すことが出来ます。

```scala
def isSame[T](a: T, b: T)(implicit ev: T => Eq[_]) = ev(a).eql(b)
```

## 参照透明な構造の宣言としての型クラス

「既存のクラスにメソッドを追加している」という視点ではモンキーパッチの一種だと思いますが、私としてはモンキーパッチという聞こえの悪い用語で扱うべきではないと思っています。

関数型プログラミングとして忘れてはいけないのが、参照透明であることです。
関数に同じxを入力したら、必ず同じyが返るように定義すべきであり、これは型クラスに関しても同様です。

Haskellでは（実装者依存とはいえ）型クラスのインスタンスは定義された関数に対して決められた法則を常に満たすべきであるとしています。

例えば `Eq` では以下を満たすべきです。(JUnitっぽい書き方にしてみました)

- `assertTrue(a == a)`
- `assertTrue((a == b) == (b == a))`
- `if ((a == b) == true && (b == c) == true) assertTrue(a == c)`

これは数学における等号が満たすべき法則ですね。[等式 - Wikipedia](http://ja.wikipedia.org/wiki/%E7%AD%89%E5%BC%8F)

既に述べたように、「ファンクタ則」や「モナド則」があり、これらも同じく代数的な性質を保つように設計されています。
これらの法則は特別難しいものではありません。例えばファンクタ則は「関数`f`が`f(a) == b`となるときは、ファンクタ`F`というコンテナでラッピングした関数`F(f)`と値`F(a)`についても`F(f)(F(a)) == F(b)`を満たしている」というもので、要は「ラッビングする前の構造を保ったままにする」ことを満たしている必要があります。

これらの法則を満たしながら設計することが守られる限り、型の持つ内部的な構造を型クラスという側面から予測することが可能になります。

## 蛇足 : データと振る舞いの分離

データの持つ構造と振る舞いを分離するのに、型クラスは重要じゃないかと思っています。

ざっくりとイメージを書いてみました。

<div class="mxgraph" style="position:relative;overflow:hidden;width:634px;height:616px;"><div style="width:1px;height:1px;overflow:hidden;">1Zpfc+I2EMA/DTPtQzv+S8jjkcu1N3Oddkpm2j4KrGBNhEVlkcB9+pPkXfxHkJCBCJsXrJW9kn5arVZrj+K71fY3Sdb5HyKjfBQF2XYUfx5FURhOIv1nJLtKMg6SSrCULIObasGMfacgDEC6YRktWzcqIbhi67ZwIYqCLlRL9ih4u4k1WaL6WjBbEO5K/2GZyivpJBrX8t8pW+bYTDi+rWpKtUMdGX0kG65+sSIY/IqgLhjVNqiKyaQq76AcApw1KVo9+i7EqiWQtKxRwWgZdAvamAuZUdkScVY8NbHF93rqpBD6QXO12t5RbqYPZ6Z67MuR2j0tSYtW08ceiGG0z4RvoO8OPik2RUbNA+Eonr7kTNHZmixM7Yu2MC17FIUCO4kSU2ac3wkuzFALUWj5NCNlDjqqBqlUFE3S7XXNQpsxFSuq5M5MEtSOgR5YcBjAMF5qE4liuCdvmIf+VUICzJd73TUkfQGcjjADCxsWswQt7irMQEUD2Vfd7bOohWi6Z1G57VCJYocK3tKEgqDOsiOHyUxJVix7iCUKoOwBC/jbBpZvrFSjdPowSqu11jM2E39sUofN/f8OkjIna3NJt7rlaSY2c1vji8/eVl6xnZsP4gNRQYPPnzLrHSAcPfocHLkHQKHrdL4qKokh0DdMnU0+GoP1e1hn2NsGpoedhhEFd5yUpYPq7Q3+fB7dmMelsQ+LmjgSuO+snQr0vhbx0CL7JKV40aU5F4snTYBumfpXl3Ukba//M9e/pqZU6B7Yqrpoaw8ERlrxFx0b2VrbKM2cI0OHoe6Y2Eg7GbVTUEQuKdxlmbikJeVEsee29rPAuXb0GjiI+5CbWWnHuLWo2boD9nYBUra/Pki5EeLASFnP6oOU68L7TArcT5OU1eSDlBtK9pkU7s1XMSo3shwaKl9WlbibYTM20Jkx4x3iLdFD/elvWq5FUbI540zpZrT2YEpz8syE/NkhrIMDE2oRzpYm2bTQCEy6aGriBqZzYp+gYsWyzDwz/YBcCkaeb6UFMHN0Fkp3e/xMFNESQ7QFUh+NNwu1kbQf1LrZFK/UTtkq9YKaQREXq7N8u5FWqaR42qdZYYm9vTRdSM2YFE9/TQp74cmLE9r4SzCTNsJZSNuTEKMpo4bKZcBDzSxoR88Yj/DHFFV+xlFkZ2o/7tMmz9294UgRPEjqHsKucLKIDxhzeOhAeomjBerodUoj7qz2BPcADyf2ZBg5DSQChFJ8WeSD0M1gchoJjhcNyWPuEF9s9Tex2smLjX0mnd13Or1JxscB+MgrcMEdtncvbsL0ilDel9Vx0mHVCQnPS6+kwwzMi6fDcEpbZye7yXg4PKXvS/P0DB16iYuiOxyShjcd+8YN9b2xbTRpK0px/i8f26ZuqDKkyQW31JpcX4ni1I1hho7OjsgHuhM+YRmYS/GG7oQvWXqM7kAS/qMWrC7WH2VVrrH+uC6+/wE=</div></div>

インターフェースを見て振る舞いを予想しそのクラスを実装に利用する、という視点では同じことだと思いますが、実装自体が型の定義に含まれるかどうかが異なります。

<script type="text/javascript" src="https://www.draw.io/embed.js"></script>
