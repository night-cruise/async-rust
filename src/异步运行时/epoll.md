# epoll

就像 `Epoll servere example` 一节中那样，为了方便地调用 `libc` 提供的 `api`，我们先创建一个 `syscall` 宏：

```rust,noplayground
macro_rules! syscall {
    ($fn: ident ( $($arg: expr),* $(,)* ) ) => {{
        let res = unsafe { libc::$fn($($arg, )*) };
        if res == -1 {
            Err(io::Error::last_os_error())
        } else {
            Ok(res)
        }
    }};
}
```



## Epoll 抽象

抽象出 `Epoll` 和 `EpollEventType` 类型：

```rust,noplayground
pub(crate) struct Epoll {
    fd: RawFd,
}

pub(crate) enum EpollEventType {
    // Only event types used in this example
    In,
    Out,
}
```

`RawFd` 表示原始文件描述符。



## 方法实现

### new

创建一个 `Epoll` 实例：

```rust,noplayground
pub(crate) fn new() -> io::Result<Self> {
    let fd = syscall!(epoll_create1(libc::EPOLL_CLOEXEC))?;
    Ok(Epoll { fd })
}
```



### 添加事件/修改事件

```rust,noplayground
fn run_ctl(&self, epoll_ctl: libc::c_int, fd: RawFd, op: EpollEventType) -> io::Result<()> {
    let mut event: libc::epoll_event = unsafe { mem::zeroed() };
    event.u64 = fd as u64;
    event.events = match op {
        EpollEventType::In => libc::EPOLLIN as u32,
        EpollEventType::Out => libc::EPOLLOUT as u32,
    };

    let event_p: *mut _ = &mut event as *mut _;
    syscall!(epoll_ctl(self.fd, epoll_ctl, fd, event_p))?;

    Ok(())
}

pub(crate) fn add_event(&self, fd: RawFd, op: EpollEventType) -> io::Result<()> {
    self.run_ctl(libc::EPOLL_CTL_ADD, fd, op)
}

#[allow(dead_code)]
pub(crate) fn mod_event(&self, fd: RawFd, op: EpollEventType) -> io::Result<()> {
    self.run_ctl(libc::EPOLL_CTL_MOD, fd, op)
}
```

`add_event` 和 `mod_event` 都是通过调用`run_ctl` 方法实现的。在 `run_ctl` 方法中根据 `op` 类型设置要注册/修改的事件类型，然后调用 `epoll_ctl` 方法来注册/修改事件。



### 删除事件

```rust,noplayground
pub(crate) fn del_event(&self, fd: RawFd) -> io::Result<()> {
    syscall!(epoll_ctl(
        self.fd,
        libc::EPOLL_CTL_DEL,
        fd,
        std::ptr::null_mut() as *mut libc::epoll_event
    ))?;

    Ok(())
}
```

删除在 `Epoll` 实例中注册描述符 `fd`。



### 等待就绪事件

```rust,noplayground
pub(crate) fn wait(&self, events: &mut [libc::epoll_event]) -> io::Result<usize> {
    let nfd = syscall!(epoll_wait(
        self.fd,
        events.as_mut_ptr(),
        events.len() as i32,
        -1
    ))?;

    Ok(nfd as usize)
}
```

调用 `epoll_wait` 函数获取所有就绪的文件描述符，并将就绪的描述符存放到 `events` 中，最后返回就绪的描述符数量。



## 关闭Epoll

为 `Epoll` 实现 `Drop trait`，在清理 `Epoll` 时关闭 `Epoll` 的文件描述符：

```rust,noplayground
impl Drop for Epoll {
    fn drop(&mut self) {
        syscall!(close(self.fd)).ok();
    }
}
```
