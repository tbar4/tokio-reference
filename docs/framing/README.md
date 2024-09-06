# Framing

We can now apply what we learned about I/O and implement the framing layer. FRaming isa the process of taking a byte stream and converting it to a stream of frames. A frame is a unit of data transmitted bewtween two peers. For redis protocl *framing* is defined as follows:

```rust
use bytes::Bytes;

enum Frame {
  Simple(String),
  Error(String),
  Integer(u64),
  Bulk(Bytes),
  Null,
  Array(Vec<Frame>),
}
```

Note how the frame only consists of data without any semantics. The command parsing and implementation happen at a higher level. 

For HTTP, a frame might look like:

```rust
enum HttpFrame {
  RequestHead {
    method: Method,
    uri: Uri,
    version: Version,
    headers: HeaderMap,
  },
  ResponseHead {
    status: StatusCode,
    version: Version,
    headers: HeaderMap,
  },
  BodyChunk {
    chunk: Bytes,
  }
}
```

Tom implement framing for Mini-Redis, we will implement a `Connection` struct that wraps a `TcpStream` and reads/writes `mini_redis::Frame` values.

```rust
use tokio::net::TcpStream;
use mini_redis::{Frame, Result};

struct Connection {
  stream: TcpStream,
  // other fields
}

impl Connection {
  // Read a frame from the connection
  // Returns `None` if EOF is reached
  pub async fn read_frame(&mut self) -> Result<Option<Frame>> {
    // implementation here
  }

  // Write a frame to the connection
  pub async fn write_frame(&mut self, frame: &Frame) -> Result<()> {
    // implementation here
  }
}
```
## Buffered Reads

The `read_frame` method waits for an entire frame to be received before returning. A single call to `TcpStream::read()` may return an arbitrary amount of data. It could contain an entire frame, a partial frame, or multiple frames. If a partial frame is received, the data is buffered and more data is read from the socket. If multiple frames are received, the first frame is returned and the rest of the data is buffered until the next call to `read_frame`.

 To implement, `Connection` needs a read buffer field. Data is read form the socket into the read buffer.  When a frame is parsed, the corresponding data is removed from the buffer. We will use `BytesMut` as the buffer type, which is a mutable version of `Bytes`.

We can breakdown our `connection.rs` file. The `read_frame` method operates in a loop. First, `self.parse_frame()` is called. This will attempt to parse a redis frame from `self.buffer`. If there is enough data to parse a frame, the frame is returned to the caller of `read_frame()`. Otherwise, we attempt to read more data from the socket into the buffer. After reading more data, `parse_frame()` is called again. This time, if enough data has been received, parsing may succeed. 

When reading from the stream, a return value of `0` indicates that no more data will be received from the peer. If the read buffer still has data in it, this indicates a partial frame has been received and the connection is being terminated abruptly. This is an error condition and Err is returned. 

### The `Buf` trait
When reading from the stream, `read_buf` is called. This version of the read function takes a value implementing `BufMut` from the `bytes` crate. First, consider how we would implement the same read loop using `read()`. `Vec<u8>` could be used instead of `BytesMut`.

```rust
use tokio::net::TcpStream;

pub struct Connection {
  stream: TcpStream,
  buffer: Vec<u8>,
  cursor: usize,
}

impl Connection {
  pub fn new(stream: TcpStream) -> Connection {
    Connection {
      stream,
      // Allocate the buffer with 4kb of capacity
      buffer: vec![0; 4096],
      cursor: 0,
    }
  }
}
```

And the `read_frame()` function on `Connection`:

```rust
use mini_redis::{Frame, Connection};

pub async fn read_frame(&mut self) -> Result<Option<Frame>> {
  loop {
    if let Some(frame) = self.parse_frame()? {
      return Ok(Some(frame));
    }

    // Ensure the buffer has capacity
    if self.buffer.len() == self.cursor {
      // grow the buffer
      self.buffer.resize(self.cursor * 2, 0);
    } 

    // Read into the buffer, tracking the
    // number of bytes read
    let n = self.stream.read(
      &mut self.buffer[self.cursor..]
    ).await?;

    if 0 == n {
      if self.cursor == 0 {
        return Ok(None);
      } else {
        return Err("connection reset by peer".into());
      }
    } else {
      self.cursor += n;
    }
  }
}
```

When working with byte arrays and `read`, we must also maintain a cursor tracking how much data has been buffered. We must make sure to pass the empty portion of the buffer to `read()`. Otherwise, we would overwrite buffered data. If our buffer gets filled up, we must grow the buffer in order to keep reading. In `parse_frame()` (not included), we would need to parse data contained by `self.buffer[..self.cursor]`.

Because pairing a byte array with a cursor is very common, the bytes crate provides an abstraction representing a byte array and cursor. the `Buf` trait is implemented by types from which data can be read. The `BufMut` trait is implemented by types into which data can be written. When passing a `T: BufMut` to `read_buf()`, the buffer's internal cursor is automtically updated by a `read_buf`. Because of this, in our version of `read_frame`, we do not need to manage our own cursor. 

Additonally, when using `Vec<u8>`, the buffer must be **initialized**. `vec![0; 4096]` allocates an array of 4096 bytes and writes zero to every entry. When resizing the buffer, the new capacity must also be initialized with 0's. The initialization processis not free. When working with `BytesMut` and `BufMut`, capacity is uninitialized. The `BytesMut` abstraction prevents us from reading the uninitialized memeory. This lets us avoid the initialization step. 

## Parsing

Let's look at the `parse_frame()` function. Parsing is done in two steps: 

1) Ensure a full frame is buffered and find the end index of the frame.
2) Parse the frame.

The `mini-redis` crate provices us with a function for both of these steps:

1) `Frame::check`
2) `Frame::parse`

We will also reuse the `Buf` abstraction to help. A `Buf` is passed into `Frame::check`. As the `check` function iterates the passed in buffer, the internal cursor will be advanced. When `check` returns the buffer's internal cursor points to the end of the frame.

The relevant thing to note is that `Buf`'s "byte iterator" style APIs are used. These fetch data and advance the internal cursor. For example, to parse a frame, the first byte is checked to determine the type of the frame. The function used is `Buf::get_u8`. This fetches the byte at the current cursor's position and advances the cursor by one.

## Buffered Writes

The other half of the framing API is the `write_frame(frame)` function. This function writes an entire frame to the socket. In order to minimize `write` syscalls, writes will be buffered. A write buffer is maintained and frames are encoded to this buffer befire being written to the socket. However, unlike `read_frame()`, the entire frame is not always buffere to a byte array before writing to the socket. 

Consider a bulk stream frame. The value being written is `Frame::Bulk(Bytes)`. the wire format of a bulk frame is a frame head, which consists of the `$` character followed by the data length in bytes. The majority of the frame is the contents of the `Bytes` value. If the data is large, copying it to an intermediate buffer would be costly. 

Tom implement buffered writes, we will use the `BufWriter` struct. This struc is initialized with a `T: AsyncWrite` and implements `AsyncWrite` itself. When `write` is called on `BufWriter`, the write does not go directly to the inner writer, but to a buffer. When the buffer is full, the contents are flusehd to the inner writer and the inner buffer is cleared. There are also optimizations that allow bypassing the buffer in cerain cases. 

The functions used here are provided by `AsyncWriteExt`. They are available on `TcpStream` as well, but it would not be advisable to issue single byte writes without the intermediate buffer. 

* `write_u8` writes a single byte to the writer
* `write_all` writes the entire slice to the writer
* `write_decimal` is implemented by mini-redis

The function ends with a call to `self.stream.flush().await`. Because `BufWriter` stores writes in an intermediate buffer, calls to `write` do no guarentee that the data is writtne to the socket. Before returning, we want the frame to be written to the socket. The call to `flush()` writes any data pending in the buffer to the socket. 

Another alternative would be to not call `flush()` in `write_frame()`. Instead, provide a `flush()` function on `Connection`. This would allow the caller to write queue multiple small frames in the write buffer then write them al to the socket with one `write` syscall. Doing this complicates the `Connection` API. Simplicity is one of Mini-Redis' goals, so we decided to include the `flush().await` call in `fn write_frame()`.
