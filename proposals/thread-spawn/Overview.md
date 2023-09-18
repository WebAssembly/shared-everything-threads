# [DRAFT] Thread Spawn Proposal

This page describes a proposal to allow WebAssembly modules to spawn threads from within the
WebAssembly language. This is done with several additions to the specification (each addition is
annotated with its criticality to the proposal; next-stage discussions could alter this list):

- __required__: new `shared` attributes on WebAssembly tables, functions, and globals
- __required in some form__: a new `thread.spawn` instruction (other mechanisms are possible; more
  discussion needed)
- __helpful__: a new `thread.hw_concurrency` instruction

Though conceptually simple, this idea has not yet been formally proposed due to several challenges,
not the least of which is the perceived complexity of implementing all of this safely. Please be
patient as we work through these challenges.



## Motivation

The 2019 paper ["Weakening WebAssembly"] described how the WebAssembly specification could add a
mechanism for spawning threads _and_ for these threads to safely interact. That idea advanced as the
[threads] proposal, which discarded the thread-spawning mechanism in order to focus exclusively on
(a) atomic loads and stores on `shared` memory, (b) read-modify-write instructions, and (c) the
`wait`/`notify` instructions. After all, browsers could implement threads with Web Workers and
expose thread-spawning with imports, as Emscripten has [done][emscripten-pthreads]. The [threads]
proposal is now at [phase 3][proposals], trending toward phase 4.

["Weakening WebAssembly"]: https://github.com/WebAssembly/spec/blob/main/papers/oopsla2019.pdf
[threads]: https://github.com/WebAssembly/threads
[emscripten-pthreads]: https://emscripten.org/docs/porting/pthreads.html
[proposals]: https://github.com/WebAssembly/proposals

Since then, concurrent execution of WebAssembly has advanced in non-browser engines. [wasi-threads]
provided a WASI API for spawning threads and, judging from the feedback received, filled a gap in
the ecosystem. Using [wasi-threads], developers could compile pthreads-based code [from
C/C++][wasi-sdk-20] ([soon Rust][rust-threads]) and run it concurrently in engines such as Wasmtime
and WAMR. Experiments with [wasi-parallel], a similar but higher-level API, hinted that concurrent
execution of WebAssembly code was possible even on non-CPU hardware (e.g., GPUs).

[wasi-threads]: https://github.com/WebAssembly/wasi-threads
[wasi-parallel]: https://github.com/WebAssembly/wasi-parallel
[wasi-sdk-20]: https://github.com/WebAssembly/wasi-sdk/releases/tag/wasi-sdk-20
[rust-threads]: https://github.com/rust-lang/rust/pull/112922

All of this related work provides momentum for this `thread-spawn` proposal, but why is a new
proposal needed? The key reasons:
- a unified thread-spawn mechanism would __improve compatibility__ between engines (browser vs.
  standalone) and toolchains (e.g., wasi-sdk, Emscripten, other language compilers), avoiding
  ecosystem fragmentation
- WebAssembly-specific thread spawning would be __simpler__, avoiding complexities in toolchains
  (e.g., Emscripten's `-sPTHREAD_POOL_SIZE` and `-sPROXY_TO_PTHREAD`) and complexity for users of
  concurrent WebAssembly (i.e., a single mechanism to target vs. what we have today)
- threads implemented using Web Workers or [wasi-threads] create a __new instance per thread__
  &mdash; this is difficult (impossible?) to reconcile with the [component model]
- thread spawning using Web Workers is __quite slow__ (2-3 orders of magnitude slower than OS
  threads &dash; [benchmarks]); a WebAssembly `thread.spawn` instruction could improve performance

[component model]: https://github.com/WebAssembly/component-model
[emscripten-pthreads]: https://emscripten.org/docs/porting/pthreads.html
[benchmarks]: https://github.com/abrown/bench-thread-spawn

The lessons learned over the past years plus the interest in the ecosystem suggests that the time
for this proposal is _now_, but you may consider this motivation insufficient. Feel free to discuss
further here: [Is this proposal necessary?][necessity-discussion].

[necessity-discussion]: https://github.com/abrown/thread-spawn/discussions/1


## Goals

- a low-level mechanism for spawning threads that can support concurrency in all kinds of languages
  (i.e., not structured concurrency, not pthreads-specific, etc.)
- support all kinds of parallelism: [task parallelism], [data parallelism]
- support all kinds of parallel libraries: at a minimum, pthreads

[task parallelism]: https://en.wikipedia.org/wiki/Task_parallelism
[data parallelism]: https://en.wikipedia.org/wiki/Data_parallelism



## Proposed Changes

### `thread.spawn`

The proposed mechanism for spawning threads in WebAssembly is a new `thread.spawn` instruction:

| Section       | Proposed change (rough)                                                                                                         |
|---------------|---------------------------------------------------------------------------------------------------------------------------------|
| Syntax        | `thread.spawn funcidx`                                                                                                          |
| Validation    | Valid with type `[i32, i32] -> []`; also, the function referred to must have type `[i32] -> []` and be `shared`                 |
| Execution     | With function index `f` and stack values `n` and `c`, enqueue `n` "parallel" invocations of `f` passing `c`; immediately return |
| Binary format | `0xFE 0x04 f:funcidx => thread.spawn f`                                                                                         |

[Binary format]: https://webassembly.github.io/spec/core/bikeshed/#instructions%E2%91%A6
[Syntax]: https://webassembly.github.io/spec/core/bikeshed/#instructions%E2%91%A0

`thread.spawn` acts almost like `call` in that it invokes a statically-known function. It differs in
that:
- the invocation occurs elsewhere &mdash; either in a separate thread or at a later time (e.g., it
  is always valid to implement `thread.spawn` by running each invocation sequentially in a single
  thread)
- it creates multiple invocations &mdash; the `n` invocations mean that "all these invocations
  _could_ start concurrently"
- it invokes a very limited function type &mdash; `c` represents a context passed to the function,
  perhaps an address pointing at more data.

#### Multiple invocations

The motivation for designing `thread.spawn` to enqueue `n` invocations is _generality_ and
_performance_. Experience with [wasi-parallel] indicates that some parallel frameworks (e.g., OpenMP
and others) and parallel hardware (e.g., GPUs) can make effective use of this information to improve
performance. Knowing that `n` invocations _could_ start concurrently allows engines to immediately
execute `n` (or batches < `n`) invocations on massively-parallel hardware. The lack of `n` would
make this quite difficult; e.g., imagine trying to batch up tasks to do this dynamically &mdash; are
we waiting too long for tasks to fill the batch? But, on the other hand, including `n` is harmless:
(a) we can always spawn a single thread with `n = 1` and (b) knowing `n` supposes no engine overhead
(quite the opposite: knowing a large amount of parallel computation is coming could help the engine
with thread scheduling). "WebAssembly on GPUs" and engine scheduling optimization may not be
convincing for some; we can discuss this further in: [Should we `spawn n`?][spawn-n-discussion]

[spawn-n-discussion]: https://github.com/abrown/thread-spawn/discussions/2

#### Type constraints

The motivation for passing a single `c` parameter to the parallel function is due to:

1. _convention_: some spawn mechanisms (e.g., `pthread_create`) pass a single parameter to the
   created thread; to pass multiple parameters in this model, one places them in memory and passes
   the address of a structure from which to retrieve them. [wasi-threads] shares this convention and
   it has not yet been problematic.
2. _already restricted_: the invoked function must already have a restricted type &mdash; it cannot
   return any values. Because this design has no concept of a "thread" handle (no need, see [thread
   joining]), there is no place to return these values to; they cannot be returned to the caller as
   execution has already progressed past the `thread.spawn` instruction.
2. _simplicity_: implementation-wise, if engines map `thread.spawn` to OS threads, they may be able
   to do less wrapping and unwrapping if `f` expects a single parameter (as do OS threads).

[thread joining]: #what-about-thread-joining

To discuss this more: [What should the spawned function's type be?][type-discussion].

[type-discussion]: https://github.com/abrown/thread-spawn/discussions/3

### `shared` attributes

The ["Weakening WebAssembly"] paper describes `shared` attributes as a key mechanism for ensuring
the thread safety of `thread.spawn`. In the [threads] proposal, WebAssembly memories can be marked
as `shared`, which allows engines to safely (i.e., atomically) implement `memory.grow` (read the
paper for the sequential consistency guarantees). Following the paper, this proposal extends
`shared` attributes to _all_ WebAssembly objects:

- `shared` tables will be grown and modified atomically
- `shared` globals will be modified atomically
- `shared` functions can never access non-`shared` objects; e.g., they only call other `shared`
  functions and only access `shared` memories, tables, and globals

__TODO__: add a WAT example

#### Engine requirements

During validation, we ensure that any targets of `thread.spawn` are `shared` functions and that all
`shared` functions do not call non-`shared` functions. This can be problematic: all host imports
(e.g., WASI, Web APIs) are currently imported as non-`shared` functions. How will a thread print
output or read from a file? This bears more discussion: [How can we have `shared`
imports?][import-discussion].

[import-discussion]: https://github.com/abrown/thread-spawn/discussions/4

__TODO__: chart what happens for interactions between non-shared, shared, and embedding contexts;
e.g., if from the host we call a non-`shared` export, does it have exclusive access to the main
thread's objects? What happens if we call `thread.spawn` here?

#### Toolchain requirements

Toolchains like LLVM will be expected to mark all WebAssembly objects that a parallel function
touches with the `shared` attribute. It is unclear how difficult this might be and one conservative
approach (for the toolchain) is to simply mark all module objects as `shared` if `thread.spawn` is
ever used. For more discussion: [How should toolchains apply
`shared`?][toolchain-shared-discussion].

[toolchain-shared-discussion]: https://github.com/abrown/thread-spawn/discussions/5

### `thread.hw_concurrency`

Some algorithms need to detect the number of threads the underlying hardware can be expected to
execute concurrently. A new `thread.hw_concurrency` instruction allows this:

| Section       | Proposed change (rough)                                                                  |
|---------------|------------------------------------------------------------------------------------------|
| Syntax        | `thread.hw_concurrency`                                                                  |
| Validation    | Valid with type `[] -> [i32]`                                                            |
| Execution     | Push on the stack the number of threads the module _may_ be able to execute concurrently |
| Binary format | `0xFE 0x05 => thread.hw_concurrency`                                                     |

This instruction has fingerprinting potential, which is a concern in some environments.
`thread.hw_concurrency`, however, is designed to be a WebAssembly analogue of the browser
[`hardwareConcurrency`] property, which is [widely available] and already exposes web users to
fingerprinting. Note that:

1. WebAssembly engines could choose to limit, randomize, or alter the value returned if privacy
   concerns outweigh the performance benefit of allowing algorithms to fit to the hardware and
2. this kind of fingerprinting is likely possible without the instruction via some careful timing of
   shared access from concurrently-executing threads

This issue can be discussed further here: [Should we include
`thread.hw_concurrency`?][hw-concurrency-discussion].

[`hardwareConcurrency`]: https://developer.mozilla.org/en-US/docs/Web/API/Navigator/hardwareConcurrency
[widely available]: https://caniuse.com/?search=hardwareConcurrency
[hw-concurrency-discussion]: https://github.com/abrown/thread-spawn/discussions/6



## Other Considerations

### What about thread-local storage (TLS)?

`shared` attributes prevent `shared` functions from accessing any non-`shared` objects. This means
that TLS must be implemented by indexing a thread (see [TIDs]) into something `shared`, e.g., a
`shared` memory. But how to do this could be problematic, see: [Do we need more for
TLS?][tls-discussion].

[tls-discussion]: https://github.com/abrown/thread-spawn/discussions/12

One might imagine splitting up tables or sets of globals for TLS, but `shared` memories are likely
more straightforward. For example, when compiling C/C++ to WebAssembly, address-taken local
variables are placed on an auxiliary stack in linear memory. For wasi-threads support, wasi-libc was
already modified [to understand][wasi-libc-tls] how to manage thread-local versions of this stack.

[TIDs]: #what-about-thread-ids-tids
[wasi-libc-tls]: https://github.com/WebAssembly/wasi-libc/blob/bd950eb128bff337153de217b11270f948d04bb4/libc-top-half/musl/src/thread/pthread_create.c#L456

### What about thread IDs (TIDs)?

This proposal does not require TIDs; TIDs are left for higher-level libraries to implement. As
discussed in [wasi-threads#38], some consider the inclusion of TIDs in [wasi-threads] to be
problematic. If in the future some kind of "thread handle" did become necessary (e.g., for setting
thread priority), we could revisit this discussion. But for now, thread libraries that require a TID
could atomically generate their own. If we needed a reason for these generated TIDs to be compatible
across libraries, we might consider a [tool convention].

[wasi-threads#38]: https://github.com/WebAssembly/wasi-threads/issues/38
[tool convention]: https://github.com/WebAssembly/tool-conventions

Another consideration for identifying threads: in data parallel applications, each thread must be
able to identify which data it should work on. This proposal provides no mechanism for a thread to
determine which of `n` invocations it is. This can, however, be easily resolved with a shared
global, e.g.: upon starting execution, each thread atomically reads and increments a counter, using
the value as its key to its subset of the data. Other mechanisms of this kind are possible.

### What about thread joining?

This proposal does not include a language-level mechanism for "joining" a thread (e.g.,
`pthread_join`). As shown with [wasi-threads] in [wasi-libc][wasi-libc-pthread-join], this can be
implemented via the existing `wait` and `notify` instructions.

[wasi-libc-pthread-join]: https://github.com/WebAssembly/wasi-libc/blob/bd950eb128bff337153de217b11270f948d04bb4/libc-top-half/musl/src/thread/pthread_join.c

### What about exiting a thread?

There are a couple of sides to this; recall that "thread" here means a function invoked via
`thread.spawn`:

1. If a thread is __finished__ (e.g., `return`), it simply ends. There is no caller to return to and
   the signature requirements (see [Type constraints]) mean no value need be returned. If any thread
   cleanup work must be done, it must be added by the toolchain in a trampoline function (as
   wasi-libc [does]) or inserted directly into the spawned function.
2. If a thread __traps__, all threads abort execution. This falls inline with the current spec:
   "traps cannot be handled by WebAssembly code." The time it takes to abort all other threads may
   be non-deterministic, but one would expect engines to do this as quickly as possible and for
   users to never rely on any "abort gap after trap" behavior.
3. If a thread must __early exit__, there are several options. The approach currently taken here is
   to rely on the [exception handling] proposal, currently at phase 3: e.g., `pthread_exit` could be
   implemented via `throw` with a corresponding `catch` in a thread-wrapping trampoline. Another
   option would be to add another instruction to the proposal: `thread.exit`. There is a related
   discussion about this issue in [wasi-threads#7] and if you have an opinion please discuss: [How
   should threads early-exit?][thread-exit-discussion]

[does]: https://github.com/WebAssembly/wasi-libc/blob/bd950eb128bff337153de217b11270f948d04bb4/libc-top-half/musl/src/thread/pthread_create.c#L307
[Type constraints]: #type-constraints
[exception handling]: https://github.com/WebAssembly/exception-handling
[wasi-threads#7]: https://github.com/WebAssembly/wasi-threads/issues/7
[thread-exit-discussion]: https://github.com/abrown/thread-spawn/discussions/7

### What changes are necessary in the JavaScript API?

See the TC39 [structs] proposal

__TODO__: more work needed here

[structs]: https://github.com/tc39/proposal-structs

### What about the stack-switching proposal?

This proposal should work in concert with the [stack-switching] proposal.

__TODO__: ask someone who knows to fill this in

[stack-switching]: https://github.com/WebAssembly/stack-switching

### What about the garbage collection proposal?

This proposal should work in concert with the [gc] proposal.

__TODO__: ask someone who knows to fill this in

[gc]: https://github.com/WebAssembly/gc
