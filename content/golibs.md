+++
date = "2013-01-17T13:42:18-05:00"
title = "Aqwari.net Go libraries"
tags = ["go"]
+++

This site references a number of Go packages. Documentation for these packages
may be found here. The goal is to produce focused libraries that work well with
Go’s standard library.

The import hierarchy of aqwari.net follows the example set by the Go standard
library; for example, aqwari.net/encoding/ndb defines functions to decode and
encode ndb text.

Of particular significance is the aqwari.net/exp prefix – this contains
libraries that are not complete. Either some functionality is missing, bugs
have to be fixed, or the API has not been set in stone yet. These library APIs
are subject to change at any time.

Other finished libraries will only have backwards-compatible changes. Once they
are moved out of aqwari.net/exp, these libraries should be considered safe to
use. If you are shipping a critical production application, the usual
disclaimers apply – this site may go down, major bug may be found, etc. Nothing
is infallible, etc etc.

## Contributing

The code repositories for aqwari.net packages are actually hosted in github.
Github pull requests, issues, etc are welcome, as are direct inquiries by e-mail
(droyo at aqwari dot net).
