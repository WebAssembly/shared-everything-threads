# Shared-Everything Threads

> __WARNING__: this proposal is under active development as we merge in various ideas discussed
> during phase 1. Expect frequent and significant changes!

This page describes a proposal to allow WebAssembly modules to share all kinds of data and module
fields between threads. It also adds additional features like thread-local storage that producers
will need when using the new sharing features. There are several planned additions to the
specification:

- `shared` annotations on WebAssembly tables, functions, globals, etc. to statically ensure that
  only shareable data can be shared between threads.
- thread-local globals on which language runtimes can build their thread-local storage.
- instructions for sequentially consistent and release-acquire accesses to shared WasmGC data.
- instructions for release-acquire accesses to shared memory.
- managed waiter queues for an efficient futex-like wait/notify mechanism usable with WasmGC.
- [component model] builtins for thread lifecycle management.

[component model]: https://github.com/WebAssembly/component-model

## Motivation

The 2019 paper ["Weakening WebAssembly"] described how the WebAssembly specification could add a
mechanism for spawning threads _and_ for these threads to safely interact. That idea advanced as the
[threads] proposal, which discarded the thread-spawning mechanism in order to focus exclusively on
(a) atomic loads and stores on `shared` memory, (b) read-modify-write instructions, and (c) the
`wait`/`notify` instructions. After all, browsers could implement threads with Web Workers and
expose thread-spawning with imports, as Emscripten has [done][emscripten-pthreads]. The [threads]
proposal is now at [phase 4][proposals].

["Weakening WebAssembly"]: https://github.com/WebAssembly/spec/blob/main/papers/oopsla2019.pdf
[threads]: https://github.com/WebAssembly/threads
[emscripten-pthreads]: https://emscripten.org/docs/porting/pthreads.html
[proposals]: https://github.com/WebAssembly/proposals

The initial threads proposal was sufficient to implement threads for linear memory languages on the
Web, but we need more functionality to improve performance for that use case and support other use
cases at all. For example:

 - The only memory order currently supported by Wasm is sequential consistency, which has large
   performance overheads.
 - The lack of shared tables makes dynamic loading extremely complicated and slow, even for
   languages that otherwise can already use threads.
 - There is no way to use threads with WasmGC programs at all because there is no way to share
   reference values across threads.
 - There is no standard way to spawn threads from Wasm programs in non-JS environments, although
   feedback on [wasi-threads] and the higher level [wasi-parallel] WASI APIs showed that this is an
   important gap to fill in the ecosystem.

[wasi-threads]: https://github.com/WebAssembly/wasi-threads
[wasi-parallel]: https://github.com/WebAssembly/wasi-parallel

This catch-all proposal aims to fill these gaps in functionality to enable multithreading across all
WebAssembly ecosystems.

## Goals

- a low-level mechanism for spawning threads that can support concurrency in all kinds of languages
  (i.e., not structured concurrency, not pthreads-specific, etc.)
- support all kinds of parallelism: [task parallelism], [data parallelism]
- support all kinds of parallel libraries: at a minimum, pthreads

[task parallelism]: https://en.wikipedia.org/wiki/Task_parallelism
[data parallelism]: https://en.wikipedia.org/wiki/Data_parallelism

## Proposed Changes

We expect the following changes to enable something like the following example (TODO):

```wat
(module
  (type $unshared (func))
  (type $shared (func shared))

  ;; An imported shared function.
  (func $baz (import "env" "baz") (type $shared))

  (memory $m 1 1)

  ;; $foo can access memory $m because neither $foo nor $m are shared.
  (func $foo (type $unshared)
    i32.const 0
    i32.load $m
    drop
  )

  ;; $bar cannot access the un-shared memory, but it can call the shared import $baz.
  (func $bar (type $shared)
    call $baz
  )
```

### `shared` Annotations

The ["Weakening WebAssembly"] paper describes `shared` annotations as a key mechanism for ensuring
the thread safety of WebAssembly threads. Due to the [threads] proposal, WebAssembly memories can be
marked as `shared`, which allows engines to safely (i.e., atomically) implement `memory.grow`. This
proposal extends `shared` annotations to _all_ WebAssembly objects: tables, globals, functions, etc.

During validation, an engine ensures that shared items refer only to other shared items. Non-shared
items can refer to shared items, though:

| From       | Can access? | To         | Notes |
|------------|-------------|------------|-------|
| non-shared |     ✅     | non-shared | This would continue to work as it does today: functions calling other functions, accessing tables, globals, and memory, etc. |
| non-shared |     ✅     | `shared`   | This would also continue to work as it does today: e.g., functions can access `shared` memory. |
| `shared`   |     ❌     | non-shared | This is not possible, by validation. |
| `shared`   |     ✅     | `shared`   | This is how we expect threads to access state: only `shared` state. |

What follows is a description of the specification changes needed to add `shared` annotations. The
new abstract syntax extensions use the
[`share`](https://webassembly.github.io/threads/core/syntax/types.html#syntax-share) metavariable
meaning `shared` or `unshared` introduced in the shared memory proposal.

#### Memories

Shared memories have already been standardized.

#### Tables

The syntax of `tabletype` is extended:

```
tabletype ::= limits share reftype
```

A `tabletype` is valid as shared only if its `reftype` is valid as shared.

Like `memory.size` and `memory.grow`, `table.size` and `table.grow` will have [sequentially
consistent] ordering when executed on a `shared` table. The existing `table.get` and `table.set`
instructions provide unordered access to shared tables. New instructions adding atomic accesses to
tables are presented [below][new-instructions]. The new instructions are valid for both shared and
unshared tables.

[sequentially consistent]: https://en.wikipedia.org/wiki/Sequential_consistency
[new-instructions]: TODO

#### Globals

The syntax of `globaltype` is extended:

```
globaltype ::= mut share valtype
```

A `globaltype` is valid as shared only if its `valtype` is valid as shared.

The existing `global.get` and `global.set` instructions provide unordered access to shared globals.
New instructions adding atomic accesses to globals are presented [below][new-instructions]. The new
instructions are valid for both shared and unshared globals.

[new-instructions]: TODO

#### Functions

The `typeidx` in the structure of a function determines whether the function is shared, so the
syntax of functions does not need to be extended. If a function is shared, its locals and body must
validate as shared. An expression validates as shared if its instruction sequence validates as
shared. An instruction sequence validates as shared if each instruction in the sequence validates as
shared. An instruction validates as shared if its instruction type validates as shared, which is the
case if each of its input and output value types validates as shared.

To capture sharedness for imported and exported functions, as well as to facilitate the validation
of instructions as shared, the structure of `functype` is extended:

```
functype ::= resulttype -> resulttype share
```

> Note: If this validation turns out to be too strict to be usable, we may relax the validation of
> functions to allow the locals and intermediate types in the body of shared functions to validate
> as unshared. This would have consequences for the permissibility of shared continuations. We would
> still have to extend validation of instructions to prohibit referencing unshared module fields in
> shared contexts.

#### Heap Types

The syntax of `absheaptype` is extended:

```
absheaptype ::= share func | share nofunc | share extern | share noextern | share any | share eq | share i31 | share struct | share array | share none
```

The syntax of `comptype` is extended:

```
comptype ::= func share functype | struct share structtype | array share arraytype
```

The validation judgments for all kinds of types are parameterized by `share`. In general, the
conclusions are `ok(shared)` iff the premises are `ok(shared)`. The validation rules for
`absheaptype` and `comptype` are extended so that they are only `ok(shared)` if the heap type is
shared. Number types and vector types are always valid as shared.

Similarly, we need to be able to validate that a `typeidx` is valid as shared. We introduce an
auxiliary function `expandshare` similar to `expand` that returns `shared` or `unshared` for a given
deftype. A `typeidx` `x` is `ok(shared)` iff `expandshare(C.types[x]) = shared`.

Shared and unshared types are never subtypes of one another. The subtype relationships among shared
abstract types mirrors the subtype relationships among unshared abstract types.

> Note: Should i31ref behave like an i32 in that it is usable in both shared and non-shared contexts
> without needing to have a separate `shared` variant? This seems like it would be ok, except we
> would still need a shared version to interoperate with `(shared eq)`

#### Element Segments

The syntax of `elem` is extended:

```
elem ::= {type reftype, init vec(expr), mode elemmode, share}
```

An element segment is valid as shared only if its reftype is valid as shared. Instructions that
refer to element segments (e.g. `elem.drop`) are only valid as shared if their element segment is
shared.

#### Data Segments

The syntax of data segments is extended:

```
data ::= {init vec(byte), mode datamode, share}
```

Instructions that refer to data segments (e.g. `data.drop`) are only valid as shared if their data
segment is shared.

> Note: we could alternatively make all data segments shared, just like we do for other
> non-reference data.

#### Exception tags

The syntax of tags does not need to be extended. They are shared iff their function types are
shared. Shared tags are used to throw and catch `(shared exnref)`. Instructions that reference tags
(e.g. `throw` and `catch`) only validate as shared if all the tags they reference are shared.

> TODO: We need to determine how sharing affects exception handling. We may want to add
> `catch_all_shared_ref`. We may want `(shared exnref) <: exnref` so `catch_all_ref` can continue to
> handle all exceptions in non-shared functions.

### Thread-local storage (TLS)

To store thread-local data, WebAssembly modules previously relied on the “instance per thread”
model. Each instance had its own set of globals which could reference the instance-specific,
thread-local state stored in linear memory. In a `shared` context, this is no longer possible:
non-`shared` globals are inaccessible from shared functions and `shared` globals are not
thread-local.

After discussing the options (see the [“Do we need more from TLS?”][tls-discussion] discussion),
this proposal gained a mechanism for TLS. This proposal chooses the `thread_local` annotation as its
preferred solution; other options are discussed [here][tls-discussion] and in a [Google document].

> Note: In a Web context, it may make sense for "thread-local globals" to actually be realm-local,
> so the spec should decouple threads of execution from execution contexts. See the related
> discussion in the JS API section below.

[tls-discussion]: https://github.com/WebAssembly/shared-everything-threads/discussions/12
[Google document]: https://docs.google.com/document/d/1b8QuRhSo_JpMKGtv9zIEubuQ-jW3dzf4NpLWlND06Zw/edit#heading=h.3fybxwjpw6r6

#### Syntax and validation

The syntax of `globaltype` is extended again:

```
globalshare = share | 'thread_local'
globaltype = mut globalshare valtype
```

Thread-local globals are valid if their valtypes are valid as shared.

> Note: We don't allow non-shared references in thread-local globals because then they would not be
> able to be accessed from shared functions. We could always relax this restriction, especially if
> we relax the restrictions on intermediate values in shared functions.

#### Execution

Thread-local globals have different values on different threads. `global.get` and `global.set` on
thread-local globals access only the value for that global for the current thread. There is no way
to access a thread's thread-local global value directly from any other thread.

Thread-local globals can be imported and exported just like normal or shared globals.

Declared (i.e. non-imported) thread-local globals have constant initializers just like any other
global. The constant initializer is executed separately on each thread, so if, for instance, the
initializer is a `struct.new` instruction, there will be a separate struct allocated for each
thread.

> Note: To initialize a thread-local global to refer to the same struct on each thread instead, the
> initializer expression can be a `global.get` of a shared global.

> Note: Since thread-local globals will tend to be initialized more frequently than normal globals,
> it may make sense to further restrict their initializers, for example by prohibiting allocations.

#### Implementation

One of the nice things about this thread-local storage design is that it is declarative enough to
admit multiple implementation strategies. A naive implementation would be to store thread-local
global values in a concurrent hash map keyed by triples of instance, global, and thread IDs.
Thread-local global accesses would lower to accesses to this hash map.

A more sophisticated implementation would be to store TLS blocks containing all the thread-local
globals for a module in the map and use pairs of instance and thread IDs as the keys instead of
triples. Thread-local global accesses would then lower to accesses to the hash map to get the TLS
block base address followed by applications of an offset to access the global within the block. This
implementation has the advantage that the TLS block base address can be reused across accesses to
multiple thread-local globals, and in the extreme could be pinned to a register to eliminate all but
the initial hash map access for each instance on each thread.

This design also allows implementations to be flexible regarding when they initialize thread-local
globals. An engine wishing to eagerly initialize the globals would initialize the new TLS for every
instance in the system every time a new thread is created and would initialize the new TLS for every
thread in the system every time a new instance is created. Alternatively, an implementation could
lazily initialize the TLS for an instance on a thread the first time a function from that instance
(or any other instance that has imported a TLS global from that instance) is executed, ensuring that
globals are initialized before they can be accessed without spending time initializing globals that
will never be accessed.

### Thread Management Builtins

Standalone WebAssembly engines (i.e., non-browser) do not have web workers as a parallel primitive.
For these environments, we propose to modify the [component model] by adding the following builtins
to allow creating and managing threads:

```
canon ::= ...
          | (canon thread.spawn <typeidx> (core func <id>?))
          | (canon thread.hw_concurrency (core func <id>?))
```

These new builtins would join the short list of current [canonical builtins]: `lift`, `lower`, and
`resource.*` management. The new `thread.*` builtins have as a goal to be polyfillable on the web.

[canonical builtins]: https://github.com/WebAssembly/component-model/blob/main/design/mvp/Explainer.md#canonical-abi

#### `thread.spawn`

The `thread.spawn` built-in has type `[f: (ref null $f) n:i32 c:i32] -> []`, where `$f` is `(rec
(type (sub final (func shared (param i32))))).0`: it invokes the function `f` `n` times, potentially
in parallel, passing `c` to each invocation.

The need to pass a function reference is discussed [here][static-function-discussion]. Note that,
though the component model ABI does not allow passing `funcref`s, the canonical builtins have no
such restriction.

Ideally we would allow thread entry points to take arbitrary parameters, as discussed
[here][function-type-discussion]. Until we get type parameters in core Wasm, however, that would
require a name mangling scheme to describe the intended type of the import. It's unclear that there
can be any satisfactory name mangling scheme that would handle arbitrary GC types. For now we avoid
that complexity by using a single concrete type.

[static-function-discussion]: https://github.com/WebAssembly/shared-everything-threads/discussions/10
[function-type-discussion]: https://github.com/WebAssembly/shared-everything-threads/discussions/3

#### `thread.hw_concurrency`

Some algorithms need to detect the number of threads the underlying hardware can be expected to
execute concurrently. The `thread.hw_concurrency` builtin has type `[] -> [i32]` and returns the
number of threads the engine may allow to execute concurrently. An engine is not required to meet
the concurrency level it returns, but not doing so could cause negative performance effects for
users.

This builtin has fingerprinting potential, which is a concern in some environments.
`thread.hw_concurrency`, however, is designed to be a WebAssembly analogue of the browser
[`hardwareConcurrency`] property, which is [widely available] and already exposes web users to
fingerprinting. Note that:

1. WebAssembly engines could choose to limit, randomize, or alter the value returned if privacy
   concerns outweigh the performance benefit of allowing algorithms to fit to the hardware and
2. this kind of fingerprinting is likely possible without the instruction via some careful timing of
   shared access from concurrently-executing threads

Though this functionality was considered non-essential in previous
[discussions][hw-concurrency-discussion], those were in the context of browsers, with the
[`hardwareConcurrency`] property available. We expect standalone engines to need some equivalent for
general parallel programming.

[`hardwareConcurrency`]: https://developer.mozilla.org/en-US/docs/Web/API/Navigator/hardwareConcurrency
[widely available]: https://caniuse.com/?search=hardwareConcurrency
[hw-concurrency-discussion]: https://github.com/abrown/thread-spawn/discussions/6

### Managed Waiter Queues

Linear memory Wasm has `memory.atomic.wait32`, `memory.atomic.wait64`, and `memory.atomic.notify`
that provide a [futex-like interface][wait-notify] to runtime-managed waiter queues. The `wait`
instructions allow threads to atomically check a value at a particular memory address and begin
waiting if the value is what they expect. The `notify` instruction wakes some number of waiters
associated with a particular memory address. WasmGC programs using threads will need a similar
primitive that does not depend on linear memory. We propose adding a new abstract heap type
`waitqueue` to serve as this new primitive.

Unlike other abstract heap types, which are only ever subtypes of other abstract heap types (or are
bottom types), `waitqueue` is a final subtype of `(rec (type (sub (struct shared (field i32))))).0`,
meaning it is a struct with one visible `i32` field that can be accessed with all the standard
struct accessors as well as the new atomic struct accessors. This field is the futex control field
that is atomically checked when waiting on the waiter queue. `waitqueue` is also always shared.
There is no non-shared version of it. It is not valid to declare a new subtype of `waitqueue`.

To wait on and notify a particular `waitqueueref, there are two additional instructions:

 - `waitqueue.wait: [waitqueueref, i32, i64] -> [i32]`

This instruction behaves just like `memory.atomic.wait32` and `memory.atomic.wait64`: the first
operand is the wait queue to wait on, the `i32` operand is the expected value of the control field,
and the `i64` operand is a relative timeout in nanoseconds. The return value is `0` when the wait
succeeded and the current thread was woken up by a notify, `1` when the thread did not go to sleep
because the control field did not have the expected value, or `2` because the timeout expired.

Like the existing linear memory wait instructions, `waitqueue.wait` disallows spurious wakeups.

> Note: We should perhaps revisit allowing spurious wakeups, since disallowing them makes various
> kinds of interesting instrumentation impossible.

- `waitqueue.notify: [waitqueueref, i32] -> [i32]`

This instruction behaves just like `memory.atomic.notify`: The first operand is the wait queue to
wait on and the `i32` operand is the maximum number of waiters to wake up. The result is the number
of waiters that were actually woken up.

> Note: It may also be necessary to have versions of the waitqueue where the control field is an i64
> or a shared eqref. An eqref version almost satisfies the use case for the i32 version as well,
> since you could store an i31ref in the eqref field, except that you would not be able to perform
> the full complement of i31 RMW operations on the eqref without it being statically typed as an
> i31.

[wait-notify]: https://github.com/WebAssembly/threads/blob/main/proposals/threads/Overview.md#wait-and-notify-operators

### Memory Model Considerations

To improve performance for linear memory languages using threading (e.g. C, C++, and Rust) and to
make it feasible to compile languages targeting WasmGC that have stronger memory models (e.g. Java
and OCaml), we introduce [release-acquire][release-acquire] ordering as an intermediate memory order
that is stronger than unordered accesses and weaker than the sequentially-consistent memory order
used by all existing WebAssembly atomics. As described [below][new-instructions], the choice of
memory order can be encoded in a `memarg`, so all existing atomic instructions that take a `memarg`
immediate will be able to be used with release-acquire ordering. For other atomic instructions, new
encodings will be introduced to provide release-acquire variants.

Implementations must take care to ensure that uninitialized data in shared tables, globals, structs,
arrays, and any other location cannot be observed, even by other threads. This requires something
like a release barrier between when a new allocation is initialized and when its address is made
available to the program.

[release-acquire]: https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Acquire_ordering

### JS API

#### Conversions

The entire point of the `shared` annotation is to prevent unshareable embedder values from being
shared across threads. As such, `ToWebAssemblyValue(v, type)` is extended so that if `type` is a
shared reference, a `TypeError` is almost always thrown with these exceptions:

 - Any value that could be converted to a reference to `i31`, `none`, `nofunc`, or `noextern` can
   also be converted to a `(shared i31)`, `(shared none)`, `(shared nofunc)`, or `(shared noextern)`
   reference, respectively.
 - JS Strings can be converted to shared externref since they are shareable according to the [JS
   shared structs][proposal-structs] proposal.
 - Shared JS structs, shared JS arrays, `Atomics.Mutex` and any other shareable type introduced by
   the JS shared structs proposal can be converted to shared externref.

> Note: It would be great for Wasm-JS interoperability if JS shared structs and arrays could also be
> converted to Wasm shared structs and arrays and vice versa. We will have to see what is possible
> there.

Conversely, `ToJSValue(w)` should convert WebAssembly shared references to shareable opaque objects
on the JS side, analogous to how it converts non-shared references to opaque JS objects, at least
until we get more ergonomic interop between JS and WasmGC. These opaque JS objects will be shared
objects that do not require rewrapping, and their identities survive round trips via postMessage
across thread boundaries.

Waitqueue references are converted to `WebAssembly.WaitQueue` objects, described below.

[proposal-structs]: https://github.com/tc39/proposal-structs

#### Shared annotations

Just like the original threads proposal exposed shared memories in the JS API, this proposal exposes
shared functions, tables, globals, tags, and exceptions. Unlike shared `WebAssembly.Memory` objects,
which are actually realm-local wrappers around SharedArrayBuffers (which are in turn realm-local
wrappers around the underlying shared buffers), the new shared tables, globals, tags, and exceptions
will be shared JS objects that do not require rewrapping. Such shared objects have fixed layout and
behave mostly like [sealed] JS objects, and their identities survive round trips via postMessage
across thread boundaries. Additionally, to facilitate calling methods on these shared objects, they
have [realm-local prototypes][tls-prototypes].

The shared versions of the JS API types (besides memories) will be represented with new
constructors:

 - `WebAssembly.SharedFunction`
 - `WebAssembly.SharedTable`
 - `WebAssembly.SharedGlobal`
 - `WebAssembly.SharedTag`
 - `WebAssembly.SharedException`

These new types provide an API that mirrors that provided by their unshared analogues. Like shared
memories, shared tables must be initialized with a maximum size. Shared exceptions may only be
created with shared tags. Storing unshared data into a shared global or table throws an exception.

While the behaviors chosen here mirror those proposed in the JS shared structs proposal,
implementation of this behavior does not require any changes in the JS specification because this
proposal does not propose any user-programmable means in JS to create new kinds of shared objects.

We also expose realm-local globals with a `realm-local` option in the `WebAssembly.SharedGlobal`
constructor.

> Note: It would also be possible to reuse the existing constructors and add a `shared` property for
> testing rather than use new constructors, which allow testing with `instanceof`. This would be
> more similar (but not quite the same) as how shared `WebAssembly.Memory` works.

[sealed]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/seal
[tls-prototypes]: https://github.com/tc39/proposal-structs/blob/main/ATTACHING-BEHAVIOR.md

#### Realm-Local JS Function Wrappers

One of the most important requirements for this proposal is that there be some way for shared
functions to call JS functions, which are inherently unshared. Without this capability, shared
functions would not be able to directly interact with their environment on the Web. To allow shared
functions to call unshared JS functions, we extend the `WebAssembly.SharedFunction` constructor with
a new `realm-local: bool` option.

On the web, JavaScript always executes in a particular [realm]. An agent (roughly, a thread of
execution) may have multiple active realms within it, such as in the case of multiple iframes. Each
agent that executes JS has at least one realm, but may have more. Importantly, each realm has its
own copy of all JS built-ins and web APIs. For example, a main document's `Array` constructor is a
different function than an iframe's `Array` constructor.

When `realm-local` is true, the resulting `WebAssembly.SharedFunction` is a "realm-local function
wrapper." It behaves like a realm-local global initialized to `ref.null nofunc` on each realm, so if
it is called immediately after being shared with another realm, it will behave as though calling
`null`.  `WebAssembly.SharedFunction` has a `set` method that sets the wrapped function on the
current realm, and it throws an error if the current realm's wrapped function has already been set
to a non-null value. The `WebAssembly.SharedFunction` constructor with the `realm-local` option
supplied implicitly calls `set`, so it cannot be called again on that original thread (unless the
function was constructed wrapping `null` in the first place).

The `realm-local` option composes with the `async` option from JSPI. A shared function wrapper that
is also async is async on every realm and thread.

As an engine-internal optimization, setting a realm-local global to a realm-local function wrapper
can skip a layer of indirection and set the global to refer directly to the bound function, since it
will never change or be observed from another realm. This optimization only works if Wasm
"thread-local" globals are actually realm-local in the Web embedding. True thread-local globals can
be emulated on top of realm-local globals by ensuring that all realms on a thread trampoline through
functions from a single realm to enter WebAssembly.

[jspi]: https://github.com/WebAssembly/js-promise-integration
[realm]: https://tc39.es/ecma262/#realm

#### Waiter Queue API

In the same way the linear memory wait and notify operations are mirrored and interoperable with
those in JS, it will be useful to have JS methods to mirror the new `waitqueue.wait` and
`waitqueue.notify` instructions. More importantly, it will also be useful to have an async version
of the wait method analogous to `Atomics.waitAsync`.

To expose this functionality, we add a new shareable JS type, `WebAssembly.WaitQueue`. Its
realm-local prototype will have `wait`, `notify`, and `waitAsync` methods.

> Note: If it turns out to be useful, we could also add methods for atomically accessing and
> modifying a wait queue's control field. In the current design, that can only be done from
> WebAssembly.

### Binary format

#### Memory orderings

Just as the multi-memory proposal uses bit 6 of the `memarg` on memory access instructions to
signify that a memory index follows, we will reserve bits 4-5 to encode the memory ordering
(although we only use bit 4 in this proposal). For backward compatibility with the shared memory
proposal, 0b00 will encode sequentially consistent ordering. 0b01 will encode release-acquire
ordering. For memargs on non-atomic operations, the bits are not interpreted and must be 0b00. This
scheme allows encoding alignments of up to 32768 bytes using bits 0-3, so there is no danger that we
will need the newly reserved bits for alignment in the future, especially for atomic accesses.

The new instructions below do not have memarg immediates because they do not operate on memories.
They instead have `u32:ordering` immediates, which are 0 for sequentially-consistent ordering or 1
for release-acquire ordering. `atomic.fence` already has a reserved zero byte immediate, which we
now interpret as a u32:ordering immediate, allowing us to express release-acquire fences as well.

#### Types

The shared annotation occupies the same bit location as it does for memory types. In the threads
proposal, the `limits` flag byte on memories is extended such that if bit 1 (the second bit) is set,
the memory is `shared`. Likewise:
- tables are `shared` if the `limit` flag byte has bit 1 set
- globals are `shared` if the `mut` flag byte has bit 1 set

| type | opcode |
| ---- | ------ |
| `func shared [valtype*] -> [valtype*]` | 0x5d (-35 as s7) |
| `struct shared fieldtype*` | 0x5c (-36 as s7) |
| `array shared fieldtype` | 0x5b (-37 as s7) |
| `(shared nofunc)` | 0x5A |
| `(shared noextern)` | 0x59 |
| `(shared none)` | 0x58 |
| `(shared func)` | 0x57 |
| `(shared extern)` | 0x56 |
| `(shared any)` | 0x55 |
| `(shared eq)` | 0x54 |
| `(shared i31)` | 0x53 |
| `(shared struct)` | 0x52 |
| `(shared array)` | 0x51 |

#### Instructions

| Instructions | opcode | notes |
| ------------ | ------ | ----- |
| `global.atomic.get <u32:ordering> <globalidx>` | 0xFE 0x4F | |
| `global.atomic.set <u32:ordering> <globalidx>` | 0xFE 0x50 | |
| `global.atomic.rmw.add <u32:ordering> <globalidx>` | 0xFE 0x51 | valid for i32 and i64 reference globals. |
| `global.atomic.rmw.sub <u32:ordering> <globalidx>` | 0xFE 0x52 | valid for i32 and i64 reference globals. |
| `global.atomic.rmw.and <u32:ordering> <globalidx>` | 0xFE 0x53 | valid for i32 and i64 reference globals. |
| `global.atomic.rmw.or <u32:ordering> <globalidx>` | 0xFE 0x54 | valid for i32 and i64 reference globals. |
| `global.atomic.rmw.xor <u32:ordering> <globalidx>` | 0xFE 0x55 | valid for i32 and i64 reference globals. |
| `global.atomic.rmw.xchg <u32:ordering> <globalidx>` | 0xFE 0x56 | |
| `global.atomic.rmw.cmpxchg <u32:ordering> <globalidx>` | 0xFE 0x57 | valid for i32, i64, and eqref globals. |
| `table.atomic.get <u32:ordering> <tableidx>` | 0xFE 0x58 | valid for anyref tables. |
| `table.atomic.set <u32:ordering> <tableidx>` | 0xFE 0x59 | valid for anyref tables. |
| `table.atomic.rmw.xchg <u32:ordering> <tableidx>` | 0xFE 0x5A | valid for anyref tables. |
| `table.atomic.rmw.cmpxchg <u32:ordering> <tableidx>` | 0xFE 0x5B | valid for eqref tables. |
| `struct.atomic.get <u32:ordering> <typeidx> <fieldidx>` | 0xFE 0x5C | valid for i32, i64, and anyref fields. |
| `struct.atomic.get_s <u32:ordering> <typeidx> <fieldidx>` | 0xFE 0x5D | valid for i8 and i16 fields. |
| `struct.atomic.get_u <u32:ordering> <typeidx> <fieldidx>` | 0xFE 0x5E | valid for i8 and i16 fields. |
| `struct.atomic.set <u32:ordering> <typeidx> <fieldidx>` | 0xFE 0x5F | valid for i32, i64, and anyref fields. |
| `struct.atomic.rmw.add <u32:ordering> <typeidx> <fieldidx>` | 0xFE 0x60 | valid for i32 and i64 fields. |
| `struct.atomic.rmw.sub <u32:ordering> <typeidx> <fieldidx>` | 0xFE 0x61 | valid for i32 and i64 fields. |
| `struct.atomic.rmw.and <u32:ordering> <typeidx> <fieldidx>` | 0xFE 0x62 | valid for i32 and i64 fields. |
| `struct.atomic.rmw.or <u32:ordering> <typeidx> <fieldidx>` | 0xFE 0x63 | valid for i32 and i64 fields. |
| `struct.atomic.rmw.xor <u32:ordering> <typeidx> <fieldidx>` | 0xFE 0x64 | valid for i32 and i64 fields. |
| `struct.atomic.rmw.xchg <u32:ordering> <typeidx> <fieldidx>` | 0xFE 0x65 | |
| `struct.atomic.rmw.cmpxchg <u32:ordering> <typeidx> <fieldidx>` | 0xFE 0x66 | valid for i32, i64, and eqref fields. |
| `array.atomic.get <u32:ordering> <typeidx>` | 0xFE 0x67 | |
| `array.atomic.get_s <u32:ordering> <typeidx>` | 0xFE 0x68 | valid for i8 and i16 arrays. |
| `array.atomic.get_u <u32:ordering> <typeidx>` | 0xFE 0x69 | valid for i8 and i16 arrays. |
| `array.atomic.set <u32:ordering> <typeidx>` | 0xFE 0x6A | |
| `array.atomic.rmw.add <u32:ordering> <typeidx>` | 0xFE 0x6B | valid for i32 and i64 arrays. |
| `array.atomic.rmw.sub <u32:ordering> <typeidx>` | 0xFE 0x6C | valid for i32 and i64 arrays. |
| `array.atomic.rmw.and <u32:ordering> <typeidx>` | 0xFE 0x6D | valid for i32 and i64 arrays. |
| `array.atomic.rmw.or <u32:ordering> <typeidx>` | 0xFE 0x6E | valid for i32 and i64 arrays. |
| `array.atomic.rmw.xor <u32:ordering> <typeidx>` | 0xFE 0x6F | valid for i32 and i64 arrays. |
| `array.atomic.rmw.xchg <u32:ordering> <typeidx>` | 0xFE 0x70 | |
| `array.atomic.rmw.cmpxchg <u32:ordering> <typeidx>` | 0xFE 0x71 | valid for i32, i64, and eqref arrays. |

> TODO: Should we allow atomic arithmetic on i8 and i16 fields? Should we allow arithmetic on i31ref fields and table slots?

## Other Considerations (FAQ)

### How will toolchains need to change?

Toolchains like LLVM will be expected to mark all WebAssembly objects that a parallel function
touches with the `shared` annotation. It is unclear how difficult this might be and one conservative
approach (for the toolchain) is to simply mark all module objects as `shared` in certain thread
models. For more discussion: [How should toolchains apply `shared`?][toolchain-shared-discussion].

[toolchain-shared-discussion]: https://github.com/abrown/thread-spawn/discussions/5

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
`pthread_join`). As shown in [wasi-libc][wasi-libc-pthread-join] and Emscripten’s
[libc][emscripten-pthread-join], this can be implemented via the existing `wait` and `notify`
instructions.

[wasi-libc-pthread-join]: https://github.com/WebAssembly/wasi-libc/blob/bd950eb128bff337153de217b11270f948d04bb4/libc-top-half/musl/src/thread/pthread_join.c
[emscripten-pthread-join]: TODO

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

### Why not make shared a property of references instead of heap types?

An alternative to marking heap types as shared would be to mark sharedness on reference types
instead, giving us `(ref $foo)`, `(ref null $foo)`, `(ref shared $foo)`, and `(ref null shared
$foo)`. However, sharedness is a property of a heap allocation, not of individual references to it,
and it depends on the contents of the heap type, so it is slightly more natural to put the
annotation on the heap type.

### What about the stack-switching proposal?

This proposal should work in concert with the [stack-switching] proposal. In particular, the
validation constraints on shared functions are intended to be future-compatible with shared
continuation references.

[stack-switching]: https://github.com/WebAssembly/stack-switching
