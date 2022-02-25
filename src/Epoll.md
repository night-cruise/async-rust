# Epoll

`Epoll` 本质上是一种 IO 事件通知机制，是前文所述的在 Linux 中 IO 多路复用的一种实现。在本章中，我们将会简略介绍 `Epoll` 的原理，并使用 `Epoll` 实现一个简单的 `echo server`。

在最后一章《异步运行时》中，我们也会使用 `Epoll` 作为基础来实现一个 `Reactor`（`Reactor` 的概念会在后面介绍）。
