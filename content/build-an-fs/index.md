+++
date = "2020-05-10T06:00:00-05:00"
title = "Let's build a high performance file system!"
tags = ["ocaml", "plan9", "9P", "fs", "performance", "programming"]
description = "In user space! In a high-level language!"
draft = true
+++

I enjoyed making a series of [articles](../9p) implementing
a 9P filesystem library in Go. It is not quite finished,
and I do plan to revisit it at some point in the future. I
did end up creating something functional, and built a
[toy](https://github.com/droyo/jsonfs) server with it. It
has been a few years since then and I have been reflecting on
some of the lessons learned since then, along with the general
concept of system performance.

In the same spirit as that effort, this series of
articles will walk through the implementation of a high
performance user space file system. For this I will be using
[ocaml](https://ocaml.org/), which I am only just beginning to
learn. This project will double as a way to get acquainted with
the language and its ecosystem.

OCaml is a high-level, functional, garbage-collected
language. It does not have a strong concurrency story; a
global lock prevents multiple threads from running at the
same time, and most open source OCaml software uses a green
threads library like [Lwt](https://ocsigen.org/lwt/) or [Async](
https://opensource.janestreet.com/async/) to provide concurrency
through cooperative multitasking on a single thread.

# Defining "high performance"

Performance is not binary. What is considered "fast" depends on
your intended usage pattern and the hardware capabilities. In
the absence of caching, a user can only read and write to a
file system at the rate that the underlying storage device
can support. There are many types of storage devices, with
different capabilities. For example, with spinning hard disks
you can expect read/write speeds in the low 100s of MB/s, and
solid state drives can do many times that. Going even further,
specialized persistent memory devices have read/write rates of
multiple GB (that's a capital B) per second.  Then there are
the "disks" offered by cloud providers, which are more like
distributed databases hiding behind a block storage facade.

It's more than just throughput, different devices will have
different characteristics. Some devices will handle sequential
IOs better than random IO. Others may be faster when IOs are
spread across multiple "sectors", or when they are concentrated
in a single "sector". Given the variation in hardware, it's incredible
how well general purpose file systems like XFS, ZFS, and ext4 are
able to adapt and scale.

To achieve something meaningful, we have to set a measurable
goal. To benchmark this file system, we will be using a set of
[filebench][2] tests curated [here][3]. The benchmarks are intended
to mimic high throughput, latency sensitive, and mixed workloads.
As a final, practical test, we will compile the Linux kernel.

I'll be comparing the file system against the following:

* Linux ext4
* Linux xfs
* Linux tmpfs

Most of the benchmarks will only be relevant in relation to each
other. I will be using the hardware available to me, which is a Dell
XPS 13, with a Core i7-6560U CPU and a 1TB Samsung PM951 NVMe
SSD. To complicate things further, I will be running these benchmarks
in a Hyper-V VM. As the project matures I will look for other hardware.

This file system will be exposed as a 9P file server, and accessed via
the Linux kernel's native 9P support.

# The overhead of a user space file system

The common perception of user space file systems is that they are
slow. After all, to write to a user space file system on a Linux machine,
a program follows this flow:

<img src="/img/ufs-simplified-write-path.svg" style="width:100%"/>

This diagram is drastically simplified; [here](/img/v9fs-2.0.jpg)
is the actual request flow. Every time the user space/kernel
space barrier is crossed, the data have to be copied from user
space memory to kernel memory. That's 3 copies compared to a
single copy when writing to a kernel space file system. How could
a user space file system be comparable with such a disadvantage?

First, for most storage devices, the device is going to be a bottleneck
before user space → kernel space transfer becomes a bottleneck.
So the increased copying will cause increased latency and CPU usage,
but should not affect throughput.

In addition, there are some tricks we can do to reduce the amount
of copying needed for large requests. I do not intend to achieve a
fully zero-copy file system as it would require some fairly drastic
changes.

# Why?

I'm fully aware that there are dozens if not hundreds of existing
file systems out there. First and foremost, this is an attempt
to learn about the design challenges and tradeoffs when building
file systems. In addition, this is an opportunity for me to
learn OCaml and become more acquainted with its performance
analysis tools.

I also hope this undertaking demonstrates the viability of running
file systems in user space. File systems like ext4 and xfs are essentially
databases embedded into the Linux kernel. Their place in the kernel
dictates the language and coding style they use, what dependencies
they can have, and how often they can release. In addition, a flaw in
the implementation could crash the entire system. When your file
system is just another process, you have none of these constraints,
and if your file system crashes, your system can continue to run. This
project will explore the trade offs.

Finally, I also intend to use this file system. I am working on building
a home server that will run all significant workloads within virtual
machines and provide most of the storage for those machines via a
9P file server. To that end, I will be undertaking reasonable measures
to ensure that the file system is reliable, and that it doesn't consume
too many resources.

# Table of contents

* [Part 0: implementing 9P in ocaml](ocaml-9p)
* [Part 1: build literally anything](1-build-literally-anything)
* [Part 2: benchmarking and profiling](2-benchmarking-profiling)
* [Part 3: a look at v9fs](3-a-look-at-v9fs)
* [Part 4: overcoming the user-kernel barrier](4-overcoming-user-kernel)
* [Part 5: file system data structures](5-fs-data-structures)
* [Part 6: persisting data](6-persisting-data)
* [Part 7: journaling](7-fs-journaling)
* [Part 8: benchmarking and profiling redux](8-benchmarking-profiling-redux)

[1]: https://fio.readthedocs.io/en/latest/fio_doc.html
[2]: https://github.com/filebench/filebench
[3]: https://github.com/droyo/filebench-tests
