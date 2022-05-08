+++
date = "2020-05-10T06:00:00Z"
title = "File system performance benchmarks"
description = "Docs referenced while building a file system"
draft = true
+++

* [The ZUFS zero-copy filesystem](https://lwn.net/Articles/756625/). This is an alternative kernel API to FUSE. Rather than running as a user space process, file systems are implemented as shared libraries that are loaded into a "ZUS" application server. The kerne component and ZUS share memory to communicate, avoiding copies.

[File System and Storage Benchmarking Tools and Techniques](https://www.fsl.cs.sunysb.edu/project-fsbench.html) has a nice collection of papers on file system benchmarking, including a few FUSE-specific papers.

* [Grave Robbers from Outer Space: Using 9P2000 Under Linux](https://www.usenix.org/legacy/events/usenix05/tech/freenix/hensbergen.html) Original paper introducing v9fs into the 2.6 Linux kernel.

* [Overview of the Linux Virtual File System](https://www.kernel.org/doc/html/latest/filesystems/vfs.html)

* [errors.c](https://github.com/torvalds/linux/blob/master/net/9p/error.c), v9fs's internal table mapping plan9-style string errors to error numbers.

* [Composable Error Handling in OCaml](https://keleshev.com/composable-error-handling-in-ocaml) - survey of error handling techniques in OCaml.

[minix vfs - Minix3](https://www.minix3.org/theses/gerofi-minix-vfs.pdf)
