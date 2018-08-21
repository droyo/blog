+++
date = "2016-07-25T17:44:22-05:00"
title = "Generating metrics for dnscache"
description = "Monitoring and visualizing everyone's favorite DNS cache"
tags = ["djb", "dns", "graphite", "monitoring"]
+++

DJB's [dnscache][0] is a recursive DNS cache with an unmatched
security record; over 15 years of widespread use with very few
vulnerabilities. In my opinion, dnscache is everything good Unix
software should strive to be; efficient, secure, single-purpose,
meticulously documented. While it may show its age as the internet
changes, it has survived bit-rot better than most, and for a program
that communicates so much with the ever-changing internet, that
is remarkable.

These days, other caches have caught up to and even surpassed
dnscache. [Unbound][1], for instance, can handle more traffic than
dnscache, can handle uneven SOA response times better, and has had
very few known vulnerabilities over the course of its development.
If you are using a central cache (as opposed to running a cache on
every server), unbound may be a better option.  However, if you have
inherited a dnscache-based infrastructure, first of all, consider
yourself lucky! Once configured properly, dnscache will be the most
reliable piece of infrastructure in your environment.

I have managed several high-volume dnscache servers before. One area I found
lacking was metrics. All of our servers tracked dozens of important metrics
and sent them to a [Graphite][2] installation, allowing us to track and
visualize server load. We didn't have any metrics for dnscache. All we could
see was the tx/rx throughput on the dnscache server links.

I wrote [dnscache-stats][3] to fill this gap in visibility. It reads output from
dnscache on its standard input, copies it to standard output, and sends
data points to a graphite service every minute. Because data "passes through",
it is suitable for use as a log processor for dnscache. Here is the `run` script
for the dnscache log processor:

	#!/bin/bash
	exec > >(exec setuidgid Gdnslog multilog '-* query *' '-* cached *' '-* sent *'  '-* rr *' t ./main)
	exec 2>/dev/null
	exec setuidgid Gdnslog dnscache-stats graphite.mydomain.local

After issuing `/command/svc -t /service/dnscache/log`, the process tree looks like this:

	\_ svscan /service
	|   \_ supervise dnscache
	|   |   \_ /command/dnscache
	|   \_ supervise log
	|       \_ dnscache-stats graphite.mydomain.local
	|           \_ multilog -* query * -* cached * -* sent * -* rr * t ./main
	\_ readproctitle service errors: ........................

The metrics (when viewed via [Grafana][4]) look like this.

<img src="/img/dnscache-stats.png" style="width: 100%"/>

This data is from my personal computer, so the numbers are very
low. However, in real-world usage, I have seen dnscache handle upwards
of 30k requests per second without breaking a sweat, and I have no reason
to believe that it cannot handle twice or thrice as many requests.

## Scaling dnscache

Dnscache works very well for certain workloads and struggles with others.
An ideal workload has the following properties:

- Many related domains (e.g. foo.com, bar.foo.com, baz.foo.com etc)
- All requests are for valid domains
- UDP only

With this type of workload, you can easily scale dnscache just by increasing
the size of your cache from the paltry 10MB that dnscache-conf configures
by default. You may run into issues with these kinds of workloads:

- Many unique domains
- Many(Mostly) invalid domains that will never resolve
- SOAs that are unreachable or take a long (>60 seconds) time to respond
- Huge variability in SOAs for a given domain

This falls out of a few design choices made for dnscache:

- dnscache does not cache negative results. That is, if a timeout is
  reached trying to resolve doesnotexist.somedomain.com, dnscache
  will try to resolve that domain anew every single time a client requests it.
- dnscache is single threaded. This is not as bad as it sounds; its
  `select`-based main loop is very fast and will usually spend most of
  its time waiting for network input.
- dnscache will service no more than 200 pending requests at a time,
  and will drop the oldest request when the 201st request comes in.
  again, this is not as bad as it sounds, but together with the first point
  it can cause real requests to be dropped if 99% of your requests will
  never resolve. You can measure this by looking for requests in the
  dnscache log that are dropped before they can reasonably be expected
  to be resolved, (say, under 2 seconds).

The 200 request limit is a consequence of using the `select(2)` system call
for I/O multiplexing. `select` can only monitor up to `FD_SETSIZE` file
descriptors at once. This tends to be 1024 on modern systems, but could
be as low as 250. In addition, in order to determine which file descriptors
are ready, you have to do a linear search of `select`'s results. So increasing
the number of concurrent requests would actually slow down dnscache's
main loop.

For these reasons, if you have the nightmare workload described above,
you really should look at using a different caching server. If you still decide
to use dnscache, you can probably continue scaling it by introducing 
a load balancer. Note that because dnscache is single-threaded, you can
scale it on a single server by load balancing between multiple processes
listening on different local IP addresses. Note that on Linux at least, this is
not as easy as it sounds; LVS, for instance, makes it difficult to balance 
between more than one local process.

## Keeping up with the times; AWS and akamai DNS records

One perplexing problem that I recently ran into was that certain records, usually
hosted by AWS or Akamai, were not being reliably resolved by dnscache. After
much investigation, it turned out to be due to intended behavior for dnscache;
When resolving any single record, dnscache puts an upper bound on the amount
of work it will do before giving up. This is not a quirk of dnscache, but part of the
[DNS specification][5]:

<blockquote>
     The amount of work which a resolver will do in response to a
     client request must be limited to guard against errors in the
     database, such as circular CNAME references, and operational
     problems, such as network partition which prevents the
     resolver from accessing the name servers it needs.
</blockquote>

In dnscache, this is limited to 100 name lookups. While in 2001 this
was quite generous, I found this limit was getting hit too frequently. One of the
main issues is that the name servers for Route53-hosted domains are themselves
hosted in a number of different domains. Here are some example Route53 name servers:

	ns-627.awsdns-14.net
	ns-507.awsdns-63.com
	ns-1198.awsdns-21.org
	ns-1859.awsdns-40.co.uk

Note that not only are they spread amongst many TLDs, but even the domain name is dynamic.
When you consider that in addition to resolving the record in question by walking from the root
domain "." up, dnscache must resolve the NS server names in the same fashion, the 100-lookup
limit no longer seems so generous. Through in a couple levels of CNAMEs and you'll end up
dropping many requests for legitimate records. This issue is also observed with akamai-hosted
DNS names, as seen [here][6]. dnscache's per-query work limit is governed by the QUERY_MAXLOOP
define and can be changed by patching the dnscache source code.

[0]: http://cr.yp.to/djbdns/dnscache.html
[1]: http://unbound.net/
[2]: http://graphite.readthedocs.org/
[3]: https://github.com/droyo/dnscache-stats/
[4]: https://grafana.org/
[5]: https://www.ietf.org/rfc/rfc1035.txt
[6]: https://marc.info/?l=djbdns&m=109422159909615
