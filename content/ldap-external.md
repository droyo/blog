+++
date = "2014-04-11T09:11:48-05:00"
draft = true
title = "LDAP TLS-based auth from C"
tags = ["ldap", "openldap", "c", "tls", "security"]
+++

One interesting method for binding to an LDAP server is using the
EXTERNAL sasl auth mechanism. This mechanism uses the transport
method for authentication when binding to the LDAP server. The two
most common cases of EXTERNAL auth are when connecting to a unix
domain socket, and creating a TLS or SSL connection with a client
certificate.

When connecting to a unix socket, the server is able to get the UID
and GID of the connecting user by calling `getpeereid` or getting
the `SO_PEERCRED` socket option. When opening a TLS or SSL connection
with a client certificate, the LDAP server can use the client
certificate to determine the client's identity.

I used the OpenLDAP library to fetch objects from an LDAP server
using client certificates. It took some studying to find out the
correct library calls to make to perform the bind. The code for the
OpenLDAP command-line clients is a good place to start, but they
are overkill; my use case does not have to support multiple sasl
mechanisms, and does not need to prompt the user for input.
