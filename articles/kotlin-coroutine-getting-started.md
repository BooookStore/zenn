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

```kotlin
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

CoroutineBuilderはCoroutineを新しく作成します。[runBlocking](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html)、[launch](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) 、 [async](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) はそれぞれ代表的なCoroutineBuilderです。

|CoroutineBuilder|戻り値|計算結果|
|:--|:--|:--|
|launch|Job|計算結果を返さない|
|async|Deffered|計算結果を返す|
|runBlocking|任意|計算結果を返す|

CoroutineBuilderがCoroutineを作成するイメージはこんな感じになります。

![](/images/coroutine-B.png)

`launch` は `Job` を返します。 `join` を使うことでCoroutineの完了を待機できます。

```kotlin
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

```kotlin
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

```kotlin
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

```kotlin
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

```kotlin
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

CoroutineScopeは名前から想像できる通り、Coroutineが存在できるスコープのことになります。Coroutineが存在するならば、必ずCoroutineScopeも同時に存在することになります。CoroutineScope自体は概念ではなく、インターフェイスとして存在します。

![](/images/coroutine-C.png)

- [API Reference - kotlinx.coroutines.CoroutineScope](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/)

`launch` や `async` といった関数は `CoroutineScope` に対する拡張関数として定義されています。なので、Coroutineを作成する `launch` や `async` はCoroutineScope内にて呼び出すことが強制されます。また、これら２つの関数は `CoroutineScope` をレシーバーとして持つ関数を引数で受け取ります。２つの関数はCoroutineBuilderなのでCoroutineを作成し、その上で何をするのかを引数で受け取ることができるというわけですね。

![](/images/coroutine-D.png)

- [API Reference - kotlinx.coroutines.launch](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html)

```kotlin
fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext, 
    start: CoroutineStart = CoroutineStart.DEFAULT, 
    block: suspend CoroutineScope.() -> Unit
): Job
```

- [API Reference - kotlinx.coroutines.async](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html)

```kotlin
fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext, 
    start: CoroutineStart = CoroutineStart.DEFAULT, 
    block: suspend CoroutineScope.() -> T
): Deferred<T>
```

ここまでの話だと「Coroutineには一つのCoroutineScopeしか存在しないので意識する必要がないのでは？」と思うかもしれません。しかし、 `coroutineScope` 関数を使うことで一つのCoroutineの中に複数のCoroutineScopeを持つことができます。CoroutineScopeを作成する関数をScopeBuilderと言います。

例えば、次の例では２つのCoroutineScopeが１つのCoroutineの中に存在しています。

```kotlin
package booookstore.playground

import kotlinx.coroutines.async
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.runBlocking
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

@Suppress("NonAsciiCharacters")
class CoroutineScopeTest {

    @Test
    fun coroutineScope() = runBlocking {
        val message = secondCoroutineScope()
        assertEquals("hello", message)
    }

    private suspend fun secondCoroutineScope(): String = coroutineScope {
        async {
            delay(100L)
            "hello"
        }.await()
    }

}
```

図にすると以下のようなイメージになります。

![](/images/coroutine-E.png)

`runBlocking` もScopeBuilderです。`runBlocking` `coroutineScope` はどちらもCoroutineの実行が終わるまで呼び出し元の処理を止めますが、 `runBlocking` は元なるスレッドを開放しません。一方、 `coroutineScope` はスレッドを開放します。このような違いから `runBlocking` はCoroutineの世界へのエントリポイントとしての役割を持っています。main関数やテストの中で呼び出す想定をしています。Coroutineの世界で `runBlocking` を呼び出してしまうと元となるスレッドを開放してくれないため、Coroutineであるべきメリットを握りつぶしてしまうことになります。Coroutineの世界では `coroutienScope` を使うべきです。

- [API Reference - kotlinx.coroutines.run-blocking](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html)
- [API Reference - kotlinx.coroutines.coroutine-scope](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html)

分かりづらいので実際のコードを見たほうがイメージがつかめると思います。

``` kotlin
package booookstore.playground

import kotlinx.coroutines.async
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.runBlocking
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

class CoroutineScopeTest {

    @Test
    fun coroutineScope() = runBlocking { // #1
        val message = otherCoroutineScope()
        println("receive message from otherCoroutineScope")
        assertEquals("hello", message)
    }

    private suspend fun otherCoroutineScope(): String = coroutineScope { // #2
        println("entry otherCoroutineScope")
        val deferred = async {
            delay(100L)
            "hello"
        }
        deferred.await()
    }

}
```

# 代表的な関数の違い

|launch|開放する|完了を待たない
|async|開放する|完了を待たない
|runBlocking|開放しない|完了を待つ
|coroutineScope|開放する|完了を待つ