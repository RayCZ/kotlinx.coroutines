
## Table of contents

* [Select expression (experimental)](#select-expression-experimental)
  * [Selecting from channels](#selecting-from-channels)
  * [Selecting on close](#selecting-on-close)
  * [Selecting to send](#selecting-to-send)
  * [Selecting deferred values](#selecting-deferred-values)
  * [Switch over a channel of deferred values](#switch-over-a-channel-of-deferred-values)

## Select expression (experimental)

Select expression (experimental) ： Select 表達式 (實驗性)

**Select 表達式同時等待多個掛起函數的結果，應用上可以想成在多個協程同時競爭中選擇一個**

Select expression makes it possible to await multiple suspending functions simultaneously and _select_
the first one that becomes available.

Select 表達式可以同時等待多個懸掛函數，並且選擇第一個可用的懸掛函數。

> Select expressions are an experimental feature of `kotlinx.coroutines`. Their API is expected to 
> evolve in the upcoming updates of the `kotlinx.coroutines` library with potentially
> breaking changes.
>
> Select 表達式是 `kotlinx.coroutines` 的實驗性功能。他們的 API 預期在即將到來的 `kotlinx.coroutines` 函式庫更新中發展，可能會發生重大變化。

### Selecting from channels

Selecting from channels ：從多個通道中選擇其中一個

Let us have two producers of strings: `fizz` and `buzz`. The `fizz` produces "Fizz" string every 300 ms:

讓我們有兩個字串的生產者： `fizz` 和 `buzz` 。 `fizz` 每 0.3 秒產生 "Fizz" 字串：

```kotlin
fun CoroutineScope.fizz() = produce<String> {
    while (true) { // sends "Fizz" every 300 ms
        delay(300)
        send("Fizz")
    }
}
```

And the `buzz` produces "Buzz!" string every 500 ms:

而 `buzz` 每 0.5 秒產生 "Buzz!" ：

```kotlin
fun CoroutineScope.buzz() = produce<String> {
    while (true) { // sends "Buzz!" every 500 ms
        delay(500)
        send("Buzz!")
    }
}
```

Using [receive][ReceiveChannel.receive] suspending function we can receive _either_ from one channel or the other. But [select][select] expression allows us to receive from _both_ simultaneously using its [onReceive][ReceiveChannel.onReceive] clauses:

使用 [receive][ReceiveChannel.receive] 懸掛函數，我們可以接收來自一個通道或其他的通道。但 [select][select] 表達式允許我們使用它的 [onReceive][ReceiveChannel.onReceive] 子句，去同時去接收來自兩者。

```kotlin
suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    select<Unit> { // <Unit> means that this select expression does not produce any result 
        fizz.onReceive { value ->  // this is the first select clause
            println("fizz -> '$value'")
        }
        buzz.onReceive { value ->  // this is the second select clause
            println("buzz -> '$value'")
        }
    }
}
```

Let us run it all seven times:

讓我們運行它所有七次：

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

// 生產者，產生 "Fizz" 字串，回傳 ReceiveChannel 類型
fun CoroutineScope.fizz() = produce<String> {
    while (true) { // sends "Fizz" every 300 ms
        delay(300)
        send("Fizz")
    }
}

// 生產者，產生 "Buzz!" 字串，回傳 ReceiveChannel 類型
fun CoroutineScope.buzz() = produce<String> {
    while (true) { // sends "Buzz!" every 500 ms
        delay(500)
        send("Buzz!")
    }
}

// 兩個生產者在 select 表達式中做處理，類似 when 、 if
suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    
    // select 表達式同時等待多個掛起函數的結果，兩個同時接收
    select<Unit> { // <Unit> means that this select expression does not produce any result 
        
        // 對應 fizz 生產者，在接收中
        fizz.onReceive { value ->  // this is the first select clause
            println("fizz -> '$value'")
        }
        
        // 對應 buzz 生產者，在接收中
        buzz.onReceive { value ->  // this is the second select clause
            println("buzz -> '$value'")
        }
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val fizz = fizz()
    val buzz = buzz()
    repeat(7) {
        selectFizzBuzz(fizz, buzz)
    }
    
    // 環境直接取消所有的子協程
    coroutineContext.cancelChildren() // cancel fizz & buzz coroutines
//sampleEnd        
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-select-01.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-select-01.kt)獲取完整的代碼

The result of this code is: 

這些代碼的結果是： 

```text
fizz -> 'Fizz'
buzz -> 'Buzz!'
fizz -> 'Fizz'
fizz -> 'Fizz'
buzz -> 'Buzz!'
fizz -> 'Fizz'
buzz -> 'Buzz!'
```

### Selecting on close

Selecting on close ：在通道關閉中選擇一個

The [onReceive][ReceiveChannel.onReceive] clause in `select` fails when the channel is closed causing the corresponding `select` to throw an exception. We can use [onReceiveOrNull][ReceiveChannel.onReceiveOrNull] clause to perform a specific action when the channel is closed. The following example also shows that `select` is an expression that returns the result of its selected clause:

當通道被關閉時，在 `select` 中的 [onReceive][ReceiveChannel.onReceive] 子句失敗，造成對應的 `select` 丟出異常。當通道被關閉時，我們可以使用 [onReceiveOrNull][ReceiveChannel.onReceiveOrNull] 去執行動作。以下範例也顯示 `select` 表達式回傳它已選子句的結果。

```kotlin
suspend fun selectAorB(a: ReceiveChannel<String>, b: ReceiveChannel<String>): String =
    select<String> {
        a.onReceiveOrNull { value -> 
            if (value == null) 
                "Channel 'a' is closed" 
            else 
                "a -> '$value'"
        }
        b.onReceiveOrNull { value -> 
            if (value == null) 
                "Channel 'b' is closed"
            else    
                "b -> '$value'"
        }
    }
```

Let's use it with channel `a` that produces "Hello" string four times and channel `b` that produces "World" four times:

讓我們使用 `select` 表達式配合通道 `a` 產生 "Hello" 字串 4 次，並且通道 `b` 產生 "world" 字串 4 次：

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

suspend fun selectAorB(a: ReceiveChannel<String>, b: ReceiveChannel<String>): String =
    select<String> { // select 表達式，兩個發送同時還擇一個接收
        
        // a 關閉 value 值會 null 
        a.onReceiveOrNull { value -> 
            if (value == null) 
                "Channel 'a' is closed" 
            else 
                "a -> '$value'"
        }
        
         // b 關閉 value 值會 null
        b.onReceiveOrNull { value -> 
            if (value == null) 
                "Channel 'b' is closed"
            else    
                "b -> '$value'"
        }
    }
    
fun main() = runBlocking<Unit> {
//sampleStart
    
    // 一個生產者協程，重覆產生 4 個數字，超過就沒有
    val a = produce<String> {
        repeat(4) { send("Hello $it") }
    }
    
    // 一個生產者協程，重覆產生 4 個數字，超過就沒有
    val b = produce<String> {
        repeat(4) { send("World $it") }
    }
    
    // 重覆 8 次
    repeat(8) { // print first eight results
        println(selectAorB(a, b))
    }
    coroutineContext.cancelChildren()  
//sampleEnd      
}    
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-select-02.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-select-02.kt)獲取完整的代碼

The result of this code is quite interesting, so we'll analyze it in mode detail:

這些代碼的結果相當有趣，所以我們將在模式細節中分析它：

```text
a -> 'Hello 0'
a -> 'Hello 1'
b -> 'World 0'
a -> 'Hello 2'
a -> 'Hello 3'
b -> 'World 1'
Channel 'a' is closed
Channel 'a' is closed
```

There are couple of observations to make out of it. 

有幾點觀察去理解它。

First of all, `select` is _biased_ to the first clause. When several clauses are selectable at the same time, the first one among them gets selected. Here, both channels are constantly producing strings, so `a` channel, being the first clause in select, wins. However, because we are using unbuffered channel, the `a` gets suspended from time to time on its [send][SendChannel.send] invocation and gives a chance for `b` to send, too.

首先， `select` 是偏向第一個子句。當在同時間可選多個子句時，在它們之中的第一個被選中。這裡，兩個生產者通道不斷的產生字串，所以 `a` 通道，在 `select` 中成為第一個子句獲勝。然而，因為我們使用未緩衝的通道， `a` 生產者通道在它的 [send][SendChannel.send] 有時被懸掛，且對 `b` 也有機會去發送。

The second observation, is that [onReceiveOrNull][ReceiveChannel.onReceiveOrNull] gets immediately selected when the channel is already closed.

第二個觀察，當通道已經被關閉， [onReceiveOrNull][ReceiveChannel.onReceiveOrNull] 中立刻被選中為 null 。

### Selecting to send

Selecting to send ： 選擇性發送

Select expression has [onSend][SendChannel.onSend] clause that can be used for a great good in combination with a biased nature of selection.

`select` 表達式有 [onSend][SendChannel.onSend] 子句，可以與選擇的偏好性良好的結合使用。

Let us write an example of producer of integers that sends its values to a `side` channel when the consumers on its primary channel cannot keep up with it:

讓我們寫一個整數生產者的範例，當在消費者的主要通道中消費者跟不上 `side` 時，生產者發送它的值到 `side` 通道。

```kotlin
fun CoroutineScope.produceNumbers(side: SendChannel<Int>) = produce<Int> {
    for (num in 1..10) { // produce 10 numbers from 1 to 10
        delay(100) // every 100 ms
        select<Unit> {
            onSend(num) {} // Send to the primary channel
            side.onSend(num) {} // or to the side channel     
        }
    }
}
```

Consumer is going to be quite slow, taking 250 ms to process each number:

消費者將相當的慢，花費 0.25 秒去處理每個數值：

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

// 第三個協程，生產者，產生 1~10 每 0.1 秒發送，生產者會回傳 ReceiveChannel 物件
fun CoroutineScope.produceNumbers(side: SendChannel<Int>) = produce<Int> {
    for (num in 1..10) { // produce 10 numbers from 1 to 10
        delay(100) // every 100 ms
        
        // 利用 select 表達式送數字
        select<Unit> {
            // 送給 primary (ReceiveChannel 類型物件)
            onSend(num) {} // Send to the primary channel
            
            // 送給 side (Channel 類型物件) 
            side.onSend(num) {} // or to the side channel     
        }
    }
}

// 第一個協程
fun main() = runBlocking<Unit> {
//sampleStart
    val side = Channel<Int>() // allocate side channel 
    
    // 第二個協程，負責 side 無間隔消費 (Channel 類型物件)
    launch { // this is a very fast consumer for the side channel
        side.consumeEach { println("Side channel has $it") }
    }
    
    // 在第一個協程中，負責 primary 間隔 0.25 秒消費 (ReceiveChannel 類型物件)
    val primary = produceNumbers(side)
    primary.consumeEach { 
        println("Consuming $it")
        delay(250) // let us digest the consumed number properly, do not hurry
    }
    
    println("Done consuming")
    coroutineContext.cancelChildren()  
//sampleEnd      
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-select-03.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-select-03.kt)獲取完整的代碼

So let us see what happens:

所以讓我們看發生什麼事：

```text
Consuming 1
Side channel has 2
Side channel has 3
Consuming 4
Side channel has 5
Side channel has 6
Consuming 7
Side channel has 8
Side channel has 9
Consuming 10
Done consuming
```

### Selecting deferred values

Selecting deferred values ： 選擇其中一個的推遲值

Deferred values can be selected using [onAwait][Deferred.onAwait] clause. Let us start with an async function that returns a deferred string value after a random delay:

可以使用 [onAwait][Deferred.onAwait] 子句選擇推遲的值。讓我們以 async 函數開始，在隨機 delay 之後回傳推遲字串值。

```kotlin
fun CoroutineScope.asyncString(time: Int) = async {
    delay(time.toLong())
    "Waited for $time ms"
}
```

Let us start a dozen of them with a random delay.

讓我們使用隨機延遲開始它們的一打 (12個) 。

```kotlin
fun CoroutineScope.asyncStringsList(): List<Deferred<String>> {
    val random = Random(3)
    return List(12) { asyncString(random.nextInt(1000)) }
}
```

Now the main function awaits for the first of them to complete and counts the number of deferred values that are still active. Note, that we've used here the fact that `select` expression is a Kotlin DSL, so we can provide clauses for it using an arbitrary code. In this case we iterate over a list of deferred values to provide `onAwait` clause for each deferred value.

注意：主函數等待它們的第一個函數完成，並且計算仍處於活動狀態的推遲值數值。注意，我們在這裡使用 `select` 表達式是一個 Kotlin DSL ，所以我們可以為它使用隨便的代碼提供子句。在這樣的情況下，我們遍歷一個推遲值的列表，為每個推遲值提供 `onAwait` 子句。

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.selects.*
import java.util.*
    
// async 函數，回傳 Deferred 類型，依參數做隨機的等待，等待完後，回傳等待的時間 "Waited for $time ms"
fun CoroutineScope.asyncString(time: Int) = async {
    // println("wait $time") 便於查看等待時間
    delay(time.toLong())
    "Waited for $time ms"
}

// 1.產生 12 個 隨機等待 0.003 ~ 1 秒的 Deferred 值到 List
fun CoroutineScope.asyncStringsList(): List<Deferred<String>> {
    val random = Random(3)
    return List(12) { asyncString(random.nextInt(1000)) }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val list = asyncStringsList()
    val result = select<String> {
        
        // 2.withIndex 讓 forEach 有 index 參數
        list.withIndex().forEach { (index, deferred) ->
                                  
            // 3.遍歷 12 個 Deferred，每個都調用 onAwait 等待結果，選擇那個等待時間最少回傳字串
            deferred.onAwait { answer ->
                "Deferred $index produced answer '$answer'"
            }
        }
    }
    println(result)
    val countActive = list.count { it.isActive }
    println("$countActive coroutines are still active")
//sampleEnd
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-select-04.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-select-04.kt)獲取完整的代碼

The output is:

輸出是：

```text
Deferred 4 produced answer 'Waited for 128 ms'
11 coroutines are still active
```

### Switch over a channel of deferred values

Switch over a channel of deferred values ：轉換

Let us write a channel producer function that consumes a channel of deferred string values, waits for each received deferred value, but only until the next deferred value comes over or the channel is closed. This example puts together [onReceiveOrNull][ReceiveChannel.onReceiveOrNull] and [onAwait][Deferred.onAwait] clauses in the same `select`:

讓我們寫一個通道生產者函數，該函數消費推遲類型字串值的通道，等待每個已收到的推遲類型值，但只有在下個推遲之值過來或通道被關閉之前。這個範例在相同的 `select` 表達式放置 [onReceiveOrNull][ReceiveChannel.onReceiveOrNull] 和 [onAwait][Deferred.onAwait] 子句在一起：

```kotlin
fun CoroutineScope.switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String> {
    var current = input.receive() // start with first received deferred value
    while (isActive) { // loop while not cancelled/closed
        val next = select<Deferred<String>?> { // return next deferred value from this select or null
            input.onReceiveOrNull { update ->
                update // replaces next value to wait
            }
            current.onAwait { value ->  
                send(value) // send value that current deferred has produced
                input.receiveOrNull() // and use the next deferred from the input channel
            }
        }
        if (next == null) {
            println("Channel was closed")
            break // out of loop
        } else {
            current = next
        }
    }
}
```

To test it, we'll use a simple async function that resolves to a specified string after a specified time:

為了測試它，我們將使用一個簡單的 async 函數，在指定的時間後解析 (回傳) 指定的字串：

```kotlin
fun CoroutineScope.asyncString(str: String, time: Long) = async {
    delay(time)
    str
}
```

The main function just launches a coroutine to print results of `switchMapDeferreds` and sends some test data to it:

main 函數只是發射一個協程去印出 `switchMapDeferreds` 的結果並發送一些測試資料給它：

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*
    
fun CoroutineScope.switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String> {
    var current = input.receive() // start with first received deferred value
    while (isActive) { // loop while not cancelled/closed
        val next = select<Deferred<String>?> { // return next deferred value from this select or null
            input.onReceiveOrNull { update ->
                update // replaces next value to wait
            }
            current.onAwait { value ->
                // 為 produce 的 send 方法，讓回傳的 ReceiveChannel 類似物件可以印出
                send(value) // send value that current deferred has produced
                input.receiveOrNull() // and use the next deferred from the input channel
            }
        }
        if (next == null) {
            println("Channel was closed")
            break // out of loop
        } else {
            current = next
        }
    }
}

fun CoroutineScope.asyncString(str: String, time: Long) = async {
    delay(time)
    str
}

fun main() = runBlocking<Unit> {
//sampleStart
    val chan = Channel<Deferred<String>>() // the channel for test
    
    // 負責印出訊息的協程
    launch { // launch printing coroutine
        for (s in switchMapDeferreds(chan)) 
            println(s) // print each received string
    }
    
    // 在主協程負責發送訊息和 async 函數之中的等待時間
    chan.send(asyncString("BEGIN", 100))
    delay(200) // enough time for "BEGIN" to be produced
    chan.send(asyncString("Slow", 500))
    delay(100) // not enough time to produce slow
    chan.send(asyncString("Replace", 100))
    delay(500) // give it time before the last one
    chan.send(asyncString("END", 500))
    delay(1000) // give it time to process
    chan.close() // close the channel ... 
    delay(500) // and wait some time to let it finish
//sampleEnd
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-select-05.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-select-05.kt)獲取完整的代碼

The result of this code:

這些代碼的結果：

```text
BEGIN
Replace
END
Channel was closed
```

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[Deferred.onAwait]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/on-await.html
<!--- INDEX kotlinx.coroutines.channels -->
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
[ReceiveChannel.onReceive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/on-receive.html
[ReceiveChannel.onReceiveOrNull]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/on-receive-or-null.html
[SendChannel.send]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/send.html
[SendChannel.onSend]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/on-send.html
<!--- INDEX kotlinx.coroutines.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/select.html
<!--- END -->
