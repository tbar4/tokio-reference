# Channels

We need to be able to have multiple clients access the server at the same time. Since the Client doesn't implement `Copy` we cannot use it to create multiple connections. We also cannot use `Mutex` from the standard lib with `.await` but this would cause blocking of the thread because it will hold a lock. If we use the Tokio `Mutex`, but it would still only utilize a single request. 

## Message Passing
The best solution is to use message passing. This pattern involves spawning a dedicated task to manage the `client` resource. The `client` task issues the request on behalf of the sender and the response is sent back to the sender. Using this strategy a single connection is established. The task managing the `clent` is able to get exclusive access in order to call `get` and `set`. Addtionally, the channel works as a buffer. operations may be sent to the `client` task while the `client` task is busy. Once the `client` task is avaialble it is able to process new requests, it pulls the next request form the channel. This can result in better throughput, and be extended to support connection pooling.

## Tokio's Channel Primatives

Tokio provides a [number of channels](https://docs.rs/tokio/1.40.0/tokio/sync/index.html):

* **mpsc** - multi-producer, single consumer channel. Many values can be sent.
* **oneshot** - single-producer, single consumer. A single Value can be sent.
* **broadcast** - multi-producer, multi-consumer. Many values can be sent. Each receiver sees every value.
* **watch** - multi-producer, multi-consumer. Many values can be sent, but no history is kept. receivers only see the most recent value. 

If you need a multi-producer, multi-consumer channel where only one consumer sees each message use the `async-channel` crate. There are also channels for use outside of asyn Rustm such as `std::sync::mpsc` and `crossbeam::channel`. These channels wait for messages by blocking the thread, which is not allowed in async code. 

## Create the Channel

We are going to use `mpsc` to **send** commandsto the task managing the redis connection. The multi-producer capability allows for messages to be sent from many tasks. Creating the channel will return two values, a sender (usually called `tx`) and a receiver (usually called `rx`). The two handles are used separately and can be send to different tasks. 

We created a channel with a capacity of 32 and if messages are sent faster than they are received the channel will store them. Once the 32 messages are stored in the channel, calling `send(...).await` will go to sleep until a message has been removed by the receiver.

We can send from multiple tasks by `clone()`ing the sender. For example: 

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
  let (tx, &mut rx) = mpsc::channel(32);
  let tx2 = tx.clone();

  // If we do not spawn the threads, tx stays in scope and the program
  // continues to run indefinetly
  tokio::spawn(async move {
    tx.send("sending form the first handle").await.unwrap();
  });

  tokio::spawn(async move {
    tx2.send("sending from the second handle").await.unwrap();
  });

  while let Some(message) = rx.recv().await {
    println!("GOT = {}", message);
  }
}
```
Both messages are sent to a single `Receiver` handle. It is not possible to clone the receiver of a `mpsc` channel. When every `Sender` has gone out of scope or been dropped it is no longer possible to send more messages. At this point, the `recv` call on the `Receiver` will return `None`, which means all of the senders are gone and the channel is closed. 

## Spawn Manager Task

We now need to spawn a task that processes messages from the channel. First, a client connection is established to Redis, then received commands are issued via that redis connection. Then we are going to need to update the two tasks to send commands over the channel instead of issuing them directlhy to redis. 

## Receive Responses

The final task is to receive responses back from the manager task. The `GET`command needs to get hte value and the `SET` command needs to know if the operation completed successfully. To pass the response, a `oneshot` channel is used. The `oneshot` channel is a single-producer, single-consumer channel optimized for sending a single value. In our case the single value is the response. 

Similar to `mpsc` the `oneshot::channel()` returns a sender and a receiver: 
```rust
use tokio::sync::oneshot;

let (tx, rx) = oneshot::channel();
```

Unlike `mpsc`, no capacity is specified as the capacity is always one, and neither handle can be cloned. To receive responses back from the manager task, before sending a command, a `oneshot` channel is created. the `Sender` half of the channel is included in the command to the manager task. The receive half is used to receive the response.
