---
title: "KotlinのCoroutineを試しに使ってみる"
emoji: "🏝️"
type: "tech"
topics: ["kotlin", "coroutine"]
published: false
---

KotlinにはCoroutineと呼ばれるものがあるので、今回はこちらを使ってみます。

ドキュメントはこちらの公式ガイドを参考にしました。

https://kotlinlang.org/docs/coroutines-guide.html

# Coroutine とは？

一通りKotlinで試したあとに分かったのですが、Kotlin固有の言葉ではないようです。wikiによるとCoroutineはSubroutineとは異なり中断・再開できる処理のまとまりということらしいです。

https://ja.wikipedia.org/wiki/%E3%82%B3%E3%83%AB%E3%83%BC%E3%83%81%E3%83%B3

KotlinのCoroutineによって、Kotlinのコードを中断・再開可能な単位に構成することができるようになります。それだけではなく、中断したCoroutineが再開するスレッドは異なっても良いためCPUのコアを効率的に使用でき、スループットの向上が期待できます。IO待ちが発生する状況や大量の計算を非同期処理したい場合に役立ちそうですね。

# Hello, Coroutine!

例えば次のように書きます。

``` kotlin
package booookstore.playground

import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

class HelloCoroutineTest {

    @Test
    fun sayHello() {
        var message = ""
        runBlocking { //#1
            launch { //#2
                delay(100L) //#3
                message += "World. " // <2>
            }
            message += "Hello, " // <1>
        }
        message += "from coroutine." // <3>

        assertEquals("Hello, World. from coroutine.", message)
    }

}
```

**#1** `runBlocking` はCoroutineBuilderというものです。Builderと呼ばれるとおりCoroutineを作成します。 `runBlock` はNone-Coroutineの世界からCoroutineの世界へのエントリポイントです。`{ }` の内部ではCoroutineを自由に生成することができます。Blockingと呼ばれるとおり、Coroutineの実行完了まで現在のスレッドをブロックします。言い換えるとCoroutineの実行が完了するまでスレッドは次のコードを実行しません。

**#2** `launch` もCoroutineBuilderの一つです。そう、CoroutineBuilderは複数登場しそれぞれ役割が少々違います。 `launch` はCoroutineを新しく生成しますが現在のスレッドをブロックしません。suspend関数によって実行が中断されると現在のスレッドを開放します。開放されたスレッドは他の仕事をすることができます。

**#3** `delay` はsuspend関数の一つです。suspend関数は現在のCoroutineを中断する関数です。ここでは0.1秒Coroutineを中断します。0.1秒後（正確には0.1秒後Coroutineがいずれかのスレッドに割り当てられてから）再開します。

上記のプログラムは <1> <2> <3> の順序でmessage変数にアクセスし、最後のアサーションに成功します。

# CoroutineBuilder

[launch](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) と [async](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) はそれぞれ代表的なCoroutineBuilderです。

|CoroutineBuilder|戻り値|計算結果|
|:--|:--|:--|
|launch|Job|計算結果を返さない|
|async|Deffered|計算結果を返す|

`launch` は `Job` を返します。 `join` を使うことでCoroutineの完了を待機できます。

``` kotlin
package booookstore.playground

import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

class JobTest {

    @Test
    fun job() = runBlocking {
        var message = ""
        val job = launch {
            delay(100L)
            message += "hello"
        }
        job.join()
        assertEquals("hello", message)
    }

}
```

`Job` はCoroutineの計算結果を持っていません。つまり、 `launch` は計算結果を返す必要のないCoroutineを生成します。

一方で `async` は `Deffered` を返します。`Deffered`も`Job`と同様に`join`により完了を待機できます。Coroutineの完了後に `getCompleted` を呼び出せば計算結果を受け取れます。

``` kotlin
package booookstore.playground

import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.async
import kotlinx.coroutines.delay
import kotlinx.coroutines.runBlocking
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

@OptIn(ExperimentalCoroutinesApi::class)
class AsyncTest {

    @Test
    fun async() = runBlocking {
        val deferred = async {
            delay(100L)
            "hello"
        }
        deferred.join()
        assertEquals("hello", deferred.getCompleted())
    }

}
```

`Job` や `Deffered` が複数ある場合はリストにしてすべての完了を待機することもできます。

次は `Job` が複数ある場合のケースです。

``` kotlin
package booookstore.playground

import kotlinx.coroutines.delay
import kotlinx.coroutines.joinAll
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

class JobTest {

    @Test
    fun job() = runBlocking {
        var message = ""
        val jobA = launch {
            delay(100L)
            message += "hello"
        }
        val jobB = launch {
            delay(200L)
            message += ",world"
        }
        listOf(jobA, jobB).joinAll()
        assertEquals("hello,world", message)
    }
}
```

次は `Deffered` が複数ある場合のケースです。

``` kotlin
package booookstore.playground

import kotlinx.coroutines.*
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

@OptIn(ExperimentalCoroutinesApi::class)
class AsyncTest {

    @Test
    fun async() = runBlocking {
        val deferredA = async {
            delay(100L)
            "hello"
        }
        val deferredB = async {
            delay(100L)
            "world"
        }
        listOf(deferredA, deferredB).joinAll()
        assertEquals("hello", deferredA.getCompleted())
        assertEquals("world", deferredB.getCompleted())
    }

}
```

サンプルコードでは伝わりづらいですがCoroutineを任意の数作成することはよくあることだと思うので上記のやり方を覚えておくと良さそうです。

# Suspend function

Coroutine内ではSuspend functionを使うことができます。逆に言うと、Coroutine以外ではSuspend functionを使うことはできません。Suspend functionは中断可能な関数で、例えばネットワーク通信やIOの処理などで処理を中断します。例えば上記の例だと `delay` はSuspend functionです。

Coroutineは中断されると現在のスレッドを開放します。スレッドの開放がCoroutineを使う、使わないで変わってくる大きなポイントだと思います。Coroutine無しでIO処理などを行うとスレッドをブロックするためそのスレッドは何も仕事をせず待機する事になります。一方、Coroutineの場合スレッドを開放するので、開放されたスレッドは他の処理を行えます。よって、スループットが向上する事に繋がります。

Suspend functionを使用するコードを関数に切り出す場合、その関数もまたSuspend functionと名乗る必要が有ります。

``` kotlin

```