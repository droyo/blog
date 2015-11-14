+++
date = "2015-06-30T22:25:33-05:00"
title = "Plumbing rules for Puppet manifests (part 2)"
description = "Quickly navigating puppet modules with Acme"
tags = ["puppet", "plan9", "acme"]
draft = true
+++

In a [previous post](/plumber-puppet/), I shared plumbing rules for navigation of puppet manifests. Those rules worked well when all of the puppet code I worked with was in a single, monolithic git repository, and there was a clear mapping between a module's name and its path in the file system. For a long time now, it has been a best practice to have one git repo per puppet module, usually named `<author>-<modulename>`.

At my workplace, we have begun moving to this model as well. While it has many benefits, it also meant that my previous plumbing rules were insufficient. 
