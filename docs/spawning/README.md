# Spawning

The redis server needs to accept sockets and we can do this by using `tokio::net::TcpListener` to port `6379`. The sockets are processed in a loop and then closed. 

## Concurrency

It is good to point out that concurrency and parallelism are not the same thing. Concurrency is like a single chef swithcing between cooking multiple dishes, while he is waitng for his pasta to cook, he may switch to cutting vegetables. Parallelism is like multiple chefs each working on a separate task. You can use concurrency and parallelism at the same time to increase throughput.

If we want to process each task concurrently, we need to spawn a new task for each connection. 

### Tasks
A Tokio `task` is an asynchronous green thread and are created by passing an `async` block to `tokio::spawn`. `tokio::spawn` returns a `JoinHandle` which the caller may use to interact with the joined task. The `async` block may have a return value, which tha caller can get by calling `await` on the `JoinHandle`. For example:

```rust
#[tokio::main]
async fn main() {
  let handle = tokio::spawn(async {
    // Do some async work
    "return value"
  });

  // Do some other work

  let out = handle.await.unwrap();
  println!("GOT {out}");
}
```

`JoinHandle` returns a `Result`, which means it returns an `Err` whena panic happens or a task is forcefully cancelled. Tasks are the unit of execution managed by the scheduler. Spawning the task submits to the Tokio scheduler which then ensures the task executes when there is work to do. Spawned tasks may execute in the same thread or a different thread. The task can be moved between threads after being spawned. 

Tasks in Tokio very lightweight, only a single allocation and 64 bytes of memory are needed. Applications can spawn millions of tasks. 

### `'static Bound`

When you spawn a task on the Tokio runtime its types lifetime must be `'static`, which means that the spawned task must not contain any references to data owned outside of the task. 

For example the following will not compile:

```rust
use tokio::task

#[tokio::main]
async fn main() {
  let v = vec![1, 2, 3];

  task::spawn( async {
    println!("Here's a vec {:?}", v);
  })
}
```

The spawned thread has a `'static` lifetime which may outlive the vec `v`, which will throw a lifetime error. The `println!` function *borrows* the `v`, but its lifetime is `'a`. This is because the variables are not *moved* into the thread. Once the `v` is moved into the spawned thread it gets a `'static` lifetime. It is interesting that the error mentions that argument *type* needs to outlive static. Keep in mind the *type* needs to out live the `'static` and not the *value*.

The reason the spawned thread gets a `'static` is because the compiler has no way to reason how long the thread would live. 

If a variable needs to be accessed by more than one task concurrently, it needs to use an `Arc`.  

### `Send` Bound

Tasks spawned by `tokio::spawn` must implement `Send`. This allows to move the tasks between threads while the are suspended at an `.await`. 

Tasks are `Send` when all data that is held across `.await` calls is `Send`. When an `.await` is called, the task yields back to the scheduler. The next time the task is executedm it resumes from the point it last yielded. To make this work, all state that is used after `.await` must be saved by the task. If this state is `Send`, i.e., can be moved across threads, then the task itself can be moved across threads. For example:

```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
  tokio::spawn(async {
    // The scope forces `rc` to drop before `.await`
    {
      let rc = Rc::new("hello");
      println!("{}", rc);
    }
  })

  // `rc` is no longer used. It is not persisted when 
  // the task yields to the scheduler
  yield_now().await;
}
```

But this fails:

```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
  tokio::spawn(async {
    let rc = Rc::new("hello");

    // `rc` is used after `await` It must be persisted
    // to the task state
    yield_now().await;
  });
}
```

## Store Values

We need to implement a process to store values. We can use `SET` to store values and `GET` to get them. We can now get and set values, but the problem is they are not shared between connections. If another socket tries to `GET` the `hello` key, it will not be found.
