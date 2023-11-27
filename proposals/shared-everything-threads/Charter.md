# WebAssembly Threads Subgroup Charter

The Threads Subgroup is a sub-organization of the
[WebAssembly Community Group](https://www.w3.org/community/webassembly/)
of the W3C.
As such, it is intended that its charter align with that of the CG. In particular, 
the sections of the [CG charter](https://webassembly.github.io/cg-charter/) relating to
[Community and Business Group Process](https://webassembly.github.io/cg-charter/#process),
[Contribution Mechanics](https://webassembly.github.io/cg-charter/#contrib),
[Transparency](https://webassembly.github.io/cg-charter/#transparency),
and
[Decision Process](https://webassembly.github.io/cg-charter/#decision)
also apply to the Subgroup.

## Goals

The mission of this subgroup is to provide a forum for collaboration on the standardisation of threading support for WebAssembly.

## Scope

The Subgroup will consider topics related to threading for Wasm, including:

- syntax extensions for allowing module fields and data to be shared between threads,
- instructions for interacting with shared data and synchronizing between threads,
- syntax extensions, instructions, and APIs for thread-local storage, thread lifetime management, and other threading-related features
- type system rules for validating such uses of such syntax extensions and instructions,
- extensions to the Wasm memory model describing the semantics of such instructions,
- APIs for accessing and managing shared data and threads outside Wasm or a Wasm engine,
- tooling and implementation considerations for languages targeting the new threading features,
- code generation and implementation considerations for Wasm engines implementing the new features.

## Deliverables

### Specifications

The Subgroup may produce several kinds of specification-related work output:

- new specifications in standards bodies or working groups
  (e.g. W3C WebAssembly WG or Ecma TC39),

- new specifications outside of standards bodies
  (e.g. similar to the LLVM object file format documentation in Wasm tool conventions).

### Non-normative reports

The Subgroup may produce non-normative material such as requirements
documents, recommendations, and case studies.

### Software

The Subgroup may produce software related to garbage collection in Wasm
(either as standalone libraries, tooling, or integration of interface-related functionality in existing threading software).
These may include

- extensions to the Wasm reference interpreter,
- extensions to the Wasm test suite,
- compilers and tools for producing code that uses Wasm threading extensions,
- tools for implementing Wasm with threading,
- tools for debugging programs using Wasm threading extensions.

## Amendments to this Charter and Chair Selection

This charter may be amended, and Subgroup Chairs may be selected by vote of the full WebAssembly Community Group.
