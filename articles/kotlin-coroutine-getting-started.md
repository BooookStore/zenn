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
        runBlocking {
            launch {
                delay(100L)
                message += "World. "
            }
            message += "Hello, "
        }
        message += "from coroutine."

        assertEquals("Hello, World. from coroutine.", message)
    }

}

```

`runBlocking`  はCoroutineBuilderというものです。Builderと呼ばれるとおりCoroutineを作成します。 `runBlock` はNone-Coroutineの世界からCoroutineの世界へのエントリポイントです。`{ }` の内部ではCoroutineを自由に生成することができます。Blockingと呼ばれるとおり、Coroutineの実行完了まで現在のスレッドをブロックします。

`launch` もCoroutineBuilderの一つです。そう、CoroutineBuilderは複数登場しそれぞれ役割が少々違います。 `launch` はCoroutineを新しく生成しますが現在のスレッドをブロックしません。suspend関数によって実行が中断されると現在のスレッドを開放します。

`delay` はsuspend関数の一つです。suspend関数は現在のCoroutineを中断する関数です。ここでは0.1秒Coroutineを中断します。