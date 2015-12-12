+++
date = "2015-09-25T12:00:00-05:00"
title = "Writing a 9P server from scratch"
tags = ["go", "plan9", "9P"]
description = "Using the plan9 file system protocol"
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

In writing a 9P server, I'm choosing to work from the bottom up, by first
writing a package to parse and produce 9P messages. I think this is a good
place to start because I want the protocol to influence the design of the
higher-level API. In my opinion, a good API should not abstract away the
important details of the underlying technology.

## Notes on versioning

Throughout this series of posts, I will refer to the 9P filesystem protocol
as 9P2000, 9P, and styx interchangably. These are all versions of the same
protocol, of which 9P2000 is the latest. To keep it short and sweet,

- 9P was the original protocol released with Plan 9
- Styx was an extension of 9P, introduced with the Inferno operating system
- Most of the Styx changes were rolled into 9P2000 with the 4th edition of Plan
  9. Inferno was updated to use 9P2000. All modern systems use the 9P2000
  protocol, and the name "styx" now refers to 9P2000.

See [9p.cat-v.org/faq](http://9p.cat-v.org/faq) for more details.

## Reference Materials

Before writing any code, we need to gather any documentation regarding
the protocol. While usually I would look for a formal specification
or RFC, there is no 9P specification that is maintained by a standards
body. Instead, the best place to start is the (well written) man pages
from section 5 of the Plan 9 manual. It introduces the 9P message format
and has a section for each of the 14 message types that make up the
9P2000 protocol.

While the documentation for 9P is pretty complete, it is helpful to look at
different implementations to get a clearer view on how it is meant to be used.
In addition to the [reference
implementation](https://github.com/9fans/plan9port/tree/master/src/lib9p) (the
link is for plan9port, which has minimal differences to plan9), there are
several other implementations with varying approaches:

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

Later on, when the tests become more sophisticated, I will record sessions for
each type of operation, and have individual tests that verify the input and
output of each routine. We have done enough research and planning, let's get to
coding!  I have decided on a namespace and separation for my Go packages:

- `aqwari.net/net/styx` - high-level Client/Server package
- `aqwari.net/net/styx/styxproto` - low-level protocol package
- `aqwari.net/cmd/nagiosfs` - example file server

I chose the name "styx" because `9p` is not a valid identifier in Go.
The styx protocol and 9P2000 are identical.

# Setting up a feedback loop

When doing programming in any domain I am unfamiliar with, I like to
keep it interactive; some of my most productive work is done at the
Read-eval-print-loop of a Python or Lisp interpreter.  Although there
is currently no compiler available for Go, I can use the fast compile
times of the main implementation to setup a satisfactory feedback loop,
by running `go test` every time a file changes. I have patched Russ Cox's
`Watch` command to run on Linux, and renamed it to `acmewatch`. This
tool runs a command, such as `go test`, in an acme window every time a
directory changes.

In a previous post I showed how to record protocol data for future testing.
I have stored some protocol data from an example 9P session in the
`testdata` directory. The `sample.client.9p` file contains sample messages
coming from the client to the server, and the `sample.server.9p` file contains
messages coming from the server to the client.

We will write two simple tests, that parse all the messages in each file and
log them. If there is a parsing error, the tests fail. We know that the data
we've recorded is valid, so this serves as a good smoke test. By necessity,
we will be making up the API of the `styxproto` package when we write
the tests. This API will change over time as we learn more.

	package styxproto
	
	import "testing"
	
	func TestRequests(t *testing.T) {
		testParseMsgFile(t, "testdata/sample.client.9p")
	}
	
	func TestResponse(t *testing.T) {
		testParseMsgFile(t, "testdata/sample.server.9p")
	}

So far so good. Now we make our first stab at what the overall package
is going to look like. When working with streaming data, I prefer an API
like the `bufio` package's `Scanner` type; a boolean function fetches the
next message and returns `true` unless an error was encountered. This
makes for a nice, compact `for` loop to read all the messages in a buffer.

	func testParseMsg(t *testing.T, r io.Reader) {
		p := newParser(r)
		for p.Next() {
			t.Logf("%s", p.Message())
		}
		if err := p.Err(); err != nil {
			t.Error(err)
		}
	}

Currently the tests will start failing with this error message:

	$ go test
	# aqwari.net/net/styx/styxproto
	./styxproto_test.go:28: undefined: newParser
	FAIL	aqwari.net/net/styx/styxproto [build failed]
	go test: exit status 2

Let's try implementing newParser. All 9P messages start with
the following 7 bytes:

	size[4] type[1] tag[2]

The size is a 32-bit little endian integer that is the size, in bytes,
of the message, including the 4 bytes of the size itself. The type
is the type of message. In the basic 9P2000 protocol, there are
28 message types, two for each operation. The 9P2000.u extensions
add a few more messages, and the 9P2000.L extensions add even
more.

For now, let's just try parsing the type of a message. We need to
define a `Parser` type for `NewParser` to return, plus a `message`
type for the parser's `message` method to return. The layout of
the Parser struct is bound to change in the future; whether or not
we use an internal buffer, how big that buffer should be, etc.
Ideally, I would like to have all messages live adjacent to one
another in a single byte slice, but managing that slice might prove
too complex. For now, we'll give each Parser an internal buffer, and
maintain enough space for at least one message it.

	type Parser struct {
		buf *bytes.Buffer
		r   io.Reader
		err error
		msg Message
	}

	func (p *Parser) Next() bool {
		if p.err != nil {
			return false
		}
		p.buf.Reset()
		if err := copyMsg(p.buf, p.r); err != nil {
			p.err = err
			return false
		}
		if err := validMsg(p.buf.Bytes()); err != nil {
			p.err = err
			return false
		}
		return true
	}
	
	func (p *Parser) Message() Message {
		return p.msg
	}
	
	func (p *Parser) Err() error {
		if p.err == io.EOF {
			return nil
		}
		return p.err
	}

Here are the helper methods `copyMsg` and `parseMsg`,
which read a single message from a stream and validate
a single message, respectively. `parseMsg` will be a much
more robust function in the future. Currently it just checks
that the message has space for a type and a tag.

	// copyMsg does *not* copy the first four bytes
	// into dst
	func copyMsg(dst io.Writer, src io.Reader) error {
		sizebuf := make([]byte, 4)
	
		if _, err := io.ReadFull(src, sizebuf); err != nil {
			return err
		}
		size := int64(binary.LittleEndian.Uint32(sizebuf))
		if _, err := io.CopyN(dst, src, size-4); err != nil {
			return err
		}
		return nil
	}

	func parseMsg(msg []byte) (Message, error) {
		if len(msg) < 3 {
			return nil, errors.New("Message is too small!")
		}
		return Message(msg), nil
	}

Finally, we'll make Message a Stringer so we get readable
output from our test:

	func (msg Message) String() string {
		mtype := msg[0]
		tag := int32(binary.LittleEndian.Uint16(msg[1:3]))
		return fmt.Sprintf("tag %d type %d\n%s",
			tag, mtype, readable(msg))
	}

After that, our tests start passing! Woohoo!

	$ go test -v
	=== RUN   TestRequests
	--- PASS: TestRequests (0.00s)
		styxproto_test.go:30: tag ffff type 100 |. ....9P2000|
		styxproto_test.go:30: tag 0000 type 104 |..........droyo.|
		styxproto_test.go:30: tag 0000 type 110 |.............Tra|
		...
	=== RUN   TestResponse
	--- PASS: TestResponse (0.01s)
		styxproto_test.go:30: tag ffff type 101 |. ....9P2000|
		styxproto_test.go:30: tag 0000 type 105 |....Vy*......|
		styxproto_test.go:30: tag 0000 type 107 |..No such file o|
		...

One thing you notice right away is that aside from the first version
message, which uses NOTAG (0xFFFF) as its tag, all the messages
have the same tag! The 9P protocol requires that for any given
connection, tags for all *pending* requests must be unique. In
the traffic we've recorded, each request finishes before the next
one starts. The most likely cause for this is that we're making
the requests over the local network device. Over high latency
connections, the number of pending requests will go up.

In my next post, I will implement both the low-level protocol
package and a high-level package for making servers and clients.
