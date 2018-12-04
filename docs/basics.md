## Table of contents

Table of contents ：內容表

* [Coroutine basics](#coroutine-basics)
  * [Your first coroutine](#your-first-coroutine)
  * [Bridging blocking and non-blocking worlds](#bridging-blocking-and-non-blocking-worlds)
  * [Waiting for a job](#waiting-for-a-job)
  * [Structured concurrency](#structured-concurrency)
  * [Scope builder](#scope-builder)
  * [Extract function refactoring](#extract-function-refactoring)
  * [Coroutines ARE light-weight](#coroutines-are-light-weight)
  * [Global coroutines are like daemon threads](#global-coroutines-are-like-daemon-threads)

**以下執行協程可利用  ${Thread.currentThread().name} 查看協程的資訊**

## Coroutine basics

Coroutine basics ：協程基礎觀念

This section covers basic coroutine concepts.

本章節涵蓋基本協程觀念。

### Your first coroutine

Your first coroutine ：你的第一個協程實作

Run the following code:

運行以下代碼：

```kotlin
import kotlinx.coroutines.*

fun main() {
    
    // 建立新的 coroutine , launch 必須在某個 CoroutineScope 底下
    GlobalScope.launch { // launch new coroutine in background and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello,") // main thread continues while coroutine is delayed
    
    // 主線程休息 2 秒
    Thread.sleep(2000L) // block main thread for 2 seconds to keep JVM alive
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-01.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-01.kt)獲取完整的代碼

You will see the following result:

你將看到以下的結果：

```text
Hello,
World!
```

Essentially, coroutines are light-weight threads. They are launched with [launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) _coroutine builder_ in a context of some [CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html). Here we are launching a new coroutine in the [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html), meaning that the lifetime of the new coroutine is limited only by the lifetime of the whole application. 

本質上， coroutines  是輕量的線程。它們在一些 [CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html) 內容中使用 [launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) 協程建造者發射異步。這裡我們 [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html) 中發射新的協程，意味者透過整個應用程序的存活時間只限制新的協程存活時間。

You can achieve the same result replacing `GlobalScope.launch { ... }` with `thread { ... }` and `delay(...)` with `Thread.sleep(...)`. Try it.

你可以使用 `thread { ... }` 取代  `GlobalScope.launch { ... }` 和 `Thread.sleep(...)` 取代 `delay(...)` 獲取相同的結果。嘗試它。

If you start by replacing `GlobalScope.launch` by `thread`, the compiler produces the following error:

如果你開始透過 `thread` 取代 `GlobalScope.launch` ，編譯器生成以下錯誤：

```
Error: Kotlin: Suspend functions are only allowed to be called from a coroutine or another suspend function
```

That is because [delay](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html) is a special _suspending function_ that does not block a thread, but _suspends_ coroutine and it can be only used from a coroutine.

這是因為 [delay](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html) 是特別的懸掛函數不會阻塞線程，但是**懸掛**協程只能從協程中使用。 

### Bridging blocking and non-blocking worlds

Bridging blocking and non-blocking worlds ：橋接阻塞和非阻塞的世界

The first example mixes _non-blocking_ `delay(...)` and _blocking_ `Thread.sleep(...)` in the same code. It is easy to lose track of which one is blocking and which one is not. Let's be explicit about blocking using [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) coroutine builder:

第一個範例在相同代碼混合**非阻塞** `delay(...)` 和**阻塞** `Thread.sleep(...)` 。很容易忘記那一個是阻塞以及那一個是非阻塞。讓我們明確說明阻塞使用 [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) 協程建造者：

```kotlin
import kotlinx.coroutines.*

fun main() { 
    
    // 建立新的 coroutine , launch 必須在某個 CoroutineScope 底下
    GlobalScope.launch { // launch new coroutine in background and continue
        delay(1000L)
        println("World!")
    }
    println("Hello,") // main thread continues here immediately
    
    // 阻塞目前的線程，運行協程在之中暫停 2 秒，結束後才繼續 main 線程
    runBlocking {     // but this expression blocks the main thread
        delay(2000L)  // ... while we delay for 2 seconds to keep JVM alive
    } 
    
    // 等阻塞函數完成才可以執行...
}

//ans:
//Hello,
//World!
```

> You can get full code [here](.https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-02b.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-02b.kt)獲取完整的代碼

The result is the same, but this code uses only non-blocking [delay](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html). The main thread, that invokes `runBlocking`, _blocks_ until the coroutine inside `runBlocking` completes. 

結果是相同的，但這代碼只使用非阻塞 [delay](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html) 。主線程，調用 `runBlocking` ，主線程阻塞直到在 `runBlocking` 內協程完成。

**`runBlocking`  是阻塞當下的線程，由於是在 main() 中呼叫 runblocking {...} 所以阻塞主線程**

This example can be also rewritten in a more idiomatic way, using `runBlocking` to wrap the execution of the main function:

這個範例也可以在更多的慣用語言方式中覆寫，使用 `runBlocking` 包裝主線程函數的執行：

```kotlin
import kotlinx.coroutines.*

// 函數的單一表達式，分配一個 runBlocking 函數，回傳類型為 Unit
// 阻塞 main 線程，產生新的協程
fun main() = runBlocking<Unit> { // start main coroutine
    GlobalScope.launch { // launch new coroutine in background and continue
        delay(1000L)
        println("World!")
    }
    println("Hello,") // main coroutine continues here immediately
    delay(2000L)      // delaying for 2 seconds to keep JVM alive
}

//ans:
//Hello,
//World!
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-02b.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-02b.kt)獲取完整的代碼

Here `runBlocking<Unit> { ... }` works as an adaptor that is used to start the top-level main coroutine. We explicitly specify its `Unit` return type, because a well-formed `main` function in Kotlin has to return `Unit`.

這裡的 `runBlocking<Unit> { ... }` 作為調配器，被用作開始最高層級的主協程。我們明確指定它的 `Unit` 回傳類型，因為 Kotlin 中格式良好的 `main` 函數必須回傳 `Unit` 。

This is also a way to write unit-tests for suspending functions:

這也是一種為懸掛函數寫單元測試的方式：

```kotlin
class MyTest {
    @Test // 在單元測試環境
    fun testMySuspendingFunction() = runBlocking<Unit> {
        // here we can use suspending functions using any assertion style that we like
    }
}
```

### Waiting for a job

Waiting for a job ：等待工作

Delaying for a time while another coroutine is working is not a good approach. Let's explicitly wait (in a non-blocking way) until the background [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html) that we have launched is complete:

當另一個協程在工作時，延遲一段時間，並不是一個好方法。讓我們明確等待 (在非阻塞的方式中) 直到我們發射的背景程序 [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html) 完成：

```kotlin
import kotlinx.coroutines.*

// 指定協程，阻塞 main 線程，產生新的協程
fun main() = runBlocking {
//sampleStart
    
    // 建立新的 coroutine , launch 必須在某個 CoroutineScope 底下
    val job = GlobalScope.launch { // launch new coroutine and keep a reference to its Job
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    
    // join 只能在協程中調用，等待 job 完成，再回來 runBlocking 的協程
    job.join() // wait until child coroutine completes
//sampleEnd    
}

//ans:
//Hello,
//World!
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-03.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-03.kt))獲取完整的代碼

Now the result is still the same, but the code of the main coroutine is not tied to the duration of
the background job in any way. Much better.

現在的結果還是一樣，但主協程的代碼不以任何方式綁定 (聯繫) 背景程序的持續時間。

### Structured concurrency

Structured concurrency ：結構化並發

There is still something to be desired for practical usage of coroutines. When we use `GlobalScope.launch` we create a top-level coroutine. Even though it is light-weight, it still consumes some memory resources while it runs. If we forget to keep a reference to the newly launched coroutine it still runs. What if the code in the coroutine hangs (for example, we erroneously delay for too long), what if we launched too many coroutines and ran out of memory? Having to manually keep a reference to all the launched coroutines and [join](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html) them is error-prone. 

還是有某些事情被期望用於實踐協程的用法。當我們使用 `GlobalScope.launch` 時，我們創建一個最高層級的協程。即使它是輕量，當它運行時，它還是有消耗某些記憶體資源。如果我們忘記保留新的已發射協程引用，已發射的協程還是在運行。如果在協程中的代碼掛起 (例如：我太長時間錯誤的延遲) ，如果我們發射太多協程且跑出記憶體不足怎麼辦？必須手動保留已發射協程的引用並 [join](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html) 它們容易出錯。

There is a better solution. We can use structured concurrency in our code. Instead of launching coroutines in the [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html), just like we usually do with threads (threads are always global), we can launch coroutines in the specific scope of the operation we are performing. 

有更好的解決方式。我們可以在我們的代碼中使用結構式並發。在 [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html) 中代替發射協程，就像我們通常使用線程一樣 (線程總是全域的) ，我們可以在我們正在執行的指定範圍發射協程。

In our example, we have `main` function that is turned into a coroutine using [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) coroutine builder. Every coroutine builder, including `runBlocking`, adds an instance of [CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html) to the scope of its code block. We can launch coroutines in this scope without having to `join` them explicitly, because an outer coroutine (`runBlocking` in our example) does not complete until all the coroutines launched in its scope complete. Thus, we can make our example simpler:

在我們的範例中，我們使用 [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) 協程建造者將 `main` 函數轉換到協程。每個協程建造者，包括  `runBlocking` ， 添加 [CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html) 的實例到 `runBlocking` 的代碼區域的圍範。我們可以在 `runBlocking` 範圍內發射協程，不必明確的 `join` 它們，因為外部的協程不會完成執行 (在我們的範例中 `runBlocking`) ，直到在 `runBlocking` 範圍內所有已發射的協程完成。

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking { // this: CoroutineScope
    launch { // launch new coroutine in the scope of runBlocking
        delay(1000L)
        println("World!")
    }
    println("Hello,") // 執行完持續等待 launch
}

//ans:
//Hello,
//World!
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-03s.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-03s.kt)獲取完整的代碼

### Scope builder

Scope builder ：範圍建造者

In addition to the coroutine scope provided by different builders, it is possible to declare your own scope using [coroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html) builder. It creates new coroutine scope and does not complete until all launched children complete. The main difference between [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) and [coroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html) is that the latter does not block the current thread while waiting for all children to complete.

除了由不同的建造者提供協程範圍，還可以使用 [coroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html) 建造者宣告你擁有的協程範圍。它會創建新的協程範圍，並且直到已發射的子協程完成才會完成整個範圍。 [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) 和 [coroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html) 之間主要的不同是後者在等待所有子協程完成時不會阻塞當前的線程。

```kotlin
import kotlinx.coroutines.*

// 阻塞主線程，跑協程
fun main() = runBlocking { // this: CoroutineScope
    
    // 發射新的協程
    launch { 
        delay(200L)
        println("Task from runBlocking")
    }
    
    // 建立另一個協程的範圍
    coroutineScope { // Creates a new coroutine scope
        
        // 在另一個協程範圍發射一個協程
        launch {
            delay(500L) 
            println("Task from nested launch")
        }
    
        delay(100L)
        println("Task from coroutine scope") // This line will be printed before nested launch
    }
    
    // 等待 coroutineScope 的 launch 跑完才執行
    println("Coroutine scope is over") // This line is not printed until nested launch completes
}

//ans:
//Task from coroutine scope
//Task from runBlocking
//Task from nested launch
//Coroutine scope is over
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-04.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-04.kt)獲取完整的代碼

### Extract function refactoring

Extract function refactoring ：提取函數重構

Let's extract the block of code inside `launch { ... }` into a separate function. When you perform "Extract function" refactoring on this code you get a new function with `suspend` modifier. That is your first _suspending function_. Suspending functions can be used inside coroutines just like regular functions, but their additional feature is that they can, in turn, use other suspending functions, like `delay` in this example, to _suspend_ execution of a coroutine.

讓我們提取 `launch { ... }` 內的代碼區域到單獨函數。當你在這份代碼執行 "提取函數" 重構時，你使用 `suspend` 修飾符取得新的函數。這是你第一個**懸掛函數**。懸掛函數可以用在協程內就像常規函數，但它們的額外功能是它們可以反過來使用其他的懸掛函數，像是在範例中的 `delay` ，懸掛 (暫停) 協程的執行。

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch { doWorld() }
    println("Hello,")
}

// this is your first suspending function
suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-05.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-05.kt)獲取完整的代碼

But what if the extracted function contains a coroutine builder which is invoked on the current scope? In this case `suspend` modifier on the extracted function is not enough. Making `doWorld` extension method on `CoroutineScope` is one of the solutions, but it may not always be applicable as it does not make API clearer. Idiomatic solution is to have either explicit `CoroutineScope` as a field in a class containing target function or implicit when outer class implements `CoroutineScope`. As a last resort, [CoroutineScope(coroutineContext)](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope.html) can be used, but such approach is structurally unsafe because you no longer have control on the scope this method is executed. Only private API can use this builder.

但是，如果提取的函數包含目前範圍上調用的協程建造者，該怎麼辦？在這種情況下 ，在提取函數上的 `suspend` 修飾符是不夠的。在 `CoroutineScope` 上製作 `doWorld` 擴展函數是解決方式之一，但它可能不總是適用，因為它不會使 API 變的清晰。常用慣語解決方式是有明確的方式是 `CoroutineScope` 在類別內作為欄位包含目標函數，或是隱式的方式在外部類別實作 `CoroutineScope` 。作為最後的手段，可以使用  [CoroutineScope(coroutineContext)](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope.html) ，但是這種方式結構上是不安全的，因為你不能在範圍中再控制這個方法被執行，只有私有 API 可以使用這樣的建造者。

### Coroutines ARE light-weight

Coroutines ARE light-weight ：協程是輕量

Run the following code:

執行以下代碼：

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // 跑十萬次 coroutine
    repeat(100_000) { // launch a lot of coroutines
        launch {
            delay(1000L)
            print(".")
        }
    }
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-06.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-06.kt)獲取完整的代碼

It launches 100K coroutines and, after a second, each coroutine prints a dot. Now, try that with threads. What would happen? (Most likely your code will produce some sort of out-of-memory error)

它發射十萬次協程，並且，在一秒之後，每個協程印出一個點。現在，嘗試使用線程。可能會發生？ (最有可能你的代碼將產生某種記憶體不足的錯誤) 。

### Global coroutines are like daemon threads

Global coroutines are like daemon threads ：全域協程像是常駐線程 (背景持續存在)

The following code launches a long-running coroutine in [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html) that prints "I'm sleeping" twice a second and then returns from the main function after some delay:

以下代碼在 [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html) 中發射一個長期運行的協程，它印出 "I'm sleeping" 一秒兩次，並接著在一段時間延遲後從主函數回傳。

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    GlobalScope.launch {
        
        // 重覆 1000 次印出，但主函數延遲 1.3 秒，而在函數之中運行一次 0.5 秒，所以只跑兩次
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    
    // 主函數延遲 1.3 秒
    delay(1300L) // just quit after delay
//sampleEnd    
}

//ans:
//I'm sleeping 0 ...
//I'm sleeping 1 ...
//I'm sleeping 2 ...
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-07.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-basic-07.kt)獲取完整的代碼

You can run and see that it prints three lines and terminates:

你可以運行及看到它印出三行並終止：

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
```

Active coroutines that were launched in [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html) do not keep the process alive. They are like daemon threads.

在 [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html) 中發射存活的協程，不會使進程保持活躍狀態。它們像是常駐的線程。