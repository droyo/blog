+++
date = "2020-05-11T18:00:00Z"
title = "Let's build a high performance file system! Part 0"
tags = ["ocaml", "plan9", "9P", "fs", "performance", "programming"]
description = "Implementing a 9P server in OCaml"
draft = true
+++

In order to serve a file system over 9P, I need an implementation
of the 9P protocol. Luckily, the protocol is incredibly simple. For
a more in-depth walkthrough of implementing a 9P server, take a look
at my [previous](../9p) post about implementing a 9P server in Go.

The rest of the article will describe the "mark 1" version of my 9P
library. It will inevitably be revisited again and again throughout
the course of this series. I also should note again that I am a novice
with OCaml, and will probably make lots of mistakes. Any feedback is
appreciated; feel free to send an email to david@aqwari.net. I also frequent
[discuss.ocaml.org](https://discuss.ocaml.org/)

## Message parsing

The first goal; parse all the 9P messages from a file. Recall that a 9P
message looks like this:

<img src="/img/9p-message.svg" style="width:75%" alt="9P Topen message" />

It's a mixture of fixed-length fields and variable-length fields (`name`
in the above graphic). Any variable-length field is prefixed with its
length. Integers are encoded in little endian byte order. The protocol is
described in the [Plan 9 manual](http://man.cat-v.org/plan_9/5/intro).

To parse these messages, I'll treat the message like a stack of bytes
that I progressively pop different-sized values off of.

	type dot = { buf : bytes; mutable l : int; mutable r : int }

	let advance dot n =
	  let old = win.l in
	  win.l <- win.l + n;
	  old

	let u64 dot = advance dot 8 |> Bytes.get_int64_le dot.buf
	let next n dot = Bytes.sub dot.buf (advance dot n) n

	let str dot =
	  let len = u16 dot in
	  Bytes.sub dot.buf (advance dot len) len

The `|>` operator in OCaml takes the result of the left-hand expression and
passes as the last argument to the right-hand function (in the code above,
the right-hand arguments are curried functions). While the implementation
is different, in practice it is similar to the pipe operator in unix shell
scripts.

To keep things tidy, I put these functions into a module called `Dot`.
For the time being, I'll follow the example above and store each message
in its own record type. To avoid clutter, I'll put each message in its own
module, like so:

	module Tversion = struct
	  let code = 100
	  type t = { tag : int; msize : int; version : string }
	  let of_dot dot = Dot.(
	    let tag = u16 dot in
	    let msize = u32 dot in
	    let version = str dot in
	    { tag; msize; version })

	  let pp ppf {tag; msize; version} =
	    Format.fprintf ppf "Tversion tag=%i msize=%i version=%s"
	      tag msize version
	end

The `pp` function is just a debugging aid to spot-check the result of our
parsing. It allows you to print the value from a `printf` call like so:

	Format.printf "message %a" Tversion.pp msg

While other languages like Go and Java have reflection that allows their
`printf` functions to lookup information about a value's type at run-time,
in OCaml, all type information is removed during compilation. See [this
passage](https://dev.realworldocaml.org/runtime-memory-layout.html) from Real
World OCaml on the memory representation of values. I was surprised to learn
this. I was both impressed at how much safety OCaml can guarantee at compile
time, and encouraged to use lots of types without worrying about performance.

Adding the pretty-printing functions for each type is tedious, so I am using
the [ppx_deriving](https://github.com/ocaml-ppx/ppx_deriving) pre-processor to
generate them. With this pre-processor, I can add this to the end of a type
expression to get `show` functions that convert the type to a human-readable
string:

	type t = { tag : int; msize : int; version : string }
	[@@deriving show]

After defining a type for each message, I can create a variant type that
accepts any one of them, along with a function to parse a full mesage:

	module Message = struct
	  type t =
	  | Tversion of Tversion.t
	  (* ... snip ... *)
	  | Rwstat of Rwstat.t

	  let of_bytes b =
	    let dot = Dot.of_bytes b in
	    match Dot.u8 dot with
	    | 100 -> Tversion (Tversion.of_dot dot)
	    (* ... snip ... *)
	    | 127 -> Rwstat (Rwstat.of_dot dot)
	    | other -> failwith ("invalid message " ^ (string_of_int other))
	end

So now I finally have what I need to parse all the messages in our test data. I
want to run this as a unit test, so I created a `test/dune` file in my project
directory that looks like this:

	(test
	  (name styx_test)
	  (deps (glob_files testdata/*.9p))
	  (libraries styx))

My 9P library is named `styx` (this was the name for the 9P protocol in
the [Inferno](http://www.vitanuova.com/inferno/) operating system, and has
the benefit of being a valid identifier). I put 9P traffic collected from a
packet capture in the `test/testdata` directory. The rest of the code below
will go into the file `test/styx_test.ml`.

The first thing I'll do is open our file containing traffic captured from a 9P
protocol session and turn it into a sequence of discrete messages. We'll
present it as the standard library's
[`Seq.t`](https://caml.inria.fr/pub/docs/manual-ocaml/libref/Seq.html), which
is a lazy list based on closures, ideal for streaming data.

	(* A lazy sequence of messages *)
	let sequence r =
		(* reads one little-endian int from the input *)
		let read_u32 r =
			let buf = Bytes.create 4 in
			really_input r buf 0 4; Bytes.get_int32_le buf 0
			|> Int32.unsigned_to_int
			|> Option.get in

		(* Get a single message from the input stream *)
		let segment r = (read_u32 r)-4
			|> really_input_string r
			|> Bytes.of_string in

		let rec next () =
		  match segment r with
		  | exception End_of_file -> Seq.Nil
		  | msg -> Seq.Cons (msg, next)
		in
		next

	let messages = open_in_bin "testdata/sample.client.9p"

Now `sequence` returns a sequence of unparsed 9P messages.  Through successive
calls to `Seq.map` I can produce human-readable strings:

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

One last basic requirement is the ability to encode messages.  I've added
`to_bytes` functions to each module, like this:

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

I can now extend our test to decode a message, encode it, decode it again,
and see if I get the same result:

	let _ = open_in_bin "testdata/sample.client.9p"
	  |> sequence
	  |> Seq.map Protocol.Msg.of_bytes
	  |> Seq.map Protocol.Msg.to_bytes
	  |> Seq.map Protocol.Msg.of_bytes
	  |> Seq.map Protocol.Msg.show
	  |> Seq.fold_left (fun n s -> Printf.printf "%03d %s\n" n s; n+1) 0

I can run my test with `OCAMLRUNPARAM=b dune test` to get stack traces on
any exceptions. The state of the `styx` library at this point can be viewed
[here](https://git.sr.ht/~droyo/filesystem/tree/c195aba9067b19c2f477efdbb64a16ca863f9602)

## Message parsing, take 2

Though the approach described above works, I have a lot of problems with
it. My main concern is that it is fragile; for variable-length fields, it
takes a number provided by a third party and uses that to determine an index
into memory. While bounds checks are performed for the `bytes` and `string`
types, so there is no danger of segfaults or security issues, this is still
a recipe for unplanned exceptions which will crash the program. In addition,
for the real server, I want to keep the number of system calls low by reading
multiple messages at a time, and I want to reduce garbage collector activity
by re-using memory. The main loop would look something like this:

1. Read all data from connection into a buffer
2. Parse next request in buffer
3. If the message is incomplete, `goto 1`
4. Process request, send response
5. `goto 2`

However, if I store multiple messages in the same buffer, a malicious message
could use one of its variable-length fields to recover data from adjacent
messages. I have some other, premature concerns about performance;
there is an excessive amount of copying and allocation taking place, since
a new `bytes` array is allocated for every message, then discarded once
it's no longer needed.

### Vectored I/O; the `Iovec` module

To allow for efficient re-use of memory, I'll be using a ring buffer to store
the messages:

<img src="/img/9p-ring-buffer.svg" style="height: 40%" />

Instead of a `bytes` type, I will be using a
[Bigarray](https://caml.inria.fr/pub/docs/manual-ocaml/libref/Bigarray.html)
for message buffers. A Bigarray can be (almost) directly passed to system
calls and C functions; the memory layout is compatible with C and the garbage
collector will not move a Bigarray during compaction.

Rather than unmarshalling messages into message-specific types,
I will re-use the protocol representation internally, and pass a reference
to the buffer.  This is the same approach I used in my [Go](../9p/parsing)
library. While I am no longer convinced of any performance benefits of this
approach, I will still do it because it makes it easier to re-use parsing code
between message types. It will also create less work for the garbage collector,
which may not provide a direct performance benefit (the OCaml garbage collector
is very fast), but may yield more predictability in memory usage.

I'll create an `Iovec` module with a type similar to the `struct iovec`
type used by POSIX vectored I/O system calls; a reference to a chunk of
memory plus a length:

	(* iovec.mli *)
	type buffer

	type view = private {
	  buf : buffer; (** backing data *)
	  off : int; (** start of this view *)
	  len : int; (** number of bytes in this view *)
	}

	type t = view list

Multiple `view` values can be strung together in a `list`. The rest of the
module provides functions to work with these lists as if they were contiguous
regions of memory. The `private` keyword is new. It means that the fields
in this record are visible, but you cannot construct a new `view` value by
constructing a new record from scratch:

	(* this will fail to compile *)
	let my_view = Iovec.{ buf = new_buffer len; off = 0; len = 0 }

You may be thinking, "what good is a type if I can't create values for it? And
why doesn't `buffer` have a definition?" This is a good time to talk about
OCaml's [module system](https://ocaml.org/learn/tutorials/modules.html).
OCaml allows you to group types, functions, and other values into *modules*,
as you saw earlier with the `Tversion` module. Modules have a *signature*,
which is a set of names visible to code outside of the module, and an
*implementation*.  In the modules above, there is no signature defined,
so it is inferred from the implementation. But an explicit signature can be
used to restrict the module interface. I can define the `Iovec` module like so:

	module Iovec : sig
	  (* module signature *)
	  type buffer
	  type view = private { buf : buffer; off : int; len : int }

	  (* ... *)
	end = struct
	  (* module implementation *)
	  type buffer =
	    (char, Bigarray.int8_unsigned_elt, Bigarray.c_layout) Bigarray.Array1.t

	  type view = { buf : buffer; off : int; len : int }

	  (* ... *)
	end

Note the lack of a `private` keyword in the implementation, and the
presence of a definition for the `buffer` type. The end result of this
is that `view` and `buffer` values can only be constructed by functions
within the `Iovec` module, and no modules outside of the `Iovec`
module have any visibility into what a `buffer` is; they cannot, for
example, use functions in the `Bigarray` module to read or write into an
`Iovec.buffer` value. The `buffer` type is what's known as an [abstract
type](https://ocaml.org/learn/tutorials/modules.html#Abstract-types). Abstract
types are a powerful tool for separation of concerns in a large codebase. Using
an abstract type allows you to arbitrarily constrain how external code
manipulates values of that type.

Note that this can also be done using interface files; putting
the implementation in a file `iovec.ml` and the signature in a file called
`iovec.mli` will accomplish the same result and save one level of indentation.

By carefully choosing the API presented by the `Iovec` module I can provide
an API that protects the following invariants:

* An `Iovec.view` value can *only* be narrowed, *never* extended.
* An `Iovec.t` can *only* be extended by concatenating views.

With these two invariants, we eliminate the risk of accessing an invalid
memory address with the `Iovec` module. It's impossible to construct
a value that points out of bounds. I only have to make sure that the
functions inside of the `Iovec` module are safe.

I will skip most of the implementation of the `Iovec` module. It is very
similar to the popular [Cstruct](https://github.com/mirage/ocaml-cstruct)
module; indeed, I will probably replace it in favor of `Cstruct` in the
future. In addition to the main `Iovec.t` type, it defines a basic ring buffer,
some primitives for writing strings and integers to an `Iovec.t`, and bindings
to POSIX vectored I/O system calls such as `readv` and `writev` using OCaml's
[C interface](https://caml.inria.fr/pub/docs/manual-ocaml/intfc.html). You
can view the iovec module [here](#TODO).

## Reducing boilerplate

I wasn't happy with the amount of duplication between each message type.
There was also a lot of mental math to calculate the expected length of
messages and the beginning of fields that I thought would be easy to get wrong.
The Plan 9 manual provides a nice definition of each method type:

	size[4] Tversion tag[2] msize[4] version[s]
	size[4] Rversion tag[2] msize[4] version[s]
	size[4] Tauth tag[2] afid[4] uname[s] aname[s]
	size[4] Rauth tag[2] aqid[13]
	size[4] Rerror tag[2] ename[s]
	size[4] Tflush tag[2] oldtag[2]
	size[4] Rflush tag[2]
	...

It would be nice if I could re-use this definition and derive parsing functions
from it. Something like this:

	let tversion = [ size; op; tag; int32; string ]
	let tauth = [ size; op; tag; fid; string; string ]

But what are the values in this list?

	type kind =
	  | Fixed of int (* A fixed-size value of [n] bytes *)
	  | Var of int (* [n]-byte integer followed by [n] bytes *)

	type layout = kind list

	let int8 = Fixed 1
	let int16 = Fixed 2
	let int32 = Fixed 4
	let int64 = Fixed 8
	let string = Var 2
	let size = int32
	let op = int8
	let tag = int16
	let fid = int32

The variable names like `int8` and `size` here aren't necessary; I could have
written

	let tversion = [ Fixed 4; Fixed 1; Fixed 2; Fixed 4; Var 2 ]

but this is less readable, and the extra names cost us nothing. To read or skip
a field, I take a subset of the `Iovec.t` based on the layout:

	let read iov = function
	  | Fixed n -> Iovec.trunc iov n
	  | Var n ->
	    let len = Iovec.(to_int (trunc iov n)) in
	    Iovec.(trunc (seek iov n)) len

	let skip iov = function
	  | Fixed n -> Iovec.trunc iov n
	  | Var n ->
	    let len = Iovec.(to_int (trunc iov n)) in
	    Iovec.seek iov (n + len)

To validate a message, I simply verify that all of its fields are present:

	(* Iovec.OOB will be raised if the message is invalid *)
	let validate layout iov = ignore (List.fold_left skip iov layout)

And accessing a single field uses a similar approach:

	let rec field layout iov n =
	  match layout, n with
	  | [], _ -> invalid_arg "layout is too short"
	  | kind::_, 0 -> read iov kind
	  | kind::xs, n -> field xs (skip iov kind) (n - 1)

I put all the helper types and functions in a helper type named `Msg`. Putting
it all together here's the code for one message:

	module Tattach : sig
	  type t
	  val size : t -> int
	  val tag : t -> Tag.t
	  val fid : t -> Fid.t
	  val afid : t -> Fid.t
	  val uname : t -> string
	  val aname : t -> string
	end = struct
	  type t = Iovec.t
	  let layout = Msg.([size; op; tag; fid; fid; string; string])

	  let size iov = Iovec.to_int (Msg.field iov 0)
	  let tag iov = Tag.of_iovec (Msg.field iov 1)
	  let fid iov = Fid.of_iovec (Msg.field iov 2)
	  let afid iov = Fid.of_iovec (Msg.field iov 3)
	  let uname iov = Iovec.to_string (Msg.field iov 4)
	  let aname iov = Iovec.to_string (Msg.field iov 5)
	end

Note how the type is abstract in the module signature. Code outside of the
`Tattach` module won't know that it's based on an `Iovec.t` and cannot use
functions for that type to arbitrarily read segments of the message. If
I trusted the verification sufficiently, I could skip bounds checks in
the accessor functions. I don't have that level of confidence yet :). An
additional future optimization could be to combine runs of fixed-width fields
when building accessor functions.

## Server implementation

It is not enough to be able to read and write protocol messages; a server
must take some semantic actions based on the requests it receives and build
the appropriate responses. To reduce the number of factors I need to keep
in my head at once, I want to have an abstraction between the 9P server and
the file system. There should be a contract that my file system must meet,
and so long as it does that, it does not need to concern itself with the
inner workings of the 9P protocol.

Most operating systems define a VFS layer that all file systems implement. This
allows file systems to be interchangeable and reduces the spread of
filesystem-specific code throughout the code base. It also constrains the
problem space for file system implementors and gives them a clear objective to
meet. In C code bases like the linux kernel, the VFS is defined as a `struct`
of function pointers. Other languages like Java and Go have `interface` types,
or abstract classes like Python.

In OCaml, I will define the contract as a module signature:

	module type Filesystem = sig
	  type t
	  type file

	  val attr : file -> Qid.t -> unit
	  val attach : t -> uname:string -> aname:string -> (file, error) result
	  val lookup : file -> string -> (file, error) result
	  val openfile : file -> flags:Flag.t -> (file, error) result
	  val create : file -> name:Iovec.t -> flags:Flag.t -> (file, error) result
	  val read : file -> offset:int -> buf:Iovec.t -> (int, error) result
	  val write : file -> offset:int -> buf:Iovec.t -> (int, error) result
	  val close : file -> (unit, error) result
	  val remove : file -> (unit, error) result
	  val stat : file -> Iovec.t -> (int, error) result
	  val wstat : file -> Stat.t -> (unit, error) result
	end

The signature closely follows the various 9P protocol request types. Note
that the module signature does not dictate what the `t` or `file` types
look like; only that they exist, and that the relevant functions accept
or return them. To turn this contract into something useful, I use a
[functor](https://ocaml.org/learn/tutorials/modules.html#Functors) which can
be thought of as a function that accepts one or more modules and returns a
new one.

	module Server (FS : Filesystem) = struct
	  (* server implementation goes here. file system callbacks and
	     types are accessible as FS.t, FS.read, FS.write, etc. *)
	end

The functor can be used like so, for a server that reads requests on standard
input and writes requests to standard output:

	module MyFS = struct
	  (* Styx.Filesystem implementation goes here *)
	end

	module Srv = Styx.Server(MyFS)

	let fs = MyFS.newfs () in
	Srv.run ~fs ~r:Unix.stdin ~w:Unix.stdout

Inside the `Server` functor I have a dispatch function that should look fairly
straightforward:

	let handle srv size tag msg_type msg =
	  match msg_type with
	  | Partial -> fill_buffer srv msg
	  | Invalid -> fatal_error srv msg

	  | Tversion -> handle_tversion srv tag msg

	  | _ when version_pending srv -> fatal_error srv "Tversion expected"
	  | _ when size > srv.msize    -> fatal_error srv "msize exceeded"
	  | _ when tag = Tag.notag     -> fatal_error srv "invalid tag"

	  | Tauth   -> handle_tauth srv tag msg
	  | Tflush  -> handle_tflush srv tag msg
	  | Tattach -> handle_tattach srv tag msg
	  | Tread   -> handle_tread srv tag msg
	  (* ... snip ... *)

One function is defined for each message type. There are two "special"
message types, `Invalid` and `Partial`. An `Invalid` message represents
a malformed message, and the only valid reaction is to end the session. A
`Partial` message occurs when there is not enough data in the input buffer
to read a full message. The `fill_buffer` function will use the `readv(2)`
system call to read data from the input file descriptor into an `Iovec.t`
covering the vacant space in a ring buffer.

Here is an example of the handler for `Topen` requests:

	let handle_topen srv tag msg =
	  let fid = Topen.fid msg in
	  let flags = Topen.flags msg in
	  let file = FidMap.find fid srv.filemap in

	  match FS.openfile file ~flags with
	  | Error err -> handle_error srv tag err
	  | Ok opened ->
	    let iounit = match FS.iounit opened with
	      | 0 -> srv.iounit
	      | n -> min n srv.iounit
	    in
	    let iov = Iovec.ring_vacant srv.wbuf in
	    let rsp = Ropen.create iov ~tag ~iounit in
	    FS.attr opened (Ropen.qid rsp);
	    {srv with wbuf = Iovec.ring_grow srv.wbuf (Ropen.size rsp)}

The `srv` module contains a mapping from 9P file identifiers (fids) to values
of the abstract `Filesystem.file` type. This way the file system does not have
to keep track of fids, only `file` values.

I want to call attention to the parameters for the main `handle` function:

	let handle (type t) srv size tag (msg_type:t message) (msg:t) =

The `(type t)` expression at the beginning declares a [locally abstract
type](https://caml.inria.fr/pub/docs/manual-ocaml/locallyabstract.html). It is
necessary to prevent OCaml's type checker from going too far in inferring the
function's type. Without the locally abstract type, the type checker will see

	let handle srv size tag msg_type msg =
	  match msg_type with
	  | Partial -> ...

and decide the function only accepts `msg_type` values whose type is `Partial`:

	val handle : t -> int -> Tag.t -> Partial message -> Partial -> t

this will fail to compile, since the rest of the cases in the match are not
compatible with the `Partial` type:

	File "lib/styx.ml", line 599, characters 13-20:
	599 |         | Invalid -> fatal_error srv msg
	                ^^^^^^^
	Error: This pattern matches values of type string message
	       but a pattern was expected which matches values of type
	         Partial message
	       Type string is not compatible with type Partial

I am using an OCaml feature called [generalized algebraic
datatypes](https://caml.inria.fr/pub/docs/manual-ocaml/gadts.html), abbreviated
as GADT. Rather than explain what these are I will contrast this use case
with an approach using a "normal" sum type. Because the server can receive
any kind of request from the client, we need a function that can react to
any message, like `handler` above, and we need a type that includes all of
these message types. We could define a sum type like this, assuming there is a
module for each message with a type `t` defined for the message representation:

	type message =
	  | Tversion of Tversion.t
	  | Rversion of Rversion.t
	  | Tauth of Tauth.t
	  | Rauth of Rauth.t

	let parse iov =
	  let code = Iovec.(to_int (trunc (seek iov 4) 1)) in
	  match code with
	  | 100 -> Tversion (Tversion.parse iov)
	  | 101 -> Rversion (Rversion.parse iov)
	  | 102 -> Tauth (Tauth.parse iov)
	  | 103 -> Rauth (Rauth.parse iov)

Because each variant has a payload, this expression:

	Tversion (Tversion.parse iov)

allocates a block on the OCaml heap containing a tag for the `Tversion` variant
and the actual `Tversion.t` representation. In this case, the messages are
represented as `Iovec.t` values which are fairly lightweight, but each
allocation is an opportunity for the garbage collector to do work and introduce
delay.

To avoid this, we can define what is called a "singleton type" as a GADT:

	type _ message =
	  | Tversion : Tversion.t message
	  | Rversion : Rversion.t message
	  | Tauth : Tauth.t message
	  | Rauth : Rauth.t message

Here, the `message` type becomes a parameterized type and the variants no
longer have payloads. Internally they can be represented as a simple integer.
This is also true of normal variants without payloads, but in the case of GADTs
we can use the type parameters in function signatures to put constraints on any
parameter. This essentially allows us to separate a message's type from its
representation. We have another problem, though; OCaml functions can only
return a single value. We can use a tuple like this:

	let parse iov =
	  let code = Iovec.(to_int (trunc (seek iov 4) 1)) in
	  match code with
	  | 100 -> Tversion, Tversion.parse iov
	  | 101 -> Rversion, Rversion.parse iov
	  | 102 -> Tauth, Tauth.parse iov
	  | 103 -> Rauth, Rauth.parse iov

but this has the exact same problem; constructing the tuple requires an
allocation. We're no better off. Since functions can take multiple arguments,
we can pass in a function to process the results instead:

	let parse iov handle =
	  let code = Iovec.(to_int (trunc (seek iov 4) 1)) in
	  match code with
	  | 100 -> handle Tversion (Tversion.parse iov)
	  | 101 -> handle Rversion (Rversion.parse iov)
	  | 102 -> handle Tauth (Tauth.parse iov)
	  | 103 -> handle Rauth (Rauth.parse iov)

The code above won't compile without additional changes. The biggest thing that
tripped me up was that I could not find a way to define the `handle` parameter,
because locally abstract type declarations are only valid for the outer
function; something like this:

	let parse : Iovec.t -> (type m. m message -> m -> 'a) -> 'a = ...

is illegal; the `type m.` declaration can't go there. The workaround is to
define the parameter as a record type:

	type 'a handler = {f: 't. 't message -> 't -> 'a} [@@unboxed]

Because this is a record type with a single field, the `[@@unboxed]` annotation
can be used to tell the compiler to represent values of this type as the type
of their single field instead. As a result, this accomplishes exactly what we
want at the cost of a different syntax, but no runtime cost. The `parse`
function then becomes:

	let parse iov {f} =
	  let code = Iovec.(to_int (trunc (seek iov 4) 1)) in
	  match code with
	  | 100 -> f Tversion (Tversion.of_iovec iov)
	  | 101 -> f Rversion (Rversion.of_iovec iov)
	  | 102 -> f Tauth (Tauth.of_iovec iov)
	  | 103 -> f Rauth (Rauth.of_iovec iov)

This style of minimizing or eliminating
allocations is described in the semi-serious blog post
[ASM.ocaml](https://www.ocamlpro.com/2016/04/01/asm-ocaml/). An astute
reader might notice that, while I may be avoiding allocations here, I'm still
making allocations for the messages themselves. Moreover, the `Filesystem`
callbacks are using a `result` type which will require allocations.

This is absolutely a premature optimization on my part. I have exactly zero
evidence at this point that allocations during protocol parsing are going to be
a problem. I am simply going off of hearsay and intuition. I decided not to go
after additional allocations because I wasn't ready to write the whole thing in
Continuation-passing style without knowing if it is even worth the trouble. The
change to the parsing function could be done without affecting the rest of the
program too much. I will test my assumptions in the next article.
