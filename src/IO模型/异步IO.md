# 异步 IO

对于异步 IO 来说，读操作的流程如下所示：

![Asynchronous IO Model](../imgs/Asynchronous-IO.png)

当用户进程发起异步框架 `AIO` 提供的 `aio_read` 系统调用后，这个系统调用会马上返回。内核会准备好数据然后把数据从内核缓冲区拷贝到用户进程缓冲区，当 IO 的两个阶段都完成后，内核会发送一个信号通知用户进程 `read` 操作完成了。

因此，**对于异步 IO 来说，用户进程在 IO 的两个阶段都不会被阻塞**。

> 题外话：实际上 `AIO` 并不理想，因此之前的 Linux 异步 IO 在实际中应用的并不多。直到2019年在 Linux 5.1 中首次引入了 `io_uring` 高性能异步 IO 框架，让 Linux 异步 IO 变得有可能真正流行起来。`tokio-rs` 提供的 [tokio-uring](https://github.com/tokio-rs/tokio-uring) 就是一个使用 `io_uring` 实现的异步运行时。



## 总结

`POSIX` 对同步 IO 和异步 IO 的定义如下：

* 同步 IO 操作会导致发起请求的进程被阻塞，直到 IO 操作完成。
* 异步 IO 操作导致发起请求的进程被阻塞。

根据 `PISIX` 的定义，可以把 IO 模型分为以下两类：

```
                                 +-+-+-+-+-+ 阻 塞 I/O (BIO)
                                 |
                                 +-+-+-+-+-+ 非 阻 塞 I/O (NIO)
                                 |
              +----------+ 同 步 I/O
              |                  |
              |                  +-+-+-+-+-+ I/O 多 路 复 用
              |                  |
              |                  +-+-+-+-+-+ 信 号 驱 动 I/O
I/O 模 型  +---+
              |
              |
              |                  +-+-+-+-+-+ Linux (AIO)
              |                  |         (io_uring)
              +----------+ 异 步 I/O
                                 |
                                 +-+-+-+-+-+ windows (IOCP)
```

最后，各个 IO 模型的比较如下所示：

![Comparison of the IO Models](imgs/Comparison-IO-Model.png)
