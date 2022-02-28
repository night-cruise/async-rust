# IO 模型

## IO 访问

对于一次 IO 访问（例如 `read` 操作），通常有两个不同的阶段：

1. 等待数据准备 (Waiting for the data to be ready)
2. 将数据从内核拷贝到进程中 (Copying the data from the kernel to the process)

例如在一个 `socket` 上读取数据，首先需要等待数据到达网络，当数据到达时将数据拷贝到内核缓冲区中，再将数据从内核缓冲区中拷贝到用户进程的缓冲区中。

正是由于 IO 访问经历的两个阶段，Linux 系统产生了下面五种 IO 模型：

* 阻塞 IO（blocking IO）
* 非阻塞 IO（nonblocking IO）
* IO 多路复用（IO multiplexing）
* 信号驱动 IO（signal driven IO）
* 异步 IO（asynchronous IO）

信号驱动 IO 在实际中用的不多，因此本章中不会介绍信号驱动 IO，感兴趣的话可以阅读 [Unix Network Programming](https://www.masterraghu.com/subjects/np/introduction/unix_network_programming_v1.3/ch06lev1sec2.html) 中关于关于信号驱动 IO 的部分。



## IO 模型与 Future

在介绍 `Future trait` 的那一章中我们提到：如果一个 `Future` 没有计算完成，例如想要等待一个 IO 事件发生，那么通常会注册 `waker` 到一个“事件通知系统”中，当这个 IO 事件就绪时，“事件通知系统”就会通过 `waker` 唤醒之前的 `Future` 继续执行。

那么“事件通知系统”要怎么知道 `Future` 想要等待的 IO 事件什么时候就绪呢？这与 IO 模型有关，因此在本章中我们将会介绍几种不同的 IO 模型以及它们的特点。
