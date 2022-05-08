+++
date = "2022-04-23T23:01:13Z"
title = "Composing linux namespaces with GNU guix"
description = "Look ma, no symlinks!"
tags = ["guix", "scheme"]
+++

[GNU Guix](https://guix.gnu.org/) is a package manager, build tool,
configuration management system, and operating system built with
[Guile scheme](https://www.gnu.org/software/guile/). Much like
[Nix](https://nixos.org/) before it, it implements *functional*
package management, where a software package is built in a hermetic
environment with only its explicit and transitive dependencies
available. The output of a package build is named using a hash of
its inputs and stored in a globally readable directory called the [GNU
store](https://guix.gnu.org/manual/en/html_node/The-Store.html), conventionally
`/gnu/store`. Importantly, given the same set of inputs, a package's output
will always be the same, bit-for-bit. This nice property gives you the
ability to audit build caches (called *substitutes*) by re-running a package
build and comparing the outputs. You can also share a GNU store among
multiple machines.

Because it is inconvenient to run commands using their absolute
paths in the GNU store, which will be a long name like
`/gnu/store/1b9aa5iwgl956xrav7p1kdq4k2ackmym-bird-2.0.8/sbin/bird`,
guix helps you build a different type of object, called a
"[profile](https://guix.gnu.org/en/blog/2019/guix-profiles-in-practice/)",
which is, essentially, a directory tree containing the union of one or more
packages' directory trees.  For example, I can use the `guix build` command
to build a profile that is suited to running a network router:

	$ cat <<EOF > profile.scm
	(use-modules
	  (guix profiles)
	  (gnu packages networking)
	  (gnu packages linux)
	  (gnu packages busybox))

	(profile
	  (content
	    (packages->manifest (list bird busybox iproute util-linux))))
	EOF
	$ guix build -f profile.scm
	/gnu/store/q6al2kmlgcaw1kzlqvqzmm213fnp48lb-profile

This builds the directory `/gnu/store/q6al2kmlgcaw1kzlqvqzmm213fnp48lb-profile`
.  This directory is the union of the `bird`, `busybox`, `iproute`, and
`util-linux` packages; for example, here is an excerpt from the `/sbin`
directory:

	$ for path in sbin/* ; do echo $path '->' $(readlink $path); done
	...
	sbin/uuidd -> /gnu/store/xxx-util-linux-2.37.2/sbin/uuidd
	sbin/vconfig -> /gnu/store/xxx-busybox-1.33.1/sbin/vconfig
	sbin/vdpa -> /gnu/store/xxx-iproute2-5.15.0/sbin/vdpa

These directories of symlinks are sometimes called
"symlink farms".  The union algorithm is implemented in the [`(guix build
union)`](https://git.savannah.gnu.org/cgit/guix.git/tree/guix/build/union.scm?h=v1.3.0&id=a0178d34f582b50e9bdbb0403943129ae5b560ff#n116)
module like so, where the *inputs* are each package, and the *output* is the
target union directory (such as the `/gnu/store/...-profile` directory in the example above):

1. Sort the inputs.
2. If a file only exists in one input, create a symlink in the output to
   the file in that input.
3. If the file path exists in multiple inputs and is not a directory, print
   a warning and choose the first file.
4. If the file exists in multiple inputs and is a directory, create an
   empty directory with the same name in the output, and repeat from step 2,
   relative to the new empty directory.

The `guix` subcommands maintain a chain of symlinks starting from
`~/.guix-profile` to one of these profile outputs, allowing you to add
`~/.guix-profile/bin` to your `PATH` variable. The `guix package` command,
for example, simply links packages from the GNU store into your currently
active profile.

This is great, and gives users the ability to freely compose different
versions of packages within their own environment without affecting other
users, and without root privileges. However, something about seeing so many
symlinks irks me. I will fully admit that my sense of disgust is irrational
and I should get over it. But here are the series of symlink walks needed
to run, for example, the `ip` program at `~/.guix-profile/sbin/ip`:

	~/.guix-profile
	-> /var/guix/profiles/per-user/david/guix-profile
	-> guix-profile-47-link
	-> /gnu/store/j9amghbs5nk3gwj27r62gv56q6rh80k2-profile
	-> /gnu/store/j9amghbs5nk3gwj27r62gv56q6rh80k2-profile/sbin/ip
	-> /gnu/store/35lj2sn5p6wfd8h1j11hb2mcvria3cfl-iproute2-5.15.0/sbin/ip

I accept that symlinks solve a real problem and are very useful. But in
my opinion, they are a wart, they required invasive changes to most unix
utilities, and they allow for directory loops. Most of the problems that
they solve would have been solved better by allowing processes to modify
their mount namespaces, and providing a union meta-filesystem, as [Plan
9](http://9p.io/magic/man2html/1/bind) did in the 90s.  They also require
careful planning and definition to use with `chroot`. An absolute symlink to
a path in `/gnu/store` requires `/gnu/store` to be present in the new root,
and be mounted in the same place. The `guix` command provides a few subcommands
and flags for running in a chroot environment that will handle this for you.

A less technical problem is that sometimes I don't want a user, human
or otherwise, know the "real" path to a file. I want `/bin/sh`, from the
perspective of a user or process trapped in a given namespace, to just be
`/bin/sh`, not a symbolic link, which the user can read the contents of, to
`/gnu/store/xxx-bash/bin/sh`. Not so much for security reasons, but because
I just don't want an uninitiated user to get confused. I want a person to
be able to use a guix-managed system without realizing it, because it
looks just like any other system, except perhaps that more directories are
read-only than they're used to.

Fortunately, many ideas from Plan 9
are available in Linux today, including [mount
namespaces](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html)
and a union file system implementation in the form of
[overlayfs](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html).
When compared to their Plan 9 counterparts, they are harder to use, less
general, and there is more ceremony and privilege required for their
use. However, I believe these are surmountable problems.

## Proposed semantics

First I started with a vision of what I wanted to accomplish; I want to write
in a file, `my-namespace.scm`, something like this:

	(namespace
	  ;; bind mount host paths into this new namespace
	  (bind (list "/var/" "/dev/" "/proc/"))

	  ;; bind this path to the output of a G-expression
	  (bind "/etc/hosts" #~#$(plain-file "hosts" "127.0.0.1 localhost\n"))
	  (bind "/etc/resolv.conf" #~#$(local-file "./resolvconf-file"))

	  ;; merge the output of these packages
	  (bind "/"
	    (list bird iproute busybox util-linux)))

More formally, if *root* is the new root of the namespace:

- `(bind path)` bind mounts *path* to *root*/*path*
- `(bind path #~G)` bind mounts the output of the
   [G-expression](https://guix.gnu.org/manual/en/html_node/G_002dExpressions.html)
   *G* to *root*/*path*.
- `(bind path package)` bind mounts the output of *package* in the GNU store to
   *root*/*path*.
- `(bind (p1 ... pN))` is equivalent to `(bind p1) ... (bind pN)`
- `(bind path (v1 ... vN))` is equivalent to `(bind path v1) ... (bind path vN)`

When the same path is mounted more than once, an
[overlay](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html)
mount is performed instead, with the first binding taking higher precedence
in the event of collisions.

I could then run `guix build -f my-namespace.scm`, and the resulting output
directory in the GNU store would contain a directory structure and supporting
programs for building the described mount namespace and executing a new process
inside of it. A mount namespace is a run-time kernel resource that zero or
more processes may share; it is different from symbolic links, which are
stored in the file system and available to everyone. The only way to "persist" a
namespace across reboots is to persist a program constructing said namespace,
and get that program to run automatically.

## Development environment

I'm not sure this work would or should ever make into the GUIX project, so for
now, I maintain my own [guix channel](https://git.sr.ht/~droyo/guix-channel)
which lets me define my own modules and packages. I've checked this repository
out to my workstation, and I followed the instructions in its `README` to
add the channel to my `~/.config/guix/channels.scm` file, so that `guix`
subcommands can access the modules defined within it. For `guix` to see a
file in my channel, that file must be committed to the repository, and that
commit must be signed with my GPG signature.

I do not want to commit every little change I make just to test if it
works. Luckily, guix subcommands have a `--load-path=DIR` flag, allowing me
to load files from an existing directory. I want to have a quick feedback
loop while developing this, so I start first with a test, creating
`tests/namespace.scm` in my `guix-channel` repository:

	(define-module (test-namespace)
	  #:use-module (srfi srfi-64)
	  #:use-module (aqwari namespace))

This expression creates a
[module](https://www.gnu.org/software/guile/manual/html_node/Using-Guile-Modules.html)
called `test-namespace`, which depends on two other modules, whose public
symbols will be available in the body of this module. The first module is
the unit test module `srfi/srfi-64`. Scheme is a very minimal language,
but there is a repository of interfaces called "Scheme Requests for
Implementation" or [SRFI](https://srfi.schemers.org/) for short, describing
libraries for common tasks like list manipulation, string processing,
records, and so on. Guile scheme comes with implementations for a [large
swath](https://git.savannah.gnu.org/cgit/guile.git/tree/module/srfi?h=master)
of finalized SRFIs, along with its own set of libraries under the `ice-9`
prefix.

The second `#:use-module` argument loads `aqwari/namespace.scm`, which
is where I will put the implementation of the `(namespace ...)` expression I
envisioned above. It doesn't exist, yet.

	(test-begin "namespace")

	(define simple-ns
	  (namespace
	    (bind "/var")
	    (bind "/dev")))

	(test-assert (namespace? simple-ns))

In this series of expressions, I define a test that constructs a namespace
with bindings from "/var" to "./var" and "/dev" to "./dev". I plan for
this expression to create a record, and my module will export the function
`namespace?` which is true if a value has that record type.

I can then run the test from the root of my guix channel like so:

	$ guix repl -L $(pwd) -- tests/namespace.scm

It currently fails with the error:

	no code for module (aqwari namespace)

which is expected. I setup a loop so that this test will run
whenever I make a change to the `aqwari/` directory. I use the
[Acme](https://research.swtch.com/acme) text editor, with the
[Watch](https://9fans.net/go/acme/Watch) command to trigger the tests
whenever I save a file with the `Put` command. You may do something similar
in a terminal with the `inotifywait` command like so:

	inotifywait -e modify -m -q aqwari | \
		xargs -n 1 guix repl -L . -- tests/namespace.scm

On to the implementation!

## Developing the namespace syntax

The way I plan to implement this feature is to define a
`<namespace>` record type, containing a list of records resembling
[*fstab*](https://man7.org/linux/man-pages/man5/fstab.5.html)(5)
entries. Then I will use the
[define-gexp-compiler](https://git.savannah.gnu.org/cgit/guix.git/tree/guix/gexp.scm?h=v1.3.0&id=a0178d34f582b50e9bdbb0403943129ae5b560ff#n291)
syntax to define a function that converts a record of this type into a
directory structure and a helper program that builds the mount namespace
and executes into it.

Similarly to the test, I start `aqwari/namespace.scm` with a `define-module`
expression:

	(define-module (aqwari namespace)
	  #:use-module (ice-9 match)
	  #:use-module (srfi srfi-9)
	  #:export
	    (namespace
	     namespace?
	     namespace-mounts))

An addition here is the use of the `#:export` keyword
argument, which defines the list of public symbols
available to other modules that import this one. The [`(ice-9
match)`](https://www.gnu.org/software/guile/manual/html_node/Pattern-Matching.html)
module provides syntax for pattern matching. You will see it in action
below. I learned about pattern matching from [Ocaml](https://ocaml.org/),
and I find it to be a very useful tool that generally leads to clearer
code and better error messages. The `(srfi srfi-9)` module provides
[records](https://www.gnu.org/software/guile/manual/html_node/SRFI_002d9-Records.html),
data types with named fields.

	(define-record-type <namespace>
	  (make-namespace mounts)
	  namespace?
	  (mounts namespace-mounts))

This defines a new record type, `<namespace>`, with a single field,
*mounts*. The constructor `make-namespace` creates new values of this type.  The
*mounts* field, accessed with the function `namespace-mounts`, will have a list
of file systems to mount.

I want to start by creating the `(namespace ...)` syntax, which I
sketched out earlier. This high-level expression will be converted into
an expression that ultimately calls the `make-namespace` constructor
to create a `<namespace>` record. In scheme, new syntax is defined using a
[`syntax-rules`](https://www.gnu.org/software/guile/manual/html_node/Syntax-Rules.html)
expression, so I will start with that.

	(define-syntax namespace
	  (syntax-rules ()
	    ((namespace expr ...)
	     (make-namespace '() '()))))

This converts any expresion of the form `(namespace ...)`, where `expr ...`
matches zero or more arbitrary expressions, to the expression `(make-namespace
'() '())`, which will create an empty namespace. With this, the test starts
passing:

	%%%% Starting test namespace
	Group begin: namespace
	Test begin:
	  test-name: #<procedure %namespace?-procedure (a)>
	  source-file: "tests/namespace.scm"
	  source-line: 15
	  source-form: (test-assert (namespace? simple-ns))
	Test end:
	  result-kind: pass
	  actual-value: #t

However, recall that `simple-ns` in the unit test defined a binding for the
"/var" directory, but the current code just creates an empty namespace.
The tests need an addition.

	(test-assert (not (nil? (namespace-mounts simple-ns))))

Now the test is failing again, as desired. Going back to the `namespace`
syntax, sometimes it can be difficult to write macros because it can be easy
to get confused about what your output should look like. It's important to
remember that macros are transforming *code* from one form to another.  A macro
is just manipulating a tree of symbols, and can't make any determination
about what those symbols mean. So the `namespace` macro takes a series of
`(bind args ...)` expressions and converts them to an expression that,
when evaluated, constructs the equivalent `<namespace>` record.

	(define-syntax namespace-args
	  (syntax-rules (bind)
	    ((namespace-args ()) '())

	    ((namespace-args ((bind args ...) . rest))
	     (cons (bind->mount args ...) (namespace-args rest)))))

	(define-syntax-rule (namespace . args)
	  (make-namespace (namespace-args args)))

The `namespace-arg` macro is a recursive macro which converts an expression
like

	(namespace-args
	  (bind "/var/")
	  (bind "/dev/"))

to this:

	(cons (bind->mount "/var/")
	  (cons (bind->mount "/dev/")
	    '()))

if you have any experience with lisp, you know `(cons x (cons y '()))`
is equivalent to `(list x y)`. The `namespace` macro then just passes the
list to the `make-namespace` constructor. I haven't shown `bind->mount`
yet. Here it is:

	(define bind->mount
	  (match-lambda*
	    (((? string? target) (? string? source))
	     (make-bind-mount target source))

	    (((? string? target) (? gexp? source))
	     (make-bind-mount target source))

	    (((? string? target) (? package? pkg))
	     (make-bind-mount target (gexp (ungexp pkg))))

	    (((? string? target))
	     (make-bind-mount target target))))

I left out the list cases to keep it brief. You might have guessed, but
`make-bind-mount` is a constructor for a record type, `<bind-mount>`.

	(define-record-type <bind-mount>
	  (make-bind-mount target source)
	  bind-mount?
	  (target bind-mount-target)
	  (source bind-mount-source))

There is also `<overlay-mount>`:

	(define-record-type <overlay-mount>
	  (make-overlay-mount target lowerdir upperdir workdir)
	  overlay-mount?
	  (target   overlay-mount-target)   ;; dir to mount on
	  (lowerdir overlay-mount-lowerdir) ;; ordered list of dirs
	  (upperdir overlay-mount-upperdir) ;; single writable dir
	  (workdir  overlay-mount-workdir)) ;; writable merge dir

I will expand on in a bit.

## Layering definitions

To enable code sharing between namespace specifications I added a simple
`include` statement to the `(namespace-args)` macro:

	((namespace-args ((include ns) . rest))
	 (append (namespace-mounts ns) (namespace-args rest)))

This allows definitions such as

	(define base-ns
	  (namespace
	    (bind '("/etc/resolv.conf" "/etc/nsswitch.conf" "/etc/gai.conf"))
	    (bind '("/etc/passwd" "/etc/group" "/etc/services"))))

	(define app1-ns
	  (include base-ns)
	  (bind "/" my-app1-package))

	(define app2-ns
	  (include base-ns)
	  (bind "/" my-app2-package))

which allows me to share some code between namespace definitions. I
defined a `%namespace-minimal` variable which contains essential files like
`resolv.conf`, SSL certificates, the time zone database, and so on.

## Merging mountpoints

When the same mountpoint is bound multiple times, I want the files in
all sources to be visible under the mountpoint. The overlay filesystem,
available in the stock Linux kernel, allows for this. So, given the raw list
of mountpoints produced by a `(namespace ...)` expression, I want a
function that produces a new list with multiple mounts to the same
mountpoint replaced with a single overlay mount.

	(define (collapse-mounts mounts)
	  ;; The order of bindings to the same mountpoint is significant,
	  ;; so stable sort is a requirement.
	  (let loop ((args (stable-sort mounts compare-mountpoints)))
	    (match args
	      ('() '())

	      ((($ <overlay-mount> mnt lower upper work)
	        ($ <bind-mount>    mnt source) . rest)
	       (let ((stack (append lower (list source))))
	         (loop
	           (cons
	             (make-overlay-mount mnt stack upper work)
	             rest))))

	      ((($ <bind-mount> mnt dir1)
	        ($ <bind-mount> mnt dir2) . rest)
	       (loop
	         (cons
	           (make-overlay-mount mnt (list dir1 dir2) #f #f)
	           rest)))

	      (((= mount-file mnt) (= mount-file mnt) . rest)
	       (error "don't know how to combine ~a" (take args 2)))

	      ((fs . rest) (cons fs (loop rest))))))

A subtle detail that you may overlook is the re-use of the identifier
*mnt* in the match cases, like:

	      ((($ <bind-mount> mnt dir1)
	        ($ <bind-mount> mnt dir2) . rest)

The syntax `($ <record-name> field1 field2 field3 ...)` allows for pattern
matching on records. In the example above, because *mnt* is used in both
cases, this pattern only matches mounts that have the same target. The initial
sorting of mounts ensures that mounts for the same mountpoints are adjacent,
and the use of a stable sort preserves the input ordering of mounts for the
same mountpoint.

## "Lowering" a namespace

When you run a command like `guix build hello`, the `guix` command spawned
by your shell will find the definition of the `hello` package in one of the
configured channels. It has a definition like this, which, similar to the
`(namespace ...)` expression, evaluates to a `<package>` record:

	(package
	  (name "hello")
	  (source ...)
	  (build-system gnu-build-system)
	  (license gpl3+)
	  ... more fields ...))

The `guix` command loads this high-level description,
and converts it into a lower-level representation called a
[*derivation*](https://guix.gnu.org/manual/en/html_node/Derivations.html). A
derivation is, more or less, a list of inputs and outputs, plus a builder
that will build the output from the inputs. Each of the inputs, outputs,
and builders are themselves derivations. The `guix-daemon` service, which
actually runs builds in sandboxed environments and puts outputs in the GNU
store, understands derivations.

In the parlance of the Guix manual, the process of
translating a high-level object like a `package` record to a
derivation is called "lowering" and is briefly described in the
[G-expressions](https://guix.gnu.org/manual/en/html_node/G_002dExpressions.html)
section of the manual. The `(guix gexp)` module provides a
`define-gexp-compiler` syntax that allows me to convert the high-level
`<namespace>` record into a derivation.

Before I do that, I'll revisit what I want the output to look like. Remember that a
namespace is part of the state of running process(es). Some tools like `lxc` and
`docker` exist to deserialize descriptions of namespaces onto a group of running
processes, but namespaces are still fundamentally a run-time concept, rather than
a build-time one. So I am not building a *namespace*; I'm building a directory
structure and startup script that can be executed to instantiate a namespace.

The directory structure will look something like this:

	+ /gnu/store/...-namespace
	  - exec
	  + root
	    - /var
	    - /dev

The `root` directory will be the new root (/) directory of the mount namespace,
pre-populated with mountpoints for directories specified in `(bind ...)`
directives. `exec` is an executable that, when run, will create a new,
anonymous mount and network namespace, mount any directories described by
`(bind ...)` directives in the `(namespace ...)` specification, and allocate
any network links described by `(link ...)` directives. it will then use the
[*pivot_root*](https://man7.org/linux/man-pages/man2/pivot_root.2.html)(2)
system call to change the root directory to `root`, and call
[*execv*](https://man7.org/linux/man-pages/man2/execve.2.html)(2) to execute
into its command-line argument. You should be able to do something like

	$ exec $(guix build -f my-namespace.scm) /bin/sh

To replace your existing shell process with a new `sh` shell running in the
mount namespace described by `my-namespace.scm`. Of course, that namespace
must contain a `/bin/sh` executable.

The `exec` program is a statically-compiled C program. While I prototyped this
program in bash, and then [execline](https://skarnet.org/software/execline/),
I want `exec` to have no dependencies, and work in any environment, regardless
of what is in the `PATH` environment variable, or what dynamic libraries are
or are not visible currently located. This allows the `exec` program to work
from another mount namespace. The static executable also produced less noisy
`strace` output which was a benefit while debugging it.

The C program is
[here](https://git.sr.ht/~droyo/guix-channel/tree/3bfc628072a3ea8dda59cde303edd7869e30c3d9/item/aqwari/ns-helper.c).
It assumes the existence of global variables `root` and
`fstab`, and the definition of a `struct fstab` with parameters for
[*mount*](https://man7.org/linux/man-pages/man2/mount.2.html)(2) calls. During
the build process, I pipe these structs and variables to the C compiler as
a header file with the `-include` flag. The C program took a lot of trial
and error to write. One issue I encountered almost immediately after getting
what looked like reasonable `strace` output was an error like this:

	execv(/bin/sh): No such file or directory

You may think this means that I messed up the mounts, and `/bin/sh` didn't
exist in the new namespace. But it did! What was *actually* going on is
evident once you inspect busybox's `/bin/sh`:

	$ file $(guix build busybox)/bin/sh
	/gnu/store/...-busybox-1.33.1/bin/sh: symbolic link to busybox
	$ ldd $(guix build busybox)/bin/sh
		linux-vdso.so.1 (0x00007ffd3e504000)
		libm.so.6 => /gnu/store/...-glibc-2.33/lib/libm.so.6 (0x00007f83ec62b000)
		libresolv.so.2 => /gnu/store/...-glibc-2.33/lib/libresolv.so.2 (0x00007f83ec612000)
		libc.so.6 => /gnu/store/...-glibc-2.33/lib/libc.so.6 (0x00007f83ec450000)
		/gnu/store/...-glibc-2.33/lib/ld-linux-x86-64.so.2 (0x00007f83ec76e000)

The `busybox` program is dynamically linked to files in the GNU store,
which wasn't mounted in the new namespace. Because almost all guix binaries
are dynamically linked, there are very few useful namespaces I can produce
without /gnu/store present, and I decided to unconditionally add it to all
mount namespaces.

With the current state of the [repo](https://git.sr.ht/~droyo/guix-channel/tree/3bfc628072a3ea8dda59cde303edd7869e30c3d9),
I can run something like this:

	sudo PATH=/bin:/sbin $(guix build -f my-namespace.scm)/exec /bin/sh
	/ # findmnt
	TARGET                       SOURCE                             FSTYPE  OPTIONS
	/                            overlay                            overlay ro,relat
	|-/dev                       dev                                devtmpf rw,nosui
	| |-/dev/shm                 tmpfs                              tmpfs   rw,nosui
	| |-/dev/pts                 devpts                             devpts  rw,nosui
	| |-/dev/hugepages           hugetlbfs                          hugetlb rw,relat
	| `-/dev/mqueue              mqueue                             mqueue  rw,nosui
	|-/etc/gai.conf              /dev/nvme0n1p2[/etc/gai.conf]      ext4    rw,relat
	|-/etc/group                 /dev/nvme0n1p2[/etc/group]         ext4    rw,relat
	|-/etc/nsswitch.conf         /dev/nvme0n1p2[/etc/nsswitch.conf] ext4    rw,relat
	|-/etc/passwd                /dev/nvme0n1p2[/etc/passwd]        ext4    rw,relat
	|-/etc/resolv.conf           /dev/nvme0n1p2[/etc/resolv.conf]   ext4    rw,relat
	|-/etc/services              /dev/nvme0n1p2[/etc/services]      ext4    rw,relat
	|-/proc                      proc                               proc    rw,nosui
	| `-/proc/sys/fs/binfmt_misc systemd-1                          autofs  rw,relat
	|   `-/proc/sys/fs/binfmt_misc
	|                            binfmt_misc                        binfmt_ rw,nosui
	`-/gnu/store                 /dev/mapper/sata-gnustore[/store]  xfs     ro,relat
	/ #

I have left out how I make read-write unions if a tmpfs is bound to a
union directory, and some other hairy details. The `exec` C program is
fairly readable, but finding the correct incantations took some careful
reading of the man pages. If you want to see all the gory bits, check the
[repo](https://git.sr.ht/~droyo/guix-channel)

There are a few open problems and areas for improvement.

### Overlay limitations

On my machine, which is running a 5.17 kernel, the overlayfs driver
[defines](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/overlayfs/super.c?h=v5.17&id=f443e374ae131c168a065ea1748feac6b2e76613#n27)
the constant:

	#define OVL_MAX_STACK 500

This is the maximum number of `lowerdir` parameters allowed. 500 may sound
like a lot, and I think most useful profiles could fit within that limit,
but if I add runtime dependencies, the union mounts could balloon in size.

### Runtime dependencies

Building a namespace such as

	(namespace
	  (bind "/" (list s6)))

Will put binaries like `s6-svscan` in `/bin`. However, it will not put binaries
from the `execline` package in `/bin`, even though they are required at run-time
for some functionality of binaries in `s6`. A binding should include all run-time
dependencies as well.

The [`<package>`](https://guix.gnu.org/manual/en/html_node/package-Reference.html)
record defines a *propagated-inputs* field, described thus:

> propagated-inputs is similar to inputs, but the specified packages will
> be automatically installed to profiles (see the role of profiles in Guix)
> alongside the package they belong to (see [`guix
> package`](https://guix.gnu.org/manual/en/html_node/Invoking-guix-package.html#package_002dcmd_002dpropagated_002dinputs),
> for information on how guix package deals with propagated inputs).

It was [fairly
easy](https://git.sr.ht/~droyo/guix-channel/tree/3bfc628072a3ea8dda59cde303edd7869e30c3d9/item/aqwari/namespace.scm#L107-112)
to extend the `bind->mount` routine to add propagated inputs of a package to
the mount set. However, this does not cover the case of dynamic libraries,
which are needed at run-time. But that point is moot, because I can't move
the dynamic libraries anyway.

### Dynamic linking

If I resolved the runtime dependency issue above, dynamically-linked
executables would still force me to include the `/gnu/store` directory in
the new namespace. It would be a much more fundamental change, but it's an
interesting thought exercise; what if guix (and nix before it) did not use
tools like [patchelf](https://github.com/NixOS/patchelf) and did not scan
all files it built for shebang lines to rewrite, and manipulated union mounts
instead? If mount namespaces were treated as first-class objects, and you could
modify and transition between them as easily as you can transition between
guix profiles today, how much of guix internals could be simplified? Would
the end-user experience be simplified?

Being realistic, I think once one started adding dynamic libraries to the
mount namespace, the limit of 500 directories gets really small, really
fast. It could be worked around by taking advantage of the fact that
collisions are unlikely, allowing us to move the mountpoints lower in the
directory hierarchy. But at that point, is the complexity worth it?

### Permissions

As you can see in the examples, in a default Linux installation,
mount namespaces can only be created by a privileged user. There is a
kernel parameter to allow unprivileged users to do this, but there are
security drawbacks; once a user has the ability to change what a name like
`/etc/sudoers` or `/etc/shadow` points to, they have the ability to trick
setuid-root programs like `mount` or `sudo` and compromise the system security.

I would lean towards addressing the problems that make unprivileged mount
namespaces insecure. It should be possible to build or modify a linux
distribution so it does not require setuid root programs. Moreover,
it should be possible to de-claw root, and remove much of root's
privilege, making it just another user that happens to have the uid 0,
granting privilege as needed to a few trusted system daemons using the
[capability](https://sites.google.com/site/fullycapable). Some
functionality could be replaced with [local
services](https://skarnet.org/software/s6/localservice.html) that can run in
their own, controlled environment, outside of user-controlled mount namespaces.

Similarly, I could expose a helper for the `exec` binary that constructs the
mount namespaces on behalf of the user. This does not really avoid the security
concerns, as any user can add a namespace description to the GNU store.

Another layer of protection would be to run `exec` in a [user
namespace](https://man7.org/linux/man-pages/man7/user_namespaces.7.html).
It's probably the more realistic approach, since the other approaches involve
undoing years of conventions. However, I hesitate to do it, because if possible
I'd like to build a set of orthogonal utilities for manipulating pid, net,
user and mount namespaces, and I don't want to merge the tools for mount
and user namespaces. Still, I'm open to it, but more research is needed.

### Transitioning between namespaces

It is trivial to modify a guix profile or switch to a new one -- you are just
updating a symlink. It is not so clear how one could transition from one
mount namespace to the next, or modify the existing namespace.

If one assumes all files are mounted from `/gnu/store`,
then the current mount namespace can be modified with
[*mount*](https://man7.org/linux/man-pages/man2/mount.2.html)(2) system
calls. However, privilege issues aside, you would not have visiblity to
mount sources outside of your namespace. Any child processes you spawned
could also prevent the namespace from being cleaned up.

For interactive use, it seems like a downgrade over symlinks. For now I
would stick to using this for processes that live and die in one namespace.

### Portability

Currently GUIX runs on Linux systems and GNU Hurd. Assuming the rest of guix
is ported to another operating system, for a `<namespace>` to be usable on
that OS it would need:

- [*chroot*](https://man7.org/linux/man-pages/man2/chroot.2.html)(2) or something like it (preferably something more secure).
- Bind mounts
- Union mounts

That may disqualify operating systems that could be made to run Guix otherwise.
However, OpenBSD and FreeBSD, at least, tick all the boxes.

## Parting thoughts

This was a fun experiment, and the end result is useful enough for me to
continue on with it. My plan is to use this work, perhaps together with [`guix
pack`](https://guix.gnu.org/manual/en/html_node/Invoking-guix-pack.html) to
deploy process trees managed by the [s6](https://skarnet.org/software/s6/)
supervision suite, that will run build servers, game servers, file servers,
irc bots, and whatever else I want to deploy, at home and in the cloud.

I found the Guix code base to be approachable. I was already familiar with
Scheme, having worked with Chez scheme and DrScheme (which grew into Racket)
in college. There are a wealth of examples in the Guix code base that I was
able to study that helped me with building the macro and the builder code. One
task I wavered on was how to transfer data between the "host-side" code and
"build-side" code; I had to think about different types of quotation and it
took awhile to get used to it.

Sometimes the error messages were terse or unhelpful. I didn't always get
stack traces when I wanted them. For debugging the build-side code, I found
the following process helpful; given an error, such as

	% guix build -f my-ns.scm -L $(pwd)
	substitute:
	substitute: updating substitutes from 'https://ci.guix.gnu.org'... 100.0%
	The following derivation will be built:
	  /gnu/store/xqi08dsq7gv8wk536gw77w3z2k5fgcs7-namespace.drv
	building /gnu/store/xqi08dsq7gv8wk536gw77w3z2k5fgcs7-namespace.drv...
	Backtrace:
	           7 (primitive-load "/gnu/store/26h4rv6dmq4jmnxfdifa2cg8acy?")
	In ice-9/eval.scm:
	    619:8  6 (_ #(#(#(#<directory (guile-user) 7fffeffcfc80>) "?") ?))
	    163:9  5 (_ #(#(#<directory (guile-user) 7fffeffcfc80>) #<outp?>))
	    159:9  4 (_ #(#(#<directory (guile-user) 7fffeffcfc80>) #<outp?>))
	In srfi/srfi-1.scm:
	   608:16  3 (map #<procedure 7fffeec01e20 at ice-9/eval.scm:383:13?> ?)
	   460:18  2 (fold #<procedure 7fffefb65310 at srfi/srfi-1.scm:608:?> ?)
	   609:38  1 (_ ",\n" 12)
	In unknown file:
	           0 (length+ ",\n")

	ERROR: In procedure length+:
	In procedure length+: Wrong type argument in position 1 (expecting proper or circular list): ",\n"
	builder for `/gnu/store/xqi08dsq7gv8wk536gw77w3z2k5fgcs7-namespace.drv' failed with exit code 1
	build of /gnu/store/xqi08dsq7gv8wk536gw77w3z2k5fgcs7-namespace.drv failed
	View build log at '/var/log/guix/drvs/xq/i08dsq7gv8wk536gw77w3z2k5fgcs7-namespace.drv.bz2'.
	guix build: error: build of `/gnu/store/xqi08dsq7gv8wk536gw77w3z2k5fgcs7-namespace.drv' failed

At first it seems inscrutable as it's given you positions in a generated
file rather than the file that generated it. But you can find the generated file
by looking at the derivation, which is visible on this line:

	build of /gnu/store/xqi08dsq7gv8wk536gw77w3z2k5fgcs7-namespace.drv failed

If you open that file, it looks something like this:

	Derive(
	  [ ... list of outputs ... ],
	  [ ... list of inputs ... ],
	  ["/gnu/store/26h4rv6dmq4jmnxfdifa2cg8acyyj5c1-namespace-builder",
	   "/gnu/store/cqz27fs66d2xm39rkn7i6095zqfiiv27-ns-helper.c",
	   "/gnu/store/pgj8653w17hsapbd1srlvd44rlnhbx8n-module-import"],"x86_64-linux",
	   "/gnu/store/1kws5vkl0glvpxg7arabsv6q9vazp0hx-guile-3.0.7/bin/guile",
	   [ ... command-line flags ... ],
	  [ ... outputs again ... ])

The file ending in `-builder` contains the generated builder program, with
all the substitutions already performed. It is all on one line, so I used
guile scheme's `,pp` helper to pretty-print it:

	> ,pp (quote (program ...))

And then I could read the code. Usually this made the error obvious, and if it
didn't, a few print statements got me the rest of the way there.
