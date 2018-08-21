+++
date = "2016-08-20T12:00:00-05:00"
title = "Writing a 9P server from scratch, pt 3: server plumbing"
tags = ["9P"]
description = "Walking and talking like a 9P server"
+++

In my [previous post][prev], I covered the implementation of a decoder and
encoder for 9P messages. However, if that was all it took to implement a 9P
server, 9P would not be a protocol; it would be a file format. In this post, I
will cover the implementation of the [net/styx][pkg] package, which provides
plumbing for writing a 9P server. This is not a hypothetical package; it is
used to implement [jsonfs][jsonfs], a 9P file server that serves a
JSON-formatted file as a file system.

# 9P transactions

A typical 9P session consists of pairs of *T-messages* and *R-messages*
that represent requests from the client and their responses from the server,
respectively. Here is one such transaction, used to open a file:

<img src="/img/9p-transaction.svg" style="width: 100%"/>

The first three fields (size, type, and tag) are purely bookkeeping
used to identify and associate a request with its response. Ignoring
these fields, we have what looks very much like a remote procedure
call:

	open(fid, flag) -> (qid, iounit)

This simple example exposes a few important details that we will
need to keep in mind:

- **Ordered**: 9P requires a transport that guarantees in-order
  transmission of messages. Because each session is essentially a sequence
  of RPCs, re-arranging said calls could have unexpected results. Also,
  because future RPCs can depend on the return value of prior RPCs, parallelizing
  or pipelining a large number of requests can prove challenging (but not impossible!)
- **Synchronous**: While it is valid in 9P to have multiple requests
  "in flight", the fact that there is a 1-1 mapping between every
  T-message and an R-message means each individual request blocks until
  the server sends a response, and there are guarantees about the
  state of the world when a client receives a response. For instance,
  when an `Rclunk` message is received, the client is free to re-use
  the file handle in question.

# Notes on identifiers

There are 3 types of identifiers in the 9P protocol.

- **Tag**: 16-bit identifiers for each transaction. Chosen by the
  client. No two unanswered *T-messages* may have the same tag.
- **Fid**: 32-bit identifier for a file pointer, akin to Unix file
  descriptors.  Chosen by the client. **Not unique**; two fids may point
  to the same file.
- **Qid**: 104-bit identifier for a file, analogous to (but not the same as)
  an inode number in a Unix filesystem. Chosen by the **server**. No
  two files may have the same Qid, even if they have the same name and
  one has been deleted.

To ensure our server is suitable for public, anonymous use, we should
pay special attention to the identifiers that are chosen by the client,
as it affects how we store them. If we had control over fids, for instance,
we could have a lookup table of open files that used the fid as an index:

	var openFiles []*file
	f := openFiles[m.Fid()]

However, because the server does not control fid (or tag) values, a client
could choose values that are more difficult for the server to handle. For
instance, in the above example, a client could use a fid of 4 billion. This
would either cause a run-time panic, or cause 4GB of memory to be
allocated for the connection, making denial-of-service type attacks trivial.

With this in mind, we must store and lookup these identifiers in a data
structure that uses the same amount of resources for any possible value.
There are plenty of candidates, such a sorted slice or a balanced tree,
but right at the top of the list is a simple `map`. Note that Go's 
implementation of maps [is resistant to collision-based attacks][hash].

# The main loop

Here is the main loop of our server. This is very similar to any other
Listen/Accept server you would write in Go.

	for {
		rwc, err := l.Accept()
		if err != nil {
			return err
		}

		conn := newConn(srv, rwc)
		go conn.serve()
	}

We spawn one goroutine per connection. Here is what a connection
looks like:

	type conn struct {
		*styxproto.Decoder
		*styxproto.Encoder
		msize int64
		sessionFid map[uint32]*Session
		qidpool *qidpool.Pool
		pendingReq map[uint16]context.CancelFunc
	}

In the [actual implementation][conn], there are a few more fields
and the two maps are replaced with thread-safe wrappers. The
embedded Decoder and Encoders read and write 9P messages on
the underlying connection. Two maps are used to lookup the session
a file is associated with (more on sessions later) and to lookup in-flight
requests that may be cancelled via a `Tflush` request. Unique qids are
retrieved for files on-demand.

The main loop of a connection looks something like this:

	func (c *conn) serve() {
		defer c.close()
	
		if !c.acceptTversion() {
			return
		}
	
		for c.Next() && c.Encoder.Err() == nil {
			if !c.handleMessage(c.Msg()) {
				break
			}
		}
		if err := c.Encoder.Err(); err != nil {
			c.srv.logf("write error: %s", err)
		}
		if err := c.Decoder.Err(); err != nil {
			c.srv.logf("read error: %s", err)
		}
		c.srv.logf("closed connection from %s", c.remoteAddr())
	}

Version negotiation must be the first transaction made, and looks
like this:

	for c.Next() {
		for _, m := range c.Messages() {
			tver, ok := m.(styxproto.Tversion)
			if !ok {
				c.Rerror(m.Tag(), "need Tversion")
				return false
			}
			msize := tver.Msize()
			if msize < styxproto.MinBufSize {
				c.Rerror(m.Tag(), "buffer too small")
				return false
			}
			if msize < c.msize {
				c.msize = msize
				c.Encoder.MaxSize = msize
				c.Decoder.MaxSize = msize
			}
			if !bytes.HasPrefix(tver.Version(), []byte("9P2000")) {
				c.Rversion(uint32(c.msize), "unknown")
			}
			c.Rversion(uint32(c.msize), "9P2000")
			return true
		}
	}

In plain english, the client proposes a version, and the server responds
with "unknown" until the client proposes a 9P2000 variant, after which the
server, which has the final say, forces the client to use 9P2000. Here is the
handler for all other messages:

	func (c *conn) handleMessage(m styxproto.Msg) bool {
		if _, ok := c.pendingReq[m.Tag()]; ok {
			c.Rerror(m.Tag(), "%s", errTagInUse)
			return false
		}
		cx, cancel := context.WithCancel(c.cx)
		c.pendingReq[m.Tag()] = cancel
	
		switch m := m.(type) {
		case styxproto.Tauth:
			return c.handleTauth(cx, m)
		case styxproto.Tattach:
			return c.handleTattach(cx, m)
		case styxproto.Tflush:
			return c.handleTflush(cx, m)
		case fcall:
			return c.handleFcall(cx, m)
		case styxproto.BadMessage:
			c.clearTag(m.Tag())
			c.Rerror(m.Tag(), "bad message: %s", m.Err)
			return true
		default:
			c.Rerror(m.Tag(), "unexpected %T message", m)
			return false
		}
		return true
	}

Each request type is passed to its own handler that is specific to
that type of transaction. You may be wondering what an `fcall`
is:

	type fcall interface {
		styxproto.Msg
		Fid() uint32
	}

Remember from the [previous][prev] post that our representation of 9P
messages in the [styxproto][styxproto] package is just a slice of bytes
with methods for the fields, with a 1-1 mapping from a message to a Go
type. Using that knowledge, we can create interfaces that select common
classes of 9P messages.  In this case, an `fcall` is a 9P message that
operates on a file, pointed to by a fid. An fcall is special, because
it is part of a session.

# 9P Sessions

9P allows for multiple sessions to be multiplexed over a single connection.
A session is established with an `attach` call:

	attach(fid, afid, user, aname) -> qid

This call establishes `fid` as a file handle for the root of a filesystem
tree `aname` (usually the empty string). After an attach, all operations
on `fid` will be associated with the user named in the attach. The afid
argument has to do with authentication and will be explained later.

It may be somewhat surprising to learn that there is no "session ID" in
9P, especially coming from an HTTP mindset, where session cookies
abound. There is no need; in 9P, sessions are implicit, not explicit. Any
fid can be traced back to the `attach` call that established its session.
This is due to a transaction we haven't covered yet, `walk`:

	walk(fid, newfid, path ...) -> qid ...

The `walk` transaction is used to move around a directory hierarchy; think
of it a more general version of what happens when you do `cd path/to/dir`
on a normal file system. Note that the first argument to `walk` is an
already established fid; the walk is relative to that file. Other than
`attach`, **`walk` is the only way to establish a new fid.** This is why
we can always use the ancestry of a fid to determine the session it is
associated with. In practical terms, this is what the `sessionFid` map
in the `conn` structure is for.

In the `styx` package, there will be one managing goroutine per session.
This goroutine is created when the server handles a `Tattach` message.

	func (c *conn) handleTattach(ctx context.Context, m styxproto.Tattach) bool {
		defer c.Flush()
		s := newSession(c, m)
		go func() {
			c.srv.Handler.Serve9P(s)
			s.cleanupHandler()
		}()
		c.sessionFid[m.Fid()] = s
		s.IncRef()
		s.files.Put(m.Fid(), file{name: ".", rwc: nil})
		c.clearTag(m.Tag())
		c.Rattach(m.Tag(), c.qid(".", styxproto.QTDIR))
		return true
	}

All sessions must have at least one file associated with them. After an
`attach` transaction, that file is the root directory. Reference counting is
then used to detect when a session is finished and notify the session
handler. The `cleanupHandler` method closes all open files on a 
connection if its handler exits prematurely.

# I/O

It is reasonable to say that the primary goal of a 9P session is to read
from and write to files. Here are the `read` and `write` transactions:

	read(fid, offset, count) -> (count, data)
	write(fid, offset, count, data) -> count

Note the `offset` field: in 9P, it is not the server's responsibility to
keep track of the current position in the file; a client must specify
the offset for read and write operations every time. In Go, this is
very similar to the `io.ReaderAt` and `io.WriterAt` interfaces. One
solution, then, is to define an interface that all files must meet in
order to be served by the `styx` package. The [styxfile][styxfile]
package does just that, with `styxfile.Interface`:

	package styxfile
	type Interface interface{
		ReadAt(p []byte, offset int64) (int, error)
		WriteAt(p []byte, offset int64) (int, error)
		Close() error
	}
	
The [actual type][styxfile.Interface] just names `io.ReaderAt` & co
rather than copying their definitions. Here is how we store these files,
in a map keyed by their fids:

	type file struct {
		rwc styxfile.Interface
		name string
	}

Here is one way we can handle the `read` transaction

	func (s *Session) handleTread(msg styxproto.Tread, f file) bool {
		if f.rwc == nil {
			s.conn.clearTag(msg.Tag())
			s.conn.Rerror(msg.Tag(), "fid %d not open for I/O", msg.Fid())
			return false
		}
		count := min(
			int(msg.Count()), 
			int(s.conn.MaxSize - styxproto.IOHeaderSize))
		buf := make([]byte, count)
		
		n, err := f.rwc.ReadAt(buf, msg.Offset())
		s.conn.clearTag(msg.Tag())

		switch err {
		case nil,io.EOF,io.ErrUnexpectedEOF:
			s.conn.Rread(msg.Tag(), buf[:n])
		default:
			s.conn.Rerror(msg.Tag(), "%s", err)
		}
		return true
	}

There's a lot here, so I'll step through it piece by piece.

	if f.rwc == nil {
		s.conn.clearTag(msg.Tag())
		s.conn.Rerror(msg.Tag(), "fid %d not open for I/O", msg.Fid())
		return false
	}

Before a file may be read from, it must be prepared for I/O using the
`open` transaction:

	open(fid, flags) -> (qid, iounit)

Our message handler for this transaction sets the file's `rwc` field
appropriately. All the message handlers in the `styx` package return
true if the session can continue or false if it should be ended.

	count := min(
		int(msg.Count()), 
		int(s.conn.MaxSize - styxproto.IOHeaderSize))
	buf := make([]byte, count)

Here we allocate a buffer, taking care not to let the client DOS us by
requesting a large buffer. This buffer will hold the results of reading
from the file.

I would have liked to avoid using a temporary buffer here. However,
there is very little we can do to avoid this, because we must know
how much data is available before writing an `Rread` message. One
possible solution would be to introduce buffering into the `styxfile.Encoder`
and implement a method that takes an `io.WriterTo`, or implement a
wrapper type that implements `io.ReaderFrom`. However, this is a non-trivial
amount of work and must be justified by measurement.

	n, err := f.rwc.ReadAt(buf, msg.Offset())

Here, we're finally reading the data from the file.

	s.conn.clearTag(msg.Tag())

The server should always clear a tag *before* sending its response,
the reason being that once a client sees a response to a given
message, it is allowed to re-use the tag immediately for new transactions.
See [this commit][cleartag] where I fixed an issue that arose when
using v9fs, where the client was fast enough to send a new request after
the server responded to the old one, but before it cleared the tag for re-use.

The following lines were put together after some trial and error on my part.

	switch err {
	case nil,io.EOF,io.ErrUnexpectedEOF:
		s.conn.Rread(msg.Tag(), buf[:n])
	default:
		s.conn.Rerror(msg.Tag(), "%s", err)
	}
	return true

In a traditional Unix filesystem, when you read from a file and you've
reached the end, further reads will return an error, usually called `EOF`
(end-of-file). If we were to copy this behavior in our 9P server, we would
do something like this:

	if err == nil || n > 0 {
		s.conn.Rread(msg.Tag(), buf[:n])
	} else {
		s.conn.Rerror(msg.Tag(), "%s", err)
	}

However, this is not what is done in practice. Instead, the way to signal
an end-of-file condition in 9P is to send a 0-length `Rread` response.
The client will note that nothing was returned and discern EOF that way.
This behavior is not explicit in the documentation for 9P, but was
observed after testing several clients against my server. See [this
commit][eof] for more details. In retrospect, this behavior is cleaner
than using a sentinel error value; It has always felt kind of wrong to
me that EOF is considered an "error" given its inevitability (many files
have a finite length), and doing it this way means clients do not have
to parse error strings to discern EOF from other errors.

# Using more than just ReadAt/WriteAt

While `*os.File` implements `styxfile.Interface`, many very useful
types do not implement `ReadAt` or `WriteAt` methods. Sometimes they
are not implemented because it was not deemed necessary.  Sometimes it
is impossible; when you consider that these methods are essentially a
more flexible, client-side `seek`, it does not really make sense to call
`seek` on sockets, fifos, pipes, and other pseudo-files.  We really want
server authors to have the flexibility to provide any kind of byte stream
as a file. With this in mind, I created the [styxfile.New][styxfile.New]
function, which takes an `interface{}` and picks a wrapper type to fill
in mising functionality:

	func New(rwc interface{}) (Interface, error) {
		switch rwc := rwc.(type) {
		case Interface:
			return rwc, nil
		case interfaceWithoutClose:
			return nopCloser{rwc}, nil
		case io.Seeker:
			return &seekerAt{rwc: rwc}, nil
		case io.Reader:
			return &dumbPipe{rwc: rwc}, nil
		case io.Writer:
			return &dumbPipe{rwc: rwc}, nil
		}
		return nil, fmt.Errorf(
			"Cannot convert %T to styxfile.Interface", rwc)
	}

For types that implement `Seek`, we can implement `ReaderAt` and `WriterAt`
by seeking to the offset before calling `Read` or `Write`. This must be protected
by a mutex.

For types that only allow `Read` or `Write`, we can track the current offset
in the stream and return an error if a client attempts to write or read elsewhere:

	type dumbPipe struct {
		rwc    interface{}
		offset int64
		sync.Mutex
	}
	
	func (dp *dumbPipe) ReadAt(p []byte, offset int64) (int, error) {
		r, ok := dp.rwc.(io.Reader)
		if !ok {
			return 0, ErrNotSupported
		}
		dp.Lock()
		defer dp.Unlock()
	
		if dp.offset != offset {
			return 0, ErrNoSeek
		}
	
		n, err := io.ReadFull(r, p)
		dp.offset += int64(n)
		return n, err
	}

By limiting the amount of tedious work server code has to do, we
also reduce the chance of mistakes; it is not entirely trivial to implement
`ReadAt` and `WriteAt`, and it is better to do it once than force users
to repeat it over and over again.

# Directory listings

In 9P, directories are not special; they are simply files full of
`Stat` structures, which are documented [here][stat]. A client
can list the contents of a directory by reading it, as if it were
any other file.

We could stipulate that server programs must marshal the contents
of directories into `styxproto.Stat` structures. But this is usually
too much work. Instead, we can lift the `Readdir` method from
`*os.File` and use it to define a new interface:

	type Directory interface {
	    Readdir(n int) ([]os.FileInfo, error)
	}

Then, we provide the `styxfile.NewDir` function that turns any
type that meets the above interface into a `styxfile.Interface`
which automatically translates `os.FileInfo` values into
`styxproto.Stat` structures. Here are the important bits of
that wrapper type:

	type dirReader struct {
		Directory
		offset int64
		sync.Mutex
		pool *qidpool.Pool
		path string
	}

	func (d *dirReader) ReadAt(p []byte, offset int64) (int, error) {
		d.Lock()
		defer d.Unlock()
	
		if offset != d.offset {
			return 0, ErrNoSeek
		}
		nstats := len(p) / styxproto.MaxStatLen
		if nstats == 0 {
			return 0, ErrSmallRead
		}
	
		fi, err := d.Readdir(nstats)
		n, marshalErr := marshalStats(p, fi, d.path, d.pool)
		d.offset += int64(n)
		if marshalErr != nil {
			return n, marshalErr
		}
		return n, err
	}

Translating an `os.FileInfo` value into a `styxproto.Stat` structure
can be somewhat OS-specific when dealing with real files, especially
when determining ownership of a file. 9P uses string identifiers for
user and group names, not the uid/gid numbers that most people
are used to. While I think this was a good design choice, having had
to deal with several id/name mismatches on NFSv3 exports in my career,
it also means we have to resolve the UID/GID of the `syscall.Stat_t`
structure returned by `FileInfo.Sys()` for real files on unix systems.

The [sys][syspkg] package contains the non-portable code required to
determine ownership of a file.

# Authentication

The 9P protocol does not prescribe an authentication method. Instead,
client and server communicate by reading from and writing to a special
file. The handle to this file is established with the `auth` transaction:

	auth(afid, uname, aname) -> qid

Client and server may then carry out nearly any authentication protocol.
This helps 9P stay current, as it can take in more authentication methods
as they arise. Once the authentication protocol is complete, a client can
use the afid as an authentication token in subsequent `attach` transactions.

To implement authentication, a server must provide the following function:

	type AuthFunc func(rwc io.ReadWriteCloser, user, aname string) error

From the server side, a read on `rwc` blocks until a `Twrite` request
comes from the client, and a write on `rwc` blocks until a `Tread`
request comes, then the data is passed to the client. In this way, we hide
the details of the marshalling and processing of 9P messages from the
authentication function, and in effect give it a private bi-directional
channel with the client that is tunnelled over 9P. This was suprisingly
easy to implement using Go's `net.Pipe` function. The implementation
is [here][tauth]. The [styxauth](https://aqwari.net/net/styx/styxauth)
package provides a few authentication functions. Note that the actual
`styx.AuthFunc` has access to the underlying network connection,
allowing it to use transport-based authentication, as demonstrated by
the `SocketPeerID` and `TLSSubjectCN` functions.

# Tracing 9P requests

Being able to see the messages as they come in and go out is invaluable
when troubleshooting errors. When testing the `styx` package, I found
I made a number of incorrect assumptions or misread the documentation
in a few key places. It would have taken me ages to find the problem if
I could not see the messages coming in to the server and going out.

I wrote the internal [tracing][tracing] package for this purpose. It
provides wrappers for `styxproto.Decoder` and `styxproto.Encoder` that
allow us to peek at messages as they pass through. Because of our
choice not to unmarshal messages, but leave them as they are, this
was surprisingly easy and, at a glance, should not incur an unreasonable
performance penalty. I added a `Write` function to the `styxproto`
package:

	func Write(w io.Writer, m Msg) (written int64, err error) {
		n, err := w.Write(m.bytes())
		if r, ok := m.(io.Reader); ok {
			written, err = io.Copy(w, r)
			return written + int64(n), err
		}
		return int64(n), err
	}

Then, implementing tracing was simply a matter of stacking
encoders/decoders together with `io.Pipe()`:

	func Decoder(r io.Reader, fn TraceFn) *styxproto.Decoder {
		rd, wr := io.Pipe()
		decoderInput := styxproto.NewDecoderSize(r, 8*kilobyte)
		decoderTrace := styxproto.NewDecoderSize(rd, 8*kilobyte)
		go func() {
			for decoderInput.Next() {
				for _, m := range decoderInput.Messages() {
					fn(m)
					styxproto.Write(wr, m)
				}
			}
			wr.Close()
		}()
		return decoderTrace
	}
	func Encoder(w io.Writer, fn TraceFn) *styxproto.Encoder {
		rd, wr := io.Pipe()
		encoder := styxproto.NewEncoder(wr)
		decoder := styxproto.NewDecoderSize(rd, 8*kilobyte)
		go func() {
			for decoder.Next() {
				for _, m := range decoder.Messages() {
					fn(m)
					styxproto.Write(w, m)
				}
			}
		}()
		return encoder
	}

In the `styx` package, tracing is accessed by setting the `TraceLog`
member on the `Server` structure. The output looks like this:

	→ 65535 Tversion msize=8192 version="9P2000"
	← 65535 Rversion msize=8192 version="9P2000"
	→ 000 Tattach fid=1 afid=NOFID uname="droyo" aname=""
	← 000 Rattach qid="type=128 ver=0 path=1"
	→ 000 Twalk fid=1 newfid=2 "apiVersion"
	← 000 Rwalk wqid="type=0 ver=0 path=2"
	→ 000 Topen fid=2 mode=0
	← 000 Ropen qid="type=0 ver=0 path=2" iounit=0
	→ 000 Tread fid=2 offset=0 count=8168
	← 000 Rread count=3
	→ 000 Tread fid=2 offset=3 count=8168
	← 000 Rread count=0 


# Interfacing with user code

So far, all that we have covered a lot of plumbing, that shuffles
messages along to their appropriate handlers. We are able to handle
many bookkeeping transactions like `flush`, `clunk`, and `attach`
without asking the user code for help. However, when it comes to the
important stuff, file IO and directory walking, we need to ask the
user code for help. Exactly how we do that will be the topic of the
next post in this series, as this post has already gotten too long.
We will cover the server API of the `styx` package using a toy
server, [jsonfs][jsonfs].

[cleartag]: https://github.com/droyo/styx/commit/0cb8c0aa712ef9579aedb9941b61263b981b98a2
[conn]: https://github.com/droyo/styx/blob/0cb8c0aa712ef9579aedb9941b61263b981b98a2/conn.go#L44
[eof]: https://github.com/droyo/styx/commit/2b3641b405a4074cabe891e830bae2f7e31461af
[hash]: https://github.com/golang/go/issues/9365#issuecomment-67327819
[jsonfs]: https://github.com/droyo/jsonfs
[pkg]: https://aqwari.net/net/styx
[prev]: ../parsing/
[SectionReader]: https://golang.org/pkg/io/#SectionReader
[styxfile]: https://aqwari.net/net/styx/internal/styxfile
[styxfile.Interface]: https://godoc.org/aqwari.net/net/styx/internal/styxfile#Interface
[styxfile.New]: https://github.com/droyo/styx/blob/0cb8c0aa712ef9579aedb9941b61263b981b98a2/internal/styxfile/file.go#L49
[styxproto]: https://aqwari.net/net/styx/styxproto
[tauth]: https://github.com/droyo/styx/blob/0cb8c0aa712ef9579aedb9941b61263b981b98a2/conn.go#L263
[stat]: https://swtch.com/plan9port/man/man9/stat.html
[syspkg]: https://aqwari.net/net/styx/internal/sys
[tracing]: https://aqwari.net/net/styx/internal/tracing
