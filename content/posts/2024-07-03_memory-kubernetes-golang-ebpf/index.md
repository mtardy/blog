---
title: "Memory Management on Kubernetes with Golang and eBPF: Deep Dive"
date: "2024-07-03"
categories: ["memory"]
tags: ["memory", "linux", "cgroups", "Go", "ebpf", "kubernetes"]
slug: "memory-kubernetes-golang-ebpf"
description: "A digest article on workload memory use, the memory control
group, the OOM killer, and their relation with applications running on
Kubernetes using Go and eBPF."
tocopen: true
---

**tl;dr**: please [jump directly to the conclusion]({{< ref "#conclusion" >}})
if you think you already have some knowledge about memory and just want the
recap. The conclusion links back to the other sections for more details.

## Disclaimer

This (long) article is a Frankenstein :zombie: compilation of personal digest
notes on various topics. Some sections are direct digests or extracts of very
good resources I found on the topic and could not write something better:
- [Systems Performance: Enterprise and the Cloud, 2nd Edition
  (2020)](https://www.brendangregg.com/systems-performance-2nd-edition-book.html)
  by Brendan Gregg.
- [Out-of-memory (OOM) in Kubernetes – Part 2: The OOM killer and application
  runtime implications](https://mihai-albert.com/2022/02/13/out-of-memory-oom-in-kubernetes-part-2-the-oom-killer-and-application-runtime-implications/) by Mihai Albert.
- [Out-of-memory (OOM) in Kubernetes - Part 3: Memory metrics sources and tools
  to collect them](https://mihai-albert.com/2022/02/13/out-of-memory-oom-in-kubernetes-part-3-memory-metrics-sources-and-tools-to-collect-them/) by Mihai Albert.
- [Golang Documentation: A Guide to the Go Garbage
  Collector](https://tip.golang.org/doc/gc-guide) by the Go maintainers.

If what you can find here is not enough on the specific sections marked as
extract, digest, or inspiration, read these articles directly instead. I also,
in this case, recommend reading most of this blogpost hyperlinks, those are the
best resources I could find so far.

---

## Introduction

To start this post about memory, let's define a few terms and quickly explain
how paging works on Linux. The following are extracts from the excellent
[Systems Performance: Enterprise and the Cloud, 2nd Edition
(2020)](https://www.brendangregg.com/systems-performance-2nd-edition-book.html)
by Brendan Gregg.

### Glossary

*This section is an extract from [Systems Performance: Enterprise and the
Cloud, 2nd Edition (2020)](https://www.brendangregg.com/systems-performance-2nd-edition-book.html)
by Brendan Gregg.*

* **Main memory**: Also referred to as physical memory, this describes the fast
  data storage area of a computer, commonly provided as DRAM.
* **Virtual memory**: An abstraction of main memory that is (almost) infinite and
  non-contended. Virtual memory is not real memory.
* **Resident memory**: Memory that currently resides in main memory.
* **Anonymous memory**: Memory with no file system location or path name. It
  includes the working data of a process address space, called the heap.
* **Address space**: A memory context. There are virtual address spaces for each
  process, and for the kernel.
* **Segment**: An area of virtual memory flagged for a particular purpose, such as
  for storing executable or writeable pages.
* **Instruction text**: Refers to CPU instructions in memory, usually in a segment.
* **OOM**: Out of memory, when the kernel detects low available memory.
* **Page**: A unit of memory, as used by the OS and CPUs. Historically it is either
  4 or 8 Kbytes. Modern processors have multiple page size support for larger
  sizes.
* **Page fault**: An invalid memory access. These are normal occurrences when using
  on-demand virtual memory.
* **Paging**: The transfer of pages between main memory and the storage devices.
* **Swapping**: Linux uses the term swapping to refer to anonymous paging to the
  swap device (the transfer of swap pages). In Unix and other operating
  systems, swapping is the transfer of entire processes between main memory and
  the swap devices. This book uses the Linux version of the term.
* **Swap**: An on-disk area for paged anonymous data. It may be an area on a
  storage device, also called a physical swap device, or a file system file,
  called a swap file. Some tools use the term swap to refer to virtual memory
  (which is confusing and incorrect).

### Memory overcommit and the OOM killer

*This section is a digest of [Out-of-memory (OOM) in Kubernetes – Part 2: The
OOM killer and application runtime implications - Overcommit](https://mihai-albert.com/2022/02/13/out-of-memory-oom-in-kubernetes-part-2-the-oom-killer-and-application-runtime-implications/#overcommit)
by Mihai Albert.*

Linux is overcommitting memory (while, for your culture, Windows is not, the
original author wrote [another post on that topic](https://mihai-albert.com/2019/04/21/out-of-memory-exception/#your-reservation-is-now-confirmed)),
however, you can tune Linux by adjusting `sysctl vm.overcommit_memory=<value>`
with different modes. For example, `1` is "always overcommit", or `2` is
"prevent overcommit" (behaving similarly to Windows). Overcommitting memory
allows accommodating applications that allocate a lot of memory but don’t use
everything and is also consistent with the nature of forking, where the kernel
copy-on-writes the parent memory to the child. Note that there are heated
debates around the fact that [overcommit is a good or a bad thing](https://lwn.net/Articles/104179/).

The OOM killer watches when memory hits a critical level when the kernel has to
eventually back virtual memory to physical memory and choose a process to kill
in order to free memory. [Its principle documentation is clear and concise](https://www.kernel.org/doc/gorman/html/understand/understand016.html).
Note that even with tuning the `vm.overcommit_memory` sysctl, it’s hard to
actually disable the OOM killer completely.

### The Paging Mechanism

*This section is an extract from [Systems Performance: Enterprise and the
Cloud, 2nd Edition (2020)](https://www.brendangregg.com/systems-performance-2nd-edition-book.html)
by Brendan Gregg.*

{{<
    figure
    align=center
    src="./page_fault_example.png"
    link="https://www.brendangregg.com/systems-performance-2nd-edition-book.html"
    caption="Figure 7.2: Page fault example"
>}}

The result of the virtual memory model and demand allocation is that any page
of virtual memory may be in one of the following states:

1. Unallocated
2. Allocated, but unmapped (unpopulated and not yet faulted)
3. Allocated, and mapped to main memory (RAM)
4. Allocated, and mapped to the physical swap device (disk)

State (4) is reached if the page is paged out due to system memory pressure. A
transition from (2) to (3) is a page fault. If it requires disk I/O, it is a
major page fault; otherwise, a minor page fault.

From these states, two memory usage terms can also be defined:
- Resident set size (RSS): The size of allocated main memory pages (3)
- Virtual memory size: The size of all allocated areas (2 + 3 + 4)

## Process Memory

Understanding a process use of memory can be tricky and it's maybe
counter-intuitive but it's pretty hard to define a universal definition for
"memory use" and get the perfect statistic. In that regard, for more details,
Brendan Gregg has been trying to define and estimate a metric: the
[`Working Set Size`](https://www.brendangregg.com/wss.html).
Here are a few basic actions to read memory usage, and some more elaborated
methods for Golang and eBPF programs.

### Reading a Linux process memory use

First, you can read the RSS stat of the process, as detailed in the `proc(5)`
manpage, the read is fast but inaccurate. You can find the same information in
a parsable form at `/proc/pid/stat`, or measured in pages at `/proc/pid/statm`.
Let's read the "human" form at `/proc/pid/status`:
```shell
grep -i rss /proc/$(pidof <process>)/status
```

For slower but more accurate results, one can use `/proc/pid/smaps_rollup` as
per the `proc(5)` manpage.
```shell
sudo grep -i rss /proc/$(pidof <process>)/smaps_rollup
```

To get the details of RSS consumption by memory segment, you can use:
```shell
sudo grep -e '^[^A-Z]' -e Rss /proc/$(pidof <process>)/smaps | less
```

Or better, you can use `pmap(1)` to get nicely formatted version of
`/proc/pid/smaps`
```shell
sudo pmap $(pidof <process>) -xp
```

If looking at the segment, especially the anonymous ones isn't helpful, you
can try to trace the memory operation of the process to see what's happening.

```shell
strace -e trace=memory -o out.trace <cmd>
```

If you are looking at a runtime allocating memory, it can be rather confusing
and you better check the runtime tools to analyze anonymous memory consumption
directly.

### Golang memory use and the garbage collector

*This section is a cut-out from [Golang Documentation: A Guide to the Go
Garbage Collector](https://tip.golang.org/doc/gc-guide), this might be
outdated, it was written as of v1.22.*

Here are some important resources:
- [Go: The Garbage collector guide](https://tip.golang.org/doc/gc-guide);
- [Go: The Optimization guide](https://tip.golang.org/doc/gc-guide#Optimization_guide);
- [Go: Diagnostics](https://go.dev/doc/diagnostics#profiling);
- [Garbage Collection (Mark & Sweep) by Computerphile](https://www.youtube.com/watch?v=c32zXYAK7CI);
  a “Mark & Sweep” algorithm walks the roots of a program to find objects (and
  follow their objects and pointers) to mark them as live. Then it recycles the
  memory used for all the unreachable objects.
- [Smarter scavenging proposal](https://github.com/golang/proposal/blob/master/design/30333-smarter-scavenging.md),
  issue [#30333](https://github.com/golang/go/issues/30333); process can be
  OOMed because RSS use is too high, improving scavenging improves how the GO
  GC gives back memories to the OS.

#### Garbage collection

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

#### The GOGC parameter

GOGC determines the trade-off between GC CPU and memory

> It works by determining the target heap size after each GC cycle, a target
> value for the total heap size in the next cycle. The GC's goal is to finish a
> collection cycle before the total heap size exceeds the target heap size.
> Total heap size is defined as the live heap size at the end of the previous
> cycle, plus any new heap memory allocated by the application since the
> previous cycle. Meanwhile, target heap memory is defined as:
>
> *Target heap memory = Live heap + (Live heap + GC roots) \* GOGC / 100*

#### The Go Memory Limit

For workload runnings in constrained environments (typically containers), Go
1.19 introduced `GOMEMLIMIT`, in order to take advantage of a high `GOGC` while
not getting OOMed during transient memory spikes.

See the following situation with `GOGC` = 100:

{{<
    figure
    align=center
    src="./gogc.png"
    caption="Memory trace with `GOGC` = 100"
>}}

￼
Now with `GOGC` = 100 and `GOMEMLIMIT` = 35 MB:
￼
{{<
    figure
    align=center
    src="./gogc_gomemlimit.png"
    caption="Memory trace with `GOGC` = 100 and `GOMEMLIMIT` = 35 MB"
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

#### Go runtime and virtual memory

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
> * On 32-bit platforms, the Go runtime reserves between 128 MiB and 512 MiB of
>   address space up-front for the heap to limit fragmentation issues.
> * The Go runtime uses large virtual memory address space reservations in the
>   implementation of several internal data structures. On 64-bit platforms,
>   these typically have a minimum virtual memory footprint of about 700 MiB.
>   On 32-bit platforms, their footprint is negligible.
> **As a result, virtual memory metrics such as "VSS" in top are typically not
> very useful in understanding a Go program's memory footprint. Instead, focus
> on "RSS" and similar measurements, which more directly reflect physical
> memory usage.**

#### Understand and reduce Go heap use

The best way to reduce memory consumption is to actually diagnose what
consumes the most memory and change or optimize the program using profiling.

See two Golang official documentation resources, the [Optimization
guide](https://tip.golang.org/doc/gc-guide#Optimization_guide) and
[Diagnostics](https://go.dev/doc/diagnostics#profiling). Using the Go embedded
profiler [pprof](https://pkg.go.dev/net/http/pprof) is crucial, Julia Evans
propose a good article on [Profiling Go programs with
pprof](https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/).

Running [gops](https://github.com/google/gops) can also help to retrieve the
runtime memory values at runtime, you can find Go [memstat
documentation](https://go.dev/src/runtime/mstats.go) in the runtime sources.

### eBPF programs' memory impact

eBPF programs' memory impact is mostly due to BPF maps. They serve as a way to
store state, data and communicate between BPF programs and userspace
programs. Due to their static nature, most of them have to be allocated and
defined at compilation and thus empty or unused maps just use as much space as
used ones.

It seems that memory cgroup v1 does not account for memory used by maps, not even
in the `kernel` version of those stats. However, it changed for cgroup v2
certainly related to [this series of patches](https://lore.kernel.org/bpf/20201201215900.3569844-1-guro@fb.com/).
This can lead to a drastic change in overall memory consumption if you
switch from cgroup v1 to v2 while having a lot of BPF maps.

If you spot some major memory consumptions from unused maps and you cannot make
the existence of a map conditional, a good option is to make that map
`max_entries` equal to one (zero is invalid) and resize it at loading time in
the agent when needed.

Note that maps can be "anonymous" if the program loading them doesn’t pin them
properly and are actually used in the BPF code (and thus are allocated). They
are then not tied to the userspace process properly but account for memory
usage.

#### Some helpful bpftool commands

Here are a few commands, using the great [`bpftool`](https://github.com/libbpf/bpftool)
and `jq`, to gauge memory consumption of loaded maps.

Retrieve total memory usage of maps (in kB):
```shell
sudo bpftool map -j | jq '[ .[] | .bytes_memlock ] | add / 1000'
```

Same but filtered for the process with `comm` equal to `tetragon`:
```shell
sudo bpftool map -j | jq '[ .[] | select(.pids[0].comm == "tetragon") | .bytes_memlock ] | add / 1000'
```

Group `bytes_memlock` with name and sort by `bytes_memlock`:
```shell
sudo bpftool map -j | jq '[ .[] | {bytes_memlock:.bytes_memlock, name:.name}  ] | sort_by(.bytes_memlock)'
```

Sum memory by map name:
```shell
sudo bpftool map -j | jq ' group_by(.name) | map({name: .[0].name, total_bytes_memlock: map(.bytes_memlock | tonumber) | add, maps: length}) | sort_by(.total_bytes_memlock)'
```

#### Visualize the stats with pie charts

If you want to visualise :eyes: that last command in a pie chart :pie:, you can
try [`bpfmemapie`](https://github.com/mtardy/bpfmemapie), a little Go utility
to render an interactive pie chart out of bpftool's output.

{{<
    figure
    align=center
    src="./bpfmemapie.png"
    caption="An example of pie chart with Cilium Tetragon running"
>}}


## Control groups

### Find out if I'm using cgroups v1 or v2

To start, it's important to know if you are using cgroups version 1 or version 2.
Here are a few techniques to quickly detect the cgroups version.

- From [this unix Stack Exchange answer](https://unix.stackexchange.com/a/619682):
  ```sh
  mount | grep '^cgroup' | awk '{print $1}' | uniq
  ```
  If the output contains cgroup2, then your kernel supports cgroups v2.

- From the [Kubernetes cgroups concepts
  documentation](https://kubernetes.io/docs/concepts/architecture/cgroups/#check-cgroup-version)
  ```sh
  stat -fc %T /sys/fs/cgroup/
  ```
  - For cgroup v2, the output is `cgroup2fs`.
  - For cgroup v1, the output is `tmpfs`.

- From [runc documentation](https://github.com/opencontainers/runc/blob/main/docs/cgroup-v2.md#am-i-using-cgroup-v2):
  > *Your are using cgroups v2* if `/sys/fs/cgroup/cgroup.controllers` is present.

- A more [bullet-proof method used by systemd](https://unix.stackexchange.com/a/617824)
  > - if `/sys/fs/cgroup` exists and is on a `cgroup2` file system, the system is
  > running with a full unified hierarchy;
  > - if `/sys/fs/cgroup` exists and is on a `tmpfs` file system,
  >   - if either `/sys/fs/cgroup/unified` or `/sys/fs/cgroup/systemd` exist and
  >     are on `cgroup2` file systems, the system is using a unified hierarchy
  >     for the systemd controller only;
  >   - if `/sys/fs/cgroup/systemd` exists and is on a `cgroup` file system (or, as
  >     a fallback, if it exists and isn’t on a `cgroup2` file system), the
  >     system is using a legacy hierarchy.

  Note that this is a bit similar to what cAdvisor is doing when [retrieving
  values from the stat file]({{< ref "#stats-file" >}}).

### The memory control group

*This section is a digest of [Out-of-memory (OOM) in Kubernetes – Part 2: The
OOM killer and application runtime implications - Cgroups](https://mihai-albert.com/2022/02/13/out-of-memory-oom-in-kubernetes-part-2-the-oom-killer-and-application-runtime-implications/#cgroups)
by Mihai Albert.*

*Note that this section is written with cgroups v1 in mind*

Cgroups are a mechanism used on Linux for limiting and accounting resources
and containers are built on cgroups (and namespaces), see a
[container terminology introduction post by Red Hat](https://developers.redhat.com/blog/2018/02/22/container-terminology-practical-introduction)
on that. However, contrary to namespaces, cgroups don’t limit what a process
can "see", see this article on [why top and free inside containers don’t show
the container memory values](https://ops.tips/blog/why-top-inside-container-wrong-memory/).
Indeed, `/proc/meminfo` is not namespaced, contrary to PID list for example.

You can use the root memory cgroup accounting to retrieve the stat of the node
memory usage: that’s what Kubernetes does on cgroups v1. The value is correct if
`memory.use_hierarchy` is enabled for the root cgroup (and it cannot be
modified dynamically). You can check with:
```shell
find /sys/fs/cgroup/memory -name memory.use_hierarchy -exec cat '{}' \;
```

To verify that all processes appearing in `/proc` are also in the root memory
cgroup `/sys/fs/cgroup/memory` you can use:
```shell
ps aux | wc -l
find /sys/fs/cgroup/memory -name cgroup.procs -exec cat '{}' \; | wc -l
```

For more details read detailed [documentation on memory cgroup v1 by Red Hat](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/6/html/resource_management_guide/sec-memory#sec-memory).

Read an interesting [recap about using cgroups from LinkedIn engineering blog](https://engineering.linkedin.com/content/engineering/en-us/blog/2016/08/don_t-let-linux-control-groups-uncontrolled).
It emphasizes that other cgroups (or just the root cgroup) can be noisy
neighbors and affect the performance of an application running in its own
cgroup, highlighting again that they are not an isolation but a resource
limitation mechanism.

### Using cgroups to measure memory usage

Looking at RSS to measure memory consumption can be insufficient. Indeed, the
Linux mechanisms to measure and restrict memory consumption (cgroups) use
different accounting.

Note that the process must start in the memory cgroup otherwise the accounting
is incorrect because all the memory usage will be accounted for in the previous
cgroup. From the [administration guide of the Linux kernel](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#memory-ownership):
> A memory area is charged to the cgroup which instantiates it and stays
> charged to the cgroup until the area is released. Migrating a process to a
> different cgroup doesn’t move the memory usages that it instantiated while in
> the previous cgroup to the new cgroup.

In one terminal, open a new shell and echo its PID
```shell
bash
echo $$
```

In another terminal, create the cgroup and move the process inside of it
```shell
sudo su
cd /sys/fs/cgroup   # for cgroup v1, create under /sys/fs/cgroup/memory
mkdir benchmark     # this is an arbitrary name
cd benchmark
echo $(pidof <process>) > cgroup.procs
```

Then start your process using the bash you opened in the first terminal and
read the stat you want to acquire, for example:
```shell
cat memory.current   # for cgroup v1, equivalent can be memory.total_in_bytes
```

From the [cgroup v2 documentation](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#memory):

> The memory.current value is the sum of memory.stat first three lines (anon + file + kernel)
> * **anon**: Amount of memory used in anonymous mappings such as brk(),
>   sbrk(), and mmap(MAP_ANONYMOUS)
> * **file**: Amount of memory used to cache filesystem data, including tmpfs
>   and shared memory.
> * **kernel (npn)**: Amount of total kernel memory, including (kernel_stack,
>   pagetables, percpu, vmalloc, slab) in addition to other kernel memory use
>   cases.


## Kubernetes

### Kubernetes containers, Pods, and cgroups

*This section is a digest of [Out-of-memory (OOM) in Kubernetes – Part 2: The
OOM killer and application runtime implications - Cgroups and Kubernetes](https://mihai-albert.com/2022/02/13/out-of-memory-oom-in-kubernetes-part-2-the-oom-killer-and-application-runtime-implications/#cgroups-and-kubernetes)
by Mihai Albert.*

Kubernetes uses one cgroup per container but shares the hierarchy in a Pod, see
more details in this [Stack Overflow question](https://stackoverflow.com/questions/62716970/are-the-container-in-a-kubernetes-pod-part-of-same-cgroup).

Be aware of the pause container used in Pods, it’s a container used
to reap zombie processes and hold shared namespaces, read the best article you
can find on the topic: [The Almighty Pause Container](https://www.ianlewis.org/en/almighty-pause-container).
Also, read this conversation I had on the Kubernetes Slack to find out [why
crictl does not list the pause container](https://kubernetes.slack.com/archives/C019LFTGNQ3/p1651848044143799).

You can use crictl to inspect the memory controller hierarchy and statistics read more on the [official Kubernetes documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/#verify-pod-cgroup-limits).
Be aware that [containerd uses the k8s.io "namespace" for
containers](https://github.com/containerd/containerd/issues/1815#issuecomment-347389634).
You can also directly check cAdvisor’s output for stats.

Kubernetes supports cgroups v2. You can check what you are using through
different methods (see related note Find out which version of cgroup is
running). Again this article is written with cgroups v1 in mind and things were
fixed and changed with cgroups v2, like group killing for containers, see [this
presentation by Giuseppe Scrivano from Red Hat](https://www.youtube.com/watch?v=u8h0e84HxcE)
at 14:45 and 17:55.

### Kubernetes, applications, and the OOM killer

*This section is a digest of [Out-of-memory (OOM) in Kubernetes – Part 2: The
OOM killer and application runtime implications - Cgroups and the OOM killer](https://mihai-albert.com/2022/02/13/out-of-memory-oom-in-kubernetes-part-2-the-oom-killer-and-application-runtime-implications/#cgroups-and-the-oom-killer)
by Mihai Albert.*

OOM killer will step in when the whole OS is low on memory, but it has been
updated to work with cgroups as well, see [Teaching the OOM killer about control
groups](https://lwn.net/Articles/761118/). As explained in [the kernel
documentation](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt),
when a cgroup goes over limits, it first tries to reclaim memory (from the
per-group LRU list) and then invokes the OOM killer. From experimenting, it’s a
bit hard to predict which process will be targeted by the OOM killer inside a
container and the result can look incoherent from the desired behavior. Reading
kernel logs (using dmesg for example) will give you more context on the
decision (see this resource on [various system logs](https://askubuntu.com/questions/26237/difference-between-var-log-messages-var-log-syslog-and-var-log-kern-log)).

> […] the OOM killer deciding to terminate processes inside cgroups is
> something that “happens” to Kubernetes containers, as the OOM killer is a
> Linux kernel component. It’s not that Kubernetes decides to invoke the OOM
> killer, as it has no such control over the kernel.

As such, applications cannot be given a signal to gracefully shutdown and this
is a problem for running production code on Kubernetes, see [this GitHub issue
about making the OOM killer not send a SIGKILL](https://github.com/kubernetes/kubernetes/issues/40157).
Another aspect is that the OOM killer has no notion of containers (like the
rest of the kernel) nor Kubernetes Pods, so it can kill a process in a
container without stopping everything else: this can lead to weird states for
containers or Pods, unfortunately, this is ["working as intended"](https://github.com/kubernetes/kubernetes/issues/50632).
However, when Kubernetes sees an OOM kill event, it tries to restart the Pod to
make him healthy given the `restartPolicy`. Finally, memory cgroup has soft
limits but Kubernetes does not support them as of now, see this gist about
[Kubernetes resource management: kcgroups](https://gist.github.com/mcastelino/b8ce9a70b00ee56036dadd70ded53e9f#memory-resource-management).

Now the application can use a runtime that will allocate memory in complex
ways, and it can be difficult to understand why memory is retained and not
released to the OS. The original article addresses .NET, I’m much more
interested in Go and the best resource you can find is the [Guide to the Go
Garbage Collector](https://tip.golang.org/doc/gc-guide). Amongst other things,
since 1.19, we can use `GOMEMLIMIT` which is very useful to hint the runtime
toward the limitation and make him smarter.

Finally, Kubernetes has resource requests and limits, see the [Resource
Management for Pods and Containers for full official documentation](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/),
but essentially:
> When you specify the resource request for containers in a Pod, the
> kube-scheduler uses this information to decide which node to place the Pod
> on. When you specify a resource limit for a container, the kubelet enforces
> those limits so that the running container is not allowed to use more of that
> resource than the limit you set.

Here are other resources on resource requests and limits:
- [Google's Kubernetes best practices: Resource requests and
  limits](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-resource-requests-and-limits)
- [A Deep Dive into Kubernetes Metrics - Part 3 Container Resource
  Metrics](https://blog.freshtracks.io/a-deep-dive-into-kubernetes-metrics-part-3-container-resource-metrics-361c5ee46e66)

Kubernetes uses Quality of Service (QoS) classes for Pods to dictate the order
in which pods are evicted and define the root memory cgroup hierarchy on the
node. [Official documentation here](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod),
but a recap could be:
> Guaranteed is assigned to pods that have all of their containers specify
> resource values that are equal to the limits ones for both CPU and memory
> respectively. Burstable is when at least one container in the pod has a CPU
> or memory request, but it doesn’t meet the “high” criteria for Guaranteed.
> The last of the classes – BestEffort – is the case of a pod that doesn’t have
> a single container specify at least one CPU or memory limit or request value.

### Kubernetes memory metrics

*This section is rewritten but inspired by [Out-of-memory (OOM) in Kubernetes –
Part 3: Memory metrics sources and tools to collect them](https://mihai-albert.com/2022/02/13/out-of-memory-oom-in-kubernetes-part-3-memory-metrics-sources-and-tools-to-collect-them/)
by Mihai Albert.*

In the Kubernetes world, people looking at memory metrics for workload usually
look at the `container_memory_working_set_bytes`, let's understand the link
between this stat and the stats, we retrieve from the OS and explain it.

#### Metrics components

{{<
    figure
    align=center
    src="./kubernetes_metrics_diagram.drawio.png"
    link="https://mihai-albert.com/wp-content/uploads/2022/02/kubernetes_metrics_diagram.drawio.png"
    caption="Metrics components diagram (Kubernetes 1.21 cluster with containerd, as of Feb 2022) by Mihai Albert"
>}}

The above diagram is from the author of the original blog post and gives a good
understanding of the flow of metrics on a node. Since we care about the memory use of
the workloads, the most interesting part is what is happening inside kubelet,
which will be the server exposing various metrics. Let's simplify it by
ignoring a few things:
- The Kubelet's own metrics endpoint.
- The kube-state-metrics, a service that listens to the Kubernetes API server
  and generates metrics. Note that it only relies only on the Kubernetes API
  events.
- The Kubernetes API Server's own metrics endpoint.
- The Prometheus node exporter can also skipped for now as it concerns the
  whole node and not the workloads, reporting various from the host itself
  (which contains memory stats though).

```goat
                    +--------------------------------------------------------------+
                    |                                                              |
                    |   +--------------+                                           |
                    |   |  Prometheus  |   +-----------------------------------+   |
        +--------------->     node     |   |                                   |   |
        |           |   |   exporter   |   |    +--------------+               |   |
+-------|------+    |   +--------------+   |    |              |               |   |
|              |    |                      |    |   cAdvisor   |               |   |
|  Prometheus  --------------------------------->   endpoint   |               |   |
|    server    |    |                      |    |              |               |   |
|              |    |                      |    +-------^------+               |   |
+-------^------+    |                      |            |   /metrics/cadvisor  |   |
        |           |                      |            |                      |   |
        |           |   +--------------+   |    +-------|------+               |   |
        |           |   |              |   |    |              |               |   |
                    |   |  Container   |   |    |   Summary    |               |   |
     Grafana        |   |   runtime    <---------     API      |               |   |
                    |   |              |   |    |   endpoint   |               |   |
                    |   +--------------+   |    |              |               |   |
                    |    a bug still       |    +-------^------+               |   |
                    |    prevent this      |            |   /stats/summary     |   |
                    |    link              |            |                      |   |
+--------------+    |                      |    +-------|------+               |   |
|              |    |                      |    |              |               |   |
|    Metrics   |    |                      |    |   Resource   |               |   |
|    server    -------------------------------->|   Metrics    |               |   |
|              |    |                      |    |   endpoint   |               |   |
+-------^------+    |                      |    |              |               |   |
        |           |                      |    +--------------+               |   |
        |           |                      |                /metrics/resource  |   |
        |           |                      |                                   |   |
                    |                      |                           Kubelet |   |
   kubectl top      |                      +-----------------------------------+   |
                    |                                                              |
                    |                                                         Node |
                    +--------------------------------------------------------------+
```

With this simplified version, we realize that:
- We end up with two principal consumers, first the Prometheus Server will
  query the cAdvisor endpoint directly and the Prometheus node exporter, and
  then the Metrics server will query all nodes to return the data exposed
  by `kubectl top pod|node`.
- The stats of the workloads exposed by the kubelet, are *mostly* coming from
  cAdvisor.

Indeed, on the last topic, you can see that in the Metrics server case, it
queries the Resource Metrics endpoint, which interrogates the Summary API
endpoint, that retrieves its metrics both from cAdvisor and the Container
runtime (CRI). Note that the current evolution is to [rely less on cAdvisor and more on the CRI](https://github.com/haircommander/enhancements-1/blob/0fa93d0d3e00356a49767b517ea9c34e18c280f9/keps/sig-node/2371-cri-pod-container-stats/README.md#summary)
for stats. For that, the way the kubelet "decides" which source to use
(cAdvisor or CRI) is detailed in
"[How does the Summary API endpoint get its metrics?](https://mihai-albert.com/2022/02/13/out-of-memory-oom-in-kubernetes-part-3-memory-metrics-sources-and-tools-to-collect-them/#how-does-the-summary-api-endpoint-get-its-metrics)".
Nowadays, the reality is that, in any case, most metrics come from cAdvisor
[due to a bug in kubelet](https://github.com/kubernetes/kubernetes/issues/107172).

In practice, if you are using the [runc](https://github.com/opencontainers/runc)
low-level runtime[^1], which is the reference implementation, used in most cases by
the popular high-level container runtimes [containerd](https://containerd.io/)
and [CRI-O](https://cri-o.io/), whether kubelet would retrieve its cgroups
stats from the CRI or cAdvisor would not make a difference as they both use
[opencontainers/libcontainer](https://github.com/opencontainers/runc/tree/main/libcontainer):
> Libcontainer provides a native Go implementation for creating containers with
> namespaces, cgroups, capabilities, and filesystem access controls. It allows
> you to manage the lifecycle of the container performing additional operations
> after the container is created.

However, this is cAdvisor or the CRI directly that will compute heuristic
returned like `memory_working_set`, so the appropriate code must be explored.

[^1]: You might not use runc if you are using [crun](https://github.com/containers/crun)
a C language container runtime implementation, or an isolation low-level
runtimes like AWS [firecracker-containerd](https://github.com/firecracker-microvm/firecracker-containerd)
or Google [gVisor](https://gvisor.dev/). See [this
article](https://www.tutorialworks.com/difference-docker-containerd-runc-crio-oci/)
on how to understand the differences between Docker, containerd, CRI-O and runc
if the notion of container runtime is unclear.

#### cAdvisor and libcontainer deep dive

So in the end, cAdvisor will call its
[`GetStats`](https://github.com/google/cadvisor/blob/v0.49.1/container/libcontainer/handler.go#L74)
method, that will, for every container, eventually call the
[`setMemoryStats`](https://github.com/google/cadvisor/blob/v0.49.1/container/libcontainer/handler.go#L801)
function. The most interesting part for us is the function's first line,
that retrieves memory usage, and the last lines, which computes the working set
stats we are trying to understand.

```go {hl_lines=[2,31,34,36,39]}
func setMemoryStats(s *cgroups.Stats, ret *info.ContainerStats) {
	ret.Memory.Usage = s.MemoryStats.Usage.Usage
	ret.Memory.MaxUsage = s.MemoryStats.Usage.MaxUsage
	ret.Memory.Failcnt = s.MemoryStats.Usage.Failcnt
	ret.Memory.KernelUsage = s.MemoryStats.KernelUsage.Usage

	if cgroups.IsCgroup2UnifiedMode() {
		ret.Memory.Cache = s.MemoryStats.Stats["file"]
		ret.Memory.RSS = s.MemoryStats.Stats["anon"]
		ret.Memory.Swap = s.MemoryStats.SwapUsage.Usage - s.MemoryStats.Usage.Usage
		ret.Memory.MappedFile = s.MemoryStats.Stats["file_mapped"]
	} else if s.MemoryStats.UseHierarchy {
		ret.Memory.Cache = s.MemoryStats.Stats["total_cache"]
		ret.Memory.RSS = s.MemoryStats.Stats["total_rss"]
		ret.Memory.Swap = s.MemoryStats.Stats["total_swap"]
		ret.Memory.MappedFile = s.MemoryStats.Stats["total_mapped_file"]
	} else {
		ret.Memory.Cache = s.MemoryStats.Stats["cache"]
		ret.Memory.RSS = s.MemoryStats.Stats["rss"]
		ret.Memory.Swap = s.MemoryStats.Stats["swap"]
		ret.Memory.MappedFile = s.MemoryStats.Stats["mapped_file"]
	}

    // [...]

    inactiveFileKeyName := "total_inactive_file"
	if cgroups.IsCgroup2UnifiedMode() {
		inactiveFileKeyName = "inactive_file"
	}

	workingSet := ret.Memory.Usage
	if v, ok := s.MemoryStats.Stats[inactiveFileKeyName]; ok {
		if workingSet < v {
			workingSet = 0
		} else {
			workingSet -= v
		}
	}
	ret.Memory.WorkingSet = workingSet
}
```

##### Memory usage link to cgroups fs

To retrieve memory usage, libcontainer reads `memory.usage_in_bytes` for
[cgroups v1](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs/memory.go#L205),
and `memory.current` for [cgroups v2](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs2/memory.go#L132).
If interested (or looking for proofs) see the code deep dive below
:ocean: :diving_mask: :squid:.


<!-- <details> -->
  <!-- <summary>Click me to expand the code deep dive!</summary> -->

<!-- <big>:ocean: :diving_mask: :squid:</big> -->

The [`setMemoryStats`](https://github.com/google/cadvisor/blob/v0.49.1/container/libcontainer/handler.go#L801)
function was called from
[`newContainerStats`](https://github.com/google/cadvisor/blob/v0.49.1/container/libcontainer/handler.go#L911)
in the
[`GetStats`](https://github.com/google/cadvisor/blob/v0.49.1/container/libcontainer/handler.go#L74)
method specified above, which [called
`h.cgroupManager.GetStats()`](https://github.com/google/cadvisor/blob/v0.49.1/container/libcontainer/handler.go#L85)
to retrieve the cgroup stats, using the `opencontainers/libcontainer` codebase.
Note that naming is very confusing here since cAdvisor has a libcontainer
package that contains the
[`GetStats`](https://github.com/google/cadvisor/blob/v0.49.1/container/libcontainer/handler.go#L74)
method that will call the libcontainer's cgroup package
[`GetStats`](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/cgroups.go#L21)
method.

Now, as expected, `libcontainer` has two implementations for its `GetStats`
method, one in the [`fs`](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/systemd/v1.go#L316)
and the other in the [`fs2`](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs2/fs2.go#L95) package, corresponding to cgroups v1 and v2
respectively.

1. For cgroups v1,
   [another `GetStats` is called](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/systemd/v1.go#L325)
   on all subsystems. For memory, we end up in the appropriate
   [`GetStats`](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs/memory.go#L142)
   method that
   [writes to `stats.MemoryStats.Usage`](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs/memory.go#L167)
   after calling [`getMemoryData(path, "")`](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs/memory.go#L163).
   So we can finally see that libcontainer reads `memory.usage_in_bytes` in
   [`getMemoryData`](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs/memory.go#L205)
   for memory usage.

   ```go {hl_lines=[4,9,15,24]}
   func getMemoryData(path, name string) (cgroups.MemoryData, error) {
       memoryData := cgroups.MemoryData{}

       moduleName := "memory"
       if name != "" {
           moduleName = "memory." + name
       }
       var (
           usage    = moduleName + ".usage_in_bytes"
           maxUsage = moduleName + ".max_usage_in_bytes"
           failcnt  = moduleName + ".failcnt"
           limit    = moduleName + ".limit_in_bytes"
       )

       value, err := fscommon.GetCgroupParamUint(path, usage)
       if err != nil {
           if name != "" && os.IsNotExist(err) {
               // Ignore ENOENT as swap and kmem controllers
               // are optional in the kernel.
               return cgroups.MemoryData{}, nil
           }
           return cgroups.MemoryData{}, err
       }
       memoryData.Usage = value
       // [...]
   }
   ```

2. For cgroups v2, the
   [`statsMemory`](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs2/memory.go#L76)
   function [is called](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs2/fs2.go#L105),
   which similarly as with v1, [writes to
   `stats.MemoryStats.Usage`](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs2/memory.go#L110)
   after calling [`getMemoryDataV2(dirpath, "")`](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs2/memory.go#L100).
   We can finally see that libcontainer reads `memory.current` in
   [`getMemoryDataV2`](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs2/memory.go#L132)
   for memory usage.

   ```go {hl_lines=[4,8,12,22]}
   func getMemoryDataV2(path, name string) (cgroups.MemoryData, error) {
       memoryData := cgroups.MemoryData{}

       moduleName := "memory"
       if name != "" {
           moduleName = "memory." + name
       }
       usage := moduleName + ".current"
       limit := moduleName + ".max"
       maxUsage := moduleName + ".peak"

       value, err := fscommon.GetCgroupParamUint(path, usage)
       if err != nil {
           if name != "" && os.IsNotExist(err) {
               // Ignore EEXIST as there's no swap accounting
               // if kernel CONFIG_MEMCG_SWAP is not set or
               // swapaccount=0 kernel boot parameter is given.
               return cgroups.MemoryData{}, nil
           }
           return cgroups.MemoryData{}, err
       }
       memoryData.Usage = value
       // [...]
    }
    ```

<!-- </details> -->

##### Stats file

Doing a similar analysis you can find that out that `s.MemoryStats.Stats` is
a map containing the key-values of the `memory.stat` file for
[cgroups v1](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs/memory.go#L143-L160)
and for [cgroups v2](https://github.com/opencontainers/runc/blob/v1.1.12/libcontainer/cgroups/fs2/memory.go#L77-L91)

The key names differ between cgroups v1 and v2, explaining why the
`setMemoryStats` function needs to discriminate using
`cgroups.IsCgroup2UnifiedMode()` and `s.MemoryStats.UseHierarchy`.

##### Memory working set

The conclusion is that Kubernetes' `container_memory_working_set_bytes` is a
heuristic trying to estimate what memory is used by the container's processes
by reading the usage and subtracting the memory used by the "inactive files".

From the [original article](https://mihai-albert.com/2022/02/13/out-of-memory-oom-in-kubernetes-part-3-memory-metrics-sources-and-tools-to-collect-them/#how-does-cadvisor-get-its-memory-metric-data):
> `inactive_file` is defined in the [docs section
> 5.2](https://www.kernel.org/doc/Documentation/admin-guide/cgroup-v1/memory.rst)
> as “number of bytes of file-backed memory on inactive LRU list“. The LRU
> lists are described in [this document](https://www.kernel.org/doc/gorman/html/understand/understand013.html)
> with the inactive list described as containing “reclaim candidates” as
> opposed to the active list that “contains all the working sets in the
> system“.
>
> So in effect the memory size for mapping files from disk that aren’t really
> required at the time is deducted from the memory usage – itself roughly
> equivalent as we’ve seen previously with rss+cache+swap (which we’re already
> reading from memory.stat).


To simplify, while the metrics always remain >= 0, we can link the
`container_memory_working_set_bytes` with the cgroups fs metrics like this:

| Cgroups   | Container Memory Working Set Bytes                           |
| -         | -                                                            |
| version 1 | `memory.usage_in_bytes` - `memory.stat[total_inactive_file]` |
| version 2 | `memory.current` - `memory.stat[inactive_file]`              |


## Conclusion

Let's recap and try to summarize the essence of the research that was compiled
in the above sections.

Let's say you have a Go application, optionally loading eBPF programs, running on
Kubernetes having memory issues. By memory issue, I mean that `kubectl top`, or
the metrics monitored on Grafana, the `container_memory_working_set_bytes`, are
"too high" to your taste. How to understand and trace the link between your
program memory usage and the final result reported by Kubernetes?

The `container_memory_working_set_bytes` is a heuristic memory metric computed
in the container world trying to estimate what the OOM killer will look for
when checking the process memory consumption. It does not exist as-is from the
cgroups statistics.

This value can technically come out of [cAdvisor](https://github.com/google/cadvisor),
embedded in the Kubelet or the container runtime. However, at the moment, it
mostly comes out of cAdvisor anyway because of a [Kubelet
bug](https://github.com/kubernetes/kubernetes/issues/107172). While the
computation of this high-level metric is done by cAdvisor or the container
runtime, if you are using [runc](https://github.com/opencontainers/runc)
as your low level container runtime[^1], the cgroup statistics will be retrieved using
[runc libcontainer](https://github.com/opencontainers/runc/tree/main/libcontainer)
reference implementation.

Depending on [whether you are using cgroups v1 or v2]({{< ref
"#find-out-if-im-using-cgroups-v1-or-v2" >}}), the following table shows which
cgroup stats will be used to compute `container_memory_working_set_bytes`. Also
keep in mind that this metrics is always >= 0, so it's technically `max(0,
<value below>)`.

| Cgroups   | Container Memory Working Set Bytes                           |
| -         | -                                                            |
| version 1 | `memory.usage_in_bytes` - `memory.stat[total_inactive_file]` |
| version 2 | `memory.current` - `memory.stat[inactive_file]`              |

So if the amount of `inactive_file` memory is low,  you can approximate this
value to the memory cgroup main statistic, `memory.usage_in_bytes` or
`memory.current` for cgroup v1 and v2 respectively. For details about how we
ended up on this approximation, see [the diagram and code deep dive above]({{< ref
"#kubernetes-memory-metrics" >}}).

Now to continue your investigation you might want to:
1. [Read the Linux process memory use and memory segments]({{< ref "#reading-a-linux-process-memory-use" >}}).
   Depending on the way your process handles memory, it can give you a first
   view on how memory is used.
1. [Reproduce the measurements locally using cgroups]({{< ref "#using-cgroups-to-measure-memory-usage" >}}).
   It can be easier than spinning a whole Kubernetes cluster and deploying your
   fixed program.
1. [Check if your Go program uses too much heap]({{< ref "#understand-and-reduce-go-heap-use" >}}).
   Using a language-specific profiler will give you information about how the
   heap is distributed. Don't forget to [learn about `GOMEMLIMIT`]({{< ref
   "#the-go-memory-limit" >}}) for containerized environment and [other stuff
   about the Go garbage collector]({{< ref "#golang-memory-use-and-the-garbage-collector" >}})
   to optimize its behavior for your use case.
1. [Check if your eBPF maps use too much kernel memory]({{< ref "#ebpf-programs-memory-impact" >}}).
   When using cgroups v2, kernel memory used by BPF maps is accounted for properly,
   you can retrieve some stats using `bpftool` and `jq`, and plot some pie
   charts to visualize them.

If you find errors in this article, please [reach out](https://github.com/mtardy/blog).

