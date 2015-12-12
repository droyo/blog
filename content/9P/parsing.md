+++
date = "2015-09-29T12:00:00-05:00"
title = "Writing a 9P server from scratch: Protocol parsing"
tags = ["go", "plan9", "9p"]
draft = true
+++

In the previous post, we began work on the protocol parser, getting
to the point where we could read 9P messages from a network
traffic. With this post we will pick up the pace, finishing the protocol
parsing and adding routines to create and send protocol messages.
The following two posts will be about the high-level server/client
library.

# Protocol  parsing


