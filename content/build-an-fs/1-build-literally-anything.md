+++
date = "2020-05-11T18:00:00Z"
title = "Let's build a high performance file system! Part 1"
tags = ["ocaml", "plan9", "9P", "fs", "performance", "programming"]
description = "Build literally anything"
draft = true
+++

This is part 1 of a [multi-part series](./index) to build a high performance
user space file system in OCaml.

## Build literally anything

The objective of this article is to get something (*anything*) running as
quickly as possible, to see where my design choices fall apart. There will
be no attempt made to make it efficient, durable, or reliable. To start, we
won't even be writing to disk. We won't even support the full list of file
operations! The objective is to work out all the build issues and setup a
test harness to enable future development.

The [previous](/build-an-fs/0-ocaml-9p) post described the implementation of an OCaml
module for the 9P2000 protocol and defined a `Filesystem` interface that
an implementation must meet. The purpose of that module is to abstract
away the details of the protocol and allow my file system implementation
to focus on what matters: managing files.

Here we will create a simple file system. This file system will implement two
files: `zero` and `null`. These files are analagous to `/dev/null` and `/dev/zero`
on Unix systems:

* `zero` contains an infinite stream of zeroes.
* `null` contains nothing at all. Reads from `null` will never complete. Writes
   to `null` will always complete.

The file system will not support file creation or removal, and will not perform
any access checks. To setup a quick feedback loop, I create an empty file,
`example/nullFS.ml` and `example/dune` file with the following:

	(library
	 (name nullFS)
	 (modules nullFS)
	 (libraries styx))

And a new test, test/nullfs_test.ml:

	(test
	  (name nullfs_test)
	  (modules nullfs_test)
	  (libraries styx unix nullfs))

with a test that just runs a `nullFS` server on a unix socket, `nullfs.sock`:

	open Printf

	let rec serve fs fd =
	  let c, _ = Unix.accept fd in

	  match Srv.run ~fs ~w:c ~r:c with
	  | Ok _ -> serve fs fd
	  | exception End_of_file -> eprintf "connection closed\n%!"
	  | Error msg -> eprintf "Error: %s%!" msg; serve fs fd

	let () =
	  let fs = Nullfs.newfs () in
	  let sock = UnixLabels.(
	    socket
	      ~cloexec:true
	      ~domain:PF_UNIX
	      ~kind:SOCK_STREAM
	      ~protocol:0)
	  in
	  Unix.(bind sock (ADDR_UNIX "nullfs.sock"));
	  Unix.listen sock 10;
	  eprintf stderr "bound to port /home/droyo/nullfs.sock\n%!";

	  serve fs sock

I can then start `dune runtest -w` in another window, which will watch my
project files for changes and run tests if it detects one. By annotating my
module with the `Styx.Filesystem` type signature, I'm forcing OCaml's type
checker to verify my module meets the `Styx.Filesystem` contract. Immediately
upon saving `null.ml`, my test output becomes the following:

	Error: Signature mismatch:
	       ...
	       The value `write' is required but not provided
	       The value `read' is required but not provided
	       The value `create' is required but not provided
	       The value `openfile' is required but not provided
	       The value `attach' is required but not provided
	       The value `lookup' is required but not provided

By keeping this test open and visible I can consistently get feedback from the
compiler on what functions are missing. As an example, here's the `write`
implementation:

	let write file ~offset:_ ~buf =
	  match file.name with
	  | Root -> Styx.errno Unix.EPERM
	  | Null | Zero -> Ok (Iovec.length buf)

The special file `null` will accept an infinite number of writes, but never
actually persists the changes, so it is sufficient to simply lie about the
number of bytes written. The `read` implementation is a little more
complicated:

	let read file ~offset ~buf =
	  match file.name with
	  | Zero ->
	    Iovec.memset buf '\000';
	    Ok (Iovec.length buf)
	  | Null -> Ok (-1)
	  | Root when offset > 0 && offset <> file.offset ->
	    Styx.errno Unix.ESPIPE
	  | Root when offset > Iovec.length file.fs ->
	    Ok 0
	  | Root ->
	    file.offset <- offset;
	    let iov = Iovec.seek file.fs offset in
	    let rest = fold_left fill_dirbuf buf iov in
	    let n = Iovec.((length buf) - (length rest)) in
	    file.offset <- file.offset + n;
	    Ok n

For `null`, I had to encode special meaning into a negative return value for
`read`, instructing the server to ignore the request. It is probably better to
capture that meaning with a specific type. However, it is inevitable that the
`read`/`write` callbacks will become asynchronous, which will require some
drastic changes, so I won't waste effort addressing this wart.

For `zero`, the callback simply zeroes out the buffer passed to it. I could
save work by constructing the API in such a way that the filesystem could
repeatedly return the same, zeroed-out buffer and copy that directly to the
socket. But that would be optimizing a toy use case. Initially `memset` was
little more than a simple `for` loop that zeroed out the buffer, but in
profiling that turned out to take up more time than I would have liked. Luckily
it is trivial to write a C binding to libc's `memset` function which is highly
optimized:

	CAMLprim value iovec_memset(value view, value c) {
		CAMLparam2(view, c);

		char *buf = Caml_ba_data_val(Field(view, 0));
		int off = Int_val(Field(view, 1);
		int len = Int_val(Field(view, 2));
		int val = Int_val(c);

		memset(buf + off, val, len);

		CAMLreturn (Val_unit);
	}

I could have spent the time to optimize the OCaml function but it would have
been a substantial effort to optimize a toy file system; I don't yet know how
often I will need to zero a buffer in a real file system. [Here](#TODO) is the
full implementation of `nullfs`. I can mount the file system like so:

	# On linux
	sudo mount -t 9p -o port=1564 127.0.0.1 /mnt/nullfs

	# On systems with plan9port and FUSE support
	9pfuse 'tcp!localhost!1564' /mnt/nullfs

The `9pfuse` command wraps a 9P client with the [Filesystem in
Userspace](https://en.wikipedia.org/wiki/Filesystem_in_Userspace) APIs to
expose a 9P file system, and is useful for operating systems which don't have
native 9P support, but do have FUSE support (*BSD). It also has a `-D` flag
which outputs debug logs which I found useful when developing the 9P server.
Getting debug logs from the Linux kernel's `v9fs` module requires building a
kernel with `CONFIG_NET_9P_DEBUG` enabled.

## Benchmarking

I have something running, it's time to benchmark and profile it to see where
my assumptions fell short. As a quick sanity check, we can do some quick
throughput tests with `dd`. This is using a max 9P message size of 64k:

	$ dd if=/dev/zero of=~/nullfs/null bs=63k count=1000000
	1000000+0 records in
	1000000+0 records out
	64512000000 bytes (65 GB, 60 GiB) copied, 71.1554 s, 907 MB/s

	$ dd if=~/nullfs/zero of=/dev/null bs=63k count=1000000
	1000000+0 records in
	1000000+0 records out
	64512000000 bytes (65 GB, 60 GiB) copied, 58.4262 s, 1.1 GB/s

Remember, this file system is doing very little "real" work; this test is
simply measuring the overhead and throughput limitations of the server itself,
and the 9P client in the linux kernel. This level of throughput is not bad,
but would be a bottleneck for some consumer devices. For instance, here is
the same test against my local NVME SSD, with a cold page cache:

	$ echo 1 > /proc/sys/vm/drop_caches
	$ dd of=/dev/null if=~/tmp.0 bs=63k count=100000
	100000+0 records in
	100000+0 records out
	6451200000 bytes (6.5 GB, 6.0 GiB) copied, 2.30062 s, 2.8 GB/s

I cleared the page cache so that the reads would go through to the device.
I did not use `direct` I/O to allow readahead requests to saturate the
device's command queue. In the future I will use a proper benchmark like
`fio`, as benchmarking with `dd` is fraught.

We can do better than this. First, let's take `perf` profiles during the above
benchmarks:

	$ sudo perf record --call-graph dwarf -p $(pgrep nullfs_test.exe) -- sleep 10

We can use Brendan Gregg's
[FlameGraph](https://github.com/brendangregg/FlameGraph/) to visualize these
profiles. Here is a read profile:

[![read profile flamegraph](/img/ocaml-9p-perf-read-02.svg)](/img/ocaml-9p-perf-read-02.svg)

And a write profile:

[![write profile flamegraph](/img/ocaml-9p-perf-write-01.svg)](/img/ocaml-9p-perf-write-01.svg)

We can see in both profiles that the CPU time is dominated by calls to
the [`readv(2)`](https://man7.org/linux/man-pages/man2/readv.2.html) and
[`writev(2)`](https://man7.org/linux/man-pages/man2/writev.2.html) system
calls. In the read test a substantial amount of CPU time is spent in the
`NullFS.read` function, using `Iovec.memset` to zero its output buffer. The
following I/O pattern is followed:

1. Fill input ring buffer from socket
2. Process all complete messages in ring buffer, writing responses to output buffer.
3. If a partial message is read, or the output buffer is full, flush the output buffer.
4. Goto 1

This pattern should avoid making lots of small read/write requests, which should
ameliorate the fixed cost of making the system calls, *if* the client buffers enough
requests in the socket buffer.

One thing I noticed while running these tests was that the `nullfs` process was only
showing 50-60% CPU usage. Since `nullfs` does not touch a storage or network device
and I am writing from `/dev/zero`, it should exhibit closer to 100% CPU usage and be
bottlenecked either by CPU speed, memory bandwidth speed, or context switching speed.
A simple test like the following has higher CPU usage and higher throughput:

	$ unixserver pipe.sock cat /dev/zero # server
	$ unixcat pipe.sock  | pv -B 32M -r > /dev/null # client

First I tried using
[offcputime](http://www.brendangregg.com/offcpuanalysis.html) from
the BCC suite of tools to see where the program spent its time when it
wasn't running:

[![Off CPU flame graph](/img/ocaml-9p-perf-offcpu.svg)](/img/ocaml-9p-perf-offcpu.svg)

This showed the most time was spent in
`schedule_timeout` as part of `unix_stream_read_generic`. This is a
[function](https://github.com/torvalds/linux/blob/7d6beb71da3cc033649d641e1e608713b8220290/net/unix/af_unix.c#L2245)
in the linux kernel. `schedule_timeout` puts the process to sleep for a fixed amount
of time. What the heck is my process sleeping for? Looking at the kernel source,
`unix_stream_read_generic` calls `unix_stream_data_wait`:

	timeo = unix_stream_data_wait(sk, timeo, last,
		      last_len, freezable);

Which calls `schedule_timeout` under certain conditions:

	for (;;) {
		tail = skb_peek_tail(&sk->sk_receive_queue);
		if (tail != last ||
		    (tail && tail->len != last_len) ||
		    sk->sk_err ||
		    (sk->sk_shutdown & RCV_SHUTDOWN) ||
		    signal_pending(current) ||
		    !timeo)
			break;

		/* ... */

		if (freezable)
			timeo = freezable_schedule_timeout(timeo);
		else
			timeo = schedule_timeout(timeo);

it looks like `schedule_timeout` is only called when there is no data in
the receive queue. The `timeo` value is obtained from the `SO_RCVTIMEO`
socket option. There's one problem; the SO_RCVTIMEO option is 0. So from a
first glance at the code, `schedule_timeout` should never be called. There
must be some other code path that triggers this.

At my day job, I occasionaly use [Schedviz](https://github.com/google/schedviz)
as a way to view the activity over time of different threads of execution. I
can see a repeating ~15μs period of sleeping in the `nullfs` process:

<img src="/img/ocaml-9p-perf-schedviz-1.png" style="width:75%" alt="schedviz nullfs" />

Normally nullfs is tag-teaming with a kernel worker thread which actually
performs the I/O. Why is it sleeping? We can see if we find the `dd` process:


<img src="/img/ocaml-9p-perf-schedviz-2.png" style="width:75%" alt="schedviz nullfs" />

So we can see `dd` is running while `nullfs` is sleeping, and
vice-versa. Ideally, `dd` should buffer enough data that `nullfs` can
always run. If we compare to the `unixcat` test listed above, you can see
the difference:

<img src="/img/ocaml-9p-perf-schedviz-3.png" style="width:75%" alt="schedviz cat" />

The `cat` on CPU 0 is the reader, and the `cat` on CPU 1 is the writer. You
can see that the reader is running constantly, and the writer wakes up about
every 50ms to "top-up" the socket buffer. That's the behavior we want to
see on `nullfs`; one side should be running all the time.

I was able to reproduce the same performance pattern (but with slightly
lower throughput) with a `nullpipe` program that looked like this:

	let rec consume fd buf =
	  match Iovec.readv fd buf with
	  | exception End_of_file -> ()
	  | _ -> consume fd buf

So there was nothing surprising going on with the OCaml runtime, like a
hidden sleep or something. The actual "problem" is somewhat intuitive;
in this main loop:

1. Fill input ring buffer from socket
2. Process all full messages in ring buffer, writing responses to output
   buffer.
3. If a partial message is read, or the output buffer is full, flush the
   output buffer.
4. Goto 1

if the input buffer is large enough to read everything in the socket buffer,
the server/client interaction will naturally stutter, since the server will not
send responses until all pending requests have been read and processed. Adding
one flush per iteration increased throughput from 1 GB/s to 1.5 GB/s, but
I still see the "stuttering", where the client only wakes up to write data
after the server has fully emptied the buffer. Perhaps this is inherent to
the use of `v9fs`.

At this point, I want to defer any optimization on I/O, because there are
some fairly important changes to make to the I/O design for the server. Most
importantly, I/O must become asynchronous, so the server will be able to
request additional requests while still working on older requests. This will
be a big enough change that the performance optimizations that I can make
on the current design could become obsolete.
