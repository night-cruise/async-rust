# Epoll server example

在本节中，我们将会编写一个简单的 `epoll server`，来看一下 `epoll` 是如何工作的。[libc crate](https://crates.io/crates/libc) 中提供了与 `epoll` 相关的系统调用，因此这个小项目需要添加 `libc crate` 依赖。 

## epoll 调用宏

为了方便地调用 `epoll` 相关的 `api`，我们可以编写如下所示的宏：

```rust,noplayground
#[macro_export]
macro_rules! syscall {
    ($fn: ident ( $($arg: expr),* $(,)* ) ) => {{
        let res = unsafe { libc::$fn($($arg, )*) };
        if res == -1 {
            Err(std::io::Error::last_os_error())
        } else {
            Ok(res)
        }
    }};
}
```

例如，现在我们可以这样调用 `epoll_wait`：

```rust,noplayground
syscall!(epoll_wait(
            epoll_fd,
            events.as_mut_ptr() as *mut libc::epoll_event,
            1024,
            1000
))
```

宏展开后的代码如下所示：

```rust,noplayground
{
    let res = unsafe {
        libc::epoll_wait(
            epoll_fd,
            events.as_mut_ptr() as *mut libc::epoll_event,
            1024,
            1000
        )
    };
    if res == -1 {
        Err(std::io::Error::last_os_error())
    } else {
        Ok(res)
}
```

## epoll 模块

接下来，我们将会利用 `epoll` 提供的 `api` 来编写 IO 事件的注册、修改等功能。本模块需要导入的项：

```rust,noplayground
use std::io;
use std::os::unix::io::RawFd;

use crate::syscall;
```

### 创建 epoll 实例

```rust,noplayground
/// 包装epoll_create，创建一个epoll实例
pub fn epoll_create() -> io::Result<RawFd> {
    // 创建一个epoll实例，返回epoll对象的文件描述符fd
    let fd = syscall!(epoll_create1(0))?;

    // fcntl(fd, libc::F_GETFD) 函数返回与 fd 关联的 close_on_exec 标志
    // close_on_exec 用于确定在系统调用 execve() 后是否需要关闭文件描述符
    if let Ok(flags) = syscall!(fcntl(fd, libc::F_GETFD)) {

        // 设置在系统调用 execve() 后关闭文件描述符 fd
        let _ = syscall!(fcntl(fd, libc::F_SETFD, flags | libc::FD_CLOEXEC));
    }

    Ok(fd)
}
```

### 注册文件描述并监听事件

```rust,noplayground
/// 包装 epoll_ctl，注册文件描述符和事件
pub fn add_interest(epoll_fd: RawFd, fd: RawFd, mut event: libc::epoll_event) -> io::Result<()> {
    // epoll_fd 是 epoll 实例的的文件描述符
    // fd 是要注册的目标文件描述符
    // event 是要在 fd 上监听的事件
    // libc::EPOLL_CTL_ADD 表示添加一个需要监视的文件描述符
    syscall!(epoll_ctl(epoll_fd, libc::EPOLL_CTL_ADD, fd, &mut event))?;

    Ok(())
}
```

### 修改注册的文件描述符

```rust,noplayground
/// 包装 epoll_ctl，修改文件描述符
pub fn modify_interest(epoll_fd: RawFd, fd: RawFd, mut event: libc::epoll_event) -> io::Result<()> {
    // epoll_fd 是 epoll 实例的的文件描述符
    // fd 是要修改目标文件描述符
    // event 是要在 fd 上监听的事件
    // libc::EPOLL_CTL_MOD 表示修改文件描述符 fd
    syscall!(epoll_ctl(epoll_fd, libc::EPOLL_CTL_MOD, fd, &mut event))?;

    Ok(())
}
```

### 删除注册的文件描述符

```rust,noplayground
/// 包装 epoll_ctl，删除文件描述符
pub fn remove_interest(epoll_fd: RawFd, fd: RawFd) -> io::Result<()> {
    // epoll_fd 是 epoll 实例的的文件描述符
    // fd 是要删除的目标文件描述符
    // libc::EPOLL_CTL_DEL 表示要删除文件描述符 fd
    syscall!(epoll_ctl(
        epoll_fd,
        libc::EPOLL_CTL_DEL,
        fd,
        std::ptr::null_mut() // 将监听的 event 设置为空
    ))?;

    Ok(())
}
```

### 关闭文件描述符

```rust,noplayground
/// 关闭文件描述符 fd
pub fn close(fd: RawFd) {
    let _ = syscall!(close(fd));
}
```

### 创建一个读事件

```rust,noplayground
const READ_FLAGS: i32 = libc::EPOLLONESHOT | libc::EPOLLIN;

/// 创建一个读事件
pub fn listener_read_event(key: u64) -> libc::epoll_event {
    // key 用于区分不同的文件描述符
    libc::epoll_event {
        events: READ_FLAGS as u32,
        u64: key,
    }
}
```

### 创建一个写事件

```rust,noplayground
const WRITE_FLAGS: i32 = libc::EPOLLONESHOT | libc::EPOLLOUT;

/// 创建一个写事件
pub fn listener_write_event(key: u64) -> libc::epoll_event {
    // key 用于区分不同的文件描述符
    libc::epoll_event {
        events: WRITE_FLAGS as u32,
        u64: key,
    }
}
```

## http 模块

在 `http` 模块中，我们将会编写处理 `HTTP` 请求相关的函数，需要导入的项：

```rust,noplayground
use std::io;
use std::net::TcpStream;
use std::io::{Read, Write};
use std::os::unix::io::{AsRawFd, RawFd};

use crate::epoll::{
    close, listener_read_event, listener_write_event, modify_interest, remove_interest,
};
```

### 请求上下文

将与客户端建立的连接抽象成请求上下文：

```rust,noplayground
/// 请求上下文，用于处理 HTTP 请求
#[derive(Debug)]
pub struct RequestContext {
    /// 与客户端建立的连接的 stream 流
    pub stream: TcpStream,
    /// 收到的 HTTP 请求的 content-length 的值
    pub content_length: usize,
    /// 收到的 HTTP 请求的数据写入的缓冲区
    pub buf: Vec<u8>,
}
```

接下来编写的函数，都是为 `RequestContext` 实现的方法。

### 创建请求上下文

```rust,noplayground
pub fn new(stream: TcpStream) -> Self {
    Self {
        stream,
        buf: Vec::new(),
        content_length: 0,
    }
}
```

### 从 stream 流中读取数据

```rust,noplayground
pub fn read_cb(&mut self, key: u64, epoll_fd: RawFd) -> io::Result<()> {
    let mut buf = [0u8; 4096];

    // 从 stream 流中读取数据写入到 buf 中
    match self.stream.read(&mut buf) {
        Ok(_) => {
            if let Ok(data) = std::str::from_utf8(&buf) {

                // 解析并且设置读取到的 HTTP 请求的 content-length 字段的值
                self.parse_and_set_content_length(data);
            }
        }
        Err(e) if e.kind() == io::ErrorKind::WouldBlock => {}
        Err(e) => {
            return Err(e);
        }
    };

    // 将读取的数据扩展到 RequestContext 的 buf 中
    self.buf.extend_from_slice(&buf);

    // 如果 buf 中的数据长度大于等于 content-length，说明从客户端发送的 HTTP 请求已经读取完毕
    if self.buf.len() >= self.content_length {
        println!("got all data: {} bytes", self.buf.len());

        // 将在 stream 上监听的事件修改为写事件
        modify_interest(epoll_fd, self.stream.as_raw_fd(), listener_write_event(key))?;
    } else {

        // 将在 stream 上监听的事件修改为读事件，继续读取剩下的 HTTP 请求
        modify_interest(epoll_fd, self.stream.as_raw_fd(), listener_read_event(key))?;
    }

    Ok(())
}
```

### 解析 HTTP 请求

```rust,noplayground
/// 解析并且设置读取到的 HTTP 请求的 content-length 字段的值
pub fn parse_and_set_content_length(&mut self, data: &str) {
    if data.contains("HTTP") {
        if let Some(content_length) = data
            .lines()
            .find(|l| l.to_lowercase().starts_with("content-length: "))
        {
            if let Some(len) = content_length
                .to_lowercase()
                .strip_prefix("content-length: ")
            {
                self.content_length = len.parse::<usize>().expect("content-length is valid");
                println!("set content length: {} bytes", self.content_length);
            }
        }
    }
}
```

### 写入返回数据到 stream 流中

为了简单起见，我们直接返回一段固定的 `HTTP` 文本。

```rust,noplayground
// 返回的响应为固定的 HTTP 文本
const HTTP_RESP: &[u8] = br#"HTTP/1.1 200 OK
content-type: text/html
content-length: 28

Hello! I am an epoll server."#;

/// 将要返回的 HTTP 数据写入到 stream 流中
pub fn write_cb(&mut self, key: u64, epoll_fd: RawFd) -> io::Result<()> {
    // 写入数据到 stream 流中
    match self.stream.write(HTTP_RESP) {
        Ok(_) => println!("answered from request {}", key),
        Err(e) => eprintln!("could not answer to request {}, {}", key, e),
    };

    // 关闭 stream 流
    self.stream.shutdown(std::net::Shutdown::Both)?;

    let fd = self.stream.as_raw_fd();
    // 移除在 epoll 中注册的文件描述符 fd
    remove_interest(epoll_fd, fd)?;

    // 关闭文件描述符 fd
    close(fd);

    Ok(())
}
```

## main 模块

接下来，我们将会编写 `server` 的入口函数，主要的逻辑为：注册文件描述符  =>  调用 `epoll_wait` 获取就绪的事件  =>  根据不同的事件进行处理。

```rust,noplayground
use std::collections::HashMap;
use std::io;
use std::net::TcpListener;
use std::os::unix::io::AsRawFd;

use rust_epoll_example::epoll::{add_interest, epoll_create, listener_read_event, modify_interest};
use rust_epoll_example::http::RequestContext;
use rust_epoll_example::syscall;

fn main() -> io::Result<()> {
    // 存储 RequestContext 实例，key 用来区分不同的 RequestContext
    let mut request_contexts: HashMap<u64, RequestContext> = HashMap::new();

    // 存储就绪的 event
    let mut events: Vec<libc::epoll_event> = Vec::with_capacity(1024);

    // key 对应 epoll_event 中的 u64 字段，用于区分文件描述、RequestContext
    let mut key = 100;

    // 创建一个 listener，并监听 8000 端口
    let listener = TcpListener::bind("127.0.0.1:8000")?;

    // 将 socket 设置为非阻塞
    listener.set_nonblocking(true)?;

    // 获取 listener 对应文件描述符
    let listener_fd = listener.as_raw_fd();

    // 创建 epoll 实例，返回 epoll 文件描述符
    let epoll_fd = epoll_create().expect("can create epoll queue");

    // 在 epoll 实例中注册 listener 文件描述符，并监听读事件
    // key 等于 100，对应 listener 文件描述符
    add_interest(epoll_fd, listener_fd, listener_read_event(key))?;

    loop {
        println!("requests in flight: {}", request_contexts.len());
        events.clear();

        // 将就绪的事件添加到 events vec 中，返回就绪的事件数量
        let res = match syscall!(epoll_wait(
            epoll_fd,
            events.as_mut_ptr() as *mut libc::epoll_event,
            1024,
            1000,
        )) {
            Ok(v) => v,
            Err(e) => panic!("error during epoll wait: {}", e),
        };

        // safe  as long as the kernel does nothing wrong - copied from mio
        // 根据就绪的事件数量设置 events vec 的长度
        unsafe { events.set_len(res as usize) };

        // 遍历处理就绪的事件
        for ev in &events {
            match ev.u64 {
                // key = 100 说明是在 listener fd 上监听的读事件就绪了
                100 => {
                    match listener.accept() {

                        // stream 是与客户端建立的连接的 stream 流
                        Ok((stream, addr)) => {
                            // 设置为非阻塞
                            stream.set_nonblocking(true)?;

                            // 有一个新的连接来了
                            println!("new client: {}", addr);
                            key += 1;

                            // 在 epoll 中注册 stream 文件描述符，并监听读事件
                            add_interest(epoll_fd, stream.as_raw_fd(), listener_read_event(key))?;

                            // 创建一个 RequestContext，并保存到 request_contexts 中
                            request_contexts.insert(key, RequestContext::new(stream));
                            // 上面使用的 key，用来区分不同的文件描述符和 RequestContext
                        }
                        Err(e) => eprintln!("couldn't accept: {}", e),
                    };

                    // 修改在 listener fd 上监听的的事件为读事件（继续等待新的连接到来）
                    modify_interest(epoll_fd, listener_fd, listener_read_event(100))?;
                }
                // key != 100，说明是其他的 fd 上监听的事件就绪了
                key => {
                    let mut to_delete = None;

                    // 获取这个 key 对应的 RequestContext
                    if let Some(context) = request_contexts.get_mut(&key) {

                        let events: u32 = ev.events;

                        // 匹配就绪的事件是读事件还是写事件
                        match events {

                            // 读事件就绪
                            v if v as i32 & libc::EPOLLIN == libc::EPOLLIN => {

                                // 读取请求数据
                                context.read_cb(key, epoll_fd)?;
                            }
                            // 写事件就绪
                            v if v as i32 & libc::EPOLLOUT == libc::EPOLLOUT => {

                                // 写入返回数据
                                context.write_cb(key, epoll_fd)?;

                                // 返回数据后，就删除对应的 RequestContext，
                                // 当客户端再次发起请求时会建立新的连接，创建新的 RequestContext
                                to_delete = Some(key);
                            }
                            v => println!("unexpected events: {}", v),
                        };
                    }

                    // 写事件处理完毕，删除对应的 RequestContext
                    if let Some(key) = to_delete {
                        request_contexts.remove(&key);
                    }
                }
            }
        }
    }
}
```

`HTTP` 协议是无状态的，我们在完整处理一次请求后就删除对应的请求上下文，当客户端再次发起请求时会建立新的连接，创建新的请求上下文。

至此，`epoll server` 编写完毕，源代码的仓库地址：[rust epoll example](https://github.com/night-cruise/rust-epoll-example)。

## 运行 server

使用 `cargo run` 启动 `server`，然后这个 `server` 会监听地址：[http://127.0.0.1:8000](http://127.0.0.1:8000/) 。

为了测试 `server`，编写一个 `Python` 小脚本，使用多线程循环发送 `HTTP` 请求：

```python
import requests

from threading import Thread

with open('image.jpeg', 'rb') as f:
    FILE = f.read()


# send request to http://127.0.0.1:8000
def send_request(host, port):
    for _ in range(100):
        r = requests.post(f"http://{host}:{port}", data={'file': FILE})
        print(f"Receive response: '{r.text}' from {r.url}")


if __name__ == '__main__':
    t_lst = []
    for _ in range(4):
        t = Thread(target=send_request, args=('127.0.0.1', 8000))
        t_lst.append(t)
        t.start()

    for t in t_lst:
        t.join()
```

`client` 端的输出：

```
.....
.....
Receive response: 'Hello! I am an epoll server.' from http://127.0.0.1:8000/
Receive response: 'Hello! I am an epoll server.' from http://127.0.0.1:8000/
Receive response: 'Hello! I am an epoll server.' from http://127.0.0.1:8000/
Receive response: 'Hello! I am an epoll server.' from http://127.0.0.1:8000/
Receive response: 'Hello! I am an epoll server.' from http://127.0.0.1:8000/
Receive response: 'Hello! I am an epoll server.' from http://127.0.0.1:8000/
Receive response: 'Hello! I am an epoll server.' from http://127.0.0.1:8000/
Receive response: 'Hello! I am an epoll server.' from http://127.0.0.1:8000/
Receive response: 'Hello! I am an epoll server.' from http://127.0.0.1:8000/
```

`server` 端的输出：

```
.....
.....
requests in flight: 3
requests in flight: 3
requests in flight: 3
requests in flight: 3
requests in flight: 3
requests in flight: 3
requests in flight: 3
requests in flight: 3
requests in flight: 3
requests in flight: 3
requests in flight: 3
got all data: 9379840 bytes
requests in flight: 3
answered from request 195
```

正如我们所看到的那样，`server` 在同时处理多个请求！
