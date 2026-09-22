---
title: "Mutex Identity: Why C++ Won't Move Them, and What Rust Does Instead"
tags: [c++, rust, concurrency, synchronization]
---

# Mutex Identity: Why C++ Won't Move Them, and What Rust Does Instead

I needed one mutex per Host/GPU doorbell channel. Each channel’s register
write can raise an IRQ on the other side (host CPU versus the GPU’s
internal microcontroller), so a read-modify-write plus that side effect
has to be atomic with respect to other threads. I reached for
`std::vector<std::mutex>` — one lock per channel — and discovered that
`std::mutex` is neither copy constructible nor move constructible.

That rule is easy to recite from the standard. The interesting question
is what would go wrong if those operations existed, and how Rust can
treat a mutex as movable without inviting the same disasters.

## If C++ allowed copy

A copy must mean something. Both readings break mutual exclusion:

- **Independent state.** The copy is a new lock. Code that copied “the
  channel mutex” would look synchronized while two threads locked
  different objects and raced on the same doorbell.
- **Shared underlying state.** That is shared ownership, not a copy.
  Destruction and ownership of the OS primitive become unclear, and the
  operation no longer matches C++ copy semantics.

Mutual exclusion is about a single identity. Copying that identity is the
wrong model, so the type forbids it.

## If C++ allowed move

Move looks reasonable by analogy with `std::unique_ptr`, but a mutex is
tied to concurrent waiters and lock owners:

- Another thread may already hold a `std::lock_guard` /
  `std::unique_lock` or wait on that mutex. Moving the object transfers
  (or abandons) the underlying state while those threads still refer to
  the old identity. Unlock then targets the wrong object, or a
  moved-from shell.
- Implementations and wait queues often treat the mutex’s address as
  part of its identity. Relocating it invalidates waiters and any
  pointers or references other threads still hold.
- Containers would make this routine: growing a `std::vector` would move
  every element and silently break concurrent lockers.

So immovability is deliberate. “The mutex for channel *i*” must keep one
address and one meaning for its whole lifetime. `std::vector<std::mutex>`
does not compile because reallocation needs movable elements.

Practical alternatives: `std::deque<std::mutex>` or
`std::list<std::mutex>` (growth does not relocate existing elements),
`std::vector<std::unique_ptr<std::mutex>>`, a fixed
`std::array<std::mutex, N>` or an owning array allocated once, or
in-place construction in a structure whose elements are never relocated.

## Rust: the mutex *is* movable

In Rust, `std::sync::Mutex<T>` is movable. That is not because moves are
magically safe while the lock is held. It is because safe Rust makes
“move while locked / while shared” unrepresentable.

1. **Moves need exclusive ownership.** A move is only allowed when
   nothing else is borrowing the value. `MutexGuard` borrows the
   `Mutex` for its lifetime. While any guard exists, moving the
   `Mutex` is a compile error. You never relocate a mutex that still
   has a lock owner or waiters tied to that borrow.

2. **Shared access goes through shared ownership.** Cross-thread use is
   normally `Arc<Mutex<T>>`. Threads share the `Arc`; the `Mutex`
   itself lives on the heap and does not move when you clone the
   `Arc`. Moving the `Mutex` out again requires unique ownership
   (`Arc::try_unwrap` / `get_mut`), which fails while other clones
   exist.

3. **The dangerous C++ case is structurally hard.** C++ freely allows
   `mutex*` / `mutex&` across threads with no lifetime checker, so
   `vector` reallocation can move elements under concurrent lockers.
   In Rust, either borrowers / `Arc` clones still exist (move
   rejected), or you have unique ownership and no guard (no other
   thread can legally be locking that object), so relocating an
   unlocked mutex is fine.

Caveats: `unsafe` plus raw pointers can recreate the C++ hazard. Types
that must not move even when unused use `Pin` / `!Unpin`; `Mutex` is
not in that camp. C++ keeps `std::mutex` immovable because it cannot
enforce Rust’s discipline, so it closes the whole class of bugs with a
blunt rule.

## Why `std::sync::MutexGuard` is `!Send`

If the guard were `Send`, you could lock on thread A, move the guard to
thread B, and unlock on B when the guard drops.

Many OS mutexes treat the locking thread as the owner. Windows
`CRITICAL_SECTION` and several pthread configurations require unlock on
the same thread that locked (or define foreign unlock as an error).
Making `MutexGuard: Send` would make that pattern easy in safe Rust, so
`std` marks the guard `!Send` even when `T` itself is `Send`.

Exclusive access to the data is not the main issue. While the guard
exists, only one thread can touch the protected `T`. The hazard is the
unlock side of the OS primitive: ownership is thread-bound on targets
`std` must support.

You can still share the mutex with `Arc<Mutex<T>>`. Each thread locks
locally, unlocks on the same thread, and drops the guard before moving
it elsewhere.

## `parking_lot`: unlock need not be thread-bound

`parking_lot::Mutex` is not a thin wrapper around `pthread_mutex` /
`SRWLOCK`. Rough shape:

- A small atomic state word (locked bit, and a bit meaning “there are
  parked waiters”).
- Fast path: uncontended lock/unlock is a few atomic ops, no syscall.
- Slow path: the thread parks in `parking_lot`’s global table. Wait
  queues are keyed by the lock’s address. Unlock clears the lock and
  unparks a waiter (often via futex / `WaitOnAddress` / similar).
- There is no OS-level “owning thread” that unlock must match.

Unlock is “release the atomic state + wake a waiter,” which any thread
may do. That is why `parking_lot::MutexGuard` can be `Send`. The crate
may spin briefly before parking; the ownership model matches a token in
an atomic, not “thread id X holds this.”

## Would a spinlock’s guard be `Send`?

Yes — with a pure `AtomicBool` spinlock, the guard *can* be `Send`.
Nothing in that protocol forbids it.

- Lock: `compare_exchange(false → true)` with Acquire (spin on failure).
- Unlock: `store(false)` with Release.
- Guard: borrows the mutex, unlocks on `Drop`.

No thread id is recorded. Unlocking on another thread is just another
Release store. Moving the guard from A to B and dropping on B correctly
releases the lock. Exclusive access to `T` moves with the guard, so
`Guard: Send` would normally require `T: Send` — about the data, not
about the bool.

That does not make a forever-spinlock a good general-purpose mutex under
contention, and if the implementation also stored an owner thread id and
checked it on unlock, foreign unlock would be wrong again and `Send`
would be a lie. `std`’s `!Send` choice for its guard is portability
policy across OS backends, not a law of mutual exclusion.

## Why some OS mutexes track a thread id

Those APIs model a mutex as **owned by a thread**, not as a bare token
in memory. Exclusion alone does not require an owner id. The features
built on top often do:

1. **Ownership as the abstraction.** A classic mutex means *this*
   thread entered the critical section, so *this* thread must leave it.
   That differs from a semaphore, where any thread may `post`.
   Recording the owner turns “unlock by the wrong thread” into a
   detectable bug.

2. **Recursive locking.** “Same thread locking again” increments a
   depth counter; “other thread” blocks. Without an owner, recursion
   and contention are indistinguishable.

3. **Priority inheritance.** When a high-priority thread blocks on a
   lock held by a low-priority thread, the kernel must boost the
   *holder*. That requires knowing which thread owns the mutex.
   Priority-ceiling protocols have the same need.

4. **Robust mutexes / owner death.** If the owning thread crashes while
   holding the lock, the kernel can mark the mutex inconsistent for
   recovery only if it tracked the owner.

5. **Historical design.** Windows `CRITICAL_SECTION` and several pthread
   flavors grew up as thread-owned critical sections for scheduling and
   error checking, not as portable atomics-plus-queues.

A spinlock or `parking_lot`-style lock only needs “locked vs free” (and
maybe a waiter list). Any thread can release the token. That is enough
for mutual exclusion of data, but it does not give recursion-by-owner,
priority inheritance, or robust recovery — the features that push OS
mutexes toward storing a thread id.

## Takeaway

C++ `std::mutex` refuses copy and move so its identity stays fixed: one
object, one synchronization state, one stable address. Rust can move a
`Mutex` because borrow checking and `Arc` make “move under concurrent
lockers” a type error. Whether a *guard* may cross threads depends on
whether unlock is allowed off the locking thread — forbidden for many OS
mutexes, fine for atomic token locks like `parking_lot` or a bool
spinlock. The thread id in an OS mutex is there for ownership,
recursion, scheduling, and robustness, not because mutual exclusion
itself requires one.
