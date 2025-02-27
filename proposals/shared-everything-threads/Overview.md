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
  (type $shared (shared (func)))

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
    call $baz ;; validation error!
  )
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

For consistency with other shared annotations in which `shared` comes first, the
existing text format for shared memory types is redefined to be an abbreviation
of the new text format:

```
memtype ::= it:index_type lim:limits => unshared it lim
          | 'shared' it:index_type lim:limits => shared it lim

index_type limits 'shared'  == 'shared' index_type limits
```

#### Tables

The abstract syntax of `tabletype` is extended:

```
tabletype ::= share index_type limits reftype
```

Similarly, the text format is extended:

```
tabletype ::= it:index_type lim:limits rt:reftype => unshared it lim rt
            | 'shared' it:index_type lim:limits rt:reftype => shared it lim rt
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
globaltype ::= share mut valtype
```

Similarly, the text format is extended:

```
globaltype ::= t:valuetype => unshared const t
             | '(' 'mut' t:valuetype ')' => unshared var t
             | '(' 'shared' t:valuetype ')' => shared const t
             | '(' 'shared '(' 'mut' t:valuetype ')' ')' => shared var t
```

A `globaltype` is valid as shared only if its `valtype` is valid as shared.

The existing `global.get` and `global.set` instructions provide unordered access to shared globals.
New instructions adding atomic accesses to globals are presented [below][new-instructions]. The new
instructions are valid for both shared and unshared globals.

[new-instructions]: TODO

> Note: Values written to globals are always treated atomically. That is, each read of a global that
> is subject to multiple simultaneous writes will always observe only one of the written values, never
> a (bytewise or otherwise) combination of the two.
>
> For certain large types such as `v128`, runtimes may need to do extra work to provide this guarantee.

#### Functions

The `typeidx` in the structure of a function determines whether the function is shared, so the
syntax of functions does not need to be extended. If a function is shared, its locals and body must
validate as shared. An expression validates as shared if its instruction sequence validates as
shared. An instruction sequence validates as shared if each instruction in the sequence validates as
shared. An instruction validates as shared if its instruction type validates as shared, which is the
case if each of its input and output value types validates as shared.

The sharedness of function types is described below along with the sharedness of other `comptype`s.

> Note: If this validation turns out to be too strict to be usable, we may relax the validation of
> functions to allow the locals and intermediate types in the body of shared functions to validate
> as unshared. This would have consequences for the permissibility of shared continuations. We would
> still have to extend validation of instructions to prohibit referencing unshared module fields in
> shared contexts.

#### Heap Types

The syntax of `absheaptype` (abstract type) is extended to make `shareabsheaptype`, which will be
used everywhere `absheaptype` is used today. This extension allows for e.g. `(shared any)` and
`(shared func)` to be used in place of `any` and `func`.

```
shareabsheaptype ::= share absheaptype
```

The text format is extended:

```
shareabsheaptype ::= ht:absheaptype => unshared ht
                   | '(' 'shared' ht:absheaptype ')' => shared ht
```

Similarly, the syntax of `comptype` (composite type) is extended to make `sharecomptype`, which will
be used everywhere `comptype` is used today. This allows for declaring e.g. `(shared (func ...))` or
`(shared (struct ...))` types.

```
sharecomptype ::= share comptype
```

The text format is extended:

```
sharecompttype ::= ct:comptype => unshared ct
                 | '(' 'shared' ct:comptype ')' => shared ct
```

The validation judgments for all kinds of types are parameterized by `share`. In general, the
conclusions are `ok(shared)` iff the premises are `ok(shared)`. The validation rules for
`absheaptype` and `comptype` are extended so that they are only `ok(shared)` if the heap type is
shared. Number types and vector types are always valid as shared.

Similarly, we need to be able to validate that a `typeidx` is valid as shared. To achieve this we
can update the auxiliary function `expand` to return a `sharecomptype` instead of a `comptype` for a
type index and pattern match on the results. A `typeidx` `x` being `ok(shared)` requires
`expand(C.types[x]) = shared comptype`.

Shared and unshared types are never subtypes of one another. The subtype relationships among shared
abstract types mirrors the subtype relationships among unshared abstract types.

> Note: Should i31ref behave like an i32 in that it is usable in both shared and non-shared contexts
> without needing to have a separate `shared` variant? This seems like it would be ok, except we
> would still need a shared version to interoperate with `(shared eq)`

#### Reference Types

Since heap types already include sharedness, reference types do not need their own additional shared
annotations.

#### Element Segments

The syntax of `elem` is extended to refer to a new `elemtype` that says whether or not an element
segment is shared.

```
elemtype ::= share reftype
elem ::= {type elemtype, init vec(expr), mode elemmode}
```

The text format is extended:

```
elemtype ::= t:reftype => unshared t
           | '(' 'shared' t:reftype ')' => shared t
elemlist ::= et:elemtype y*:vec(elemexpr) => (type et, init y*)
```

An element segment is valid as shared only if its reftype is valid as shared. Instructions that
refer to element segments (e.g. `elem.drop`) are only valid as shared if their element segment is
shared.

> Note: We could alternatively make an element segment shared iff its elements refer to shared data,
> however, we have chosen to add an explicit annotation to avoid future complications if we ever
> want to add first-class references to element segments.

#### Data Segments

The syntax of data segments is extended to refer to a new `datatype` that says whether or not a data
segment is shared:

```
datatype ::= share
data ::= {type datatype, init vec(byte), mode datamode}
```

The text format is extended:

```
data ::= '(' 'data' id? share b*:datastring ')' => {type share, init b*, mode passive}
       | '(' 'data' id? x:memuse '(' 'offset' e:expr ')' share b*:datastring ')'
           => {type share, init b*, mode active {memory x, offset e}}
share ::= eps => unshared
        | 'shared' => shared
```

Instructions that refer to data segments (e.g. `data.drop`) are only valid as shared if their data
segment is shared.

> Note: We could alternatively make all data segments shared, just like we do for other
> non-reference data. This would require allocating all data segments in a shared heap, though,
> which may have performance or memory use implications. We should evaluate whether there is a
> reason to support non-shared data segments once we have prototype implementations.

#### Exception tags

The syntax of tags does not need to be extended. They are shared iff their function types are
shared. Shared tags are used to throw and catch `(shared exnref)`. Instructions that reference tags
(e.g. `throw` and `try_table`) only validate as shared if all the tags they reference are shared.

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

> Note: In a Web context, "thread-local globals" could alternatively be realm-local. See the related
> discussion in the JS API section below.

[tls-discussion]: https://github.com/WebAssembly/shared-everything-threads/discussions/12
[Google document]: https://docs.google.com/document/d/1b8QuRhSo_JpMKGtv9zIEubuQ-jW3dzf4NpLWlND06Zw/edit#heading=h.3fybxwjpw6r6

#### Syntax and validation

The syntax of `globaltype` is extended again:

```
globalshare = share | 'thread_local'
globaltype = globalshare mut valtype
```

Thread-local globals are valid only if their valtypes are valid as shared. Their types must also be
defaultable. Declared (i.e. non-imported) thread-local globals must be initialized to the default
value for their types.

> Note: We don't allow non-shared references in thread-local globals because then they would not be
> able to be accessed from shared functions. We could always relax this restriction, especially if
> we relax the restrictions on intermediate values in shared functions.

> Note: The restricted initializers are a conservative starting point that minimizes the work the
> engine must do to initialize the thread-local storage. If useful, we could relax this to allow any
> constant instructions in thread-local initializers, or perhaps only non-allocating constant
> instructions. If we allow allocations, we can choose whether they happen once or separately on
> each thread.

#### Execution

Thread-local globals have different values on different threads. `global.get` and `global.set` on
thread-local globals access only the value for that global for the current thread. There is no way
to access a thread's thread-local global value directly from any other thread.

Thread-local globals can be imported and exported just like normal or shared globals.

#### Implementation

One of the nice things about this thread-local storage design is that it is declarative enough to
admit multiple implementation strategies. A naive implementation would be to store thread-local
global values in a concurrent hash map keyed by global and thread IDs. Thread-local global accesses
would lower to accesses to this hash map.

A more sophisticated implementation would be to store TLS blocks containing all the thread-local
globals for a module in a map keyed by pairs of instance and thread IDs. Thread-local global
accesses would then lower to accesses to the hash map to get the TLS block base address followed by
applications of an offset to access the global within the block. This implementation has the
advantage that the TLS block base address can be reused across accesses to multiple thread-local
globals, and in the extreme could be pinned to a register to eliminate all but the initial hash map
access for each instance on each thread.

A further optimization would be to store a table of TLS bases indexed by thread ID on each instance
or indexed by instance ID in engine-native TLS, depending on whether the number of threads or
instances in the system is expected to be lower. Assuming indices can be safely reused, this allows
the lookup to use direct indexing rather than hashing, speeding up the fast path.

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
bottom types), `waitqueue` is a final subtype of `(rec (type (sub (struct shared (field (mut i32)))))).0`
, meaning it is a struct with one visible `i32` field that can be accessed with all the standard
struct accessors as well as the new atomic struct accessors. This field is the futex control field
that is atomically checked when waiting on the waiter queue. `waitqueue` is also always shared.
There is no non-shared version of it.

> Note: Should we have a non-shared version of `waitqueue` just for orthogonality? The type would be
> useless, but orthogonality would be helpful for optimizers.

To wait on and notify a particular `waitqueueref`, there are two additional instructions:

 - `waitqueue.wait: [waitqueueref, i32, i64] -> [i32]`

This instruction behaves just like `memory.atomic.wait32` and `memory.atomic.wait64`: the first
operand is the wait queue to wait on, the `i32` operand is the expected value of
the control field, and the `i64` operand is a relative timeout in nanoseconds. The return value is
`0` when the wait succeeded and the current thread was woken up by a notify, `1` when the thread did
not go to sleep because the control field did not have the expected value, or `2` because the
timeout expired.

Like the existing linear memory wait instructions, `waitqueue.wait` disallows spurious wakeups.

> Note: We should perhaps revisit allowing spurious wakeups, since disallowing them makes various
> kinds of interesting instrumentation impossible.

- `waitqueue.notify: [waitqueueref, i32] -> [i32]`

This instruction behaves just like `memory.atomic.notify`: The first operand is the wait queue to
wait on and the `i32` operand is the maximum number of waiters to wake up. The result is the number
of waiters that were actually woken up.

> Note: It may also be necessary to have versions of the waitqueue where the control field is an i64
> or some subtype of shared eqref. This would allow more bits to be stored in the control word and
> would allow construction of things like shared WasmGC queues. We should consider parameterizing
> `waitqueue` and its operations with the control word type.

Threads that are waiting indefinitely on a wait queue that is no longer reachable on threads that
might possibly notify it are eligible to be garbage collected along with the wait queue itself. This
may be observable if it causes finalizers registered in the host to be fired.

[wait-notify]: https://github.com/WebAssembly/threads/blob/main/proposals/threads/Overview.md#wait-and-notify-operators

### Spinlock Relaxation: `pause`

Efficient lock implementations do not just use waiter queues; they also use bounded spinlocks. To
improve the performance and power efficiency of these spinlocks, we introduce a `pause` instruction
that is semantically a no-op, but is lowered to architecture-specific instructions like
[`PAUSE`][pause]. This instruction was [discussed][threads-spinloop] on the original threads
proposal, but did not make it into the spec at that point. There is also TC39
[proposal][tc39-microwait] adding a higher-level version of this primitive to JS as
`Atomics.microwait`.

[pause]: https://www.felixcloutier.com/x86/pause.html
[threads-spinloop]: https://github.com/WebAssembly/threads/issues/15
[tc39-microwait]: https://github.com/tc39/proposal-atomics-microwait

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

For consistency with existing rules about tearing accesses to shared linear memory, accesses to
shared globals, structs, and arrays follow these rules about when tearing is allowed:

 - Atomic accesses never tear.

 - Accesses to reference-typed fields and globals never tear. (It would be unsafe to allow such
   accesses to tear.)

 - Accesses to i8, i16, and i32 fields and globals never tear. (For linear memory, such accesses are
   only tear-free if they are aligned, but all struct, array, and global accesses are assumed to be
   a aligned.)

 - All other accesses can tear.

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

#### Shared Annotations

Just like the original threads proposal exposed shared memories in the JS API, this proposal exposes
shared functions, tables, globals, tags, and exceptions. Unlike shared `WebAssembly.Memory` objects,
which are actually realm-local wrappers around SharedArrayBuffers (which are in turn realm-local
wrappers around the underlying shared buffers), the new shared tables, globals, tags, and exceptions
will be shared JS objects that do not require rewrapping. Such shared objects have fixed layout and
behave mostly like [sealed] JS objects, and their identities survive round trips via postMessage
across thread boundaries. Additionally, to facilitate calling methods on these shared objects, they
have [realm-local prototypes][tls-prototypes] just like shared structs are proposed to have in JS.

The shared versions of the JS API types will share constructors with their existing unshared
versions, but will take an additional `shared: bool` option in their type descriptor arguments.
The type descriptor arguments to these constructors will be extended:

 - `WebAssembly.Function`
 - `WebAssembly.Table`
 - `WebAssembly.Global`
 - `WebAssembly.Tag`

These new shared objects provide an API that mirrors that provided by their unshared analogues. Like
shared memories, shared tables must be initialized with a maximum size. Shared
`WebAssembly.Exception` objects are created by passing shared `WebAssembly.Tag` objects to the
`WebAssembly.Exception` constructor. Storing unshared data into a shared global or table throws an
exception. In addition, all of these objects will be extended with a read-only `shared` boolean
property that can be used to test whether the object is shared or not.

> Note: It would be good to get the sharedness boolean into the types used with the proposed [type
> reflection JS
> API](https://github.com/WebAssembly/js-types/blob/main/proposals/js-types/Overview.md).

While the behaviors chosen here mirror those proposed in the JS shared structs proposal,
implementation of this behavior does not require any changes in the JS specification because this
proposal does not propose any user-programmable means in JS to create new kinds of shared objects.

We also expose thread-local globals with a `thread-local` option in the `WebAssembly.Global`
constructor's type descriptor. The `thread-local` option, if it is set, takes precedence over the
`shared` option.

> Note: It would also be possible to introduce new `Shared` variants of the constructors rather than
> using an option bag argument. This would allow testing with `instanceof` instead of the `shared`
> read-only property.

[sealed]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/seal
[tls-prototypes]: https://github.com/tc39/proposal-structs/blob/main/ATTACHING-BEHAVIOR.md

#### Thread-Local JS Function Wrappers

One of the most important requirements for this proposal is that there be some way for shared
functions to call JS functions, which are inherently unshared. Without this capability, shared
functions would not be able to directly interact with their environment on the Web. To allow shared
functions to call unshared JS functions, we extend the `WebAssembly.Function` constructor with a new
`thread-local: bool` option.

When `thread-local` is true, the resulting `WebAssembly.Function` is a "thread-local function
wrapper." It behaves like a thread-local global initialized to `ref.null nofunc` on each thread, so
if it is called immediately after being shared with another thread, it will behave as though calling
`null`. `WebAssembly.Function` has a `initialize` method that sets the wrapped function on the
current thread, and it throws an error if the current thread's wrapped function has already been set
to a non-null value. The `WebAssembly.Function` constructor with the `thread-local` option supplied
implicitly calls `initialize`, so it cannot be called again on that original thread (unless the
function was constructed wrapping `null` in the first place).

The `thread-local` option composes with the `async` option from JSPI. A shared function wrapper that
is also async is async on every realm and thread.

As an engine-internal optimization, setting a thread-local global to a thread-local function wrapper
can skip a layer of indirection and set the global to refer directly to the bound function, since it
will never change or be observed from another thread. This optimization only works because the
dynamic contexts of thread-local globals and thread-local function wrappers are the same, i.e. an
entire thread or agent.

> Note: On the web, JavaScript always executes in a particular [realm]. An agent (roughly, a thread
> of execution) may have multiple active realms within it, such as in the case of multiple iframes.
> Each agent that executes JS has at least one realm, but may have more. Importantly, each realm has
> its own copy of all JS built-ins and web APIs. For example, a main document's `Array` constructor
> is a different function than an iframe's `Array` constructor.
>
> It would be possible for thread-local globals and function wrappers to actually be realm-local
> instead, which would allow the functions to be re-bound on each realm. This would be consistent
> with the way many things work on the Web, but would be less consistent with how shared memory
> multithreading works in Wasm today, where functions are bound (i.e. imported) once per thread, not
> once per realm. Note that the prototypes on shared objects in the JS API, including function
> wrappers, will be realm-local regardless of what context we choose for thread-local globals and
> function wrappers
>
> TODO: JSPI no longer uses an option on `WebAssembly.Function`. We should consider using separate
> wrappers that compose with JSPI's wrappers and share their benefit of not requiring type
> reflection.

[jspi]: https://github.com/WebAssembly/js-promise-integration
[realm]: https://tc39.es/ecma262/#realm

#### Thread-Bound Data

Just as WebAssembly users expect to be able to call JS functions, they also expect to be able to
reference JS objects. For example, a language compiled to shared WasmGC may need to hold references
to DOM nodes in some of its shared objects. Normally this would be disallowed because DOM nodes are
not shareable and shared-to-unshared references are disallowed. To bridge this gap, we need a
utility that allows shared-to-unshared objects as an exception to the general rule, assuming that
browser GCs can support such an exception (see the discussion in the FAQ below). To avoid
complicating the core language and forcing every engine to support shared-to-unshared edges, this
utility is a new JS API.

`ThreadBoundData` is a shared JS object that wraps any other arbitrary JS object. `ThreadBoundData`
provides access to the wrapped object via a `get` method that throws an exception when called on any
agent besides the one on which the `ThreadBoundData` wrapper was created, ensuring that the wrapped
object can be observed only on one thread, even if the `ThreadBoundData` is shared between multiple
threads. Like other shared JS objects, it can be converted to a shared externref on the JS/Wasm
boundary.

See the FAQ below for discussion on how the lifetime of the shared `ThreadBoundData` may affect the
lifetime of the wrapped object.

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
signify that a memory index follows, we reserve bit 5 to signify that an ordering immediate follows.
For backward compatibility, if bit 5 is not set, the instruction uses `seqcst` ordering for all
loads and stores it executes. If a memory index immediate is also present, the ordering immediate
follows it.

Ordering immediates are encoded as `u8`s. Read-modify-write operations require two orderings: one
for the read and one for the write. For RMWs, the low four bits of the `u8` encode the read ordering
and the high four bits encode the write ordering. For other atomic operations, the low four bits
encode the ordering and the high four bits must be 0.

| ordering  | encoding |
|-----------|----------|
| `seqcst`  | `0b0000` |
| `acqrel`  | `0b0001` |

Reads with the `acqrel` ordering are acquire reads and writes with the `acqrel` ordering are release
writes. Fences with the `acqrel` ordering are full acquire-release fences.

> Note: We may want to add separate `acquire` and `release` orderings to express weaker fences.

RMW operations take two ordering immediates, but these immediates must match, i.e. an RMW op can
only have two `seqcst` orderings or two `acqrel` orderings. This restriction may be relaxed in the
future.

> Note: We may also want to give cmpxchg a third ordering, since some compilation schemes are able
> to give its read different orderings depending on whether it succeeds or fails.

The new instructions below do not have memarg immediates because they do not operate on memories, so
they unconditionally take `u8` ordering immediates. `atomic.fence` already has a reserved zero byte
immediate, which we now interpret as a `u8` ordering immediate, allowing us to express fences with
different orderings as well.

#### Types

The shared annotation occupies the same bit location as it does for memory types. In the threads
proposal, the `limits` flag byte on memories is extended such that if bit 1 (the second bit) is set,
the memory is `shared`. Likewise:

 - tables are `shared` if the `limit` flag byte has bit 1 set, just like for memories.
 - globals are `shared` if the `mut` flag byte has bit 1 set.
 - element segments are `shared` if bit 3 of their first u32 field is set (bits
   0-2 are already used).
 - data segments are `shared` if bit 3 of their first u32 fields is set (only bits
   0-1 are already used, but we choose bit 3 for consistency with element segments).


The binary formats for `waitqueue`, shared abstract heap types, and `sharecomptype` are given below.

```
absheaptype ::= ... | 0x68 (-24 as s7) => waitqueue
heaptype ::= ... | 0x65 ht:absheaptype => (shared ht)
sharecomptype ::= 0x65 ct:comptype => (shared ct) | ct:comptype => (unshared ct)
```

#### Instructions

Several existing instructions need to be updated to accept references to shared heap types in
addition to the references to unshared heap types they already accept. To allow this flexibility,
our principal type rule is amended to allow unconstrained sharedness metavariables, just like it
allows unconstrained nullability metavariables. In general, all instructions that operate on
references to unshared heap types are allowed to operate on references to shared heap types as well.

This is the full list of instructions that need to be updated to accept references to shared heap
types by way of an unconstrained sharedness metavariable:

 - `any.convert_extern`
 - `extern.convert_any`
 - `ref.eq` (the sharedness of both operands must match)
 - `i31.get_s`
 - `i31.get_u`
 - `array.len`

Most instructions that create reference values (e.g. `struct.new`, `ref.func`, `ref.null`) do not
need new versions for allocating shared references because their type annotation immediates
determine whether the reference will be shared or not. In the case of `ref.func`, the reference will
be shared if and only if the referred-to function is shared. An exception is `ref.i31`, which takes
no immediate that can determine the sharedness of the result. We therefore need a new
`ref.i31_shared` instruction as well.

To maintain the forward principal types property, we additionally need to introduce a bottom
sharedness, `bot-share`, that is used only during validation. This is analogous to the bottom heap
type also used during validation. `bot-share` is a subtype of both `shared` and `unshared`, so for
example `(ref null (bot-share any))` is a subtype of both `anyref` and `(ref null (shared any))`.
Popping a reference from an empty, unreachable stack produces a value of type `(ref (bot-share
bot))`. A subsequent `any.convert_extern`, for example, would then push a `(ref (bot-share any))`.

In addition, the following instructions are introduced:

| Instructions | opcode | notes |
| ------------ | ------ | ----- |
| `pause` | 0xFE 0x04 | |
| `global.atomic.get <u8:ordering> <globalidx>` | 0xFE 0x4F | valid for i32, i64, and <: anyref globals. |
| `global.atomic.set <u8:ordering> <globalidx>` | 0xFE 0x50 | valid for i32, i64, and <: anyref globals. |
| `global.atomic.rmw.add <u8:ordering> <globalidx>` | 0xFE 0x51 | valid for i32 and i64 globals. |
| `global.atomic.rmw.sub <u8:ordering> <globalidx>` | 0xFE 0x52 | valid for i32 and i64 globals. |
| `global.atomic.rmw.and <u8:ordering> <globalidx>` | 0xFE 0x53 | valid for i32 and i64 globals. |
| `global.atomic.rmw.or <u8:ordering> <globalidx>` | 0xFE 0x54 | valid for i32 and i64 globals. |
| `global.atomic.rmw.xor <u8:ordering> <globalidx>` | 0xFE 0x55 | valid for i32 and i64 globals. |
| `global.atomic.rmw.xchg <u8:ordering> <globalidx>` | 0xFE 0x56 | valid for i32, i64, and <: anyref globals. |
| `global.atomic.rmw.cmpxchg <u8:ordering> <globalidx>` | 0xFE 0x57 | valid for i32, i64, and <: eqref globals. |
| `table.atomic.get <u8:ordering> <tableidx>` | 0xFE 0x58 | valid for <: anyref tables. |
| `table.atomic.set <u8:ordering> <tableidx>` | 0xFE 0x59 | valid for <: anyref tables. |
| `table.atomic.rmw.xchg <u8:ordering> <tableidx>` | 0xFE 0x5A | valid for <: anyref tables. |
| `table.atomic.rmw.cmpxchg <u8:ordering> <tableidx>` | 0xFE 0x5B | valid for <: eqref tables. |
| `struct.atomic.get <u8:ordering> <typeidx> <fieldidx>` | 0xFE 0x5C | valid for i32, i64, and <: anyref fields. |
| `struct.atomic.get_s <u8:ordering> <typeidx> <fieldidx>` | 0xFE 0x5D | valid for i8 and i16 fields. |
| `struct.atomic.get_u <u8:ordering> <typeidx> <fieldidx>` | 0xFE 0x5E | valid for i8 and i16 fields. |
| `struct.atomic.set <u8:ordering> <typeidx> <fieldidx>` | 0xFE 0x5F | valid for i8, i16, i32, i64, and <: anyref fields. |
| `struct.atomic.rmw.add <u8:ordering> <typeidx> <fieldidx>` | 0xFE 0x60 | valid for i32 and i64 fields. |
| `struct.atomic.rmw.sub <u8:ordering> <typeidx> <fieldidx>` | 0xFE 0x61 | valid for i32 and i64 fields. |
| `struct.atomic.rmw.and <u8:ordering> <typeidx> <fieldidx>` | 0xFE 0x62 | valid for i32 and i64 fields. |
| `struct.atomic.rmw.or <u8:ordering> <typeidx> <fieldidx>` | 0xFE 0x63 | valid for i32 and i64 fields. |
| `struct.atomic.rmw.xor <u8:ordering> <typeidx> <fieldidx>` | 0xFE 0x64 | valid for i32 and i64 fields. |
| `struct.atomic.rmw.xchg <u8:ordering> <typeidx> <fieldidx>` | 0xFE 0x65 | valid for i32, i64, and <: anyref fields. |
| `struct.atomic.rmw.cmpxchg <u8:ordering> <typeidx> <fieldidx>` | 0xFE 0x66 | valid for i32, i64, and <: eqref fields. |
| `array.atomic.get <u8:ordering> <typeidx>` | 0xFE 0x67 | valid for i32, i64, and <: anyref arrays. |
| `array.atomic.get_s <u8:ordering> <typeidx>` | 0xFE 0x68 | valid for i8 and i16 arrays. |
| `array.atomic.get_u <u8:ordering> <typeidx>` | 0xFE 0x69 | valid for i8 and i16 arrays. |
| `array.atomic.set <u8:ordering> <typeidx>` | 0xFE 0x6A | valid for i8, i16, i32, i64, and <: anyref arrays. |
| `array.atomic.rmw.add <u8:ordering> <typeidx>` | 0xFE 0x6B | valid for i32 and i64 arrays. |
| `array.atomic.rmw.sub <u8:ordering> <typeidx>` | 0xFE 0x6C | valid for i32 and i64 arrays. |
| `array.atomic.rmw.and <u8:ordering> <typeidx>` | 0xFE 0x6D | valid for i32 and i64 arrays. |
| `array.atomic.rmw.or <u8:ordering> <typeidx>` | 0xFE 0x6E | valid for i32 and i64 arrays. |
| `array.atomic.rmw.xor <u8:ordering> <typeidx>` | 0xFE 0x6F | valid for i32 and i64 arrays. |
| `array.atomic.rmw.xchg <u8:ordering> <typeidx>` | 0xFE 0x70 | valid for i32, i64, and <: anyref arrays. |
| `array.atomic.rmw.cmpxchg <u8:ordering> <typeidx>` | 0xFE 0x71 | valid for i32, i64, and <: eqref arrays. |
| `ref.i31_shared` | 0xFB 0x1F | |

Atomic accesses to references are deliberately restricted to anyref, shared anyref, and their
subtypes because other references (e.g. funcref and externref) may have arbitrarily large
representations that do not allow for efficient atomic accesses on the underlying hardware. Engines
still must ensure that non-atomic accesses to any reference do not tear.

The operands of RMW instructions are typed the same as operands to the corresponding `set`
instructions. For example, the typing of `struct.atomic.rmw.xchg` is as follows:

```
struct.atomic.rmw.add typeidx fieldidx

C |- struct.atomic.rmw.add x y : [(ref null x) t] -> [t]
 - C.types[x] = struct fields
 - fields[y] = mut t
 - t = i32 \/ t = i64 \/ (C |- t <: anyref) \/ (C |- t <: (ref null (shared any))
```

Cmpxchg operations take an additional operand for the expected value. Its type may be a supertype of
the field type, as long as it is still equality-comparable. For example:

```
struct.atomic.rmw.cmpxchg typeidx fieldidx

C |- struct.atomic.rmw.cmpxchg x y : [(ref null x) t1 t2] -> [t2]
 - C.types[x] = struct fields
 - fields[y] = mut t2
 - (t2 = i32 /\ t1 = i32) \/
   (t2 = i64 /\ t1 = i64) \/
   ((C |- t2 <: eqref) /\ t1 = eqref) \/
   ((C |- t2 <: (ref null (shared eq))) /\ t1 = (ref null (shared eq)))
```

> Note: Should we allow atomic arithmetic on i8 and i16 fields? Should we allow arithmetic on i31ref
> fields and table slots?

> Note: Should we allow atomic accesses to other reference types as well, despite their possible
> large sizes? It may be the case that strategies to prevent tearing also admit synchronizing
> accesses.

## Other Considerations (FAQ)

### How will toolchains used `shared`?

Because of its virality—`shared` things can only refer to other `shared` things—we expect that
toolchains using shared-everything threads will switch entirely to emitting `shared` items and will
not use non-shared items at all, except perhaps in special cases around host interop.

### How will GCs and engines need to change?

GCs that historically assume that there is only a single mutator thread will need to handle heaps
with multiple mutators running in parallel. Since sharedness is part of static types, engines can
know at allocation or access time when they need to do something different for shared objects, so it
is possible for them to add a separate concept of shared heap on top of their existing
single-threaded heaps. This might be a reasonable intermediate architecture, even for engines that
plan on eventually switching to having shared heaps as their only primitive.

References from shared objects to unshared are normally disallowed, but we could choose to support
them in limited cases to make TLS or thread-bound data more ergonomic. If we do, there are varying
guarantees implementations could make about how the GC treats these references. (As usual, the spec
will say nothing about how garbage is collected.)

The weakest behavior would be that shared-to-unshared references never suffice to keep the unshared
object alive, i.e. they are only allowed as weak references. Users would have to root unshared
objects for the duration of their intended lifetimes, so this is no more expressive than simply
disallowing shared-to-unshared references.

The strongest behavior would be that shared-to-unshared references behave exactly like normal
references with respect to garbage collection. To collect cross-heap cycles, the garbage collector
would need visibility over the heaps from all threads.

An intermediate behavior would be the strong behavior but without cross-heap cycle collection. This
is as expressive as the weak behavior if we were to hypothetically augment it by allowing shared
objects to be keys in `FinalizationRegistry` but not `WeakMap` (which would be too inconsistent for
us to actually ship). The `FinalizationRegistry` would be able to automatically manage the lifetimes
of rooted unshared objects as long as they do not cyclically keep the shared objects that refer to
them alive.

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
[emscripten-pthread-join]: https://github.com/emscripten-core/emscripten/blob/main/system/lib/libc/musl/src/thread/pthread_join.c

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
