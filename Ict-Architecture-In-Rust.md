!!! THIS IS A DRAFT AND NOT A FINISHED DOCUMENT FOR EARLY FEEDBACK !!!

# Summary

This document is about an early - probably the first - attempt to port the [official IOTA Ict node](https://github.com/iotaledger/ict.git) implementation written in Java to the Rust programming language. This rather basic implementation was named **Ictarus** and we will be using this term for simple reference. Its source code can be found [here](https://github.com/Alex6323/Ictarus.git). The author's intention was to get a good understanding about the inner workings of the Ict node software and at the same time deepen his knowledge about the Rust programming language. This document is intended to summarize the findings and explain some of the architectural deviations from the Java prototype that were necessary to end up with an idiomatically written Rust software.

# About using Rust

According to Wikipedia [Rust](https://en.wikipedia.org/wiki/Rust_(programming_language)) is a multi-paradigm systems programming language focused on safety, especially safe concurrency, which is syntactically similar to C++, but is designed to provide better memory safety while maintaining high performance. 

Some of Rust's most **notable** features are:
* compiled language
* no automated garbage collection
* statically typed (but uses type inference to reduce boilerplate)
* forced error handling
* immutable variables by default
* follows composition over inheritance
* functional programming patterns (iterators, generators, closures)
* built-in support for tests and benchmarks
* easy-to-use tools for toolchain and dependency management for improved productivity

What makes Rust **unique** though is its ability to be not only a very safe language (guaranteed memory safety, no data races) but also be on par with the most performant languages in existence today that traditionally would have been picked instead. But compared to those with Rust one no longer has to sacrifice safety over performance or vice versa. 

How does Rust achieve that? First and foremost by restricting itself to only employ *zero-cost abstractions*. In simple terms that basically means, that
1. You do not "pay" for abstractions that you don't use.
2. If you use a certain abstraction you cannot improve performance by hand-coding it.

 On the other hand Rust introduces some **new concepts**, most notably those of *Ownership and Borrowing* which are an evolutionary step forward in language design. Those concepts tackle the problem of secure memory and resource management, and applied make whole classes of bugs in Rust just impossible which plague software development since decades. It does so without impact on performance unlike other "solutions" to that problem like garbage collection.

The consequence of those concepts is however, that more care has to be taken upfront when laying out the architecture. When porting software written in an OOP style to Rust one quickly realizes that some solutions cannot be reimplemented 1:1 with just changed syntax. The Rust compiler will often times reject those attempts and for good reasons. As an example: It is not as trivial as in other languages to create self-referential objects like nodes in a linked list or vertices in a graph. There is a whole (open) [book](https://rust-unofficial.github.io/too-many-lists/index.html) written to address this topic in Rust. Finding a good implementation here will be necessary to create a performant Tangle implementation that is capable of running even on low-end devices. All in all one can expect the final Rust Ict node to be looking very different internally compared to its Java precursor.

In the future the Ict network will be used for moving not only data but also large amounts (in sum over time) of value by utilizing directly connected and usually low-powered embedded IoT devices. The chosen programming language to build such a system should therefore be as safe *and* as performant as possible. Rust fulfills such requirements, and therefore should be a very good fit. But it should also be noted, that Rust is still a very young language and many libraries are still in heavy development themselves. There might be problems ahead related to some dependencies being in beta state themselves, or the Rust community still being relatively small. But due to the ever increasing popularity of Rust this can be expected to become less and less of a problem in the years to come.  

# About Ict

The **I**ota **c**ontrolled agen**t** (Ict) is a very flexible, modularized IOTA node. It can be very light-weight, but is not restricted to be so. Depending on the usecase it can be a very powerful node as well. It achieves this by implementing the IOTA eXtension Interface (IXI). The Ict node is specifically designed for the Internet of Things (IoT) and its diverse use cases. Targeted platforms are at first single-board computers (SBCs) like Raspberry Pis and similar devices. The final vision also includes a combination of microcontrollers (MCUs) with capabilities like the [Cortex-M4](https://developer.arm.com/ip-products/processors/cortex-m/cortex-m4) and upwards, and [tiny FPGAs](https://hackaday.io/project/26848-tinyfpga-b-series) to do the heavy lifting. To achieve this the node software is logically and programmatically separated into one single core component (Ict core) and in many so-called IXI modules, which can be written in different programming languages. The core itself might consist of replaceable units that allow for customized compilations to reduce the size of the binary if necessary.

While the Ict core implements the IOTA gossip protocol and is responsible for establishing a P2P network, buffering a certain amount of gossip, holding and updating the Tangle, and functioning as a gateway between the P2P network and its attached modules, the modules themselves define the actual functionality and the behavior of the node. For example, some module might turn the node into an access point for a P2P chat application, another module might turn the node into a data filtering permanode that diverts data into a database. The more resources available on the target platform the node is running on, the more modules can be hosted by the core and the more powerful the node becomes overall. On very constrained devices however it will be possible to create a very specific single-use behavior.

# About Ictarus

`Ictarus` is a Rust port of the Java Ict node software implementation. It was developed as a proof-of-concept (PoC) to show the feasibility of developing the Ict node software in Rust and not as a finished replacement for the Java version. Instead of adding features quickly and achieve feature parity the focus was on finding possible optimizations. Examples of that approach are bringing BCT-Curl and Troika to Rust, which can be found [here](https://github.com/Alex6323/bct_curl_rust), evaluating the use of transaction compression, which can be found [here](https://github.com/Alex6323/iota-lz4-udp) and helping the Java team with showcasing the use of protocol buffers in bridge.ixi which can be found [here](https://github.com/Alex6323/bridge.ixi-rust). There will be more such smaller "practical research" projects related to the network layer and the in-memory storage of the Tangle. Then we'll try to find consensus in our working group how to move forward.

# Differences between `Ictarus` and the Java reference implementation

On a high level `Ictarus` follows the Java reference implementation very closely, though not 100%. In fact both implementations cannot run on the same network at the moment. Why that is will be explained in the following paragraphs. Due to the Rust language specifics the internal realization differs in many ways. That is because Rust is more of a functional programming language than an object-oriented one. That has many consequences on the overall architecture. For example, there are no classes and no object inheritence in Rust. Rust uses containment hierarchies instead. 

## Using poll-based futures for concurrency

For concurrency in Rust it is idiomatic to use `futures` and `tokio` crates/libraries. Those are based on cooperative multitasking as opposed to preemptive multitasking, which simply means, that tasks try to make some progress for example on some I/O resource, and voluntarily yield execution back to the task executor if they can't. While this has the danger of locking up the whole application when implemented not carefully, it has the benefit of being a zero-cost abstraction resulting in maximum performance. Since being able to support weak devices is a major goal of this project this is of utmost importance.

## Attach request hashes only if required to spare bandwidth

The reason why the Rust based Ict implementation is not compatible with the Java version is that transactions only carry a request hash if an actual request needs to be send. In the Java implementation the packet size is always the same. It fills this space with 9s to indicate that there is no request. Peeking into the future it is very likely that some form of compression will be used which results in a dynamic packet size anyway. Another problem with using transactions as request carrier is that sometimes packets get dropped by ISPs due to their size. And another problem is, that requests cannot be sent idependently from the gossip which introduces unnecessary latencies. We are looking into protocols that support stream multiplexing to being able to send gossip and requests in parallel, which also reduces individual packet size.

## Sender is not implemented as a gossip listener

The Java version makes the sender a gossip listener, which could have been done in Rust as well. However, we tried a different approach, i.e. facilitating a shared sending queue across all threads. Upon receiving a transaction via gossip the `receiver` decides whether the transaction needs to be forwarded to the peers. If so, then it determines a random forward delay and adds the transaction hash to a priority-based sending queue which orders its elements depending on timestamps. The `sender` on the other hand also has access to that queue and pops the next transaction hash when the delay has been exceeded. If so the `sender` will pull the bytes from the local buffer, and then send the assembled packet.

## Disallow multiple requests for the same transaction from the same peer

During implementation the author decided to disallow requesting the same transaction over and over, because he reasoned this would encourage lazy neighbors who simply rely on the storage capacity of their peers. This will be further discussed among the working group whether adopt this mechanism, remove it, or find something in between.

## Missing features

Missing features are: 

* IXI interface
* Bundle creation
* Proof-of-work
* Tangle pruning
* Webinterface
* More...

# Architecture Overview

The [architecture](https://raw.githubusercontent.com/iotaledger/ict/master/docs/assets/deployment.png) is almost identical to the Java version, but doesn't have the `Node` abstraction yet. Instead it just uses a `Receiver` and a `Sender` abstraction for dealing with its peers.

## Project structure
The following table gives a basic overview about the project structure. Important types that hold most of the business logic are highlighted:

| File              | Submodule | Function                                          | Dependencies                                               |
| ----------------- | --------- | ------------------------------------------------- | ---------------------------------------------------------- |
| `config.rs`       | root      | type for handling node configuration              | libstd, log                                                |
| `constants.rs`    | root      | central location of project wide constants        | lazy_static, regex                                         |
| `main.rs`         | root      | program entry point                               | libstd, pretty_env_logger                                  |
| `ixi.rs`          | root      | definition of the IOTA extension interface        | libstd                                                     |
| **`ictarus.rs`**  | root      | the actual node                                   | libstd, futures, log, priority_queue, stream_cancel, tokio |
| `listener.rs`     | network   | trait/interface definition of a gossip listener   | libstd                                                     |
| `neighbor.rs`     | network   | representation of a peer                          | libstd                                                     |
| **`receiver.rs`** | network   | handling of incoming transactions                 | libstd, futures, log, rand, tokio                          |
| **`sender.rs`**   | network   | handling of outgoing transactions                 | libstd, futures, log, tokio                                |
| **`tangle.rs`**   | model     | the Tangle and Vertex datastructure               | libstd, lazy_static                                        |
| `transaction.rs`  | model     | the datastructure for an IOTA transaction         | libstd                                                     |
| `time.rs`         | util      | utily functions and macros to handle time         | libstd                                                     |
| `curl.rs`         | crypto    | implementation of the Curl hashfunction           | *none*                                                     |
| `ascii.rs`        | convert   | converting other representations to ascii         | libstd                                                     |
| `bytes.rs`        | convert   | converting other representations to bytes         | *none*                                                     |
| `number.rs`       | convert   | converting other representations to numbers       | *none*                                                     |
| `trits.rs`        | convert   | converting other representations to trits         | *none*                                                     |
| `tryte_string.rs` | convert   | converting other representations to tryte strings | libstd                                                     |
| `trytes.rs`       | convert   | converting other representations to trytes        | *none*                                                     |
| `luts.rs`         | convert   | contains lookup tables for faster conversions     | libstr, lazy_static                                        |

We will now in more detail describe how those highlighted modules function internally:

## Implementation of `Ictarus`
The basic representation of an Ictarus node looks like this:
```Rust
pub struct Ictarus {
    config: SharedConfig,
    runtime: Runtime,
    state: State,
    tangle: SharedTangle,
    neighbors: SharedNeighbors,
    listeners: SharedListeners,
    sending_queue: SharedSendingQueue,
    request_queue: SharedRequestQueue,
    kill_switch: (Trigger, Tripwire),
}
```
The `Ictarus` type holds its own **configuration** that allows to specify its overall behavior and define its neighbors, a **runtime** to start asynchronous tasks using poll-based futures, a **state**, has access to the **Tangle**, access to its **neighbors** to update their gossip stats, a **sending queue** to being able to issue transactions on its own, a **request queue** to being able to add requests on its own, and a **kill switch** mechanism to gracefully end all asychronous tasks.

The most important method on type is the **run** method. It will called from the **main** entry point of the application after reading the configuration file `ictarus.cfg` from the local file system.

```Rust
pub fn run(&mut self) -> Result<(), Box<std::error::Error>> {...}
```
It will do the following steps in that order:

1. Update the node state to `State::Initializing`.
2. Bind an UDP socket to the specified socket address. 
3. Spawn an asynchronous task responsible for listening on the socket for incoming IOTA transactions.
4. Spawn an asynchronous task responsible for sending and forwarding transactions to its neighbors as configured.
5. Spawn an asynchronous task for printing printing round stats to the terminal.
6. Update the node state to `State::Running`.

An important difference to the Java implementation is, that instead of using raw threads, the Rust implementation uses poll-based futures which are based on cooperative multitasking instead of preemptive multitasking where instead of context switching a processes voluntarily yield control when they cannot make any progress.

## Implementation of the `Receiver`

Flow chart of what how what spawned Receiver task is doing:

<img src="https://raw.githubusercontent.com/Alex6323/Ict-Architecture-In-Rust/master/images/Receiver.png" />

## Implementation of the `Sender`

Flow chart of what how what spawned Sender task is doing:

<img src="https://raw.githubusercontent.com/Alex6323/Ict-Architecture-In-Rust/master/images/Sender.png" />

## Implementation of the `Tangle`

The `Tangle` datastructure looks like this:

```Rust
pub struct Tangle {
    vertices_by_hash: HashMap<SharedKey81, (SharedVertex, Flags, MaybeTrunk, MaybeBranch)>,
    vertices_by_addr: HashMap<SharedKey81, HashSet<SharedVertex>>,
    vertices_by_tag: HashMap<SharedKey27, HashSet<SharedVertex>>,
}
```
Note: In the current implementation is it used as a store of transaction or as in-memory database. In a new implementation this will become what probably will be called `TransactionStore` as it better describes it functionality. A separate type will be created and given the name `Tangle`, which then is optimized to store and update references between vertices.

The most important method of this type is `attach_vertex`. It takes a key that is derived from the transaction hash, a `Vertex`, some metadata called `flags` and maybe a reference to a `trunk` vertex and a maybe a reference to a `branch` vertex. This is achieved by using an enumeration type, that can either hold `Some(vertex)` or `None` to represent that this vertex is not available at the moment and might be updated at a later point in time.

```Rust
pub fn attach_vertex(
        &mut self,
        tx_key: SharedKey81,
        vertex: Vertex,
        flags: Flags,
        trunk: MaybeTrunk,
        branch: MaybeBranch,
    ) -> SharedVertex {...}
```





