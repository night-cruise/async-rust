# IO 多路复用

IO 多路复用是指通过一种机制实现在单个线程中可以监视多个文件描述符（例如 `socket` 描述符），当文件描述读/写就绪时，用户进程就可以获取就绪的文件句柄。`select`、`poll`、`epoll` 都是 IO 多路复用的一种实现。

以 `select` 为例，读操作的流程如下所示：

![IO Multiplexing Model](../imgs/IO-Multiplexing-Model.png)

当用户进程发起 `select` 系统调用后，用户进程被阻塞，而内核会监控 `select` 负责的所有文件描述符，当任意一个文件描述符的数据准备好时，`select` 会返回就绪的文件描述符。此时，用户进程就可以对就绪的文件描述符发起 `recvfrom` 系统调用，开始 IO 的第二个阶段：将数据从内核缓冲区拷贝到用户进程的缓冲区，当拷贝结束后 `recvfrom` 调用正常返回。 

因此，**对于 IO 多路复用来说，用户进程在 IO 的两个阶段都被阻塞了：在 IO 的第一个阶段被 `select` 系统调用阻塞，在 IO 的第二个阶段被 `recvfrom` 系统调用阻塞**。
