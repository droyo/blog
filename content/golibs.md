+++
date = "2013-01-17T13:42:18-05:00"
title = "Aqwari.net Go libraries"
tags = ["go"]
+++

This site references a number of Go packages. Documentation for
these packages may be found by visiting the package's import path
in your web browser. The goal is to produce focused libraries that
compose well with Go’s standard library.

The import hierarchy of aqwari.net follows the example set by the
Go standard library; for example,
[aqwari.net/encoding/ndb](https://aqwari.net/encoding/ndb) defines
functions to decode and encode ndb text.

Of particular significance is the aqwari.net/exp prefix – this
contains libraries that are not complete. Either some functionality
is missing, serious bugs have to be fixed, or the API has not been
set in stone yet. These library APIs are subject to change at any
time.

Other finished libraries will only have backwards-compatible changes.
Once they are moved out of aqwari.net/exp, these libraries should
be considered safe to use. There will be exceptions to this rule,
and the README of each project takes precedence over what is written
here. If you are shipping a critical production application, the
usual disclaimers apply – this site may go down, major bug may be
found, etc. Nothing is infallible, etc etc. Vendor your dependencies.

## List of libraries

All of the packages below are in the aqwari.net namespace, for
example, importing the `io/tailpipe` package would look like this:

	import "aqwari.net/io/tailpipe"

These import paths are served up by the [gogive](https://github.com/droyo/gogive) command.
<dl>
<dt><a href="https://aqwari.net/exp/ndb">exp/ndb</a></dt>
<dd>Parser for the <a href="https://swtch.com/plan9port/man/man7/ndb.html">ndb</a> file format.</dd>
<dt><a href="https://aqwari.net/exp/soap">exp/soap</a></dt>
<dd>Helper functions for dealing with SOAP envelopes.</dd>
<dt><a href="https://aqwari.net/exp/display">exp/display</a></dt>
<dd>Package for setting up an OpenGL window. Uses SDL2 on Linux, GLUT on OSX.</dd>
<dt>[exp/gl](https://aqwari.net/exp/gl</dt>
<dd>OpenGL bindings.</dd>
<dt><a href="https://aqwari.net/xml">xml/xmltree</a></dt>
<dd>Manipulate an XML document as a tree. Supports xml namespace resolution at arbitrary points within the tree.</dd>
<dt>[retry](https://aqwari.net/retry</dt>
<dd>Exponential backoff and other retry policies.</dd>
<dt><a href="https://aqwari.net/net/styx">net/styx</a></dt>
<dd><a href="https://en.wikipedia.org/wiki/9P_(protocol)">9P2000</a> network filesystem protocol, client/server implementation.</dd>
<dt><a href="https://aqwari.net/io/tailpipe">io/tailpipe</a></dt>
<dd>Pure-go implementation of `tail -F` (that's a capital F)</dd>
</dl>

## Contributing

The code repositories for aqwari.net packages are hosted on github.
Github pull requests, issues, etc are welcome, as are direct inquiries
by e-mail (droyo at aqwari dot net).
