# Shared State

## Strategies

There are a few ways to share state in Tokio:

1) Guard the shared State with a `Mutex`
2) Spawn a task to manage the state and use message passing to operate on it

Usually you want to use the first approach for simple data, and the second approach for things that require async  work such as I/O primatives. Currently, the shared state is a `HashMap` and the operations are `insert` and `get` and since these operations aren't async, we will use a `Mutex`.

### The `bytes` Dependency

Instead of using `Vec<u8>` isntead we will be using `bytes`. The goal of `bytes` is to provide a robust byte array structure for network programming. The biggest feature it adds over `Vec<u8>` is the shallow cloning. If you were to call a `clone()` on a `Bytes` instance, it does not copy the underlying data, but instead increments a reference counter, making the `Bytes` type essentially an `Arc<Vec<u8>>` with some additional features.

### Initializing the `HashMap`

we will ened to share this `HashMap` across many threads. We need to use an `Arc` (Atomic Reference Counter) to share the state across threads. \

#### On using `std::sync::Mutex` and `tokio::sync::Mutex`

Note that the `std::sync::Mutex` an not `tokio::sync::Mutex` is being used to *guard* the `HashMap`. `tokio::sync::Mutex` is an async `Mutex` that is locked across calls to `.await`. `std::sync::Mutex` is synchronous and will block the current thread while waiting to acquire the lock. This in turn will block other tasks from processing, but switching to an async Mutex normally doesn't help because because internally it uses a synchronous Mutex. 

As a rule of thumb, using a synchronous Mutex from wihtin asynchronous code is fine as long as contention remains low and the lock is not held across calls to `.await`. 

## Holding a `MutexGuard` across an `.await`

You might write code that looks like this:

```rust
use std::sync::MutexGuard;

async fn increment_and_do_other_stuff(mutex: &Mutex<i32>) {
  let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
  *lock += 1;

  do_something_async().await;
} //lock goes out of scope here
```

When trying to spawn something that calls this function you will get the following error:
```rust
error: future cannot be sent between threads safely
   --> src/lib.rs:13:5
    |
13  |     tokio::spawn(async move {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    |
   ::: /playground/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-0.2.21/src/task/spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/lib.rs:7:5
    |
4   |     let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    |         -------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
...
7   |     do_something_async().await;
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ await occurs here, with `mut lock` maybe used later
8   | }
    | - `mut lock` is later dropped here
```

This happens because `std::sync::MutexGuard` type is not `Send`. This means that you can't send a Mutex lock to another thread and the error happens because the Tokio runtime can move a task between threads at every `.await`. This means that you need call the destructor before it runs. 

```rust
use std::sync::MutexGuard

async fn increment_and_do_other_stuff(mutex: &Mutex<i32>) {
  {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1; 
  }//lock goes out of scope here

  do_something_async();
}
```

You should not try to circumvent the issue by spawning away a task that does not require it to be `Send`, because if Tokio suspends our task at an `.await` while the task is holding the lock, some other task may be scheduled to run on the same thread, and that task may aslo try to lock the Mutex, which would result in a deadlock, as the task waiting to lock the mutex would preven the task holding the mutex to release the mutex. 

### Restructure your code to not hold a lock across an `.await`

The safest way to handle a mutex is to wrap it in a struct, and lock the mutex only inside non-async methods on that struct. 

```rust
use std::sync::Mutex

struct CanIncrement {
  mutex: Mutex<i32>,
}

impl CanIcrement {
  // This function is not marked async
  fn increment(&self) {
    let mut lock = self.mutex.lock.unwrap();
    *lock += 1;
  }
}


async fn increment_and_do_stuff(can_incr: &CanIncrement) {
  can_incr.increment();
  do_something_async().await;
}
```

## Tasks, Threads, and Contention

Using a blocking mutex to guard short critical sections is an acceptable strategy when contention is minimal. When a lock is contended, the thread executing the task must block and wait on the mutex. This will not only block the current task, but it will also vlock all other tasks scheduled on the current thread. 

If contention on a synchronous mutex becomes a problem, the best fix is rarely to switch to the Tokio Mutex. Instead, the options to consider are:

* Let a dedicated task manage state and use message passing
* Shard the mutex
* Restructure the code to avoid the mutex

### Mutex Sharding 

In our use case, each key is independent, mutex sharding will work well. To do this, instead of having a single `Mutex<HashMap<_, _>>` instance, we would introduce `N` distinct instances.

```rust
type ShardedDb = Arc<Vec<Mutex<HashMap<String, Vec<u8>>>>>;

fn new_sharded_db(num_shards: usize) -> ShardedDb {
  let mut db = Vec::with_capacity(num_shards);
  for _ in 0..num_shards {
    db.push(Mutex::new(HashMap::new()));
  }
  Arc::new(db)
}
```

Then, finding a cell for any given key becomes a two-step process. First, the key is used to identify which shard it is a part of. Then, the key is looked up in the `HashMap`.

```rust
let shard = db[hash(key) % db.len()].lock().unwrap();
shard.insert(key, value);
```

The `dashmap` crate provides a implementation of a sharded `HashMap` that is a little more sophisticated. There are also crates for concurrent hash table implementations called `leapfrog` and `flurry` while the latter is a part of Java's `ConcurrentHashMap` data structure.
