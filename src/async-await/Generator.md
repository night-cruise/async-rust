# Generator

`Future` 的底层依赖于生成器，因此在本节中我们将会介绍生成器的概念，以及生成器是如何转化为 `Future` 的。



## Generator 定义

`Generator` 的定义位于标准库的 `ops` 模块中，具体如下所示：

```rust,noplayground
pub trait Generator<R = ()> {
    type Yield;
    type Return;
    fn resume(
        self: Pin<&mut Self>, 
        arg: R
    ) -> GeneratorState<Self::Yield, Self::Return>;
}

pub enum GeneratorState<Y, R> {
    Yielded(Y),
    Complete(R),
}
```

`Generator` 通常也被称为协程，主要目的是为 `async/await` 语法提供构建块，但是未来也可能会扩展到为 `Iterator` 和其他类型提供符合人体工程学的定义。

`Generator` 的关联类型 `Yield` 对应于使用`yield` 表达式产出的值的类型。

`Generator` 的关联类型 `Return` 对应于使用 `return` 语句或者生成器中的最后一个表达式返回的值的类型。

>注意：`Generator trait` 中涉及到的 `Pin` 将会在后面的章节中介绍。



## resume

调用 `Generator` 的 `resume` 方法会恢复生成器的运行，如果还没有启动生成器的话则会启动生成器。

在执行生成器的过程中，如果遇到 `yield` 表达式，那么生成器就会在这个 `yield` 点挂起，并产出 `yield` 表达式的值：`GeneratorState::Yielded(Y)`。当再次调用 `resume` 方法时生成器就会在挂起的 `yield` 点恢复运行。

在运行过程中，如果遇到的是 `return` 语句或者生成器末尾的最后一个表达式，那么生成器执行完毕，并返回 `GeneratorState::Complete(R)`，`R` 就是 `return` 语句或者末尾表达式的值。

如果生成器已经执行完毕，返回了 `GeneratorState::Complete`，那么当再次调用 `Generator` 的 `resume` 方法时将会导致 `panic`。



## Generator 使用

在闭包中使用 `yield` 关键字就可以创建一个生成器：

```rust
#![feature(generators, generator_trait)]

use std::pin::Pin;
use std::ops::{Generator, GeneratorState};


fn main() {
    let mut gen = || {
        for i in 0..10 {
            yield i;
        }
        
        return ();
    };
    
    loop {
        match Pin::new(&mut gen).resume(()) {
            GeneratorState::Yielded(y) => println!("Yielded: {}", y),
            GeneratorState::Complete(r) => {
                println!("Complete: {:?}", r);
                break;
            }
        }
    }
}
```

通过为自定义类型实现 `Generator trait` 来创建生成器：

```rust
#![feature(generators, generator_trait)]

use std::pin::Pin;
use std::ops::{Generator, GeneratorState};


fn main() {
    let mut gen = MyGenerator { counter: 1, completed: false };
    
    loop {
        match Pin::new(&mut gen).resume(()) {
            GeneratorState::Yielded(y) => println!("Yielded: {}", y),
            GeneratorState::Complete(r) => {
                println!("Complete: {}", r);
                break;
            }
        }
    }
}


struct MyGenerator {
    counter: i32,
    completed: bool
}


impl<R> Generator<R> for MyGenerator {
    type Yield = i32;
    type Return = char;
    
    fn resume(self: Pin<&mut Self>, _arg: R) -> GeneratorState<Self::Yield, Self::Return> {
        if self.completed {
            panic!("MyGenerator has been completed.");
        }
        
        let counter = self.counter;
        if counter < 10 {
            self.get_mut().counter = counter + 1;
            GeneratorState::Yielded(counter)
        } else {
            self.get_mut().completed = true;
            GeneratorState::Complete('🎉')
        }
    }
}
```

把生成器当作迭代器使用：

```rust
#![feature(generators, generator_trait)]

use std::pin::Pin;
use std::iter::Iterator;
use std::ops::{Generator, GeneratorState};


fn main() {
    let gen = MyGenerator { counter: 0, completed: false };
    
    for val in gen {
        println!("Got: {}", val);
    }

}


struct MyGenerator {
    counter: i32,
    completed: bool
}


impl<R> Generator<R> for MyGenerator {
    type Yield = i32;
    type Return = ();
    
    fn resume(self: Pin<&mut Self>, _arg: R) -> GeneratorState<Self::Yield, Self::Return> {
        if self.completed {
            panic!("MyGenerator has been completed.");
        }
        
        let counter = self.counter;
        if counter < 10 {
            self.get_mut().counter = counter + 1;
            GeneratorState::Yielded(counter)
        } else {
            self.get_mut().completed = true;
            GeneratorState::Complete(())
        }
    }
}

impl Iterator for MyGenerator {
    type Item = i32;
    
    fn next(&mut self) -> Option<Self::Item> {
        match Pin::new(self).resume(()) {
            GeneratorState::Yielded(y) => Some(y),
            GeneratorState::Complete(_) => None
        }
    }
}
```



## From Generator to Future

Rust 的 `core` 库中的 `future` 模块定义了将生成器转化为 `Future` 的函数（为了便于阅读去掉了注释部分）：

```rust,noplayground
pub const fn from_generator<T>(gen: T) -> impl Future<Output = T::Return>
	where T: Generator<ResumeTy, Yield = ()>
{
    struct GenFuture<T: Generator<ResumeTy, Yield = ()>>(T);

    impl<T: Generator<ResumeTy, Yield = ()>> !Unpin for GenFuture<T> {}

    impl<T: Generator<ResumeTy, Yield = ()>> Future for GenFuture<T> {
        type Output = T::Return;
        
        fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
            let gen = unsafe { Pin::map_unchecked_mut(self, |s| &mut s.0) };
   
            match gen.resume(ResumeTy(NonNull::from(cx).cast::<Context<'static>>())) {
                GeneratorState::Yielded(()) => Poll::Pending,
                GeneratorState::Complete(x) => Poll::Ready(x),
            }
        }
    }

    GenFuture(gen)
}
```

从源码中可以看出，实际上我们使用 `async` 创建的 `Future` 是一个实现了 `Future trait` 的结构体 `GenFuture`，这个结构体的内部是一个生成器。

在我们调用 `Future` 的 `poll` 方法时，实际上就是在调用底层的生成器的 `resume` 方法，并且生成器返回的 `GeneratorState::Yielded/Complete(val)` 会被分别转化为 `poll` 的返回类型：`Poll::Pending/Ready(val)`。



## 小栗子

在本节的最后，我们通过一个小栗子把前面讲的 `async/await`、`Future`、`Generator` 的知识串联起来。

有如下的代码：

```rust
#[inline(never)]
async fn foo() -> i32 {
    10
}

#[inline(never)]
async fn bar() -> i32 {
    foo().await
}
```

[HIR](https://rustc-dev-guide.rust-lang.org/hir.html) 是 Rust 代码编译的中间产物，可以帮助我们直到代码在脱糖后是什么样子。可以使用 Rust `Playground` 的 `HIR` 功能编译上述代码，结果如下：

```rust,noplayground
#[inline(never)]
async fn foo()
    ->
        /*impl Trait*/ #[lang = "from_generator"](move |mut _task_context|
        { { let _t = { 10 }; _t } })

#[inline(never)]
async fn bar()
    ->
        /*impl Trait*/ #[lang = "from_generator"](move |mut _task_context|
        {
                {
                        let _t =
                            {
                                    match #[lang = "into_future"](foo()) {
                                            mut pinned =>
                                                loop {
                                                        match unsafe {
                                                                            #[lang = "poll"](#[lang = "new_unchecked"](&mut pinned),
                                                                                #[lang = "get_context"](_task_context))
                                                                        } {
                                                                #[lang = "Ready"] { 0: result } => break result,
                                                                #[lang = "Pending"] {} => { }
                                                            }
                                                        _task_context = (yield ());
                                                    },
                                        }
                                };
                        _t
                    }
            })
```

原生的 `HIR` 代码难以阅读，我们将其转化为下面的 Rust 伪代码：

```rust,noplayground
#[inline(never)]
async fn foo() -> impl Future<Output = i32> {
    from_generator(move |mut _task_context| {
        let _t = 10;
        _t
    })
}

#[inline(never)]
async fn bar() -> impl Future<Output = i32> {
    from_generator(move |mut _task_context| {
        let _t = {
            match into_future(foo()) {
                mut pinned => {
                    loop {
                        match unsafe Pin::new_unchecked(&mut pinned).poll(get_context(_task_context)) {
                            Poll::Ready(result) => break result,
                            Poll::Pending => {}
                        }
                        _task_context = (yield ());
                    }
                }
            }
        };
        _t
    })
}
```

可以看到 `async` 函数体内的代码被转化成了一个生成器，然后再调用 `from_generator` 函数传入生成器创建一个 `Future` ，这与我们上面介绍的 `from_generator` 函数的功能一致。

`await` 部分则被转化为了一个无限循环，在循环的内部会调用 `await` 的 `Future` 的 `poll` 方法，如果结果是 `Poll::Ready`，则终止循环并返回 `result`，继续执行剩余的代码；如果结果是 `Poll::Pending`，则会使用 `yield` 挂起生成器，将控制权转移给调用方。当调用方激活这个挂起的生成器时，生成器就会恢复运行，执行循环体中的代码。

因此，只有当 `await` 的 `Future` 执行完毕时，才会继续往下执行 `async` 块中的代码，这样就确保了能够以同步的方式编写异步代码，让我们能拥有良好的开发体验。
