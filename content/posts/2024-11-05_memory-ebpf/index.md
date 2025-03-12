---
title: "eBPF programs memory impact"
date: "2024-11-05"
categories: ["memory"]
tags: ["memory", "ebpf"]
slug: "memory-ebpf"
---

BPF programs' memory impact is often mostly due to BPF maps. Maps serve as a
way to store state or more generally data, and communicate between BPF programs
and userspace programs. These can grow arbitrarily in size, while the program
is somewhat bounded.

BPF maps are statics and need to be defined at compilation, however, some of
their characteristics can be easily modified at loading time. Most of map types
also have to be pre-allocated on load, and this is today the default (see the
`BPF_F_NO_PREALLOC` flag), explaining why empty or unused maps use as much
space as used ones.

## How to measure the memory impact?

On recent kernels, the mechanism used to measure and limit the resource
consumptions are control groups. They exist in different flavors, the legacy v1
version and the new unified v2 version. Note that you can also run both
versions at the same time, using systemd hybrid mode.

What's interesting is that it seems that v1 memory control group does not
correctly account for memory used by maps or program, not even in the `kernel`
version of those stats. It means that, in the context of a container
orchestrator like Kubernetes, measure and limiting resource consumption using
cgroup underneath, you could get away with using a lot of "free" kernel memory.

However, it changed for cgroup v2 certainly related to [this series of
patches](https://lore.kernel.org/bpf/20201201215900.3569844-1-guro@fb.com/).
This can lead to a drastic change in overall memory consumption if you switch
from cgroup v1 to v2 while using a lot of BPF maps. So for the following, [make
sure you are using control group v2](/posts/memory-kubernetes-golang-ebpf/#find-out-if-im-using-cgroups-v1-or-v2).

Now if you want to measure your program kernel memory impact (including the BPF
memory impact), the most reliable way is to use a control group. If you are
doing it manually, make sure that the process starts in the memory cgroup you
are looking at otherwise the accounting might be incorrect, see this from
[administration guide of the Linux kernel](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#memory-ownership).

> A memory area is charged to the cgroup which instantiates it and stays
> charged to the cgroup until the area is released. Migrating a process to a
> different cgroup doesn’t move the memory usages that it instantiated while in
> the previous cgroup to the new cgroup.

For more conveniance, I wrote a small utility called
[`cmemstat`](https://github.com/mtardy/cmemstat) to automatically start a
process in its new control group and print regularly the memory stats. See more
information about how to download, compile and use on the project's repository
at [github.com/mtardy/cmemstat](https://github.com/mtardy/cmemstat).

## How to know which maps have the largest impact

Recent kernels expose the `memlock` stat on a BPF object, usually you can
retrieve it in the fdinfo file. Note that the `memlock` stat got actually
accurate from kernel v6.4 and onward thanks to a series of patches introducing
a callback to measure the memory from various map types.

## How to fix

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
    height=400px
    class=keep-aspect-ratio
>}}

