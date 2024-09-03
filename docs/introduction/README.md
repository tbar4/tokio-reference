# Using Tokio
> I am using notes from Tokio's website for future quick reference

Tokio is an asynchronous runtime for the Rust programming language. It is used as the build blocks for many async Rust networking applications. At a high level tokio provides a few major components:

- A multi-threaded runtime for executing asynchronous code
- An asynchronous version of the standard library
- A large ecosystem of libraries

### Why Async?

Async code allows you to write your programs in an asynchronous manner, which allows for greater scaling and lower costs because you no longer have processes blocking threads. Rust needs you to define an async runtime in oder to enable your code to be async. Tokio happens to be the most widely used runtime and is used more than all other Rust async runtimes combined. Tokio works to mirror the standard Rust library where it makes sense. 

### Why Tokio
Tokio boasts 4 major advantages:

- **Fast** - Tokio is built on top of Rust, which means it is very fast. The goal being that you should not be able to improve the performance of equivalent code by writing it by hand. Tokio is also scalable, due to network constraints there is a limit of how fast you can handle a connection due to latency, the alternative is that you can handle more connections at once. Tokio's async/await language features allow you to scale the number of connections
- **Reliable** - Since tokio is built on Rust it is very reliable. 70% of bugs are due to memory safety and Rust eliminates those bugs through how it handles memory. Tokio also aims to provide consistent behavior. 
- **Easy** - With Rust's async/await features, the complexity of writing async apps has been significantly reduced. Tokio also follows standard library naming conventions, which healps with conversion. 
- **Flexible** - Tokio has multiple runtimes, including a `work-stealing` runtime and a `single threaded` runtime.

### When not to use Tokio

- **Speeding up CPU Bound computation** - I learned this one the hard way... Tokio is designed for I/O bound operations, making it very good for managing several connections on a web server trying to get information from a database, but if you are looking to parallelize a complex analytics job (think of a Apache Spark type workload) then you should use [rayon](https://docs.rs/rayon/latest/rayon/), instead. It is possbile to use both, e.g. maybe multiple users are trying to run multiple complex workloads. Tokio could handle the user connections while `rayon` manages the processing. See [this blog post](https://ryhl.io/blog/async-what-is-blocking/#the-rayon-crate) for more info.  
- **Reading a lot of files** - This is mainly due to operating systems not providing async file APIs, but do not confuse this with reading files from object stores like S3 or Minio, which are OS independent and have async interfaces. 
- **Single Threaded Tasks** - Tokio is designed to give you an advantage when you need to do many things at once. If you have a small user pool or do not need async, then Tokio does not give you much of an advantage. 

## What is Async Programming?

Most programs are executed in the order that they are written, so when a program encounters a process in which it takes time to finish, all other operations are *blocked*. Establishing a TCP connection over a network takes time, that means that if a user is trying to establish a connection then all other operations are blocked until it is completed. In async programming, all processes that cannot be completed immediately are suspended in the background. Mara Bos book [Rust Atomics and Locks](https://marabos.nl/atomics/) goes over the more primitive details of Rust concurrecny. Once the other task is complete, the task is unsuspended and continues this work. Consider now that we can easily not only use all the threads that our computers have to offer, but also single threaded async operations. Normally, async programming can be rather complex, you have to keep track of async operatiions and track changes.

### Compile-time Green Threading

Rust implements async programming using a feature called `async/await`. 

> Many other languages use `async/await` but what makes Rust unique is that Rust's async operations are lazy. 

Functions that are async are labeled with the `async` keyword. At compile time Rust transforms async functions into a routine that operates asynchronously. Using the `async/await` keywords doesn't actually execute the function. `await` returns a value representing the operation, which is pretty much a zero-argument closure. The `await` operator actually calls the function. The return value of an `async fn` is an anonymous type that miplements the `Future` trait.

### Async `main` Function

The `async fn` used to launch the application is different from the usual one found in most crates. It is used an asynchronous context because they need to be executed by a runtime, but it doesn't automatically start, so the main function needs to start it. The `#[tokio::main]` function is actually a macro. It actually transforms `async fn main()` into a synchronous `fn main()` that initializes a runtime instance and executes the async main function.

```rust
#[tokio::main]
async fn main() {
  println!("Hello!");
}
```
gets transformed into:

```rust
fn main() {
  let mut rt = tokio::runtime::Runtime::new()::unwrap();
  rt.block_on(async {
    println!("Hello!");
  })
}
```

Tokio has a lot of features, so make sure to optimize compile time and ending application size by only adding the features you need.
