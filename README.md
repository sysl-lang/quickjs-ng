# quickjs

**JavaScript for sysl — a quickjs-ng embedding, with no C shim.**

```
dependencies {
  quickjs { git = "github.com/sysl-lang/quickjs-ng", version = "0.2.0" }
}
```

The module is `sh.sysl.quickjs`: quickjs-ng is what it is bound over, and the module says which engine
a program is embedding, because a JavaScript engine's dialect, garbage collector and job queue are not
interchangeable.

```sysl
val r = realm()?
val v = r.eval("[1, 2, 3].map(x => x * 2).join('-')")?

print(v.text()?)                              // 2-4-6
```

## Installing quickjs-ng

```
brew install quickjs-ng
```

**It ships no `pkg-config` file** — the upstream CMake install writes a package configuration and
nothing else — so the paths are passed rather than probed:

```
sysl test . --include-path quickjs=/opt/homebrew/include --link-path /opt/homebrew/lib
```

That is also why the manifest says `requires { headers { quickjs = … } }` where most bindings in this
org say `pkg_config`. The library is `libqjs`, so the link directive is `@link("qjs")` and not
`quickjs`.

## Nothing is vendored

The org vendors a library so that it reaches a bare-metal target — that is what miniz is carried for.
A JavaScript engine with a bytecode interpreter, a garbage collector and the whole ECMAScript library
is not what goes on a microcontroller, so vendoring would buy nothing and cost a CVE watch on a
fast-moving upstream.

## A `&Value` owns exactly one reference, and the destructor gives it back

Every value quickjs hands over carries a reference count somebody has to return. Here that somebody is
`impl Drop for Value`, so **nothing in a program calls free**, and the rest of the binding is arranged
around keeping that invariant true:

- a C function that **returns** a value hands over a count, and the box adopts it;
- a C function that **takes ownership** — `JS_SetPropertyStr`, the trampoline's return — is given a
  duplicate, because the caller's box still holds its own;
- a value merely being **looked at** — an argument vector, which is `JSValueConst` — is duplicated
  before it is boxed, since the callee owns none of it.

A `Value` keeps its `Realm` alive and a `Realm` keeps its `Runtime` alive, so the three frees are
ordered by construction rather than by a comment asking for it. A value may be read long after the
last name for its realm has gone, and there is a test for exactly that.

**Measured rather than asserted.** Resident set over a loop creating realms, objects, strings, arrays
and native calls: **2.92 MB at 10,000 iterations and 3.01 MB at 100,000** — flat across a tenfold
increase. A leak here would be invisible to every test in the suite, because the answers are correct
the whole time.

## A sysl closure can be a JavaScript function

```sysl
var r = realm()?

val factor = 3

r.define("scale", 1, (call) -> Ok(call.realm.integer(call.arg(0).integer()? * factor)))?

print(r.eval("scale(14)")?.integer()?)        // 42
```

It is a **closure**, not just a named function: `factor` is captured and reaches the call. That works
because `JS_NewCClosure` gives every function object its own `void *`, which carries the boxed sysl
closure — so this binding needs no magic index and does not claim the context's single opaque slot.

**One trampoline serves them all, and it is `@export`ed because it has to be.** A sysl function taking
a `JSValue` by value cannot have its address taken otherwise: an aggregate crosses to C in whichever
registers that machine's convention names rather than the ones a sysl call uses, and the compiler
refuses `&f` saying so in as many words. `@export` is what gives it a C-callable address.

An `Err` returned from a native function becomes a JavaScript exception the script can `catch`.

## No shim, and quickjs-ng is why

The org's other engine-sized bindings need a line of C for the functions a header only declares
`static inline`. quickjs-ng promoted the three that would have forced one here — `JS_FreeValue`,
`JS_DupValue` and `JS_ToCStringLen2` are real symbols in the fork and `static inline` in Bellard's
original. What is left inline is tag arithmetic: `JS_MKVAL` and the `JS_Is*` predicates are a struct
field comparison, which sysl does for itself once it knows the layout.

**281 exported symbols were enumerated with `nm` before a single `extern` was written**, per the org's
rule about headers that declare more than the library ships.

## The layout is the thing to check first

A `JSValue` is 16 bytes on a 64-bit target: an eight-byte union and an eight-byte tag. The binding
declares that as two words and everything else rests on it, so the suite asserts the size, both
offsets and the alignment against the header's own `sizeof` and `offsetof`. A quickjs-ng release that
changed the representation then fails a build rather than returning wrong numbers.

**This package needs a 64-bit target and no capability clause can say so.** `quickjs.h` picks a
NaN-boxed `JSValue` — a bare `uint64_t` — whenever `INTPTR_MAX < INT64_MAX`, and the two-word struct
would then describe the wrong thing. The layout tests are what notice.

## Promises and the job queue

**A promise's continuation is a job, and nothing runs a job but a drain.** quickjs enqueues one every
time a promise settles or an `async` function suspends, and `eval` answers the moment the
*synchronous* part of the script is done — so an embedding that never drains has promises that
resolve and continuations that never happen, which reads as a program that hung rather than as a
missing call.

```sysl
val r = realm()?

print(r.eval("(async () => { const n = await Promise.resolve(6); return n * 7 })()")?
       .settled()?.integer()?)                 // 42
```

`settled` is the whole of what an embedding usually wants: it drains and then reads. Underneath it,
`realm.jobs()` runs the queue to empty and `realm.job()` runs one; `value.state()` answers
`Pending`, `Fulfilled`, `Rejected` or `NotAPromise`, and `value.result()` is what a settled one
settled to.

**The drain is a REALM's and not a runtime's**, because a job that throws leaves its exception on a
*context* and a runtime has none to read it from. One runtime may hold several realms, so a job
belonging to another one is named as that rather than reported by reading this realm's exception slot
and saying whatever happened to be there.

**A drain is bounded**, because a job may enqueue another one — a promise chain that resolves in a
loop does it by accident — and an unbounded drain is then a hang with nothing on screen.

## What is not here yet

ES modules with a resolver, classes with sysl-owned instance data, and
typed arrays. Each is a coherent piece of work of its own; the surface here is the one an embedding
needs before any of them are interesting.

## Testing

```
sysl test . --include-path quickjs=/opt/homebrew/include --link-path /opt/homebrew/lib
```

27 tests. Under AddressSanitizer they are clean, and the instrumentation was checked rather than
assumed — `nm -u` on a built program answers 17 ASan symbols including `__asan_memcpy`, which is what
distinguishes an instrumented binary from one that merely linked the runtime. **It covers the sysl
half only**: quickjs-ng is a prebuilt system library here, so no flag of ours reaches its objects.

## Upstream

quickjs-ng 0.16.2 is what this was written against — <https://github.com/quickjs-ng/quickjs>.
