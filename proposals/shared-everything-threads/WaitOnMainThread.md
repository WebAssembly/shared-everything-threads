# Explainer: Allowing Atomics.wait on the main thread

## Authors:

 - Thomas Lively (Google)

## Participate

 - [GitHub Discussion](https://github.com/WebAssembly/shared-everything-threads/discussions/112)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [User-Facing Problem](#user-facing-problem)
  - [`Atomics.wait`](#atomicswait)
  - [Multithreaded Programs on the Web](#multithreaded-programs-on-the-web)
  - [Goals](#goals)
  - [Non-goals](#non-goals)
- [User Research](#user-research)
- [Proposed Approach.](#proposed-approach)
- [Alternatives Considered](#alternatives-considered)
  - [Do nothing](#do-nothing)
  - [Discourage use of locks on the main thread](#discourage-use-of-locks-on-the-main-thread)
  - [Encourage use of `Atomics.waitAsync` instead](#encourage-use-of-atomicswaitasync-instead)
  - [Allow `Atomics.wait` with a required timeout](#allow-atomicswait-with-a-required-timeout)
  - [Allow blocking only from WebAssembly](#allow-blocking-only-from-webassembly)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References and Acknowledgements](#references-and-acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction 

End users have an interest in being able to run high-performance Web applications that use shared-memory multithreading to take advantage of the user's multicore hardware.
End users also want these applications to run as efficiently as possible without wasted work to reduce power consumption and extend battery life.
But the current restriction that `Atomics.wait` cannot be executed on the main thread has caused developers to busy-wait on the main thread
instead of rearchitecting their applications like the original `Atomics.wait` designers hoped.
Busy waiting is bad for battery use and performance, so end-users would be best served if we lifted this restriction.

## User-Facing Problem

Understanding why preventing `Atomics.wait` from being used on the main browser thread causes suboptimal performance and power consumption for end users
requires understanding what problems `Atomics.wait` is meant to solve in the first place.
It also requires understanding how multithreaded Web applications are developed in practice.

### `Atomics.wait`

`Atomics.wait` is a JS API that provides a low-level multithreaded synchronization primitive that operates on SharedArrayBuffers.
It allows the calling thread to go to sleep until another thread calls the related `Atomics.notify` function to wake it back up.
It is not expected that most developers will use `Atomics.wait` directly.
Instead, it is expected to be used as a building block for constructing higher-level synchronization primitives such as mutual exclusion locks (mutexes) and condition variables.
Consider the following code that implements a simple lock _without_ using `Atomics.wait`:

```js
function lock(mutex_address) {
    // Try to change the value at mutex_address from 0 to 1 to acquire the lock.
    while (Atomics.exchange(int8Array, mutex_address, 1) == 1) {
        // The previous value was already 1, so some other thread must already own the lock.
        // Continue trying to acquire the lock until the other thread releases it and we succeed.
        continue;
    }
}
```

This code correctly implements acquiring a lock, but it is not efficient.
Because this code spins in a tight loop until it acquires the lock, it looks to the operating system and hardware like it is doing a lot of pure computation.
This is a "spinlock" and has two bad consequences:

1. The CPU continually uses power doing the same `Atomics.exchange` operation over and over.
   The operating system might even transition the CPU to a high-power mode because it thinks there is useful computation happening that it should make faster.

3. The operating system will keep the looping thread scheduled longer, again because it looks like it is doing useful computation.
   This may delay the scheduling of the thread that actually holds the lock, increasing the time until the lock can be acquired and slowing down the program.

These are the problems that `Atomics.wait` exists to solve. Consider this similar lock implementation that uses `Atomics.wait`:

```js
function lock(mutex_address) {
    // Try to change the value at mutex_address from 0 to 1 to acquire the lock.
    while (Atomics.exchange(int8Array, mutex_address, 1) == 1) {
        // Put this thread to sleep until the thread that owns it releases it and calls `Atomics.notify`.
        Atomics.wait(int8Array, mutex_address, 1);
        // The thread was released, but we have to try again to acquire it in case another thread has
        // acquired it in the meantime.
        continue;
    }
}
```

This implementation still has a loop, but now it puts itself to sleep until the lock is released every time it fails to acquire the lock.
Compared to the previous implementation, this implementation:

1. Does not use as much power because the thread sleeps (and does not consume CPU resources) until the lock is released.

2. Allows the operating system to schedule other threads that will do useful work, reducing the time to acquire the lock and making the program faster overall.

In practice, locks are more complicated and typically use a hybrid of the two approaches because `Atomics.wait` is itself an expensive operation.

Currently `Atomics.wait` unconditionally throws on the main browser thread,
following the API [design principle](https://w3ctag.github.io/design-principles/#worker-only) that we should not allow blocking the main thread.
This means that the only way to implement synchronous locking on the main thread is with the initial spinlock design,
with all its performance and power consumption downsides for the end user.
(There is also `Atomics.waitAsync`, which allows _asynchronous_ locking on the main thread, but [see  below](#encourage-use-of-atomicswaitasync-instead) for why this is not sufficient.)

When the decision was made to disallow `Atomics.wait` on the main thread,
the [assumption](https://github.com/tc39/proposal-ecmascript-sharedmem/issues/100#issuecomment-219986764) was that this would nudge developers to use mechanisms like `postMessage`
rather than spinlocks to receive messages on the main browser thread.
We now have a decade of experience with multithreaded programs on the Web showing that this is not the case.

### Multithreaded Programs on the Web

Shared-memory multithreading on the Web currently works by sharing a `SharedArrayBuffer` between multiple `WebWorkers` and possibly also the main browser thread.
Developers of multithreaded applications and libraries do not typically manage the WebWorkers and the low-level synchronization themselves.
For that matter, they rarely write JavaScript or even TypeScript.
Instead, they write in languages like C, C++, or Rust and compile their programs to a mix of WebAssembly and JavaScript using toolchains like [Emscripten](https://emscripten.org/).

These toolchains provide ports of the C, C++, and Rust standard libraries, including their multithreading APIs, that are implemented in terms of Web APIs.
For example, Emscripten provides an implementation of POSIX threads (pthreads), including functions like `pthread_mutex_lock`.
The implementation of these threading primitives ultimately boils down to code that looks very similar to the simplified lock implementations shown in the previous section.

Because toolchains like Emscripten compile from languages that have rich ecosystems outside the Web, they are heavily motivated to prioritize portability.
The more they can make existing applications and libraries "Just Work" on the Web without significant rewrites, the more useful they are to developers and the more high-performance experiences developers can provide to end users.

Part of making existing code "Just Work" is making synchronization primitives like locks work no matter what thread they are called from, including the main browser thread.
Since `Atomics.wait` cannot currently be called on the main thread, Emscripten uses a spinlock implementation on the main thread.
This has the previously discussed downsides for end users, but the alternative is that the developer's code would require potentially cost-prohibitive[^1] rewriting to avoid locking on the main thread,
so end users would have fewer high-performance Web experiences available to them.

[^1]: Emscripten does not offer locking with JSPI+`Atomics.waitAsync` out of the box, but it does offer other features, namely profile-guided module splitting and deferred loading,
that come with the same requirement to make all the application JS that interacts with Wasm asynchronous.
Developers have found this requirement to be prohibitively expensive even for such high-value features as deferred code loading, so they are unlikely to put in the same required effort just to avoid spinlocks.

### Goals

 - Improve performance and reduce power consumption for end users.
 - Allow toolchains like Emscripten to avoid spin-locking on the main thread.
 - Avoid creating extra work for developers who use such toolchains.
 - Avoid making it easier for developers to accidentally create new performance problems by blocking the main thread.

### Non-goals

 - Allowing `Atomics.wait` on `AudioWorklets`.
   Although many of the same arguments made here for the main thread may also apply to `AudioWorklets`, they have much stricter realtime requirements, so we explicitly do not propose any changes to them here.

## User Research

Although the performance and power use problems of the status quo are largely invisible to end users, the lack of `Atomics.wait` on the main thread has historically been a significant pain point for developers
(and specifically toolchain developers, since normal developers just get a spinlock implementation that Just Works):

 - GitHub issue (2016): [Reconsider restrictions on Atomics.wait in the browser thread](https://github.com/tc39/proposal-ecmascript-sharedmem/issues/100)
 - GitHub issue (2018, 10 👍): [Lack of atomic.wait on the main thread seems limiting to a fault](https://github.com/WebAssembly/threads/issues/106)
 - GitHub issue (2021, 17 👍): [Can we relax Atomics.wait to allow waiting on the main thread?](https://github.com/WebAssembly/threads/issues/177)
 - Polyfill (2024): [juj/js-main-thread-atomics-wait](https://github.com/juj/js-main-thread-atomics-wait)

This problem has also been significant enough that it has inspired the specification of two partial workarounds (although they are also independently useful):

 - [`Atomics.waitAsync`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/waitAsync), an async version of `Atomics.wait` that does not block and is allowed on the main thread.
   WebAssembly's JavaScript Promise Integration (JSPI) makes this easier to adopt, but [see below](#encourage-use-of-atomicswaitasync-instead) for why this doesn't generally solve the problem.
 - [`Atomics.pause`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/pause), an API for transitioning the CPU into a lower-power state during spinloops.

## Proposed Approach.

Allow `Atomics.wait` (and analogous WebAssembly instructions) to be used on the main browser thread.

Specifically, set `[[CanBlock]]` to `true` for the [agent](https://tc39.es/ecma262/#sec-agents) corresponding to the main browser thread.

Allowing `Atomics.wait` on the main thread will let toolchains like Emscripten use efficient locks rather than spinlocks on the main thread, improving performance and reducing power consumption for end users.

Application developers will see no change. After updating their toolchains, they will silently get the more efficient locking.

In practice there will be no new opportunities for creating performance problems by blocking the main thread.
Developers who want to use `Atomics.wait` already fall back to using spinlocks on the main thread.
Using `Atomics.wait` instead is a strict improvement over this status quo.

Note that this rationale does not apply to any other blocking APIs such as sync XHR.
It is of course still important to avoid blocking the main thread where possible, but it is not always possible to avoid locking on the main thread.

As a potential extra benefit, allowing `Atomics.wait` on the main thread would give the browser more visibility into locking behavior than it has with the spinlocks used on the main thread today.
For example, it would be possible for DevTools to differentiate between deadlocks and infinite loops, which is not generally possible with a spinlock.

## Alternatives Considered

### Do nothing

Multithreaded programs have been providing value to users of the Web for nearly a decade despite the problems this proposal aims to solve, and they will continue to do so.
The only downside of doing nothing is a missed opportunity to improve performance and power consumption for end users.

### Discourage use of locks on the main thread

This is not meaningfully different from the status quo.
It is well-known that blocking the main browser thread can cause user-visible performance problems, but at the same time shared-memory multithreading allows efficient offloading of expensive work to background threads.
When the background threads need to communicate results back to the main thread, the simplest and most efficient option is often for the main thread to briefly acquire a lock to receive the results.
(Notably, the browser itself will internally acquire a lock to receive postMessage messages from Workers, so it is never possible to avoid locks entirely, even if the application were to be completely rewritten to use message passing exclusively.)

Locks are not just acquired for application-level coordination between threads, though.
They are also widely used in basic library code such as memory allocators.
Saying the main thread should not take locks is equivalent to saying the main thread should not do anything as basic as allocating memory to hold a string.
This is not a realistic restriction on application architecture, so it is inevitable that the main thread will need to acquire locks.

There is also no way to completely disallow developers from acquiring locks on the main thread, even if we wanted to do so.
Spinlocks do not depend on any APIs and there is no way to statically disallow unbounded loops in JavaScript or WebAssembly.

### Encourage use of `Atomics.waitAsync` instead

`Atomics.waitAsync` is already allowed on the main thread, but unfortunately this is not a silver bullet.
Because the source languages (C, C++, Rust) that developers are writing specify lock acquisition as a synchronous operation,
developers would need to opt-in to using features like [JavaScript Promise Integration](https://v8.dev/blog/jspi) or [Asyncify](https://emscripten.org/docs/porting/asyncify.html) to make the asynchronous wait operation appear synchronous.
Both solutions accomplish this by suspending the WebAssembly execution stack and allowing it to be resumed later after the asynchronous task has been completed.
But both solutions have downsides that mean they cannot always be used:

 - Asyncify is a whole-program transformation that comes with significant code size and performance downsides.
   Even JSPI can have significant performance overhead for the most performance-sensitive workloads.

 - Both solutions make exported WebAssembly functions that might suspend asynchronous, whereas WebAssembly exports are usually synchronous.
   Because locks can be acquired in arbitrary library code, developers must conservatively assume any export can suspend, so the entire interface between toolchain-generated code and application JavaScript must become async.
   Rewriting the application JS to support this can be cost-prohibitive.

 - Suspending WebAssembly execution creates new sources of subtle bugs where the program is re-entered by calling an exported WebAssembly function, even though the source language semantics expect the thread to be blocked on a synchronous call.
   Using these features can be risky for developers.

The downsides ensure that there will always be developers that cannot use Asyncify or JSPI, so cannot use `Atomics.waitAsync`.

###  Allow `Atomics.wait` with a required timeout

The motivation for such a proposal would be to limit the damage developers can do by accidentally blocking the main thread for too long with `Atomics.wait`.
However, given that developers already fall back to spinlocks on the main thread, they would certainly just add a loop around the bounded waits to achieve the effect of an unbounded wait, so this restriction would have no benefit in practice.

On the other hand, this would almost fully solve the power consumption and performance problem.

### Allow blocking only from WebAssembly

Since approximately all multithreaded Web applications are developed with WebAssembly, this would be sufficient to solve the problem in most cases.
It would introduce an unnecessary divergence in behavior between JS and WebAssembly, though.

As with the previous alternative solution, the motivation would be to prevent developers from accidentally blocking the main thread with `Atomics.wait`.
But again, this wouldn't have any benefit in practice because developers would just continue using spinlocks instead, which is even worse.

Allowing `Atomics.wait` on the main thread does not introduce a new performance footgun; it rather allows developers an alternative to a spinlock, which is the real performance footgun.

## Stakeholder Feedback / Opposition

To be collected in response to this explainer :)

## References and Acknowledgements

Thanks to Jukka Jylänki for his persistence in raising these issues and advocating for this proposed change over the years.

These issues were first considered by Lars Hansen [over a decade ago](https://github.com/tc39/proposal-ecmascript-sharedmem/blob/main/issues/FutexWaitOnMainThread.md).

This proposal is motivated by other ongoing work on adding more general and efficient [wait and notify primitives](https://github.com/WebAssembly/shared-everything-threads/blob/main/proposals/shared-everything-threads/Overview.md#managed-waiter-queues) to WebAssembly.
Thanks to Shu-Yu Guo for first suggesting new work in this area.

Thanks to Jeffrey Yasskin for early feedback on this explainer.
