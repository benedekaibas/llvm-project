# Teach the Clang Static Analyzer to understand lifetime annotations

**[Google Summer of Code 2026](https://summerofcode.withgoogle.com/programs/2026/projects/FVArxbU6) @ [LLVM Compiler Infrastructure](https://llvm.org/)**

Benedek Kaibás, Allegheny College, United States

[Commits on GitHub](https://github.com/llvm/llvm-project/commits?author=benedekaibas) | [Pull
requests](https://github.com/llvm/llvm-project/pulls?q=is%3Apr+author%3Abenedekaibas)

## Problem statement 

Around 70% of the severe security bugs at Microsoft and in Chromium come from memory safety violations [2][3], and in 2022 the NSA recommended moving away from memory unsafe languages such as C and C++ [1], a
recommendation the White House repeated in 2024 [4]. Many of them are lifetime errors, where a pointer or a reference outlives the object it points to. Rewriting the C++ that is already out there is rarely practical,
so these bugs have to be caught in the code as it is. The Clang Static Analyzer reasons about the paths of a program without running it and teaching it to understand lifetime annotations lets it catch dangling pointers
and references the compiler cannot detect.

## Background

Clang has a lifetime analysis behind the `-Wlifetime-safety` flag [7], inspired by Rust's Polonius borrow checker. It works with an Origins and Loans model. An origin is the set of memory locations a pointer may refer to
and a loan is a single act of borrowing from one. Together they tell whether a pointer still refers to something alive. The analysis also reads lifetime annotations such as `[[clang::lifetimebound]]` and
`[[clang::lifetime_capture_by(X)]]` [10]. `-Wlifetime-safety` has to run on every translation unit of every build, so it is intra-procedural. That makes a single missing annotation enough to hide a bug. Once the value
travels through an unannotated function in the chain, the analysis treats it as opaque and stays silent. This project closes that gap by teaching the CSA to read the same annotations and to follow the lifetime through the
function chain by inlining.

`-Wlifetime-safety` is intra-procedural because it has to run on every translation unit of every build. A single missing annotation in the chain is then enough to hide a bug. The CSA has no such limit which means it can
inline the callee and follow the lifetime through the chain.

The following example shows the difference:

```cpp
// Defined in another translation unit.
char *findHeader(char *buffer [[clang::lifetimebound]], const char *name);

char *contentTypeOf(char *buffer) { return findHeader(buffer, "Content-Type"); }

char *parseResponse() {
  return contentTypeOf(buffer);
}
```

Only `findHeader` is annotated so the compiler has nothing to work with:

```console
$ clang --analyze -Xclang -analyzer-checker=core,alpha.cplusplus.UseAfterLifetimeEnd \
        -Xclang -analyzer-output=text header-parse.cpp
```

The CSA inlines `contentTypeOf` and finds the annotated call

```text
warning: Returning value bound to 'buffer[0]' that will go out of scope [alpha.cplusplus.UseAfterLifetimeEnd]
    8 |   return contentTypeOf(buffer);
      |   ^~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: Calling 'contentTypeOf'
    8 |   return contentTypeOf(buffer);
      |          ^~~~~~~~~~~~~~~~~~~~~
note: Value's lifetime bound to the lifetime of 'buffer[0]' here
    4 | char *contentTypeOf(char *buffer) { return findHeader(buffer, "Content-Type"); }
      |                                                       ^~~~~~
note: Returning from 'contentTypeOf'
    8 |   return contentTypeOf(buffer);
      |          ^~~~~~~~~~~~~~~~~~~~~
note: Lifetime of 'buffer[0]' ended here
    8 |   return contentTypeOf(buffer);
      |   ^~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

## The goal of the project

This project closes that gap by giving the CSA the same contracts the compiler reads. When the analyzer can inline a callee it follows the lifetime through the chain itself and does not need the annotation at all.
When it cannot inline the callee then the annotation is what it falls back on. That combination lets it report two things the compiler stays silent about: a value that is returned while it still borrows from the
frame it leaves and a pointer that is dereferenced or passed on after its object has gone out of scope. The analyzer works path by path which means every report carries the execution path that leads to the bug. The path
shows where the value was borrowed and where its storage died. Both checkers are in Clang today and remain in the alpha package until they are stable. The rest of this report shows how they were built and what they catch
that the compiler misses. The future work section describes what is left to do.

## Implementation

### From an attribute to a report

Nothing can be reported before the analyzer records where a value borrows from, so that recording came first. Clang attaches `[[clang::lifetimebound]]` to the
annotated `ParmVarDecl`, so the contract is in the AST before the analyzer starts. It becomes usable in `checkPostCall`: the call has been evaluated, so the
return value and every argument have an `SVal` and the checker can pair the returned value with the region of the argument it borrows from.

The harder half was where to keep that pair. Every `ProgramState` carries a generic data map next to the store and the constraints, a key-value area where a
checker registers a container of its own. What lives there is path specific. It is duplicated when a branch splits the exploration and dropped when the analyzer
backtracks, so a borrow that only exists when a condition holds stays on that branch. Path sensitivity is what separates this from the intra-procedural
analysis, and it costs nothing once the data lives in the right place.

```text
              ProgramState of the node before the branch
              +-------------------------------------+
              | environment, store                  |
              | generic data map                    |
              |   LifetimeBoundMap   (empty)        |
              +-------------------------------------+
                              |
                   if (cond) p = borrow(&y);
                     /                    \
              cond true                cond false
   +------------------------------+   +------------------------------+
   |  LifetimeBoundMap            |   |  LifetimeBoundMap            |
   |    returned value -> { y }   |   |    (empty)                   |
   +------------------------------+   +------------------------------+
     the successor nodes on this        the nodes on this path never
     path carry the borrow              see the entry
```

```cpp
int *test_func(int *p [[clang::lifetimebound]]);

int *variable_return() {
  int y = 5;
  int *p = test_func(&y);   // (1)
  return p;                 // (2)
}
```

```text
  (1) LifetimeModeling::checkPostCall fires on the call to 'test_func'

      The callee's parameter carries the attribute, the argument has a region
      and the call has a return value, so the pair is written into the state:

      ProgramState of this node
      +--------------------------------------------------------------+
      | store         p -> &SymRegion{conj_$0<int *>}                 |
      | constraints   ...                                             |
      | generic data map                                              |
      |   LifetimeBoundMap                                            |
      |     +------------------------------+---------------------+   |
      |     | borrowing value              | lifetime sources    |   |
      |     +------------------------------+---------------------+   |
      |     | &SymRegion{conj_$0<int *>}   | { y }               |   |
      |     +------------------------------+---------------------+   |
      +--------------------------------------------------------------+

  (2) UseAfterLifetimeEnd::checkEndFunction fires on the return

      The checker asks the map what the returned value borrows from and
      whether those regions belong to the frame that is about to die.

      'y' lives in this frame  ->  warning: Returning value bound to 'y'
                                   that will go out of scope
```

```console
$ clang -cc1 -analyze -analyzer-checker=core,alpha.cplusplus.UseAfterLifetimeEnd \
        -analyzer-config cfg-lifetime=true -analyzer-output=text example.cpp
```

```text
warning: Returning value bound to 'y' that will go out of scope [alpha.cplusplus.UseAfterLifetimeEnd]
    4 |   int y = 5;
      |       ~
    5 |   int *p = test_func(&y);
    6 |   return p;
      |   ^~~~~~~~
note: 'y' initialized here
    4 |   int y = 5;
      |   ^~~~~
note: Value's lifetime bound to the lifetime of 'y' here
    4 |   int y = 5;
      |       ~
    5 |   int *p = test_func(&y);
      |                      ^~
note: Lifetime of 'y' ended here
    4 |   int y = 5;
      |       ~
    5 |   int *p = test_func(&y);
    6 |   return p;
      |   ^~~~~~~~
```

Because a checker that both owns state and reports bugs cannot be reused, I decided to split the logic into two checkers. This allows other checkers to reuse the state and makes the reporting logic independent.

- [#200143](https://github.com/llvm/llvm-project/pull/200143) and [#200145](https://github.com/llvm/llvm-project/pull/200145) — the prototype, closed after the review
- [#205521](https://github.com/llvm/llvm-project/pull/205521) — the `UseAfterLifetimeEnd` checker

### One writer for the state

With the borrow recorded, the next question was who is allowed to write it. `LifetimeModeling` owns everything that touches the program state and exposes nothing but queries. The reporting checkers
(`UseAfterLifetimeEnd` and `DanglingPtrDeref`) read it, decide whether what they see is a bug and build the report that explains the path to it.

```text
   [[clang::lifetimebound]]                    end of a scope
              \                                      /
               v                                    v
        +--------------------------------------------------+
        |                LifetimeModeling                  |
        |        the only checker that writes state        |
        |                                                  |
        |   what borrows from what  |  what has died       |
        +--------------------------------------------------+
                    |                          |
                    | queries                  | queries
                    v                          v
        +----------------------+    +--------------------------+
        | UseAfterLifetimeEnd  |    |     DanglingPtrDeref     |
        | a returned value     |    | a pointer used after     |
        | outlives its frame   |    | its object is gone       |
        +----------------------+    +--------------------------+
```

- [#205951](https://github.com/llvm/llvm-project/pull/205951) — moves every state operation into `LifetimeModeling`

### Catching a use after the scope ends

The annotated case was now covered. The second kind of lifetime error needs no annotation at all: a pointer used after the object it points to has gone out of
scope. While the modelling was taking shape in June, Arseniy Zaostrovnykh added `CFGLifetimeEnds` handling to the analyzer, which turns every end of a scope
into a `checkLifetimeEnd` callback and lets the CSA reason about lifetimes directly. I picked it up as soon as it landed and built `DanglingPtrDeref` on top of
it. `LifetimeModeling` listens to that callback and adds the region to `DeallocatedSourceSet`. `checkPreStmt(DeclStmt)` clears a variable's region from that set
whenever its declaration is executed, so a stale entry cannot make fresh storage look dead.

`DanglingPtrDeref` subscribes to `check::Location` and asks on every load and store through a pointer whether the pointee is gone.

```cpp
void use_after_scope() {
  int *ptr = nullptr;
  {
    int num = 5;
    ptr = &num;
  }
  *ptr = 6;
}
```

```console
$ clang -cc1 -analyze -analyzer-checker=core,alpha.cplusplus.DanglingPtrDeref \
        -analyzer-config cfg-lifetime=true -analyzer-output=text example.cpp
```

```text
warning: Use of 'num' after its lifetime ended [alpha.cplusplus.DanglingPtrDeref]
    7 |   *ptr = 6;
      |   ~~~~~^~~
note: 'num' initialized to 5
    4 |     int num = 5;
      |     ^~~~~~~
note: Value assigned to 'ptr'
    5 |     ptr = &num;
      |     ^~~~~~~~~~
note: 'num' is destroyed here
    6 |   }
      |   ^
note: Use of 'num' after its lifetime ended
    7 |   *ptr = 6;
      |   ~~~~~^~~
```

- [#206460](https://github.com/llvm/llvm-project/pull/206460) — the first version, closed
- [#209278](https://github.com/llvm/llvm-project/pull/209278) — `DanglingPtrDeref` in its reviewed form

### A pointer that is only passed on

While evaluating the first version of the checker I found a case it did not cover: a dangling pointer handed to another function. `check::Location` does not fire there because the load at a call site is the load of the
pointer variable itself and not an access through the pointer. `DanglingPtrDeref` now also implements `checkPostCall` and inspects every argument, but only when the analyzer did not step into the callee's body. If it did,
`checkLocation` already covers every dereference in there and reporting at the call site as well would produce the same bug twice.

```cpp
void escape(int *ptr);

void passing_dangling_ptr_to_opaque_func() {
  int *ptr = nullptr;
  {
    int num = 5;
    ptr = &num;
  }
  escape(ptr);
}
```

```text
   escape(ptr);
        |
        +-- check::Location fires once, for the load of 'ptr'
        |      location it is given:   the region of 'ptr'   (alive)
        |      'num' is only the value that comes out of that load, never the location
        |      so the callback has nothing dead to look at
        |
        +-- checkPostCall sees the call after it was evaluated
               argument SVal:  &num   ->  its lifetime has ended  ->  report
```

- [#211045](https://github.com/llvm/llvm-project/pull/211045) — inspects call arguments in `checkPostCall`

### A pointer into part of a dead object

So far we only talked about primitive types, but C++ also has aggregates and inheritance as well. A pointer can point at a field, an array element or a base class subobject, while what the modeling checker recorded as
deallocated was the region of the whole object. `isDeallocated` now looks up the base region of the region it is asked about, so a pointer into any part of a dead object is recognized as dangling.

```cpp
struct MyBuffer {
  char buffer[8];
};

char member_subregion_dangling_deref() {
  const char *p = nullptr;
  {
    MyBuffer tmp_buffer = {};
    p = tmp_buffer.buffer;
  }
  return *p;
}
```

```text
   p = tmp_buffer.buffer;   ->   p points at   ElementRegion  tmp_buffer.buffer[0]
                                                     |
                                  getBaseRegion()    |   FieldRegion  tmp_buffer.buffer
                                                     v
                                                VarRegion  tmp_buffer

   checkLifetimeEnd records the region of the variable that died:  { tmp_buffer }
   a lookup of the element region misses it, so isDeallocated asks for the base
   region first and finds the entry
```

- [#211552](https://github.com/llvm/llvm-project/pull/211552) — matches dangling subobjects by their base region

### Making it quiet enough for real code

With both checkers passing their own tests, the next step was a large real code base. The LLVM monorepo is a good test bed: it is large, actively maintained and heavily reviewed, so it exercises code shapes no hand written
test suite covers. Every report has to be read on its own. The ones I went through fell into three classes, each a place where the model was too coarse, and closing them made both checkers more precise.

**A `lifetimebound` call during destruction.** A destructor is the one place where handing out a borrow of the dying object is normal: the caller is the
destructor itself and the borrow does not escape it. The fix walks the frames on the current stack and suppresses the report if any of them is a destructor.

```text
   +-------------------------------------------------+
   |  Widget::~Widget()          <- destructor frame  |
   |     Widget::getName()       [[lifetimebound]]    |
   +-------------------------------------------------+
      any destructor frame on the stack  ->  no report
```

**A source owned by another frame.** This was the largest of the three classes. A returned value can only dangle if the frame that owns its lifetime source is the frame being left. When the source belongs to a caller that
stays alive, or to a frame the analyzer never entered, the storage outlives the value and there is nothing to report. 

```text
   caller()                          owns 'buf', stays alive
      |  &buf
      v
   helper()  returns a value bound to 'buf'
      |
      v  helper's frame dies here, caller's does not
   no report: the owning frame is still on the stack
```

**The same variable reported again and again.** `checkLocation` fires on every access through a pointer, so a dead variable used five times produced five
warnings that all describe the same bug. `markAsReported` keeps the first and drops the rest.

```text
   before                        after
   *p = 1;   warning             *p = 1;   warning
   *p = 2;   warning             *p = 2;   -
   use(p);   warning             use(p);   -
```

- [#210801](https://github.com/llvm/llvm-project/pull/210801) — suppresses the report inside a destructor
- [#213779](https://github.com/llvm/llvm-project/pull/213779) — discards sources owned by another frame
- [#215409](https://github.com/llvm/llvm-project/pull/215409) — reports each region once

## Patches

Each step in the Implementation section names the patch that carried it. The patches below did the work around those steps: the diagnostics, the documentation, the cleanups and what is still in review. Within each group
the patches follow the order I opened them over the summer.

**Diagnostics**

- [#207052](https://github.com/llvm/llvm-project/pull/207052): implemented a `BugReporterVisitor` for `UseAfterLifetimeEnd` that adds the note showing where the value's lifetime was bound to its source.
- [#211818](https://github.com/llvm/llvm-project/pull/211818): added `trackExpressionValue` to `DanglingPtrDeref` so the report shows where the dangling value originated.
- [#212158](https://github.com/llvm/llvm-project/pull/212158): replaced the debug only `getString()` with `getDescriptiveName()` in the diagnostics, added the shared `getRegionName()` helper and switched the path notes to `trackStoredValue()`.
- [#215651](https://github.com/llvm/llvm-project/pull/215651): underlined only the parameter that the return value is actually bound to when a function has several parameters.
- [#215905](https://github.com/llvm/llvm-project/pull/215905): corrected the highlighted range so it matches the variable the note refers to.

**Documentation**

- [#216688](https://github.com/llvm/llvm-project/pull/216688): documentation for `DanglingPtrDeref`.
- [#216739](https://github.com/llvm/llvm-project/pull/216739): moves both checkers from `alpha.cplusplus` to `alpha.core`.
- [#217122](https://github.com/llvm/llvm-project/pull/217122): documentation for `UseAfterLifetimeEnd`.

**Fixes and cleanups**

- [#207472](https://github.com/llvm/llvm-project/pull/207472): an earlier take on the destructor suppression, superseded by [#210801](https://github.com/llvm/llvm-project/pull/210801).
- [#209862](https://github.com/llvm/llvm-project/pull/209862): NFC, corrected the `DanglingPtrDeref` checker's filename.
- [#211582](https://github.com/llvm/llvm-project/pull/211582): NFC, rewrote the destructor frame walk with `llvm::any_of`.
- [#212254](https://github.com/llvm/llvm-project/pull/212254): NFC, added test cases to the lifetime test suite.

## Results

Both checkers are in Clang today and they report bugs that nothing else in the toolchain reports. The analyzer had no model of `[[clang::lifetimebound]]` at
all. `-Wlifetime-safety` does read it, but it has to run on every translation unit of every build, so it stops at the first call it cannot see into. Each case
below is ordinary C++ that the compiler and the analyzer's existing checkers pass over in silence.

### The analyzer had nothing to say about a dead stack object

```cpp
void use_after_scope() {
  int *ptr = nullptr;
  {
    int num = 5;
    ptr = &num;
  }
  *ptr = 6;
}
```

None of the analyzer's existing checkers for this kind of error cover it, `core.StackAddressEscape` and `cplusplus.InnerPointer` included: with
`-analyzer-checker=core,cplusplus` the analyzer produces no diagnostic at all. `DanglingPtrDeref` reports it.

```console
$ clang -cc1 -analyze -analyzer-checker=core,alpha.cplusplus.DanglingPtrDeref \
        -analyzer-config cfg-lifetime=true -analyzer-output=text use-after-scope.cpp
```

```text
warning: Use of 'num' after its lifetime ended [alpha.cplusplus.DanglingPtrDeref]
    7 |   *ptr = 6;
      |   ~~~~~^~~
note: 'num' initialized to 5
    4 |     int num = 5;
      |     ^~~~~~~
note: Value assigned to 'ptr'
    5 |     ptr = &num;
      |     ^~~~~~~~~~
note: 'num' is destroyed here
    6 |   }
      |   ^
note: Use of 'num' after its lifetime ended
    7 |   *ptr = 6;
      |   ~~~~~^~~
```

### One unannotated call and the compiler goes quiet

The compiler handles the direct case. With the local passed straight to the annotated parameter, `-Wlifetime-safety` says so:

```cpp
const char *findOption(const char *config [[clang::lifetimebound]], const char *key);

const char *readTimeoutDirect() {
  char config[64] = "timeout=30";
  return findOption(config, "timeout");
}
```

```text
warning: address of stack memory associated with local variable 'config' returned [-Wreturn-stack-address]
    5 |   return findOption(config, "timeout");
      |                     ^~~~~~
warning: stack memory associated with local variable 'config' is returned [-Wlifetime-safety-return-stack-addr]
    5 |   return findOption(config, "timeout");
      |                     ^~~~~~
note: returned here
    5 |   return findOption(config, "timeout");
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

Put one unannotated function between the local and the annotated call and the warning disappears, because the analysis cannot look inside `optionValue` and
`optionValue` carries no contract of its own. A chain only has to lose one link for the diagnostic to go with it.

```cpp
// Defined in another translation unit.
const char *findOption(const char *config [[clang::lifetimebound]], const char *key);

const char *optionValue(const char *config, const char *key) {
  return findOption(config, key);
}

const char *readTimeout() {
  char config[64] = "timeout=30";
  return optionValue(config, "timeout");
}
```

The CSA inlines `optionValue`, reaches the annotated call inside it and falls back on the contract. The report names `config[0]` because that is the region the
array decays to at the call.

```console
$ clang -cc1 -analyze -analyzer-checker=core,alpha.cplusplus.UseAfterLifetimeEnd \
        -analyzer-config cfg-lifetime=true -analyzer-output=text config-lookup.cpp
```

```text
warning: Returning value bound to 'config[0]' that will go out of scope [alpha.cplusplus.UseAfterLifetimeEnd]
   10 |   return optionValue(config, "timeout");
      |   ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: Calling 'optionValue'
   10 |   return optionValue(config, "timeout");
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: Value's lifetime bound to the lifetime of 'config[0]' here
    5 |   return findOption(config, key);
      |                     ^~~~~~
note: Returning from 'optionValue'
   10 |   return optionValue(config, "timeout");
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: Lifetime of 'config[0]' ended here
   10 |   return optionValue(config, "timeout");
      |   ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

### A borrow made in one function and used in another

```cpp
void submit(const int *sample);

const int *keep(const int *value) { return value; }

void collectSample(int reading) {
  const int *sample = nullptr;
  {
    int corrected = reading + 1;
    sample = keep(&corrected);
  }
  submit(sample);
}
```

Neither function is annotated and `sample` is never dereferenced in `collectSample`. `-Wlifetime-safety` sees `keep` as an opaque call and has no origin to
follow, so it stays silent. The CSA inlines `keep`, follows the pointer back to `corrected` and reports the call that passes it on.

```console
$ clang -cc1 -analyze -analyzer-checker=core,alpha.cplusplus.DanglingPtrDeref \
        -analyzer-config cfg-lifetime=true -analyzer-output=text collect-sample.cpp
```

```text
warning: Use of 'corrected' after its lifetime ended [alpha.cplusplus.DanglingPtrDeref]
   11 |   submit(sample);
      |   ^~~~~~~~~~~~~~
note: 'corrected' initialized here
    8 |     int corrected = reading + 1;
      |     ^~~~~~~~~~~~~
note: Passing value via 1st parameter 'value'
    9 |     sample = keep(&corrected);
      |                   ^~~~~~~~~~
note: Value assigned to 'sample'
    9 |     sample = keep(&corrected);
      |     ^~~~~~~~~~~~~~~~~~~~~~~~~
note: 'corrected' is destroyed here
   10 |   }
      |   ^
note: Use of 'corrected' after its lifetime ended
   11 |   submit(sample);
      |   ^~~~~~~~~~~~~~
```

Together the two checkers move lifetime checking past the point where the compiler has to stop. `-Wlifetime-safety` remains the right tool for the common case,
because it is cheap enough to run on every build. What it cannot do is follow a value through a call it cannot see into, and a chain only has to lose one link
for the warning to disappear. Every report the checkers produce carries the execution path from the borrow to the death of the storage, including the branch
that had to be taken, so the reader can judge it without rerunning the analysis. Both checkers ship in Clang today, both have been run over the LLVM monorepo,
and `LifetimeModeling` is a base that the next lifetime checker can consume instead of rebuilding.


## Future work

**Support for `LazyCompoundVal`.** `std::string_view` and `std::span` are among the types the annotations are written on. They are class types returned by value, which the analyzer represents as a `LazyCompoundVal`
and none of the maps in the modeling checker cover that today, so the lifetime origin is lost:

```cpp
struct View { int *p; };
View makeView(int &x [[clang::lifetimebound]]);

void caller_view() {
  int v = 42;
  View w = makeView(v);
  // FIXME: Currently none of the maps cover LazyCompoundVal.
}
```

Covering it brings the checkers to the code the annotations were introduced for.

**Interoperability with the `MallocChecker`.** Both checkers reason about stack regions only, so a lifetime source that lives on the heap is out of their reach. The plan is to extend them to heap sources through the
interface the `MallocChecker` already exposes for this in `AllocationState.h`, which is how `cplusplus.InnerPointer` works with it today, instead of modeling the heap in the lifetime checkers.

**The `[[clang::lifetime_capture_by(X)]]` annotation.** The original proposal covered both annotations. During the summer I have prioritized `[[clang::lifetimebound]]`, because it is the annotation people actually use. In
the LLVM monorepo alone libc++ applies `_LIBCPP_LIFETIMEBOUND` around 80 times across 22 headers, while `lifetime_capture_by` does not appear in the library at all, and outside LLVM the same asymmetry holds: Abseil ships
`ABSL_ATTRIBUTE_LIFETIME_BOUND` [9] and uses it throughout its string and container types, while `lifetime_capture_by` is a much newer addition [10]. Supporting the capture annotation is the work I intend to do after the
summer.

**Maintenance.** I intend to maintain these checkers. That means fixing the false positives that will show up once people enable them on their own code bases, moving them out of `alpha` once they are stable enough, and
continuing to contribute to the Clang Static Analyzer in general.

## Special thanks

I would like to say thank you to my mentors, in the order they appear on the LLVM GSoC page: Gábor Horváth, Balázs Benics and Dániel Domján, for mentoring this project, for the reviews and design discussions that shaped
both checkers and for answering my questions throughout the summer. The time they put into the project and into teaching me went well beyond what I expected. My questions were answered quickly and my patches were reviewed
even at weekends. 

I would also like to say thank you to Donát Nagy for his code reviews on some of these patches and to Arseniy Zaostrovnykh for the `CFGLifetimeEnds` support in the CFG.

## References

[1] National Security Agency, "Software Memory Safety", Cybersecurity Information Sheet, November 2022.
https://media.defense.gov/2022/Nov/10/2003112742/-1/-1/0/CSI_SOFTWARE_MEMORY_SAFETY.PDF

[2] M. Miller, "Trends, Challenges, and Strategic Shifts in the Software Vulnerability Mitigation Landscape", BlueHat IL, February 2019.
https://github.com/microsoft/MSRC-Security-Research

[3] The Chromium Projects, "Memory safety". https://www.chromium.org/Home/chromium-security/memory-safety/

[4] Office of the National Cyber Director, "Back to the Building Blocks: A Path Toward Secure and Measurable Software", February 2024.

[5] B. Stroustrup, "A call to action: Think seriously about 'safety'; then do something sensible about it", P2739R0, December 2022.
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2739r0.pdf

[6] H. Sutter, "Lifetime safety: Preventing common dangling", P1179R1, version 1.1, November 2019.
https://github.com/isocpp/CppCoreGuidelines/blob/master/docs/Lifetime.pdf

[7] U. Saxena, D. Hrybenko, Y. Mandelbaum, J. Voung and K. Yasuda, "[RFC] Intra-procedural lifetime analysis in Clang", LLVM Discussion Forums, May 2025.
https://discourse.llvm.org/t/rfc-intra-procedural-lifetime-analysis-in-clang/86291

[8] Clang documentation, "Available Checkers". https://clang.llvm.org/docs/analyzer/checkers.html

[9] Abseil, definition of `ABSL_ATTRIBUTE_LIFETIME_BOUND` in `absl/base/attributes.h`. https://github.com/abseil/abseil-cpp/blob/master/absl/base/attributes.h

[10] G. Horvath and U. Saxena, "[RFC] Introduce `[[clang::lifetime_capture_by(X)]]`", LLVM Discussion Forums, November 2024.
https://discourse.llvm.org/t/rfc-introduce-clang-lifetime-capture-by-x/81371
