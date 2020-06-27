+++
date = "2020-05-11T18:00:00Z"
title = "Let's build a high performance file system! Part 0"
tags = ["ocaml", "plan9", "9P", "fs", "performance", "programming"]
description = "Implementing a 9P server (and client) in OCaml"
+++

In order to serve a file system over 9P, we need an
implementation of the 9P protocol. Luckily, the protocol is
incredibly simple, especially the base 9P2000 variant without
the unix or Linux extensions. For a more in-depth walkthrough of
implementing a 9P server, take a look at my [previous](../9p)
post about implementing a 9P server in Go.  This article will
focus less on 9P itself and more on how to match the protocol
to OCaml, and using the build toolchain.

# Why?

There are existing implementations of 9P in OCaml:

* [The implementation in
moby/datakit](https://github.com/moby/datakit/tree/master/src/datakit-server-9p)
* [MirageOS has a 9P
  implementation](https://github.com/mirage/ocaml-9p)
* [ocaml9p](https://github.com/oscarh/ocaml9p) implements a
  client, and could be extended to support a server.

Why not use or extend an existing implementation? To be
honest, it's very likely I will do that in the future. However,
throughout this project I intend to do terrible things to the
library in the name of ¡¡science!! that would not be suitable
to contribute back. Until I understand the problem better,
I want full control over the implementation. The same goes for
other external dependencies; I can only be convinced that
I need a dependency after I have struggled without it.

Luckily 9P is such a simple protocol that it can be implemented
fairly quickly. The rest of the article will describe the "mark
1" version of my 9P library. It will inevitably be revisited
again and again throughout the course of this series.

I also should note again that I am a novice with OCaml, and will
probably make lots of mistakes. Any feedback is welcome; feel
free to send an email to david@aqwari.net.

# Message parsing

The first goal; parse all the 9P messages from a file. Recall that a
9P message looks like this:

<img src="/img/9p-message.svg" style="width:100%"/>

It's a mixture of fixed-length fields and variable-length fields. Any
variable-length field is prefixed with its length. Integers larger than
a byte use little endian byte order.

To parse these messages, we'll treat the message like a stack
of bytes that we progressively pop different sized values off of.

	(* A dot is a movable view into a byte buffer *)
	type dot = {
		buf : bytes;
		mutable l : int;
		mutable r : int
	}

	(* advance the left edge of the window and return the old value *)
	let advance dot n =
		let old = win.l in
		win.l <- win.l + n; old

	let u8  dot = advance dot 1 |> Bytes.get_uint8 dot.buf
	let u16 dot = advance dot 2 |> Bytes.get_uint16_le dot.buf
	let u32 dot = advance dot 4 |> Bytes.get_int32_le dot.buf
		|> Int32.unsigned_to_int |> Option.get
	let u64 dot = advance dot 8 |> Bytes.get_int64_le dot.buf

	(* The next n bytes from dot as a byte array *)
	let next n dot = Bytes.sub dot.buf (advance dot n) n

The `|>` operator in OCaml takes the left-hand argument and
adds it to the end of the right-hand function's argument
list. Variable-length fields are made of a 2-byte integer
`length` followed by `length` bytes:

	let str dot =
		let len = u16 dot in
		Bytes.sub dot.buf (advance dot len) len

The `Twalk` and `Rwalk` messages contain multiple variable-length
fields:

<img src="/img/9p-twalk.svg" style="width:100%"/>

To parse this easily, we define a `loop` function that can
be used to call one of the other functions multiple times:

	let loop f n dot =
	  let rec aux xs f n dot =
	    if n <= 0 then xs else aux ((f dot)::xs) f (n-1) dot in
	  List.rev (aux [] f n dot)

I want to point out the inferred type signature for the `loop function`:

	val loop : (t -> 'a) -> int -> t -> 'a list

the item in the resulting list is the return value of the passed function.
Here's an example of parsing a `Twalk` and an `Rwalk` message:

	type twalk = {
		tag : int;
		fid : int;
		newfid : int;
		wname : string list
	};

	type rwalk = {
		tag : int;
		wqid : bytes list
	};

	let twalk_of_dot dot = {
		tag = u16 dot;
		fid = u32 dot;
		newfid = u32 dot;
		wname = loop str dot (u16 dot)
	}

	let qid_length = 13
	let rwalk_of_dot dot = {
		tag = u16 dot;
		wqid = loop (next qid_length) dot (u16 dot)
	}

The `loop` function works just as well for reading variable-length
strings as it did for reading 13-byte `qids`. Also note the use of
currying in `rwalk_of_dot`. This line:

		wqid = loop (next qid_length) dot (u16 dot)

is the same as this:

		wqid = loop (fun x -> next qid_length x) dot (u16 dot)

To keep things tidy, I put these functions into a module called
[Dot](#TODO).  For the time being, we'll follow the example
above and store each message in its own record type. To avoid
clutter, I'll put each message in its own module, like so:

	module Tversion = struct
		let code = 100
		type t = { tag : int; msize : int; version : string }
		let of_dot dot = Dot.({
			tag = u16 dot;
			msize = u32 dot;
			version = str dot
		})
		let pp ppf {tag; msize; version} =
			Format.fprintf ppf "Tversion tag=%i msize=%i version=%s"
				tag msize version
	end

There's a big problem here:

	let of_dot dot = Dot.({
		tag = u16 dot;
		msize = u32 dot;
		version = str dot
	})

The `Dot` type and the functions on it are *not*
purely functional, meaning calling these functions in
a different order will produce different results. And
the order of evaluation of record expressions is [not
specified](https://caml.inria.fr/pub/docs/manual-ocaml/expr.html#sss:expr-records).
Currently, with an OCaml 4.09 compiler on an amd64 system,
these expressions are evaluated from last to first, but this
is not behavior that can be relied upon. We need to do this:

	let of_dot dot = Dot.(
		let tag = u16 dot in
		let msize = u32 dot in
		let version = str dot in
		{tag; msize; version})

This is less concise, but it works. At a later date I
will probably revisit this and use a different approach
like using recursion instead of mutatin, or using
[bitstring](https://bitstring.software) to pattern match
over the messages themselves.

The `pp` function is just a debugging aid to spot-check the
result of our parsing. It allows you to print the value from
a `printf` call like so:

	Printf.printf "message %a" Tversion.pp msg

While other languages like Go and Java have
reflection that allows you to lookup information
about a value's type at run-time, in OCaml, all type
information is removed during compilation. See [this
passage](https://dev.realworldocaml.org/runtime-memory-layout.html)
from Real World OCaml on the memory representation of values.

Adding the pretty-printing functions for
each type is tedious, so we will use the
[ppx_deriving](https://github.com/ocaml-ppx/ppx_deriving)
pre-processor to do it for us instead. I add this to the
`dune` file in my library directory:

	  (preprocess ppx_deriving.show)

And add this to the end of my type expressions:

	type t = { tag : int; msize : int; version : string }
	[@@deriving show]

Many 9P messages share some common fields like `tag` and `fid`
There's an opportunity to share some parsing code between the
message type, but I will keep them separate for now until I
become more familiar with the language.

After defining a type for each message, we can create a
variant type that accepts any of them, along with a
function to parse a full mesage:

	module Message = struct
		type t =
		| Tversion of Tversion.t
		| Rversion of Rversion.t
		(* ... snip ... *)
		| Twstat of Twstat.t
		| Rwstat of Rwstat.t
		[@@deriving show]

		let of_bytes b =
			let dot = Dot.of_bytes b in
			match Dot.u8 dot with
			| 100 -> Tversion(Tversion.of_dot dot)
			| 101 -> Rversion(Rversion.of_dot dot)
			(* ... snip ... *)
			| 126 -> Twstat(Twstat.of_dot dot)
			| 127 -> Rwstat(Rwstat.of_dot dot)
			| other -> failwith
				("invalid message " ^ (string_of_int other))
	end

So now we finally have what we need to parse all the messages in
our test data. I want to run this as a unit test, so I created a
`test/dune` file in my project directory that looks like this:

	(test
	  (name styx_test)
	  (deps (glob_files testdata/*.9p))
	  (libraries styx))

My 9P library is named `styx` (this was the name of the 9P
protocol in the [Inferno](http://www.vitanuova.com/inferno/)
operating system, and has the benefit of being a valid
identifier). The rest of the code below will go into the file
`test/styx_test.ml`.

The first thing we'll do is open our file containing traffic captured
from a 9P protocol session and turn it into a sequence of discrete
messages. We'll present it as the standard library's `Seq.t`, which
is a lazy list:

	(* A lazy sequence of messages *)
	let sequence r =
		(* reads a single int from the input file *)
		let read_u32 r =
			let buf = Bytes.create 4 in
			really_input r buf 0 4; Bytes.get_int32_le buf 0
			|> Int32.unsigned_to_int
			|> Option.get in

		(* Get a single message from the input stream *)
		let segment r = (read_u32 r)-4
			|> really_input_string r
			|> Bytes.of_string in

		let rec next () = match segment r with
		| exception End_of_file -> Seq.Nil
		| msg -> Seq.Cons(msg, next) in next

	let messages = open_in_bin "testdata/sample.client.9p"

Now `msg_bufs` contains a sequence of unparsed 9P messages.
Through successive calls to `Seq.map` we can produce
human-readable strings:

	let () = open_in_bin "testdata/sample.client.9p"
	  |> sequence
	  |> Seq.map Protocol.Msg.of_bytes
	  |> Seq.map Protocol.Msg.show
	  |> Seq.iter print_endline

Running `dune test` now produces something like this:

	********** NEW BUILD **********

	   styx_test alias test/runtest
	(Protocol.Msg.Tversion
	   { Protocol.Tversion.tag = 65535; msize = 8192; version = "9P2000" })
	(Protocol.Msg.Tattach
	   { Protocol.Tattach.tag = 0; fid = 0; afid = 4294967295; uname = "droyo";
	     aname = "" })
	(Protocol.Msg.Twalk
	   { Protocol.Twalk.tag = 0; fid = 0; newfid = 1; wname = [".Trash"] })
	(Protocol.Msg.Twalk
	   { Protocol.Twalk.tag = 0; fid = 0; newfid = 1; wname = [".Trash-1000"] })

One last basic requirement is the ability to encode messages.
I've added `to_bytes` functions to each module, like this:

	module Tversion = struct
	  ...
	  let length v = 4 + 1 + 2 + 4 + 2 + (String.length v.version)
	  let to_bytes (v:t) = Dot.of_bytes (Bytes.create (length v))
	    |> Dot.pu32 (length v)
	    |> Dot.pu8  code
	    |> Dot.pu16 v.tag
	    |> Dot.pu32 v.msize
	    |> Dot.pstr v.version
	    |> Dot.get_buf
	end

We can now extend our test to decode a message, encode it,
decode it again, and see if we get the same result:

	let _ = open_in_bin "testdata/sample.client.9p"
	  |> sequence
	  |> Seq.map Protocol.Msg.of_bytes
	  |> Seq.map Protocol.Msg.to_bytes
	  |> Seq.map Protocol.Msg.of_bytes
	  |> Seq.map Protocol.Msg.show
	  |> Seq.fold_left (fun n s -> Printf.printf "%03d %s\n" n s; n+1) 0

I can run my test with `OCAMLRUNPARAM=b dune test` to get
stack traces on any exceptions.

Note that, in both the encoding and decoding steps, I am
allocating fresh buffers instead of writing to existing ones.
One of the first optimizations I am likely to make is to avoid
copying large messages and avoid allocating lots of tiny
buffers by redesigning the API around IO vectors and a shared
buffer, as described in the [Ensemble (pdf)](https://ecommons.cornell.edu/bitstream/handle/1813/7316/98-1662.pdf) paper.
However, before taking that step I want it to be informed by
benchmarks.

The state of the `styx` library at thi point can be viewed
[here](https://git.sr.ht/~droyo/filesystem/tree/c195aba9067b19c2f477efdbb64a16ca863f9602)

# Implementing a server

We can now decode and encode 9P messages, so what is
left is to design a server API surface. There are a lot
of ways we can go here. One place to look for inspiration
is in the VFS layers of various operating systems: Here's
[Linux](https://www.kernel.org/doc/Documentation/filesystems/vfs.txt)
and [Plan 9](http://man.cat-v.org/plan_9/2/9p). One obvious
approach would be to define a contract that has a 1:1 correlation
with the various 9P operations, like so:

	module type ServerType =
	  sig
	    open Protocol

	    val version : t ->  Tversion.t -> Rversion.t result
	    (** [version t req] negotiates the 9P version and message size *)

	    val auth : t -> Tauth.t -> Rauth.t result
	    (** [auth t req] establishes an open file to use for auth *)

	    val flush : t -> Tflush.t -> Rflush.t
	    (** [flush t req] cancels the request with tag [req.oldtag] *)

	    val open : t -> Topen.t -> Ropen.t result
	    (** [open t req] prepares a file for I/O

	    val walk : t -> Twalk.t -> Rwalk.t result
	    (** [walk t req] traverses the elements in [req.wname] *)
	    ...

and so on. The user of the library would then create a module
implementing this module signature. Then the styx library can implement
routines to actually run the server, while still allowing for multiple
implementations. Implementations could share common routines by using
`include` like so:

	module Common = struct
	  val version t req =
	    (* common implementation of version here *)
	end

	module MyServer = struct
	  include Common

	  type t = {(* implementation of Server.t *)}
	end

Glossing over the fact that we don't have an implementation yet, we can
then implement a server like so:

	let M = Server.Make(not_implemented_yet)


# Execution loop

When I implemented a 9P server in Go, I spawned a pair of
goroutines for each 9P session. One goroutine consumed 9P
messages, performing any book keeping for file handles, and
another ran user-provided code that would implement the file
system. I let the Go runtime distribute the goroutines across
system threads however it saw fit.

The OCaml runtime does not have strong support for
paralellism; while there is a `Thread` module that allows
for the usage of OS threads, a Global Interpreter Lock
prevents those threads from executing OCaml code (but they
can make system calls or execute C code). There is [ongoing
work](http://ocamllabs.io/doc/multicore.html) to provide better
multi-core support for OCaml.

However, none of that matters for what we're trying to do. The
intended use case for this file system is to provide persistent
storage for a single machine, or at worst, a home network. A
single CPU thread is capable of transferring enough data to/from
memory to dwarf what most conventional storage devices can
achieve. We will have far more bottlenecks at the storage layer.

OCaml does have a few libraries to provide concurrency, typically
in the form of asynchronous I/O and callbacks. There are many,
but here are the few that I found the most attractive:

* [Lwt](https://ocsigen.org/lwt/) for its widespread use and
  maturity.
* [Async](https://opensource.janestreet.com/async/) is similar
  but has a cleaner API (in my opinion).
* [Luv](https://github.com/aantron/luv) is a binding to
  [libuv](https://github.com/libuv/libuv), a highly optimized event
  loop system that was spun out of Node.js. It's lower level than
  the other two and there are plans to make Luv a backend for Lwt.

To start, I will be using the most popular library, `Lwt`.


