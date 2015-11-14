+++
date = "2015-09-26T12:00:00-05:00"
title = "Writing a 9P server from scratch: Setting Up"
tags = ["go", "plan9"]
draft = true
+++

In writing a 9P server, I'm choosing to work from the bottom up, by
first writing a package to parse and produce 9P messages. I think this
is a good place to start because I want the protocol to influence the
design of the higher-level packages. In my opinion, a good API should
not abstract away the important details of the underlying technology.

## Notes on versioning

Throughout this series of posts, I will refer to the 9P filesystem protocol
as 9P2000, 9P, and styx interchangably. These are all versions of the same
protocol, of which 9P2000 is the latest. To keep it short and sweet,

- 9P was the original protocol released with Plan 9
- Styx was an extension of 9P, introduced with the Inferno operating system
- Most of the Styx changes were rolled into 9P2000 with the 4th edition of Plan 9. Inferno was updated to use 9P2000. All modern systems use the 9P2000 protocol, and Styx is synonymous with 9P2000.

See [9p.cat-v.org/faq](http://9p.cat-v.org/faq) for more details.

## Reference Materials

Before writing any code, we need to gather any documentation regarding
the protocol. While usually I would look for a formal specification
or RFC, there is no 9P specification that is maintained by a standards
body. Instead, the best place to start is the (well written) man pages
from section 5 of the Plan 9 manual. It introduces the 9P message format
and has a section for each of the 14 message types that make up the
9P2000 protocol.

While the documentation for 9P is pretty complete, it is helpful to look
at different implementations to get a clearer view on how it is meant
to be used. In addition to the [reference implementation](https://github.com/9fans/plan9port/tree/master/src/lib9p) (the link is for plan9port, which has minimal differences to plan9), there are several other implementations with varying approaches:

- libixp, originally written for the wmii window manager, is a 
  tiny 9P client/server library.
- diod, a high-performance 9P fileserver, has an implementation
  of 9P with linux-centric extensions, called 9P2000.L
- go9p is a Go package for writing 9P file servers and clients.

## Sample data

In addition to formal documentation, it is useful to get packet captures
of various protocol operations. This will allow us to setup automated
tests and establish a feedback loop, which is an important tool for
developing quickly.

To capture sample data, we will use `u9fs` which is a simple 9P file
server, socat (while it is capable of much more, it is used as an inetd
replacement), and tcpflow, a tool for sniffing tcp streams. In one shell,
I executed the following command line to run my 9P server:

	socat TCP4-LISTEN:5640,range=127.0.0.1/32 EXEC:"./u9fs -a none -u `whoami` /tmp/9root"

In another, I started up tcpflow to capture the data:

	sudo tcpflow -i lo port 5640

I then mounted the 9P file system using the `9pfuse` utility that comes
with plan9port and performed a few operations:

	$ 9pfuse 'tcp!localhost!5640' /n/9tmp
	$ cd /n/9tmp
	$ ls
	50057-0.txt  pg1232.txt   pg16328.txt  pg74.txt
	50059-0.txt  pg16247.txt  pg1661.txt   pg76.txt
	$ mkdir hello-world
	$ echo foo > hello-world/msg
	$ sed 90q pg74.txt >/dev/null
	$ cd
	$ fusermount -u /n/9tmp

`tcpflow` captured the tcp stream and wrote it to two files, one for each
direction. Here are the first few bits of data sent from the client to
the server:

	00000000  13 00 00 00 64 ff ff 00  20 00 00 06 00 39 50 32  |....d... ....9P2|
	00000010  30 30 30 18 00 00 00 68  00 00 00 00 00 00 ff ff  |000....h........|
	00000020  ff ff 05 00 64 72 6f 79  6f 00 00 19 00 00 00 6e  |....droyo......n|
	00000030  00 00 00 00 00 00 01 00  00 00 01 00 06 00 2e 54  |...............T|

Later on, when the tests become more sophisticated, I will record sessions
for each type of operation, and have individual tests that verify the input
and output of each routine.
