+++
date = "2022-04-16T17:46:37Z"
title = "Sorting mailing list posts"
description = "Making mailing lists better than forums"
tags = [""]
+++

I recently spent some effort to overhaul the way I organize my email,
specifically email for the many mailing lists I subscribe to. I'm pretty
happy with the result, and it helped me subscribe to more mailing lists
than I had before. In most of the ways that I care about, my mailing list
experience is better than my experience with most web forums, now.

## Sorting mailing list e-mails hierarchically

Anyone who ever used
[NNTP](https://en.wikipedia.org/wiki/Network_News_Transfer_Protocol) can
recall the dotted, hierarchical naming scheme for news groups:

	comp.lang.lisp
	sci.math
	rec.arts.sf.movies

... and so on. This naming scheme means, to some degree, that groups
with related content will share a common naming prefix. So, with an
appropriately capable news reader, you could, for example, view content from
all programming-language related news groups by matching groups against the
pattern `comp.lang.*`.

Most mailing lists will contain a "List ID" header. For example, all mails
from the Linux kernel's "netdev" mailing list will contain the header:

	List-ID: <netdev.vger.kernel.org>

If we can reverse the List ID, then mailing lists from the same organization
will be naturally grouped together:

	org.kernel.vger.netdev
	org.kernel.vger.linux-fsdevel
	org.kernel.vger.kvm
	net.sourceforge.lists.v9fs-developer

Now, the naming is not quite as content-based as news groups; rather,
mailing lists run by the same organization will have the same prefix. This
is good enough for my needs. I am a Fastmail customer, and they allow
you to sort incoming and outgoing mail with the [Sieve mail filtering
language](https://en.wikipedia.org/wiki/Sieve_\(mail_filtering_language\)). So
I added the following custom action:

	if exists "list-id" {
	  if header :regex "list-id" "<(([^.]+)\\.)?([^.]+\\.)?([^.]+\\.)?([^.]+\\.)?([^.]+\\.)?([^.]+\\.)?([^.]+\\.)?([^.]+\\.)?([^.]+)>" {
	    set :lower "folder" "ml.${9}${8}${7}${6}${5}${4}${3}${2}";
	    fileinto :create "INBOX.${folder}";
	    stop;
	  }
	}

The regular expression is a bit ugly. Sieve programs often cross organizational
boundaries; here, I, the user/customer, am writing a program that will
run on another organization (Fastmail)'s servers. So, unsurprisingly, the
language's capabilities are very limited, to make it harder for a malicious
or buggy program to compromise the service. Notably, the language has no loops
or recursion.

The regular expression above takes a string such as

	<netdev.vger.kernel.org>

and cuts it up into these groups, numbered 1 through 10:

	((netdev)) (.vger) (.kernel) () () () () () (.org)

Then the expression

	set :lower "folder" "ml.${9}${8}${7}${6}${5}${4}${3}${2}";

creates the variable `folder` with the groups substituted:

	ml.kernel.vger.netdev

Note that I leave out the final group. This usually contains a top-level domain
name, like `org` or `net` or `com`. I may or may not revert that decision,
but I do that because there is a very small group of commonly-used TLDs and
I didn't see a point in grouping all messages that share the same TLD.

Finally, the expression

	fileinto :create "INBOX.${folder}";

Files the e-mail into the constructed folder, creating the folder if it does not
already exist. In Fastmail, a dot (`.`) is the folder separator. With this filter
in place, plus an extra filter for mailing lists which use the `Mailing-List` header
instead of `List-ID`, all of my mailing list e-mail is automatically sorted into
a nice hierarchy:

	ml/
		kernel/
			vger/
				linux-fsdevel
				linux-bluetooth
				netdev
		ocaml/
			discuss/
				community/
				learning/
				ecosystem/
		googlegroups/
			plan9port-dev
			golang-nuts

Occasionally I view one of these folders, but more commonly, I use Fastmail's
"saved searches" feature to view many lists at once. For example, I have a
few searches such as:

	in:ml/* is:unread after:1d
	in:ml/kernel/* is:unread after:1d
	in:ml/ocaml/* is:unread after:7d
	(in:kernel/* OR in:sourceforge/*) after:1d is:unread

If there is a thread that I want to follow, I 'pin' it, and then it will
show up in my "pinned threads" view. I have not currently found a way to be
notified of messages to pinned conversations.

That's it! I was doing something similar before, filing mailing list emails into
a folder based on the name of the list, but I was not reversing the domain name.
Such a small change has increased the number of lists I can subscribe to without
being overwhelmed.

## Email versus web forums

Email is old, and it shows. However, with the right client and filters, it can
be very pleasant to use. I used to follow several [discourse](https://www.discourse.org/)-based
forums through my web browser. However, discourse supports receiving and
posting forum messages over e-mail, so after implementing my filters, I enabled this feature
for the [discuss.ocaml.org](https://discuss.ocaml.org) forums. Since doing so, I prefer the email
experience, and almost never use the web interface.

### The content is immutable

Once an e-mail is delivered to you, the sender cannot "take it back" or modify
it. As long as you don't delete your e-mails, the message is never lost. While
mutability is a feature for something like a wiki, for open discussions it
is important that a message cannot be modified or redacted after the fact.

### You can compose cross-forum views

A web forum will never be able to compete with this feature. In the same
window, I can view messages from different mailing lists, run by different
organizations, who have no knowledge of each other.

### Most content is plain text

Depending on who you are, this may be a pro or con for email. For me,
I prefer plain text for most communication. Images and such can still
be provided as attachments. Many clients will send HTML emails as
well, but it is frowned upon on many of the lists I follow.

## Room for improvement

What I have is not perfect. I'm happy with it, but there are a few features I'd like
to have.

### Search based on threads, not messages

In my saved searches, I tried to filter out code reviews by including the term:

	-subject:*PATCH*

unfortunately, this rarely works, because it is common for replies to the
message to modify the subject and omit the `PATCH` phrase, like:

	[PATCH BlueZ 1/3] storage: Add support for STATE_DIRECTORY environment variable
	→ RE: [BlueZ,1/3] storage: Add support for STATE_DIRECTORY environment variable

I could try to chase these conventions and build more complicated filters,
but it would be nice to include or exclude a thread if *any* of its messages
match the filter.

### Limiting the results per folder

It would be really nice to have a view that just shows the most recent N
unread messages from each folder. This would give me an overview of every
list without the more active lists dominating the results. This is similar
to what the [Fraidycat](https://fraidyc.at/) extension does for feeds,
giving each feed the same amount of real estate regardless of how active it is.

### Better control over notifications

The fastmail web & mobile clients allow me to be notified when messages are
sent to a specific label, or from specific people, but I can't be notified
of new messages in a specific thread, or messages matching a specific search
filter.

### A proper desktop client

Fastmail's mobile and web clients are quite nice, so I have been dragging my
feet here. I still find myself composing messages in a text editor and then
copying it into Fastmail's web interface. I would like to setup a proper mail
client on my desktop, but there are so many, and I try not to customize the
software I use too much, that I am taking my time to find one with default
behavior that I like. A mail client may be able to fill some of the holes I've
described above.

## Crazy idea - mailbox as a web forum

A specialized mail client could provide a web forum-like user interface for
the mailing list posts in a user's mailbox. Mailboxes could be supplemented
with mailing lists archives, to show threads which existed before the user
subscribed to the list. There could be one section per list id, grouped
similarly to what my sieve filter accomplishes. The user could also create
"dynamic" boards, which are the union of all messages in one or more folders,
or all messages which match some search expression.

Is this a good idea? I'm not sure! The end result seems like it would
be useful, but I think there would be a lot of details that could end
up being a huge time sink. Even something as simple as deciding what
messages are part of the same thread, is [not as simple as you would
think.](https://www.jwz.org/doc/threading.html)
