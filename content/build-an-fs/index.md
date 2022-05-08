+++
date = "2020-05-10T06:00:00-05:00"
title = "Let's build a high performance file system!"
tags = ["ocaml", "plan9", "9P", "fs", "performance", "programming"]
description = "In user space! In a high-level language!"
draft = true
+++

I enjoyed making a series of [articles](../9p) implementing a 9P file
system library in Go. It is not quite finished, and I do plan to revisit
it at some point. I did end up creating something functional, and built a
[toy](https://github.com/droyo/jsonfs) server with it.

In the same spirit as that effort, in this series I will build a "real" file
system. By "real", I mean this is a file system in the traditional sense; a
database for persisting hierarchically organized, named blobs of data (files)
to persistent storage. For this I will be using [ocaml](https://ocaml.org/),
which I am only just beginning to learn. This project will double as a
way to get acquainted with the language, which is quickly becoming one of
my favorites.

[OCaml](https://www.ocaml.org/) is a high-level, functional, garbage-collected
language with a world-class type system. It has been around for more than 20
years, yet it is remarkably modern and expressive. It is stereotypically known
as a language for writing compilers, but it is a general purpose language
that can produce native binaries that are competitive with the likes of Go,
Java, and even C/C++ in some cases.

# Defining "high performance"

Performance is not binary. What is considered "fast" depends on your intended
usage patterns and the hardware's capabilities. In the absence of caching, a
user can only read and write to a file system at the rate that the underlying
storage device can support. There are many types of storage devices, each with
different capabilities. For example, with spinning hard disks you can expect
read/write throughput in the low 100s of MB/s for sequential I/O. Modern SSDs
will have an order of magnitude higher throughput. Many modern hard drives
and SSDs often have built-in cache, sometimes battery-backed, that allows
for many small updates to be batched together into fewer, larger updates, or
even to allow for "bursts" of I/O beyond what the device can otherwise support.

Going even further, specialized persistent memory devices have read/write rates
of multiple GB/s.  Then there are the "disks" offered by cloud providers, which
are more like distributed databases hiding behind a block storage protocol with
their own unique latency, throughput, and concurrency profiles.  Given the
variation in hardware, it's incredible how well general purpose file systems
like XFS, ZFS, and ext4 are able to adapt and scale from a laptop with 1TB
of storage, to a server with 56 cores and 100TB or more of storage capacity.

At a high level, I want to produce a file system that is suitable for daily
use as a primary file system for a single-user workstation. Because that is a
vague goal, in practice I will be using a series of benchmarks approximating
real-world usage. The file system will be compared to XFS and ext4 on various
dimensions:

* **latency** - how long does it take for a single IO operation to be served?
* **throughput** - what is the maximum sustained bit rate that can be read
  from or written to the file system?
* **iops** - how many IO operations can be served per second? This is similar
  to throughput, but instead is a measure of the fixed per-request overhead.
* **efficiency** - how much system resources (CPU, memory) does the file system
  consume relative to how much work is done?

# The overhead of a user space file system

This file system will be exposed as a 9P file server. We will use this
series to evaluate the performance overhead of the Linux kernel's native
9P support, and explore potential improvements therein.

The common perception of user space file systems is that they are slow
and unreliable. I believe the perception of unreliability stems from a
proliferation of file systems that shoehorn a remote service into a local
file system API. There are some legitimate arguments about overhead. After
all, to write to a user space file system serving 9P on a Linux machine with
the kernel's `v9fs` module, a program follows this flow:

<img src="/img/ufs-simplified-write-path.svg" style="width:90%; max-width: 800px"/>

This diagram is drastically simplified; [here](/img/v9fs-2.0.jpg) is the actual
request flow. Every time the user space/kernel space barrier is crossed, the
data must be copied from user space memory to kernel memory. That's 3 copies
compared to a single copy when writing to a kernel space file system. How
could a user space file system ever be comparable with such a disadvantage?

To be honest, I don't know if it is a surmountable obstacle. My intuition
tells me that most consumer and enterprise hardware is not yet fast enough
for this extra overhead to become a limiting factor. I also plan to leverage
newer Linux kernel features that can cut down on the amount of copying
needed.

# Why?

I'm fully aware that there are dozens if not hundreds of existing production
file systems out there, with hundreds of thousands of engineering hours poured
into them. I will not presume to know better than they. First and foremost,
this is an attempt to learn about the design challenges and tradeoffs when
building file systems. In addition, this is an opportunity for me to learn
more about Ocaml, the Linux kernel, and performance analysis in general.

I also hope this undertaking demonstrates the feasibility of running file
systems in user space. File systems like ext4 and xfs are essentially
databases embedded into the Linux kernel. Their place in the kernel dictates
the language and coding style they use, what dependencies they can have, how
often they can release, and the severity of their defects.  With a user space
file system, you trade performance overhead for freedom from the constraints
of kernel development. And with 9P in particular, it is trivial to repurpose this
work to serve files over a network.

Finally, I also intend to use this file system. I am building a home server
that will run all significant workloads within virtual machines and provide
all of the storage for those machines via a 9P file server. To that end,
I will be undertaking reasonable measures to ensure that the file system is
safe, reliable, and fast.

This series is written as a build log rather than a review of a finished
project.  I've tried to capture my thought process, rather than a curated
tutorial. There will be dead ends, and I will come up with bad solutions,
and throw them out later, or rewrite them completely. I hope that my trials
prove entertaining, despite the meandering.

# Table of contents

* [Part 0: implementing 9P in ocaml](0-ocaml-9p)
* [Part 1: build literally anything](1-build-literally-anything)
* [Part 2: an in-memory file system](2-memfs)
* [Part 3: a look at v9fs](3-a-look-at-v9fs)
* [Part 4: overcoming the user-kernel barrier](4-overcoming-user-kernel)
* [Part 5: file system data structures](5-fs-data-structures)
* [Part 6: persisting data](6-persisting-data)
* [Part 7: journaling](7-fs-journaling)
* [Part 8: benchmarking and profiling redux](8-benchmarking-profiling-redux)

[1]: https://fio.readthedocs.io/en/latest/fio_doc.html
[2]: https://github.com/filebench/filebench
[3]: https://github.com/droyo/filebench-tests
