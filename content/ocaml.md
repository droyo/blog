+++
date = "2020-05-10T06:00:00Z"
title = "Learning Ocaml"
tags = ["ocaml", "programming"]
description = ""
draft = true
+++

I've decided to learn [Ocaml](https://ocaml.org/). This document is more of
a running list of losses and victories that I keep as I learn.


# Things I like

# Things I love

# Things I don't like

## LWT is viral

[LWT](https://ocsigen.org/lwt/) is a library implementing cooperative
multi-tasking. It is the most common way for ocaml packages and programs
to provide concurrency. An `Lwt.t` type is a promise for work to be
done later. You "bind" a function to it that will be called when the
work is done. It is effectively an implementation of [futures and
promises](https://en.wikipedia.org/wiki/Futures_and_promises).

There's nothing wrong with that. What I dislike is that when you use Lwt,
it permeates your API. Your functions now return `Lwt.t` types, and consumers
of your API are forced to use and understand Lwt. Here is a list of libraries
under Lwt version 5.2.0:

- `Lwt`
- `Lwt_bytes`
- `Lwt_chan`
- `Lwt_condition`
- `Lwt_daemon`
- `Lwt_engine`
- `Lwt_gc`
- `Lwt_glib`
- `Lwt_io`
- `Lwt_list`
- `Lwt_log`
- `Lwt_log_core`
- `Lwt_log_rules`
- `Lwt_main`
- `Lwt_mutex`
- `Lwt_mvar`
- `Lwt_pool`
- `Lwt_pqueue`
- `Lwt_preemptive`
- `Lwt_process`
- `Lwt_react`
- `Lwt_result`
- `Lwt_sequence`
- `Lwt_ssl`
- `Lwt_stream`
- `Lwt_switch`
- `Lwt_sys`
- `Lwt_throttle`
- `Lwt_timeout`
- `Lwt_unix`

Most of these make sense and are necessary. However, they demonstrate the
drawback of bolting event-based programming onto the language -- you need
replacements or wrappers for any function that could potentially block,
to give the Lwt main thread the opportunity to service other events.

## Fragmented or under-documented tooling and libraries

In addition to the standard library provided with OCaml, I found three
replacements for or extensions of the standard library:

* [Core](https://opensource.janestreet.com/core/)
* [Containers](https://c-cube.github.io/ocaml-containers/)
* [Batteries](http://batteries.forge.ocamlcore.org/)

among them, `Core` appears to be the most popular. I don't want to
sound ungrateful that these very large undertakings are shared and
maintained. However, I wish that they had been added to the standard library,
with minimal dependencies between modules. In my experience with other
language, building a solid understanding of the standard library was a big
step in becoming productive with the language and being able to read code
quickly. With the current situation, I have additional standard libraries
to learn.

It doesn't help that the `Core` library doesn't run on all the platforms OCaml
can run (namely Windows), so if I want to target Windows I have to make sure
to avoid `Core` and avoid depending on anything that may depend on `Core`.

# Things that drove me nuts

