# [Self-Review Questionnaire: Security and Privacy](https://w3c.github.io/security-questionnaire/)

__01.  What information does this feature expose,
     and for what purposes?__

This feature does not directly expose any information because it is a pure computation feature in WebAssembly.
However, it may indirectly expose information about the timing and scheduling of computation,
including the timing effects of browser implementation details
(e.g. timing of optimizations in the JS/Wasm engine, browser scheduling)
and the timing effects of the underlying computation platform
(e.g. OS scheduling, hardware performance).

Notably, all of this timing information is already exposed by SharedArrayBuffer and related primitives.
This feature should not expose any new information.

__02.  Do features in your specification expose the minimum amount of information
     necessary to implement the intended functionality?__

Yes. There is no way around indirectly exposing timing information in some form
while achieving our goal of enabling high-performance shared-memory multithreaded applications on the Web,
since shared memory allows fine-grained timers to be built in userspace.

__03.  Do the features in your specification expose personal information,
     personally-identifiable information (PII), or information derived from
     either?__

Yes, to the extent that indirectly exposed timing information may be used to identify
the user's computing platform or system load characteristics.

__04.  How do the features in your specification deal with sensitive information?__

The new shared objects will only be shareable across threads
(and therefore usable to indirectly expose timing information)
in cross-origin isolated contexts.
In these contexts,
SharedArrayBuffer would already be available to expose the same information.

__05.  Does data exposed by your specification carry related but distinct
     information that may not be obvious to users?__

Not beyond the timing information already discussed.

__06.  Do the features in your specification introduce state
     that persists across browsing sessions?__

No.

__07.  Do the features in your specification expose information about the
     underlying platform to origins?__

Yes, indirectly via the timing information already discussed.

__08.  Does this specification allow an origin to send data to the underlying
     platform?__

Not directly, but perhaps indirectly via timing side channels.

__09.  Do features in this specification enable access to device sensors?__

No.

__10.  Do features in this specification enable new script execution/loading
     mechanisms?__

No.

__11.  Do features in this specification allow an origin to access other devices?__

No.

__12.  Do features in this specification allow an origin some measure of control over
     a user agent's native UI?__

No.

__13.  What temporary identifiers do the features in this specification create or
     expose to the web?__

None.

__14.  How does this specification distinguish between behavior in first-party and
     third-party contexts?__

The new shared objects can only be shared between threads in cross-origin isolated contexts,
so third parties must opt-in to making their resources available in a browsing context group
that shares objects between threads.

__15.  How do the features in this specification work in the context of a browser’s
     Private Browsing or Incognito mode?__

They work the same as in regular mode.

__16.  Does this specification have both "Security Considerations" and "Privacy
     Considerations" sections?__

[Yes](https://github.com/WebAssembly/shared-everything-threads/blob/main/proposals/shared-everything-threads/Overview.md#what-are-the-security-considerations).

__17.  Do features in your specification enable origins to downgrade default
     security protections?__

No.

__18.  What happens when a document that uses your feature is kept alive in BFCache
     (instead of getting destroyed) after navigation, and potentially gets reused
     on future navigations back to the document?__

Because this is a pure compute feature, there are no directly observable effects.

__19.  What happens when a document that uses your feature gets disconnected?__

Because this is a pure compute feature, there are no directly observable effects.

__20.  Does your spec define when and how new kinds of errors should be raised?__

No.
The spec defines new situations in which WebAssembly traps can occur,
but these are not new kinds of errors.
The spec also has postMessage throw an error when a shared object would be sent outside a cross-origin isolated context,
but this is the same kind of error that is thrown in the same situation when sending a SharedArrayBuffer.

__21.  Does your feature allow sites to learn about the user's use of assistive technology?__

No.

__22.  What should this questionnaire have asked?__

There are no privacy or security issues beyond what has already been discussed.
