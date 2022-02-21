# Introduction

本书主要介绍在 Rust 中 `async/await` 与 `async runtime` 的原理和工作机制，并不涉实际的异步代码编写。全书的内容主要分为以下几个章节：

* **异步编程**：介绍 Rust 异步编程的基础概念，以及在 Rust 中应用的异步模型。

  

* **async/await**：介绍Rust为支持异步编程而提供的语言层面的支持，包括 `async/await` 语法和它们的工作原理。

  

* **IO 模型**：介绍几种主要的 IO 模型，包括同步阻塞 IO、同步非阻塞 IO、IO 多路复用、异步 IO，其中 IO 多路复用是后文介绍 `Epoll` 的基础。



* **Epoll**：介绍 `Epoll` 的工作原理并提供一个简单的 `Epoll` server 的实现例子。`Epoll` 是 Linux 中 IO 多路复用的一种实现，为后文介绍异步运行时的基础。

  

* **异步运行时**：通过实现一个简单的异步运行时来介绍 `Reactor`、`Waker`、`Executor` 的基本概念。



## References

* <https://rust-lang.github.io/async-book/03_async_await/01_chapter.html>
* <https://www.zhihu.com/question/389262477/answer/1566255353>
* <https://doc.rust-lang.org/std/keyword.async.html>
* <https://doc.rust-lang.org/std/keyword.await.html>
* <https://doc.rust-lang.org/std/future/trait.Future.html>
* <https://cfsamson.github.io/books-futures-explained/1_futures_in_rust.html#futures-in-rust>
* <https://doc.rust-lang.org/std/task/struct.Context.html>
* <https://rust-lang.github.io/async-book/02_execution/02_future.html>
* <https://github.com/ZhangHanDong/inviting-rust>
* <https://doc.rust-lang.org/std/ops/trait.Generator.html>
* <https://doc.rust-lang.org/std/ops/enum.GeneratorState.html>
* <https://github.com/rust-lang/rust/blob/master/library/core/src/future/mod.rs>
* <https://ipotato.me/article/70>
* <https://cfsamson.github.io/books-futures-explained/4_generators_async_await.html>
* <https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html>
* <https://rust-lang.github.io/async-book/01_getting_started/02_why_async.html>
* <https://cfsamson.github.io/books-futures-explained/5_pin.html>
* <https://rust-lang.github.io/async-book/04_pinning/01_chapter.html>
* <https://folyd.com/blog/rust-pin-unpin/>
* <https://doc.rust-lang.org/core/pin/struct.Pin.html>
