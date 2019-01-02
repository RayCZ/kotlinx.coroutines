# Guide to reactive streams with coroutines

Guide to reactive streams with coroutines ：使用協程配合 Reactive Streams 函式庫的指導

This guide explains key differences between Kotlin coroutines and reactive streams and shows how they can be used together for greater good. Prior familiarity with basic coroutine concepts that are covered in [Guide to kotlinx.coroutines](../docs/coroutines-guide.md) is not required, but is a big plus. If you are familiar with reactive streams, you may find this guide a better introduction into the world of coroutines.

這份指導解釋在 Kotlin 協程跟 Reactive Streams 函式庫的關鍵區別，並展示它們如何一起使用的更大好處。熟悉之前在 [kotlinx.coroutines 指導](../docs/coroutines-guide.md)章節涵蓋基本協程觀念不是必須的，但是一個大優勢。如果你熟悉使用 Reactive Streams 函式庫，你可以發現這份指導更好的介紹協程的世界。

There are several modules in `kotlinx.coroutines` project that are related to reactive streams:

在 `kotlinx.coroutines` 專案有幾個模組與 Reactive Streams 函式庫相關的。

* [kotlinx-coroutines-reactive](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-reactive) -- utilities for [Reactive Streams](http://www.reactive-streams.org)
* [kotlinx-coroutines-reactor](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-reactor) -- utilities for [Reactor](https://projectreactor.io)
* [kotlinx-coroutines-rx2](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2) -- utilities for [RxJava 2.x](https://github.com/ReactiveX/RxJava)

This guide is mostly based on [Reactive Streams](http://www.reactive-streams.org) specification and uses its `Publisher` interface with some examples based on [RxJava 2.x](https://github.com/ReactiveX/RxJava), which implements reactive streams specification.

這份指導主要的根據 [Reactive Streams](http://www.reactive-streams.org) 規範，並根據 [RxJava 2.x](https://github.com/ReactiveX/RxJava) 使用它的 `Publisher` 介面與某些範例， Rxjava 2.x 實作 Reactive Streams 規範。

You are welcome to clone [`kotlinx.coroutines` project](https://github.com/Kotlin/kotlinx.coroutines) from GitHub to your workstation in order to run all the presented examples. They are contained in [reactive/kotlinx-coroutines-rx2/test/guide](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide) directory of the project.

歡迎你從 Github 複製 [`kotlinx.coroutines` 專案](https://github.com/Kotlin/kotlinx.coroutines) 到你的工作站，以便運行所有呈現的範例。他們被包含在 [reactive/kotlinx-coroutines-rx2/test/guide](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide) 專案的目錄下。

## Table of contents

* [Differences between reactive streams and channels](#differences-between-reactive-streams-and-channels)
  * [Basics of iteration](#basics-of-iteration)
  * [Subscription and cancellation](#subscription-and-cancellation)
  * [Backpressure](#backpressure)
  * [Rx Subject vs BroadcastChannel](#rx-subject-vs-broadcastchannel)
* [Operators](#operators)
  * [Range](#range)
  * [Fused filter-map hybrid](#fused-filter-map-hybrid)
  * [Take until](#take-until)
  * [Merge](#merge)
* [Coroutine context](#coroutine-context)
  * [Threads with Rx](#threads-with-rx)
  * [Threads with coroutines](#threads-with-coroutines)
  * [Rx observeOn](#rx-observeon)
  * [Coroutine context to rule them all](#coroutine-context-to-rule-them-all)
  * [Unconfined context](#unconfined-context)

## Differences between reactive streams and channels

Differences between reactive streams and channels ：在 Reactive Streams 函式庫與 Channel 之間的區別

This section outlines key differences between reactive streams and coroutine-based channels. 

這個章節大綱在 Reactive Streams 函式庫與基於協程的 Channel 之間的關鍵區別。

### Basics of iteration

The [Channel][Channel] is somewhat similar concept to the following reactive stream classes:

[Channel][Channel] 與以下 Reactive Streams 類別有些相似：

* Reactive stream [Publisher](https://github.com/reactive-streams/reactive-streams-jvm/blob/master/api/src/main/java/org/reactivestreams/Publisher.java);
* Rx Java 1.x [Observable](http://reactivex.io/RxJava/javadoc/rx/Observable.html);
* Rx Java 2.x [Flowable](http://reactivex.io/RxJava/2.x/javadoc/), which implements `Publisher`.

They all describe an asynchronous stream of elements (aka items in Rx), either infinite or finite, and all of them support backpressure.

它們都描述一個異步的元素串流 (也就是在 Rx 中的項目) ，無限或有限，並且它們都支援背壓。

However, the `Channel` always represents a _hot_ stream of items, using Rx terminology. Elements are being sent into the channel by producer coroutines and are received from it by consumer coroutines. Every [receive][ReceiveChannel.receive] invocation consumes an element from the channel. Let us illustrate it with the following example:

然而， `Channel` 始終代表一個熱門項目的串流，以 Rx 術語來說。透過 producer 協程送來通道中並透過消費者協程從通道中接收。每個 [receive][ReceiveChannel.receive] 調用從通道中消耗一個元素。讓我們使用以下範例解釋它：

```kotlin
fun main() = runBlocking<Unit> {
    
    // produce 協程建造者是一種通道類型也是一個協程，可以送資料到通道
    // create a channel that produces numbers from 1 to 3 with 200ms delays between them
    val source = produce<Int> {
        println("Begin") // mark the beginning of this coroutine in output
        for (x in 1..3) {
            delay(200) // wait for 200ms
            send(x) // send number x to the channel
        }
    }
    // print elements from the source
    println("Elements:")
    
    // 回傳的通道做每次元素的消耗
    source.consumeEach { // consume elements from it
        println(it)
    }
    // print elements from the source AGAIN
    println("Again:")
    
    // consumeEach 會消耗掉一個元素，在上一個 consumeEach 就消耗光了
    source.consumeEach { // consume elements from it
        println(it)
    }
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-01.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-01.kt)獲取完整的代碼

This code produces the following output: 

這些代碼產生以下輸出：

```text
Elements:
Begin
1
2
3
Again:
```

Notice, how "Begin" line was printed just once, because [produce][produce] _coroutine builder_, when it is executed, launches one coroutine to produce a stream of elements. All the produced elements are consumed with [ReceiveChannel.consumeEach][consumeEach] extension function. There is no way to receive the elements from this channel again. The channel is closed when the producer coroutine is over and the attempt to receive from it again cannot receive anything.

注意， 有關 "Begin" 行就被列印一次，因為 [produce][produce] 協程建造者，當它被執行時，發射一個產生元素串流的協程。所有已產生的元素使用 [ReceiveChannel.consumeEach][consumeEach] 擴展函數消耗。無法辦法再次從這個通道接收元素。當生產者協程結束時，通道被關閉，並且再次嘗試從通道接收，直到不可接收任何的東西。

Let us rewrite this code using [publish][publish] coroutine builder from `kotlinx-coroutines-reactive` module instead of [produce][produce] from `kotlinx-coroutines-core` module. The code stays the same, but where `source` used to have [ReceiveChannel][ReceiveChannel] type, it now has reactive streams [Publisher](http://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Publisher.html) type.

讓我們從 `kotlinx-coroutines-reactive` 中使用 [publish][publish] 協程建造者重寫這些代碼，而不是從 `kotlinx-coroutines-core` 模組的 [produce][produce] 。代碼保持不變，但其中  `source` 之前有 [ReceiveChannel][ReceiveChannel] 類型，它現在有 Reactive Streams 函式庫 [Publisher](http://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Publisher.html) 類型。

```kotlin
fun main() = runBlocking<Unit> {
    
    //不同於之前是 ReceiveChannel 類型， publish 如同字面做發怖、發送的事情
    // create a publisher that produces numbers from 1 to 3 with 200ms delays between them
    val source = publish<Int> {
    //           ^^^^^^^  <---  Difference from the previous examples is here
        println("Begin") // mark the beginning of this coroutine in output
        for (x in 1..3) {
            delay(200) // wait for 200ms
            send(x) // send number x to the channel
        }
    }
    // print elements from the source
    println("Elements:")
    
    // 回傳的類型不會因為消耗而消失
    source.consumeEach { // consume elements from it
        println(it)
    }
    // print elements from the source AGAIN
    println("Again:")
    
    // 重覆再印一次
    source.consumeEach { // consume elements from it
        println(it)
    }
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-02.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-02.kt)獲取完整的代碼

Now the output of this code changes to:

現在，這些代碼的輸出改變為：

```text
Elements:
Begin
1
2
3
Again:
Begin
1
2
3
```

This example highlights the key difference between a reactive stream and a channel. A reactive stream is a higher-order functional concept. While the channel _is_ a stream of elements, the reactive stream defines a recipe on how the stream of elements is produced. It becomes the actual stream of elements on _subscription_. Each subscriber may receive the same or a different stream of elements, depending on how the corresponding implementation of `Publisher` works.

這個範例突顯在 Reactive Streams 函式庫和 Channel 關鍵的區別。 Reactive Streams 函式庫是高階函數的觀念。而 Channel 是元素的串流，Reactive Streams 函式庫定義關於如何產生元素串流的配方。它變成訂閱時的實際元素串流。每個訂閱者可以接收相同或不同的元素串流，取決於對應的 `Publisher` 如何實作運行。

The [publish][publish] coroutine builder, that is used in the above example, launches a fresh coroutine on each subscription. Every [Publisher.consumeEach][org.reactivestreams.Publisher.consumeEach] invocation creates a fresh subscription. We have two of them in this code and that is why we see "Begin" printed twice. 

在上面的範例中使用 [publish][publish] 協程建造者，在每個訂閱上發射一個全新的協程。每個 [Publisher.consumeEach][org.reactivestreams.Publisher.consumeEach] 調用創建一個全新的訂閱。我們在這些代碼中有 publish 和 consumeEach 它們兩個，並且這是為何我們看到 `Begin` 被印兩次。

In Rx lingo this is called a _cold_ publisher. Many standard Rx operators produce cold streams, too. We can iterate over them from a coroutine, and every subscription produces the same stream of elements.

在 Rx 行話中，這被稱為冷的發行者。很多標準的 Rx 運算符也產生冷的串流。我們可以從協程遍歷它們，以及每個訂閱產生相同的元素串流。

**WARNING**: It is planned that in the future a second invocation of `consumeEach` method on an channel that is already being consumed is going to fail fast, that is immediately throw an `IllegalStateException`. See [this issue](https://github.com/Kotlin/kotlinx.coroutines/issues/167) for details.

警告：計劃在將來在已消耗通道上第二次 `consumeEach` 方法調用將快速失敗，立即丟出 `IllegalStateException` 。更多細節參閱 [this issue](https://github.com/Kotlin/kotlinx.coroutines/issues/167) 。 

> Note, that we can replicate the same behaviour that we saw with channels by using Rx [publish](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#publish()) operator and [connect](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/flowables/ConnectableFlowable.html#connect()) method with it.
>
> 注意，我們可以透過使用 Rx [publish](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#publish()) 運算符和它的 [connect](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/flowables/ConnectableFlowable.html#connect()) 方法，我們可以複製我們在通道中看到的相同行為。

### Subscription and cancellation

Subscription and cancellation ：訂閱和取消

An example in the previous section uses `source.consumeEach { ... }` snippet to open a subscription and receive all the elements from it. If we need more control on how what to do with the elements that are being received from the channel, we can use [Publisher.openSubscription][org.reactivestreams.Publisher.openSubscription] as shown in the following example:

在前一個章節中範例使用 `source.consumeEach { ... }` 片段去打開訂閱並從它接收所有元素。如果我們需要更多控制如何處理從通道正在接收元素，我們可以使用 [Publisher.openSubscription][org.reactivestreams.Publisher.openSubscription] 如下所示範例：

```kotlin
fun main() = runBlocking<Unit> {
    val source = Flowable.range(1, 5) // a range of five numbers
        .doOnSubscribe { println("OnSubscribe") } // provide some insight
        .doOnComplete { println("OnComplete") }   // ...
        .doFinally { println("Finally") }         // ... into what's going on
    var cnt = 0 
    
    // openSubscription() 回傳 ReceiveChannel 做消耗
    source.openSubscription().consume { // open channel to the source
        for (x in this) { // iterate over the channel to receive elements from it
            println(x)
            if (++cnt >= 3) break // break when 3 elements are printed
        }
        // Note: `consume` cancels the channel when this block of code is complete
    }
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-03.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-03.kt)獲取完整的代碼

It produces the following output:

它產生以下輸出：

```text
OnSubscribe
1
2
3
Finally
```

With an explicit `openSubscription` we should [cancel][ReceiveChannel.cancel] the corresponding subscription to unsubscribe from the source. There is no need to invoke `cancel` explicitly -- under the hood `consume` does that for us. The installed [doFinally](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#doFinally(io.reactivex.functions.Action)) listener prints "Finally" to confirm that the subscription is actually being closed. Note that "OnComplete" is never printed because we did not consume all of the elements.

明確的使用 `openSubscription` ，我們應該 [cancel][ReceiveChannel.cancel] 對應的訂閱從 `source` 取消訂閱。沒要必要明顯的調用 `cancel` -- `consume` 底層為我們做這事。已設置的 [doFinally](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#doFinally(io.reactivex.functions.Action)) 監聽者印出 "Finally" 去確認訂閱實際上正在被關閉。注意，因為我們不會消耗所有元素，所以 "onCompelete" 從不會被印出。

We do not need to use an explicit `cancel` either if iteration is performed over all the items that are emitted by the publisher, because it is being cancelled automatically by `consumeEach`:

如果由 publisher 發射所有的項目執行遍歷，我們不需要明顯的使用 `cancel` ，因為它透過 `consumeEach` 正在被自動的取消：

```kotlin
fun main() = runBlocking<Unit> {
    val source = Flowable.range(1, 5) // a range of five numbers
        .doOnSubscribe { println("OnSubscribe") } // provide some insight
        .doOnComplete { println("OnComplete") }   // ...
        .doFinally { println("Finally") }         // ... into what's going on
    
    // 完全遍歷 source ，包括訂閱 (Subscribe) 、完成 (Complete) 、最終 (Finally) 等週期
    // iterate over the source fully
    source.consumeEach { println(it) }
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-04.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-04.kt)獲取完整的代碼

We get the following output:

我們獲得以下輸出：

```text
OnSubscribe
1
2
3
4
OnComplete
Finally
5
```

Notice, how "OnComplete" and "Finally" are printed before the last element "5". It happens because our `main` function in this example is a coroutine that we start with [runBlocking][runBlocking] coroutine builder. Our main coroutine receives on the channel using `source.consumeEach { ... }` expression. The main coroutine is _suspended_ while it waits for the source to emit an item. When the last item is emitted by `Flowable.range(1, 5)` it _resumes_ the main coroutine, which gets dispatched onto the main thread to print this last element at a later point in time, while the source completes and prints "Finally".

注意，在最後元素 "5" 之前 "OnComplete" 和 "Finally" 如何被印出。它發生是因為在這個範例中我們的 `main` 函數是協程，我們從 [runBlocking][runBlocking] 協程建造者開始。我們的 `main` 協程在通道上使用 `source.consumeEach { ... }` 表達式接收。在 `main` 等待 `source` 發射項目時，同時 `main` 協程被懸掛。當透過 `Flowable.range(1, 5)` 發射最後項目時，它恢復 `main` 協程，協程被分配到主線程，在稍後的時間點印出這最後元素，同時 `source` 完成並印出 "Finally" 。

### Backpressure

Backpressure ：[背壓](http://terms.naer.edu.tw/detail/1325748/) ，類似於「發送」與「接收」的壓力測試

Backpressure is one of the most interesting and complex aspects of reactive streams. Coroutines can _suspend_ and they provide a natural answer to handling backpressure. 

背壓是 Reactive Streams 函式庫最有趣和最複雜的方面之一。協程可以被懸掛，並且他們提供自然的答案去處理背壓。

In Rx Java 2.x a backpressure-capable class is called [Flowable](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html). In the following example we use [rxFlowable][rxFlowable] coroutine builder from `kotlinx-coroutines-rx2` module to define a flowable that sends three integers from 1 to 3. It prints a message to the output before invocation of suspending [send][SendChannel.send] function, so that we can study how it operates.

在 Rx Java 2.x 中，背壓能力類別被稱為 [Flowable](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html) 。在以下範例中，我們從 `kotlinx-coroutines-rx2` 模組中使用 [rxFlowable][rxFlowable] 協程建造者，去定義一個從 1 到 3 發送 3 個整數的 Flowable 。它在懸掛 [send][SendChannel.send] 函數調用之前，印出一個輸出訊息，以便我們研究它如何操作。

The integers are generated in the context of the main thread, but subscription is shifted to another thread using Rx [observeOn](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#observeOn(io.reactivex.Scheduler,%20boolean,%20int)) operator with a buffer of size 1. The subscriber is slow. It takes 500 ms to process each item, which is simulated using `Thread.sleep`.

在主線程的環境中產生 3 個整數，但訂閱被轉移到另一個線程，使用 Rx [observeOn](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#observeOn(io.reactivex.Scheduler,%20boolean,%20int)) 運算符和大小 1 的緩衝。訂閱者很慢，處理每個項目花費 0.5 秒，它使用 `Thread.sleep` 模擬。

```kotlin
fun main() = runBlocking<Unit> { 
    // 在主線程環境中快速產生元素
    // coroutine -- fast producer of elements in the context of the main thread
    val source = rxFlowable {
        for (x in 1..3) {
            send(x) // this is a suspending function
            println("Sent $x") // print after successfully sent item
        }
    }
    // Schedulers.io() 轉移到另一個線程
    // subscribe on another thread with a slow subscriber using Rx
    source
        .observeOn(Schedulers.io(), false, 1) // specify buffer size of 1 item
        .doOnComplete { println("Complete") }
        .subscribe { x ->
            Thread.sleep(500) // 500ms to process each item
            println("Processed $x")
        }
    delay(2000) // suspend the main thread for a few seconds
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-05.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-05.kt)獲取完整的代碼

The output of this code nicely illustrates how backpressure works with coroutines:

這些代碼的輸出很好的解釋背壓如何使用協程運行：

```text
Sent 1
Processed 1
Sent 2
Processed 2
Sent 3
Processed 3
Complete
```

We see here how producer coroutine puts the first element in the buffer and is suspended while trying to send another one. Only after consumer processes the first item, producer sends the second one and resumes, etc.

我們在這裡看到生產者協程如何放置第一個元素到緩衝，並在嘗試發送另一個元素時懸掛。只在消費者處理第一個元素之後，生產者發送第二個元素並恢復，等等。

### Rx Subject vs BroadcastChannel

Rx Subject vs BroadcastChannel ：Rx Subject 與 BroadcastChannel API 的比較

RxJava has a concept of [Subject](https://github.com/ReactiveX/RxJava/wiki/Subject) which is an object that effectively broadcasts elements to all its subscribers. The matching concept in coroutines world is called a [BroadcastChannel][BroadcastChannel]. There is a variety of subjects in Rx with [BehaviorSubject](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/subjects/BehaviorSubject.html) being the one used to manage state:

Rxjava 有 [Subject](https://github.com/ReactiveX/RxJava/wiki/Subject) 的觀念。它是一個有效的廣播元素給它所有的訂閱者物件。在協程世界中匹配的概念稱為 [BroadcastChannel][BroadcastChannel] 。在 Rx 中有各種 Subject 類別，使用 [BehaviorSubject](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/subjects/BehaviorSubject.html) 用於管理狀態：

```kotlin
fun main() {
    val subject = BehaviorSubject.create<String>()
    subject.onNext("one")
    subject.onNext("two") // updates the state of BehaviorSubject, "one" value is lost
    
    // 在訂閱時，它發送最近一次的項目，所以才會印出 "two"
    // now subscribe to this subject and print everything
    subject.subscribe(System.out::println)
    subject.onNext("three")
    subject.onNext("four")
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-06.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-06.kt)獲取完整的代碼

This code prints the current state of the subject on subscription and all its further updates:

在訂閱時這些代碼印出目前 `subject` 的狀態和它所有進一步的更新：

```text
two
three
four
```

You can subscribe to subjects from a coroutine just as with any other reactive stream:

你可以從一個協程就像使用任何其他的 Reactive Streams 函式庫訂閱主題：

```kotlin
fun main() = runBlocking<Unit> {
    val subject = BehaviorSubject.create<String>()
    subject.onNext("one")
    subject.onNext("two")
    // 利用協程來訂閱
    // now launch a coroutine to print everything
    GlobalScope.launch(Dispatchers.Unconfined) { // launch coroutine in unconfined context
        subject.consumeEach { println(it) }
    }
    subject.onNext("three")
    subject.onNext("four")
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-07.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-07.kt)獲取完整的代碼

The result is the same:

結果是一樣：

```text
two
three
four
```

Here we use [Dispatchers.Unconfined][Dispatchers.Unconfined] coroutine context to launch consuming coroutine with the same behaviour as subscription in Rx. It basically means that the launched coroutine is going to be immediately executed in the same thread that is emitting elements. Contexts are covered in more details in a [separate section](#coroutine-context).

在這裡，我們使用 [Dispatchers.Unconfined][Dispatchers.Unconfined] 分配協程環境，去發射進行消耗的協程，與在 Rx 中的 `subscribe` 相同行為。它基本上意味著已發射的協程將在發射元素的相同線程中立即執行。協程環境在[單獨章節](#coroutine-context)中更細節的涵蓋。

The advantage of coroutines is that it is easy to get conflation behavior for single-threaded UI updates. A typical UI application does not need to react to every state change. Only the most recent state is relevant. A sequence of back-to-back updates to the application state needs to get reflected in UI only once, as soon as the UI thread is free. For the following example we are going to simulate this by launching consuming coroutine in the context of the main thread and use [yield][yield] function to simulate a break in the sequence of updates and to release the main thread:

協程的優點是對於單線程的 UI 更新更容易的合併行為。典型的 UI 應用程式不會需要為每個狀態更新做反應。只有最近的狀態是相關的。只要 UI 線程是空閒時，依序更新應用程式狀態只需要在 UI 中反應一次。對於以下範例，我們經由在主線程環境中發射消耗的協程模擬這點，並使用 [yield][yield] 函數在更新順序和釋放主線程中模擬中斷：

```kotlin
fun main() = runBlocking<Unit> {
    val subject = BehaviorSubject.create<String>()
    subject.onNext("one")
    subject.onNext("two")
    // now launch a coroutine to print the most recent update
    launch { // use the context of the main thread for a coroutine
        subject.consumeEach { println(it) }
    }
    subject.onNext("three")
    subject.onNext("four")
    yield() // yield the main thread to the launched coroutine <--- HERE
    subject.onComplete() // now complete subject's sequence to cancel consumer, too    
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-08.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-08.kt)獲取完整的代碼

Now coroutine process (prints) only the most recent update:

 現在協程只處理最近的更新：

```text
four
```

The corresponding behavior in a pure coroutines world is implemented by [ConflatedBroadcastChannel][ConflatedBroadcastChannel] that provides the same logic on top of coroutine channels directly, without going through the bridge to the reactive streams:

在原始協程世界中對應的行為透過 [ConflatedBroadcastChannel][ConflatedBroadcastChannel] 實作，直接在協程通道上提供相同的邏輯，而不是透過橋接到 Reactive Streams 函式庫： 

```kotlin
fun main() = runBlocking<Unit> {
    val broadcast = ConflatedBroadcastChannel<String>()
    broadcast.offer("one")
    broadcast.offer("two")
    // now launch a coroutine to print the most recent update
    launch { // use the context of the main thread for a coroutine
        broadcast.consumeEach { println(it) }
    }
    broadcast.offer("three")
    broadcast.offer("four")
    yield() // yield the main thread to the launched coroutine
    broadcast.close() // now close broadcast channel to cancel consumer, too    
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-09.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-basic-08.kt)獲取完整的代碼

It produces the same output as the previous example based on `BehaviorSubject`:

它產生與前一個範例基於 `BehaviorSubject` API 相同的輸出：

```text
four
```

Another implementation of [BroadcastChannel][BroadcastChannel] is `ArrayBroadcastChannel` with an array-based buffer of a specified `capacity`. It can be created with `BroadcastChannel(capacity)`. It delivers every event to every subscriber since the moment the corresponding subscription is open. It corresponds to [PublishSubject](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/subjects/PublishSubject.html) in Rx. The capacity of the buffer in the constructor of `ArrayBroadcastChannel` controls the numbers of elements that can be sent before the sender is suspended waiting for receiver to receive those elements.

[BroadcastChannel][BroadcastChannel] 的另一個實作是指定 `capacity` 基於陣列緩衝的 `ArrayBroadcastChannel` 。它可以使用 `BroadcastChannel(capacity)` 創建。從對應訂閱開放那一刻起。它傳每個事件給每個訂閱者。它對應於 Rx 中的 [PublishSubject](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/subjects/PublishSubject.html) 。在 `ArrayBroadcastChannel` 的建構元當中緩衝的容量控制可以被發送的數個元素，在發送者懸掛的等待接收者去接收那些元素之前

## Operators

Operators ：運算符

Full-featured reactive stream libraries, like Rx, come with [a very large set of operators](http://reactivex.io/documentation/operators.html) to create, transform, combine and otherwise process the corresponding streams. Creating your own operators with support for back-pressure is [notoriously](http://akarnokd.blogspot.ru/2015/05/pitfalls-of-operator-implementations.html) [difficult](https://github.com/ReactiveX/RxJava/wiki/Writing-operators-for-2.0).

功能齊全的 Reactive Streams 函式庫，像 Rx，伴隨[非常大組的運算符](http://reactivex.io/documentation/operators.html)去創建、轉換、結合以及處理對應的串流。創建你擁有的運算符並對背壓的支援是[眾所周知](http://akarnokd.blogspot.ru/2015/05/pitfalls-of-operator-implementations.html)的[困難](https://github.com/ReactiveX/RxJava/wiki/Writing-operators-for-2.0)。

Coroutines and channels are designed to provide an opposite experience. There are no built-in operators, but processing streams of elements is extremely simple and back-pressure is supported automatically without you having to explicitly think about it.

協程和通道被設計提供相反的經驗。沒有內建的運算符，但處理元素串流是非常簡單，並且自動的支援背壓，你不必明確的考慮它。

This section shows coroutine-based implementation of several reactive stream operators.

這個章節展示幾個 Reactive Streams 函式庫運算符基於協程的實作。

### Range

Range ：使用 pulish 協程實作發送範圍值

Let's roll out own implementation of [range](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#range(int,%20int)) operator for reactive streams `Publisher` interface. The asynchronous clean-slate implementation of this operator for reactive streams is explained in [this blog post](http://akarnokd.blogspot.ru/2017/03/java-9-flow-api-asynchronous-integer.html). It takes a lot of code. Here is the corresponding code with coroutines:

讓我們為 Reactive Streams 函式庫的 `Publisher` 介面推出擁有的 [range](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#range(int,%20int)) 運算符實作。在這個[部落格](http://akarnokd.blogspot.ru/2017/03/java-9-flow-api-asynchronous-integer.html)中解釋對於 Reactive Streams 函式庫這個運算符異步重新實作。它帶入很多的代碼，這裡是使用協程對應的代碼：

```kotlin
fun CoroutineScope.range(context: CoroutineContext, start: Int, count: Int) = publish<Int>(context) {
    for (x in start until start + count) send(x)
}
```

In this code `CoroutineScope` and `context` are used instead of an `Executor` and all the backpressure aspects are taken care of by the coroutines machinery. Note, that this implementation depends only on the small reactive streams library that defines `Publisher` interface and its friends.

在這些代碼 `CoroutineScope` 和 `context` 用於代替 `Executor` ，以及所有背壓方面透過協程機制處理。注意，這些實作只取決於小型 Reactive Streams 函式庫，函式庫定義 `Publisher` 介面和它的朋友。

It is straightforward to use from a coroutine:

從協程簡單的使用：

```kotlin
fun main() = runBlocking<Unit> {
    // Range inherits parent job from runBlocking, but overrides dispatcher with Dispatchers.Default
    range(Dispatchers.Default, 1, 5).consumeEach { println(it) }
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-operators-01.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-operators-01.kt)獲取完整的代碼

The result of this code is quite expected:

這些代碼的結果相當可預期的：

```text
1
2
3
4
5
```

### Fused filter-map hybrid

Fused filter-map hybrid ：融合 filter 和 map 方法混用

Reactive operators like [filter](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#filter(io.reactivex.functions.Predicate)) and [map](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#map(io.reactivex.functions.Function)) are trivial to implement with coroutines. For a bit of challenge and showcase, let us combine them into the single `fusedFilterMap` operator: 

反應式運算符像是使用協程實作 [filter](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#filter(io.reactivex.functions.Predicate)) 和 [map](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#map(io.reactivex.functions.Function)) 很簡單。對於一些挑戰和展示，讓我們結合它們到單一 `fusedFilterMap` 運算符：

```kotlin
fun <T, R> Publisher<T>.fusedFilterMap(
    context: CoroutineContext,   // the context to execute this coroutine in
    predicate: (T) -> Boolean,   // the filter predicate
    mapper: (T) -> R             // the mapper function
) = GlobalScope.publish<R>(context) {
    consumeEach {                // consume the source stream 
        if (predicate(it))       // filter part
            send(mapper(it))     // map part
    }        
}
```

Using `range` from the previous example we can test our `fusedFilterMap` by filtering for even numbers and mapping them to strings:

從前面範例使用 `range` ，我們可以透過對於偶數的過濾並映射它們到字串來測試我們的 `fusedFilterMap` 。

```kotlin
fun main() = runBlocking<Unit> {
   range(1, 5)
       .fusedFilterMap(coroutineContext, { it % 2 == 0}, { "$it is even" })
       .consumeEach { println(it) } // print all the resulting strings
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-operators-02.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-operators-02.kt)獲取完整的代碼

It is not hard to see, that the result is going to be:

不難看出，結果將是：

```text
2 is even
4 is even
```

### Take until

Take until ：直到另一個 publish 發送就停止目前的發送。

Let's implement our own version of [takeUntil](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#takeUntil(org.reactivestreams.Publisher)) operator. It is quite a [tricky one](http://akarnokd.blogspot.ru/2015/05/pitfalls-of-operator-implementations.html) to implement, because of the need to track and manage subscription to two streams. We need to relay all the elements from the source stream until the other stream either completes or emits anything. However, we have [select][select] expression to rescue us in coroutines implementation:

讓我們實作我們自己的 [takeUntil](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#takeUntil(org.reactivestreams.Publisher)) 運算符版本。實作它相當[棘手](http://akarnokd.blogspot.ru/2015/05/pitfalls-of-operator-implementations.html)，因為需要追蹤和管理兩個串流的訂閱。我們需要從來源串流轉換所有元素到其他的串流完成或發射任何內容。然而，我們有 [select][select] 表達式在協程環境中去拯救我們：

```kotlin
fun <T, U> Publisher<T>.takeUntil(context: CoroutineContext, other: Publisher<U>) = GlobalScope.publish<T>(context) {
    // 轉換為通道做消耗處理
    this@takeUntil.openSubscription().consume { // explicitly open channel to Publisher<T>
        val current = this
        
        // 轉換為通道做消耗處理
        other.openSubscription().consume { // explicitly open channel to Publisher<U>
            val other = this
            
            // select 表達式是做同時 (競爭選擇) ，回調 onReceive 函數
            whileSelect {
                // 第二個 publish 在同時收到就回傳 false 停止 while loop
                other.onReceive { false }          // bail out on any received element from `other`
                
                // 第一個 publish 在同時收到就發送做印出，並回傳 true 繼承執行
                current.onReceive { send(it); true }  // resend element from this channel and continue
            }
        }
    }
}
```

This code is using [whileSelect][whileSelect] as a nicer shortcut to `while(select{...}) {}` loop and Kotlin's [use](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html) expression to close the channels on exit, which unsubscribes from the corresponding publishers. 

這些代碼正在使用 [whileSelect][whileSelect] 作為 `while(select{...}) {}` 循環，和 Kotlin 的 [use](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html) 表達式在離開時去關閉通道更好的捷徑，從對應的 publisher 取消訂閱。

The following hand-written combination of [range](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#range(int,%20int)) with [interval](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#interval(long,%20java.util.concurrent.TimeUnit,%20io.reactivex.Scheduler)) is used for testing. It is coded using a `publish` coroutine builder (its pure-Rx implementation is shown in later sections):

以下手寫 [range](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#range(int,%20int)) 和 [interval](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#interval(long,%20java.util.concurrent.TimeUnit,%20io.reactivex.Scheduler)) 的結合用於測試。使用一個 `publish` 協程編碼 (它是原生 Rx 實作在後面的部分中顯示) ：

```kotlin
fun CoroutineScope.rangeWithInterval(time: Long, start: Int, count: Int) = publish<Int> {
    for (x in start until start + count) { 
        delay(time) // wait before sending each number
        send(x)
    }
}
```

The following code shows how `takeUntil` works: 

以下代碼顯示 `takeUntil` 如何運行：

```kotlin
fun main() = runBlocking<Unit> {
    // 跑 1 ~ 11 間隔 0.2 秒
    val slowNums = rangeWithInterval(200, 1, 10)         // numbers with 200ms interval
    
    // 跑 1 ~ 11 間隔 0.5 秒
    val stop = rangeWithInterval(500, 1, 10)             // the first one after 500ms
    slowNums.takeUntil(coroutineContext, stop).consumeEach { println(it) } // let's test it
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-operators-03.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-rx2/test/guide/example-reactive-operators-03.kt)獲取完整的代碼

Producing 

產生

**因為第一個 publish 每 0.2 秒發送一個數字，所以發送 2 是 0.4 秒，而第二個 publish 每 0.5 發送一個數字，剛好停止**

```text
1
2
```

### Merge

There are always at least two ways for processing multiple streams of data with coroutines. One way involving
[select] was shown in the previous example. The other way is just to launch multiple coroutines. Let
us implement 
[merge](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#merge(org.reactivestreams.Publisher))
operator using the later approach:

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import org.reactivestreams.*
import kotlin.coroutines.*
-->

```kotlin
fun <T> Publisher<Publisher<T>>.merge(context: CoroutineContext) = GlobalScope.publish<T>(context) {
  consumeEach { pub ->                 // for each publisher received on the source channel
      launch {  // launch a child coroutine
          pub.consumeEach { send(it) } // resend all element from this publisher
      }
  }
}
```

Notice, the use of 
[coroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/coroutine-context.html)
in the invocation of [launch] coroutine builder. It is used to refer
to the context of the enclosing `publish` coroutine. This way, all the coroutines that are
being launched here are [children](../docs/coroutines-guide.md#children-of-a-coroutine) of the `publish`
coroutine and will get cancelled when the `publish` coroutine is cancelled or is otherwise completed. 
Moreover, since parent coroutine waits until all children are complete, this implementation fully
merges all the received streams.

For a test, let us start with `rangeWithInterval` function from the previous example and write a 
producer that sends its results twice with some delay:

<!--- INCLUDE

fun CoroutineScope.rangeWithInterval(time: Long, start: Int, count: Int) = publish<Int> {
​    for (x in start until start + count) { 
​        delay(time) // wait before sending each number
​        send(x)
​    }
}
-->

```kotlin
fun CoroutineScope.testPub() = publish<Publisher<Int>> {
    send(rangeWithInterval(250, 1, 4)) // number 1 at 250ms, 2 at 500ms, 3 at 750ms, 4 at 1000ms 
    delay(100) // wait for 100 ms
    send(rangeWithInterval(500, 11, 3)) // number 11 at 600ms, 12 at 1100ms, 13 at 1600ms
    delay(1100) // wait for 1.1s - done in 1.2 sec after start
}
```

The test code is to use `merge` on `testPub` and to display the results:

```kotlin
fun main() = runBlocking<Unit> {
    testPub().merge(coroutineContext).consumeEach { println(it) } // print the whole stream
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-04.kt)

And the results should be: 

```text
1
2
11
3
4
12
13
```

<!--- TEST -->

## Coroutine context

All the example operators that are shown in the previous section have an explicit
[CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) 
parameter. In Rx world it roughly corresponds to 
a [Scheduler](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Scheduler.html).

### Threads with Rx

The following example shows the basics of threading context management with Rx.
Here `rangeWithIntervalRx` is an implementation of `rangeWithInterval` function using Rx 
`zip`, `range`, and `interval` operators.

<!--- INCLUDE
import io.reactivex.*
import io.reactivex.functions.BiFunction
import io.reactivex.schedulers.Schedulers
import java.util.concurrent.TimeUnit
-->

```kotlin
fun rangeWithIntervalRx(scheduler: Scheduler, time: Long, start: Int, count: Int): Flowable<Int> = 
    Flowable.zip(
        Flowable.range(start, count),
        Flowable.interval(time, TimeUnit.MILLISECONDS, scheduler),
        BiFunction { x, _ -> x })

fun main() {
    rangeWithIntervalRx(Schedulers.computation(), 100, 1, 3)
        .subscribe { println("$it on thread ${Thread.currentThread().name}") }
    Thread.sleep(1000)
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-context-01.kt)

We are explicitly passing the 
[Schedulers.computation()](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/schedulers/Schedulers.html#computation()) 
scheduler to our `rangeWithIntervalRx` operator and
it is going to be executed in Rx computation thread pool. The output is going to be similar to the following one:

```text
1 on thread RxComputationThreadPool-1
2 on thread RxComputationThreadPool-1
3 on thread RxComputationThreadPool-1
```

<!--- TEST FLEXIBLE_THREAD -->

### Threads with coroutines

In the world of coroutines `Schedulers.computation()` roughly corresponds to [Dispatchers.Default], 
so the previous example is similar to the following one:

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlin.coroutines.CoroutineContext
-->

```kotlin
fun rangeWithInterval(context: CoroutineContext, time: Long, start: Int, count: Int) = GlobalScope.publish<Int>(context) {
    for (x in start until start + count) { 
        delay(time) // wait before sending each number
        send(x)
    }
}

fun main() {
    Flowable.fromPublisher(rangeWithInterval(Dispatchers.Default, 100, 1, 3))
        .subscribe { println("$it on thread ${Thread.currentThread().name}") }
    Thread.sleep(1000)
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-context-02.kt)

The produced output is going to be similar to:

```text
1 on thread ForkJoinPool.commonPool-worker-1
2 on thread ForkJoinPool.commonPool-worker-1
3 on thread ForkJoinPool.commonPool-worker-1
```

<!--- TEST LINES_START -->

Here we've used Rx 
[subscribe](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#subscribe(io.reactivex.functions.Consumer))
operator that does not have its own scheduler and operates on the same thread that the publisher -- on a default
shared pool of threads in this example.

### Rx observeOn 

In Rx you use special operators to modify the threading context for operations in the chain. You
can find some [good guides](http://tomstechnicalblog.blogspot.ru/2016/02/rxjava-understanding-observeon-and.html)
about them, if you are not familiar. 

For example, there is
[observeOn](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#observeOn(io.reactivex.Scheduler)) 
operator. Let us modify the previous example to observe using `Schedulers.computation()`:   

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import io.reactivex.schedulers.Schedulers
import kotlin.coroutines.CoroutineContext
-->

```kotlin
fun rangeWithInterval(context: CoroutineContext, time: Long, start: Int, count: Int) = GlobalScope.publish<Int>(context) {
    for (x in start until start + count) { 
        delay(time) // wait before sending each number
        send(x)
    }
}

fun main() {
    Flowable.fromPublisher(rangeWithInterval(Dispatchers.Default, 100, 1, 3))
        .observeOn(Schedulers.computation())                           // <-- THIS LINE IS ADDED
        .subscribe { println("$it on thread ${Thread.currentThread().name}") }
    Thread.sleep(1000)
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-context-03.kt)

Here is the difference in output, notice "RxComputationThreadPool":

```text
1 on thread RxComputationThreadPool-1
2 on thread RxComputationThreadPool-1
3 on thread RxComputationThreadPool-1
```

<!--- TEST FLEXIBLE_THREAD -->

### Coroutine context to rule them all

A coroutine is always working in some context. For example, let us start a coroutine
in the main thread with [runBlocking] and iterate over the result of the Rx version of `rangeWithIntervalRx` operator, 
instead of using Rx `subscribe` operator:

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import io.reactivex.functions.BiFunction
import io.reactivex.schedulers.Schedulers
import java.util.concurrent.TimeUnit
-->

```kotlin
fun rangeWithIntervalRx(scheduler: Scheduler, time: Long, start: Int, count: Int): Flowable<Int> =
    Flowable.zip(
        Flowable.range(start, count),
        Flowable.interval(time, TimeUnit.MILLISECONDS, scheduler),
        BiFunction { x, _ -> x })

fun main() = runBlocking<Unit> {
    rangeWithIntervalRx(Schedulers.computation(), 100, 1, 3)
        .consumeEach { println("$it on thread ${Thread.currentThread().name}") }
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-context-04.kt)

The resulting messages are going to be printed in the main thread:

```text
1 on thread main
2 on thread main
3 on thread main
```

<!--- TEST LINES_START -->

### Unconfined context

Most Rx operators do not have any specific thread (scheduler) associated with them and are working 
in whatever thread that they happen to be invoked in. We've seen it on the example of `subscribe` operator 
in the [threads with Rx](#threads-with-rx) section.

In the world of coroutines, [Dispatchers.Unconfined] context serves a similar role. Let us modify our previous example,
but instead of iterating over the source `Flowable` from the `runBlocking` coroutine that is confined 
to the main thread, we launch a new coroutine in `Dispatchers.Unconfined` context, while the main coroutine
simply waits its completion using [Job.join]:

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import io.reactivex.functions.BiFunction
import io.reactivex.schedulers.Schedulers
import java.util.concurrent.TimeUnit
-->

```kotlin
fun rangeWithIntervalRx(scheduler: Scheduler, time: Long, start: Int, count: Int): Flowable<Int> =
    Flowable.zip(
        Flowable.range(start, count),
        Flowable.interval(time, TimeUnit.MILLISECONDS, scheduler),
        BiFunction { x, _ -> x })

fun main() = runBlocking<Unit> {
    val job = launch(Dispatchers.Unconfined) { // launch new coroutine in Unconfined context (without its own thread pool)
        rangeWithIntervalRx(Schedulers.computation(), 100, 1, 3)
            .consumeEach { println("$it on thread ${Thread.currentThread().name}") }
    }
    job.join() // wait for our coroutine to complete
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-context-05.kt)

Now, the output shows that the code of the coroutine is executing in the Rx computation thread pool, just
like our initial example using Rx `subscribe` operator.

```text
1 on thread RxComputationThreadPool-1
2 on thread RxComputationThreadPool-1
3 on thread RxComputationThreadPool-1
```

<!--- TEST LINES_START -->

Note, that [Dispatchers.Unconfined] context shall be used with care. It may improve the overall performance on certain tests,
due to the increased stack-locality of operations and less scheduling overhead, but it also produces deeper stacks 
and makes it harder to reason about asynchronicity of the code that is using it. 

If a coroutine sends an element to a channel, then the thread that invoked the 
[send][SendChannel.send] may start executing the code of a coroutine with [Dispatchers.Unconfined] dispatcher.
The original producer coroutine that invoked `send`  is paused until the unconfined consumer coroutine hits its next
suspension point. This is very similar to a lock-step single-threaded `onNext` execution in Rx world in the absense
of thread-shifting operators. It is a normal default for Rx, because operators are usually doing very small chunks
of work and you have to combine many operators for a complex processing. However, this is unusual with coroutines, 
where you can have an arbitrary complex processing in a coroutine. Usually, you only need to chain stream-processing
coroutines for complex pipelines with fan-in and fan-out between multiple worker coroutines.

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[Dispatchers.Unconfined]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html
[yield]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/yield.html
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
<!--- INDEX kotlinx.coroutines.channels -->
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/index.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
[consumeEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/consume-each.html
[ReceiveChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/index.html
[ReceiveChannel.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/cancel.html
[SendChannel.send]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/send.html
[BroadcastChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-broadcast-channel/index.html
[ConflatedBroadcastChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-conflated-broadcast-channel/index.html
<!--- INDEX kotlinx.coroutines.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/select.html
[whileSelect]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/while-select.html
<!--- MODULE kotlinx-coroutines-reactive -->
<!--- INDEX kotlinx.coroutines.reactive -->
[publish]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-reactive/kotlinx.coroutines.reactive/kotlinx.coroutines.-coroutine-scope/publish.html
[org.reactivestreams.Publisher.consumeEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-reactive/kotlinx.coroutines.reactive/org.reactivestreams.-publisher/consume-each.html
[org.reactivestreams.Publisher.openSubscription]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-reactive/kotlinx.coroutines.reactive/org.reactivestreams.-publisher/open-subscription.html
<!--- MODULE kotlinx-coroutines-rx2 -->
<!--- INDEX kotlinx.coroutines.rx2 -->
[rxFlowable]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-rx2/kotlinx.coroutines.rx2/kotlinx.coroutines.-coroutine-scope/rx-flowable.html
<!--- END -->


