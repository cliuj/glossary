# Part 10 — Async

[← Concurrency](09-concurrency.md) · [Next: Modules, crates, and Cargo →](11-modules-crates-and-cargo.md)

This is the part of Rust that asks the most of you as a Go developer, and it
helps to know why up front. Go built async **into the runtime**: every goroutine
is already async I/O under the hood, so you write blocking-looking code and the
scheduler quietly parks you when you hit a syscall. Rust refuses to bundle a
runtime, so async is explicit, opt-in, and library-driven — you assemble it from
`async`/`.await`, a runtime crate you choose, and the ownership rules you already
fought in Parts 2, 3, and 8. The payoff is the same as goroutines (thousands of
concurrent I/O tasks on a handful of OS threads); the path there is bumpier.

---

## 1. Why async exists at all

The problem async solves is *many concurrent I/O-bound tasks on few OS
threads*. Part 9's `thread::spawn` gives you real OS threads — fine for a few,
wasteful for ten thousand idle connections, because each thread costs a stack
and the OS scheduler has to juggle them. Go's answer is goroutines: green
threads multiplexed onto a small thread pool, with the runtime swapping them at
I/O points. You never think about it.

Rust's `std` threads are *only* OS threads. There is no green-thread layer in
the standard library. Async is that layer — but you opt into it, and it lives in
crates, not the language runtime.

```rust
// One OS thread can drive thousands of these concurrently, because each
// one yields control at every .await instead of blocking the thread.
async fn handle(conn: Conn) {
    let req = conn.read().await;   // yields here while waiting on the socket
    let resp = process(req).await;
    conn.write(resp).await;        // yields here too
}
```

```go
// Go
// You'd just spawn a goroutine per connection. The runtime turns the
// blocking-looking reads/writes into yield points for you.
func handle(conn Conn) {
    req := conn.Read()      // looks blocking; runtime parks the goroutine
    resp := process(req)
    conn.Write(resp)
}
```

The Go version has no `async` keyword, no `.await`, no chosen runtime. That
machinery is all *there* — it's just inside the Go runtime instead of in your
source. Rust drags it into the open.

---

## 2. The centerpiece: futures are lazy

Here is the single most important contrast in this whole part, so internalize it
before anything else. **`go f()` starts running immediately. A Rust future does
nothing until it is awaited or spawned.**

An `async fn` (or an `async {}` block) does not run its body when you call it.
It returns a `Future` — a value describing work to be done, which sits inert
until something *polls* it. `.await` is what drives it. This is exactly the
laziness of iterators in Part 7 (`07-collections-iterators-closures.md`): an
iterator adapter does nothing until a consumer pulls on it, and a future does
nothing until `.await` pulls on it.

```rust
async fn say_hi() {
    println!("hi");
}

#[tokio::main]
async fn main() {
    let fut = say_hi(); // NOTHING prints here — fut is an inert Future
    println!("before");
    fut.await;          // only NOW does say_hi's body run
    // prints:
    // before
    // hi
}
```

```go
// Go
func sayHi() { fmt.Println("hi") }

func main() {
    go sayHi()             // STARTS running immediately, concurrently
    fmt.Println("before")
    time.Sleep(time.Millisecond) // crude wait so it can print
    // prints (order not guaranteed):
    // before
    // hi   (or "hi" then "before" — it's already off and running)
}
```

⚡ *Where the Go analogy breaks: `go f()` is eager and fire-and-forget — the
goroutine is scheduled the instant the statement runs. `f()` where `f` is an
`async fn` is lazy and inert — you've built a value, not started a task. The
verb that "starts" it is `.await` (drive it here) or `tokio::spawn` (hand it to
the runtime). There is no Rust syntax that means exactly what bare `go` means.*

> **Trap:** A future you never `.await` (and never `spawn`) simply never runs.
> No warning at runtime, just silence — `let _ = expensive().await;` is fine,
> but `let _ = expensive();` builds the future and drops it unrun. The compiler
> *does* warn with `#[must_use]` on most futures, so heed that lint.

---

## 3. You must pick a runtime

`std` ships no async runtime. Nothing in the standard library can *execute* a
future — `std` defines the `Future` trait and the `async`/`.await` syntax, and
stops there. To actually run futures you bring a runtime crate. The default
choice is **tokio**; alternatives like `async-std` and `smol` exist but tokio
dominates the ecosystem.

```rust
// Cargo.toml: tokio = { version = "1", features = ["full"] }

// The #[tokio::main] macro wraps your async main in a runtime, spins up a
// worker thread pool, and blocks on the top-level future.
#[tokio::main]
async fn main() {
    println!("running on tokio");
}
```

```go
// Go
// There is no equivalent, and here's why: the runtime is already linked
// into every Go binary. `func main()` is async-capable from byte one.
func main() {
    fmt.Println("running on the built-in runtime")
}
```

Why does Rust refuse to pick for you? The same zero-cost, no-forced-runtime
philosophy that keeps `std` usable on an embedded board with no OS. A web server
and a microcontroller shouldn't be forced to link the same scheduler. The cost
is real and you pay it on day one: the ecosystem splits along runtime lines (a
library written against tokio's I/O types won't run on `async-std` unmodified),
and *you* have to choose. For application code, the answer is almost always
"use tokio and move on."

---

## 4. Concurrent vs sequential awaits

Awaiting two futures back to back runs them **sequentially** — the second
doesn't start until the first finishes. To overlap them you combine them into
one future with `tokio::join!` (or `futures::future::join_all` for a dynamic
collection), then await that.

```rust
async fn fetch(url: &str) -> usize {
    // pretend this hits the network; returns the body length
    url.len()
}

#[tokio::main]
async fn main() {
    // SEQUENTIAL: total time = a + b
    let a = fetch("https://a.example").await;
    let b = fetch("https://b.example").await;

    // CONCURRENT: both run at once on one thread, total time = max(a, b)
    let (c, d) = tokio::join!(
        fetch("https://c.example"),
        fetch("https://d.example"),
    );

    // dynamic fan-out over a Vec of futures:
    use futures::future::join_all;
    let urls = ["https://e.example", "https://f.example"];
    let results: Vec<usize> = join_all(urls.iter().map(|u| fetch(u))).await;

    println!("{a} {b} {c} {d} {results:?}");
}
```

```go
// Go
// The Go shape is spawn-then-wait: launch goroutines, collect with a
// WaitGroup or channels. Spawning is the concurrency; there is no "join!".
func main() {
    var wg sync.WaitGroup
    results := make([]int, 2)
    urls := []string{"https://e.example", "https://f.example"}
    for i, u := range urls {
        wg.Add(1)
        go func(i int, u string) {
            defer wg.Done()
            results[i] = fetch(u)
        }(i, u)
    }
    wg.Wait()
    fmt.Println(results)
}
```

⚡ *Where the Go analogy breaks: in Go, concurrency comes from `go` — the moment
you spawn, work is in flight, and `WaitGroup` only collects the results. In
Rust, `join!` is where the concurrency happens: the futures passed to it run
**concurrently but on the same task/thread**, interleaved at their `.await`
points. It's cooperative multitasking inside one future, not N goroutines. For
true parallelism across threads you need §5's `tokio::spawn`.*

---

## 5. `tokio::spawn`: the closest thing to `go`

`tokio::spawn(future)` hands a future to the runtime as an independent task,
scheduled across the worker threads — this is the real analogue of `go f()`. It
returns a `JoinHandle<T>` you can `.await` to get the task's result (wrapped in a
`Result`, because the task might panic).

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // runs concurrently, possibly on another worker thread
        fetch("https://x.example").await
    });

    // ... do other work here, concurrently ...

    let n: usize = handle.await.unwrap(); // .await yields the task's value
    println!("{n}");
}

async fn fetch(url: &str) -> usize { url.len() }
```

```go
// Go
func main() {
    ch := make(chan int, 1)
    go func() { ch <- fetch("https://x.example") }()
    // ... other work ...
    n := <-ch
    fmt.Println(n)
}
```

The catch ties straight back to Part 9. A spawned task may move to another
thread, so its future must be `Send + 'static`: everything it captures has to be
sendable across threads and owned (no borrows of local variables). That
`'static` bound is why you reach for `Arc` to share data into a spawned task,
exactly as you would to share across `thread::spawn` in
`09-concurrency.md` (§Send/Sync).

> **Trap:** `tokio::spawn(async { let r = &local; ... })` won't compile if
> `local` is a stack variable — the task can outlive the function, so it can't
> hold a borrow into the function's frame. The fix is `move` the data in (often
> via `Arc::clone`), not borrow it. The error is the familiar lifetime/`'static`
> complaint from Part 3, now showing up because tasks are detached.

### Racing with `select!`

`tokio::select!` drives several futures at once and acts on whichever finishes
first, cancelling the rest. It is the direct map of Go's `select` over channels.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        _ = sleep(Duration::from_millis(50)) => println!("timed out"),
        n = fetch("https://x.example")        => println!("got {n}"),
    }
}

async fn fetch(url: &str) -> usize { url.len() }
```

```go
// Go
func main() {
    done := make(chan int)
    go func() { done <- fetch("https://x.example") }()
    select {
    case <-time.After(50 * time.Millisecond):
        fmt.Println("timed out")
    case n := <-done:
        fmt.Println("got", n)
    }
}
```

---

## 6. Async meets ownership again

A future is a state machine that *holds its borrows across `.await` points*.
Between awaits the function is suspended, but its captured references stay alive
the whole time — so every lifetime and `Send` rule from Parts 2, 3, and 8 comes
back, now stretched across suspension points. This is why `Arc` and `Mutex`
reappear constantly in async code.

There are two `Mutex` types and choosing wrong is a classic trap.
`std::sync::Mutex` (the one from Part 9) is fine in async code **as long as you
don't hold its guard across an `.await`** — the guard isn't `Send`, and blocking
on a std mutex blocks the whole worker thread. `tokio::sync::Mutex` is
async-aware: its lock is itself a future you `.await`, and the guard *can* be
held across awaits.

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0u64));

    let mut handles = vec![];
    for _ in 0..4 {
        let c = Arc::clone(&counter);
        handles.push(tokio::spawn(async move {
            let mut guard = c.lock().await; // .await — async-aware lock
            *guard += 1;                    // guard held across no await here
        }));
    }
    for h in handles { h.await.unwrap(); }
    println!("{}", *counter.lock().await); // prints: 4
}
```

> **Trap:** Reach for `std::sync::Mutex` by default; switch to
> `tokio::sync::Mutex` only when you genuinely must hold the lock across an
> `.await`. The async mutex is heavier, and holding *any* lock across an await
> is a contention smell. Holding a `std::sync::Mutex` guard across an `.await`,
> by contrast, often won't even compile (the guard isn't `Send`), which is the
> compiler steering you right.

The practical rule: short critical section, no await inside it →
`std::sync::Mutex` + `Arc`. Need to await while holding the lock →
`tokio::sync::Mutex` + `Arc`. Both wrap in `Arc` for the same `'static` reason
spawned tasks demand.

---

## 7. Function color: async is contagious

Here is the genuine ergonomic tax, stated plainly. **You cannot `.await` inside
a non-`async` function.** `.await` only works in an async context, so the moment
one function needs to await, its callers tend to become `async` too, and the
async-ness spreads up the call tree. People call this the "colored functions"
problem: sync and async are two colors, and a sync function can't transparently
call an async one.

```rust
fn sync_caller() {
    let n = fetch("https://x.example").await; // ERROR: `.await` only allowed
                                               // inside async fn / async block
}
async fn fetch(url: &str) -> usize { url.len() }
```

To call async from sync you must *enter the runtime* explicitly — e.g.
`tokio::runtime::Runtime::new()?.block_on(fut)` — which blocks the current thread
until the future completes. That's a real boundary, not a free conversion.

```go
// Go
// There is no equivalent, and here's why: Go has no function colors. ANY
// function can do I/O and yield to the scheduler. fetch is just a function;
// you call it from anywhere, sync or "concurrent", with no keyword.
func syncCaller() {
    n := fetch("https://x.example") // just a call. No await, no color.
}
```

⚡ *Where the Go analogy breaks: this whole section has no Go counterpart. Go's
runtime makes every function implicitly yield-capable, so "blocking" and
"async" aren't visible in the type system at all. Rust pushed async into the
type system to keep it zero-cost and runtime-agnostic, and the price is that
async colors your function signatures. It's a deliberate trade — honesty about
it is more useful than pretending the ergonomics match Go's.*

---

## 8. Streams and Pin, briefly

A `Stream` is the async analogue of `Iterator` (Part 7): instead of `next() ->
Option<T>`, it yields `Option<T>` *over time*, and you pull values with
`.next().await` (via the `StreamExt` trait from the `futures`/`tokio-stream`
crates). Think "a channel you can `for`-each-await over" — it's how you model a
sequence of incoming messages, paginated API results, or socket frames.

`Pin` is the advanced bit. Because a future is a self-referential state machine,
the runtime needs a guarantee it won't be moved in memory while suspended; `Pin`
encodes that guarantee in the type system. You'll see `Pin<Box<dyn Future>>` in
signatures and can mostly treat it as boilerplate. It is genuinely advanced —
when you need to write a manual `Future` or work with `Pin` directly, reach for
the async deep-dives rather than reasoning about it from first principles here.

---

## One-sentence mental model

Async in Rust is the green-thread layer Go bakes into its runtime, but unbundled
and made explicit: an `async fn` builds a **lazy** `Future` that does nothing
until a runtime you chose (tokio) drives it via `.await` (sequential),
`join!`/`select!` (concurrent on one task), or `tokio::spawn` (a real `go`-like
task that must be `Send + 'static`) — and because futures hold their borrows
across every `.await`, all the ownership, `Arc`, and `Send` rules from Parts 2,
3, 8, and 9 come back, plus the new tax that `async` colors your functions.

---

[← Concurrency](09-concurrency.md) · [Next: Modules, crates, and Cargo →](11-modules-crates-and-cargo.md)
