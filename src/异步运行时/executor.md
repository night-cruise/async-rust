# executor

## Executor

```rust,noplayground
pub struct Executor {
    task_queue: Receiver<Arc<Task>>,
    waker_cache: BTreeMap<TaskId, Waker>,
}
```

`task_queue` 是一个 `channel` 的接收端，当 `spawn` 或者 `wake` 一个 `task` 时，就会发送 `Arc<Task>` 到 `task_queue` 中。

`waker_cache` 使用 `BTreeMap` 缓存可能会重复使用的 `Waker`，这是为了减小构造 `Waker` 的开销。

> 实际上，`Executor` 中的 `task_queue` 只是一个管道的接收端，并不是队列，只是我个人更习惯称之为队列。

### 方法实现

```rust,noplayground
impl Executor {
    fn new(task_queue: Receiver<Arc<Task>>) -> Self {
        Self {
            task_queue,
            waker_cache: BTreeMap::new(),
        }
    }

    fn run_ready_task(&mut self) {
        while let Ok(task) = self.task_queue.recv() {
            let waker = self
                .waker_cache
                .entry(task.task_id())
                .or_insert_with(|| Waker::from(task.clone()));

            let mut context = Context::from_waker(waker);
            match task.poll(&mut context) {
                Poll::Ready(_) => {
                    self.waker_cache.remove(&task.task_id());
                }
                Poll::Pending => {}
            }
        }
    }

    pub fn run(&mut self) {
        self.run_ready_task();
    }
}
```

`new` 方法接收 `Receiver<Arc<Task>>` 参数，然后创建一个执行器实例。

`run_ready_task` 方法中：

* 从 `task_queue` 中接收 `task: Arc<Task>`，然后从 `waker_cache` 中查找是否存在对应的 `waker`，如果没有则构造一个 `Waker`。

  

* 使用 `Context` 的 `from_waker` 方法通过 `waker` 的引用创建 `context`。

  

* 调用 `task` 的 `poll` 方法，传入 `&mut context` 参数，开始执行`task`。

  * 如果返回的是 `Poll::Ready`，说明 `task` 执行完毕，从 `waker_cache` 中删除缓存的 `waker`。
  * 如果返回的是 `Poll::Pending`，则什么都不做（最终执行的 `Leaf-Future` 中会注册等待的事件和 `waker`）。



## Spawner

在初始状态下，`executor` 的执行队列中是空的，我们需要一种机制能够让用户手动地创建 `task` 并将 `task` 发送到 `executor` 的执行队列中，最后开启 `executor` 的执行。`Spawner` 抽象便提供了这种机制：

```rust,noplayground
#[derive(Clone)]
pub struct Spawner {
    task_sender: Sender<Arc<Task>>,
}
```

`Spawner` 中的 `task_sender` 和 `Task` 的 `task_sender` 一样，都是为了把 `task` 发送到 `executor` 的执行队列中。



### 方法实现

```rust,noplayground
impl Spawner {
    fn new(task_sender: Sender<Arc<Task>>) -> Self {
        Self { task_sender }
    }

    pub fn spawn(&self, future: impl Future<Output = ()> + 'static + Send) {
        let task = Task::new(future, self.task_sender.clone());
        self.task_sender
            .send(Arc::new(task))
            .expect("send task failed");
    }
}
```

`new` 方法接收`Sender<Arc<Task>>` 参数，然后创建一个 `Spawner` 实例。

`spawn` 方法中，使用传入的  `future`  参数创建一个 `Task` 实例，然后把这个 `task` 发送到 `executor` 的执行队列中。当 `executor` 开始执行的时候就可以从队列中接收 `task`，驱动 `task` 的执行了。



## 创建 Spawner & Executor

定义一个公开的函数，创建 `Spawner` 和 `Executor` 实例：

```rust,noplayground
pub fn spawner_and_executor() -> (Spawner, Executor) {
    let (task_sender, task_queue) = bounded(10000);
    let spawner = Spawner::new(task_sender);
    let executor = Executor::new(task_queue);
    (spawner, executor)
}
```

在 `spawner_and_executor` 函数中，我们使用 `crossbeam-channel` 提供的 `unbounded` 函数创建一个容量为 10000 的管道，分别返回管道的发送端和接收端，然后创建 `Spawner` 和 `Executor` 实例并返回。
