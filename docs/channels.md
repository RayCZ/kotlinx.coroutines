## Table of contents

Table of contents ：內容表

* [Channels (experimental)](#channels-experimental)
  * [Channel basics](#channel-basics)
  * [Closing and iteration over channels](#closing-and-iteration-over-channels)
  * [Building channel producers](#building-channel-producers)
  * [Pipelines](#pipelines)
  * [Prime numbers with pipeline](#prime-numbers-with-pipeline)
  * [Fan-out](#fan-out)
  * [Fan-in](#fan-in)
  * [Buffered channels](#buffered-channels)
  * [Channels are fair](#channels-are-fair)
  * [Ticker channels](#ticker-channels)

## Channels (experimental) 

Channels (experimental) ：通道 (實驗性)

Deferred values provide a convenient way to transfer a single value between coroutines. Channels provide a way to transfer a stream of values.

延期的值，提供協程之間傳輸單一值的便利方式。通道提供傳輸值的串流方式。

> Channels are an experimental feature of `kotlinx.coroutines`. Their API is expected to evolve in the upcoming updates of the `kotlinx.coroutines` library with potentially breaking changes.
>
> 通道是 `kotlinx.coroutines` 的實驗性功能。他們的 API 預期在即將到來的 `kotlinx.coroutines` 函式庫更新中發展，可能會發生重大變化。

### Channel basics

Channel basics ：通道基礎

A [Channel][Channel] is conceptually very similar to `BlockingQueue`. One key difference is that instead of a blocking `put` operation it has a suspending [send][SendChannel.send], and instead of a blocking `take` operation it has a suspending [receive][ReceiveChannel.receive].

[通道][Channel]的概念與 `BlockingQueue` 非常相似。一個關鍵的不同是，代替阻塞的 `put` 操作，它有懸掛函數 `send` ，而且代替阻塞的 `take` 操作，它有懸掛函數 `receive` 。 

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    val channel = Channel<Int>()
    launch {
        // this might be heavy CPU-consuming computation or async logic, we'll just send five squares
        for (x in 1..5) channel.send(x * x)
    }
    // here we print five received integers:
    repeat(5) { println(channel.receive()) }
    println("Done!")
//sampleEnd
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-01.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-01.kt)獲取完整的代碼

The output of this code is:

這些代碼輸出是：

```text
1
4
9
16
25
Done!
```

### Closing and iteration over channels 

Closing and iteration over channels ：關閉和遍歷通道

Unlike a queue, a channel can be closed to indicate that no more elements are coming. On the receiver side it is convenient to use a regular `for` loop to receive elements from the channel. 

不像佇列，通道可以被關閉 `channel.close()` 去指示沒有更多的元素到來。在接收端它使用常規 `for` 環循從通道接收元素是方便的。

Conceptually, a [close][SendChannel.close] is like sending a special close token to the channel. The iteration stops as soon as this close token is received, so there is a guarantee that all previously sent elements before the close are received:

觀念上， [close][SendChannel.close] 函數像是發送一個特別結束記號給通道。一旦收到這個關閉的記號，遍歷停止，所以可以保證在收到閉關時所有先前發送的元素。

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) channel.send(x * x)
        channel.close() // we're done sending
    }
    // here we print received values using `for` loop (until the channel is closed)
    for (y in channel) println(y)
    println("Done!")
//sampleEnd
}

//ans:
//1
//4
//9
//16
//25
//Done!
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-02.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-02.kt)獲取完整的代碼

### Building channel producers

Building channel producers ：建造通道生產者

The pattern where a coroutine is producing a sequence of elements is quite common. This is a part of _producer-consumer_ pattern that is often found in concurrent code. You could abstract such a producer into a function that takes channel as its parameter, but this goes contrary to common sense that results must be returned from functions. 

協程生成一系列的元素是很常見的模式。這是**生產者-消費者**模式的一部份，通常在並發行為的代碼常發現。你可以抽象化 (提取) 這樣的生產者到函數，帶入通道當作它的參數。而是這與必須從函式回傳結果的常識相反。

There is a convenient coroutine builder named [produce][produce] that makes it easy to do it right on producer side, and an extension function [consumeEach][consumeEach], that replaces a `for` loop on the consumer side:

這有一個方便的協程建造者命為 [produce][produce] ，在生產者端使它容易完成。以及擴展函數 [consumeEach][consumeEach] ，它在消費者端代替 `for` 循環：

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

// CoroutineScope 新增擴展函數 produceSquares() 、回傳 ReceiveChannel<Int> 、生成一個協程在 produce {}  中處理，而回傳的值做取出的動作
fun CoroutineScope.produceSquares(): ReceiveChannel<Int> = produce {
    for (x in 1..5) send(x * x)
}

fun main() = runBlocking {
//sampleStart
    
    // ReceiveChannel<Int> 物件
    val squares = produceSquares()
    
    // ReceiveChannel<Int> 物件為消費者端做接收處理，並減少 for 循環
    squares.consumeEach { println(it) }
    println("Done!")
//sampleEnd
}

//ans:
//1
//4
//9
//16
//25
//Done!
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-03.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-03.kt)獲取完整的代碼

### Pipelines

Pipelines ：管道

A pipeline is a pattern where one coroutine is producing, possibly infinite, stream of values:

管道是一個協程正在生成的模式，可能是無限的，值的串流 (一個接續一個)：

```kotlin
// 生成一個協程，無限的送出數值
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // infinite stream of integers starting from 1
}
```

And another coroutine or coroutines are consuming that stream, doing some processing, and producing some other results. In the example below, the numbers are just squared:

並且另一個協程或多個協程正在消費這些串流，做某些處理，以及生成某些其他的結果。在下面的範例，只是求平方值的數值：

```kotlin
// 生成一個協程，做平方處理
fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (x in numbers) send(x * x)
}
```

The main code starts and connects the whole pipeline:

main 函數的代碼啟動和連結整個管道：

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    
    // ReceiveChannel 物件，做取出的動作
    val numbers = produceNumbers() // produces integers from 1 and on
    
    // ReceiveChannel 物件
    val squares = square(numbers) // squares integers
    
    // 接送發送的數值
    for (i in 1..5) println(squares.receive()) // print first five
    println("Done!") // we are done
    coroutineContext.cancelChildren() // cancel children coroutines
//sampleEnd
}

// 本範例目前沒用到
fun CoroutineScope.produceSquares(): ReceiveChannel<Int> = produce {
    for (x in 1..5) send(x * x)
}

// 生成一個 produce 協程，發送數值
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // infinite stream of integers starting from 1
}

// 生成一個 produce 協程，將前一個發送的數值，循環取出做平方處理，再發送出去
fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (x in numbers) send(x * x)
}

//ans:
//1
//4
//9
//16
//25
//Done!
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-04.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-04.kt)獲取完整的代碼

> All functions that create coroutines are defined as extensions on [CoroutineScope][CoroutineScope], so that we can rely on [structured concurrency](https://github.com/RayCZ/kotlinx.coroutines/blob/ray/docs/composing-suspending-functions.md#structured-concurrency-with-async) to make sure that we don't have lingering global coroutines in our application.
>
> 所有函數建立的協程被定義為 [CoroutineScope][CoroutineScope] 擴展函數，以便我們可以依賴[結構化並發](https://github.com/RayCZ/kotlinx.coroutines/blob/ray/docs/composing-suspending-functions.md#structured-concurrency-with-async)，確保我們不會有常駐全域的協程在我們的應用程式。

### Prime numbers with pipeline

Prime numbers with pipeline ：使用管道處理質數

Let's take pipelines to the extreme with an example that generates prime numbers using a pipeline 
of coroutines. We start with an infinite sequence of numbers. 

讓我們帶管道到極端的例子，該例子使用協程的管道生成質數。我們從生成無限數值的序列開始。

```kotlin
// 生成一個協程，處理無限的生成數值
fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++) // infinite stream of integers from start
}
```

The following pipeline stage filters an incoming stream of numbers, removing all the numbers that are divisible by the given prime number:

以下管道階段過濾傳入的數值串流，刪除由參數 `prime` 傳入被整除的數值。

```kotlin
// 生成一個協程，過濾質數， x % prime != 0 餘數才可以送出
fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

Now we build our pipeline by starting a stream of numbers from 2, taking a prime number from the current channel, and launching new pipeline stage for each prime number found:

現在，我們開始從 2 的數值串流，建立我們的管道，從目前管道帶入一個值數，並為找到的每個質數，發射新的管道階段： 

```
numbersFrom(2) -> filter(2) -> filter(3) -> filter(5) -> filter(7) ... 
```

The following example prints the first ten prime numbers, running the whole pipeline in the context of the main thread. Since all the coroutines are launched in the scope of the main [runBlocking][runBlocking] coroutine we don't have to keep an explicit list of all the coroutines we have started. We use [cancelChildren][kotlin.coroutines.CoroutineContext.cancelChildren] extension function to cancel all the children coroutines after we have printed the first ten prime numbers. 

以下範例印出前十個質數，在主線程中正在執行整個管道。因為所有的協程在主要的 [runBlocking][runBlocking] 協程範圍中發射，我們不必明確保留我們已經發射的所有協程列表。在我們已經印出前十個質值後，我們使用 [cancelChildren][kotlin.coroutines.CoroutineContext.cancelChildren] 擴展函數去取消所有的子協程。

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    
    // 擴展函數，建立數值串流
    var cur = numbersFrom(2)
    
    // 處理十個質數
    for (i in 1..10) {
        val prime = cur.receive()
        println(prime)
        cur = filter(cur, prime)
    }
    
    // 在 runBlocking 範圍中取消所有子協程
    coroutineContext.cancelChildren() // cancel all children to let main finish
//sampleEnd    
}

// 建立一個協程，處理無限的生成數值
fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++) // infinite stream of integers from start
}

// 建立一個協程，過濾質數
fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-05.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-05.kt)獲取完整的代碼

The output of this code is:

這些代碼的輸出：

```text
2
3
5
7
11
13
17
19
23
29
```

Note, that you can build the same pipeline using [`buildIterator`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.experimental/build-iterator.html) coroutine builder from the standard library. Replace `produce` with `buildIterator`, `send` with c, `receive` with `next`, `ReceiveChannel` with `Iterator`, and get rid of the coroutine scope. You will not need `runBlocking` either. However, the benefit of a pipeline that uses channels as shown above is that it can actually use multiple CPU cores if you run it in [Dispatchers.Default][Dispatchers.Default] context.

注意：我們可以從標準的函式庫使用 [`buildIterator`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.experimental/build-iterator.html) 協程建造者建立相同的管道。使用 `buildIterator` 代替 `produce` ， 使用 `yield` 代替 `send` ，使用 `next` 代替 `receive` ，使用 `Iterator` 代替 `ReceiveChannel` ，並擺脫協程範圍。你也將不需要 `runBlocking` 。然而，如上所示使用通道的管道好處是，如果你在 [Dispatchers.Default][Dispatchers.Default] 內容中執行它，它實際上可以使用多個 CPU 核心處理。

Anyway, this is an extremely impractical way to find prime numbers. In practice, pipelines do involve some other suspending invocations (like asynchronous calls to remote services) and these pipelines cannot be built using `buildSequence`/`buildIterator`, because they do not allow arbitrary suspension, unlike `produce`, which is fully asynchronous.

無論如何，這是極端不切實際的方式去找質數。在實踐中，管道確實涉及一些其他的懸掛調用 (像異步調用遠端服務) 並且這些管道不可以使用 `buildSequence`/`buildIterator` 建立，因為它們不會隨便允許懸掛，不像 `produce` ，它是完全的異步。

### Fan-out

Multiple coroutines may receive from the same channel, distributing work between themselves.
Let us start with a producer coroutine that is periodically producing integers 
(ten numbers per second):

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1 // start from 1
    while (true) {
        send(x++) // produce next
        delay(100) // wait 0.1s
    }
}
```

</div>

Then we can have several processor coroutines. In this example, they just print their id and
received number:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }    
}
```

</div>

Now let us launch five processors and let them work for almost a second. See what happens:

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
//sampleStart
    val producer = produceNumbers()
    repeat(5) { launchProcessor(it, producer) }
    delay(950)
    producer.cancel() // cancel producer coroutine and thus kill them all
//sampleEnd
}

fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1 // start from 1
    while (true) {
        send(x++) // produce next
        delay(100) // wait 0.1s
    }
}

fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }    
}
```

</div>

> You can get full code [here](../core/kotlinx-coroutines-core/test/guide/example-channel-06.kt)

The output will be similar to the the following one, albeit the processor ids that receive
each specific integer may be different:

```
Processor #2 received 1
Processor #4 received 2
Processor #0 received 3
Processor #1 received 4
Processor #3 received 5
Processor #2 received 6
Processor #4 received 7
Processor #0 received 8
Processor #1 received 9
Processor #3 received 10
```

<!--- TEST lines.size == 10 && lines.withIndex().all { (i, line) -> line.startsWith("Processor #") && line.endsWith(" received ${i + 1}") } -->

Note, that cancelling a producer coroutine closes its channel, thus eventually terminating iteration
over the channel that processor coroutines are doing.

Also, pay attention to how we explicitly iterate over channel with `for` loop to perform fan-out in `launchProcessor` code. 
Unlike `consumeEach`, this `for` loop pattern is perfectly safe to use from multiple coroutines. If one of the processor 
coroutines fails, then others would still be processing the channel, while a processor that is written via `consumeEach` 
always consumes (cancels) the underlying channel on its normal or abnormal completion.     

### Fan-in

Multiple coroutines may send to the same channel.
For example, let us have a channel of strings, and a suspending function that 
repeatedly sends a specified string to this channel with a specified delay:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

</div>

Now, let us see what happens if we launch a couple of coroutines sending strings 
(in this example we launch them in the context of the main thread as main coroutine's children):

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    val channel = Channel<String>()
    launch { sendString(channel, "foo", 200L) }
    launch { sendString(channel, "BAR!", 500L) }
    repeat(6) { // receive first six
        println(channel.receive())
    }
    coroutineContext.cancelChildren() // cancel all children to let main finish
//sampleEnd
}

suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

</div>

> You can get full code [here](../core/kotlinx-coroutines-core/test/guide/example-channel-07.kt)

The output is:

```text
foo
foo
BAR!
foo
foo
BAR!
```

<!--- TEST -->

### Buffered channels

The channels shown so far had no buffer. Unbuffered channels transfer elements when sender and receiver 
meet each other (aka rendezvous). If send is invoked first, then it is suspended until receive is invoked, 
if receive is invoked first, it is suspended until send is invoked.

Both [Channel()] factory function and [produce] builder take an optional `capacity` parameter to
specify _buffer size_. Buffer allows senders to send multiple elements before suspending, 
similar to the `BlockingQueue` with a specified capacity, which blocks when buffer is full.

Take a look at the behavior of the following code:


<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
//sampleStart
    val channel = Channel<Int>(4) // create buffered channel
    val sender = launch { // launch sender coroutine
        repeat(10) {
            println("Sending $it") // print before sending each element
            channel.send(it) // will suspend when buffer is full
        }
    }
    // don't receive anything... just wait....
    delay(1000)
    sender.cancel() // cancel sender coroutine
//sampleEnd    
}
```

</div>

> You can get full code [here](../core/kotlinx-coroutines-core/test/guide/example-channel-08.kt)

It prints "sending" _five_ times using a buffered channel with capacity of _four_:

```text
Sending 0
Sending 1
Sending 2
Sending 3
Sending 4
```

<!--- TEST -->

The first four elements are added to the buffer and the sender suspends when trying to send the fifth one.

### Channels are fair

Send and receive operations to channels are _fair_ with respect to the order of their invocation from 
multiple coroutines. They are served in first-in first-out order, e.g. the first coroutine to invoke `receive` 
gets the element. In the following example two coroutines "ping" and "pong" are 
receiving the "ball" object from the shared "table" channel. 


<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

//sampleStart
data class Ball(var hits: Int)

fun main() = runBlocking {
    val table = Channel<Ball>() // a shared table
    launch { player("ping", table) }
    launch { player("pong", table) }
    table.send(Ball(0)) // serve the ball
    delay(1000) // delay 1 second
    coroutineContext.cancelChildren() // game over, cancel them
}

suspend fun player(name: String, table: Channel<Ball>) {
    for (ball in table) { // receive the ball in a loop
        ball.hits++
        println("$name $ball")
        delay(300) // wait a bit
        table.send(ball) // send the ball back
    }
}
//sampleEnd
```

</div>

> You can get full code [here](../core/kotlinx-coroutines-core/test/guide/example-channel-09.kt)

The "ping" coroutine is started first, so it is the first one to receive the ball. Even though "ping"
coroutine immediately starts receiving the ball again after sending it back to the table, the ball gets
received by the "pong" coroutine, because it was already waiting for it:

```text
ping Ball(hits=1)
pong Ball(hits=2)
ping Ball(hits=3)
pong Ball(hits=4)
```

<!--- TEST -->

Note, that sometimes channels may produce executions that look unfair due to the nature of the executor
that is being used. See [this issue](https://github.com/Kotlin/kotlinx.coroutines/issues/111) for details.

### Ticker channels

Ticker channel is a special rendezvous channel that produces `Unit` every time given delay passes since last consumption from this channel.
Though it may seem to be useless standalone, it is a useful building block to create complex time-based [produce] 
pipelines and operators that do windowing and other time-dependent processing.
Ticker channel can be used in [select] to perform "on tick" action.

To create such channel use a factory method [ticker]. 
To indicate that no further elements are needed use [ReceiveChannel.cancel] method on it.

Now let's see how it works in practice:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
    val tickerChannel = ticker(delayMillis = 100, initialDelayMillis = 0) // create ticker channel
    var nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Initial element is available immediately: $nextElement") // initial delay hasn't passed yet

    nextElement = withTimeoutOrNull(50) { tickerChannel.receive() } // all subsequent elements has 100ms delay
    println("Next element is not ready in 50 ms: $nextElement")

    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() }
    println("Next element is ready in 100 ms: $nextElement")

    // Emulate large consumption delays
    println("Consumer pauses for 150ms")
    delay(150)
    // Next element is available immediately
    nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Next element is available immediately after large consumer delay: $nextElement")
    // Note that the pause between `receive` calls is taken into account and next element arrives faster
    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() } 
    println("Next element is ready in 50ms after consumer pause in 150ms: $nextElement")

    tickerChannel.cancel() // indicate that no more elements are needed
}
```

</div>

> You can get full code [here](../core/kotlinx-coroutines-core/test/guide/example-channel-10.kt)

It prints following lines:

```text
Initial element is available immediately: kotlin.Unit
Next element is not ready in 50 ms: null
Next element is ready in 100 ms: kotlin.Unit
Consumer pauses for 150ms
Next element is available immediately after large consumer delay: kotlin.Unit
Next element is ready in 50ms after consumer pause in 150ms: kotlin.Unit
```

<!--- TEST -->

Note that [ticker] is aware of possible consumer pauses and, by default, adjusts next produced element 
delay if a pause occurs, trying to maintain a fixed rate of produced elements.

Optionally, a `mode` parameter equal to [TickerMode.FIXED_DELAY] can be specified to maintain a fixed
delay between elements.  


<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[kotlin.coroutines.CoroutineContext.cancelChildren]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/kotlin.coroutines.-coroutine-context/cancel-children.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
<!--- INDEX kotlinx.coroutines.channels -->
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/index.html
[SendChannel.send]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/send.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
[SendChannel.close]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/close.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
[consumeEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/consume-each.html
[Channel()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel.html
[ticker]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/ticker.html
[ReceiveChannel.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/cancel.html
[TickerMode.FIXED_DELAY]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-ticker-mode/-f-i-x-e-d_-d-e-l-a-y.html
<!--- INDEX kotlinx.coroutines.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/select.html
<!--- END -->
