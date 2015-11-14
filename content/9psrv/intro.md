+++
date = "2015-09-25T12:00:00-05:00"
title = "Writing a 9P server from scratch"
tags = ["go", "plan9"]
draft = true
+++

9P is a network protocol for serving file systems. It was created by the
inventors of Unix as part of Plan 9, a new operating system that tried
to be more Unix than Unix. I am not terribly familiar with any network
file system protocols, so any judgements or observations I can offer
will be incomplete. However, with that caveat, I will note that the 9P
protocol, at the very least, is refreshingly small and simple. There are
messages for CRUD operations on files, plus a message to walk the file
hierarchy, and a handful of session management messages.  That's it! Even
authentication is implemented as a series of read/write operations on
a special file; the protocol itself is agnostic about the authentication.

There is not much to 9P. However, what was interesting about Plan 9 was
how pervasive 9P was. Everything really *was* a file, and all files were
accessed via 9P, whether they were local or remote. The file system became
a user interface; traditional programs like `lftp` were replaced with a
synthetic file system that translated 9P operations into ftp calls. To
read the `plan9` newsgroup, you open `/n/nttp/comp/os/plan9`. There are
more creative applications of the file system as a UI in user-contributed
file servers, such as `jirafs`, which lets you browse jira as a file
system.

In an effort to write more substantial programs, I'm going to write
a 9P server from scratch. It is my hope that keeping a journal of the
process will help me write better code, and motivate me to complete the
project. I will be writing the server in Go, which I find to be ideal for
writing network services. To set expectations, this series is not about
Go, it is about network protocols and the design of network services.

While don't hold any illusions about writing better code than what
is currently available, I would like to produce something that others
would consider using.  It is my hope that with the right API and a sound
implementation, I can increase the attractiveness of using 9P when it
is a better fit than other protocols, such as HTTP.

# Goals

I have three goals for this project:

- A low-level package for producing and parsing 9P messages.
- A high-level package for building file servers with a minimum
  of boilerplate.
- An example file server that does something fun and useful.

# Example server - `graphitefs`

For the example server, I will produce a file server UI for graphite,
the popular metrics collection. If you are not familiar with graphite,
it composed of a metrics collection service and an HTTP API for rendering
graphs. Services send newline-delimited metrics, such as

	servers.myhost.network.eth0.rx_bit 982372 1443235316

This is a good fit for a file server because the data is already hierarchical;
replace the dots with slashes and you have

	servers/myhost/network/eth0/rx_bit

To make things interesting, the `graphitefs` file server won't just serve
data; it will render and serve a streaming gif that draws a graph of the
metric in real time.

Throughout this series, I will be talking a lot about caching, exponential
backoff, and different ways to "be nice" to the backend service. I am an
operations engineer at my day job, so writing code that uses shared
services in a responsible manner is important to me.

Enough planning, let's get coding!
