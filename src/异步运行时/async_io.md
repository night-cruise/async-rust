# async_io

在 `async_io` 模块中，我们将会创建 `Leaf Future`，异步化网络IO的监听和读写操作。



## Ipv4Addr 抽象

```rust,noplayground
pub struct Ipv4Addr(libc::in_addr);

impl Ipv4Addr {
    pub fn new(a: u8, b: u8, c: u8, d: u8) -> Self {
        Ipv4Addr(libc::in_addr {
            s_addr: ((u32::from(a) << 24)
                | (u32::from(b) << 16)
                | (u32::from(c) << 8)
                | u32::from(d))
            .to_be(),
        })
    }
}
```

`Ipv4Addr` 就是 `IPv4` 地址，`new` 方法负责创建一个 `Ipv4Addr` 类型。



## TcpListener 抽象

```rust,noplayground
pub struct TcpListener(RawFd);

impl TcpListener {
    // NOTE: bind() may be block. So this should be an async function in reality.
    pub fn bind(addr: Ipv4Addr, port: u16) -> io::Result<TcpListener> {
        let backlog = 128;
        let sock = syscall!(socket(
            libc::PF_INET,
            libc::SOCK_STREAM | libc::SOCK_CLOEXEC,
            0
        ))?;
        let opt: i32 = 1;
        syscall!(setsockopt(
            sock,
            libc::SOL_SOCKET,
            libc::SO_REUSEADDR,
            &opt as *const _ as *const libc::c_void,
            std::mem::size_of_val(&opt) as u32
        ))?;

        let sin: libc::sockaddr_in = libc::sockaddr_in {
            sin_family: libc::AF_INET as libc::sa_family_t,
            sin_port: port.to_be(),
            sin_addr: addr.0,
            ..unsafe { mem::zeroed() }
        };
        let addr_p: *const libc::sockaddr = &sin as *const _ as *const _;
        let len = mem::size_of_val(&sin) as libc::socklen_t;

        syscall!(bind(sock, addr_p, len))?;
        syscall!(listen(sock, backlog))?;

        info!("(TcpListener) listen: {}", sock);
        let listener = TcpListener(sock);
        listener.nonblocking()?;
        Ok(listener)
    }

    pub(crate) fn accept(&self) -> io::Result<TcpStream> {
        let mut sin_client: libc::sockaddr_in = unsafe { mem::zeroed() };
        let addr_p: *mut libc::sockaddr = &mut sin_client as *mut _ as *mut _;
        let mut len: libc::socklen_t = unsafe { mem::zeroed() };
        let len_p: *mut _ = &mut len as *mut _;
        let sock_client = syscall!(accept(self.0, addr_p, len_p))?;
        info!("(TcpStream)  accept: {}", sock_client);
        Ok(TcpStream(sock_client))
    }

    pub fn incoming(&self) -> Incoming<'_> {
        Incoming(self)
    }

    fn nonblocking(&self) -> io::Result<()> {
        let flag = syscall!(fcntl(self.0, libc::F_GETFL, 0))?;
        syscall!(fcntl(self.0, libc::F_SETFL, flag | libc::O_NONBLOCK))?;
        Ok(())
    }
}

impl Drop for TcpListener {
    fn drop(&mut self) {
        info!("(TcpListener) close : {}", self.0);
        syscall!(close(self.0)).ok();
    }
}


pub struct Incoming<'a>(&'a TcpListener);

impl<'a> Incoming<'a> {
    pub fn next(&self) -> AcceptFuture<'a> {
        AcceptFuture(self.0)
    }
}
```

`bind` 方法负责绑定传入的的 `IpV4` 地址和端口号，创建一个 `TcpListener` 实例，需要注意的是要把 `TcpListener` 设置为非阻塞：`listener.nonblocking()`，这样在调用 `accept` 方法接收客户端连接时才不会阻塞。

`accept` 方法负责接收到来的客户端连接，然后创建 `TcpStream`，如果没有连接到来就返回一个 `io error`。

`nonblocking` 方法调用 `libc::fcntl` 函数把 `TcpListener` 设置为非阻塞。

`incoming` 方法把 `TcpListener` 的引用包在 `Incoming` 中，然后返回一个 `Incoming` 的实例。

`Incoming` 表示 `TcpListener` 接收连接的流式处理，每当我们想要接收一个新的连接时，就调用 `next` 方法返回一个 `AcceptFuture`（后面会讲这个）。



## TcpStream 抽象

```rust,noplayground
pub struct TcpStream(RawFd);

impl TcpStream {
    fn nonblocking(&self) -> io::Result<()> {
        let flag = syscall!(fcntl(self.0, libc::F_GETFL, 0))?;
        syscall!(fcntl(self.0, libc::F_SETFL, flag | libc::O_NONBLOCK))?;
        Ok(())
    }

    pub fn read<'a>(&'a self, buf: &'a mut [u8]) -> ReadFuture<'a> {
        ReadFuture(self, buf)
    }

    pub fn write<'a>(&'a self, buf: &'a [u8]) -> WriteFuture<'a> {
        WriteFuture(self, buf)
    }

    pub fn raw_fd(&self) -> RawFd {
        self.0
    }
}

impl Drop for TcpStream {
    fn drop(&mut self) {
        info!("(TcpStream)  close : {}", self.0);
        syscall!(close(self.0)).ok();
    }
}
```

`nonblocking` 方法调用 `libc::fcntl` 函数把 `TcpStream` 设置为非阻塞。

`read/write` 方法分别返回 `RreadFture/WriteFuture`，和上面的 `AcceptFuture` 一样，我们将会在下面讲解这些 `Future` 的定义和作用。



## Leaf Future 抽象

```rust,noplayground
pub struct AcceptFuture<'a>(&'a TcpListener);
pub struct ReadFuture<'a>(&'a TcpStream, &'a mut [u8]);
pub struct WriteFuture<'a>(&'a TcpStream, &'a [u8]);

impl<'a> Future for AcceptFuture<'a> {
    type Output = Option<io::Result<TcpStream>>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        match self.0.accept() {
            Ok(stream) => {
                stream.nonblocking()?;
                Poll::Ready(Some(Ok(stream)))
            }
            Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
                REACTOR.add_event((self.0).0, EpollEventType::In, cx.waker().clone())?;
                Poll::Pending
            }
            Err(e) => Poll::Ready(Some(Err(e))),
        }
    }
}

impl<'a> Future for ReadFuture<'a> {
    type Output = io::Result<usize>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let res = syscall!(read(
            (self.0).0,
            self.1.as_mut_ptr() as *mut libc::c_void,
            self.1.len()
        ));
        match res {
            Ok(n) => Poll::Ready(Ok(n as usize)),
            Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
                REACTOR.add_event((self.0).0, EpollEventType::In, cx.waker().clone())?;
                Poll::Pending
            }
            Err(e) => Poll::Ready(Err(e)),
        }
    }
}

impl<'a> Future for WriteFuture<'a> {
    type Output = io::Result<usize>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let res = syscall!(write(
            (self.0).0,
            self.1.as_ptr() as *mut libc::c_void,
            self.1.len()
        ));
        match res {
            Ok(n) => Poll::Ready(Ok(n as usize)),
            Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
                REACTOR.add_event((self.0).0, EpollEventType::Out, cx.waker().clone())?;
                Poll::Pending
            }
            Err(e) => Poll::Ready(Err(e)),
        }
    }
}
```

在同步的处理方式中，监听 `TcpListener` 和读写 `TcpStream` 是阻塞式的，即会阻塞线程直到相应的 IO 事件发生；

而在异步的处理方式中，监听 `TcpListener` 和读写 `TcpStream`  不会阻塞掉线程，而是会返回一个对应的 `Leaf Future`：

* `Incoming` 的 `next` 方法会返回一个 `AcceptFuture`。
* `TcpStream` 的 `read/write` 方法分别返回 `ReadFuture/WriteFuture`。

为 `AcceptFuture/ReadFuture/WriteFuture` 实现 `Future trait`，这样它们就有了 `poll` 方法。上述三个 `Future` 的 `poll` 方法的执行流程时类似的，因此下面我们只讲解 `ReadFutrue` 的执行流程。

当调用 `ReadFutrue.await` 时，会调用 `ReadFutrue` 的 `poll` 方法，在 `poll` 方法内部：

* 调用 `libc::read` 函数从 `TcpStream` 中读取数据，返回 `res`。

* 匹配 `res` 的值：

  * 如果是 `Ok(n)`，则读取到了数据，此时直接返回 `Poll::Ready(OK(n))`，调用方继续执行 `ReadFutrue.await` 下面的代码。

    

  * 如果是 `Err(e)`，并且 `e.kind() == io::ErrorKind::WouldBlock`，则说明 `TcpStream` 中没有数据可读，这时就调用 `REACTOR` 的 `add_event` 方法注册文件描述符（关联读事件）和 `waker`。最后返回 `Poll::Pending`，调用方接收到 `Poll::Pending` 后就会调用 `yield` 表达式挂起当前的执行流（`Task`）。

    

  * 如果是其他的 `Err(e)`，则说明读取数据发生了其他错误，此时返回 `Poll::Ready(Err(e))` 表示读取失败，调用方继续执行 `ReadFutrue.await` 下面的代码。

当注册到 `REACTOR` 中的事件就绪时，`REACTOR` 就会使用注册的 `waker` 唤醒挂起的 `Task`，继续调用 `ReadFutrue` 的 `poll` 方法，重复上述的执行流程。
