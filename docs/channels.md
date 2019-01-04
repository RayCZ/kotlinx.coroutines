**Table of contents**

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

[通道][Channel]的概念與 `BlockingQueue` 非常相似。一個關鍵的不同是，代替阻塞的 `put` 操作，它有懸掛函數 [`send`][SendChannel.send] ，而且代替阻塞的 `take` 操作，它有懸掛函數 [`receive`][ReceiveChannel.receive] 。 

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

Fan-out ：扇出 (由一個點擴散多個點出去像扇子一樣，以下例子：一個生產、多個消費)

Multiple coroutines may receive from the same channel, distributing work between themselves. Let us start with a producer coroutine that is periodically producing integers (ten numbers per second):

多協程可以從相同的通道接受，在它們之間分配工作。讓我們以生產者協程開始，定期生產整數 (1 秒 10 個數值) ：

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1 // start from 1
    while (true) {
        send(x++) // produce next
        delay(100) // wait 0.1s
    }
}
```

Then we can have several processor coroutines. In this example, they just print their id and received number:

接著我們可以有幾個處理器協程，在這個例子，它們就是印出它們的 id 和已接收的數值：

```kotlin
fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }    
}
```

Now let us launch five processors and let them work for almost a second. See what happens:

現在，讓我們發射五個處理器，並讓它們工作幾乎一秒 `delay(950)` 。看發生什麼事：

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
//sampleStart
    
    // 開始自動生成數字的協程
    val producer = produceNumbers()
    
    // 呼叫五次產生五個協程，五個協程異步的取得數字
    repeat(5) { launchProcessor(it, producer) }
    delay(950)
    producer.cancel() // cancel producer coroutine and thus kill them all
//sampleEnd
}

// 建立一個協程，持續每 0.1 秒生成數字
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1 // start from 1
    while (true) {
        send(x++) // produce next
        delay(100) // wait 0.1s
    }
}


// 每次呼叫建立一個協程，都去跟生成數字的協程取得數字
fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-06.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-06.kt)獲取完整的代碼

The output will be similar to the the following one, albeit the processor ids that receive each specific integer may be different:

輸出將類似於以下輸出，儘管接收每個特定整數的處理器 id 可能會不同：

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

Note, that cancelling a producer coroutine closes its channel, thus eventually terminating iteration over the channel that processor coroutines are doing.

注意：取消生產者協程關閉它的通道，因此最終，終止每個處理器協程正在進行遍歷 `for` 的通道。

Also, pay attention to how we explicitly iterate over channel with `for` loop to perform fan-out in `launchProcessor` code. Unlike `consumeEach`, this `for` loop pattern is perfectly safe to use from multiple coroutines. If one of the processor coroutines fails, then others would still be processing the channel, while a processor that is written via `consumeEach` always consumes (cancels) the underlying channel on its normal or abnormal completion.

另外，注意我們如何在 `launchProcessor` 代碼中執行扇出使用 `for` 循環明確遍歷。不像 `consumeEach` ，這種 `for` 循環模式從多個協程中使用是非常安全。如果處理器協程之一失敗，接著其他的處理器協程仍然會處理通道，而通過 `consumeEach` 寫入處理器總是在它正常或非正常完成時消費 (取消) 底層的通道。

### Fan-in

Fan-in  ：扇入 (由多個點聚集一個像扇子一樣，以下例子：多個生產、一個消費)

Multiple coroutines may send to the same channel. For example, let us have a channel of strings, and a suspending function that repeatedly sends a specified string to this channel with a specified delay:

多個協程可能送出給相同通道。例如，讓我們有一個字串的通道，並且新增一個懸掛函數，重覆的指定延遲時間及送出指定的字串給這個通道：

```kotlin
suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time) // 延屬時間
        channel.send(s) // 送給通道
    }
}
```

Now, let us see what happens if we launch a couple of coroutines sending strings (in this example we launch them in the context of the main thread as main coroutine's children):

現在，讓我們看發生什麼事，如果我們發射幾個協程送出字串 (在這個例子中，我們在主線程環境中作為主協程的後代) ：

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    
    // 字串通道
    val channel = Channel<String>()
    
    // 新增一個協程每 0.2 秒發送一個 "foo"
    launch { sendString(channel, "foo", 200L) }
    
    // 新增一個協程每 0.5 秒發送一個 "BAR!"
    launch { sendString(channel, "BAR!", 500L) }
    
    // 重覆取六次
    repeat(6) { // receive first six
        println(channel.receive())
    }
    coroutineContext.cancelChildren() // cancel all children to let main finish
//sampleEnd
}

// 懸掛函數處理延遲跟發送
suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

> You can get full code [here](https://github.com/hltj/kotlinx.coroutines-cn/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-07.kt)
>
> 你可以在[這裡](https://github.com/hltj/kotlinx.coroutines-cn/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-07.kt)獲取完整的代碼

The output is:

輸出：

```text
foo 0.2s
foo 0.2s
BAR! 0.5s
foo 0.2s
foo 0.2s
BAR! 0.5s
```

**扇出與扇入的差別：「扇出」是利用 produce 協程作為參數，送給每個處理器的協程取去取得值，「扇入」是利用 Channel 的 API 處理，當然也可以做「扇出」的行為**

### Buffered channels

Buffered channels ：緩衝的通道

The channels shown so far had no buffer. Unbuffered channels transfer elements when sender and receiver meet each other (aka rendezvous). If send is invoked first, then it is suspended until receive is invoked, if receive is invoked first, it is suspended until send is invoked.

到目前為止展示的通道沒有緩衝。當發送者跟接收著彼此相遇 (又名會合) 時，無緩衝通道傳輸元素。如果首先調用發送 ，接著在調用接收之前它是懸掛的，如果首先調用接收，在調用發送之前它是懸掛的，。

Both [Channel()][Channel()] factory function and [produce][produce] builder take an optional `capacity` parameter to specify _buffer size_. Buffer allows senders to send multiple elements before suspending, similar to the `BlockingQueue` with a specified capacity, which blocks when buffer is full.

[Channel()][Channel()] 工廠函數與 [produce][produce] 建造者帶一個可選的 `capacity` 參數去指定**緩衝大小**。在懸掛之前緩衝允許發送者去發送多個元素，類似於使用指定容量的 `BlockingQueue` ，當緩衝是滿的時候阻塞。

Take a look at the behavior of the following code:

看一下以下代碼的行為：

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
//sampleStart
    
    // 指定 4 個緩衝
    val channel = Channel<Int>(4) // create buffered channel
    
    // 建立協程重覆發送字串
    val sender = launch { // launch sender coroutine
        repeat(10) {
            println("Sending $it") // print before sending each element
            channel.send(it) // will suspend when buffer is full
        }
    }
    
    // 主協程延遲一秒
    // don't receive anything... just wait....
    delay(1000)
    sender.cancel() // cancel sender coroutine
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-08.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-08.kt)獲取完整的代碼

It prints "sending" _five_ times using a buffered channel with capacity of _four_:

它使用 4 的容量緩衝通道，印出 "sending" 5 次使用：

```text
Sending 0
Sending 1
Sending 2
Sending 3
Sending 4
```

The first four elements are added to the buffer and the sender suspends when trying to send the fifth one.

首先添加 4 個元素到緩衝，當嘗試送出第 5 個元素時發送者懸掛 (暫停)。

### Channels are fair

Channels are fair ：通道是公平

Send and receive operations to channels are _fair_ with respect to the order of their invocation from multiple coroutines. They are served in first-in first-out order, e.g. the first coroutine to invoke `receive` gets the element. In the following example two coroutines "ping" and "pong" are receiving the "ball" object from the shared "table" channel. 

從多個協程調用它們的順序，向通道發送與接收操作是公平的。它們在先進先出的順序中提供，例如第一個協程調用 `receive` 取得元素。在以下例子兩個協程 "ping" 和 "pong" 正在從共享的 "table" 通道接收 "ball" 物件。

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

//sampleStart
data class Ball(var hits: Int)

fun main() = runBlocking {
    // 共享通道
    val table = Channel<Ball>() // a shared table
    
    // 一個協程，發送 ping 
    launch { player("ping", table) }
    
    // 一個協程，發送 pong
    launch { player("pong", table) }
    
    // 提供最初始的 ball , hit = 0
    table.send(Ball(0)) // serve the ball
    delay(1000) // delay 1 second
    coroutineContext.cancelChildren() // game over, cancel them
}

suspend fun player(name: String, table: Channel<Ball>) {
    for (ball in table) { // receive the ball in a loop
        ball.hits++
        println("$name $ball")
        
        // 延遲 0.3 秒
        delay(300) // wait a bit
        
        // 重覆發送，但 hit 每次 +1
        table.send(ball) // send the ball back
    }
}
//sampleEnd
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-09.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-09.kt)獲取完整的代碼

The "ping" coroutine is started first, so it is the first one to receive the ball. Even though "ping" coroutine immediately starts receiving the ball again after sending it back to the table, the ball gets received by the "pong" coroutine, because it was already waiting for it:

"ping" 協程首先啟動，所以它是第一個去接收球的協程。即使 "ping" 協程送它回桌面之後，再次即時啟動接收球，球也會被由 "pong" 協程接收，因為它已經等待接收球：

```text
ping Ball(hits=1)
pong Ball(hits=2)
ping Ball(hits=3)
pong Ball(hits=4)
```

Note, that sometimes channels may produce executions that look unfair due to the nature of the executor that is being used. See [this issue](https://github.com/Kotlin/kotlinx.coroutines/issues/111) for details.

注意：由於執行者正在使用的性質，有時通道可能產生執行看起來不公平。更多細節參閱 [this issue](https://github.com/Kotlin/kotlinx.coroutines/issues/111) 。

### Ticker channels

Ticker channels ：定時器通道 (定期生成器)

Ticker channel is a special rendezvous channel that produces `Unit` every time given delay passes since last consumption from this channel. Though it may seem to be useless standalone, it is a useful building block to create complex time-based [produce][produce] pipelines and operators that do windowing and other time-dependent processing. Ticker channel can be used in [select][select] to perform "on tick" action.

定時器通道是特殊的約定會合通道，從上次開始由這個通道的消耗，每次給定延遲通過時生成 `Unit`。雖然它可能似乎單獨無用，但它是一個有用的建造時間阻斷，用來建立以時間為基礎複雜的 [produce][produce](生產) 管線和運算符，進行窗口和其他時間相關的處理。定時器通道可以被用在 [select][select](選擇懸掛中的協程) 去執行 "定期調用" 的動作。

To create such channel use a factory method [ticker][ticker]. To indicate that no further elements are needed use [ReceiveChannel.cancel][ReceiveChannel.cancel] method on it.

創建這樣的通道使用工廠方法 [ticker][ticker] 。表示在 ticker 回傳物件中沒有更進一步的元素被需要使用 [ReceiveChannel.cancel][ReceiveChannel.cancel] 方法。

Now let's see how it works in practice:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
    // delayMillis 每次延遲時間 0.1 秒 ， initialDelayMillis 第一次發送的延遲時間 0 秒
    val tickerChannel = ticker(delayMillis = 100, initialDelayMillis = 0) // create ticker channel
    
    // 第一次取值，如果沒有值就回傳 null ，有就 Unit
    var nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Initial element is available immediately: $nextElement") // initial delay hasn't passed yet

    // 超過 0.05 秒就超時，回傳 null ，還沒到 0.1 秒生成的時間，所以取不到
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

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-10.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-channel-10.kt)獲取完整的代碼

It prints following lines:

它印出以下行：

```text
Initial element is available immediately: kotlin.Unit
Next element is not ready in 50 ms: null
Next element is ready in 100 ms: kotlin.Unit
Consumer pauses for 150ms
Next element is available immediately after large consumer delay: kotlin.Unit
Next element is ready in 50ms after consumer pause in 150ms: kotlin.Unit
```

Note that [ticker][ticker] is aware of possible consumer pauses and, by default, adjusts next produced element delay if a pause occurs, trying to maintain a fixed rate of produced elements.

注意： [ticker][ticker] 定時器可能知道消費者暫停，並且，預設模式下 [TickerMode.FIXED_PERIOD](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-ticker-mode/-f-i-x-e-d_-p-e-r-i-o-d.html)，如果暫停發生，調整下一個生成元素的延遲，嘗試維持生成元素的固定速率。

Optionally, a `mode` parameter equal to [TickerMode.FIXED_DELAY][TickerMode.FIXED_DELAY] can be specified to maintain a fixed delay between elements.  

可選的， `mode` 參數相當於可以指定 [TickerMode.FIXED_DELAY][TickerMode.FIXED_DELAY] 在元素之間維持固定的延遲時間。

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
