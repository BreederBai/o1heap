# O(1) heap

[![Main Workflow](https://github.com/pavel-kirienko/o1heap/actions/workflows/main.yml/badge.svg)](https://github.com/pavel-kirienko/o1heap/actions/workflows/main.yml)
[![Reliability Rating](https://sonarcloud.io/api/project_badges/measure?project=pavel-kirienko_o1heap&metric=reliability_rating)](https://sonarcloud.io/dashboard?id=pavel-kirienko_o1heap)
[![Coverage](https://sonarcloud.io/api/project_badges/measure?project=pavel-kirienko_o1heap&metric=coverage)](https://sonarcloud.io/dashboard?id=pavel-kirienko_o1heap)

O1heap is a highly deterministic constant-complexity memory allocator designed for
hard real-time high-integrity embedded systems.
The name stands for *O(1) heap*.

The allocator offers
a constant worst-case execution time (WCET) and
a well-characterized worst-case memory fragmentation (consumption) (WCMC).
The allocator allows the designer to statically prove its temporal and spatial properties for a given application,
which makes it suitable for use in high-integrity embedded systems.

The codebase is implemented in C99/C11 following MISRA C:2012, with several intended deviations which are unavoidable
due to the fact that a memory allocator has to rely on inherently unsafe operations to fulfill its purpose.
The codebase is extremely compact (<500 LoC) and is therefore trivial to validate.

The allocator is designed to be portable across all conventional architectures, from 8-bit to 64-bit systems.
Multi-threaded environments are supported with the help of external synchronization hooks provided by the application.

## Design

### Objectives

The core objective of this library is to provide a dynamic memory allocator that meets the following requirements:

- Memory allocation and deallocation routines are constant-time.

- For a given peak memory requirement *M*, the worst-case memory consumption *H* is easily and robustly predictable
  (i.e., the worst-case heap fragmentation is well-characterized).
  The application designer shall be able to easily predict the amount of memory that needs to be provided to the
  memory allocator to ensure that out-of-memory (OOM) failures provably cannot occur at runtime
  under any circumstances.

- The implementation shall be simple and conform to relevant high-integrity coding guidelines.

### Theory

This implementation derives from or is based on the ideas presented in:

- "Timing-Predictable Memory Allocation In Hard Real-Time Systems" -- J. Herter, 2014.
- "An algorithm with constant execution time for dynamic storage allocation" -- T. Ogasawara, 1995.
- "Worst case fragmentation of first fit and best fit storage allocation strategies" -- J. M. Robson, 1975.

This implementation does not make any non-trivial assumptions about the behavioral properties of the application.
Per Herter [2014], it can be described as a *predictably bad allocator*:

> An allocator that provably performs close to its worst-case memory behavior which, in turn,
> is better than the worst-case behavior of the allocators discussed [in Herter 2014],
> but much worse than the memory consumption of these for normal programs without (mostly theoretical)
> bad (de-)allocation sequences.

While it has been shown to be possible to construct a constant-complexity allocator which demonstrates better
worst-case and average-case memory requirements by making assumptions about the memory (de-)allocation patterns
and/or by relying on more sophisticated algorithms, this implementation chooses a conservative approach
where no assumptions are made about the application and the codebase is kept simple to facilitate its integration
into verified and validated high-integrity software.

The library implements a modified Half-Fit algorithm -- a constant-complexity strategy originally proposed by Ogasawara.
In this implementation, memory is allocated in fragments whose size is rounded up to the next integer power of two.
The worst-case memory consumption (WCMC) *H* of this allocation strategy has been shown to be:

H(M,n) = 2 M (1 + ⌈log<sub>2</sub> n⌉)

Where *M* is the peak total memory requirement of the application
(i.e., sum of sizes of all allocated fragments when the heap memory utilization is at its peak)
and *n* is the maximum contiguous fragment that can be requested by the application.

Note that the provided equation is a generalized case that does not explicitly take into account
a possible per-fragment metadata allocation overhead.

The state of *catastrophic fragmentation* is a state where the allocator is unable to serve
a memory allocation request even if there is enough free memory due to its suboptimal arrangement.
By definition, if the amount of memory available to the allocator is not less than *H*, then the state of
catastrophic fragmentation cannot occur.

Memory allocators used in general-purpose (non-real-time) applications often leverage a different class of algorithms
which may feature poorer worst-case performance metrics but perform (much) better on average.
For a hard real-time system, the average case performance is generally less relevant,
so it can be excluded from analysis.

The above-defined theoretical worst-case upper bound H may be prohibitively high for some
memory-constrained applications.
It has been shown [Robson 1975] that under a typical workload,
for a sufficiently high amount of memory available to the allocator which is less than *H*,
the probability of a (de-)allocation sequence that results in catastrophic fragmentation is low.
When combined with an acceptable failure probability and a set of adequate assumptions about the behaviors of
the application, this property may allow the designer to drastically reduce the amount of memory dedicated to
the heap while ensuring a sufficient degree of predictability and reliability.
The methods of such optimization are outside the scope of this document;
interested readers are advised to consult with the referred publications.

Following some of the ideas expressed in the discussion about memory caching in real-time systems in [Herter 2014],
this implementation takes caching-related issues into consideration.
The Half-Fit algorithm itself is inherently optimized to minimize the number of random memory accesses.
Furthermore, the allocation strategy favors most recently used memory fragments
to minimize cache misses in the application.

### Implementation

The implemented variant of Half-Fit allocates memory in fragments of size:

F(r) = 2<sup>⌈log<sub>2</sub>(r+a)⌉</sup>

Where *r>0* is the requested allocation size and *a* is the fixed per-allocation metadata overhead
implicitly introduced by the allocator for memory management needs.
The size of the overhead *a* is represented in the codebase as `O1HEAP_ALIGNMENT`,
because it also dictates the allocated memory pointer alignment.
Due to the overhead, the maximum amount of memory available to the application per allocation is F'(r) = F(r) - a.
The amount of the overhead per allocation and, therefore, pointer alignment is 4×(pointer width);
e.g., for a 32-bit platform, the overhead/alignment is 16 bytes (128 bits).

From the above follows that *F(r) ≥ 2 a*.
Remember that *r>0* -- following the semantics of `malloc(..)`,
the allocator returns a null pointer if a zero-sized allocation is requested.

It has been mentioned that the abstract definition of *H* does not take into account the
implementation-specific overheads.
Said overheads should be considered when calculating the amount memory needed for a specific application.
We define a refined worst-case memory consumption (WCMC) model below.

nf = ⌈n/l⌉

Mf = ⌈M/l⌉

k = Mf - nf + 1

Where *l* -- the smallest amount of memory that may be requested by the application;
*nf* -- the size of the largest allocation expressed as the number of min-size fragments;
*Mf* -- the total amount of heap space that may be requested by the application in min-size fragments;
*k* -- the maximum number of fragments.
The worst case number of min-size memory fragments required is *Hf(Mf,nf) = H(Mf,nf)*.
The total amount of space needed to accommodate the per-fragment overhead is *(k×a)*.
Then, the total WCMC, expressed in bytes, is:

Hb(M,n,l,a) = a k + (2 l n Mf (⌈log<sub>2</sub>nf⌉ + 1)) / (l+n)

**The above equation should be used for sizing the heap space.**
Observe that the case where *l=n* degenerates into the standard fixed-size block allocator.
Increase of *l* lowers the upper bound because larger fragments inherently reduce the fragmentation.

The following illustration shows the worst-case memory consumption (WCMC) for some common memory sizes;
as explained above, *l* is chosen by the application designer freely,
and *a* is the value of `O1HEAP_ALIGNMENT` which is platform-dependent:

![WCMC](docs/H.png "Total worst-case memory consumption (H) as a function of max fragment size (n) and total memory need (M)")

## Usage

### Integration

Copy the files `o1heap.c` and `o1heap.h` (find them under `o1heap/`) into your project tree.
Either keep them in the same directory, or make sure that the directory that contains the header
is added to the set of include look-up paths.
No special compiler options are needed to compile the source file (if you find this to be untrue, please open a ticket).

Dedicate a memory arena for the heap, and pass a pointer to it along with its size to the initialization function
`o1heapInit(..)`.

Allocate and deallocate memory using `o1heapAllocate(..)` and `o1heapFree(..)`.
Their semantics are compatible with `malloc(..)` and `free(..)` plus additional behavioral guarantees
(constant timing, bounded fragmentation).

If necessary, periodically invoke `o1heapDoInvariantsHold(..)` to ensure that the heap is functioning correctly
and its internal data structures are not damaged.

Avoid concurrent access to the heap. Use locking if necessary.

### Build configuration options

The preprocessor options given below can be overridden to fine-tune the implementation.
None of them are mandatory to use.

#### O1HEAP_CONFIG_HEADER

Define this optional macro like `O1HEAP_CONFIG_HEADER="path/to/my_o1heap_config.h"` to pass build configuration macros.
This is useful because some build systems do not allow passing function-like macros via command line flags.

#### O1HEAP_ASSERT(x)

The macro `O1HEAP_ASSERT(x)` can be defined to customize the assertion handling or to disable it.
To disable assertion checks, the macro should expand into `(void)(x)`.
If not specified, the macro expands into the standard assertion check macro `assert(x)` as defined in `<assert.h>`.

#### O1HEAP_LIKELY(x)

Some of the conditional branching statements are equipped with this annotation to hint the compiler that
the generated code should be optimized for the case where the corresponding branch is taken.
This is done to reduce the worst-case execution time.

The macro should expand into a compiler-specific branch weighting intrinsic,
or into the original expression `(x)` if no such hinting is desired.
If not specified, the macro expands as follows:

- For some well-known compilers the macro automatically expands into appropriate branch weighting intrinsics.
  For example, for GCC, Clang, and ARM Compiler, it expands into `__builtin_expect((x), 1)`.
- For other (unknown) compilers it expands into the original expression with no modifications: `(x)`.

## Development

### Dependencies

The following tools should be available locally to conduct library development:

- Modern versions of CMake, GCC, Clang, and Clang-Tools.
- An AMD64 machine.
- (optional) Valgrind.

### Conventions

The codebase shall follow the [Zubax C/C++ Coding Conventions](https://kb.zubax.com/x/84Ah).
Compliance is enforced through the following means:

- Clang-Tidy -- invoked automatically while building the test suite.
- Clang-Format -- invoked manually as `make format`; enforced in CI/CD automatically.
- SonarCloud -- invoked by CI/CD automatically.

### Testing

Please refer to the continuous integration configuration to see how to invoke the tests.

### Releasing

Update the version number macro in the header file and create a new git tag like `1.0`.

### MISRA compliance

MISRA compliance is enforced with the help of the following tools:

- Clang-Tidy -- invoked automatically during the normal build process.
- SonarCloud -- invoked as part of the continuous integration build.

Every intentional deviation shall be documented and justified in-place using the following notation,
followed by the appropriate static analyser warning suppression statement:

```c
// Intentional violation of MISRA: <valid reason here>
// NOSONAR
// NOLINT
```

The list of intentional deviations can be obtained by simply searching the codebase for the above comments.

Do not suppress compliance warnings using the means provided by static analysis tools because such deviations
are impossible to track at the source code level.
An exception applies for the case of false-positive (invalid) warnings -- those should not be mentioned in the codebase.

## Further reading

- [Timing-Predictable Memory Allocation In Hard Real-Time Systems](https://publikationen.sulb.uni-saarland.de/bitstream/20.500.11880/26614/1/diss.pdf), J. Herter, 2014.
- [Worst case fragmentation of first fit and best fit storage allocation strategies](https://academic.oup.com/comjnl/article/20/3/242/751782), J. M. Robson, 1975.
- [Dynamic Memory Allocation In SQLite](https://sqlite.org/malloc.html) -- on Robson proof and deterministic fragmentation.
- *[Russian]* [Динамическая память в системах жёсткого реального времени](https://habr.com/ru/post/486650/) -- issues with dynamic memory allocation in modern embedded RTOS and related popular misconceptions.

## Changelog

### v2.0

- Remove critical section hooks to enhance MISRA conformance [#4](https://github.com/pavel-kirienko/o1heap/issues/4)
- Add support for config header via `O1HEAP_CONFIG_HEADER` [#5](https://github.com/pavel-kirienko/o1heap/issues/5)

### v1.0

The first release.

## License

The library is available under the terms of the MIT License.
Please find it in the file `LICENSE`.
