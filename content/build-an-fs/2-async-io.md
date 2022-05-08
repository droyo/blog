
Ideally these two questions should have the same answer. And that answer should
give me a goal to strive for. It should be noted that if the capabilities of
the unix socket becomes a problem, that can be overcome; the Linux 9P client
supports `virtio` and `rdma` transports` which should be more capable. But
I seriously doubt those will be necessary for the devices I have.

To answer the first question I can use the `unixserver` and `unixclient` utilities
from [Bruce Guenter's site](https://untroubled.org/ucspi-unix/). These tools
allow ordinary programs to read and write from unix sockets. Combined with
the `pv` tool which measures the rate of transfer, I can use shell commands
like this:

	# server
	unixserver test.sock -- sh -c 'exec pv -r -B 32M >/dev/null'

	# client
	unixclient test.sock sh -c 'exec cat </dev/zero >&7'

On my machine this gives me a transfer rate of 7.3 GB/s, and increasing the
buffer size of `pv` doesn't change that. This is a fine target to shoot for; storage
devices that can support this rate of transfer are still quite expensive.

Here's a similar Ocaml program that uses the `Iovec` module to simply read
data from a socket as quickly as possible:

	let rec consume fd buf =
	  match Iovec.readv fd buf with
	  | exception End_of_file -> ()
	  | n -> consume fd buf

The throughput of this program is a little over half of what `cat` and `pv` could
do:

	$ unixclient /tmp/iovec_nullpipe.sock sh -c 'exec pv -r -B 32M </dev/zero >&7'
	[3.89GiB/s]

Notably, the OCaml program shows 50% CPU usage, whereas in the `cat`/`pv` test,
both sender and receiver showed 100% CPU usage. If I use the `funccount` tool from
`bcc`, I can see that `cat` is calling `read` at a higher rate than my program is calling
`readv`:


