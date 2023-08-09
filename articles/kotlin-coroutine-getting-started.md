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

少しの期間触ってみて感じたことは、次の３要素がKotlin Coroutineにおける主要登場人物であることです。

- Coroutine Builder
- Suspend Function
- Coroutine Scope

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

大雑把なイメージにすると以下となります。

![](/images/coroutine-A.png)

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

後述するCoroutineScope内でSuspend functionを呼び出すことができます。さらに、他のSuspend functionからもSuspend functionを呼び出すことができます（以下のサンプルコードを参照）。Suspend functionは中断可能な関数で、例えばネットワーク通信やIOの処理などで処理を中断します。例えば上記の例だと `delay` はSuspend functionです。

Suspend functionによって中断されるとCoroutineは現在のスレッドを開放します。このスレッドの開放がパフォーマンスにおいて重要な点になります。Coroutine無しでIO処理などを行うとスレッドをブロックするためその間スレッドは何も仕事をせず待機する事になります。一方、Coroutineの場合スレッドを開放するので、開放されたスレッドは他の処理に割り当てることができます。

Suspend functionを使用するコードを関数に切り出す場合、その関数もまたSuspend functionと名乗る必要が有り、先頭に `suspend` と宣言します。

次のコードでは２つのSuspend functionを定義しています。Suspend functionからSuspend functionを呼び出すことができるため、 `message1` は `message2` を呼び出しています。この例では合計で0.2秒関数が中断されます。 `message1` は単に `message2` を呼び出しているだけなのでdelayで指定した時間の合計が中断される時間の合計と一致するというわけです。

``` kotlin
package booookstore.playground

import kotlinx.coroutines.delay
import kotlinx.coroutines.runBlocking
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

class SuspendFunctionTest {

    @Test
    fun suspendFunctionTest() = runBlocking {
        val message = message1()
        assertEquals("hello world", message)
    }

    private suspend fun message1(): String {
        delay(100L)
        val message = message2()
        return "hello$message"
    }

    private suspend fun message2(): String {
        delay(100L)
        return " world"
    }

}
```

# CoroutineScope

CoroutineScopeは名前から想像できる通り、Coroutineが存在できるスコープのことになります。CoroutineScopeは概念ではなく、インターフェイスとしてコード上に存在します。 `launch` や `async` といった関数は `CoroutineScope` に対する拡張関数として定義されています。なので、Coroutineを作成する `launch` や `async` はCoroutineScopeがないと呼び出すことができません。

- [API Reference - kotlinx.coroutines.CoroutineScope](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/)
- [API Reference - kotlinx.coroutines.launch](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html)
- [API Reference - kotlinx.coroutines.async](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html)

CoroutineScopeは `runBlocking` や `coroutineScope` から作成できます。CoroutineScopeを作成する関数をScopeBuilderと言います。CoroutineBuilderと比較すると以下のようなイメージです。

|名前|作るもの|
|:--|:--|
|CoroutineBuilder|Coroutine|
|ScopeBuilder|CoroutineScope|


`runBlocking` と `coroutineScope` はどちらもCoroutineの実行が終わるまで呼び出し元の処理を止めますが、 `runBlocking` は元なるスレッドを開放しません。一方、 `coroutineScope` はスレッドを開放します。このような違いから `runBlocking` はCoroutineの世界へのエントリポイントとしての役割を持っています。main関数やテストの中で呼び出す想定をしています。Coroutineの世界で `runBlocking` を呼び出してしまうと元となるスレッドを開放してくれないため、Coroutineであるべきメリットを握りつぶしてしまうことになります。Coroutineの世界では `coroutienScope` を使うべきです。

- [API Reference - kotlinx.coroutines.run-blocking](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html)
- [API Reference - kotlinx.coroutines.coroutine-scope](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html)

分かりづらいので実際のコードを見たほうがイメージがつかめると思います。