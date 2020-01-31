 - title = 100 行代码实现 Rust stack-less 的协程库
 - url = a-stack-less-rust-coroutine-100-loc
 - tags = 
 - datetime = 2020-01-31 19:15:34 +0800
 - origin-url = https://blog.aloni.org/posts/a-stack-less-rust-coroutine-100-loc/


[Rust](https://www.rust-lang.org/) 1.39.0 稳定之后，基于 [`async/await` 语法](https://www.infoq.com/presentations/rust-2019/)可以用低于 100 行代码实现一个非常基础和安全的[协程](https://en.wikipedia.org/wiki/Coroutine)库。该实现是完全基于 `std` 的，同时也是 stack-less 的（换句话说是不依赖于独立的 CPU 栈）。


该基础的协程库包含了最基础的、无事件触发的 `yield` 实现，它可以中断当前协程的执行来让其他协程执行。我用了一种最简洁的例子来演示携程库的实现。

## Yielder
我们用一个只含有简单二进制状态的 `Fib` 结构体来模拟协程。该结构体 `Fib` 有一个 `waiter` 方法，该方法用来创建一个等待(awaited)状态的 [`Future`](https://doc.rust-lang.org/std/future/trait.Future.html) 以便协程调用。

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Poll, Context};

enum State {
    Halted,
    Running,
}

struct Fib {
    state: State,
}

impl Fib {
    fn waiter<'a>(&'a mut self) -> Waiter<'a> {
        Waiter { fib: self }
    }
}

struct Waiter<'a> {
    fib: &'a mut Fib,
}

impl<'a> Future for Waiter<'a> {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, _cx: &mut Context) -> Poll<Self::Output> {
        match self.fib.state {
            State::Halted => {
                self.fib.state = State::Running;
                Poll::Ready(())
            }
            State::Running => {
                self.fib.state = State::Halted;
                Poll::Pending
            }
        }
    }
}
```

## Executor
执行器(Executor) 用 `Vec` 来储存未完成的 Future，每一个 Future 的状态都储存在堆上。作为最简单的实现，它仅支持在运行前添加 Future，而不能在执行时或者之后添加。 `push` 方法用来添加一个新的 Future 到队列中， `run`方法逐个执行队列中的 Future，直到所有都完成。

```rust
use std::collections::VecDeque;

struct Executor {
    fibs: VecDeque<Pin<Box<dyn Future<Output=()>>>>,
}

impl Executor {
    fn new() -> Self {
        Executor {
            fibs: VecDeque::new(),
        }
    }

    fn push<C, F>(&mut self, closure: C)
    where
        F: Future<Output=()> + 'static,
        C: FnOnce(Fib) -> F,
    {
        let fib = Fib { state: State::Running };
        self.fibs.push_back(Box::pin(closure(fib)));
    }

    fn run(&mut self) {
        let waker = waker::create();
        let mut context = Context::from_waker(&waker);

        while let Some(mut fib) = self.fibs.pop_front() {
            match fib.as_mut().poll(&mut context) {
                Poll::Pending => {
                    self.fibs.push_back(fib);
                },
                Poll::Ready(()) => {},
            }
        }
    }
}
```

## Null Waker
对于上面的 Executor 实现，我们还需要一个类似 [genawaiter](https://github.com/whatisaphone/genawaiter)([代码链接](https://github.com/whatisaphone/genawaiter/blob/master/src/waker.rs)) 中用的 Null Waker。
 > 译者： 对于 Null Waker 这个词就不错翻译了，对于 Waker 的概念，在 Rust 中就是用于把「该 Future 可以从 await 状态被激活了」这个消息发送给 Executor 以便 Executor 执行 Future 的模块，即称为 Waker。简而言之便是唤醒 await 状态中的 Future。

```rust
use std::task::{RawWaker, RawWakerVTable, Waker},

pub fn create() -> Waker {
    // Safety: The waker points to a vtable with functions that do nothing. Doing
    // nothing is memory-safe.
    unsafe { Waker::from_raw(RAW_WAKER) }
}

const RAW_WAKER: RawWaker = RawWaker::new(std::ptr::null(), &VTABLE);
const VTABLE: RawWakerVTable = RawWakerVTable::new(clone, wake, wake_by_ref, drop);

unsafe fn clone(_: *const ()) -> RawWaker { RAW_WAKER }
unsafe fn wake(_: *const ()) { }
unsafe fn wake_by_ref(_: *const ()) { }
unsafe fn drop(_: *const ()) { }
```

## 跑起来看看
我们可以通过下面的代码来测试这个库：
```rust
pub fn main() {
    let mut exec = Executor::new();

    for instance in 1..=3 {
        exec.push(move |mut fib| async move {
            println!("{} A", instance);
            fib.waiter().await;
            println!("{} B", instance);
            fib.waiter().await;
            println!("{} C", instance);
            fib.waiter().await;
            println!("{} D", instance);
        });
    }

    println!("Running");
    exec.run();
    println!("Done");
}
```
输出：
```rust
Running
1 A
2 A
3 A
1 B
2 B
3 B
1 C
2 C
3 C
1 D
2 D
3 D
Done
```


## 性能表现
在 `lto=true` 的编译条件和 Intel i7-7820X CPU 执行条件下，内部循环每一轮循环大概花费了 5 纳秒。
```rust
pub fn bench() {
    let mut exec = Executor::new();

    for _ in 1..=2 {
        exec.push(move |mut fib| async move {
            for _ in 0..100_000_000 {
                fib.waiter().await;
            }
        });
    }

    println!("Running");
    exec.run();
    println!("Done");
}
```

## 最后
Rust `async/await` 的一个优点就是它不依赖于任何特定的运行时，因此如果你不满足于当前的任何运行时，你可以自由地实现自己的 Executor。

运行时的独立性也有其缺点。例如，我们在上文中编写的库并不兼容 `async-std` 这样的运行时。实际上，这份实现违背了 `Future` 的 `poll` 设计原理，因为我们假定 Future 处于 `Pending` 状态之后就始终处于 `Ready` 状态。

Combined uses of several run-times in a single program is possible but requires extra care (see Reddit discussion).
在一个程序中共同使用多个运行时是科兴的，但是需要**格外小心**（详情见 [Reddit 讨论](https://www.reddit.com/r/rust/comments/eagjyf/using_libraries_depending_on_different_async/)）