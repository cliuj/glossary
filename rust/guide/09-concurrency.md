# Part 9 — Concurrency

[← Smart pointers and interior mutability](08-smart-pointers-and-interior-mutability.md) · [Next: Async →](10-async.md)

This is the part where the two languages most visibly trade places. Go made its
name on **easy** concurrency: `go f()` is one keyword, goroutines are cheap, and
channels are built into the language. Rust's pitch is **safe** concurrency —
"fearless concurrency," meaning the compiler proves at build time that your
threaded code has no data races, the bug class Go's `-race` detector can only
*sometimes* catch at runtime. Both claims are true; this part is about what each
costs and where the lines are drawn.

---

## 1. `thread::spawn`: a real OS thread

The direct analog of `go f()` is `std::thread::spawn`, which takes a closure and
runs it on a new thread. The first thing to internalize: **these are different
animals.** A goroutine is a green thread multiplexed onto an OS thread by Go's
M:N scheduler and its built-in runtime — spawning a million of them is normal.
`thread::spawn` creates one actual OS thread, with a real stack and a kernel
scheduler entry. Spawning a million of *those* will melt your machine.

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..=3 {
            println!("worker: {i}");
        }
    });

    handle.join().unwrap(); // wait for the thread to finish
    println!("done");
}
```

```go
// Go
func main() {
    done := make(chan struct{})
    go func() {
        for i := 1; i <= 3; i++ {
            fmt.Println("worker:", i)
        }
        close(done)
    }()
    <-done // Go has no JoinHandle; you wait via a channel or WaitGroup
    fmt.Println("done")
}
```

`spawn` returns a `JoinHandle<T>`. Calling `.join()` blocks until the thread
finishes and hands back its return value as a `Result` (the `Err` case is "the
thread panicked"). Go has no built-in join handle — you reach for a channel, a
`sync.WaitGroup`, or `errgroup`. Rust folds "wait for it" and "collect its
result/panic" into the one handle.

⚡ *Where the Go analogy breaks: `go f()` is fire-and-forget by default — the
runtime owns the goroutine and you synchronize separately if you care. A Rust
`JoinHandle` is a value you own; if you drop it without joining, the thread is
**detached** and keeps running, but you've lost the result and the panic. There's
no scheduler hiding the cost: one `spawn`, one OS thread.*

The lightweight, millions-of-tasks model — green threads on an M:N scheduler,
the thing that actually feels like goroutines — does exist in Rust, but it's
**async** (`async`/`await` + a runtime like Tokio), which is Part 10
(`10-async.md`). `std::thread` is deliberately the heavyweight, runtime-free
option. Keep the two separate in your head.

---

## 2. Why the borrow checker follows you into the thread

Here's the wrinkle that has no Go counterpart. A spawned thread may outlive the
function that spawned it, so any reference it captures must be guaranteed valid
for as long as the thread might run — which the compiler can't bound, so it
demands `'static` (Part 3, `03-lifetimes.md`) or full ownership. Borrowing a
local and handing the borrow to a thread is rejected:

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];

    let handle = thread::spawn(|| {
        println!("{data:?}"); // ERROR: closure may outlive `data`, which it borrows
    });

    handle.join().unwrap();
}
```

The fix is `move` — the keyword from Part 7
(`07-collections-iterators-closures.md`). It forces the closure to **take
ownership** of `data` instead of borrowing it, so the thread owns the `Vec`
outright and there's nothing to dangle:

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];

    let handle = thread::spawn(move || { // `move`: data is OWNED by the thread now
        println!("{data:?}"); // prints: [1, 2, 3]
    });

    handle.join().unwrap();
    // data is gone here — it moved into the thread.
}
```

```go
// Go
func main() {
    data := []int{1, 2, 3}
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {            // captures `data` by reference; compiles, runs, no fuss
        defer wg.Done()
        fmt.Println(data)
    }()
    wg.Wait()
    fmt.Println(data) // still usable — Go shares it freely, GC keeps it alive
}
```

This is the same `move` you used to send a closure past its scope, now load-
bearing for thread safety. Go's goroutine captures `data` by reference and the GC
keeps it alive as long as anything points at it; the spawning function and the
goroutine share the slice with zero ceremony. Rust refuses to *share* a stack
borrow with a thread because there's no GC to underwrite the lifetime — you
either move ownership in (this section) or use a scoped thread (§7) or shared-
ownership pointer (§4).

> **Trap:** `move` moves **everything** the closure captures, not just the one
> value you were thinking about. If the closure also reads an unrelated `String`,
> that gets moved in too and is unusable afterward. When you need the thread to
> own one thing but the parent to keep another, clone the parent's copy before
> the `move`, or restructure with `Arc` (§4) / `thread::scope` (§7).

---

## 3. Channels: `mpsc`, and how they differ from Go's

Go's channel is the headline concurrency primitive, so this maps closely. Rust's
standard channel lives in `std::sync::mpsc`. `channel()` returns a `(tx, rx)`
pair — a transmitter and a receiver. You `tx.send(x)` and `rx.recv()`:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        for i in 1..=3 {
            tx.send(i).unwrap(); // send MOVES i into the channel
        }
        // tx dropped here -> the channel closes
    });

    // rx is an iterator: yields each value, ends when the channel closes.
    for received in rx {
        println!("got {received}"); // prints: got 1 / got 2 / got 3
    }
}
```

```go
// Go
func main() {
    ch := make(chan int)
    go func() {
        for i := 1; i <= 3; i++ {
            ch <- i // sends a COPY of i
        }
        close(ch) // you close it explicitly
    }()
    for received := range ch { // range ends when the channel is closed
        fmt.Println("got", received)
    }
}
```

The shapes rhyme — send into one end, range over the other, "channel closes ⇒
loop ends" — but three differences matter:

- **`mpsc` is multi-producer, single-consumer.** The name says it: many `tx`,
  exactly one `rx`. Go channels are MPMC — any number of goroutines can both send
  and receive on the same channel. To fan *in* in Rust you clone `tx`; to fan
  *out* to multiple consumers you need a different tool (the `crossbeam-channel`
  crate gives you MPMC).
- **`send` moves ownership of the value into the channel.** Once you `send(x)`,
  `x` is gone from the sender — the receiver now owns it. Go copies the value, so
  both sides have their own. The move is *why* channels are race-free: there's no
  shared value left to race over (this is the Part 2 ownership model doing
  concurrency work).
- **Closing is implicit.** Dropping the last `tx` closes the channel (the `rx`
  iterator ends). There's no `close(ch)`; you just let the senders go out of
  scope. Sending on a Go channel after `close` panics; in Rust there's no way to
  send on a dropped channel because you no longer have a `tx`.

Multiple producers via `tx.clone()`:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    for id in 0..3 {
        let tx = tx.clone(); // each thread gets its own sender handle
        thread::spawn(move || tx.send(format!("from {id}")).unwrap());
    }
    drop(tx); // drop the ORIGINAL too, or the channel never closes

    for msg in rx {
        println!("{msg}"); // prints the three messages, in some order
    }
}
```

> **Trap:** the `rx` loop only ends when **every** `tx` (including the original
> and all clones) has been dropped. Forget the `drop(tx)` above and `for msg in
> rx` hangs forever, because the original `tx` is still alive in `main`. This is
> the Rust version of "forgot to `close(ch)`," except the trigger is ownership,
> not an explicit call.

`mpsc::channel()` is **unbounded** (`send` never blocks, like a buffered Go
channel with infinite capacity). `mpsc::sync_channel(n)` is bounded — `send`
blocks when `n` items are in flight, the analog of Go's `make(chan T, n)` with
backpressure. For anything richer (MPMC, `select` over multiple channels), the
`crossbeam-channel` crate is the de-facto standard.

---

## 4. Shared state: `Arc<Mutex<T>>`

Channels move ownership around; sometimes you genuinely want several threads
touching *one* piece of mutable state. That's `Arc<Mutex<T>>`, and it's worth
seeing it as the exact composition of two things from Part 8
(`08-smart-pointers-and-interior-mutability.md`):

- `Mutex<T>` gives **interior mutability** guarded by a lock — mutate through a
  shared reference, but only while holding the lock.
- `Arc<T>` gives **shared ownership** — an atomically reference-counted pointer,
  so multiple threads can each own a handle to the same value.

You need *both*. A bare `Mutex<T>` has a single owner and can't be handed to
several threads; a bare `Arc<T>` is shared but immutable. Composed,
`Arc<Mutex<T>>` is "shared ownership of a thing you're allowed to mutate under a
lock."

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0)); // shared, lockable integer
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter); // bump the refcount; cheap pointer copy
        let handle = thread::spawn(move || {
            let mut n = counter.lock().unwrap(); // acquire the lock -> a guard
            *n += 1;
            // guard drops at end of scope -> lock RELEASED automatically
        });
        handles.push(handle);
    }

    for h in handles {
        h.join().unwrap();
    }

    println!("{}", *counter.lock().unwrap()); // prints: 10
}
```

```go
// Go
func main() {
    var mu sync.Mutex
    counter := 0
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            defer mu.Unlock() // you must remember the defer
            counter++
        }()
    }
    wg.Wait()
    fmt.Println(counter) // 10
}
```

The lock ergonomics are the headline contrast. In Go the mutex and the data it
protects are *separate* — `mu` guards `counter` only by convention, and you write
`mu.Lock(); defer mu.Unlock()` and trust yourself never to touch `counter`
without the lock. In Rust the data lives **inside** the `Mutex<T>`: the only way
to reach the `0` is `.lock()`, which returns a `MutexGuard` — a smart pointer
(Part 8) that derefs to the inner value. You literally cannot access the data
without holding the lock, because the data is behind the lock.

And you cannot forget to unlock. The guard releases the lock when it's dropped
(RAII — Part 8's `Drop`), which happens automatically at end of scope. There's no
`defer mu.Unlock()` to remember and no path where an early return skips it.

⚡ *Where the Go analogy breaks: Go's mutex protects data by discipline — nothing
stops a goroutine from reading `counter` without `mu.Lock()`, and the `-race`
detector only flags it if that path happens to run under instrumentation. Rust
makes the lock the *only door* to the data, enforced at compile time. "Forgot to
take the lock" and "forgot to release the lock" are both unrepresentable, not
just discouraged.*

`RwLock<T>` is the read-write variant — many concurrent readers (`.read()`) or
one writer (`.write()`), the analog of Go's `sync.RWMutex`. Same guard-on-drop
mechanics.

### Poisoning

One genuinely new concept: **lock poisoning**. If a thread panics while holding a
`Mutex`, the lock is marked *poisoned*, and every subsequent `.lock()` returns an
`Err`. That's why you see `.lock().unwrap()` everywhere — the `unwrap` is
propagating a potential poison error. The reasoning: a panic mid-mutation may
have left the protected data in a half-updated, inconsistent state, so Rust
refuses to silently hand it out.

```rust
let guard = counter.lock().unwrap();
//                         ^^^^^^^^ panics if another thread poisoned this lock
```

Go has no equivalent — a goroutine that panics while holding `mu` (and doesn't
recover) typically crashes the process, and an unlocked-via-`defer` mutex is just
reusable afterward regardless of what state the data is in. Rust's poisoning is a
deliberate "a panic during mutation is suspicious; make the next acquirer
acknowledge it." You can recover the guard with `e.into_inner()` if you know the
data is fine.

---

## 5. `Send` and `Sync`: the machinery behind "fearless"

Now the centerpiece — the part that explains *why* all of the above is safe, and
the single biggest difference from Go's model. Everything in this part (spawning,
channels, `Arc<Mutex<T>>`) is held together by two **marker traits** the compiler
checks automatically:

- **`Send`** — a type is safe to **move** to another thread. (`tx.send`,
  `thread::spawn`'s closure, all require their values to be `Send`.)
- **`Sync`** — a type is safe to **share** by reference (`&T`) across threads.
  `T` is `Sync` if and only if `&T` is `Send`.

These are *auto traits*: the compiler derives them structurally — a struct is
`Send` if all its fields are `Send`, and so on. You almost never write them; you
just feel them when something *isn't* one and the compiler stops you. That stop
is fearless concurrency. The two canonical cases, both tracing straight back to
Part 8:

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let shared = Rc::new(5);
    thread::spawn(move || {
        println!("{}", shared); // ERROR: `Rc<i32>` cannot be sent between threads
    });                         // (Rc is !Send)
}
```

`Rc` (Part 8) uses a **non-atomic** reference count for speed — two threads
bumping it at once would race and corrupt the count, leaking or double-freeing.
So `Rc` is **`!Send`**: the compiler simply won't let it cross a thread boundary.
`Arc` uses an *atomic* count, which is exactly why §4 used `Arc` and not `Rc` — it
*is* `Send` (and `Sync`). The naming is the tell: **A**tomic **R**eference
**C**ounted.

Likewise `RefCell<T>` (Part 8) does its borrow-tracking with plain non-atomic
flags, so two threads could both "borrow mutably" at once and violate aliasing —
`RefCell` is **`!Sync`**. `Mutex<T>` does the same job (interior mutability) with
real locking, so it *is* `Sync`. That's the deep reason §4 reached for `Mutex`
and not `RefCell` to share mutable state.

```go
// Go — the comparison that matters most
//
// There is NO compile-time Send/Sync. Any value can be shared with any
// goroutine; the compiler never objects. This compiles and runs fine:
func main() {
    counter := 0
    for i := 0; i < 1000; i++ {
        go func() { counter++ }() // DATA RACE — unsynchronized shared write
    }
    time.Sleep(time.Second)
    fmt.Println(counter) // some number <= 1000, nondeterministic, UB
}
// `go run -race` MIGHT report this — but only if the racing accesses happen
// to execute and interleave during the instrumented run. It is a runtime,
// sampling detector. It finds races it observes, not races that exist.
```

This is the whole ballgame. In Go, the `counter++` race above compiles, runs, and
produces a wrong answer; you find it only if `-race` happens to witness the bad
interleaving in a test that happens to exercise it. In Rust the equivalent —
sharing a non-`Sync` value, or moving a non-`Send` one — is a **compile error**,
every time, on every path, before the program ever runs. Rust converts an entire
category of "ran a thousand times in CI, blew up in production once" bugs into
red squiggles in your editor.

⚡ *Where the Go analogy breaks: this is the breaking point, the reason the parts
trade ergonomics for guarantees. Go's model is "share anything, synchronize by
convention, detect violations at runtime if you're lucky." Rust's is "the type
system tracks which values can cross or be shared across threads, and proves the
absence of data races at compile time." `Send`/`Sync` are why `Arc<Mutex<T>>`
just works and why `Rc`-in-a-thread just won't — and there is no Go construct
that gives you the compile-time half of that.*

> **Trap:** don't read "no data races" as "no concurrency bugs." Rust still lets
> you write **deadlocks** (lock A then B in one thread, B then A in another),
> **livelocks**, and logic races over *when* things happen. `Send`/`Sync`
> eliminate the *data race* — unsynchronized memory access — specifically. A
> deadlocked `Mutex` is a perfectly type-safe hang, same as in Go.

---

## 6. Atomics: the lock-free option

For a single shared counter or flag, a `Mutex` is heavier than you need. The
`std::sync::atomic` types — `AtomicUsize`, `AtomicBool`, `AtomicI64`, and friends
— give you lock-free reads and writes, the direct analog of Go's `sync/atomic`.

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicUsize::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            counter.fetch_add(1, Ordering::SeqCst); // atomic +1, no lock
        }));
    }
    for h in handles {
        h.join().unwrap();
    }
    println!("{}", counter.load(Ordering::SeqCst)); // prints: 10
}
```

```go
// Go
func main() {
    var counter atomic.Int64
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() { defer wg.Done(); counter.Add(1) }()
    }
    wg.Wait()
    fmt.Println(counter.Load()) // 10
}
```

The one extra knob versus Go is the explicit `Ordering` argument — the memory
ordering (`SeqCst` is the strongest and the safe default; `Relaxed`, `Acquire`,
`Release` are weaker, faster, and require you to actually understand the memory
model). Go's atomics are sequentially consistent and don't make you choose. When
in doubt, `SeqCst` matches Go's behavior. Note atomics are `Sync`, so you still
wrap in `Arc` to share ownership across threads — same composition as §4, minus
the lock.

---

## 7. `thread::scope`: borrowing without `Arc`

Sections 2 and 4 left an ergonomic gap: to share local data you had to either
`move` it (giving it away) or wrap it in `Arc` (heap allocation + refcounting).
Often you just want a few threads to borrow some stack data, run, and *finish
before the function returns* — at which point borrowing would be perfectly safe.
`thread::scope` (stable since Rust 1.63) expresses exactly that.

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3, 4, 5, 6];

    thread::scope(|s| {
        s.spawn(|| {                       // borrows `data` — no move, no Arc
            let sum: i32 = data[..3].iter().sum();
            println!("first half: {sum}");  // prints: first half: 6
        });
        s.spawn(|| {
            let sum: i32 = data[3..].iter().sum();
            println!("second half: {sum}"); // prints: second half: 15
        });
        // scope() does NOT return until every spawned thread has joined.
    });

    println!("{data:?}"); // prints: [1, 2, 3, 4, 5, 6] — still ours, never moved
}
```

The guarantee that makes this sound: `scope` **joins all its threads before it
returns**, so the borrows provably can't outlive `data`. The borrow checker knows
this and allows the plain `&data` capture that §2 forbade for `thread::spawn`. No
`move`, no `Arc`, no `'static` — just borrowed local data, exactly as you'd
expect coming from Go, except still race-checked (a `&mut data` shared across two
scoped threads is still a compile error).

This is the modern, ergonomic default for "run some work over local data in
parallel and wait for it." It's the closest Rust gets to the casualness of
launching a couple of goroutines over a shared slice and waiting on a
`WaitGroup` — with the data race ruled out at compile time rather than hoped
away.

---

## One-sentence mental model

`thread::spawn` is a real OS thread (goroutines are green and async, Part 10);
`move` and `'static` follow the borrow checker into every thread; channels
(`mpsc`) move ownership instead of copying so there's nothing left to race over;
`Arc<Mutex<T>>` is just shared-ownership plus locked-interior-mutability composed
from Part 8 — and the `Send`/`Sync` marker traits are the compiler-checked
guarantee that turns Go's runtime, sampling `-race` detector into a build error
you cannot ship past.

---

[← Smart pointers and interior mutability](08-smart-pointers-and-interior-mutability.md) · [Next: Async →](10-async.md)
