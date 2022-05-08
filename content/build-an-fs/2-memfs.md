+++
date = "2021-04-11T06:00:00-05:00"
title = "Let's build a high performance file system! Part 2"
tags = ["ocaml", "plan9", "9P", "fs", "performance", "programming"]
description = "An in-memory file system"
+++

In the [last article](1-build-literally-anything) I implemented a toy file
system that just provided the special files `/dev/zero` and `/dev/null`.
In this article, I will implement a complete in-memory file system. The end
goal of this series is to implement a file system that persists to durable
storage. The intent of this intermediate step is to test the API I've
constructed so far, and to look for any future bottlenecks in the 9P client.

Because this is an intermediate step, I will try to stay away from any
optimizations or API changes that will not make sense when writing to
durable storage.

In previous articles, I defined a module type `Styx.Server`, which looks
like this:

	module type Filesystem = sig
	  type t
	  type file
	  val lookup : file -> string -> (file, error) result
	  val attr : file -> Qid.t -> unit
	  val iounit : file -> int
	  val attach : t -> uname:string -> aname:string -> (file, error) result
	  val openfile : file -> flags:Flag.t -> (file, error) result
	  val create : file -> name:string -> flags:Flag.t -> (file, error) result
	  val read : file -> offset:int -> buf:Iovec.t -> (int, error) result
	  val write : file -> offset:int -> buf:Iovec.t -> (int, error) result
	  val close : file -> (unit, error) result
	  val remove : file -> (unit, error) result
	  val stat : file -> Iovec.t -> (int, error) result
	  val wstat : file -> Stat.t -> (unit, error) result
	  val cleanup : t -> (unit, error) result
	end

In this article, I'll describe and test a `Filesystem` implementation that
stores files in memory, called `memfs`. By keeping the filesystem in-memory
I hope to defer most optimizations of the filesystem's data structures and
identify future bottlenecks that could be harder to address later on.

A file can be a file or a directory:

	type file =
	  | File of { stat : Styx.Stat.t; mutable data : Iovec.t }
	  | Dir of { stat : Styx.Stat.t; mutable entries : file list }

A filesystem is just a file representing the root directory.
