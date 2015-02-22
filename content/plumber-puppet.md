+++
date = "2014-03-03T14:42:28-05:00"
title = "Plumbing rules for Puppet manifests"
description = "Quickly navigating puppet modules with Acme"
tags = ["puppet", "plan9", "acme"]
+++

I often use [Puppet][0] to manage configuration for servers. One
thing that makes Puppet a joy to work with are the [plumbing][1]
rules I've put in place to help me navigate puppet manifests. Here
is a rule I use to match puppet module names:

	puppet = /home/droyo/puppet
	type	is	text
	data	matches	'([a-z0-9][A-Za-z0-9_\-]+)((::[a-z0-9][A-Za-z0-9_\-]+)+)('$addr')?'
	arg	isdir	$puppet/modules/$1
	plumb	start	rc -c 'plumb `{echo '''$dir/manifests/$2.pp$4'''|sed ''s,::,/,g''}'

Here is a rule I use to match `puppet:///` URLs:

	type	is	text
	data	matches	'puppet:///modules/([a-zA-Z0-9\-_\.]+)/([a-zA-Z0-9\-_/\.]+)'
	arg	isfile $puppet/modules/$1/files/$2
	data	set	$file
	plumb	to	edit
	plumb	client	window $editor

And another for ERB templates:

	type	is	text
	data	matches	'template\([^\)]+\)'
	data	matches	'template\(.([a-zA-Z0-9_\.\-]+)/([a-zA-Z0-9_/\.\-]+).\)'
	arg	isfile	$puppet/modules/$1/templates/$2
	data	set	$file
	plumb	to	edit
	plumb	client	window $editor

These three plumbing rules allow me to quickly navigate large puppet
code bases from [Acme][2]. By right-clicking on text like `foo::bar`,
`template("foo/bar.cfg.erb")`, or `puppet:///foo/bar.txt`, the
relevant file will be opened in a new window in Acme. When people
watch over my shoulder they find it a little alarming how quickly
I open up all files referenced in a puppet manifest, but after using
these rules for about a year I can't imagine living without them.

[0]: http://puppetlabs.com/
[1]: http://plan9.bell-labs.com/sys/doc/plumb.html
[2]: http://acme.cat-v.org/
