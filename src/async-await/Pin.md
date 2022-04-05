# Pin

前文的 `Future trait`、`Geneartor` 和状态机中都出现了 `Pin`，那么 `Pin` 到底有什么用呢？ 在本节中，我们将会详细地介绍它。



## 自引用结构

在 Safe Rust 中，我们无法创建自引用结构体：

```rust,compile_fail
fn main() {
    let s = "Hello World".to_string();
    let _ = SelfReference {
        a: s,
        b: &s
    };
}

struct SelfReference<'a> {
	a: String,
	b: &'a String
}
```

如果编译，将会发生报错：

```rust,noplayground
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:5:12
  |
2 |     let s = "Hello World".to_string();
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
3 |     let _ = SelfReference {
4 |         a: s,
  |            - value moved here
5 |         b: &s
  |            ^^ value borrowed here after move
```

这是因为 `s` 已经发生了 `move`，因此 `b` 就不能借用已经 `move` 了的 `s`。

为了创建自引用结构，我们需要使用裸指针：

```rust
fn main() {
    let mut sr_1 = SelfReference::new("Hello");
    sr_1.init();
    
    let mut sr_2 = SelfReference::new("World");
    sr_2.init();
    
    println!("sr_1: {{ a: {}, b: {} }}", sr_1.get_a(), sr_1.get_b());
    println!("sr_2: {{ a: {}, b: {} }}", sr_2.get_a(), sr_2.get_b());
}

#[derive(Debug)]
struct SelfReference {
	a: String,
	b: *const String
}

impl SelfReference {
    fn new(msg: &str) -> Self {
        Self {
            a: msg.to_string(),
            b: std::ptr::null()
        }
    }
    
    fn init(&mut self) {
        let ptr_to_a = &self.a as *const _;
        self.b = ptr_to_a;
    }
    
    fn get_a(&self) -> &str {
        &self.a
    }
    
    fn get_b(&self) -> &str {
        unsafe {
            &*self.b
        }
    }
}
```

编译运行，结果如下所示：

```rust,noplayground
sr_1: { a: Hello, b: Hello }
sr_2: { a: World, b: World }
```

接下来，让我们交换 `sr_1` 和 `sr_2` 的内存位置的数据，即 `sr_1` 和 `sr_2` 互相 `move` 给对方：

```rust
fn main() {
    let mut sr_1 = SelfReference::new("Hello");
    sr_1.init();
    
    let mut sr_2 = SelfReference::new("World");
    sr_2.init();
    
    println!("Before swap:");
    println!("sr_1: {{ a: {}, b: {} }}", sr_1.get_a(), sr_1.get_b());
    println!("sr_2: {{ a: {}, b: {} }}", sr_2.get_a(), sr_2.get_b());
    
    std::mem::swap(&mut sr_1, &mut sr_2);
    println!("\nAfter swap:");
    println!("sr_1: {{ a: {}, b: {} }}", sr_1.get_a(), sr_1.get_b());
    println!("sr_2: {{ a: {}, b: {} }}", sr_2.get_a(), sr_2.get_b());
}

# #[derive(Debug)]
# struct SelfReference {
# 	 a: String,
#	 b: *const String
# }
# impl SelfReference {
#    fn new(msg: &str) -> Self {
#        Self {
#            a: msg.to_string(),
#            b: std::ptr::null()
#        }
#    } 
#    fn init(&mut self) {
#        let ptr_to_a = &self.a as *const _;
#        self.b = ptr_to_a;
#    }   
#    fn get_a(&self) -> &str {
#        &self.a
#    }    
#    fn get_b(&self) -> &str {
#        unsafe {
#            &*self.b
#        }
#    }
# }
```

编译运行，结果如下所示：

```rust,noplayground
Before swap:
sr_1: { a: Hello, b: Hello }
sr_2: { a: World, b: World }

After swap:
sr_1: { a: World, b: Hello }
sr_2: { a: Hello, b: World }
```

可以看出，在交换 `sr_1` 和 `sr_2`  后，字段 `a` 的数据也发生了交换，但是字段 `b` 的数据没有改变，仍然指向之前的位置，如图所示：

![swap problem](../imgs/swap_problem.jpg)

这意味着，`sr`（`sr_1`、`sr_2`）将不再是自引用结构体，并保存了一个指向其他对象的裸指针。因此，`sr` 的字段 `b` 的生命周期将不再和其结构体本身相关联，我们将难以保证 `sr.b` 指针不会变成悬垂指针。

在上面的例子中，由于使用 `swap` 函数导致出现了我们不想要的结果，在后续的代码中对 `sr` 的使用很可能会出现段错误、UB 等其他类型的错误。



## Let's pin it!

Rust 是一门极为注重内存安全的语言，为了能够安全地使用自引用结构，Rust 发明了 `Pin`。



### Pin

`Pin` 位于 `std` 库的 `pin` 模块中，源代码定义如下所示：

```rust,noplayground
#[stable(feature = "pin", since = "1.33.0")]
#[lang = "pin"]
#[fundamental]
#[repr(transparent)]
#[derive(Copy, Clone)]
pub struct Pin<P> {
    pointer: P,
}

#[stable(feature = "pin", since = "1.33.0")]
impl<P: Deref> Deref for Pin<P> {
    type Target = P::Target;
    fn deref(&self) -> &P::Target {
        Pin::get_ref(Pin::as_ref(self))
    }
}

#[stable(feature = "pin", since = "1.33.0")]
impl<P: DerefMut<Target: Unpin>> DerefMut for Pin<P> {
    fn deref_mut(&mut self) -> &mut P::Target {
        Pin::get_mut(Pin::as_mut(self))
    }
}
```

`Pin` 实现了 `Deref` 和 `DerefMut` `trait`，因此 `Pin` 是一个智能指针。并且 `Pin` 的内部包裹了另一个指针 `P`，因此我们一般使用 `Pin<P<T>>` 的方式来表示一个 `Pin` 结构（`T` 是指针 `P` 指向的类型）。

既然有 `Pin`，那么自然就有 `Unpin`，那么 `Unpin` 是什么呢？`Unpin` 是一个 `auto trait`，编译器会默认为所有的类型实现 `Unpin`，除非这些类型实现了 `!Unpin`。

要想获取 `Pin<P<T>>` 中 `T` 的可变引用 `&mut T`，可以使用 `Pin` 提供的 `get_mut` 方法，这也是 `Pin` 提供的 `api` 中**唯一**可以**安全地**获取 `&mut T` 的方法，其函数签名如下所示：

```rust,noplayground
pub fn get_mut(self) -> &'a mut T
where
    T: Unpin,
```

发现了吗？要想安全地拿到 `&mut T`，`T` 就必须实现 `Unpin`。如果 `T` 实现了 `!Unpin`，那么就不可能安全地拿到 `T` 的可变引用，我们自然也就无法使用 `std::mem::swap(x: &mut T, y: &mut T)` 等类似的函数 `move` `T`，就不会发生前文的例子中出现的未定义行为。

因此，`Pin<P<T>>` 利用 Rust 的类型系统保证：如果 `T` 实现了 `!Unpin`，那么就不可能在 Safe Rust 中获取 `T` 的可变引用。相反，如果 `T` 实现了 `Unpin`，那么 `Pin` 就仅仅是对 `P<T>` 的一层包装，我么可以随意地拿到 `&mut T`。

接下来，我们将会使用 `Pin` 解决上面的那个例子中出现的问题。



### Pin to stack

`Pin` 到栈上是指我们想要 `Pin` 住的值在栈上，使用 `Pin::new_unchecked` 函数把 `&mut T` 包装成 `Pin<&mut T>` 即可：

```rust
#![feature(negative_impls)]
use std::pin::Pin;

fn main() {
    let mut sr_1 = SelfReference::new("Hello");
    let mut sr_1 = unsafe { Pin::new_unchecked(&mut sr_1) };
    sr_1.as_mut().init();
    
    let mut sr_2 = SelfReference::new("World");
    let mut sr_2 = unsafe { Pin::new_unchecked(&mut sr_2) };
    sr_2.as_mut().init();
    
    println!("Before swap:");
    println!("sr_1: {{ a: {}, b: {} }}", sr_1.as_ref().get_a(), sr_1.as_ref().get_b());
    println!("sr_2: {{ a: {}, b: {} }}", sr_2.as_ref().get_a(), sr_2.as_ref().get_b());
    
    println!("If we want to swap:");
    std::mem::swap(sr_1.get_mut(), sr_2.get_mut());
}

#[derive(Debug)]
struct SelfReference {
	a: String,
	b: *const String
}

impl !Unpin for SelfReference {}

impl SelfReference {
    fn new(msg: &str) -> Self {
        Self {
            a: msg.to_string(),
            b: std::ptr::null()
        }
    }
    
    fn init(self: Pin<&mut Self>) {
        let ptr_to_a = &self.a as *const _;
        unsafe {
            self.get_unchecked_mut().b = ptr_to_a;
        }
    }
    
    fn get_a(self: Pin<&Self>) -> &str {
        &self.get_ref().a
    }
    
    fn get_b(self: Pin<&Self>) -> &str {
        unsafe {
            &*self.b
        }
    }
}
```

此时代码将不会通过编译：

```rust,noplayground
error[E0277]: `SelfReference` cannot be unpinned
   --> src/main.rs:18:25
    |
18  |     std::mem::swap(sr_1.get_mut(), sr_2.get_mut());
    |                         ^^^^^^^ the trait `Unpin` is not implemented for `SelfReference`
    |
    = note: consider using `Box::pin`
note: required by a bound in `Pin::<&'a mut T>::get_mut`

error[E0277]: `SelfReference` cannot be unpinned
   --> src/main.rs:18:41
    |
18  |     std::mem::swap(sr_1.get_mut(), sr_2.get_mut());
    |                                         ^^^^^^^ the trait `Unpin` is not implemented for `SelfReference`
    |
    = note: consider using `Box::pin`
note: required by a bound in `Pin::<&'a mut T>::get_mut`
```

这说明当我们把 `&mut SelfReference` `Pin` 到栈上之后，无法通过 `get_mut` 方法拿到 `&mut SelfReference`，那么自然就无法使用 `swap` 函数，在编译阶段就保证了不会出现内存安全问题。

`Pin::new_unchecked` 是一个 `unsafe` 函数，这是因为**需要使用者自己遵守约定**只使用 `Pin` 提供的 `api` 来获取并使用可变引用。

假如使用者提前 `drop` 掉 `Pin`，这样就可以直接获取 `T` 的可变引用，仍然会导致内存安全问题：

```rust
#![feature(negative_impls)]
use std::pin::Pin;

fn main() {
    let mut sr_1 = SelfReference::new("Hello");
    let mut sr_1_pin = unsafe { Pin::new_unchecked(&mut sr_1) };
    sr_1_pin.as_mut().init();
    
    let mut sr_2 = SelfReference::new("World");
    let mut sr_2_pin = unsafe { Pin::new_unchecked(&mut sr_2) };
    sr_2_pin.as_mut().init();
    
    println!("Before swap:");
    println!("sr_1: {{ a: {}, b: {} }}", sr_1_pin.as_ref().get_a(), sr_1_pin.as_ref().get_b());
    println!("sr_2: {{ a: {}, b: {} }}", sr_2_pin.as_ref().get_a(), sr_2_pin.as_ref().get_b());
    
    drop(sr_1_pin);
    drop(sr_2_pin);
    

    println!("\nAfter swap:");
    std::mem::swap(&mut sr_1, &mut sr_2);
    
    let sr_1_pin = unsafe { Pin::new_unchecked(&mut sr_1) };
    let sr_2_pin = unsafe { Pin::new_unchecked(&mut sr_2) };
    println!("sr_1: {{ a: {}, b: {} }}", sr_1_pin.as_ref().get_a(), sr_1_pin.as_ref().get_b());
    println!("sr_2: {{ a: {}, b: {} }}", sr_2_pin.as_ref().get_a(), sr_2_pin.as_ref().get_b());
}
# #[derive(Debug)]
#struct SelfReference {
#	a: String,
#	b: *const String
#}
#impl !Unpin for SelfReference {}
#impl SelfReference {
#    fn new(msg: &str) -> Self {
#        Self {
#            a: msg.to_string(),
#            b: std::ptr::null()
#        }
#    }    
#    fn init(self: Pin<&mut Self>) {
#        let ptr_to_a = &self.a as *const _;
#        unsafe {
#            self.get_unchecked_mut().b = ptr_to_a;
#        }
#    }
#    fn get_a(self: Pin<&Self>) -> &str {
#        &self.get_ref().a
#    }   
#    fn get_b(self: Pin<&Self>) -> &str {
#        unsafe {
#            &*self.b
#        }
#    }
#}    
```

编译运行，将会出现和之前的例子中一样的问题：

```rust,noplayground
Before swap:
sr_1: { a: Hello, b: Hello }
sr_2: { a: World, b: World }

After swap:
sr_1: { a: World, b: Hello }
sr_2: { a: Hello, b: World }
```



### Pin to heap

`Pin` 到堆上是指把我们想要 `Pin` 住的值装箱到堆上面，使用`Box::pin` 函数即可把 `T` 包装成 `Pin<Box<T>>`：

```rust
#![feature(negative_impls)]
use std::pin::Pin;

fn main() {
    let mut sr_1 = SelfReference::new("Hello");
    let mut sr_2 = SelfReference::new("World");
    
    println!("Before swap:");
    println!("sr_1: {{ a: {}, b: {} }}", sr_1.as_ref().get_a(), sr_1.as_ref().get_b());
    println!("sr_2: {{ a: {}, b: {} }}", sr_2.as_ref().get_a(), sr_2.as_ref().get_b());
    
    println!("If we want to swap:");
    std::mem::swap(sr_1.as_mut().get_mut(), sr_2.as_mut().get_mut());
    
}

#[derive(Debug)]
struct SelfReference {
	a: String,
	b: *const String
}

impl !Unpin for SelfReference {}

impl SelfReference {
    fn new(msg: &str) -> Pin<Box<Self>> {
        let sr = Self {
            a: msg.to_string(),
            b: std::ptr::null()
        };
        let mut boxed = Box::pin(sr);
        let ptr_to_a = &boxed.a as *const _;
        unsafe {
            boxed.as_mut().get_unchecked_mut().b = ptr_to_a;
        }
        
        boxed
    }
    
    fn get_a(self: Pin<&Self>) -> &str {
        &self.get_ref().a
    }
    
    fn get_b(self: Pin<&Self>) -> &str {
        unsafe {
            &*self.b
        }
    }
}
```

此时代码将不会通过编译：

```rust,noplayground
error[E0277]: `SelfReference` cannot be unpinned
   --> src/main.rs:13:34
    |
13  |     std::mem::swap(sr_1.as_mut().get_mut(), sr_2.as_mut().get_mut());
    |                                  ^^^^^^^ the trait `Unpin` is not implemented for `SelfReference`
    |
    = note: consider using `Box::pin`
note: required by a bound in `Pin::<&'a mut T>::get_mut`

error[E0277]: `SelfReference` cannot be unpinned
   --> src/main.rs:13:59
    |
13  |     std::mem::swap(sr_1.as_mut().get_mut(), sr_2.as_mut().get_mut());
    |                                                           ^^^^^^^ the trait `Unpin` is not implemented for `SelfReference`
    |
    = note: consider using `Box::pin`
note: required by a bound in `Pin::<&'a mut T>::get_mut`
```

`Pin` 到堆上的优点是不需要使用者编写 `unsafe` 函数来构造 `Pin`，也不需要使用者自己遵守约定只使用 `Pin` 提供的 `api` 来获取可变引用，因为 `Pin` 到堆上后，用户只能使用 `Pin<Box<T>>`；缺点是 `Pin` 到堆上会有额外的性能开销。



## Pin and async

在前文中我们给出了 `Future` 和 `Generator` 的定义：

```rust,noplayground
pub trait Future {
    type Output;	
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub trait Generator<R = ()> {
    type Yield;
    type Return;
    fn resume(
        self: Pin<&mut Self>, 
        arg: R
    ) -> GeneratorState<Self::Yield, Self::Return>;
}
```

还有将 `Generator` 转化为 `Future` 的函数：

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

可以看到要调用 `Future` 的 `Poll` 方法和 `Generator` 的 `resume` 方法必须使用 `Pin<&mut Self>` 才行。并且在 `from_generator` 函数中为 `GenFuture` 实现了 `!Unpin`。

经过前面的学习，我们知道为 `T` 实现了 `!Unpin` 后，就无法在 Safe Rust 中获取 `T` 的可变引用，而 Rust 会主动为 `Future` 实现 `!Unpin`，那么为什么 Rust 需要 `Pin` 住 `Future` 呢？

假设我们编写了一个生成器：

```rust
#![feature(generators, generator_trait)]

fn main(){
    let _gen = || {
        let s = "Hello World".to_string();
        let borrowed_s = &s;
        
        yield borrowed_s.len();
        
        println!("{}", borrowed_s);
    };
}
```

编译后将会发生报错：

```rust,noplayground
error[E0626]: borrow may still be in use when generator yields
 --> src/main.rs:6:26
  |
6 |         let borrowed_s = &s;
  |                          ^^
7 |         
8 |         yield borrowed_s.len();
  |         ---------------------- possible yield occurs here

```

编译器提示我们生成器中存在跨 `yield` 借用，那么为什么编译器不允许跨 `yield` 借用呢？

想要知道原因，我们还要继续深入底层，上述的生成器会被编译成一个状态机：

```rust
#![feature(generators, generator_trait)]

use std::pin::Pin;
use std::ops::{Generator, GeneratorState};

fn main() {
    let mut gen = Gen::new();
    
    loop {
        match Pin::new(&mut gen).resume(()) {
            GeneratorState::Yielded(y) => println!("Yielded: {}", y),
            GeneratorState::Complete(c) => {
                println!("Complete: {:?}", c);
                break;
            }
        }
    }
}

enum Gen {
    Enter,
    Yielded{
        s: String,
        borrowed_s: *const String
    },
    Exit
}


impl<R> Generator<R> for Gen {
    type Yield = usize;
    type Return = ();
    
    fn resume(self: Pin<&mut Self>, _arg: R) -> GeneratorState<Self::Yield, Self::Return> {
        let mut_gen = self.get_mut();
        match mut_gen {
            Gen::Enter => {
                let s = "Hello World".to_string();
                let borrowed_s = &s;
                let len = borrowed_s.len();
                
                *mut_gen = Gen::Yielded {
                    s,
                    borrowed_s: std::ptr::null()
                };
                if let Gen::Yielded { s, borrowed_s } = mut_gen {
                    *borrowed_s = s as *const _;
                }
                
                GeneratorState::Yielded(len)
            }
            Gen::Yielded{ borrowed_s, .. } => {
                let borrowed_s: &String = unsafe { &**borrowed_s };
                println!("{}", borrowed_s);
                *mut_gen = Gen::Exit;
                
                GeneratorState::Complete(())
            }
            Gen::Exit => panic!("Generator has been completed.")
        }
    }
}

impl Gen {
    fn new() -> Self {
        Self::Enter
    }
}
```

编译上述代码，结果似乎就是我们所期待的：

```rust,noplayground
Yielded: 11
Hello World
Complete: ()
```

从上述的代码中可以看出，**生成的状态机中存在自引用结构**。因此如果生成器中存在跨 `yield` 点借用，那么就可能产生内存安全问题，编译器干脆就禁止存在跨 `yield` 点借用的生成器通过编译。

例如，如果我们使用 `swap` 函数 `move` 生成器就可能发生异常：

```rust
#![feature(generators, generator_trait)]

use std::pin::Pin;
use std::ops::{Generator, GeneratorState};

fn main() {
    let mut gen_1 = Gen::new();
    let mut gen_2 = Gen::new();
    
    match Pin::new(&mut gen_1).resume(()) {
        GeneratorState::Yielded(y) => println!("Yielded: {}", y),
        GeneratorState::Complete(c) => println!("Complete: {:?}", c)
    }
    match Pin::new(&mut gen_2).resume(()) {
        GeneratorState::Yielded(y) => println!("Yielded: {}", y),
        GeneratorState::Complete(c) => println!("Complete: {:?}", c)
    }
    
    std::mem::swap(&mut gen_1, &mut gen_2);
    
    match Pin::new(&mut gen_1).resume(()) {
        GeneratorState::Yielded(y) => println!("Yielded: {}", y),
        GeneratorState::Complete(c) => println!("Complete: {:?}", c)
    }
    match Pin::new(&mut gen_2).resume(()) {
        GeneratorState::Yielded(y) => println!("Yielded: {}", y),
        GeneratorState::Complete(c) => println!("Complete: {:?}", c)
    }
}
#enum Gen {
#    Enter,
#    Yielded{
#        s: String,
#        borrowed_s: *const String
#    },
#    Exit
#}
#impl<R> Generator<R> for Gen {
#    type Yield = usize;
#    type Return = ();    
#    fn resume(self: Pin<&mut Self>, _arg: R) -> GeneratorState<Self::Yield, Self::Return> {
#        let mut_gen = self.get_mut();
#        match mut_gen {
#            Gen::Enter => {
#                let s = "Hello World".to_string();
#                let borrowed_s = &s;
#                let len = borrowed_s.len();
#                
#                *mut_gen = Gen::Yielded {
#                    s,
#                    borrowed_s: std::ptr::null()
#                };
#                if let Gen::Yielded { s, borrowed_s } = mut_gen {
#                    *borrowed_s = s as *const _;
#                }               
#                GeneratorState::Yielded(len)
#            }
#            Gen::Yielded{ borrowed_s, .. } => {
#                let borrowed_s: &String = unsafe { &**borrowed_s };
#                println!("{}", borrowed_s);
#                *mut_gen = Gen::Exit;               
#                GeneratorState::Complete(())
#            }
#            Gen::Exit => panic!("Generator has been completed.")
#        }
#    }
#}
#impl Gen {
#    fn new() -> Self {
#        Self::Enter
#    }
#}
```

编译运行将会发生段错误：

```rust,noplayground
/playground/tools/entrypoint.sh: line 11:    12 Segmentation fault
Yielded: 11
Yielded: 11
Hello World
Complete: ()
```

为了防止 `move` 掉生成器，我们需要为 `Gen` 实现 `!Unpin`：

```rust
#![feature(negative_impls)]
#![feature(generators, generator_trait)]

use std::pin::Pin;
use std::ops::{Generator, GeneratorState};

fn main() {
    let mut gen_1 = Gen::new();
    let mut gen_2 = Gen::new();
    
    let mut boxed_pin_1 = Box::pin(gen_1);
    let mut boxed_pin_2 = Box::pin(gen_2);
    
    match boxed_pin_1.as_mut().resume(()) {
        GeneratorState::Yielded(y) => println!("Yielded: {}", y),
        GeneratorState::Complete(c) => println!("Complete: {:?}", c)
    }
    match boxed_pin_2.as_mut().resume(()) {
        GeneratorState::Yielded(y) => println!("Yielded: {}", y),
        GeneratorState::Complete(c) => println!("Complete: {:?}", c)
    }
    
    std::mem::swap(boxed_pin_1.as_mut().get_mut(), boxed_pin_2.as_mut().get_mut());
}

enum Gen {
    Enter,
    Yielded{
        s: String,
        borrowed_s: *const String
    },
    Exit
}

impl !Unpin for Gen {}

#impl<R> Generator<R> for Gen {
#    type Yield = usize;
#    type Return = ();    
#    fn resume(self: Pin<&mut Self>, _arg: R) -> GeneratorState<Self::Yield, Self::Return> {
#        let mut_gen = unsafe { self.get_unchecked_mut() };
#        match mut_gen {
#            Gen::Enter => {
#                let s = "Hello World".to_string();
#                let borrowed_s = &s;
#                let len = borrowed_s.len();
#                
#                *mut_gen = Gen::Yielded {
#                    s,
#                    borrowed_s: std::ptr::null()
#                };
#                if let Gen::Yielded { s, borrowed_s } = mut_gen {
#                    *borrowed_s = s as *const _;
#                }               
#                GeneratorState::Yielded(len)
#            }
#            Gen::Yielded{ borrowed_s, .. } => {
#                let borrowed_s: &String = unsafe { &**borrowed_s };
#                println!("{}", borrowed_s);
#                *mut_gen = Gen::Exit;               
#                GeneratorState::Complete(())
#            }
#            Gen::Exit => panic!("Generator has been completed.")
#        }
#    }
#}
#impl Gen {
#    fn new() -> Self {
#        Self::Enter
#    }
#}
```

编译修改后的代码将会直接报错：

```rust,noplayground
error[E0277]: `Gen` cannot be unpinned
   --> src/main.rs:23:41
    |
23  |     std::mem::swap(boxed_pin_1.as_mut().get_mut(), boxed_pin_2.as_mut().get_mut());
    |                                         ^^^^^^^ the trait `Unpin` is not implemented for `Gen`
    |
    = note: consider using `Box::pin`
note: required by a bound in `Pin::<&'a mut T>::get_mut`

error[E0277]: `Gen` cannot be unpinned
   --> src/main.rs:23:73
    |
23  |     std::mem::swap(boxed_pin_1.as_mut().get_mut(), boxed_pin_2.as_mut().get_mut());
    |                                                                         ^^^^^^^ the trait `Unpin` is not implemented for `Gen`
    |
    = note: consider using `Box::pin`
note: required by a bound in `Pin::<&'a mut T>::get_mut`
```

通过为生成器实现 `!Unpin`，我们有效的防止了可能会出现的内存安全问题。

但是，我们无法为使用闭包编写的生成器实现 `!Unpin`，那么怎么让我们的初版代码编译通过呢？答案是使用 `static` 关键字标记生成器，这就相当于为我们的生成器实现了 `!Unpin`：

```rust
#![feature(generators, generator_trait)]

use std::ops::{Generator, GeneratorState};


fn main(){
    let gen = static || {
        let s = "Hello World".to_string();
        let borrowed_s = &s;
        
        yield borrowed_s.len();
        
        println!("{}", borrowed_s);
    };
    
    let mut boxed_pin_gen = Box::pin(gen);
    
    loop {
        match boxed_pin_gen.as_mut().resume(()) {
            GeneratorState::Yielded(y) => println!("Yielded: {}", y),
            GeneratorState::Complete(c) => {
                println!("Complete: {:?}", c);
                break;
            }
        }
    }
}
```

编译运行，一切正常：

```rust,noplayground
Yielded: 11
Hello World
Complete: ()
```



### 小总结

`async` 创建的 `Future` 在编译后会生成一个状态机，如果 `async` 代码中存在跨 `await` 借用，那么对应的底层生成器中也会存在跨 `yield` 点借用，最终生成的状态机中就会存在自引用结构，为了避免可能发生的内存安全问题，Rust 自动为 `Future` 实现了 `!Unpin`，并且只能使用 `Pin<&mut Self>` 来调用 `Future` 的 `poll` 方法和 `Generator` 的 `resume` 方法，从而避免了使用者在 Safe Rust 中获取 `Future` 或 `Generator` 的可变引用，最终避免了使用者使用 `swap` 之类的函数 `move` 掉 `Future` 或 `Generator` 而造成的内存安全问题。 



## Pin 总结

官方的 `Async Book` 上给出了关于 `Pin` 的黄金八条：

1. 如果 `T: Unpin`（默认会实现），那么 `Pin<'a, T>` 完全等价于 `&'a mut T`。换言之： `Unpin` 意味着这个类型被移走也没关系，就算已经被固定了，所以 `Pin` 对这样的类型毫无影响。



2. 如果 `T: !Unpin`， 获取已经被固定的 `T` 类型示例的 `&mut T`需要 `unsafe`。

   

3. 标准库中的大部分类型实现 `Unpin`，在 Rust 中遇到的多数普通类型也是一样。但是， `async/await` 生成的 `Future` 是个例外。



4. 你可以在 `nightly` 通过特性标记来给类型添加 `!Unpin` 约束，或者在 `stable` 给你的类型加 `std::marker::PhatomPinned` 字段。



5. 你可以将数据固定到栈上或堆上。



6. 固定 `!Unpin` 对象到栈上需要 `unsafe`



7. 固定 `!Unpin` 对象到堆上不需要` unsafe`，`Box::pin`可以快速完成这种固定。



8. 对于 `T: !Unpin` 的被固定数据，你必须维护好数据内存不会无效的约定，或者叫固定时起直到释放。这是 `Pin` 约定中的重要部分。
