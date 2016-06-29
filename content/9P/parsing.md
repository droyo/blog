+++
date = "2015-09-29T12:00:00-05:00"
title = "Writing a 9P server from scratch: Protocol parsing"
tags = ["9p"]
description = "Decoding the 9P message format"
+++

In the previous post, we began work on the protocol parser, getting
to the point where we could read 9P messages from a network
connection. With this post we will pick up the pace, writing a full
decoder and encoder for the 9P protocol. You will see that with even
as simple a protocol as 9P, there are a wide range of choices to make
when designing the API and its implementation.

The code for this post can be found in the [styxproto][3] package. It
is always changing, so do not be surprised if there are discrepancies.

# 9P message format

9P protocol data comes as a series of _messages_, consisting of a
`size` field, an unsigned 32-bit little-endian integer, followed
by `size-4` bytes. There are no sentinel values or delimiters.

<img src="/img/9p-generic-msg.svg" style="width:100%"/>

Within the payload of a message are several fixed-sized and
variable-length fields. The first byte of the payload determines
the message type.

<img src="/img/9p-tmessage.svg" style="width:100%"/>

Variable-length fields, such as the `version` field of a `Tversion`
message, always consist of a 16-bit, little-endian integer `n`, followed
by `n` bytes of UTF8 text. The `Twalk` message is the most complex
message to parse; its body contains a count, `nwname`, followed by
`nwname` variable-length fields of UTF8 text. 

<img src="/img/9p-twalk.svg" style="width:100%"/>

That's it! That's the most complex message we'll have to deal with!
The messages that are of particular note are those used for I/O, specifically
the `Twrite` and `Rread` messages. These write data to and return data
from a file on a server, respectively. The headers of these messages are
not unlike the other messages. 

<img src="/img/9p-iohdr.svg" style="width:100%"/>

The final `count` field is a 32-bit integer specifying the size of
the message body. This means that `Tread` and `Rread` messages can
be very large, close to 4 gigabytes in size. You would be hard-pressed,
however, to find a server in the wild that did not negotiate a maximum
message size _much_ lower than that. Regardless, how to handle these two
little message types is one of the more important design choices we will
face.

# Message representation

As you can see above, the 9P message format is quite simple. Because most
of the fields are of a fixed-width, each field can be accessed quickly. In fact,
this format is so trivial that it almost seems a waste to copy each message
out into a `struct`. I will be taking the same approach used by packages like
[capnproto][0] and [flatbuffers][1]; the protocol representation of a message
will be used directly rather than being unmarshalled into an intermediate
type. I am not going to make predictions on cache hits, because honestly
I wouldn't know what I was talking about. Instead I'll talk about the benefits
I _do_ know about:

- The connection buffers themselves can be used to store the messages while
  they are being processed. With this approach, less work is made for the
  garbage collector, and time is not spent copying the message into an 
  internal format.
- We do not need to allocate additional memory for every message. This helps
  with our goal for predictable per-connection resource usage. At the very minimum,
  each connection would only need enough memory to hold a single message in
  its connection buffer.
- We do not waste space on struct fields that do not apply to a given message type.
  Go does not have `union` types like in C.

Here is our representation of a `Tread` message using this strategy. It is almost the
same number of lines of code as a struct definition, but without the need to unmarshal
a message (only validation is necessary):

	type Tread []byte
	func (m Tread) Tag() uint16   { return guint16(m[5:7]) }
	func (m Tread) Len() int64    { return int64(guint32(m[:4])) }

	// Fid is the handle of the file to read from.
	func (m Tread) Fid() uint32 { return guint32(m[7:11]) }

	// Offset is the starting point in the file from which to begin
	// returning data.
	func (m Tread) Offset() int64 { return int64(guint64(m[11:19])) }

	// Count is the number of bytes to read from the file. Count
	// cannot be more than the maximum value of a 32-bit unsigned
	// integer.
	func (m Tread) Count() int64 { return int64(guint32(m[19:23])) }

For completeness, here are the definitions for the integer parsing functions:

	// Shorthand for parsing numbers
	var (
		guint16 = binary.LittleEndian.Uint16
		guint32 = binary.LittleEndian.Uint32
		guint64 = binary.LittleEndian.Uint64
	)

# Dealing with large messages

As I mentioned before, certain 9P messages can be quite large. A
server can negotiate a maximum message size below the protocol
limit, but there are some benefits to allowing large messages. If,
for instance, you are persisting large files to disk, it can be
more efficient to make 1 large write instead of 100 small writes.
If your server implements some sort of transactional semantics, it
is easier to implement if you can accept an entire request in a
single message. For these reasons, the `styxproto` package should
support messages close to the maximum allowed by the protocol.

# Fixed-size buffer

Supporting arbitrarily-large (up to 4GB) messages with fixed memory
usage can be somewhat tricky, and has an influence on our API design.
Here are all the T-messages (client → server) in the 9P2000 protocol, as
a reminder:

<img src="/img/9p-all.svg" style="width:100%"/>

The pale yellow fields are variable length. By imposing reasonable
maximum sizes on file names, user names, and the other variable-length
fields, we can come up. See the [limits.go][2] file for the limits imposed on
various fields. With the limits chosen, the largest message (excluding
`Twrite` and `Rread`) is `Twalk`, because it can include up to 16
file names of variable width. Thus, at the very minimum, our buffer must
be at least

	const MinBufSize = MaxWElem*(MaxFilenameLen+2) + 13 + 4

`MinBufSize` bytes long. We add 2 to `MaxFilenameLen to account
for the 2-byte length preceding variable-length strings. We add 13 to
the product to account for the preceding fixed-width fields. And we add
4 to that to account for the `size` field.

In order to randomly address any field in a message we must be able to
hold all of it in memory. That presents a problem when a message is
extremely large. For this reason, we will treat `Twrite` and `Rread`
messages specially; their `data` fields will *not* be randomly accessible.
They will implement the `io.Reader` interface, and must be accessed
as a stream of bytes.

# A Streaming API for message decoding

A `Twrite` message looks like this:

	type Twrite struct {
		r io.Reader
		msg msg // headers plus any extra buffered data
	}

In the `msg` member are the bytes containing the fixed-length fields
of the message. These fields are accessed just like the other messages:

	func (m Twrite) Fid() uint32 { return guint32(m.msg[7:11]) }

The `r` member provides an interface for accessing the rest of the
message, some of which may be already buffered in the `msg` member,
some of which may still be waiting to be read from the underlying
connection.

	func (m Twrite) Read(p []byte) (int, error) {
		return m.r.Read(p)
	}

The `Twrite` type implements the `io.Reader` interface, so that a `Twrite`
value can be used anywhere a byte stream can be used. For example:

	// Assuming msg is a Twrite value, write its data to realFile
	n, err := io.Copy(realFile, msg)
	// send an Rwrite message
	if err != nil {
		// send an Rerror message
	}

Using a streaming API for potentially large data fields gives us the freedom
to create an implementation that does not incur unexpected resource usage.

# Decoders

Following the example set by packages like `encoding/xml` and
`encoding/json`, we define a [Decoder][4], which wraps a connection
value. Decoding 9P messages is done through methods on the `Decoder`,
which fill and flush its connection buffers.

	type Decoder struct {
		MaxSize int64
		r io.Reader
		br *bufio.Reader
		start, pos int
		msg []Msg
		err error
	}

You'll see that I expose both the raw connection in the `r` member
in addition to the buffered reader in the `br` member. This will
be important later. The `start` and `pos` members are used during
parsing. The `msg` and `err` messages implement an API similar to
a `bufio.Scanner`.

A decoder is intended to be used in a loop, similar to a `bufio.Scanner`:

	d := NewDecoder(conn)
	for d.Next() {
		for _, msg := range d.Messages() {
			switch m := msg.(type) {
			case Twrite:
				// handle Twrite
			case Tauth:
				// handle Tauth
			default:
				// unexpected message
			}
		}
	}
	if d.Err() != nil {
		log.Fatal(d.Err())
	}

Note how error handling is done later, to reduce clutter in the
main loop. The `Next` method fills a `Decoder`'s buffer with data
from the underlying connection, overwriting any previously buffered
messages. If an error occurs reading or parsing data, `Next` returns
false, breaking the loop. Within the loop, the `Messages` method
returns a slice of 1 or more messages read during the last call to
`Next`.

The parsing code can be found [here][5]. Even with a simple message
format like 9P, parsing can be difficult and dangerous, as you are
dealing with untrusted data. I won't cover the parsing too closely,
as it is likely to change. However, my approach is to establish as
many invariants as I can about the state of the `Decoder` and the
structure of the incoming message. For instance, asserting early
on that enough data has been read to decode universal fields like
`size`, `tag`, and `type`.  This is one of the areas I will come
back to again and again. I have implemented fuzz testing against
the parser using [go-fuzz][6], but this is one area that would
benefit from more rigor.

The construction of `Twrite` and `Rread` messages during parsing
is worth repeating here:

		// dot contains the currently read bytes for this message, up to size
		m := Twrite{msg: dot}
		
		// even though we may not have read the whole message, because
		// of the representation we've chosen, we can still call methods on it
		count := m.Count()
		
		buffered := dot[23:] 
		m.r = bytes.NewReader(buffered)
		if int64(len(buffered)) < count {
			m.r = io.MultiReader(
				m.r,
				io.LimitReader(r, count-int64(len(buffered))))
		}
	
		return m, nil

By using an `io.MultiReader`, we can hide the fact that a message's
data is partially buffered. To calling code, whether the data is
all in memory, or coming from a connection (or a file, even), it
is always accessed via the `Read` method, like any other byte stream.

# Encoders

Given the representation we've chosen for 9P messages, there is not
much to encode; an encoder simply needs to copy the bytes of a
message to the connection.  However, it seems like an appropriate
place to put methods that create messages from higher-level parameters.

	type Encoder struct {
		w  *wire.TxWriter
		ew *util.ErrWriter
	}

The `[wire.TxWriter][7]` isolates concurrent writes to an `io.Writer`.
The `[util.ErrWriter][8]` captures errors occured during writing
(see [errors are values][9]). Here is an example method that writes
an Rauth message to an underlying connection (some functions ellided).

	func pheader(w io.Writer, size uint32, mtype uint8, tag uint16, extra ...uint32) {
		puint32(w, size)
		puint8(w, mtype)
		puint16(w, tag)
		puint32(w, extra...)
	}
	func (enc *Encoder) Rauth(tag uint16, qid Qid) error {
		size := uint32(maxSizeLUT[msgRauth])
	
		tx := enc.w.Tx()
		defer tx.Close()
	
		pheader(tx, size, msgRauth, tag)
		pqid(tx, qid)
		return enc.Err()
	}

And here is an `Rread` method:

	func (enc *Encoder) Rread(tag uint16, data []byte) error {
		if math.MaxUint32-minSizeLUT[msgTwrite] < len(data) {
			return errTooBig
		}
		size := uint32(minSizeLUT[msgRread]) + uint32(len(data))
	
		tx := enc.w.Tx()
		defer tx.Close()
	
		pheader(tx, size, msgRread, tag, uint32(len(data)))
		tx.Write(data)
		return enc.Err()
	}

Note that on the encoder size, we take a `[]byte` parameter instead
of an `io.Reader`. The reasoning behind this is that it makes the
calculation of the `size` and `count` fields simpler, and if a
program wants to send more data than it wants to hold in memory,
it can send multiple `Rread` messages.

# Next steps

See the [styxproto][3] package for the full implementation of the low-level
encoder and decoder for the 9P2000 protocol. It contains moderate tests,
including fuzz testing (which uncovered a couple of bugs). With this package,
we can read and write 9P messages from the network. However, it is not
enough to just speak the protocol. In the next post, I will cover what it takes
to implement a 9P server, including (but not limited to): sessions, fid management,
authentication, protocol negotiation. We will implement the higher-level `styx`
package that will interface with user-written servers.

[0]: https://capnproto.org/
[1]: https://google.github.io/flatbuffers/
[2]: https://github.com/droyo/styx/blob/18f7d3183d1c4df45e1d4d7c24b76356b57b58ef/styxproto/limits.go
[3]: https://aqwari.net/net/styx/styxproto
[4]: https://github.com/droyo/styx/blob/18f7d3183d1c4df45e1d4d7c24b76356b57b58ef/styxproto/decoder.go#L47-L69
[5]: https://github.com/droyo/styx/blob/18f7d3183d1c4df45e1d4d7c24b76356b57b58ef/styxproto/parse.go
[6]: https://github.com/dvyukov/go-fuzz
[7]: https://github.com/droyo/styx/blob/18f7d3183d1c4df45e1d4d7c24b76356b57b58ef/internal/wire/txwriter.go#L16
[8]: https://github.com/droyo/styx/blob/18f7d3183d1c4df45e1d4d7c24b76356b57b58ef/internal/util/errwriter.go#L8
[9]: https://blog.golang.org/errors-are-values
