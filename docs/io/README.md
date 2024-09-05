# I/O

I/O in Tokio operates in much the same wasy as in `std` but asynchronously. there is a trait for reading (`AsyncRead`) and a trait for writing (`AsyncWrite`). Specific types implement these traits as appropriate(`TcpStream`, `File`, `Stdout`). `AysncRead` and `AsyncWrite` are also implemented by a number of datastructures, such as `Vec<u8>` and `&[u8]`. This allows using byte arrays where a reader or writer is expected. 

## `AsyncRead` and `AsyncWrite`

These two traits provide the facilities to async read and write form, byte streams. The methods on these traits are typically not called directly, similar to how you don't manually call the `poll` method from the `Future` trait. Instead, you will use them through the utility methods provided by `AsyncReadExt` and `AsyncWriteExt`. Let's briefly look at a few of these methods.

### `async fn read()`

`AsyncReadExt::read` provides an async method for reading data into a buffer, returning the number of bytes read. 

> Note: when `read()` returns `Ok(0)` that signifies the stream is closed. any further calls to `read()` will complete immediately with `Ok(0)`. With `TcpStream` instances, this signifies that the read half of the socket is closed. 

```rust
use tokio::fs::File;
use tokio::io::{self, AsyncReadExt};

#[tokio::main]
async fn main() -> io::Result<()> {
  let mut f = File::open("foo.txt").await?;
  let mut buffer = [0; 10];

  // read up to 10 bytes
  let n = f.read(&mut buffer[..]).await?;

  println!("The bytes: {:?}", &buffer[..n]);
  Ok(())
}
```

### `async fn read_to_end()`

`AsyncReadExt::read_to_end` reads all bytes from the stream until EOF.

```rust
use tokio::io::{self, AsyncWriteExt};
use tokio::fs::file;

#[tokio::main]
async fn main() -> io::Result<()> {
  let mut f = File::open("foo.txt").await?;
  let mut buffer = Vec::new();

  // read the whole file
  f.read_to_end(&mut buffer).await?;
  Ok(())
}
```

### `asnc fn write()`
`AsyncWriteExt::write` writes a buffer into the writer, returning how many bytes were written. 

```rust
use tokio::io::{self, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
  let mut file = File::create("foo.txt").await?;

  // Writes some prefix of the byte string, but not necessarily all of it
  let n = file.write(b"some "bytes").await?;

  println!("Wrote the first {} bytes of 'some bytes'.", n);
  Ok(())
}
```

### `async fn write_all()`
`AsyncWriteExt::write_all` writes the entire buffer into the writer.

```rust
use tokio::io::{self, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
  let mut file = File::create("foo.txt").await?;

  file.write_all(b"some bytes").await?;
  Ok(())
}
```

Both traits include a number of other helper methods.

## Helper Functions

Additionally, just like `std`, the `tokio::io` module contains a number of helpful utility functions as well as APIs for working with standard input, standard output and standard error. For example, `tokio::io::copy` asynchronously copies the entire contents of a reader into a writer.

```rust
use tokio::fs::File;
use tokio::io;

#[tokio::main]
async fn main() -> io::Result<()> {
  let mut reader: &[u8] = b"hello";
  let mut file = File::create("foo.txt").await?;

  io::copy(&mut reader, &mut file).await?;
  Ok(())
}
```

## Echo Server

Let's practice some async I/O. we will be writing an echo server.

The echo server binds a `TcpListener` and accepts inbound connections in a loop. For each inbound connection, data is read from the socket and written immediately back to the socket. the client sends data to the server and receives the exact same data back.

We will implement the echo server twice using slightly different strategies.

### Using `io::copy`

We will implement the logic using `io::copy` utility.

As seen earlier, this utility function takes a reader and a writer and copies data from one to the other. However, we only have a single `TcpStream`. The single value implements both `AsyncRead` and `AsyncWrite`. Because `io::copy` requires `&mut` for both the reader and the writer, the socket cannot be used for both arguments. 

### Splitting a reader + writer
To work around this probklem, we must split the socket into a reader hadnle and a writer handle. The best way to split a reader/writer combo depends on the specific type. Any reader + writer type can be split using `io::split` utility. This function takes a single value and returns seperate reader and writer handles. These two handles can be used independently, including from separate tasks. 

Because `io::split` supports any value that implements `AsyncRead + AsyncWrite` and returns independent handles, internally `io::split` uses an `Arc` and a `Mutex`. This overhead can be avoided with `TcpStream`, which offers two specialized functions.

`TcpStream::split` takes a reference to the stream and returns a reader and writer handle. Because a reference is used, both handles must stay on the same task that `split()` was called from. This specialized `split` is zero-cost. there is no `Arc` or `Mutex` needed. `TcpStream` also provides `into_split` which supports handles that can move across tasks at the cost of only an `Arc`.

Because `io::copy()` is called on the same task that owns the `TcpStream`, we can use `TcpStream::split`. 

### Manual Copying
We can copy the data manually by using `AsyncReadExt::read` and `AsyncWriteExt::write_all`.

### Allocating a Buffer

The strategy is to read some data from the socket into a buffer then write the contents of the buffer back to the socket.

```rust
let mut buf = vec![0; 1042];
```

A stack buffer is explicitly avoided. REcall from earlier, we noted that all task data that lives across calls to `.await` must be stored by the task. In this case, `buf` is used across `.await` calls. All task data is stored in a single allocation. You can think of it aas an `enum` where each variant is the data that needs to be stored  for a specific call to `.await`.

If the buffer is represented by a stack array, the internal structure for tasks spawned per accepted socket might look something like: 

```rust
struct Task {
  // Internal task fields here
  task: enum {
    AwaitingRead {
      socket: TcpStream,
      buf: [BufferType],
    },
    AwaitingWriteAll {
      socket: TcpStream,
      buf: [BufferType],
    }
  }
}
```

If a stack array is used as the buffer type, it will be stored inline in the task structure. This will make the task structure very big. Additionally, buffer sizes are often page sized. This will make `Task` an awkward size: `$page-size + a-few-bytes`.

The compiler optimizes the layout of async blocks furher than a basic `enum`. In practice, variables are not moved around between variants as would be required with an `enum`. However, the task struct size is at least as big as the largest variable. 

Because of this, it is usually more efficient to use a dedicated allocation for the buffer. 

### Handling EOF

When the read half of the TCP stream is shut down, a call to `read()` returns `Ok(0)`. It is important to exit the read loop at this point. Forgetting to break from the read loop on EOF is a common source of bugs.

```rust
loop {
  match socket.read(&mut buf).await {
    // Return value of `Ok(0)` signifies that the remote has closed
    Ok(0) => return,
    // other cases
  }
}
```

Forgetting to break from the read loop usually results in a 100% CPU infinite loop situation. As the socket is closed, `socket.read()` return immediately. The loop then repeats forever. 
