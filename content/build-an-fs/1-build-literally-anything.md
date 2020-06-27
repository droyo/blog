+++
date = "2020-05-11T18:00:00Z"
title = "Let's build a high performance file system! Part 1"
tags = ["ocaml", "plan9", "9P", "fs", "performance", "programming"]
description = "Build literally anything"
draft = true
+++

This is part 1 of a [multi-part series](./index) to build a
high performance user space file system in OCaml.

# Part 1: Build literally anything

The objective of this article is to get something running as
quickly as possible. There will be no attempt made to make it
efficient, durable, or reliable. To start, we won't even be
writing to disk. The objective is to work out all the build
issues and setup a test harness to enable future development.
