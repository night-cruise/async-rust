# Future trait

在前文中，我们提到使用 `async` 标记的 `fn`、`block`、`closure` 都会返回一个 `Future`，本节将会详细地介绍 `Future` 的概念。

在标准库中，`Future` 的定义如下所示：

```rust,noplayground
pub trait Future {
    type Output;	// Future计算完成时产生的值的类型
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

`Future` 表示一个异步计算，或者说会在未来完成计算的操作。`Future` 的核心是 `poll` 方法，当调用 `poll` 方法时会尝试计算 `Future` 得到最终的值。如果值还没有准备好（例如等待某些事件发生），则此方法不会阻塞，而是会直接返回一个结果表示 `Future` 还没有计算完毕。

>注意：`Future trait` 中涉及到的 `Pin` 将会在后面的章节中介绍。



## poll

在上面对 `Future` 的介绍中，我们简要提到了 `poll` 方法，下面我们会对 `poll` 方法进行更详细的介绍。当调用 `Future` 的 `poll` 方法时会返回一个枚举类型的值：

* `Poll::Pending`，表示这个 `Future` 还没计算完成
* `Poll::Ready(val)`，表示这个 `Future` 计算完毕，并附带计算结果：`val`

如果 `Future` 没有计算完成，例如想要等待一个 IO 事件发生，那么在 `poll` 方法体内，我们通常会调用传递给 `poll` 方法的 `Context` 的 `waker` 方法拿到一个 `Waker`（通常把 `Waker` 叫做唤醒器），然后注册这个 `Waker` 到一个“事件通知系统”中，最后返回 `Pending` 表示 `Future` 没有计算完成。

在未来某一时刻，`Future` 等待的 IO 事件就绪了，那么“事件通知系统”就会利用我们注册的 Waker 通过某种唤醒机制唤醒这个 `Future`，通过 `poll` 继续计算执行该 `Future`。

通过 `Waker` 唤醒器，我们可以只在 `Future` 想要等待的事件就绪时，才去唤醒 `Future`。这样我们就不需要通过一个死循环不断的调用 `poll` 方法来驱动 `Future` 的执行，这是异步编程之所以高效的关键所在。



## 小栗子

下面我们使用一个具体的例子来介绍 `Future trait` 的使用。

假设我们准备读取一个 `socket`，但是它可能还有准备好数据。如果数据准备好了，我们就可以读取它然后然后返回 `Poll::Ready(data)`，但是如果数据没有准备好，我们可以注册一个唤醒器到“事件通知系统”中：

```rust,noplayground
struct SocketRead<'a> {
	socket: &'a Socket
}

impl<'a> Future for SocketRead<'a> {
	type Output = Vec<u8>;
	
	fn poll(self: Pin<&mut Self>, cx: &mut Context<'_'>) -> Poll<Self::Output> {
		let data = self.socket.no_block_read::<Option<Vec<u8>>>(1024);
		match data {
			Some(data) => Poll::Ready(data),
			None => {
				REACTOR.registe_waker_and_event(self.socket, Type::Read, cx.waker().clone());
				Poll::Pending
			}
		}
	}
}
```

代码中的 `REACTOR `就是前文中所提到过的“事件通知系统”。当 `socket` 中有数据可读时，`REACTOR `就会使用注册的 `Waker` 唤醒负责 `SocketRead `，然后调用 `poll` 方法再次计算该 `Future`。



## Leaf / Non-leaf Future

在前文中我们提到使用 `async` 关键字可以创建一个 `Future` 类型，而在上面的小栗子中我们通过实现 `Future trait` 的方式也创建了一个 `Future` 类型，那么这两个 `Future` 有什么区别呢？



### Leaf Future

通过为我们的自定义类型实现 `Future trait` 的方式创建的 `Future` 被称为 `Leaf Future`。例如上面的小栗子中的 `SocketRead` 类型：

```rust,noplayground
struct SocketRead<'a> {
	socket: &'a Socket
}

impl<'a> Future for SocketRead<'a> {
	/
}
```

`Leaf Future` 中通常会涉及到对 IO 的操作，例如从一个 `socket` 中读取数据，并且对 IO 的操作是非阻塞式的。

当调用异步运行时提供的异步读 `socket` 的方法时就会返回上述的 `Future`：

```rust,noplayground
impl async_runtime {
	fn read_socket(&self) -> SocketRead {
		// ...
	}
}

let mut leaf_future: SocketRead = async_runtime.read_socket();
```

通常情况下，这些 `Leaf Future` 都是由异步运行时自己创建的，用户只需要使用 `async/await` 关键字即可。



### Non-leaf Future

`Non-leaf Future` 是我们使用 `async` 关键字创建 `Future`，并且会由 `async runtime` 来调度运行。

在 `Non-leaf Future` 中可以创建多个 `Leaf Future`， 并且通过 `await` `Leaf Future` 来完成对 IO 的操作：

```rust,noplayground
let non_leaf_future = async {
	let data = async_runtime.read_socket().await;
	println!("Receive data: {:?}", data);
	
	let data = async_runtime.read_socket().await;
	println!("Receive data: {:?}", data);
	
	let data = async_runtime.read_socket().await;
	println!("Receive data: {:?}", data);
}
```

在 `await` 一个 `Leaf Future` 时，如果返回的是 `Pending`，那么`Non-Leaf Future` 就会让出对当前线程的控制权，此时 `async runtime` 就能够调度执行其他的 `Non-Leaf Future` 。当 `Non-Leaf Future` 中的 IO 操作就绪时，`async runtime` 就会重新激活挂起的 `Future`，在**上次离开的地方继续运行**。
