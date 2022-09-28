# async/await 的使用

`async/await `是 Rust 中特殊的语法，它使得让出当前线程的控制权而不阻塞线程成为可能，从而允许在等待一个操作完成时可以运行其他代码。

有两种主要的方式使用 `async`：`async fn` 和 `async {}`。这两中使用方式都会返回一个实现了 `Future trait` 的值：

```rust,noplayground
// `foo()` 返回一个实现了 `Future<Output = u8>` 的类型。
// `foo().await` 将会产生一个 u8 类型的值。
async fn foo() -> u8 { 5 }

fn bar() -> impl Future<Output = u8> {
    // 这个 `async` 块会产生一个实现了 `Future<Output = u8>` 的类型。
    async {
        let x: u8 = foo().await;
        x + 5
    }
}

```

`async fn` 和 `async {}` 返回的 `Future` 是惰性的：在真正开始运行之前它什么也不会做。运行一个 `Future` 的最普遍的方式是 `await` 这个 `Future`： `Future.await`。

当 `await` 一个 `Future` 时，会暂停当前函数的运行，直到完成对 `Future` 的运行。如果这个 `Future` 被阻塞住了（例如等待网络IO），它会让出当前线程的控制权。当 `Future` 中的阻塞操作就绪时（例如等待的网络IO返回了响应），`executor` 会通过 `poll` 会恢复 `Future` 的运行。



## async lifetime

与普通的函数不一样，`async fn` 会获取引用或其他非静态生命周期的参数，然后返回被这些参数的生命周期约束的 `Future`：

```rust,noplayground
async fn foo(x: &u8) -> u8 { *x }

// 这与上面的函数完全等价
fn foo_expanded<'a>(x: &'a u8) -> impl Future<Output = u8> + 'a {
    async move { *x }
}
```

这意味着，`async fn` 返回的 `Future` 必须在非静态生命周期参数仍然有效时 `.await`。在大多数情况下，我们在调用 `async` 函数后会立马 `.await`（例如 `foo(&x).await`），因此 `async lifetime` 不会对执行产生什么影响。但是，如果我们存储这种 `Future` 或者发送给其他的 `task` 或者 `thread`，就可能会造成问题。

把带有引用参数的 `async fn` 转化为静态 `Future` 的解决方法是：把参数和对 `async fn` 的调用封装到 `async` 块中：

```rust,noplayground
fn bad() -> impl Future<Output = u8> {
    let x = 5;
    borrow_x(&x) // ERROR: `x` does not live long enough
}

fn good() -> impl Future<Output = u8> {
    async {
        let x = 5;
        borrow_x(&x).await
    }
}
```

通过把参数移动到 `async` 块中，我们把它的生命周期扩展到了匹配调用 `good` 返回的 `Future` 的生命周期。



## async move

`async` 块和闭包允许像普通闭包那样使用 `move` 关键字。一个 `async move` 块会获取变量的所有权，但是这会导致无法与其他的代码共享这些变量：

```rust,noplayground
// 不同的 async 块可以访问相同的变量s，只要它们都在s的作用域范围内执行
async fn blocks() {
    let s = String::new("Hello World");
    let future_one = async {
        println!("{:?}", s);
    };
    let future_two = async {
        println!("{:?}", s);
    };
    
    executor::join(future_one, future_two);
}

// s 被 move 进行 async 块中，因此只能在该 async 块内才能访问 
fn move_block() -> impl Future<Output = ()> {
    let s = String::from("Hello World");
    async move {
        println!("{:?}", s);
    }
}
```



