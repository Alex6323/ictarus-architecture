# Abstract

This document is about the first attempt to port the official Java Ict implementation to Rust. It was mainly my intention to get a deeper understanding of Ict on one hand, and to find an interesting project to further deepen my Rust programming abilities. I never thought it would be seriously considered by the IOTA foundation as a way to actually take something from it. I hope that it will be helpful to the people involved writing a new Ict node that will be troth: stable, safe, and performant and helps IOTA to become a standardized protocol for the IoT.

# Why Rust?

According to Wikipedia: "[Rust](https://en.wikipedia.org/wiki/Rust_(programming_language) is a multi-paradigm systems programming language focused on safety, especially safe concurrency, which is syntactically similar to C++, but is designed to provide better memory safety while maintaining high performance." 

Some of Rust's most notable, yet not unique features are:
	* compiled language
	* no automated garbage collection
	* statically typed (but uses type inference to reduce boilerplate)
	* forced error handling
	* immutable variables by default
	* functional programming patterns (iterators, generators, closures)
	* built-in support for tests and benchmarks
	* easy toolchain management using the tool 'rustup'
	* easy project management using the tool 'cargo'
	* central package repository (crates.io)

While those are nice-to-have features, they aren't defining enough to truly set Rust apart from the manifold language landscape. So what is unique about Rust? 

Well, to make a long story short: it achieves to overcome a seemingly nature-given dichotomy between being a safe programming language and being a performant one. Until recently you had to make a trade-off decision. Going with something garbage collected like Java and be relatively safe but pay for it with some performance penalties here and there, or go with something performant like C/C++ and expect to spend some time on tracking down some ugly concurrency bugs and making sure you have a good review system in place at all times. With Rust you no longer have to compromise one over the other, or make any trade-offs in that regard. How does it do that? First and foremost by making *zero-cost abstractions* one of its core principles just like C++. What that basically means is that: 
	* You don't pay for something that you don't use, and
	* if you use something, then you couldn't handcode it any better.
There have been abstractions that were part of the language once e.g. green threads (that other languages like Go and Elixir/Erlang make heavy use of), that were excommunicated, because they didn't adhere to this principle. 

On the other hand Rust brings something completely new to the table, which indeed makes it truly unique among all the thousands of languages. And that is what is commonly referred to as *Ownership and Borrowing*. It turns out that those simple concepts taken from our day-to-day lives are enough to make whole classes of bugs in Rust just impossible, and that without compromising performance at all. In fact Rust can even achieve better performance that C++. If you want to know more about those concepts I recommand start reading [this](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html). 

I lied a bit though: it is actually not for free: as a developer you have to rewire your brain a bit to learn those new concepts and experience something that is already meme'd as "fighting the borrow-checker". In a very good presentation I saw about Rust the presenter made the argument that you will become a better programmer even in other languages by knowing Rust as you'll start wondering why some code in language X compiles that you know the Rust compiler would never allow you to write.

I thought it might be necessary to add this little paragraph to make some architetural design decisions I had to make to satisfy the compiler better understandable. To some not familiar with the Rust idiosyncrasies some of those might be looking weird. Having said all that I want to make clear that I am not a Rust expert yet. As with all mastery it takes time and commitment to eventually have visited almost each rabbit hole there is in a certain domain providing a sufficient level of complexity.

# About Ict

The *I*ota *c*ontrolled agent*t* (Ict) is a very flexible, module-based IOTA node that can be as light-weight or as heavy-weight as the usecase requires it to be. It achieves this by facilitating the IOTA eXtending Interface (IXI). The node is specifically designed for the Internet of Things (IoT) and its diverse use cases. Targeted platforms are mainly single-board computers (SBCs) like Raspberry Pis and microcontrollers (MCUs) like the Cortex-M4 and upwards, but also more powerful platforms will be supported. To achieve this the node software is logically and programmatically separated into a core component (Ict core) and modules implementing IXI. Modules are therefore usually referred to as IXI modules. While the core implements the IOTA gossip protocol and is responsible for establishing a friend-to-friend P2P network, buffering a certain amount of gossip, forming and updating the Tangle, and functioning as a gateway between the p2p network and its attached modules, the modules themselves define the actual functionality and the behavior of the node. Some module might turn the node into an access point for a p2p chat application or it might turn the node into a data-specific permanode that filters and diverts data into a database. The more resources available on the target platform the node is running on, the more modules can be can be hosted by the core and the more powerful the node becomes overall. On very constrained devices however it is possible to target a very specific behavior. It all depends on the use case.

