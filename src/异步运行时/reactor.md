# reactor

## Reactor

```rust,noplayground
pub(crate) struct Reactor {
    pub epoll: Epoll,
    pub wakers: Mutex<BTreeMap<RawFd, Waker>>,
}
```

字段 `epoll` 存储创建的 `Epoll` 实例，`wakers` 存储等待的IO事件的文件描述符和对应的 `waker`。

我们稍后将会创建 `Epoll` 的静态变量，为了内部可变性，就把 `BTreeMap<RawFd, Waker>` 包在 `Mutex` 中。



## 添加事件

```rust,noplayground
impl Reactor {
    pub(crate) fn add_event(&self, fd: RawFd, op: EpollEventType, waker: Waker) -> io::Result<()> {
        info!("(Reactor) add event: {}", fd);
        self.epoll.add_event(fd, op)?;
        self.wakers.lock().unwrap().insert(fd, waker);
        Ok(())
    }
}
```

在 `Reactor` 的添加事件的方法中，首先调用 `epoll` 的 `add_event` 方法注册文件描述符和监听的事件，然后把描述符和对应的 `waker` 存储在 `BTreeMap<RawFd, Waker>` 中。



## reactor 循环

```rust,noplayground
fn reactor_main_loop() -> io::Result<()> {
    info!("Start reactor main loop");
    let max_event = 32;
    let event: libc::epoll_event = unsafe { mem::zeroed() };
    let mut events = vec![event; max_event];
    let reactor = &REACTOR;

    loop {
        let nfd = reactor.epoll.wait(&mut events)?;
        info!("(Reactor) wake up. nfd = {}", nfd);

        #[allow(clippy::needless_range_loop)]
        for i in 0..nfd {
            let fd = events[i].u64 as RawFd;
            if let Some(waker) = reactor.wakers.lock().unwrap().remove(&fd) {
                info!("(Reactor) delete event: {}", fd);
                reactor.epoll.del_event(fd)?;
                waker.wake();
            }
        }
    }
}
```

在 `reactor_main_loop` 函数中，我们使用一个 `loop` 循环，在循环中调用 `epoll` 的 `wait` 方法获取所有就绪的 IO 事件的文件描述符，如果没有事件就绪，`wait` 方法就会阻塞 `reactor` 线程，避免 CPU 空转。

然后遍历就绪的描述符，从 `wakers` 中获取描述符对应的 `waker`，之后调用 `epoll` 的 `delete_event` 方法删除描述符，表示这个事件已经处理完毕。

最后，调用 `waker` 的 `wake` 方法，把因为等待IO事件而挂起的 `task` 发送到 `executor` 的执行队列中。



## REACTOR 静态变量

```rust,noplayground
lazy_static! {
    pub(crate) static ref REACTOR: Reactor = {
        // Start reactor main loop
        std::thread::spawn(move || {
            reactor_main_loop()
        });

        Reactor {
            epoll: Epoll::new().expect("failed to create epoll"),
            wakers: Mutex::new(BTreeMap::new())
        }
    };
}
```

`Executor` 在主线程运行，负责调度执行 `task`，而 `reactor_main_loop` 内部使用一个无线循环不断地获取就绪的 `fd`，并唤醒挂起的 `task`。为了避免 `reactor_main_loop` 阻塞 `Executor`，我们就开一个线程去执行 `reactor_main_loop`。

之所以把 `REACTOR` 创建成全局静态变量，是为了在其他的模块中方便地调用 `REACTOR` 的方法。
