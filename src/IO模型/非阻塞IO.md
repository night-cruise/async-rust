# 非阻塞 IO

在 Linux 中，我们可以把一个 `socket` 设置为非阻塞（nonblocking）。对于非阻塞 IO 来说，读操作的流程如下所示：

![Nonblocking IO Model](../imgs/Nonblocking-IO.png)

当用户进程发起 `recvfrom` 系统调用后，如果数据没有准备好，`recvfrom` 系统调用会立即返回 `EWOULDBLOCK` 错误。用户进程可以通过一个死循环不断发起 `recvfrom` 系统调用，一旦数据准备好了，就进入 IO 的第二个阶段：把数据从内核缓冲区拷贝到用户用进程的缓冲区，当拷贝完成后，`recvfrom` 系统调用正常返回。

因此，**对于 Nonblocking IO 来说，用户进程需要不断轮询内核数据准备好了没有，并且用户进程在 IO 的第二个阶段仍然会被 `recvfrom` 系统调用阻塞**。
