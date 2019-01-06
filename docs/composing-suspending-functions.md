**Table of contents**

* [Composing suspending functions](#composing-suspending-functions)
  * [Sequential by default](#sequential-by-default)
  * [Concurrent using async](#concurrent-using-async)
  * [Lazily started async](#lazily-started-async)
  * [Async-style functions](#async-style-functions)
  * [Structured concurrency with async](#structured-concurrency-with-async)

## Composing suspending functions

Composing suspending functions ：組合懸掛函數

This section covers various approaches to composition of suspending functions.

本章節涵蓋各種懸掛函數的組合方法。

### Sequential by default

Sequential by default ：根據預設依序

Assume that we have two suspending functions defined elsewhere that do something useful like some kind of remote service call or computation. We just pretend they are useful, but actually each one just delays for a second for the purpose of this example:

假設我們有兩個懸掛函數定義在別處，負責做某些有用的事像是某種遠端服務調用或計算。我們只是假裝它們有用的，但實際上每個函數只是為了這個例子目的延遲一秒：

```kotlin
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

What do we do if need to invoke them _sequentially_ -- first `doSomethingUsefulOne` _and then_  `doSomethingUsefulTwo` and compute the sum of their results? In practice we do this if we use the results of the first function to make a decision on whether we need to invoke the second one or to decide on how to invoke it.

如果需要依序調用它們，我們該做什麼 -- 首先 `doSomethingUsefulOne` 而接著 `doSomethingUsefulTwo` 並計算它們結果的總合？在實踐上，如果我們使用第一個函數的結果來決定，我們是否需要調用第二個函數或決定如何調用它，我們會這樣做。

We use a normal sequential invocation, because the code in the coroutine, just like in the regular code, is _sequential_ by default. The following example demonstrates it by measuring the total time it takes to execute both suspending functions:

我們使用正常順序的調用，因為在協程中的代碼，與常規的代碼一樣，是預設順序。以下範例透過測量 `measureTimeMillis` 執行兩個懸掛函數花費的總時間來展示它：

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    
    // 測量函數，計算花費的時間
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    
    // 花費時間的結果
    println("Completed in $time ms")
//sampleEnd    
}

// 懸掛函數執行等待、回傳結果
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

// 懸掛函數執行等待、回傳結果
suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-compose-01.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-compose-01.kt)獲取完整的代碼

It produces something like this:

它產生這樣的東西：

```text
The answer is 42
Completed in 2017 ms
```

### Concurrent using async

Concurrent using async ：並發 (同時) 使用 async 函數

What if there are no dependencies between invocation of `doSomethingUsefulOne` and `doSomethingUsefulTwo` and we want to get the answer faster, by doing both _concurrently_? This is where [async][async] comes to help. 

如果在 `doSomethingUsefulOne` 和 `doSomethingUsefulTwo` 調用之間沒有依賴，並且我們想要透過並發 (同時) 執行兩者，更快的取得答案，該怎麼做？這是 [async][async] 來幫忙的地方。

Conceptually, [async][async] is just like [launch][launch]. It starts a separate coroutine which is a light-weight thread that works concurrently with all the other coroutines. The difference is that `launch` returns a [Job][Job] and does not carry any resulting value, while `async` returns a [Deferred][Deferred] -- a light-weight non-blocking future that represents a promise to provide a result later. You can use `.await()` on a deferred value to get its eventual result, but `Deferred` is also a `Job`, so you can cancel it if needed.

觀念上，[async][async] 就像 [launch][launch] 。它啟動一個單獨協程，協程是輕量的線程，與所有其他的協程同時工作。不同的是， `launch` 回傳 [Job][Job]，並不會附帶任何結果值，而 `async` 回傳 [Deferred][Deferred] -- 一個輕量不阻塞的 Future (Java API Future) ，表示稍後提供結果的承諾。你可以在回傳 [Deferred][Deferred] 值上使用 `.await()` 來獲取它最終的結果，但 `Deferred` 也是一個 `Job` ，所以如果需要你可以取消它。


```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    
    // 測量函數
    val time = measureTimeMillis {
        
        // Deferred 類型的回傳值
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        
        // await() 取值
        println("The answer is ${one.await() + two.await()}")
    }
    
    // 費時的結果
    println("Completed in $time ms")
//sampleEnd    
}

// 懸掛函數執行等待、回傳結果
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

// 懸掛函數執行等待、回傳結果
suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-compose-02.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-compose-02.kt)獲取完整的代碼

It produces something like this:

它產生這樣的東西：

```text
The answer is 42
Completed in 1017 ms
```

This is twice as fast, because we have concurrent execution of two coroutines. Note, that concurrency with coroutines is always explicit.

這是兩倍的速度，因為我們並發 (同時) 執行兩個協程。注意，並發協程必須明確。

### Lazily started async

Lazily started async ：惰性啟動 async 函數，「惰性」代表只有當調用時 (start) 才會執行

There is a laziness option to [async][async] using an optional `start` parameter with a value of [CoroutineStart.LAZY][CoroutineStart.LAZY]. It starts coroutine only when its result is needed by some [await][Deferred.await] or if a [start][Job.start] function is invoked. Run the following example:

這是一個懶惰的選項，[async][async] 使用可選的 `start` 參數帶 [CoroutineStart.LAZY][CoroutineStart.LAZY] 的值。只當某個 [await][Deferred.await] 需要執行它的結果，或者如果 [start][Job.start] 函數被調用，它才執行協程，執行以下範例：

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        
        // 參數 CoroutineStart.LAZY ，當調用時才執行 {...}
        val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        // some computation
        
        // 啟動方式一，呼叫 start()，不會阻塞線程，會在背景執行，回傳 boolean，知道有沒有啟動
        // one.start() // start the first one
        // two.start() // start the second one
        println("${one.start()}") // true
        println("${one.start()}") // false
        println("${two.start()}") // true
        println("${two.start()}") // false
        
        // 啟動方式二，呼叫 await()，阻塞當下的線程，取得結果值
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

// 懸掛函數執行等待、回傳結果
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

// 懸掛函數執行等待、回傳結果
suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-compose-03.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-compose-03.kt)獲取完整的代碼

It produces something like this:

它產生這樣的東西：

```text
The answer is 42
Completed in 1017 ms
```

So, here the two coroutines are defined but not executed as in the previous example, but the control is given to the programmer on when exactly to start the execution by calling [start][Job.start]. We first start `one`, then start `two`, and then await for the individual coroutines to finish. 

所以，在前面的例子中，這裡定義兩個協程而不執行，當透過調用 [start][Job.start] 函數精確啟動執行，將控制權給予程序員。我們首先啟動 `one` ，接著啟動 `two` ，接著期待各個協程完成。

Note, that if we have called [await][Deferred.await] in `println` and omitted [start][Job.start] on individual coroutines, then we would have got the sequential behaviour as [await][Deferred.await] starts the coroutine execution and waits for the execution to finish, which is not the intended use-case for laziness. The use-case for `async(start = CoroutineStart.LAZY)` is a replacement for the standard `lazy` function in cases when computation of the value involves suspending functions.

注意：如果我們在 `println` 中已經調用 [await][Deferred.await] 並且在各個協程上省略 [start][Job.start] ，接著我們會得到依序 (有序) 的行為，因為 [await][Deferred.await] 先啟動協程執行並且等到執行到結束的行為，這不是懶性預期的使用案例。當計算值涉及懸掛函數時，在這種情況時 `async(start = CoroutineStart.LAZY)` 的使用案例是代替標準的 `lazy` 函數。

### Async-style functions

Async-style functions ： Async 風格的函數

We can define async-style functions that invoke `doSomethingUsefulOne` and `doSomethingUsefulTwo` _asynchronously_ using [async][async] coroutine builder with an explicit [GlobalScope][GlobalScope] reference. We name such functions with "Async" suffix to highlight the fact that they only start asynchronous computation and one needs to use the resulting deferred value to get the result.

我們可以明確使用 [GlobalScope][GlobalScope] 參照中的 [async][async] 協程建造者來定義 async 風格的函數，該函數異步調用 `doSomethingUsefulOne` 和 `doSomethingUsefulTwo` 。我們使用 "Async" 為後綴命名這樣的函數，突顯它們只啟動於異步計算以及使用 Deferred 類型值去取得結果。

```kotlin
// The result type of somethingUsefulOneAsync is Deferred<Int>
fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

// The result type of somethingUsefulTwoAsync is Deferred<Int>
fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}
```

Note, that these `xxxAsync` functions are **not** _suspending_ functions. They can be used from anywhere. However, their use always implies asynchronous (here meaning _concurrent_) execution of their action with the invoking code.

注意：這些 `xxxAsync` 函數不是懸掛函數。他們可以在任何地方使用。然而，他們的用途總是意味著它們的動作是異步執行調用代碼。

The following example shows their use outside of coroutine: 

以下範例展示它們在協程之外的用法：

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

//sampleStart
// note, that we don't have `runBlocking` to the right of `main` in this example
fun main() {
    
    // 測量執行時間
    val time = measureTimeMillis {
        
        // 調用異步
        // we can initiate async actions outside of a coroutine
        val one = somethingUsefulOneAsync()
        val two = somethingUsefulTwoAsync()
        // but waiting for a result must involve either suspending or blocking.
        // here we use `runBlocking { ... }` to block the main thread while waiting for the result
        
        // 印出結果
        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}
//sampleEnd

// 使用 Global 範圍的 async ，不是懸掛函數可能會有問題
fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

// 使用 Global 範圍的 async ，不是懸掛函數可能會有問題
fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}

// 懸掛函數執行等待、回傳結果
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

// 懸掛函數執行等待、回傳結果
suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}

//ans:
//The answer is 42
//Completed in 1108 ms
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-compose-04.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-compose-04.kt)獲取完整的代碼

> This programming style with async functions is provided here only for illustration, because it is a popular style in other programming languages. Using this style with Kotlin coroutines is **strongly discouraged** for the reasons that are explained below.
>
> 這裡提供使用異步函數的這種編碼風格只為了解釋，在其他的編程語言中它是一種流行風格。使用這種風格的 Kotlin 協程是強烈不鼓勵的，以下解譯原因：

Consider what happens if between `val one = somethingUsefulOneAsync()` line and `one.await()` expression there is some logic error in the code and the program throws an exception and the operation that was being performed by the program aborts. Normally, a global error-handler could catch this exception, log and report the error for developers, but the program could otherwise continue doing other operations. But here we have `somethingUsefulOneAsync` still running in background, despite the fact, that operation that had initiated it aborts. This problem does not happen with structured concurrency, as shown in the section below.

考慮，如果在 `val one = somethingUsefulOneAsync()` 一行和 `one.await()` 表達式之間發生什麼事，在代碼中有些邏輯錯誤並程式丟出異常和程式中止已執行的操作。通常，全域的錯誤處理器可以捕獲這樣的異常、日誌、錯誤報告給開發者，但程式可以另外繼續做其他的操作。但是在這裡我們有一些 `somethingUsefulOneAsync` 還是在背景程序執行，儘管實際上，已初始化它的操作中止。使用結構並發不會發生這樣的問題，如下面章節所示：

### Structured concurrency with async 

Structured concurrency with async ：使用 async 函數結構並發

Let us take [Concurrent using async](#concurrent-using-async) example and extract a function that concurrently performs `doSomethingUsefulOne` and `doSomethingUsefulTwo` and returns the sum of their results. Because [async][async] coroutines builder is defined as extension on [CoroutineScope][-CoroutineScope] we need to have it in the scope and that is what [coroutineScope][coroutineScope] function provides:

讓我們以[並發使用 async 函數](#concurrent-using-async)為例並提取函數並發執行 `doSomethingUsefulOne` 和 `doSomethingUsefulTwo` 並回傳它們結果的總合。因為 [async][async] 協程建造器被定義在 [CoroutineScope][-CoroutineScope] 的擴展函數，我們需要在協程範圍中放置 async 函數，這就是 [coroutineScope][coroutineScope] 函數提供的功能：

**coroutineScope 提供了如果在這個範圍發生錯誤協程的處理機制**

```kotlin
suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
     one.await() + two.await()
}
```

This way, if something goes wrong inside the code of `concurrentSum` function and it throws an exception, all the coroutines that were launched in its scope are cancelled.

這種方式，如果在 `concurrentSum` 函數的代碼內發生錯誤並且它拋出異常，取消在 `coroutineScope` 的協程範圍內所有已發射的協程。

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    
    // 執行測量函數，並執行懸掛函數 concurrentSum()
    val time = measureTimeMillis {
        println("The answer is ${concurrentSum()}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

// 懸掛函數，建立協程範圍，執行另外兩個協程
suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
     one.await() + two.await()
}

// 懸掛函數執行等待、回傳結果
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

// 懸掛函數執行等待、回傳結果
suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-compose-05.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-compose-05.kt)獲取完整的代碼

We still have concurrent execution of both operations as evident from the output of the above main function: 

從上面 main 函數的輸出 ，我們仍然同時執行這兩個操作：

```text
The answer is 42
Completed in 1017 ms
```

Cancellation is always propagated through coroutines hierarchy:

取消總是透過協程階層的傳播：

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    try {
        
        // 在這裡補獲異常
        failedConcurrentSum()
    } catch(e: ArithmeticException) {
        println("Computation failed with ArithmeticException")
    }
}

// 建立一個協程的範圍
suspend fun failedConcurrentSum(): Int = coroutineScope {
    
    // 第一個協程，負責持續等待另一個協程發生異常
    val one = async<Int> { 
        try {
            delay(Long.MAX_VALUE) // Emulates very long computation
            42
        } finally {
            println("First child was cancelled")
        }
    }
    
    // 第二個協程，負責丟出異常
    val two = async<Int> { 
        println("Second child throws an exception")
        throw ArithmeticException()
    }
        one.await() + two.await()
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-compose-06.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-compose-06.kt)獲取完整的代碼

Note, how both first `async` and awaiting parent are cancelled on the one child failure:

注意：如何在一個子協程失敗上，取消第一個 `async` 和等待父親：

```text
Second child throws an exception
First child was cancelled
Computation failed with ArithmeticException
```

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Deferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html
[CoroutineStart.LAZY]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-start/-l-a-z-y.html
[Deferred.await]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/await.html
[Job.start]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/start.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[-CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
<!--- END -->
