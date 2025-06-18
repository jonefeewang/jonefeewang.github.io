---
layout: post
title: >-
  Rewriting Kafka in Rust Async: Insights and Lessons Learned
  in Rust
date: 2025-06-18 09:27:40
description: >-
 Here is a summary of the lessons and insights gained from rewriting Kafka using Rust Async.
tags:
  - Message Queue
  - Rust
  - Kafka
  - Pulsar
  - StoneMQ
toc: true
---

## TL;DR:

1. Rewriting Kafka in Rust not only leverages Rust's language advantages but also allows redesigning for superior performance and efficiency.
2. Design Experience: Avoid Turning Functions into async Whenever Possible
3. Design Experience: Minimize the Number of Tokio Tasks
4. Design Experience: Judicious Use of Unsafe Code for Performance-Critical Paths
5. Design Experience: Separating Mutable and Immutable Data to Optimize Lock Granularity
6. Design Experience: Separate Asynchronous and Synchronous Data Operations to Optimize Lock Usage
7. Design Experience: Employ Static Dispatch in Performance-Critical Paths Whenever Possible

At the project's inception, I initially considered implementing [StoneMQ](https://github.com/jonefeewang/stonemq) using C/C++. After grappling with the frustrating and persistent off-heap memory leak issues encountered in DDMQ (built on Pulsar) within the JVM environment, I realized that relying on off-heap memory for caching or buffering in the JVM is far from ideal. Having operated the Mafka cluster (based on Kafka) for nearly five years, we never faced memory-related problems such as abnormal memory growth, prolonged garbage collection times, or memory leaks. When a memory leak occurs on a single node in a cluster, memory usage escalates uncontrollably, leaving you no choice but to perform periodic restarts—a daunting and risky task when managing thousands of nodes.

Kafka and Pulsar both run as services on Linux; more precisely, they operate within the JVM and have no direct interaction with the native Linux system. Strictly speaking, they do not belong to the domain of system-level programming. So why choose C/C++ as the underlying language? This choice clearly results in reduced development efficiency and increased complexity.  

My perspective is that Kafka and Pulsar, as messaging queues, have become de facto industrial standards, having matured and stabilized over nearly eight years. Could we not reimplement them in a lower-level language to reap greater benefits in performance and stability? With some enhancements, might we transform them into first-class native Linux processes, achieving a level of integration and efficiency akin to that of the Linux kernel’s own services?

Through extensive research, it has become evident that C/C++ is gradually becoming a sunset language, nearing its twilight. Its successor, Rust, which emerged around 2010, has spent over a decade steadily conquering the domain of systems programming. Even the notoriously exacting Linus Torvalds could no longer ignore Rust’s presence; in a recent kernel update, he resolutely overcame opposition from the C community to formally integrate Rust into the Linux kernel mainline for the first time.  

Google has employed Rust for years in Android’s low-level development, claiming efficiency and safety several times superior to C++. The CTO of Microsoft Azure has even recommended that new projects move away from C/C++ in favor of Rust. Amazon AWS, notably, has recruited key Rust contributors and utilized Rust to build innovative projects such as the new S3 backend storage and the renowned Firecracker microVM.  

Against this backdrop, I decisively chose Rust as the development language for StoneMQ. Admittedly, Rust has a steep learning curve, a common critique among developers. However, while challenging, it is by no means insurmountable. Moreover, system developers constitute a relatively small niche within the broader developer community, so widespread adoption is not imperative.  

After a year of iterative development—learning Rust while coding—StoneMQ has encountered and anticipated numerous pitfalls. Here, I summarize some of these lessons, with plans to elaborate further in the future, hoping to aid others in avoiding similar missteps. (Repository: https://github.com/jonefeewang/stonemq)  

Though StoneMQ was developed concurrently with my Rust learning journey, it was built to exacting standards: first, to outperform Kafka in terms of performance; second, to maintain clear and readable code. Achieving these goals demanded meticulous attention to design and optimization throughout development. StoneMQ underwent two major restructurings and numerous minor refactorings—details of which can be explored through its extensive git commit history.

## 1. Avoid Turning Functions into async Whenever Possible

In Rust, declaring a function as `async fn` compiles it into a state machine, and its invocation yields a Future. While asynchronous functions enable more concise code, excessive use of `async`—especially when applied to logic that could be executed synchronously—results in:

- Additional overhead from the Future state machine;
- Proliferation of tasks (e.g., Tokio tasks);
- Increased load on the backend scheduler.

Therefore, the best practice when designing asynchronous interfaces is to **prefer synchronous functions for logic that can run synchronously, reserving async functions solely for genuinely asynchronous operations.**  

If only a portion of a function requires asynchronous handling, separate the synchronous logic from the asynchronous part. This approach allows finer control over task granularity and scheduling overhead, ultimately enhancing overall efficiency and responsiveness.

Consider the following example:

```rust
// INEFFICIENT: Entire function is async, though only the IO operations require
async fn fetch_and_parse(url: &str) -> anyhow::Result<Data> {
    let body = reqwest::get(url).await?.text().await?;
    let parsed = serde_json::from_str::<Data>(&body)?;
    Ok(parsed)
}

// EFFICIENT: Separate synchronous parsing from asynchronous downloading
async fn fetch_and_parse(url: &str) -> anyhow::Result<Data> {
    let body = reqwest::get(url).await?.text().await?;
    Ok(parse_json(&body)?)              // parse_json is sync
}

fn parse_json(raw: &str) -> anyhow::Result<Data> {
    Ok(serde_json::from_str(raw)?)
}

```

## 2. Minimize the Number of Tokio Tasks

The Tokio runtime manages asynchronous operations by scheduling tasks across CPU cores. Excessive task counts can cause several issues:

- Frequent task switching leads to CPU cache thrashing, degrading pipeline efficiency;
- The scheduler may engage in "busy waiting" among numerous idle tasks, wasting CPU cycles;
- Excessive task preemption slows response times and increases latency.

Therefore, judiciously consolidating tasks and avoiding the creation of numerous tiny asynchronous tasks is crucial for optimizing Tokio’s scheduling efficiency. By controlling the number of tasks, the scheduler’s overhead is reduced, allowing critical tasks to receive more CPU time slices, thereby improving system throughput and latency.

**Example code:**

```rust
// Excessive small tasks: spawning a task per record
async fn process_many(records: Vec<String>) {
    for r in records {
        tokio::spawn(async move {
            process_record(r).await;
        });
    }
}
```

Reduce task count by batching processing:

```rust
async fn process_many(records: Vec<String>) {
    // Spawn a single task to process multiple records in batch
    tokio::spawn(async move {
        for r in records {
            process_record(r).await;
        }
    });
}
```

Alternatively, design a pipeline using asynchronous channels to further reduce the number of tasks.

### 2.1 Design Experience: Efficient Task Management and Request Handling in StoneMQ

When designing the server architecture for StoneMQ, I paid special attention to how asynchronous tasks (`tokio::task`) are managed, aiming to maximize resource efficiency and system stability.

#### The Problem with Excessive Task Spawning

A common but problematic pattern in async server design is to spawn a new task for every incoming request. While this approach is simple, it can quickly lead to resource exhaustion under high load, as each request consumes memory and scheduling overhead. This can degrade performance and even cause the server to become unresponsive.

#### My Approach: Connection-per-Task, Centralized Request Processing

Instead of spawning a task for every request, I designed the server so that:

- **Each client connection gets a single dedicated task**:  
  In the code, every accepted TCP connection is handled by a `ConnectionHandler`, which is run in its own Tokio task:

  ```rust
  tokio::spawn(async move {
      if let Err(err) = handler.handle_connection().await {
          error!("Connection error: {:?}", err);
      }
      drop(permit);
  });
  ```

- **All requests from all connections are funneled into a shared channel**:  
  Each request, regardless of which connection it comes from, is sent into a central channel (`async_channel::Sender<RequestTask>`).

- **A fixed-size pool of worker tasks processes all requests**:  
  Instead of spawning a new task per request, a configurable number of worker tasks are started at server boot. These workers continuously pull requests from the channel and process them:

  ```rust
  for i in 0..num_channels {
      let rx: async_channel::Receiver<RequestTask> = request_rx.clone();
      let replica_manager = replica_manager.clone();
      let group_coordinator = group_coordinator.clone();
      let handle = tokio::spawn(async move {
          debug!("request handler worker {} started", i);
          while let Ok(request) = rx.recv().await {
              process_request(request, replica_manager.clone(), group_coordinator.clone()).await;
          }
          debug!("request handler worker {} exited", i);
      });
      workers.insert(i, handle);
  }
  ```

  This thread-pool-like model ensures that system resources are used efficiently, and the number of concurrent tasks is controlled.

#### Automatic Worker Recovery

To further enhance robustness, I implemented a monitoring mechanism:  
A dedicated monitor task periodically checks the health of each worker. If a worker panics or exits unexpectedly, the monitor automatically spawns a replacement to maintain the desired pool size:

```rust
if join_error.is_panic() {
    // ... log error ...
    // re-generate a new task
    let rx = request_rx.clone();
    let replica_manager = replica_manager.clone();
    let group_coordinator = group_coordinator.clone();
    let new_worker = tokio::spawn(async move {
        while let Ok(request) = rx.recv().await {
            process_request(request, replica_manager.clone(), group_coordinator.clone()).await;
        }
    });
    workers.insert(id, new_worker);
}
```

#### Benefits

- **Resource Efficiency**: By limiting the number of worker tasks, the server avoids the overhead of thousands of concurrent tasks under heavy load.
- **Predictable Performance**: The thread-pool model provides more consistent latency and throughput.
- **Fault Tolerance**: The monitor ensures that if a worker fails, it is quickly replaced, maintaining system stability.
- **Scalability**: The number of worker tasks can be tuned according to system capabilities and expected load.

#### Conclusion

This design pattern—dedicating a task per connection, centralizing request processing, and using a fixed-size worker pool—strikes a balance between concurrency and resource control. It’s a practical approach for building scalable, robust, and efficient async servers in Rust with Tokio.

Such requirements frequently arise across various scenarios. To address this, I developed a utility class that facilitates implementing such designs with ease. For detailed reference, please consult the `multiple_channel_worker_pool` and `single_channel_worker_pool` modules within the Utils package of [StoneMQ](https://github.com/jonefeewang/stonemq).

## 3. Design Experience: Favoring Lock-Free Architectures and Minimizing Use of Tokio Async Locks

When architecting high-performance async systems in Rust, I strongly prefer designs that avoid locks—especially asynchronous locks—whenever possible. This principle is rooted in both performance considerations and code safety.

#### Why Avoid Locks, Especially Async Locks?

- **Contention and Complexity**: Traditional locks (`Mutex`, `RwLock`) can become contention points and introduce complexity, especially in concurrent environments.
- **Async Lock Performance**: Tokio’s asynchronous locks (`tokio::sync::Mutex`, `tokio::sync::RwLock`) are significantly slower than their synchronous counterparts. They are designed for correctness in async contexts, but their performance overhead can be substantial, especially under high contention or frequent lock/unlock cycles.
- **Deadlock Risk**: Holding any lock across `.await` points is risky and can easily lead to deadlocks or subtle bugs.

#### My Approach: Channels, Message Passing, and Task Ownership

- **Single-threaded Task Ownership**: Each async task owns its state, eliminating the need for locks in most cases.
- **Channel-based Communication**: Tasks interact via channels, passing messages instead of sharing mutable state.
- **Minimal, Synchronous Locking**: When shared mutable state is unavoidable, I use fine-grained, synchronous locks (`std::sync::Mutex`/`RwLock`), and always ensure locks are held for the shortest possible time and never across `.await`.
- **Rare Use of Tokio Async Locks**: I avoid Tokio’s async locks unless absolutely necessary. In almost all cases, the architecture is designed so that async locks are not required.

#### Example: Journal Log Write Path

In the journal log write implementation, all critical sections that require locking are handled synchronously and released before any async operation:

```rust
{
    // Synchronously update active segment index
    let mut active_seg_index = self.active_segment_index.write();
    let old_segment_index = std::mem::replace(&mut *active_seg_index, new_seg);
    self.active_segment_base_offset
        .store(new_base_offset, Ordering::Release);
    // ... update other state ...
} // lock released here

// Async operations follow, with no lock held
get_journal_segment_writer()
    .append_journal(journal_log_write_op)
    .await?;
```

#### Benefits

- **Performance**: Avoiding Tokio’s async locks removes a major source of latency and contention in async Rust applications.
- **Safety**: By never holding locks across `.await`, I eliminate a whole class of async deadlocks.
- **Simplicity**: The code is easier to reason about, as ownership and mutability are clear and explicit.

#### Conclusion

By leveraging message passing, task-local state, and only minimal, synchronous locking, I achieve both high efficiency and safety in async Rust systems. Avoiding Tokio’s async locks is a deliberate choice, as their performance overhead is rarely justified except in very specific scenarios. This approach leads to scalable, maintainable, and robust code.

## 4. Design Experience: Judicious Use of Unsafe Code for Performance-Critical Paths

In Rust, safety is a core language feature, but there are scenarios—especially in performance-critical systems programming—where the cost of absolute safety can be significant. In such cases, I carefully consider the use of `unsafe` code to achieve the necessary performance, while still maintaining overall correctness and encapsulation.

#### Why Use Unsafe Code?

- **Performance**: Some operations, such as memory-mapped file access or direct byte manipulation, can be much faster when performed without the overhead of Rust’s safety checks.
- **System-level Access**: Certain low-level APIs (e.g., memory mapping, FFI, or direct buffer manipulation) require `unsafe` blocks to interact with OS or hardware resources.

#### My Approach: Isolate and Audit Unsafe Code

- **Encapsulation**: Unsafe code is always encapsulated within well-tested, minimal, and clearly documented functions or modules.
- **Justification**: Unsafe is only used when profiling or design analysis shows that safe alternatives are a bottleneck.
- **Testing**: I ensure that all unsafe code is covered by thorough tests and code reviews.

#### Example: Memory-Mapped Index Files

In the `index_file.rs` module, I use the `memmap2` crate to memory-map index files for fast, zero-copy access. Creating a memory map inherently requires `unsafe` code, as shown here:

```rust
let mmap = unsafe { MmapOptions::new().map(&file)? };
```

This allows the index file to be accessed as a byte slice, enabling efficient binary search and direct manipulation without extra copying or allocation. Similarly, for writable index files:

```rust
let mmap = unsafe { MmapOptions::new().map_mut(&file)? };
```

By isolating the `unsafe` block to just the memory mapping operation, the rest of the code can remain safe and idiomatic Rust.

#### Benefits

- **Maximum Performance**: Direct memory access and zero-copy operations are critical for high-throughput log/index management.
- **Controlled Risk**: By limiting the scope of `unsafe`, I minimize the risk of undefined behavior.
- **Maintainability**: Encapsulated unsafe code is easier to audit and reason about.

#### Conclusion

While Rust’s safety guarantees are invaluable, there are times when performance requirements justify the careful use of `unsafe` code. By isolating and rigorously testing these sections, I can achieve both the speed and reliability needed for system-level components like log and index management.

---

## 5. Design Experience: Separating Mutable and Immutable Data to Optimize Lock Granularity

A key architectural principle I follow in high-performance systems is to separate mutable and immutable data, and to use the finest possible lock granularity. This approach minimizes contention, improves concurrency, and makes the system easier to reason about.

#### Why Separate Mutable and Immutable Data?

- **Reduced Contention**: By isolating mutable state, only the truly changing parts of the system require synchronization, while immutable data can be freely shared and accessed without locks.
- **Optimized Locking**: Fine-grained locks (e.g., per-segment or per-structure) allow multiple operations to proceed in parallel, rather than serializing all access through a single global lock.
- **Clearer Ownership**: The distinction between mutable and immutable data clarifies which parts of the code can safely share data and which require careful coordination.

#### My Approach: Example from Journal Log Management

In the journal log implementation, I apply this principle by:

- **Using separate structures for mutable and immutable data**:  
  - The `active_segment_index` (the currently writable segment) is protected by a `RwLock`, allowing multiple readers or a single writer.
  - The set of segment base offsets (`segments_order`) is also protected by a `RwLock`, but is only modified when segments are added or removed.
  - All read-only segments are stored as `Arc<ReadOnlySegmentIndex>` in a concurrent `DashMap`, allowing lock-free concurrent reads.

```rust
#[derive(Debug)]
pub struct JournalLog {
    // Ordered set of segment base offsets (non-active segments)
    segments_order: RwLock<BTreeSet<i64>>,

    // Map of base offsets to read-only segments
    segment_index: DashMap<i64, Arc<ReadOnlySegmentIndex>>,

    // Currently active segment
    active_segment_index: RwLock<ActiveSegmentIndex>,

    // ... other fields ...
}
```

- **Read-only data is stored in `ReadOnlySegmentIndex`**:  
  Once a segment is no longer active, it is converted to a read-only structure and stored in an `Arc`, making it safe to share across threads without any locking.

- **Lock scope is minimized**:  
  For example, when rolling a segment, the lock on `active_segment_index` is only held for the duration of the swap, and not across any async or I/O operations.

#### Benefits

- **High Concurrency**: Multiple threads can read from different segments or query the segment index concurrently, without blocking each other.
- **Minimal Locking Overhead**: Only the small, mutable parts of the system are protected by locks, and those locks are held for the shortest possible time.
- **Scalability**: The system can efficiently handle many concurrent operations, such as reads, writes, and segment management.

#### Conclusion

By separating mutable and immutable data, and applying the smallest possible lock granularity, I achieve both high performance and maintainability. This pattern is especially effective in log-structured storage and similar systems, where most data is append-only or read-mostly, and only a small portion is actively mutated.

## 6. Separate Asynchronous and Synchronous Data Operations to Optimize Lock Usage

The locking requirements of asynchronous and synchronous code differ fundamentally: holding a synchronous lock within asynchronous code can cause the entire Future dependency chain to block, negating the benefits of the asynchronous model. Conversely, asynchronous locks incur higher overhead and employ more complex mechanisms.

The best practice is to **segregate the data involved in asynchronous operations from that used in synchronous operations**, ensuring that locks protecting synchronous data do not span asynchronous contexts. This allows synchronous portions to safely utilize efficient synchronous locks (such as `std::sync::Mutex`), while the asynchronous parts avoid blocking or contention.

By isolating data domains and restructuring access patterns accordingly, you can significantly reduce lock contention and blocking, thereby enhancing asynchronous execution efficiency.

#### Example: [StoneMQ]() Journal Log Write Path

In my project, the journal log write implementation demonstrates this pattern clearly. Here’s a simplified version:

```rust
// Synchronously update the active segment index
{
    let mut active_seg_index = self.active_segment_index.write();
    let old_segment_index = std::mem::replace(&mut *active_seg_index, new_seg);
    // ... update other state ...
} // lock released here

// Now perform async operations, with no lock held
get_journal_segment_writer()
    .append_journal(journal_log_write_op)
    .await?;
```

Notice how the lock is acquired, the necessary mutation is performed, and then the lock is released **before** any async operation is awaited.

#### Example: Group Coordinator

In the group coordinator logic, the same principle is applied. Before calling any async function, the lock is explicitly dropped:

```rust
drop(locked_group);
self.maybe_prepare_rebalance(group).await;
```

This ensures that no lock is ever held across an `.await`, preventing deadlocks and maximizing concurrency.

### Benefits

- **Performance**: Synchronous locks are much faster than async locks, and can be used safely when not crossing async boundaries.
- **Safety**: Avoids deadlocks and subtle bugs that can arise from holding locks across `.await`.
- **Clarity**: The code is easier to reason about, as the boundaries between sync and async operations are explicit.

### Conclusion

By separating async and sync data, and ensuring that locks are only used for sync data and never held across `.await`, you can write high-performance, safe, and maintainable async Rust code. This pattern is a cornerstone of robust async system design, and has paid off time and again in my own projects.

## 7. Employ Static Dispatch in Performance-Critical Paths Whenever Possible

### Design Experience: Using Enums for Protocol Type Polymorphism in Performance-Critical Code

When building the protocol layer for [StoneMQ](https://github.com/jonefeewang/stonemq), I faced a key design decision: how to represent and handle the various data types that appear in protocol messages. The two main options were using Rust’s trait objects for dynamic polymorphism, or using an enum to statically represent all possible protocol types.

#### Why Not Trait Objects?

Trait objects (`dyn Trait`) are a common way to achieve polymorphism in Rust. They allow different types to be handled through a common interface, with the actual method implementations determined at runtime. This approach is flexible and can make code easier to extend. However, it comes with some downsides:

- **Dynamic dispatch**: Every method call on a trait object involves a runtime lookup, which adds overhead.
- **Heap allocation**: Trait objects often require heap allocation, especially when used in collections.
- **Performance impact**: In a performance-critical layer like a network protocol, even small inefficiencies can add up.

#### The Enum Approach

To avoid these costs, I chose to use an enum—`ProtocolType`—to represent all protocol data types. Each variant of the enum corresponds to a specific protocol type, such as `I8`, `I16`, `PString`, `Array`, etc. Here’s a simplified excerpt from the actual code:

```rust
pub enum ProtocolType {
    Bool(Bool),
    I8(I8),
    I16(I16),
    I32(I32),
    U32(U32),
    I64(I64),
    PString(PString),
    NPString(NPString),
    PBytes(PBytes),
    NPBytes(NPBytes),
    PVarInt(PVarInt),
    PVarLong(PVarLong),
    Array(ArrayType),
    Records(MemoryRecords),
    Schema(Arc<Schema>),
    ValueSet(ValueSet),
}
```

This design allows me to match on the enum and handle each type explicitly, with all dispatch resolved at compile time. For example, when calculating the wire format size of a protocol value, I can use a match statement:

```rust
pub fn size(&self) -> i32 {
    match self {
        ProtocolType::Bool(bool) => bool.wire_format_size() as i32,
        ProtocolType::I8(i8) => i8.wire_format_size() as i32,
        // ... other types ...
        ProtocolType::Array(array) => array.size() as i32,
        // ...
        ProtocolType::Schema(_) => {
            panic!("Unexpected calculation of schema size");
        }
        ProtocolType::ValueSet(valueset) => valueset.size() as i32,
    }
}
```

#### Benefits in Practice

- **Performance**: All type dispatch is handled at compile time, with no runtime overhead from dynamic dispatch.
- **Type Safety**: The compiler ensures that all protocol types are handled, reducing the risk of runtime errors.
- **Extensibility**: Adding a new protocol type is as simple as adding a new enum variant and updating the relevant match arms.

#### Example: Array Construction

I also used macros and generic functions to make it easy to construct arrays of protocol types:

```rust
macro_rules! array_of {
    ($func_name:ident, $type:ty) => {
        pub fn $func_name(value: $type) -> ProtocolType {
            ProtocolType::Array(ArrayType {
                can_be_empty: false,
                p_type: Arc::new(value.into()),
                values: None,
            })
        }
    };
}

// Usage:
array_of!(array_of_i32_type, i32);
array_of!(array_of_string_type, String);
```

#### Conclusion

By using an enum to represent protocol types, I achieved efficient, type-safe, and maintainable code for the protocol layer. This approach is especially valuable in systems where performance is critical and the set of types is known and relatively stable.

## Summary

Achieving high-performance asynchronous Rust projects transcends mere usage of the async/await syntax; it fundamentally relies on a deep understanding of the underlying task scheduling, lock optimization, and architecture design principles. From eschewing gratuitous async functions to judiciously managing lock granularity; from minimizing the number of spawned tasks to adopting a message-passing architecture; from cautious use of unsafe code to segregating asynchronous and synchronous data; and finally, to maximizing the use of static dispatch—each practice serves to bolster both the efficiency and robustness of your application.

I hope this summary offers valuable insights and assistance to those engaged in Rust asynchronous development. Contributions and discussions are warmly welcomed, as we collectively strive to elevate the practical maturity of the Rust async ecosystem!