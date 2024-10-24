---
title: "A Deep Dive into Golang Memory"
date: "2024-10-29"
categories: ["memory"]
tags: ["memory", "Go"]
slug: "memory-golang"
---

Let's try to understand the memory use of an application written in Golang. For
memory management, Golang uses garbage collection which means that
allocating and freeing memory is mostly transparent to the user. While it makes
the manipulation of memory easy at first glance, troubleshooting memory issues
requires you to understand how the garbage collector works.

*The quote part of this article are cut-outs from [Golang Documentation: A
Guide to the Go Garbage Collector](https://tip.golang.org/doc/gc-guide). Note
that it might be outdated, the article was written as of v1.22. Here are some
other important resources:
[Go: The Optimization guide](https://tip.golang.org/doc/gc-guide#Optimization_guide)
and [Go: Diagnostics](https://go.dev/doc/diagnostics#profiling). There are so
many resources out there (and I'm adding another one!), but I found this
[Google groups message](https://groups.google.com/g/golang-nuts/c/LsOYrYc_Occ/m/QrIW1sryBwAJ)
to have good links as well. You will find more links through the article.*

## Garbage collection

> [...] In the context of this document, garbage collection refers
> to tracing garbage collection, which identifies in-use, so-called live,
> objects by following pointers transitively.
>
> Together, objects and pointers to other objects form the object graph. To
> identify live memory, the GC walks the object graph starting at the
> program's roots, pointers that identify objects that are definitely in-use by
> the program. Two examples of roots are local variables and global variables.
> The process of walking the object graph is referred to as scanning.
>
> [...] Go's GC uses the mark-sweep technique, which means that in order to
> keep track of its progress, the GC also marks the values it encounters as
> live. Once tracing is complete, the GC then walks over all memory in the heap
> and makes all memory that is not marked available for allocation. This
> process is called sweeping.

See this video for details about the Mark & Sweep technic: [Garbage Collection
(Mark & Sweep) by Computerphile](https://www.youtube.com/watch?v=c32zXYAK7CI);
a “Mark & Sweep” algorithm walks the roots of a program to find objects (and
follow their objects and pointers) to mark them as live. Then it recycles the
memory used for all the unreachable objects.

### The GOGC parameter

The GOGC parameter determines the trade-off between GC CPU use and memory. To
put it simply, the more CPU cycles you spend on garbage collecting, the
closest you can stick to the live heap (the memory you actually need), but the
less you actually spend time executing the actual program. So by default,
GOGC is equal to 100 which means that the process may use 100% more memory than
needed to run. It can lead to frustrating situations where you consume around
50MB of heap but the actual impact of the Go heap is around 100MB or a bit
more. See [this
thread](https://groups.google.com/g/golang-nuts/c/SNRW-f1F9aM/m/BZiO0AgGAgAJ)
from someone confused about why `memstats.HeapInUse` is twice the total in
pprof.

> The key takeaway is that doubling GOGC will double heap memory overheads and
> roughly halve GC CPU cost, and vice versa.

Here are some details on how GOGC is defined:

> It works by determining the target heap size after each GC cycle, a target
> value for the total heap size in the next cycle. The GC's goal is to finish a
> collection cycle before the total heap size exceeds the target heap size.
> Total heap size is defined as the live heap size at the end of the previous
> cycle, plus any new heap memory allocated by the application since the
> previous cycle. Meanwhile, target heap memory is defined as:
>
> *Target heap memory = Live heap + (Live heap + GC roots) \* GOGC / 100*

See the following situation with `GOGC` = 100, you can see that most of the
time after startup, this program used around 20MiB of memory but the impact of
its heap was at maximum around 40MiB, and the slight spike at 30MiB made the
process needing 60MiB.

{{<
    figure
    align=center
    src="./gogc.png"
    caption="Memory trace with `GOGC` = 100"
    height=200px
    class=keep-aspect-ratio
>}}

A problem arises from the above situation, what if you need to run your program
in a memory-constrained environment and you don't want the OOM killer to
immediately reap your process because of a sudden spike that concerns only a
small portion of its execution. You actually need to know your program well to
bump into that specific issue, a good first step would be to just
reduce the overall memory consumption before trying to tackle transient memory
spikes. However one can note that this situation often occurs at application
startup when initializing subsystems.

### The Go Memory Limit

For workload runnings in constrained environments (typically containers), Go
1.19 introduced `GOMEMLIMIT`, in order to take advantage of a high `GOGC` while
not getting OOMed during transient memory spikes.

Now with `GOGC` = 100 (like the previous figure) and `GOMEMLIMIT` = 35 MB:
￼
{{<
    figure
    align=center
    src="./gogc_gomemlimit.png"
    caption="Memory trace with `GOGC` = 100 and `GOMEMLIMIT` = 35 MB"
    height=200px
    class=keep-aspect-ratio
>}}

> Now, while the memory limit is clearly a powerful tool, the use of a memory
> limit does not come without a cost, and certainly doesn't invalidate the
> utility of GOGC.
>
> [...] This situation, where the program fails to make reasonable progress due
> to constant GC cycles, is called thrashing. It's particularly dangerous
> because it effectively stalls the program. Even worse, it can happen for
> exactly the same situation we were trying to avoid with GOGC: a large enough
> transient heap spike can cause a program to stall indefinitely!
>
> [...] In many cases, an indefinite stall is worse than an out-of-memory
> condition, which tends to result in a much faster failure.
>
> For this reason, the memory limit is defined to be soft. The Go runtime makes
> no guarantees that it will maintain this memory limit under all
> circumstances; it only promises some reasonable amount of effort.

The guide thus recommends using memory limit when executing in an environment
under control or restricted, like containers.
> [...] A good example is the deployment of a web service into containers with
> a fixed amount of available memory. In this case, a good rule of thumb is to
> leave an additional 5-10% of headroom to account for memory sources the Go
> runtime is unaware of.

So while the `GOMEMLIMIT` can be useful when fine-tuning an application
running under known load, we'll now focus on understanding the link between
what we have seen in RSS use before and the Go memory statistics. We'll also
see how to profile memory allocation to find the location in your program that
are guilty of consuming too much memory.

## Understand and Reduce Go Heap

The best way to reduce memory consumption from our Go process is to actually
diagnose what consumes the most memory and change or optimize the program using
profiling. Most of the time, what is under our control is the heap since it is
directly influenced by how we allocate and keep reference of objects.

### Memory statistics with memstats

A good tool to diagnose Go processes and retrieve the memory statistics at
runtime is [gops](https://github.com/google/gops). You can find Go
[memstat documentation](https://cs.opensource.google/go/go/+/refs/tags/go1.23.2:src/runtime/mstats.go;l=52)
in the runtime sources. Let's try to understand those stats:

```shell
gops memstats localhost:8118
```

The output should look similar to this:

```text {linenos=inline,hl_lines=[1,3,7,8,9,10,11,21]}
alloc: 9.92MB (10398000 bytes)
total-alloc: 3.51GB (3768355616 bytes)
sys: 173.28MB (181693736 bytes)
lookups: 0
mallocs: 30603655
frees: 30531193
heap-alloc: 9.92MB (10398000 bytes)
heap-sys: 163.19MB (171114496 bytes)
heap-idle: 146.89MB (154025984 bytes)
heap-in-use: 16.30MB (17088512 bytes)
heap-released: 142.20MB (149110784 bytes)
heap-objects: 72462
stack-in-use: 832.00KB (851968 bytes)
stack-sys: 832.00KB (851968 bytes)
stack-mspan-inuse: 317.19KB (324800 bytes)
stack-mspan-sys: 1.07MB (1126080 bytes)
stack-mcache-inuse: 7.03KB (7200 bytes)
stack-mcache-sys: 15.23KB (15600 bytes)
other-sys: 1.58MB (1653162 bytes)
gc-sys: 4.53MB (4751888 bytes)
next-gc: when heap-alloc >= 16.04MB (16817032 bytes)
last-gc: 2024-10-28 11:21:09.087026419 +0100 CET
gc-pause-total: 19.390736ms
gc-pause: 225204
gc-pause-end: 1730110869087026419
num-gc: 356
num-forced-gc: 1
gc-cpu-fraction: 0.000693765282256322
enable-gc: true
debug-gc: false
```

Let's focus on the highlighted lines:

| Statistic | Definition | Details |
| --------- | ---------- | ------- |
| `alloc` | Same as `heap-alloc`. | Same as `heap-alloc`. |
| `sys` | Sys is the total bytes of memory obtained from the OS. | Sys is the sum of the XSys fields below. Sys measures the virtual address space reserved by the Go runtime for the heap, stacks, and other internal data structures. It's likely that not all of the virtual address space is backed by physical memory at any given moment, though in general it all was at some point. |
| `heap-alloc` | HeapAlloc is bytes of allocated heap objects. | "Allocated" heap objects include all reachable objects, as well as unreachable objects that the garbage collector has not yet freed. Specifically, HeapAlloc increases as heap objects are allocated and decreases as the heap is swept and unreachable objects are freed. |
| `heap-sys` | HeapSys is bytes of heap memory obtained from the OS. | HeapSys measures the amount of virtual address space reserved for the heap. This includes virtual address space that has been reserved but not yet used, which consumes no physical memory, but tends to be small, as well as virtual address space for which the physical memory has been returned to the OS after it became unused (see HeapReleased for a measure of the latter). HeapSys estimates the largest size the heap has had. |
| `heap-idle` | HeapIdle is bytes in idle (unused) spans. | Idle spans have no objects in them. These spans could be (and may already have been) returned to the OS, or they can be reused for heap allocations, or they can be reused as stack memory. HeapIdle minus HeapReleased estimates the amount of memory that could be returned to the OS, but is being retained by the runtime so it can grow the heap without requesting more memory from the OS. If this difference is significantly larger than the heap size, it indicates there was a recent transient spike in live heap size. |
| `heap-in-use` | HeapInuse is bytes in in-use spans. | In-use spans have at least one object in them. These spans can only be used for other objects of roughly the same size. HeapInuse minus HeapAlloc estimates the amount of memory that has been dedicated to particular size classes, but is not currently being used. This is an upper bound on fragmentation, but in general this memory can be reused efficiently. |
| `heap-released` | HeapReleased is bytes of physical memory returned to the OS. | This counts heap memory from idle spans that was returned to the OS and has not yet been reacquired for the heap. |
| `next-gc` | NextGC is the target heap size of the next GC cycle. | The garbage collector's goal is to keep HeapAlloc ≤ NextGC. At the end of each GC cycle, the target for the next cycle is computed based on the amount of reachable data and the value of GOGC. |

NextGC gives us interesting information, if most memory has been released to
OS and the program is in a stable state, the heap alloc target should be the
approximate impact of the heap.

We can find other interesting information in the documentation of
`runtime/debug`'s [`SetMemoryLimit`](https://pkg.go.dev/runtime/debug#SetMemoryLimit):
> More specifically, the following expression accurately reflects the value the
> runtime attempts to maintain as the limit: `runtime.MemStats.Sys - runtime.MemStats.HeapReleased`

As this article from Datadog [Go memory metrics demystified](https://www.datadoghq.com/blog/go-memory-metrics/) explains, this
value is the runtime's best estimate for the amount of physical Go memory that
it is managing. Note that it would still be imprecise because the Go runtime
only deals with virtual memory so it can only estimate what it thinks it is
currently using.

So there's chance that this value is close to the process RSS and thus a good
next idea is to use profiling to find out where is allocated the memory you
see. If it's well higher, it might be because the Go runtime overestimates the
value because of virtual memory, and if it's well lower, it may be because your
program is using more non-Go memory, from CGO or mmap directly, this might get
trickier to diagnose.

### Memory profiling with pprof

To understand how your heap memory is actually used, the embedded Go profiler
is a good solution. Keep in mind that what you will see are the allocations of
memory of live objects. The snapshot is taken after the GC has been running and
it's again normal that the RSS of your heap memory segment is twice the size
you see (see the part section about the GOGC).

While you can take pprof heap dumps with gops, you can also start an HTTP
server that serves [pprof](https://pkg.go.dev/net/http/pprof) profiles directly
from your process. Then you can use `go tool pprof` to expose a web UI and
interact with the profile easily.

If you have the pprof HTTP server listening on `localhost:6116`, you can then
use this to read the heap profile:
```shell
go tool pprof --http localhost:3333 localhost:6116/debug/pprof/heap
```

Going on `http://localhost:3333` you will be able to explore the data with
different visualizations:

{{<
    figure
    align=center
    src="./graph.png"
    caption="Graph view of heap the profile"
    height=730px
    class=keep-aspect-ratio
>}}

You can also use the flame graph representation:

{{<
    figure
    align=center
    src="./flame_graph.png"
    caption="Flame graph view of the heap profile"
    height=730px
    class=keep-aspect-ratio
>}}

And the best thing is that if you have the code available, you can directly see
where in the code the allocation took place by right-clicking on the flame
graph for example.


{{<
    figure
    align=center
    src="./src.png"
    caption="Source code indicating heap allocation"
    height=600px
    class=keep-aspect-ratio
>}}

The visualisations should be pretty clear and you should find who is guilty of
allocating all that memory. Note that we often only look at `inuse_space`, but
`alloc_objects` can also be useful for seeing which areas of your code allocate
the most objects and make the work of the garbage collector hard (and thus might
force more CPU cycles into collecting).

You can read Julia Evans article on [Profiling Go programs with
pprof](https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/).

## Go runtime and virtual memory

> Because virtual memory is just a mapping maintained by the operating system,
> it is typically very cheap to make large virtual memory reservations that
> don't map to physical memory.
>
> The Go runtime generally relies upon this view of the cost of virtual memory
> in a few ways:
> * The Go runtime never deletes virtual memory that it maps. Instead, it uses
>   special operations that most operating systems provide to explicitly
>   release any physical memory resources associated with some virtual memory
>   range. This technique is used explicitly to manage the memory limit and
>   return memory to the operating system that the Go runtime no longer needs.
>   The Go runtime also releases memory it no longer needs continuously in the
>   background. See the additional resources for more information.
>
> [...]
>
> As a result, virtual memory metrics such as "VSS" in top are typically not
> very useful in understanding a Go program's memory footprint. Instead, focus
> on "RSS" and similar measurements, which more directly reflect physical
> memory usage.

See more about that in the [Smarter scavenging proposal](https://github.com/golang/proposal/blob/master/design/30333-smarter-scavenging.md),
issue [#30333](https://github.com/golang/go/issues/30333); process can be OOMed
because RSS use is too high, improving scavenging improves how the GO GC gives
back memories to the OS.

Again here, similarly to what we saw with the GOGC parameter, there's a
tradeoff between returning the free memory to the OS to reduce the amount of
memory the process used and the performance cost of doing so.

> * Returning all free memory back to the underlying system at once is expensive,
> and can lead to latency spikes as it holds the heap lock through the whole
> process.
> * [...]
> * Reusing free chunks of memory becomes more expensive. On UNIX-y systems that
> means an extra page fault (which is surprisingly expensive on some systems).

In any case, this part is mostly out of control for the end user and is
maintained and improved by the people working on the Go runtime.

### Memory advise DONTNEED and FREE

The [Go v1.12 release notes](https://go.dev/doc/go1.12#runtime) brought
something new about how it released physical memory. While on one hand, the Go
runtime now released memory back to the OS more aggressively, it also started
to use `MADV_FREE` on Linux which basically means that the OS would not
actually free the memory if it's not under pressure. It was implemented by [CL
135395](https://go-review.googlesource.com/c/go/+/135395) (I just noticed it
was by my colleague Tobias!):
> The Go runtime now releases memory back to the operating system more
> aggressively, particularly in response to large allocations that can’t reuse
> existing heap space.
>
> [...]
>
> On Linux, the runtime now uses MADV_FREE to release unused memory. This is
> more efficient but may result in higher reported RSS. The kernel will reclaim
> the unused data when it is needed. To revert to the Go 1.11 behavior
> (MADV_DONTNEED), set the environment variable GODEBUG=madvdontneed=1.

See more about `MADV_DONTNEED` and `MADV_FREE` in the man page `madvise(2)`:
```
MADV_DONTNEED
        Do not expect access in the near future.  (For the time
        being, the application is finished with the given range,
        so the kernel can free resources associated with it.)

        After a successful MADV_DONTNEED operation, the semantics
        of memory access in the specified region are changed:
        subsequent accesses of pages in the range will succeed,
        but will result in either repopulating the memory contents
        from the up-to-date contents of the underlying mapped file
        (for shared file mappings, shared anonymous mappings, and
        shmem-based techniques such as System V shared memory
        segments) or zero-fill-on-demand pages for anonymous
        private mappings.

        Note that, when applied to shared mappings, MADV_DONTNEED
        might not lead to immediate freeing of the pages in the
        range.  The kernel is free to delay freeing the pages
        until an appropriate moment.  The resident set size (RSS)
        of the calling process will be immediately reduced
        however.

[...]

MADV_FREE (since Linux 4.5)
        The application no longer requires the pages in the range
        specified by addr and len.  The kernel can thus free these
        pages, but the freeing could be delayed until memory
        pressure occurs.  For each of the pages that has been
        marked to be freed but has not yet been freed, the free
        operation will be canceled if the caller writes into the
        page.  After a successful MADV_FREE operation, any stale
        data (i.e., dirty, unwritten pages) will be lost when the
        kernel frees the pages.  However, subsequent writes to
        pages in the range will succeed and then kernel cannot
        free those dirtied pages, so that the caller can always
        see just written data.  If there is no subsequent write,
        the kernel can free the pages at any time.  Once pages in
        the range have been freed, the caller will see zero-fill-
        on-demand pages upon subsequent page references.

        The MADV_FREE operation can be applied only to private
        anonymous pages (see mmap(2)).  Before Linux 4.12, when
        freeing pages on a swapless system, the pages in the given
        range are freed instantly, regardless of memory pressure.
```

Only for this change to be reverted in Go 1.16, by [CL 267100](https://go-review.googlesource.com/c/go/+/267100),
mostly because it was creating confusion on RSS consumption and generally more
harm than good:
> In Go 1.12, we changed the runtime to use MADV_FREE when available on Linux
> (falling back to MADV_DONTNEED) in CL 135395 to address issue #23687. While
> MADV_FREE is somewhat faster than MADV_DONTNEED, it doesn't affect many of
> the statistics that MADV_DONTNEED does until the memory is actually reclaimed
> under OS memory pressure. This generally leads to poor user experience, like
> confusing stats in top and other monitoring tools; and bad integration with
> management systems that respond to memory usage.
>
> We've seen numerous issues about this user experience, including #41818,
> #39295, #37585, #33376, and #30904, many questions on Go mailing lists, and
> requests for mechanisms to change this behavior at run-time, such as #40870.
> There are also issues that may be a result of this, but root-causing it can
> be difficult, such as #41444 and #39174. And there's some evidence it may
> even be incompatible with Android's process management in #37569.
>
> This CL changes the default to prefer MADV_DONTNEED over MADV_FREE, to favor
> user-friendliness and minimal surprise over performance. I think it's become
> clear that Linux's implementation of MADV_FREE ultimately doesn't meet our
> needs. We've also made many improvements to the scavenger since Go 1.12. In
> particular, it is now far more prompt and it is self-paced, so it will simply
> trickle memory back to the system a little more slowly with this change. This
> can still be overridden by setting GODEBUG=madvdontneed=0.

Appearing in the [v1.16 changelog](https://go.dev/doc/go1.16#runtime)
> On Linux, the runtime now defaults to releasing memory to the operating
> system promptly (using MADV_DONTNEED), rather than lazily when the operating
> system is under memory pressure (using MADV_FREE). This means process-level
> memory statistics like RSS will more accurately reflect the amount of
> physical memory being used by Go processes. Systems that are currently using
> GODEBUG=madvdontneed=1 to improve memory monitoring behavior no longer need
> to set this environment variable.

Here is [an excellent StackOverflow answer](https://stackoverflow.com/a/73633428/4561420)
that tries to put together many things we've seen here and this very nice
article from Chris Siebenmann [Go basically never frees heap memory back to the
operating system](https://utcc.utoronto.ca/~cks/space/blog/programming/GoNoMemoryFreeing).


### Golang runtime memory overhead

While looking at the memory segments of Tetragon running, I noticed a segment I
couldn't explain and that was using significant physical memory.

```text {linenos=inline,hl_lines=[15]}
725883:   ./tetragon --bpf-lib bpf/objs/ --tracing-policy-dir /home/mtardy.linux/tetragon/examples/tracingpolicy/set --gops-address localhost:8118
Address           Kbytes     RSS   Dirty Mode  Mapping
0000000000010000   41988   25732       0 r-x-- /home/mtardy.linux/tetragon/tetragon
0000000002920000   31288   19704       0 r---- /home/mtardy.linux/tetragon/tetragon
00000000047b0000    1120    1120     204 rw--- /home/mtardy.linux/tetragon/tetragon
00000000048c8000     344     176     176 rw---   [ anon ]
0000004000000000  184320   22792   22792 rw---   [ anon ]
000000400b400000   12288       0       0 -----   [ anon ]
0000ee86fc740000      68      68       4 rw-s-   [ anon ]
0000ee86fc751000      68      68       4 rw-s-   [ anon ]
0000ee86fc762000      68      68       4 rw-s-   [ anon ]
0000ee86fc773000      68      68       4 rw-s-   [ anon ]
0000ee86fc784000      68      68       4 rw-s-   [ anon ]
0000ee86fc795000      68      68       4 rw-s-   [ anon ]
0000ee86fc7a6000    7528    6464    6464 rw---   [ anon ]
0000ee86fcf00000   33792      12      12 rw---   [ anon ]
0000ee86ff000000     512       0       0 -----   [ anon ]
0000ee86ff080000       4       4       4 rw---   [ anon ]
0000ee86ff081000  524284       0       0 -----   [ anon ]
0000ee871f080000       4       4       4 rw---   [ anon ]
0000ee871f081000  523836       0       0 -----   [ anon ]
0000ee873f010000       4       4       4 rw---   [ anon ]
0000ee873f011000   65476       0       0 -----   [ anon ]
0000ee8743002000       4       4       4 rw---   [ anon ]
0000ee8743003000    8180       0       0 -----   [ anon ]
0000ee8743808000     576     564     564 rw---   [ anon ]
0000ee87438a8000      72      72      72 rw---   [ anon ]
0000ee87438ba000    1020       0       0 -----   [ anon ]
0000ee87439b9000     384      56      56 rw---   [ anon ]
0000ee8743a19000       8       0       0 r----   [ anon ]
0000ee8743a1b000       4       4       0 r-x--   [ anon ]
0000ffffe0a0d000     132      16      16 rw---   [ stack ]
---------------- ------- ------- -------
total kB         1437576   77136   30396
```

You can see the four first memory segments that is the machine code, the
read-only data, the plain and the zeroed data. This is pretty much linked
directly to the binary and its global variables. Next after that, on lines 7 and
8 you see the heap.

But how to actually identify all the rest? We know that Golang needs to
allocate memory for its Goroutines stacks but it seemed like a lot of memory
for only that so I dug deeper. Looking at the memory stats, we can see some
fields that could make up for what is indeed missing.

```text {linenos=inline,hl_lines=[15,16,17,18,19,20]}
alloc: 10.43MB (10933720 bytes)
total-alloc: 3.50GB (3761223816 bytes)
sys: 189.37MB (198569256 bytes)
lookups: 0
mallocs: 30573730
frees: 30490529
heap-alloc: 10.43MB (10933720 bytes)
heap-sys: 179.16MB (187858944 bytes)
heap-idle: 161.93MB (169795584 bytes)
heap-in-use: 17.23MB (18063360 bytes)
heap-released: 157.88MB (165543936 bytes)
heap-objects: 83201
stack-in-use: 864.00KB (884736 bytes)
stack-sys: 864.00KB (884736 bytes)
stack-mspan-inuse: 340.16KB (348320 bytes)
stack-mspan-sys: 1.32MB (1387200 bytes)
stack-mcache-inuse: 7.03KB (7200 bytes)
stack-mcache-sys: 15.23KB (15600 bytes)
other-sys: 1.38MB (1442284 bytes)
gc-sys: 4.59MB (4815256 bytes)
next-gc: when heap-alloc >= 15.81MB (16576104 bytes)
last-gc: 2024-10-29 11:47:20.632253233 +0100 CET
gc-pause-total: 27.900551ms
gc-pause: 259956
gc-pause-end: 1730198840632253233
num-gc: 337
num-forced-gc: 1
gc-cpu-fraction: 0.0026844331524572876
enable-gc: true
debug-gc: false
```

Some of these stats seem to be miss-label as "stack" related while they are
not, I think [it's just a mistake from gops](https://github.com/google/gops/pull/63).
Indeed, `MSpanInuse`, `MSpanSys`, `MCacheInuse` and `MCacheSys` are [documented as](https://go.dev/src/runtime/mstats.go):
> The following statistics measure runtime-internal structures that are not
> allocated from heap memory (usually because they are part of implementing the
> heap). Unlike heap or stack memory, any memory allocated to these structures
> is dedicated to these structures.
>
> These are primarily useful for debugging runtime memory overheads.

| Statistic | Definition |
| --------- | ---------- |
| `mspan-inuse` | MSpanInuse is bytes of allocated mspan structures. |
| `mspan-sys` | MSpanSys is bytes of memory obtained from the OS for mspan structures. |
| `mcache-inuse` | MCacheInuse is bytes of allocated mcache structures. |
| `mcache-sys` | MCacheSys is bytes of memory obtained from the OS for mcache structures. |
| `gc-sys` | GCSys is bytes of memory in garbage collection metadata. |
| `other-sys` | OtherSys is bytes of memory in miscellaneous off-heap runtime allocations. |

We can note that `BackHashSys` is another of such stats and it's missing from
gops exports.

Making a sum of all the above we find something like 8015860 bytes (8.02MB)
which kinda make sense for this memory and all the memory segments we see from
lines 9 to 31.

To go a little deeper, I found this excellent blog post about [A deep dive into
the OS memory use of a simple Go program](https://utcc.utoronto.ca/~cks/space/blog/programming/GoProgramMemoryUse)
and it had a good recommendation for trying to make sense of all that, "try to
breakpoint at `runtime.sysAlloc`". While debugging the Go runtime I bumped into
the `runtime.(*mheap).allocManual` function and the
[`NotInHeap`](https://cs.opensource.google/go/go/+/refs/tags/go1.23.2:src/runtime/internal/sys/nih.go;l=41)
internal type.
> Other types can embed NotInHeap to make it not-in-heap. Specifically,
> pointers to these types must always fail the `runtime.inheap` check. The type
> may be used for global variables, or for objects in unmanaged memory (e.g.,
> allocated with `sysAlloc`, `persistentalloc`, r`fixalloc`, or from a
> manually-managed span).

This is also explained in the
[`HACKING.md`](https://github.com/golang/go/blob/851ebc2dca616754fa2bd6c48241e498bf306a50/src/runtime/HACKING.md#unmanaged-memory)
of the runtime source code. But you can search for it in the runtime source
code and it's used to mark a type to be allocated manually.

You can find many more resources on how the Go allocator works, here are some
links to ["Allocator Wrestling"](https://utcc.utoronto.ca/~cks/space/blog/links/GoAllocatorWrestling),
a good presentation about the allocator from GopherCon 2018

## Conclusion

To debug the memory consumption of a Go process:

1. A good first step is to [look at memory segments](/posts/memory-kubernetes-golang-ebpf/#reading-a-linux-process-memory-use)
   using `pmap(1)` for example. You will typically see three main consumption
   poles:
   - the binary being mapped to memory;
   - the Go heap;
   - the overhead of the Go runtime.
1. You could try to reduce the size of the binary by stripping debug symbols or
   getting rid of dependencies.
1. Then, the typically most efficient idea is to investigate the Go heap with
   [memstats](#memory-statistics-with-memstats) and [pprof](#memory-profiling-with-pprof).
   Again, note that the Go heap will usually be twice as large as necessary due
   to the garbage collector, see [more details above](#garbage-collection).
1. Finally, if you have too much free time, try to [understand
   more](#go-runtime-and-virtual-memory) about how the runtime returns memory to
   the OS and its memory overhead.
