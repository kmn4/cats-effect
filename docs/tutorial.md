---
id: tutorial
title: Tutorial
---

## Introduction

This tutorial tries to help newcomers to cats-effect to get familiar with its
main concepts by means of code examples, in a learn-by-doing fashion. Two small
programs will be coded, each one in its own section. [The first
one](#copyingfiles) copies the contents from one file to another, safely
handling _resources_ and _cancelation_ in the process. That should help us to
flex our muscles. [The second one](#producerconsumer) implements a solution to
the producer-consumer problem to introduce cats-effect _fibers_.

このチュートリアルはcats-effectが初めての人が主要な概念に慣れ親しむことができるように、
コード例を使って補助することを、learn-by-doingな方法で試みます。
2つの小さなプログラムが、それぞれ1つの節を使ってコーディングされます。
[1つ目](#copyingfiles)はあるファイルの中身を別のファイルにコピーしますが、
その過程で_リソース_と_キャンセル_を安全に扱います。
肩慣らしの助けになるはずです。
[2つ目](#producerconsumer)はproducer-consumer問題の解を実装することで、
cats-effectの_fiber_を導入します。

This tutorial assumes some familiarity with functional programming. It is
also a good idea to read the cats-effect documentation prior to starting this
tutorial, at least the [Getting Started page](getting-started.md).

このチュートリアルは関数型プログラミングにいくらか親しみがあることを仮定します。
このチュートリアルを始める前に、cats-effectのドキュメント、
少なくとも[Getting Started ページ](getting-started.md)を読むのもいいアイデアでしょう。

Please read this tutorial as training material, not as a best-practices
document. As you gain more experience with cats-effect, you will probably find
your own solutions to deal with the problems presented here. Also, bear in mind
that using cats-effect for copying files or implementing basic concurrency
patterns (such as the producer-consumer problem) is suitable for a 'getting
things done' approach, but for more complex systems/settings/requirements you
might want to take a look at [fs2](http://fs2.io) or [Monix](https://monix.io)
to find powerful network and file abstractions that integrate with cats-effect.
But that is beyond the purpose of this tutorial, which focuses solely on
cats-effect.

どうかこのチュートリアルをトレーニング資料として読んでください。
ベストプラクティスドキュメントではなくて。
cats-effectの経験を積むにしたがって、
ここで提示される問題を処理するためのあなた自身の解をおそらくは見つけるでしょう。
また、cats-effectをファイルコピーや（produce-consumer問題のような）基礎的な並行性パターンの
実装に使うのは 'getting things done' アプローチに適しているものの、
より複雑なシステム/状況/要件に対しては、[fs2](http://fs2.io) や [Monix](https://monix.io)
を確認して、cats-effectと統合された強力なネットワークとファイルの抽象化を見つけたいと思うかもしれません。
しかしこれらはこのチュートリアルの目的を超えます。cats-effectのみに集中したいので。

That said, let's go!

前置きがすんだところで、それじゃあ始めましょうか！

## Setting things up

This [Github
repo](https://github.com/lrodero/cats-effect-tutorial/tree/series/3.x) includes
all the software that will be developed during this tutorial (branch
`series/3.x`). It uses `sbt` as the build tool. To ease coding, compiling and
running the code snippets in this tutorial, it is recommended to use the same
`build.sbt`, or at least one with the same dependencies and compilation options:

この[Githubレポジトリ](https://github.com/lrodero/cats-effect-tutorial/tree/series/3.x)
はこのチュートリアルで開発される全てのソフトウェアを含んでいます (ブランチ`series/3.x`)。
`sbt`をビルドツールとして使用します。
このチュートリアルのコードスニペットを楽にコーディング・コンパイル・実行するために、
同じ`build.sbt`か、少なくとも同じ依存性とコンパイルオプションなものを使うことをお勧めします：

```scala
name := "cats-effect-tutorial"

version := "3.3.8"

scalaVersion := "2.13.6"

libraryDependencies += "org.typelevel" %% "cats-effect" % "3.3.8" withSources() withJavadoc()

scalacOptions ++= Seq(
  "-feature",
  "-deprecation",
  "-unchecked",
  "-language:postfixOps"
)
```

Also make sure that you use a recent version of `sbt`, at least `1.4.2`. You can
set the `sbt` version in `project/build.properties` file:

最近のバージョンの`sbt`、少なくとも`1.4.2`以降のものを使うようにもしてください。
`sbt`バージョンは`project/build.properties`ファイルで設定できます：

```scala
sbt.version=1.4.2
```

Almost all code snippets in this tutorial can be pasted and compiled right in
the scala console of the project defined above (or any project with similar
settings).

このチュートリアルのほぼ全てのコードスニペットは、
上で定義した（または似た設定の）プロジェクトのScalaコンソール内に
ペーストしてコンパイルすることができます。

## <a name="copyingfiles"></a>Copying files - basic concepts, resource handling and cancelation

ファイルのコピー - 基礎概念、リソースの扱いとキャンセル

Our goal is to create a program that copies files. First we will work on a
function that carries out such a task, and then we will create a program that can be
invoked from the shell and uses that function.

私達の目標は、ファイルをコピーするプログラムを作成することです。
私達はまずそのようなタスクを実行する関数に取り組み、
その次にシェルから呼び出されてその関数を使うプログラムを作成します。

First of all we must code the function that copies the content from a file to
another file. The function takes the source and destination files as parameters.
But this is functional programming! So invoking the function shall not copy
anything, instead it will return an `IO` instance that encapsulates all the side
effects involved (opening/closing files, reading/writing content), that way
_purity_ is kept.  Only when that `IO` instance is evaluated all those
side-effectful actions will be run. In our implementation the `IO` instance will
return the amount of bytes copied upon execution, but this is just a design
decision. Of course errors can occur, but when working with any `IO` those
should be embedded in the `IO` instance. That is, no exception is raised outside
the `IO` and so no `try` (or the like) needs to be used when using the function,
instead the `IO` evaluation will fail and the `IO` instance will carry the error
raised.

まず、あるファイルから別のファイルへ内容をコピーする関数をコーディングしなければなりません。
この関数はソースとデスティネーションのファイルをパラメータとして受けとります。
しかし今やっているのは関数型プログラミングです！
なのでこの関数の呼び出しは何もコピーをしてはならず、
代わりに全ての副作用をカプセル化した`IO`インスタンスを返します
（ここでの副作用は、ファイルを開いたり閉じたりすること、
その中身を読んだり書き込んだりすることを指します）。そうすると_純粋性_が保たれます。
その`IO`インスタンスが評価されたときのみ、それら全ての副作用的なアクションが実行されます。
私達の実装ではこの`IO`インスタンスは実行時にコピーされたバイト数を返しますが、
これは単に設計上の決定でしかありません。
もちろんエラーは起こりうるのですが、
どんな`IO`を使うときもそれらは`IO`インスタンスタンスに埋め込まれるべきです。
つまり、`IO`の外で例外が発生することはなく、
したがってこの関数を使うにあたって`try`のようなものは一切使う必要がありません。
代わりに`IO`の評価が失敗して、`IO`インスタンスが発生したエラーを持つことになります。

Now, the signature of our function looks like this:

いま、私達の関数のシグネチャはこんな感じになります：

```scala mdoc:compile-only
import cats.effect.IO
import java.io.File

def copy(origin: File, destination: File): IO[Long] = ???
```

Nothing scary, eh? As we said before, the function just returns an `IO`
instance. When run, all side-effects will be actually executed and the `IO`
instance will return the bytes copied in a `Long` (note that `IO` is
parameterized by the return type). Now, let's start implementing our function.
First, we need to open two streams that will read and write file contents.

何も怖いことはありませんね？
前に言ったとおり、この関数は単に`IO`インスタンスを返します。
走らせたら、全ての副作用が実際に実行されて、
`IO`インスタンスはコピーされたバイト数を`Long`の中に返します
（`IO`が返却値の型によってパラメタライズされていることに注意してください）。
それでは、私達の関数の実装を始めましょう。
まず、ファイルの中身を読み書きするための2つのストリームを開く必要があります。

### Acquiring and releasing `Resource`s

`Resource`の取得と開放

We consider opening a stream to be a side-effectful action, so we have to
encapsulate those actions in their own `IO` instances. We can just embed the
actions by calling `IO(action)`, but when dealing with input/output actions it
is advised to use instead `IO.blocking(action)`. This way we help cats-effect to
better plan how to assign threads to actions. We will return to this topic when
we introduce _fibers_ later on in this tutorial.

ストリームを開くことは副作用を伴うアクションだと考えられるので、
これらのアクションはそれ自身`IO`インスタンスにカプセル化する必要があります。
アクションを`IO(action)`の形の呼び出しで埋め込むこともできるのですが、
入出力アクションを扱うときは代わりに`IO.blocking(action)`を使うほうが良いです。
こうすると、どうやってスレッドをアクションに割り当てるか、cats-effectが
より良い計画を立てるための助けになります。
このチュートリアルの後の方で_ファイバー_を導入するとき、またこの話題に戻ってきます。

Also, we will make use of cats-effect `Resource`. It allows to orderly create,
use and then release resources. See this code:

それから、私達はcats-effectの`Resource`を利用します。
これはリソースの作成、使用、開放を行儀よく行うことを可能にします。
このコードを見てください：

```scala mdoc:compile-only
import cats.effect.{IO, Resource}
import java.io._ 

def inputStream(f: File): Resource[IO, FileInputStream] =
  Resource.make {
    IO.blocking(new FileInputStream(f))                         // build
  } { inStream =>
    IO.blocking(inStream.close()).handleErrorWith(_ => IO.unit) // release
  }

def outputStream(f: File): Resource[IO, FileOutputStream] =
  Resource.make {
    IO.blocking(new FileOutputStream(f))                         // build 
  } { outStream =>
    IO.blocking(outStream.close()).handleErrorWith(_ => IO.unit) // release
  }

def inputOutputStreams(in: File, out: File): Resource[IO, (InputStream, OutputStream)] =
  for {
    inStream  <- inputStream(in)
    outStream <- outputStream(out)
  } yield (inStream, outStream)
```

We want to ensure that streams are closed once we are done using them, no matter
what. That is precisely why we use `Resource` in both `inputStream` and
`outputStream` functions, each one returning one `Resource` that encapsulates
the actions for opening and then closing each stream.  `inputOutputStreams`
encapsulates both resources in a single `Resource` instance that will be
available once the creation of both streams has been successful, and only in
that case. As seen in the code above `Resource` instances can be combined in
for-comprehensions. Note also that when releasing resources we must also take
care of any possible error during the release itself, for example with the
`.handleErrorWith` call as we do in the code above.  In this case we just
ignore the error, but normally it should be at least logged. Often you will see
that `.attempt.void` is used to get the same 'swallow and ignore errors'
behavior.

私達は、なにがなんでも、使い終わったらストリームが閉じられることを保証したいです。
これこそ、私達が`Resource`を関数`inputStream`と`outputStream`の両方で使っている理由であり、
各関数はストリームの開閉アクションをカプセル化した`Resource`を一つ返します。
`inputOutputStreams`は両方のリソースを一つの`Resource`インスタンスにカプセル化します。
このインスタンスは両方のストリームが無事に作成されたとき、またそのときに限り利用可能になります。
上のコードで見られるように、`Resource`インスタンスはfor内包表記で合成できます。
リソースを解放するとき、解放操作それ自身の間に発生しうるエラーのケアも
しなければいけないことにも注意してください。
例えば上のコードでやっているように、`.handleErrorWith`呼び出しによってケアします。
この場合は単にエラーを無視しているのですが、通常であれば少なくともログを取るべきです。
しばしば、`.attempt.void`が同じように「エラーを飲み込んで無視する」振る舞いに使われているところを
皆さんも目にするでしょう。

Optionally we could have used `Resource.fromAutoCloseable` to define our
resources, that method creates `Resource` instances over objects that implement the
`java.lang.AutoCloseable` interface without having to define how the resource is
released. So our `inputStream` function would look like this:

任意ですが、`Resouce.fromAutoCloseable`を使ってリソースを定義することもできたでしょう。
このメソッドは`java.lang.AutoCloseable`インターフェースを実装するオブジェクト上の
`Resource`インスタンスを、リソースが解放される方法を定義する必要なく作成します。
そのため`inputStream`関数は次のようになったでしょう。

```scala mdoc:compile-only
import cats.effect.{IO, Resource}
import java.io.{File, FileInputStream}

def inputStream(f: File): Resource[IO, FileInputStream] =
  Resource.fromAutoCloseable(IO(new FileInputStream(f)))
```

That code is way simpler, but with that code we would not have control over what
would happen if the closing operation throws an exception. Also it could be that
we want to be aware when closing operations are being run, for example using
logs. In contrast, using `Resource.make` allows to easily control the actions
of the release phase.

このコードのほうが遥かにシンプルですが、
このコードでは閉じる操作が例外を投げたときに何が起こるかコントロールすることができません。

Let's go back to our `copy` function, which now looks like this:

`copy`関数の話に戻りましょう。これは次のようになります：

```scala mdoc:compile-only
import cats.effect.{IO, Resource}
import java.io._

// as defined before
def inputOutputStreams(in: File, out: File): Resource[IO, (InputStream, OutputStream)] = ???

// transfer will do the real work
def transfer(origin: InputStream, destination: OutputStream): IO[Long] = ???

def copy(origin: File, destination: File): IO[Long] = 
  inputOutputStreams(origin, destination).use { case (in, out) => 
    transfer(in, out)
  }
```

The new method `transfer` will perform the actual copying of data, once the
resources (the streams) are obtained. When they are not needed anymore, whatever
the outcome of `transfer` (success or failure) both streams will be closed. If
any of the streams could not be obtained, then `transfer` will not be run. Even
better, because of `Resource` semantics, if there is any problem opening the
input file then the output file will not be opened.  On the other hand, if there
is any issue opening the output file, then the input stream will be closed.

新しいメソッド`transfer`は、リソース（ストリーム）が取得できたら、実際のデータコピーを行います。
ストリームがもう必要なくなったら、`transfer`の結果（成功または失敗）によらず、それらは閉じられます。
もしストリームのどちらかでも取得ができなかったら、そのときは`transfer`が走りません。
もっと良いことに、`Resource`のセマンティクスのおかげで、
入力ファイルのオープンで問題が生じていたら出力ファイルはオープンされません。
一方で、出力ファイルのオープンで問題が生じたら、入力ファイルはクローズされます。

### What about `bracket`?
Now, if you are familiar with cats-effect's `Bracket` you may be wondering why
we are not using it as it looks so similar to `Resource` (and there is a good
reason for that: `Resource` is based on `bracket`). Ok, before moving forward it
is worth taking a look at `bracket`.

いま、もしあなたがcats-effectの`Bracket`に親しみがあれば、
なぜそれを使わないんだろうか、と考えているかもしれません。
`Resource`に似ていますからね（そう考える良い理由もあります：`Resource`は`bracket`に基づいています）。
いいでしょう、次に進む前に、`bracket`について見ておく価値はあります。

There are three stages when using `bracket`: _resource acquisition_, _usage_,
and _release_. Each stage is defined by an `IO` instance.  A fundamental
property is that the _release_ stage will always be run regardless whether the
_usage_ stage finished correctly or an exception was thrown during its
execution. In our case, in the _acquisition_ stage we would create the streams,
then in the _usage_ stage we will copy the contents, and finally in the release
stage we will close the streams.  Thus we could define our `copy` function as
follows:

`bracket`の使用には3つの段階があります：_リソース取得_、_使用_、そして_解放_です。
各段階は`IO`インスタンスによって定義されます。
基本となる性質は、_使用_段階が正しく終わったか実行時に例外が投げられたかによらず、
_解放_段階は常に走る、という点です。
私達のケースでは、_取得_段階でストリームを作成して、次いで_使用_段階で内容をコピーして、
最後に解放ステージでストリームを閉じるでしょう。
したがって私達は`copy`関数を次のように定義することもできました：

```scala mdoc:compile-only
import cats.effect.IO
import cats.syntax.all._ 
import java.io._ 

// function inputOutputStreams not needed

// transfer will do the real work
def transfer(origin: InputStream, destination: OutputStream): IO[Long] = ???

def copy(origin: File, destination: File): IO[Long] = {
  val inIO: IO[InputStream]  = IO(new FileInputStream(origin))
  val outIO:IO[OutputStream] = IO(new FileOutputStream(destination))

  (inIO, outIO)              // Stage 1: Getting resources 
    .tupled                  // From (IO[InputStream], IO[OutputStream]) to IO[(InputStream, OutputStream)]
    .bracket{
      case (in, out) =>
        transfer(in, out)    // Stage 2: Using resources (for copying data, in this case)
    } {
      case (in, out) =>      // Stage 3: Freeing resources
        (IO(in.close()), IO(out.close()))
        .tupled              // From (IO[Unit], IO[Unit]) to IO[(Unit, Unit)]
        .handleErrorWith(_ => IO.unit).void
    }
}
```

The new `copy` definition is more complex, even though the code as a whole is way
shorter as we do not need the `inputOutputStreams` function. But there is a
catch in the code above.  When using `bracket`, if there is a problem when
getting resources in the first stage, then the release stage will not be run.
Now, in the code above, first the origin file and then the destination file are
opened (`tupled` just reorganizes both `IO` instances into a single one). So
what would happen if we successfully open the origin file (_i.e._ when
evaluating `inIO`) but then an exception is raised when opening the destination
file (_i.e._ when evaluating `outIO`)? In that case the origin stream will not
be closed! To solve this we should first get the first stream with one `bracket`
call, and then the second stream with another `bracket` call inside the first.
But, in a way, that's precisely what we do when we `flatMap` instances of
`Resource`. And the code looks cleaner too. So, while using `bracket` directly
has its place, `Resource` is likely to be a better choice when dealing with
multiple resources at once.

新しい`copy`の定義は、`inputOutputStreams`が必要ないので全体としては短くなったものの、より複雑です。
しかし新しい発見もあります。
`bracket`を使うと、もし最初の段階でリソースの取得に問題があれば、解放の段階が走りません。
いま、上記のコードでは、まずオリジンファイルが、次にデスティネーションファイルがオープンされます
（`tupled`は単に両方の`IO`インスタンスを単一のものに再構成しているだけです）。
それでは、オリジンファイルをオープンするとき（_つまり_`inIO`の評価時）には成功して、
デスティネーションファイルのオープン時（_つまり_`outIO`の評価時）には例外が発生したとしたら、
一体何が起きるでしょう？
この場合、オリジンストリームはクローズされません！
これを解決するため、私達はまず最初のストリームをひとつの`bracket`呼び出しで取得し、
次に2つ目のストリームを最初の呼び出し内で別の`bracket`呼び出しをすることで取得します。
しかし、これはある意味ちょうど`Resource`インスタンスの`flatMap`でやっていたことです。
そしてコードはよりクリーンにも見えます。
なので、`bracket`を直接使うとよい場面もある一方で、複数のリソースを一度に扱うときには
`Resource`のほうが良い選択になりがちです。

### Copying data
Finally we have our streams ready to go! We have to focus now on coding
`transfer`. That function will have to define a loop that at each iteration
reads data from the input stream into a buffer, and then writes the buffer
contents into the output stream. At the same time, the loop will keep a counter
of the bytes transferred. To reuse the same buffer we should define it outside
the main loop, and leave the actual transmission of data to another function
`transmit` that uses that loop. Something like:

ついに、ストリームの準備ができました！
いまや`transfer`のコーディングに集中するときです。
この関数はループを定義し、各反復では入力ストリームのデータをバッファに読み込み、
バッファの内容を出力ストリームに書き出さねばならないでしょう。
同時に、このループは輸送されたバイト数のカウンタを管理するでしょう。
バッファは、同じものを再利用するために、メインループの外で定義し、
実際のデータ輸送はループを使うまた別の関数`transmit`に任せるべきです。
次のように：

```scala mdoc:compile-only
import cats.effect.IO
import cats.syntax.all._ 
import java.io._ 

def transmit(origin: InputStream, destination: OutputStream, buffer: Array[Byte], acc: Long): IO[Long] =
  for {
    amount <- IO.blocking(origin.read(buffer, 0, buffer.size))
    count  <- if(amount > -1) IO.blocking(destination.write(buffer, 0, amount)) >> transmit(origin, destination, buffer, acc + amount)
              else IO.pure(acc) // End of read stream reached (by java.io.InputStream contract), nothing to write
  } yield count // Returns the actual amount of bytes transmitted

def transfer(origin: InputStream, destination: OutputStream): IO[Long] =
  transmit(origin, destination, new Array[Byte](1024 * 10), 0L)
```

Take a look at `transmit`, observe that both input and output actions are
created by invoking `IO.blocking` which return the actions encapsulated in a
(suspended in) `IO`. We can also just embed the actions by calling `IO(action)`,
but when dealing with input/output actions it is advised that you instead use
`IO.blocking(action)`. This way we help cats-effect to better plan how to assign
threads to actions. We will return to this topic when we introduce _fibers_
later on in this tutorial.

`transmit`を見て、入出力アクションの両方が、`IO`でカプセル化（中断）されたアクションを返却する
`IO.blocking`呼び出しで作成されることを観察してください。
`IO(action)`を呼び出すことでアクションを単に埋め込むこともできますが、
入出力アクションを扱う場合には代わりに`IO.blocking(action)`を使ったほうが良いです。
こうすると、どうやってスレッドをアクションに割り当てるか、cats-effectが
より良い計画を立てるための助けになります。
このチュートリアルの後の方で_ファイバー_を導入するとき、またこの話題に戻ってきます。

`IO` being a monad, we can sequence our new `IO` instances using a
for-comprehension to create another `IO`. The for-comprehension loops as long as
the call to `read()` does not return a negative value that would signal that the
end of the stream has been reached. `>>` is a Cats operator to sequence two
operations where the output of the first is not needed by the second (_i.e._ it
is equivalent to `first.flatMap(_ => second)`). In the code above that means
that after each write operation we recursively call `transmit` again, but as
`IO` is stack safe we are not concerned about stack overflow issues. At each
iteration we increase the counter `acc` with the amount of bytes read at that
iteration. 

`IO`はモナドなので、私達の新しい`IO`インスタンスをfor内包表記で並べて別の`IO`を作成できます。
for内包表記は`read()`がストリームの終端に達したことを知らせる負の値を返さない限りループします。
`>>`はCatsの演算子で、2つの操作を、1つ目の出力が2つ目に必要ないような場合に、
逐次的に並べるためのものです（_つまり_`first.flatMap(_ => second)`と等価です）。
上のコードでは、各書き出し操作のあとで再帰的に`transmit`を再び呼び出しますが、
`IO`はスタック安全なのでスタックオーバーフロー問題を気にする必要はありません。
各反復では、カウンター`acc`を、その反復で読み出されたバイト数だけ増やします。

We are making progress, and already have a version of `copy` that can be used.
If any exception is raised when `transfer` is running, then the streams will be
automatically closed by `Resource`. But there is something else we have to take
into account: `IO` instances execution can be **_canceled!_** And cancelation
should not be ignored, as it is a key feature of cats-effect. We will discuss
cancelation in the next section.

私達は前へ進んでおり、使えるバージョンの`copy`を既に持っています。
`transfer`が走っているときに何らかの例外が起きると、
ストリームは`Resource`によって自動的にクローズされます。
しかし他にも考慮しなければならないことがあります：`IO`インスタンスの実行は**_キャンセル_**できるのです！
そしてキャンセルはcats-effectの鍵となる機能なので、それが無視されることはありえません。
次の節ではキャンセルについて議論します。

### Dealing with cancelation
Cancelation is a powerful but non-trivial cats-effect feature. In cats-effect,
some `IO` instances can be canceled ( _e.g._ by other `IO` instances running
concurrently) meaning that their evaluation will be aborted. If the programmer is
careful, an alternative `IO` task will be run under cancelation, for example to
deal with potential cleaning up activities.

キャンセルはcats-effectの強力で非自明な機能です。
cats-effectでは、`IO`インスタンスは（_例えば_並行に走っている別の`IO`インスタンスによって）
キャンセルでき、評価は中止されます。
プログラマーが注意深ければ、例えば潜在的なクリーンアップに対処するために、
キャンセルに対して代わりの`IO`タスクが走ることになるでしょう。

Thankfully, `Resource` makes dealing with cancelation an easy task. If the `IO`
inside a `Resource.use` is canceled, the release section of that resource is
run. In our example this means the input and output streams will be properly
closed. Also, cats-effect does not cancel code inside `IO.blocking` instances.
In the case of our `transmit` function this means the execution would be
interrupted only between two calls to `IO.blocking`. If we want the execution of
an IO instance to be interrupted when canceled, without waiting for it to
finish, we must instantiate it using `IO.interruptible`.

ありがたいことに、`Resource`はキャンセルへの対処を簡単にします。
`Resource.use`内の`IO`がキャンセルされたら、そのリソースの解放セクションが走ります。
このことは、私達の例において、入出力ストリームが適切にクローズされることを意味します。
さらに、cats-effectは`IO.blocking`インスタンス内のコードをキャンセルしません。
このことは、関数`transmit`において、実行が割り込まれるのは
`IO.blocking`の呼び出し2つの間に限ることを意味します。
もしIOインスタンスの実行が、キャンセルされたときに完了を待たず割り込まれてほしいなら、
`IO.interruptible`を使ってそのIOインスタンスを作らなければなりません。

### `IOApp` for our final program

最終版プログラムの`IOApp`

We will create a program that copies files, this program only takes two
parameters: the name of the origin and destination files. For coding this
program we will use `IOApp` as it allows to maintain purity in our definitions
up to the program main function.

私達はファイルをコピーするプログラムを作成します。
このプログラムは2つのパラメータしか取りません：オリジンファイルとデスティネーションファイルの名前です。
このプログラムのコーディングのために、私達は`IOApp`を使います。
定義の純粋性をプログラムのメイン関数のレベルまで確保することを可能にするので。

`IOApp` is a kind of 'functional' equivalent to Scala's `App`, where instead of
coding an effectful `main` method we code a pure `run` function. When executing
the class a `main` method defined in `IOApp` will call the `run` function we
have coded. Any interruption (like pressing `Ctrl-c`) will be treated as a
cancelation of the running `IO`.

`IOApp`はScalaの`App`に対する「関数的な」対応物の一つで、
ここではeffectfulな`main`メソッドをコーディングする代わりに純粋な`run`関数を書きます。
このクラスを実行すると、`IOApp`に定義された`main`メソッドは、私達が書いた`run`関数を呼び出します。
どんな割り込み（例えば`Ctrl-c`を押すなど）も実行中の`IO`のキャンセルとして扱われます。

When coding `IOApp`, instead of a `main` function we have a `run` function,
which creates the `IO` instance that forms the program. In our case, our `run`
method can look like this:

`IOApp`をコーディングするとき、私達は`main`関数の代わりに`run`関数を持っており、
この関数はプログラムを成す`IO`インスタンスを作成します。
私達のケースでは、`run`メソッドはこんな感じになるかもしれません：

```scala mdoc:compile-only
import cats.effect._
import java.io.File

object Main extends IOApp {

  // copy as defined before
  def copy(origin: File, destination: File): IO[Long] = ???

  override def run(args: List[String]): IO[ExitCode] =
    for {
      _      <- if(args.length < 2) IO.raiseError(new IllegalArgumentException("Need origin and destination files"))
                else IO.unit
      orig = new File(args(0))
      dest = new File(args(1))
      count <- copy(orig, dest)
      _     <- IO.println(s"$count bytes copied from ${orig.getPath} to ${dest.getPath}")
    } yield ExitCode.Success
}
```

Heed how `run` verifies the `args` list passed. If there are fewer than two
arguments, an error is raised. As `IO` implements `MonadError` we can at any
moment call to `IO.raiseError` to interrupt a sequence of `IO` operations. The log
message is printed by means of handy `IO.println` method.

`run`が渡された`args`リストを検証する方法に注意してください。
引数が2つより少ないときはエラーが発生します。
`IO`は`MonadError`を実装しているので、私達はいつでも`IO.raiseError`を呼び出して
`IO`操作の逐次実行に割り込むことができます。
ログメッセージは便利な`IO.println`メソッドによってプリントされます。

#### Copy program code
You can check the [final version of our copy program
here](https://github.com/lrodero/cats-effect-tutorial/blob/series/3.x/src/main/scala/catseffecttutorial/copyfile/CopyFile.scala).

コピープログラムの最終版は[ここ](https://github.com/lrodero/cats-effect-tutorial/blob/series/3.x/src/main/scala/catseffecttutorial/copyfile/CopyFile.scala)で確認できます。

The program can be run from `sbt` just by issuing this call:

プログラムは`sbt`から次の呼び出しをするだけで実行できます：

```scala
> runMain catseffecttutorial.CopyFile origin.txt destination.txt
```

It can be argued that using `IO{java.nio.file.Files.copy(...)}` would get an
`IO` with the same characteristics of purity as our function. But there is a
difference: our `IO` is safely cancelable! So the user can stop the running code
at any time for example by pressing `Ctrl-c`, our code will deal with safe
resource release (streams closing) even under such circumstances. The same will
apply if the `copy` function is run from other modules that require its
functionality. If the `IO` returned by this function is canceled while being
run, still resources will be properly released.

`IO{java.nio.file.Files.copy(...)}`を使えば私達の関数の純粋性と同じ特性を持った`IO`が
手に入ると主張されるかもしれません。
しかし違いがあるのです：私達の`IO`は安全にキャンセルできます！
なので、ユーザーは走ってるコードを、例えば`Ctrl-c`を押すことで好きなときに止めることができ、
私達のコードはそのような状況においてさえも安全なリソースの解放（ストリームのクローズ）を処理できます。
同じことが、`copy`関数の機能を必要とする別のモジュールからこの関数が呼び出されたときにも当てはまります。
この関数によって返される`IO`が、走っているときにキャンセルされても、リソースは適切に解放されるのです。

### Polymorphic cats-effect code
There is an important characteristic of `IO` that we should be aware of. `IO` is
able to suspend side-effects asynchronously thanks to the existence of an
instance of `Async[IO]`. Because `Async` extends `Sync`, `IO` can also suspend
side-effects synchronously. On top of that `Async` extends typeclasses such as
`MonadCancel`, `Concurrent` or `Temporal`, which bring the possibility to cancel
an `IO` instance, to run several `IO` instances concurrently, to timeout an
execution, to force the execution to wait (sleep), etc.

気づいておくべき重要な特性が`IO`にはあります。
`IO`は副作用を非同期的にサスペンドすることが可能です。
これは`Async[IO]`のインスタンスが存在するためです。
`Async`は`Sync`を継承しているので、`IO`は同期的に副作用をサスペンドすることもできます。
それに加え、`Aync`は`MonadCancel`, `Concurrent`, `Temporal`のような型クラスを継承しており、
これによって、`IO`インスタンスをキャンセルしたり、複数の`IO`インスタンスを並行に走らせたり、
実行をタイムアウトしたり、実行を待機（スリープ）させたり、といったことが可能になります。

So well, `Sync` and `Async` can suspend side effects. We have used `IO` so far
mostly for that purpose. Now, going back to the code we created to copy files,
could have we coded its functions in terms of some `F[_]: Sync` and `F[_]:
Async` instead of `IO`?  Truth is we could, see for example how we would define
a polymorphic version of our `transfer` function with this approach, just by
replacing any use of `IO` by calls to the `delay` and `pure` methods of the
`Sync[F]` instance:

いいでしょう。`Sync`と`Async`は副作用をサスペンドできます。
私達はここまで、主にそのために`IO`を使ってきました。
それでは、ファイルをコピーするために書いたコードに戻るとして、
`IO`の代わりに`F[_]: Sync`や`F[_]: Async`を使ってコーディングすることはできたでしょうか？
実のところそれは可能で、多相的なバージョンの`transfer`関数が、
このアプローチによってどう定義されるか見てください。
単に`IO`を`Sync[F]`インスタンスの`delay`と`pure`メソッドの呼び出しに置き換えるだけです：

```scala mdoc:compile-only
import cats.effect.Sync
import cats.syntax.all._
import java.io._

def transmit[F[_]: Sync](origin: InputStream, destination: OutputStream, buffer: Array[Byte], acc: Long): F[Long] =
  for {
    amount <- Sync[F].blocking(origin.read(buffer, 0, buffer.length))
    count  <- if(amount > -1) Sync[F].blocking(destination.write(buffer, 0, amount)) >> transmit(origin, destination, buffer, acc + amount)
              else Sync[F].pure(acc) // End of read stream reached (by java.io.InputStream contract), nothing to write
  } yield count // Returns the actual amount of bytes transmitted
```

We leave as an exercise to code the polymorphic versions of `inputStream`,
`outputStream`, `inputOutputStreams`, `transfer` and `copy` functions.

多相バージョンの関数`inputStream`,
`outputStream`, `inputOutputStreams`, `transfer` そして `copy` をコーディングするのは
練習問題とします。

```scala mdoc:compile-only
import cats.effect._
import java.io._

def transmit[F[_]: Sync](origin: InputStream, destination: OutputStream, buffer: Array[Byte], acc: Long): F[Long] = ???
def transfer[F[_]: Sync](origin: InputStream, destination: OutputStream): F[Long] = ???
def inputStream[F[_]: Sync](f: File): Resource[F, FileInputStream] = ???
def outputStream[F[_]: Sync](f: File): Resource[F, FileOutputStream] = ???
def inputOutputStreams[F[_]: Sync](in: File, out: File): Resource[F, (InputStream, OutputStream)] = ???
def copy[F[_]: Sync](origin: File, destination: File): F[Long] = ???
```

Only in our `main` function we will set `IO` as the final `F` for our program.
To do so, of course, a `Sync[IO]` instance must be in scope, but that instance
is brought transparently by `IOApp` so we do not need to be concerned about it.

私達のプログラムで`IO`を最終的な`F`としてセットするのは`main`関数の中だけです。
そうするためにはもちろん、`Sync[IO]`のインスタンスがスコープに入っていなければなりませんが、
そのインスタンスは`IOApp`によって透過的にもたらされるので、気にする必要はありません。

During the remainder of this tutorial we will use polymorphic code, only falling
to `IO` in the `run` method of our `IOApp`s. Polymorphic code is less
restrictive, as functions are not tied to `IO` but are applicable to any `F[_]`
as long as there is an instance of the type class required (`Sync[F[_]]` ,
`Async[F[_]]`...) in scope. The type class to use will depend on the
requirements of our code.

このチュートリアルの残りでは、多相的なコードだけを使い、
`IOApp`の`run`メソッドにおいてのみ`IO`に収まります。
多相的なコードは制限が少ないです。
関数が`IO`に紐付いておらず、必要な型クラス (`Sync[F[_]]`, `Async[F[_]]` ...)
のインスタンスがスコープに存在する限り任意の`F[_]`に適用可能であるので。
使用する型クラスは私達のコードの要件よって変わるでしょう。
 
#### Copy program code, polymorphic version

コピープログラム、多相的バージョン

The polymorphic version of our copy program in full is available
[here](https://github.com/lrodero/cats-effect-tutorial/blob/series/3.x/src/main/scala/catseffecttutorial/copyfile/CopyFilePolymorphic.scala).

多相バージョンのコピープログラムは全体が
[ここ](https://github.com/lrodero/cats-effect-tutorial/blob/series/3.x/src/main/scala/catseffecttutorial/copyfile/CopyFilePolymorphic.scala)
で利用可能です。

### Exercises: improving our small `IO` program

練習問題：私達の小さな`IO`プログラムを改善

To finalize we propose you some exercises that will help you to keep improving
your IO-kungfu:

仕上げとして、あなたのIO-カンフー [訳注：厳しい訓練によって得られるスキルをカンフーと呼ぶことがあります]
を向上する助けになるであろう練習問題を提案します：

1. Modify the `IOApp` so it shows an error and abort the execution if the origin
   and destination files are the same, the origin file cannot be open for
   reading or the destination file cannot be opened for writing. Also, if the
   destination file already exists, the program should ask for confirmation
   before overwriting that file.
2. Modify `transmit` so the buffer size is not hardcoded but passed as
   parameter.
3. Test safe cancelation, checking that the streams are indeed being properly
   closed. You can do that just by interrupting the program execution pressing
   `Ctrl-c`. To make sure you have the time to interrupt the program, introduce
   a delay of a few seconds in the `transmit` function (see `IO.sleep`). And to
   ensure that the release functionality in the `Resource`s is run you can add
   some log message there (see `IO.println`).
4. Create a new program able to copy folders. If the origin folder has
   subfolders, then their contents must be recursively copied too. Of course the
   copying must be safely cancelable at any moment.

1. `IOApp`を修正して、次の場合にエラーを表示し、実行をやめるようにしてください。
   オリジンとデスティネーションファイルが同じ、またはオリジンファイルが読み込み用にオープンできない、
   またはデスティネーションファイルが書き込みようにオープンできない。
   また、デスティネーションファイルが既に存在しているなら、プログラムは上書きする前に確認を求めるべきです。
2. `transmit`を修正して、バッファサイズがハードコードされるのではなく、
   パラメータとして渡されるようにしてください。
3. 実際にストリームが適切にクローズされることを確認して、
   安全にキャンセルできることをテストしてください。
   単にプログラム実行を`Ctrl-c`によって割り込むだけでできます。
   プログラムに割り込める時間があるようにするため、
   `transmit`関数に数秒のディレイを入れてください（`IO.sleep`を見よ）。
   また、`Resource`の解放機能が走ることを保証するためには、
   ログメッセージを追加しても良いでしょう（`IO.println`を見よ）。
4. フォルダをコピーするための新しいプログラムを作成してください。
   オリジンフォルダがサブフォルダを持っているなら、その中身も再帰的にコピーされなければなりません。
   もちろん、コピーはいつでも安全にキャンセルできなければなりません。

## <a name="producerconsumer"></a>Producer-consumer problem - concurrency and fibers
The _producer-consumer_ pattern is often found in concurrent setups. Here one or
more producers insert data on a shared data structure like a queue while one or
more consumers extract data from it. Readers and writers run concurrently. If
the queue is empty then readers will block until data is available, if the queue
is full then writers will wait for some 'bucket' to be free. Only one writer at
a time can add data to the queue to prevent data corruption. Also only one
reader can extract data from the queue so no two readers get the same data item.

Variations of this problem exist depending on whether there are more than one
consumer/producer, or whether the data structure sitting between them is
size-bounded or not. The solutions discussed here are suited for multi-consumer
and multi-reader settings. Initially we will assume an unbounded data structure,
and later present a solution for a bounded one.

But before we work on the solution for this problem we must introduce _fibers_,
which are the basic building block of cats-effect concurrency.

### Intro to fibers
A fiber carries an `F` action to execute (typically an `IO` instance). Fibers
are like 'light' threads, meaning they can be used in a similar way than threads
to create concurrent code. However, they are _not_ threads. Spawning new fibers
does not guarantee that the action described in the `F` associated to it will be
run if there is a shortage of threads. Internally cats-effect uses thread pools
to run fibers when running on the JVM. So if there is no thread available in the
pool then the fiber execution will 'wait' until some thread is free again. On
the other hand when the execution of some fiber is blocked _e.g._ because it
must wait for a semaphore to be released, the thread running the fiber is
recycled by cats-effect so it is available for other fibers. When the fiber
execution can be resumed cats-effect will look for some free thread to continue
the execution. The term "_semantically blocked_" is used sometimes to denote
that blocking the fiber does not involve halting any thread. Cats-effect also
recycles threads of finished and canceled fibers. But keep in mind that, in
contrast, if the fiber is truly blocked by some external action like waiting for
some input from a TCP socket, then cats-effect has no way to recover back that
thread until the action finishes. Such calls should be wrapped by `IO.blocking`
to signal that the wrapped code will block the thread.  Cats-effect uses that
info as a hint to optimize `IO` scheduling.

Another difference with threads is that fibers are very cheap entities. We can
spawn millions of them at ease without impacting the performance. 

A worthy note is that you do not have to explicitly shut down fibers. If you spawn
a fiber and it finishes actively running its `IO` it will get cleaned up by the 
garbage collector unless there is some other active memory reference to it. So basically
you can treat a fiber as any other regular object, except that when the fiber is _running_ 
(present tense), the cats-effect runtime itself keeps the fiber alive.

This has some interesting implications as well. Like if you create an `IO.async` node and 
register the callback with something, and you're in a Fiber which has no strong object 
references anywhere else (i.e. you did some sort of fire-and-forget thing), then the callback 
itself is the only strong reference to the fiber. Meaning if the registration fails or the 
system you registered with throws it away, the fiber will just gracefully disappear.

Cats-effect implements some concurrency primitives to coordinate concurrent
fibers: [Deferred](std/deferred.md), [Ref](std/ref.md), `Semaphore`...

Way more detailed info about concurrency in cats-effect can be found in [this
other tutorial 'Concurrency in Scala with
Cats-Effect'](https://github.com/slouc/concurrency-in-scala-with-ce).

Ok, now we have briefly discussed fibers we can start working on our
producer-consumer problem.

### First (and inefficient) implementation
We need an intermediate structure where producer(s) can insert data to and
consumer(s) extracts data from. Let's assume a simple queue. Initially there
will be only one producer and one consumer. Producer will generate a sequence of
integers (`1`, `2`, `3`...), consumer will just read that sequence.  Our shared
queue will be an instance of an immutable `Queue[Int]`.

Accesses to the queue can (and will!) be concurrent, thus we need some way to
protect the queue so only one fiber at a time is handling it. The best way to
ensure an ordered access to some shared data is [Ref](std/ref.md). A
`Ref` instance wraps some given data and implements methods to manipulate that
data in a safe manner. When some fiber is runnning one of those methods, any
other call to any method of the `Ref` instance will be blocked.

The `Ref` wrapping our queue will be `Ref[F, Queue[Int]]` (for some `F[_]`).

Now, our `producer` method will be:

```scala mdoc:compile-only
import cats.effect._
import cats.effect.std.Console
import cats.syntax.all._
import collection.immutable.Queue

def producer[F[_]: Sync: Console](queueR: Ref[F, Queue[Int]], counter: Int): F[Unit] =
  for {
    _ <- if(counter % 10000 == 0) Console[F].println(s"Produced $counter items") else Sync[F].unit
    _ <- queueR.getAndUpdate(_.enqueue(counter + 1))
    _ <- producer(queueR, counter + 1)
  } yield ()
```

First line just prints some log message every `10000` items, so we know if it is
'alive'. It uses type class `Console[_]`, which brings the capacity to print
and read strings (`IO.println` just uses `Console[IO].println` underneath).

Then our code calls `queueR.getAndUpdate` to add data into the queue. Note
that `.getAndUpdate` provides the current queue, then we use `.enqueue` to
insert the next value `counter+1`. This call returns a new queue with the value
added that is stored by the ref instance. If some other fiber is accessing to
`queueR` then the fiber is (semantically) blocked.

The `consumer` method is a bit different. It will try to read data from the
queue but it must be aware that the queue can be empty:

```scala mdoc:compile-only
import cats.effect._
import cats.effect.std.Console
import cats.syntax.all._
import collection.immutable.Queue

def consumer[F[_]: Sync: Console](queueR: Ref[F, Queue[Int]]): F[Unit] =
  for {
    iO <- queueR.modify{ queue =>
      queue.dequeueOption.fold((queue, Option.empty[Int])){case (i,queue) => (queue, Option(i))}
    }
    _ <- if(iO.exists(_ % 10000 == 0)) Console[F].println(s"Consumed ${iO.get} items") else Sync[F].unit
    _ <- consumer(queueR)
  } yield ()
```

The call to `queueR.modify` allows to modify the wrapped data (our queue) and
return a value that is computed from that data. In our case, it returns an
`Option[Int]` that will be `None` if queue was empty. Next line is used to log
a message in console every `10000` read items. Finally `consumer` is called
recursively to start again.

We can now create a program that instantiates our `queueR` and runs both
`producer` and `consumer` in parallel:

```scala mdoc:compile-only
import cats.effect._
import cats.effect.std.Console
import cats.syntax.all._
import collection.immutable.Queue

object InefficientProducerConsumer extends IOApp {

  def producer[F[_]: Sync](queueR: Ref[F, Queue[Int]], counter: Int): F[Unit] = ??? // As defined before
  def consumer[F[_]: Sync](queueR: Ref[F, Queue[Int]]): F[Unit] = ??? // As defined before

  override def run(args: List[String]): IO[ExitCode] =
    for {
      queueR <- Ref.of[IO, Queue[Int]](Queue.empty[Int])
      res <- (consumer(queueR), producer(queueR, 0))
        .parMapN((_, _) => ExitCode.Success) // Run producer and consumer in parallel until done (likely by user cancelling with CTRL-C)
        .handleErrorWith { t =>
          Console[IO].errorln(s"Error caught: ${t.getMessage}").as(ExitCode.Error)
        }
    } yield res

}
```

The full implementation of this naive producer consumer is available
[here](https://github.com/lrodero/cats-effect-tutorial/blob/series/3.x/src/main/scala/catseffecttutorial/producerconsumer/InefficientProducerConsumer.scala).

Our `run` function instantiates the shared queue wrapped in a `Ref` and boots
the producer and consumer in parallel. To do to it uses `parMapN`, that creates
and runs the fibers that will run the `IO`s passed as parameter. Then it takes
the output of each fiber and applies a given function to them. In our case
both producer and consumer shall run forever until user presses CTRL-C which
will trigger a cancelation.

Alternatively we could have used `start` method to explicitly create new
`Fiber` instances that will run the producer and consumer, then use `join` to
wait for them to finish, something like:

```scala mdoc:compile-only
import cats.effect._
import collection.immutable.Queue

object InefficientProducerConsumer extends IOApp {

  def producer[F[_]: Sync](queueR: Ref[F, Queue[Int]], counter: Int): F[Unit] = ??? // As defined before
  def consumer[F[_]: Sync](queueR: Ref[F, Queue[Int]]): F[Unit] = ??? // As defined before

  def run(args: List[String]): IO[ExitCode] =
    for {
      queueR <- Ref.of[IO, Queue[Int]](Queue.empty[Int])
      producerFiber <- producer(queueR, 0).start
      consumerFiber <- consumer(queueR).start
      _ <- producerFiber.join
      _ <- consumerFiber.join
    } yield ExitCode.Error
}
```

However in most situations it is not advisable to handle fibers manually as
they are not trivial to work with. For example, if there is an error in a fiber
the `join` call to that fiber will _not_ raise it, it will return normally and
you must explicitly check the `Outcome` instance returned by the `.join` call
to see if it errored. Also, the other fibers will keep running unaware of what
happened.

Cats Effect provides additional `joinWith` or `joinWithNever` methods to make
sure at least that the error is raised with the usual `MonadError` semantics
(e.g., short-circuiting).  Now that we are raising the error, we also need to
cancel the other running fibers. We can easily get ourselves trapped in a
tangled mess of fibers to keep an eye on.  On top of that the error raised by a
fiber is not promoted until the call to `joinWith` or `.joinWithNever` is
reached. So in our example above if `consumerFiber` raises an error then we
have no way to observe that until the producer fiber has finished. Alarmingly,
note that in our example the producer _never_ finishes and thus the error would
_never_ be observed!  And even if the producer fiber did finish, it would have
been consuming resources for nothing.
 
In contrast `parMapN` does promote any error it finds to the caller _and_ takes
care of canceling the other running fibers. As a result `parMapN` is simpler to
use, more concise, and easier to reason about. _Because of that, unless you
have some specific and unusual requirements you should prefer to use higher
level commands such as `parMapN` or `parSequence` to work with fibers_.

Ok, we stick to our implementation based on `.parMapN`. Are we done? Does it
Work? Well, it works... but it is far from ideal. If we run it we will find that
the producer runs faster than the consumer so the queue is constantly growing.
And, even if that was not the case, we must realize that the consumer will be
continually running regardless if there are elements in the queue, which is far
from ideal. We will try to improve it in the next section by using
[Deferred](std/deferred.md). Also we will use several consumers and
producers to balance production and consumption rate.

### A more solid implementation of the producer/consumer problem
In our producer/consumer code we already protect access to the queue (our shared
resource) using a `Ref`. Now, instead of using `Option` to represent elements
retrieved from a possibly empty queue, we should instead block the caller fiber
somehow if queue is empty until some element can be returned. This will be done
by creating and keeping instances of `Deferred`. A `Deferred[F, A]` instance can
hold one single element of some type `A`. `Deferred` instances are created
empty, and can be filled only once. If some fiber tries to read the element from
an empty `Deferred` then it will be semantically blocked until some other fiber
fills (completes) it.

Thus, alongside the queue of produced but not yet consumed elements, we have to
keep track of the `Deferred` instances created when the queue was empty that are
waiting for elements to be available. These instances will be kept in a new
queue `takers`. We will keep both queues in a new type `State`:

```scala mdoc:compile-only
import cats.effect.Deferred
import scala.collection.immutable.Queue
case class State[F[_], A](queue: Queue[A], takers: Queue[Deferred[F,A]])
```

Both producer and consumer will access the same shared state instance, which
will be carried and safely modified by an instance of `Ref`. Consumer shall
work as follows:
1. If `queue` is not empty, it will extract and return its head. The new state
   will keep the tail of the queue, not change on `takers` will be needed.
2. If `queue` is empty it will use a new `Deferred` instance as a new `taker`,
   add it to the `takers` queue, and 'block' the caller by invoking `taker.get`

Assuming that in our setting we produce and consume `Int`s (just as before),
then new consumer code will then be:

```scala mdoc:compile-only
import cats.effect.{Deferred, Ref, Async}
import cats.effect.std.Console
import cats.syntax.all._
import scala.collection.immutable.Queue

case class State[F[_], A](queue: Queue[A], takers: Queue[Deferred[F,A]])

def consumer[F[_]: Async: Console](id: Int, stateR: Ref[F, State[F, Int]]): F[Unit] = {

  val take: F[Int] =
    Deferred[F, Int].flatMap { taker =>
      stateR.modify {
        case State(queue, takers) if queue.nonEmpty =>
          val (i, rest) = queue.dequeue
          State(rest, takers) -> Async[F].pure(i) // Got element in queue, we can just return it
        case State(queue, takers) =>
          State(queue, takers.enqueue(taker)) -> taker.get // No element in queue, must block caller until some is available
      }.flatten
    }

  for {
    i <- take
    _ <- if(i % 10000 == 0) Console[F].println(s"Consumer $id has reached $i items") else Async[F].unit
    _ <- consumer(id, stateR)
  } yield ()
}
```

The `id` parameter is only used to identify the consumer in console logs (recall
we will have now several producers and consumers running in parallel). The
`take` instance implements the checking and updating of the state in `stateR`.
Note how it will block on `taker.get` when the queue is empty.

The producer, for its part, will:
1. If there are waiting `takers`, it will take the first in the queue and offer
   it the newly produced element (`taker.complete`).
2. If no `takers` are present, it will just enqueue the produced element.

Thus the producer will look like:

```scala mdoc:compile-only
import cats.effect.{Deferred, Ref, Sync}
import cats.effect.std.Console
import cats.syntax.all._
import scala.collection.immutable.Queue

case class State[F[_], A](queue: Queue[A], takers: Queue[Deferred[F,A]])

def producer[F[_]: Sync: Console](id: Int, counterR: Ref[F, Int], stateR: Ref[F, State[F,Int]]): F[Unit] = {

  def offer(i: Int): F[Unit] =
    stateR.modify {
      case State(queue, takers) if takers.nonEmpty =>
        val (taker, rest) = takers.dequeue
        State(queue, rest) -> taker.complete(i).void
      case State(queue, takers) =>
        State(queue.enqueue(i), takers) -> Sync[F].unit
    }.flatten

  for {
    i <- counterR.getAndUpdate(_ + 1)
    _ <- offer(i)
    _ <- if(i % 10000 == 0) Console[F].println(s"Producer $id has reached $i items") else Sync[F].unit
    _ <- producer(id, counterR, stateR)
  } yield ()
}
```

Finally we modify our main program so it instantiates the counter and state
`Ref`s. Also it will create several consumers and producers, 10 of each, and
will start all of them in parallel:

```scala mdoc:compile-only
import cats.effect._
import cats.effect.std.Console
import cats.instances.list._
import cats.syntax.all._
import scala.collection.immutable.Queue

object ProducerConsumer extends IOApp {

  case class State[F[_], A](queue: Queue[A], takers: Queue[Deferred[F,A]])

  object State {
    def empty[F[_], A]: State[F, A] = State(Queue.empty, Queue.empty)
  }

  def producer[F[_]: Sync](id: Int, counterR: Ref[F, Int], stateR: Ref[F, State[F,Int]]): F[Unit] = ??? // As defined before

  def consumer[F[_]: Async](id: Int, stateR: Ref[F, State[F, Int]]): F[Unit] = ??? // As defined before

  override def run(args: List[String]): IO[ExitCode] =
    for {
      stateR <- Ref.of[IO, State[IO,Int]](State.empty[IO, Int])
      counterR <- Ref.of[IO, Int](1)
      producers = List.range(1, 11).map(producer(_, counterR, stateR)) // 10 producers
      consumers = List.range(1, 11).map(consumer(_, stateR))           // 10 consumers
      res <- (producers ++ consumers)
        .parSequence.as(ExitCode.Success) // Run producers and consumers in parallel until done (likely by user cancelling with CTRL-C)
        .handleErrorWith { t =>
          Console[IO].errorln(s"Error caught: ${t.getMessage}").as(ExitCode.Error)
        }
    } yield res
}
```

The full implementation of this producer consumer with unbounded queue is
available
[here](https://github.com/lrodero/cats-effect-tutorial/blob/series/3.x/src/main/scala/catseffecttutorial/producerconsumer/ProducerConsumer.scala).

Producers and consumers are created as two list of `IO` instances. All of them
are started in their own fiber by the call to `parSequence`, which will wait for
all of them to finish and then return the value passed as parameter. As in the
previous example this program shall run forever until the user presses CTRL-C.

Having several consumers and producers improves the balance between consumers
and producers... but still, on the long run, queue tends to grow in size. To
fix this we will ensure the size of the queue is bounded, so whenever that max
size is reached producers will block as consumers do when the queue is empty.


### Producer consumer with bounded queue
Having a bounded queue implies that producers, when queue is full, will wait (be
'semantically blocked') until there is some empty bucket available to be filled.
So an implementation needs to keep track of these waiting producers. To do so we
will add a new queue `offerers` that will be added to the `State` alongside
`takers`.  For each waiting producer the `offerers` queue will keep a
`Deferred[F, Unit]` that will be used to block the producer until the element it
offers can be added to `queue` or directly passed to some consumer (`taker`).
Alongside the `Deferred` instance we need to keep as well the actual element
offered by the producer in the `offerers` queue. Thus `State` class now becomes:

```scala mdoc:compile-only
import cats.effect.Deferred
import scala.collection.immutable.Queue

case class State[F[_], A](queue: Queue[A], capacity: Int, takers: Queue[Deferred[F,A]], offerers: Queue[(A, Deferred[F,Unit])])
```

Of course both consumer and producer have to be modified to handle this new
queue `offerers`. A consumer can find four escenarios, depending on if `queue`
and `offerers` are each one empty or not. For each escenario a consumer shall:
1. If `queue` is not empty:
    1. If `offerers` is empty then it will extract and return `queue`'s head.
    2. If `offerers` is not empty (there is some producer waiting) then things
       are more complicated. The `queue` head will be returned to the consumer.
       Now we have a free bucket available in `queue`. So the first waiting
       offerer can use that bucket to add the element it offers. That element 
       will be added to `queue`, and the `Deferred` instance will be completed
       so the producer is released (unblocked).
2. If `queue` is empty:
    1. If `offerers` is empty then there is nothing we can give to the caller,
       so a new `taker` is created and added to `takers` while caller is
       blocked with `taker.get`.
    2. If `offerers` is not empty then the first offerer in queue is extracted,
       its `Deferred` instance released while the offered element is returned to
       the caller.

So consumer code looks like this:

```scala mdoc:compile-only
import cats.effect._
import cats.effect.std.Console
import cats.syntax.all._
import scala.collection.immutable.Queue

case class State[F[_], A](queue: Queue[A], capacity: Int, takers: Queue[Deferred[F,A]], offerers: Queue[(A, Deferred[F,Unit])])

def consumer[F[_]: Async: Console](id: Int, stateR: Ref[F, State[F, Int]]): F[Unit] = {

  val take: F[Int] =
    Deferred[F, Int].flatMap { taker =>
      stateR.modify {
        case State(queue, capacity, takers, offerers) if queue.nonEmpty && offerers.isEmpty =>
          val (i, rest) = queue.dequeue
          State(rest, capacity, takers, offerers) -> Async[F].pure(i)
        case State(queue, capacity, takers, offerers) if queue.nonEmpty =>
          val (i, rest) = queue.dequeue
          val ((move, release), tail) = offerers.dequeue
          State(rest.enqueue(move), capacity, takers, tail) -> release.complete(()).as(i)
        case State(queue, capacity, takers, offerers) if offerers.nonEmpty =>
          val ((i, release), rest) = offerers.dequeue
          State(queue, capacity, takers, rest) -> release.complete(()).as(i)
        case State(queue, capacity, takers, offerers) =>
          State(queue, capacity, takers.enqueue(taker), offerers) -> taker.get
      }.flatten
    }

  for {
    i <- take
    _ <- if(i % 10000 == 0) Console[F].println(s"Consumer $id has reached $i items") else Async[F].unit
    _ <- consumer(id, stateR)
  } yield ()
}
```

Producer functionality is a bit easier:
1. If there is any waiting `taker` then the produced element will be passed to
   it, releasing the blocked fiber.
2. If there is no waiting `taker` but `queue` is not full, then the offered
   element will be enqueued there.
3. If there is no waiting `taker` and `queue` is already full then a new
   `offerer` is created, blocking the producer fiber on the `.get` method of the
   `Deferred` instance.

Now producer code looks like this:

```scala mdoc:compile-only
import cats.effect._
import cats.effect.std.Console
import cats.syntax.all._
import scala.collection.immutable.Queue

case class State[F[_], A](queue: Queue[A], capacity: Int, takers: Queue[Deferred[F,A]], offerers: Queue[(A, Deferred[F,Unit])])

def producer[F[_]: Async: Console](id: Int, counterR: Ref[F, Int], stateR: Ref[F, State[F,Int]]): F[Unit] = {

  def offer(i: Int): F[Unit] =
    Deferred[F, Unit].flatMap[Unit]{ offerer =>
      stateR.modify {
        case State(queue, capacity, takers, offerers) if takers.nonEmpty =>
          val (taker, rest) = takers.dequeue
          State(queue, capacity, rest, offerers) -> taker.complete(i).void
        case State(queue, capacity, takers, offerers) if queue.size < capacity =>
          State(queue.enqueue(i), capacity, takers, offerers) -> Async[F].unit
        case State(queue, capacity, takers, offerers) =>
          State(queue, capacity, takers, offerers.enqueue(i -> offerer)) -> offerer.get
      }.flatten
    }

  for {
    i <- counterR.getAndUpdate(_ + 1)
    _ <- offer(i)
    _ <- if(i % 10000 == 0) Console[F].println(s"Producer $id has reached $i items") else Async[F].unit
    _ <- producer(id, counterR, stateR)
  } yield ()
}
```

As you see, producer and consumer are coded around the idea of keeping and
modifying state, just as with unbounded queues.

As the final step we must adapt the main program to use these new consumers and
producers. Let's say we limit the queue size to `100`, then we have:

```scala mdoc:compile-only
import cats.effect._
import cats.effect.std.Console
import cats.syntax.all._

import scala.collection.immutable.Queue

object ProducerConsumerBounded extends IOApp {

  case class State[F[_], A](queue: Queue[A], capacity: Int, takers: Queue[Deferred[F,A]], offerers: Queue[(A, Deferred[F,Unit])])

  object State {
    def empty[F[_], A](capacity: Int): State[F, A] = State(Queue.empty, capacity, Queue.empty, Queue.empty)
  }

  def producer[F[_]: Async](id: Int, counterR: Ref[F, Int], stateR: Ref[F, State[F,Int]]): F[Unit] = ??? // As defined before

  def consumer[F[_]: Async](id: Int, stateR: Ref[F, State[F, Int]]): F[Unit] = ??? // As defined before

  override def run(args: List[String]): IO[ExitCode] =
    for {
      stateR <- Ref.of[IO, State[IO, Int]](State.empty[IO, Int](capacity = 100))
      counterR <- Ref.of[IO, Int](1)
      producers = List.range(1, 11).map(producer(_, counterR, stateR)) // 10 producers
      consumers = List.range(1, 11).map(consumer(_, stateR))           // 10 consumers
      res <- (producers ++ consumers)
        .parSequence.as(ExitCode.Success) // Run producers and consumers in parallel until done (likely by user cancelling with CTRL-C)
        .handleErrorWith { t =>
          Console[IO].errorln(s"Error caught: ${t.getMessage}").as(ExitCode.Error)
        }
    } yield res
}
```

The full implementation of this producer consumer with bounded queue is
available
[here](https://github.com/lrodero/cats-effect-tutorial/blob/series/3.x/src/main/scala/catseffecttutorial/producerconsumer/ProducerConsumerBounded.scala).

### Taking care of cancelation
We shall ask ourselves, is this implementation cancelation-safe? That is, what
happens if the fiber running a consumer or a producer gets canceled? Does state
become inconsistent? Let's check `producer` first. State is handled by its
internal `offer`, so we will focus on it. And, for the sake of clarity in our
analysis let's reformat the code using a for-comprehension:

```scala
import cats.effect.Deferred

def offer[F[_]](i: Int): F[Unit] =
  for {
    offerer <- Deferred[F, Int]
    op      <- stateR.modify {???} // `op` is an F[] to be run
    _       <- op
  } yield ()
```

So far so good. Now, cancelation steps into action in each `.flatMap` in `F`,
that is, in each step of our for-comprehension. If the fiber gets canceled right
before or after the first step, well, that is not an issue. The `offerer` will
be eventually garbage collected, that's all. But what if the cancelation
happens right after the call to `modify`? Well, then `op` will not be run.
Recall that, by the content of `modify`, that `op` can be
`taker.complete(i).void`, `Sync[F].unit` or `offerer.get`. Cancelling after
having removed the `taker` or added the `offerer` to the state but without
running `op` will leave the state inconsistent. We can quickly fix this by
making that code uncancelable:

```scala
def offer[F[_]](i: Int): F[Unit] =
  for {
    offerer <- Deferred[F, Int]
    _       <- F.uncancelable { poll => // `poll` ignored at this point, we'll discuss it later
                 for {
                   op <- stateR.modify {???} // `op` is an F[] to be run
                   _  <- op // `taker.complete(i).void`, `Sync[F].unit` or `offerer.get`
                 } yield ()
              }
  } yield ()
```

What is the problem here? If `op` is non-blocking, that is, if it is either
`F.unit` or `taker.complete(a).void`, then our solution would be ok. But when
the operation is `offerer.get` we have an issue as `.get` will block until
`offerer` is completed (recall it is a `Deferred` instance). So the fiber will
not be able to progress, but at the same time we have set that operation inside
an uncancelable region. So there is no way to cancel that blocked fiber! For
example, we cannot set a timeout on its execution! Thus, if the `offerer` is
never completed then that fiber will never finish.

This can be addressed using `Poll[F]`, which is passed as parameter by
`F.uncancelable`. `Poll[F]` is used to define cancelable code inside the
uncancelable region. So if the operation to run was `offerer.get` we will embed
that call inside the `Poll[F]`, thus ensuring the blocked fiber can be canceled.
Finally, we must also take care of cleaning up the state if there is indeed a
cancelation. That cleaning up will have to remove the `offerer` from the list
of offerers kept in the state, as it shall never be completed. Our `offer`
function has become:

```scala mdoc:compile-only
import cats.effect._
import cats.effect.std.Console
import cats.syntax.all._
import cats.effect.syntax.all._
import scala.collection.immutable.Queue

case class State[F[_], A](queue: Queue[A], capacity: Int, takers: Queue[Deferred[F,A]], offerers: Queue[(A, Deferred[F,Unit])])

def producer[F[_]: Async: Console](id: Int, counterR: Ref[F, Int], stateR: Ref[F, State[F,Int]]): F[Unit] = {

  def offer(i: Int): F[Unit] =
    Deferred[F, Unit].flatMap[Unit]{ offerer =>
      Async[F].uncancelable { poll => // `poll` used to embed cancelable code, i.e. the call to `offerer.get`
        stateR.modify {
          case State(queue, capacity, takers, offerers) if takers.nonEmpty =>
            val (taker, rest) = takers.dequeue
            State(queue, capacity, rest, offerers) -> taker.complete(i).void
          case State(queue, capacity, takers, offerers) if queue.size < capacity =>
            State(queue.enqueue(i), capacity, takers, offerers) -> Async[F].unit
          case State(queue, capacity, takers, offerers) =>
            val cleanup = stateR.update { s => s.copy(offerers = s.offerers.filter(_._2 ne offerer)) }
            State(queue, capacity, takers, offerers.enqueue(i -> offerer)) -> poll(offerer.get).onCancel(cleanup)
        }.flatten
      }
    }

  for {
    i <- counterR.getAndUpdate(_ + 1)
    _ <- offer(i)
    _ <- if(i % 10000 == 0) Console[F].println(s"Producer $id has reached $i items") else Async[F].unit
    _ <- producer(id, counterR, stateR)
  } yield ()
}
```

The consumer part must deal with cancelation in the same way. It will use
`poll` to enable cancelation on the blocking calls, but at the same time it
will make sure to clean up the state when a cancelation occurs. In this case,
the blocking call is `taker.get`, when such call is canceled the `taker` will
be removed from the list of takers in the state. So our `consumer` is now:

```scala mdoc:compile-only
import cats.effect._
import cats.effect.std.Console
import cats.syntax.all._
import cats.effect.syntax.all._
import scala.collection.immutable.Queue

case class State[F[_], A](queue: Queue[A], capacity: Int, takers: Queue[Deferred[F,A]], offerers: Queue[(A, Deferred[F,Unit])])

def consumer[F[_]: Async: Console](id: Int, stateR: Ref[F, State[F, Int]]): F[Unit] = {

  val take: F[Int] =
    Deferred[F, Int].flatMap { taker =>
      Async[F].uncancelable { poll =>
        stateR.modify {
          case State(queue, capacity, takers, offerers) if queue.nonEmpty && offerers.isEmpty =>
            val (i, rest) = queue.dequeue
            State(rest, capacity, takers, offerers) -> Async[F].pure(i)
          case State(queue, capacity, takers, offerers) if queue.nonEmpty =>
            val (i, rest) = queue.dequeue
            val ((move, release), tail) = offerers.dequeue
            State(rest.enqueue(move), capacity, takers, tail) -> release.complete(()).as(i)
          case State(queue, capacity, takers, offerers) if offerers.nonEmpty =>
            val ((i, release), rest) = offerers.dequeue
            State(queue, capacity, takers, rest) -> release.complete(()).as(i)
          case State(queue, capacity, takers, offerers) =>
            val cleanup = stateR.update { s => s.copy(takers = s.takers.filter(_ ne taker)) }
            State(queue, capacity, takers.enqueue(taker), offerers) -> poll(taker.get).onCancel(cleanup)
        }.flatten
      }
    }

  for {
    i <- take
    _ <- if(i % 10000 == 0) Console[F].println(s"Consumer $id has reached $i items") else Async[F].unit
    _ <- consumer(id, stateR)
  } yield ()
}
```

We have made our producer-consumer implementation able to handle cancelation.
Notably, we have not needed to change the `producer` and `consumer` functions
signatures. This last implementation is
available
[here](https://github.com/lrodero/cats-effect-tutorial/blob/series/3.x/src/main/scala/catseffecttutorial/producerconsumer/ProducerConsumerBoundedCancelable.scala).

### Exercise: build a concurrent queue
A _concurrent queue_ is, well, a queue data structure that allows safe
concurrent access. That is, several concurrent processes can safely add and
retrieve data from the queue. It is easy to realize that during the previous
sections we already implemented that kind of functionality, it was implemented
by the `take` and `offer` methods embedded in our `producer` and `consumer`
functions. To build a concurrent queue we only need to extract from those
methods the part that handles the concurrent access. As the last exercise we
propose you to implement a concurrent queue that implements the `take` and
`offer` functions, and then rewrite the `producer` and `consumer` to use your
queue. A possible implementation of the queue is given
[here](https://github.com/lrodero/cats-effect-tutorial/blob/series/3.x/src/main/scala/catseffecttutorial/producerconsumer/exerciseconcurrentqueue/Queue.scala),
while the main program using that queue can be found
[here](https://github.com/lrodero/cats-effect-tutorial/blob/series/3.x/src/main/scala/catseffecttutorial/producerconsumer/exerciseconcurrentqueue/Main.scala).

When you are done, take a look to [Queue implementation in cats-effect std
package](https://github.com/typelevel/cats-effect/blob/series/3.x/std/shared/src/main/scala/cats/effect/std/Queue.scala),
you will notice your code is a simplified version of cats-effect own `Queue`!

## Conclusion

With all this we have covered a good deal of what cats-effect has to offer (but
not all!). Now you are ready to use to create code that operate side effects in
a purely functional manner. Enjoy the ride!
