+++
draft = true
date = "2020-05-22T01:11:40Z"
title = "Simulating factorio's circuit network"
tags = ["factorio", "gaming", "circuits", "ocaml"]
+++

The game [Factorio](https://www.factorio.com/) provides a circuit network
that can be used to control portions of the factory based on arbitrary
logic. For example, you could halt production of an item until there is
a shortage. Or you could disconnect portions of the factory from power
generation if there is a shortage in your power plant. Here is an example
from the Sea Block mod:

<img src="/img/factorio/circuits-sulfur.jpg" style="width: 100%" />

There are two decider combinators in the south west that will output
one green square if there is a purified water or sulfur shortage,
respectively. These connect via the green wire to a pump in the northeast
that will turn on if it receives at least one green square. Otherwise,
no sulfuric waste water will pass through the pump, and it can be exported
to other parts of the factory. Sulfuric waste water purification provides
more than enough sulfur _and_ purified water to completely sustain this
particular production chain.

In a circuit network, a signal is an arbitrary symbol plus a magnitude,
in the range (-2³¹ .. 2³¹-1). In the example, the symbol is a green square and
its magnitude is 1.  A circuit network can carry zero or more signals
simultaneously. Signals for the same symbol are combined with addition,
and signals with zero magnitude are not transmitted.

    module Signal = struct
      include Map.Make(String)

      let combine name s1 s2 =
        match Int32.add s1 s2 with 0l -> None | x -> Some x

      let add_signal name strength m =
        let combine_signals old =
          combine name (Option.value old ~default:0l) strength in
        update name combine_signals m

      let print = iter (Printf.printf "%s = %ld\n")
    end

There are various sources that can be connected to a circuit network
to emit signals. An inserter can emit the items it picks up as a signal.
A belt can emit the items that pass over it. Chests and storage tanks
can emit their contents, and so on.

    let chest = Signals.empty
      |> Signals.add_signal "iron-ore" 200l
      |> Signals.add_signal "iron-ore" (-50l)
      |> Signals.add_signal "copper-plate" 100l

    # Signals.print chest;;
    copper-plate = 100
    iron-ore = 150

When two circuit networks are connected, a single network is formed,
and the new contents are the result of adding the signals of one network
to the other, in any order.

    let connect s1 s2 = Signals.(union combine s1 s2)

    let storage_tank = Signals.empty
      |> Signals.add_signal "petroleum-gas" 25000l

    # Signals.print (connect chest storage_tank);;
    copper-plate = 100
    iron-ore = 150
    petroleum-gas = 25000

