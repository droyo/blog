+++
date = "2015-09-27T12:00:00-05:00"
title = "Writing a 9P server from scratch: Getting Started"
tags = ["go", "plan9"]
draft = true
+++

We have done enough research and planning, let's get to coding!
I have decided on a namespace and separation for my Go packages:

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

