+++
draft = true
date = "2020-05-22T01:11:40Z"
title = "Adaptive sushi in Factorio"
description = "round n' round"
tags = ["factorio", "gaming", "math", "circuits"]
+++

I am a big fan of [Factorio](https://factorio.com/). It's truly
satisfying to design a production process, plug in raw
resources, and watch them flow through your factory,
transforming into finished products. For me, it tickles the
same part of my brain that programming does, and many
have drawn analogies between building a base in Factorio
and building software.

My current mod pack of choice is [Sea
Block](https://mods.factorio.com/mod/SeaBlock). Whereas in
vanilla factorio, your raw materials typically come from
mining, in Sea Block, you start on a tiny island and all of
your resources (including landfill) are derived from water.

To balance the infinite and ubiquitous resources, the authors
of the various mods introduced production chains that are
dizzying in their complexity, often containing several loops,
where the output of a later process is needed by the input
of an earlier process, and unwanted byproducts that take
care to get rid of. Some of the higher tier circuit boards, used
to build advanced factories, are the product of literally
hundreds of intermediate products.

Contrasting with vanilla Factorio, where you need a *lot* of
a handful of materials like iron and copper plates, in Sea Block,
you need a little of a *lot* of different materials. To take things
further, the mod even provides ways for you to compress items.
For example, you can cast metals into coils instead of plates,
which are four times as dense. This is important in controlling the
amount of conveyor belts or trains that you need to move products
around.

Some products you need so little of that dedicating even half
of the slowest belt is a waste -- the slowest belt can move 3.75
items per second on a single side, but some items you need *much*
less frequently. Nevertheless, you have to dedicate at least
half of the belt to every item you need to move. This can end
up taking a *lot* of space.

There is also the challenge that you need different amounts of
different materials at specific stages of the game. I recently watched
[rain9441](twitch.tv/rain9441/)'s play through of Sea Block where
he made extensive use of what are called "sushi belts" -- belts that
carried more than two items. He used circuitry to maintain a good
distribution of items on the belt. I was inspired.

However, it also made me think about how to make it better--a fixed
distribution of items meant the distribution of the belt contents could
not change dynamically to meet shifting factory demands.

Here is a high level description of the system:

* The belt *must* be a loop.
* If an item is placed on the belt, but does not complete a cycle, it
  has been consumed by the factory. Increase the rate at which the
  item is placed on the belt.
* If the item does complete a cycle, it was not consumed. Decrease
  the rate at which the item is placed on the belt.
* Enforce minimum and maximum rates. Since the consumption of
  an item is used as feedback, there needs to be _some_ of every
  item available for the system to work.

These four rules are simple, but for me the
implementation was daunting. Factorio provides a [circuit
network](https://wiki.factorio.com/Circuit_network) and various
combinators that can manipulate the signals on the network.

* Arithmetic combinators can apply arithmetic and bitwise operations
  to one or many signals.
* Decider combinators can compare and filter one or many signals.
* Belts and inserters can emit signals for the items that they pick up.
* Filter inserters can read signals to decide which items to place.

I spent an hour or so messing around with the combinators, but I
did not get very far. Combinators are powerful, but they are basic
building blocks. I struggled with the gap between my high-level
description and the low-level implementation details. It's also hard
to debug and experiment with circuits in-game, since the circuit
network runs at 60 ticks per second, it can be hard to follow what
is happening.

To understand the problem better, I wrote a simulation of factorio's
circuit network, this gave me the ability to pause, resume, and step
through the simulation. It also gave me more tools to verify the
circuit.

# Simulating a circuit network

At the simplest level, a circuit network can be described as an
undirected graph, where each node is a combinator (or some other
device that can produce a signal like a belt or inserter), and each
edge is either red or green.

The network executes one tick at a time. During one tick, a node
can broadcast a signal to its directly-connected neighbors. Propagation
of a signal happens one tick at a time.
