# example

在这一节中，我们将使用之前实现的异步运行时创建一个 `tcp echo server`。需要导入的模块如下所示：

```rust,noplayground
use std::env;
use std::io::Write;

use log::info;

use async_runtime::async_io::{Ipv4Addr, TcpListener, TcpStream};
use async_runtime::executor::{spawner_and_executor, Spawner};
```



## 日志打印

```rust,noplayground
fn init_log() {
    // format = [file:line] msg
    env::set_var("RUST_LOG", "info");
    env_logger::Builder::from_default_env()
        .format(|buf, record| {
            writeln!(
                buf,
                "[{}:{:>3}] {}",
                record.file().unwrap_or("unknown"),
                record.line().unwrap_or(0),
                record.args(),
            )
        })
        .init();
}
```

`init_log` 函数是为了设置日志打印的消息格式，这跟异步运行时的使用没啥关系，这里就不再赘述了。



## handle_client

```async
async fn handle_client(stream: TcpStream) {
    let mut buf = [0u8; 1024];
    info!("(handle client) {}", stream.raw_fd());
    loop {
        let n = stream.read(&mut buf).await.unwrap();
        if n == 0 {
            break;
        }
        stream.write(&buf[..n]).await.unwrap();
    }
}
```

在 `handle_client` 函数中，我们首先创建一个 `buf` 数组，然后打印一个 `handle client` 的日志消息，接着开启一个无限循环：

* 调用 `stream.read(&mut buf)` 方法后会返回一个 `ReadFuture`，在 `ReadFuture` 上调用 `await` 方法：
  * `ReadFuture.await` 会展开成一个无限循环，在循环内部会调用 `ReadFuture` 的 `poll` 方法。
  
  * 如果返回 `Poll::Pending`，则使用 `Yield` 表达式挂起当前的 `task`；
  
  * 如果返回 `poll::Ready` 则中断循环并返回结果。
  
    
  
* 如果 `n== 0`，则说明客户端已经断开了连接，则退出循环。

  

* 调用 `stream.write(&mut buf[.n])` 会返回一个 `WriteFuture`，在 `WriteFuture` 上调用 `await` 方法后执行流程与 `ReadFuture` 一致。



## server_loop

```rust,noplayground
async fn server_loop(spawner: Spawner) {
    let addr = Ipv4Addr::new(127, 0, 0, 1);
    let port = 8080;
    let listener = TcpListener::bind(addr, port).unwrap();

    let incoming = listener.incoming();

    while let Some(stream) = incoming.next().await {
        let stream = stream.unwrap();
        spawner.spawn(handle_client(stream));
    }
}
```

在 `server_loop` 函数中，我们调用 `TcpListener` 的 `bind` 方法创建一个 `TcpListener` 实例，然后调用 `incoming` 方法创建一个 `Incoming` 实例。

接着，在 `While let` 循环中，调用 `incoming` 的 `next` 方法返回一个 `AcceptFuture` 实例，在 `AcceptFuture` 上调用 `await` 方法后，如果返回的 `Poll::Pending`，则挂起当前的 `task`。

当有客户端连接到来时，`AcceptFuture` 等待的 IO 事件就绪，会返回 `io::Readult<TcpStream>` 的实例并绑定到 `stream` 变量上，接着使用 `spawner` 调用 `spawn` 方法创建一个 `task` 处理与客户端的交互。

最后，又进入循环的开始位置，继续等待新的连接到来。



## main 函数

```rust,noplayground
fn main() {
    init_log();

    let (spawner, mut executor) = spawner_and_executor();

    spawner.spawn(server_loop(spawner.clone()));

    executor.run();
}
```

在 `main` 函数中，我们首先调用 `init_log` 函数设置日志消息格式，接着使用 `spawner_and_executor` 函数创建 `Spawner` 和 `Executor` 的实例。

然后调用 `spawner.spawn` 方法创建一个 `task` 用于执行 `server_loop`。

最后调用 `executor.run()` 方法开启 `executor` 的运行，开始调度执行各个 `task`。



## 运行示例

开启运行后的 `echo server` 会监听地址：`127.0.0.1:8080`：

```rust,noplayground
cargo run --example echo_server
    Finished dev [unoptimized + debuginfo] target(s) in 0.69s
     Running `target/debug/examples/echo_server`
[src/async_io.rs: 56] (TcpListener) listen: 3
[src/reactor.rs: 27] (Reactor) add event: 3
[src/reactor.rs: 35] Start reactor main loop
```



使用 `Python` 写一个小脚本模拟 `TCP` 客户端：

```python
import socket
import threading

HOST = '127.0.0.1'
PORT = 8080


def send_request():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((HOST, PORT))
    for i in range(1, 1025):
        s.send(f"HELLO WORLD[{i}]".encode())
        data = s.recv(1024).decode()
        print(f"RECEIVE DATA: '{data}' in THREAD[{threading.currentThread().name}]")
    s.close()


def main():
    t_lst = []
    for _ in range(10):
        t = threading.Thread(target=send_request)
        t_lst.append(t)
        t.start()

    for t in t_lst:
        t.join()


if __name__ == '__main__':
    main()
```

运行脚本，服务端会输出以下内容：

```
.....
.....
.....
[src/reactor.rs: 43] (Reactor) wake up. nfd = 2
[src/reactor.rs: 49] (Reactor) delete event: 6
[src/reactor.rs: 49] (Reactor) delete event: 7
[src/reactor.rs: 43] (Reactor) wake up. nfd = 3
[src/reactor.rs: 49] (Reactor) delete event: 9
[src/reactor.rs: 27] (Reactor) add event: 6
[src/reactor.rs: 49] (Reactor) delete event: 10
[src/reactor.rs: 49] (Reactor) delete event: 11
[src/reactor.rs: 43] (Reactor) wake up. nfd = 2
```

客户端的输出内容如下所示：

```
.....
.....
.....
RECEIVE DATA: 'HELLO WORLD[1022]' in THREAD[Thread-3]
RECEIVE DATA: 'HELLO WORLD[1015]' in THREAD[Thread-7]
RECEIVE DATA: 'HELLO WORLD[1013]' in THREAD[Thread-6]
RECEIVE DATA: 'HELLO WORLD[1021]' in THREAD[Thread-1]
RECEIVE DATA: 'HELLO WORLD[1023]' in THREAD[Thread-3]
RECEIVE DATA: 'HELLO WORLD[1008]' in THREAD[Thread-10]
RECEIVE DATA: 'HELLO WORLD[1014]' in THREAD[Thread-6]
RECEIVE DATA: 'HELLO WORLD[1016]' in THREAD[Thread-7]
```

可以看出，我们的 `echo server` 正确地返回了响应，`wake up. nfd = 3` 表示有3个事件同时就绪，这说明 `server` 确实在并发地处理多个请求！

