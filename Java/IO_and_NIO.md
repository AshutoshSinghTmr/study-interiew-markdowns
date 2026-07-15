# Java IO and NIO

## Two I/O Models

Java offers two I/O APIs:

* **Classic I/O (`java.io`)** — stream-based, blocking, simple. Built around `InputStream`/`OutputStream` (bytes) and `Reader`/`Writer` (characters).
* **NIO (`java.nio`, Java 1.4+; NIO.2 in Java 7)** — buffer- and channel-based, supports non-blocking and multiplexed I/O, plus a modern file API (`Path`/`Files`).

Use `java.io` for simple sequential reads/writes; use NIO for scalable networking, memory-mapped files, and rich filesystem operations.

## Byte Streams vs Character Streams

* **Byte streams** (`InputStream`/`OutputStream`) handle raw binary data (images, serialized objects).
* **Character streams** (`Reader`/`Writer`) handle text with an explicit **charset**, decoding bytes to `char`. Always specify the charset (`StandardCharsets.UTF_8`); relying on the platform default causes portability bugs.

```java
try (var reader = Files.newBufferedReader(path, StandardCharsets.UTF_8)) {
    reader.lines().forEach(System.out::println);
}
```

## Buffering and the Decorator Pattern

`java.io` streams compose via the **decorator pattern**: wrap a low-level stream to add behaviour. Buffering is the most important — it reads/writes in chunks instead of per byte, drastically reducing system calls.

```java
try (var in = new BufferedInputStream(new FileInputStream(file))) { /* ... */ }
```

Other decorators: `DataInputStream`/`DataOutputStream` (primitives), `ObjectInputStream`/`ObjectOutputStream` (serialization), `InputStreamReader`/`OutputStreamWriter` (byte↔char bridge with a charset).

## try-with-resources for Streams

All streams implement `Closeable`, so use try-with-resources to guarantee closure (and flush) even on exceptions. Forgetting to close leaks file handles/sockets.

## NIO Core Abstractions

* **Buffer** — a fixed-size container (`ByteBuffer`, `CharBuffer`) with `position`, `limit`, and `capacity`; you `flip()` between writing and reading.
* **Channel** — a bidirectional conduit to a file or socket (`FileChannel`, `SocketChannel`) that transfers data to/from buffers.
* **Selector** — lets a single thread monitor many channels for readiness (multiplexed, non-blocking I/O) — the basis of scalable servers like Netty.

```java
ByteBuffer buf = ByteBuffer.allocate(1024);
int n = channel.read(buf);   // fill buffer
buf.flip();                  // switch to read mode
while (buf.hasRemaining()) process(buf.get());
```

## Blocking vs Non-Blocking

Classic streams and blocking channels park the thread until data is available — simple but one-thread-per-connection. Non-blocking channels + a `Selector` let one thread service thousands of connections by reacting to readiness events, trading simplicity for scalability. (Virtual threads now make the simple blocking style scale too — see Virtual Threads.)

## NIO.2 File API (`Path` and `Files`)

`java.nio.file` (Java 7) replaces the clunky `java.io.File` with a far richer API:

```java
Path path = Path.of("data", "input.txt");
List<String> lines = Files.readAllLines(path, StandardCharsets.UTF_8);
Files.writeString(path, "hello", StandardCharsets.UTF_8);
try (Stream<Path> tree = Files.walk(dir)) { /* recursive traversal */ }
boolean exists = Files.exists(path);
Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);
```

`Files` offers atomic moves, symbolic-link handling, file attributes, and a `WatchService` for directory change events. Prefer `Path`/`Files` over legacy `File`. Use `Files.newBufferedReader`, `Files.lines` (a lazy stream — close it), and `Files.readString`/`writeString` (Java 11).

## Memory-Mapped Files

`FileChannel.map(...)` maps a file region into memory as a `MappedByteBuffer`, letting the OS page data in/out. This gives very fast random access to large files without explicit read/write calls — used by databases and high-performance caches.

## Serialization and Its Risks

Java serialization (`Serializable`, `ObjectOutputStream`) converts objects to a byte stream and back. It is convenient but carries serious problems:

* **Security** — deserializing untrusted data enables remote code execution via gadget chains; this is a top real-world vulnerability. Never deserialize untrusted input; use `ObjectInputFilter` allowlists if you must.
* **Fragility** — the `serialVersionUID` couples class evolution to the wire format.
* **Performance** — slow and verbose.

Prefer explicit formats (JSON via Jackson, Protobuf, Avro) for persistence and messaging. `transient` fields are skipped; `serialVersionUID` should be declared explicitly to control compatibility.

## Interview Q&A

**Q: Difference between byte streams and character streams?**
A: Byte streams (`InputStream`/`OutputStream`) handle raw binary; character streams (`Reader`/`Writer`) handle text using a charset to decode/encode. Always specify the charset explicitly.

**Q: Why is buffering important?**
A: It batches reads/writes into larger chunks, cutting the number of expensive system calls and greatly improving throughput versus byte-at-a-time I/O.

**Q: What are the core NIO abstractions?**
A: Buffers (fixed-size data containers with position/limit/capacity), channels (bidirectional conduits to files/sockets), and selectors (one thread monitoring many channels for readiness).

**Q: Blocking vs non-blocking I/O?**
A: Blocking parks a thread per connection (simple); non-blocking with a selector lets one thread handle many connections by reacting to readiness — scalable but more complex. Virtual threads let blocking code scale too.

**Q: `java.io.File` vs `java.nio.file.Path`/`Files`?**
A: `Path`/`Files` (NIO.2) is the modern API with atomic operations, attributes, symlink handling, tree walking, and a watch service; `File` is legacy with poor error reporting.

**Q: What does `flip()` do on a buffer?**
A: It switches a buffer from write mode to read mode by setting the limit to the current position and resetting position to zero, so you can read what you wrote.

**Q: What is a memory-mapped file?**
A: A file region mapped into memory via `FileChannel.map` as a `MappedByteBuffer`, letting the OS handle paging for fast random access to large files.

**Q: Why is Java serialization dangerous?**
A: Deserializing untrusted data can trigger remote code execution via gadget chains; it's also fragile (versioning) and slow. Avoid it for untrusted input; prefer JSON/Protobuf and use `ObjectInputFilter` if unavoidable.

**Q: What do `transient` and `serialVersionUID` do?**
A: `transient` excludes a field from serialization; `serialVersionUID` is a version identifier controlling deserialization compatibility across class changes.

**Q: Why must you close `Files.lines(...)`?**
A: It returns a lazy stream backed by an open file handle; failing to close it (via try-with-resources) leaks the handle.

## Interview Notes

* Contrast `java.io` (blocking, stream/decorator) with `java.nio` (buffers/channels/selectors, non-blocking).
* Explain byte vs char streams and always specifying a charset.
* Describe buffering and try-with-resources for resource safety.
* Know buffer mechanics (`flip`, position/limit/capacity) and selector-based multiplexing.
* Prefer `Path`/`Files` (NIO.2) and know memory-mapped files.
* Be able to explain serialization's security risks and safer alternatives.
