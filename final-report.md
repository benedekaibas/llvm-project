# Teach the Clang Static Analyzer to understand lifetime annotations

**[Google Summer of Code 2026](https://summerofcode.withgoogle.com/programs/2026/projects/FVArxbU6) @ [LLVM Compiler Infrastructure](https://llvm.org/)**

Benedek Kaibás, Allegheny College, United States

[Commits on GitHub](https://github.com/llvm/llvm-project/commits?author=benedekaibas) | [Pull
requests](https://github.com/llvm/llvm-project/pulls?q=is%3Apr+author%3Abenedekaibas)

## Motivation

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

## The goal of the project

This project closes that gap by giving the CSA the same contracts the compiler reads. When the analyzer can inline a callee it follows the lifetime through the chain itself and does not need the annotation at all.
When it cannot inline the callee then the annotation is what it falls back on. That combination lets it report two things the compiler stays silent about: a value that is returned while it still borrows from the
frame it leaves and a pointer that is dereferenced or passed on after its object has gone out of scope. The analyzer works path by path which means every report carries the execution path that leads to the bug. The path
shows where the value was borrowed and where its storage died. Both checkers are in Clang today and remain in the alpha package until they are stable. The rest of this report shows how they were built and what they catch
that the compiler misses. The future work section describes what is left to do.

## Implementation

### The first prototype

[#200143](https://github.com/llvm/llvm-project/pull/200143) and its reopened version [#200145](https://github.com/llvm/llvm-project/pull/200145) are where I
checked how the annotation reaches a checker callback and how the recorded relationship appears in the program state and in the ExplodedGraph. The
`checkPostCall` there reads `[[clang::lifetimebound]]` from the callee's parameter declarations and records the relationship in the Generic Data Map. The
discussion on the pull request settled that the modeling and the reporting have to live in separate checkers and the work continued in
[#205521](https://github.com/llvm/llvm-project/pull/205521) and [#205951](https://github.com/llvm/llvm-project/pull/205951).

### The UseAfterLifetimeEnd checker

The reporting checker for `[[clang::lifetimebound]]` landed in [#205521](https://github.com/llvm/llvm-project/pull/205521). When a call returns a value that the
annotation ties to one of its arguments, the checker records which local variable the returned value borrows from. When that value is then returned from the
enclosing function the storage it borrows from is about to be destroyed, so the checker reports a dangling return value.

```cpp
int *test_func(int *p [[clang::lifetimebound]]);

int *variable_return() {
  int y = 5;
  int *p = test_func(&y);
  return p;
}
```

The report names the variable the value is bound to, points at the place where the binding was established and at the place where the lifetime ends:

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

### Splitting the modeling from the reporting

In [#205951](https://github.com/llvm/llvm-project/pull/205951) I moved everything that touches the program state into a separate `LifetimeModeling` checker. It
keeps a `LifetimeBoundMap` that maps an `SVal` to the set of `MemRegion`s it borrows from, a `DeallocatedSourceSet` that holds the regions whose lifetime has
already ended and a `ReportedDeadRegions` set that is used for deduplication. The reporting checkers query it through a small interface in `LifetimeModeling.h`:

```cpp
namespace clang::ento::lifetime_modeling {
std::vector<const MemRegion *>
getDanglingRegionsAfterReturn(SVal Source, ProgramStateRef State,
                              CheckerContext &C);
bool isDeallocated(ProgramStateRef State, const MemRegion *Region);
bool isBoundToLifetimeSource(ProgramStateRef State, SVal Val);
ProgramStateRef markAsReported(ProgramStateRef State, const MemRegion *Region);
} // namespace clang::ento::lifetime_modeling
```

The benefit of this split is that the lifetime state is maintained in one place and any checker can subscribe to it. `DanglingPtrDeref` was written after this
patch and it consumes the same state without adding any state tracking of its own.

### The DanglingPtrDeref checker

`DanglingPtrDeref` reports uses of a pointer after the stack object it points to has gone out of scope. It is not annotation driven. It subscribes to
`check::Location` so on every load or store through a pointer it asks the modeling checker whether the pointee's lifetime has already ended. The check therefore
covers a dereference anywhere in the function, including a dereference inside a return statement. A return statement that only returns the pointer without
dereferencing it is not reported here, that case belongs to the `core.StackAddressEscape` checker.

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

What made this checker possible is [#201123](https://github.com/llvm/llvm-project/pull/201123) by Arseniy Zaostrovnykh, which added handling of the
`CFGLifetimeEnds` element to the CSA and produced a new `checkLifetimeEnd` callback for each occurrence of it. It landed on June 8, right at the beginning of my
coding period. The `LifetimeModeling` checker subscribes to `check::LifetimeEnd` and inserts the region into the `DeallocatedSourceSet` when the callback fires,
and `DanglingPtrDeref` asks `isDeallocated` about that set in `checkLocation`.

The two reporting checkers therefore read two different parts of the same state. `UseAfterLifetimeEnd` reads the `LifetimeBoundMap` to find out which local a
value borrows from, while `DanglingPtrDeref` reads the `DeallocatedSourceSet` to find out whether the pointee is already gone.

The checker first landed as [#206460](https://github.com/llvm/llvm-project/pull/206460) and was merged in its reviewed form as
[#209278](https://github.com/llvm/llvm-project/pull/209278), with [#209862](https://github.com/llvm/llvm-project/pull/209862) as a follow up.

### Dangling arguments and subobjects

[#211045](https://github.com/llvm/llvm-project/pull/211045) implements `checkPostCall` for `DanglingPtrDeref`, which inspects every call argument.
`checkLocation` is called on a load from and a store to a location, and the load that happens at a call site is the load of the pointer variable itself, not an
access through the pointer, so on its own it never sees the dead region:

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
warning: Use of 'num' after its lifetime ended [alpha.cplusplus.DanglingPtrDeref]
    9 |   escape(ptr);
      |   ^~~~~~~~~~~
note: 'num' initialized to 5
    6 |     int num = 5;
      |     ^~~~~~~
note: Value assigned to 'ptr'
    7 |     ptr = &num;
      |     ^~~~~~~~~~
note: 'num' is destroyed here
    8 |   }
      |   ^
note: Use of 'num' after its lifetime ended
    9 |   escape(ptr);
      |   ^~~~~~~~~~~
```

A dangling pointer can also point at a field, at an array element or at a base class subobject. The checker recorded only the whole object's region as
deallocated, so such a pointer was not recognized as dangling. [#211552](https://github.com/llvm/llvm-project/pull/211552) changes `isDeallocated` to look up a
region's base region:

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
warning: Use of 'tmp_buffer.buffer[0]' after its lifetime ended [alpha.cplusplus.DanglingPtrDeref]
   11 |   return *p;
      |          ^~
note: Initializing to 0
    8 |     MyBuffer tmp_buffer = {};
      |     ^~~~~~~~~~~~~~~~~~~
note: 'tmp_buffer.buffer[0]' is destroyed here
   10 |   }
      |   ^
note: Use of 'tmp_buffer.buffer[0]' after its lifetime ended
   11 |   return *p;
      |          ^~
```

### False positives

I ran both checkers on the LLVM project to get feedback on the rate of false positives they emit on a large, high quality, real world code base. The reports I
got back are what the following patches fix.

The first class came from destructors. When a `lifetimebound` method is called during the destruction of an object the storage it borrows from is not dangling.
[#210801](https://github.com/llvm/llvm-project/pull/210801) suppresses the report when any frame on the current stack belongs to a destructor, and
[#211582](https://github.com/llvm/llvm-project/pull/211582) rewrote that walk with `llvm::any_of`.

The second class produced most of the noise on LLVM. If the stack frame that owns a lifetime source is no longer live on the current stack then that source is
not something the returned value can outlive, so it must not be treated as a dangling stack source. [#213779](https://github.com/llvm/llvm-project/pull/213779)
discards those frames.

[#215409](https://github.com/llvm/llvm-project/pull/215409) implements `markAsReported`, which marks a region as reported the first time it is seen, so each
variable produces one warning instead of one per dereference.

### Improving the quality of the reports

A report that only points at the place where the program misbehaves does not make it clear to the user where the bad value came from.
[#207052](https://github.com/llvm/llvm-project/pull/207052) implements a `BugReporterVisitor` for `UseAfterLifetimeEnd` that adds the note about where the
value's lifetime was bound to its source, and [#211818](https://github.com/llvm/llvm-project/pull/211818) adds `trackExpressionValue` to `DanglingPtrDeref` so
that the report also shows where the dangling value originated from.

[#212158](https://github.com/llvm/llvm-project/pull/212158) replaces `getString()`, which is a debug only stringification, with `getDescriptiveName()` in the
diagnostics, adds the `getRegionName()` helper that both reporting checkers use, and switches the path notes to `trackStoredValue()`.

[#215651](https://github.com/llvm/llvm-project/pull/215651) and [#215905](https://github.com/llvm/llvm-project/pull/215905) fixed the source ranges, so that only
the annotated parameter is underlined when a function has several parameters, and the highlighted range matches the variable the note refers to.

### Documentation

Bringing the checkers out of `alpha` is one of the goals of the project and that requires documentation.
[#216688](https://github.com/llvm/llvm-project/pull/216688) documents `DanglingPtrDeref` and [#217122](https://github.com/llvm/llvm-project/pull/217122)
documents `UseAfterLifetimeEnd` in the checker documentation [8].

### Work outside the project

Two patches from the same period are not part of the project itself. [#212883](https://github.com/llvm/llvm-project/pull/212883) is an NFC patch that matches the
parameter order of `getEndPath` and `finalizeVisitor` with `VisitNode`, and [#210474](https://github.com/llvm/llvm-project/pull/210474) adds
`[[clang::lifetimebound]]` annotations to `Twine.h`.

## Results

Neither the compiler nor the CSA with only the `core` and `cplusplus` packages enabled reports any of the following three cases. Each Compiler Explorer link runs
clang with `-Wall -Wextra -Wdangling -Wlifetime-safety`.

### The annotated function is not available for inlining

```cpp
// Defined in another translation unit.
int *find(int *table [[clang::lifetimebound]], int key);

int *lookup(int *table, int key) { return find(table, key); }

int *stale_entry(int key) {
  int table[4] = {1, 2, 3, 4};
  return lookup(table, key);
}
```

Compiler Explorer: https://godbolt.org/z/546P59fY5

`find` is only declared, so its body cannot be inlined, and `lookup` carries no annotation. The CSA inlines `lookup` and falls back to the annotation for `find`,
and the report names `table[0]` rather than `table`.

```text
warning: Returning value bound to 'table[0]' that will go out of scope [alpha.cplusplus.UseAfterLifetimeEnd]
    8 |   return lookup(table, key);
      |   ^~~~~~~~~~~~~~~~~~~~~~~~~
note: Calling 'lookup'
    8 |   return lookup(table, key);
      |          ^~~~~~~~~~~~~~~~~~
note: Value's lifetime bound to the lifetime of 'table[0]' here
    4 | int *lookup(int *table, int key) { return find(table, key); }
      |                                                ^~~~~
note: Returning from 'lookup'
    8 |   return lookup(table, key);
      |          ^~~~~~~~~~~~~~~~~~
note: Lifetime of 'table[0]' ended here
    8 |   return lookup(table, key);
      |   ^~~~~~~~~~~~~~~~~~~~~~~~~
```

### The pointer only dangles on one path

```cpp
int *select(int *a, int *b, bool use_a) { return use_a ? a : b; }

void render(int width) {
  int fallback = 80;
  int *chosen = nullptr;
  {
    int computed = width * 2;
    chosen = select(&computed, &fallback, width > 0);
  }
  *chosen += 1;
}
```

Compiler Explorer: https://godbolt.org/z/16YYYsKEP

`chosen` is dangling only when `width > 0` holds, because on the other path it points to `fallback`, which outlives the block. The CSA explores the two paths
separately, so it reports the dangling one and shows the condition that leads there.

```text
warning: Use of 'computed' after its lifetime ended [alpha.cplusplus.DanglingPtrDeref]
   10 |   *chosen += 1;
      |   ~~~~~~~~^~~~
note: 'computed' initialized here
    7 |     int computed = width * 2;
      |     ^~~~~~~~~~~~
note: Assuming 'width' is > 0
    8 |     chosen = select(&computed, &fallback, width > 0);
      |                                           ^~~~~~~~~
note: Passing value via 1st parameter 'a'
    8 |     chosen = select(&computed, &fallback, width > 0);
      |                     ^~~~~~~~~
note: Calling 'select'
    8 |     chosen = select(&computed, &fallback, width > 0);
      |              ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: 'use_a' is true
    1 | int *select(int *a, int *b, bool use_a) { return use_a ? a : b; }
      |                                                  ^~~~~
note: '?' condition is true
note: Returning pointer
    1 | int *select(int *a, int *b, bool use_a) { return use_a ? a : b; }
      |                                           ^~~~~~~~~~~~~~~~~~~~
note: Returning from 'select'
    8 |     chosen = select(&computed, &fallback, width > 0);
      |              ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: Value assigned to 'chosen'
    8 |     chosen = select(&computed, &fallback, width > 0);
      |     ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: 'computed' is destroyed here
    9 |   }
      |   ^
note: Use of 'computed' after its lifetime ended
   10 |   *chosen += 1;
      |   ~~~~~~~~^~~~
```

### The borrow is created inside another function

```cpp
void publish(int *value);

int *stash(int *v) { return v; }

void collect(int n) {
  int *saved = nullptr;
  {
    int total = n + 1;
    saved = stash(&total);
  }
  publish(saved);
}
```

Compiler Explorer: https://godbolt.org/z/M7rKPKrra

The borrow is created inside `stash`, which carries no annotation, and the dangling pointer is never dereferenced in `collect`, it is only passed to `publish`.
The CSA inlines `stash` and follows the pointer back to `total`.

```text
warning: Use of 'total' after its lifetime ended [alpha.cplusplus.DanglingPtrDeref]
   11 |   publish(saved);
      |   ^~~~~~~~~~~~~~
note: 'total' initialized here
    8 |     int total = n + 1;
      |     ^~~~~~~~~
note: Passing value via 1st parameter 'v'
    9 |     saved = stash(&total);
      |                   ^~~~~~
note: Value assigned to 'saved'
    9 |     saved = stash(&total);
      |     ^~~~~~~~~~~~~~~~~~~~~
note: 'total' is destroyed here
   10 |   }
      |   ^
note: Use of 'total' after its lifetime ended
   11 |   publish(saved);
      |   ^~~~~~~~~~~~~~
```

## Future work

**Support for `LazyCompoundVal`.** `std::string_view` and `std::span` are among the types the annotations are written on. They are class types returned by value,
which the analyzer represents as a `LazyCompoundVal`, and none of the maps in the modeling checker cover that today, so the lifetime origin is lost:

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

**Interoperability with the `MallocChecker`.** Both checkers reason about stack regions only, so a lifetime source that lives on the heap is out of their reach.
The plan is to extend them to heap sources through the interface the `MallocChecker` already exposes for this in `AllocationState.h`, which is how
`cplusplus.InnerPointer` works with it today, instead of modeling the heap in the lifetime checkers.

**The `[[clang::lifetime_capture_by(X)]]` annotation.** The original proposal covered both annotations. During the summer I have prioritized
`[[clang::lifetimebound]]`, because it is the annotation people actually use. In the LLVM monorepo alone libc++ applies `_LIBCPP_LIFETIMEBOUND` around 80 times
across 22 headers, while `lifetime_capture_by` does not appear in the library at all, and outside LLVM the same asymmetry holds: Abseil ships
`ABSL_ATTRIBUTE_LIFETIME_BOUND` [9] and uses it throughout its string and container types, while `lifetime_capture_by` is a much newer addition [10]. Supporting
the capture annotation is the work I intend to do after the summer.

**Maintenance.** I intend to maintain these checkers. That means fixing the false positives that will show up once people enable them on their own code bases,
moving them out of `alpha` once they are stable enough, and continuing to contribute to the Clang Static Analyzer in general.

## Special thanks

TODO: ADDRESS THIS AS WELL!!!!!

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
