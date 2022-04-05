# task

`Task` 是对 `async fn` 或者 `async {}` 创建的 `Non-Leaf Future` 的抽象，一个 `task` 就代表一个异步执行的任务：

```rust,noplayground
#[derive(Debug, Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
pub(crate) struct TaskId(u64);

impl TaskId {
    pub(crate) fn new() -> Self {
        static NEXT_ID: AtomicU64 = AtomicU64::new(0);
        TaskId(NEXT_ID.fetch_add(1, Ordering::Relaxed))
    }
}

pub(crate) struct Task {
    id: TaskId,
    future: Mutex<Pin<Box<dyn Future<Output = ()> + 'static + Send>>>,
    task_sender: Sender<Arc<Task>>,
}
```

`Task` 中有三个字段：

* `id`：每个 `task` 都有一个唯一的 `TaskId`，`TaskId` 是有可能在不同的线程中创建的，因此使用原子类型 `AtomicU64` 来创建 `TaskId` 的实例，保证唯一性。

  

* `future`：对用户创建的 `Non-Leaf Future` 的包装，使用 `Pin` 的目的是为了安全地使用自引用结构（`Future` 生成的状态机中可能存在自引用结构），使用 `Mutex` 的目的稍后讲解。

  

* `task_sender`：一个 `channel` 的发送端，发送的 item 是 `Arc<Task>`，之所以使用 `Arc<Task>` 一方面是想要减小克隆 `Task` 的开销，另一方面与 `Waker` 的实现机制有关（稍后讲解）。



## 方法实现

```rust,noplayground
impl Task {
    pub(crate) fn new(
        future: impl Future<Output = ()> + 'static + Send,
        task_sender: Sender<Arc<Task>>,
    ) -> Self {
        Task {
            id: TaskId::new(),
            future: Mutex::new(Box::pin(future)),
            task_sender,
        }
    }

    pub(crate) fn task_id(&self) -> TaskId {
        self.id
    }

    pub(crate) fn poll(&self, context: &mut Context) -> Poll<()> {
        self.future
            .lock()
            .expect("get lock failed")
            .as_mut()
            .poll(context)
    }
}
```

`new` 方法中传入参数 `future` 和 `task_sender` 后创建一个 `Task` 实例：

* 参数 `Future` 要求满足 `'static` 生命周期是因为 `task` 的存在时间可能是任意长的，因此需要 `Future` 具有静态生命周期。
* 要求 `Future` 满足 `Send` 是因为 `Task` 需要跨线程发送。
* 由于 `Future` 最终使用 `Mutex` 包了起来，因此 `future` 字段最终同时满足 `Send + Sync + 'static`。
* `Task` 的其他两个字段也满足 `Send + Sync + 'static` ，因此 `Task` 满足 `Send + Sync + 'static`。

由于 `task_sender` 发送的 item 是 `Arc<Task>`，`executor` 的执行队列中收到的也是 `Arc<Task>`，因此 `poll` 方法的定义中只能使用 `&self` 不变引用。

又因为 `Future` 的 `poll` 方法调用需要可变引用，为了实现内部可变性，我们就用 `Mutex` 把 `Pin<Box<Future>>` 包了起来，这就是使用 `Mutex` 的原因。

在 `poll` 方法中，首先调用 `self.future.lock()` 获取锁，然后将调用 `.as_mut()` 方法获取 `Pin<&mut dyn Future>`，最后再调用 `Future` 中的 `poll` 方法执行 `Future`。



## 实现 Wake trait

为 `Task` 实现 `Wake trait`，这样就可以通过 `Task` 来构建一个 `Waker`：

 ```rust,noplayground
 impl Wake for Task {
     fn wake(self: Arc<Self>) {
         self.task_sender
             .send(self.clone())
             .expect("send task failed");
     }
 
     fn wake_by_ref(self: &Arc<Self>) {
         self.task_sender
             .send(self.clone())
             .expect("send task failed");
     }
 }
 ```

`Wake` 中的 `wake/wake_by_ref` 方法实现就是具体的唤醒 `task` 的机制，在这个实现中，我们把想要唤醒的 `task` 通过 `task_sender` 发送到 `executor` 的执行队列中，这样 `executor` 就可以执行这个 `task` 了，这也是在 `Task` 定义中，需要 `task_sender` 字段的原因。

此外，`wake/wake_by_ref` 方法中都需要 `Arc<Task>`，这是 `task_sender` 的 item 类型为 `Arc<Task>` 的原因之一。



## 构造 Waker

对于实现了 `Wake trait` 的 `Task`，可以使用  `std::task::Waker` 的 `from` 方法构造一个 `Waker`：

```rust,noplayground
impl<W: Wake + Send + Sync + 'static> From<Arc<W>> for Waker {
    fn from(waker: Arc<W>) -> Waker {}
}
```

通过前面的分析，我们知道 `Task` 已经同时满足 `Wake + Send + Sync + 'static`，因此可以安全地使用 `from` 方法构造一个 `Waker`。
