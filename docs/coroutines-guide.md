Kotlin, as a language, provides only minimal low-level APIs in its standard library to enable various other libraries to utilize coroutines. Unlike many other languages with similar capabilities, `async` and `await` are not keywords in Kotlin and are not even part of its standard library. Moreover, Kotlin's concept of _suspending function_ provides a safer and less error-prone abstraction for asynchronous operations than futures and promises.  

Kotlin 作為一種語言，在標準函式庫中只提供底層 API ，以便使各種其他的函式庫利用協程。不像很多其他的語用使用類似的功能， `async` 和 `await` 在 Kotlin 中不是關鍵字，甚至不會是它的標準函式庫的一部份。然而， Kotlin 的**懸掛函數**觀念為異步操作提供了比 `future` 與 `promises` API 更安全、不容易出錯的抽象觀念。

`kotlinx.coroutines` is a rich library for coroutines developed by JetBrains. It contains a number of high-level coroutine-enabled primitives that this guide covers, including `launch`, `async` and others. 

`kotlinx.coroutines` 是豐富的函式庫為 JetBrains 開發 coroutines 。 這份指導涵蓋許多高階協程 API 啟用的原生語言，包括 `launch` 、 `async` 和其他的。

This is a guide on core features of `kotlinx.coroutines` with a series of examples, divided up into different topics.

在 `kotlinx.coroutines` 的核心功能這是一份指導，使用一系列的範例，分開到不同的主題介紹。

In order to use coroutines as well as follow the examples in this guide, you need to add a dependency on `kotlinx-coroutines-core` module as explained [in the project README ](https://github.com/kotlin/kotlinx.coroutines/blob/master/README.md#using-in-your-projects).

為了使用 coroutines 並在這份指導中遵循的範例，你需要添加 `kotlinx-coroutines-core` 模組的依賴，如[專案的 README](https://github.com/kotlin/kotlinx.coroutines/blob/master/README.md#using-in-your-projects) 所述。

## Table of contents

* [Coroutine basics](basics.md)
* [Cancellation and timeouts](cancellation-and-timeouts.md)
* [Composing suspending functions](composing-suspending-functions.md)
* [Coroutine context and dispatchers](coroutine-context-and-dispatchers.md)
* [Exception handling and supervision](exception-handling.md)
* [Channels (experimental)](channels.md)
* [Shared mutable state and concurrency](shared-mutable-state-and-concurrency.md)
* [Select expression (experimental)](select-expression.md)

## Additional references

* [Guide to UI programming with coroutines](../ui/coroutines-guide-ui.md)
* [Guide to reactive streams with coroutines](../reactive/coroutines-guide-reactive.md)
* [Coroutines design document (KEEP)](https://github.com/Kotlin/kotlin-coroutines/blob/master/kotlin-coroutines-informal.md)
* [Full kotlinx.coroutines API reference](http://kotlin.github.io/kotlinx.coroutines)
