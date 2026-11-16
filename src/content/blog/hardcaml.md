---
title: 'Hardcaml; The Caml is what?'
description: 'A hardware programmer’s introduction to Hardcaml: what clicked, why I like it, and what still gets in the way.'
pubDate: '2026-11-15'
---

*All opinions are my own and do not represent any other entities mentioned here. This post was revised with AI assistance.*

What is **[Hardcaml](https://github.com/janestreet/hardcaml/)**, and why am I so interested in it?

I encountered the [Advent of FPGA challenge](https://blog.janestreet.com/advent-of-fpga-challenge-2025/), but ultimately didn't make a submission as my internship with [Astranis](https://www.astranis.com/) was beginning. Post coming soon on that. Shoutout [Anish Singhani](https://github.com/asinghani) for his support of the open-source community.

My background was SystemVerilog and VHDL, and originally Hardcaml was *very* difficult for me to comprehend. The unfamiliar part wasn't the idea of a circuit. It was thinking about a circuit as something I could construct using a functional programming language.

This is an introduction from that perspective: some familiarity with programming and basic digital hardware, but no OCaml knowledge assumed. It isn't a tutorial yet. The question here is **what Hardcaml does, why I want to use it, and what makes that choice less straightforward than the enthusiasm might suggest.**

## The Bread, Butter, and... Ham?

Hardcaml is an **[OCaml](https://ocaml.org/) library for describing and testing digital hardware**. OCaml is the programming language; Hardcaml gives it operations for constructing circuits.

The important distinction is that **the OCaml program runs to build a circuit description**. It isn't generally a software program that gets translated instruction-by-instruction into hardware.

That description can be simulated, or emitted as Verilog or VHDL for conventional hardware tools to process. For an FPGA project, the broad path looks like this:

**OCaml circuit-building code → Hardcaml circuit description → emitted RTL → synthesis and implementation tools → FPGA.**

RTL means *register-transfer level*: a description of hardware in terms of stored state and the logic operating on it. Synthesis turns that description into a network of hardware resources; implementation maps and connects those resources on the device.

So Hardcaml doesn't require Vivado to understand OCaml. It produces a hardware description that Vivado can consume.

This also isn't a promise that an arbitrary software algorithm becomes an efficient circuit automatically. You still decide what is parallel, where state lives, and how data moves between clock cycles. Hardcaml changes how you **express and construct** that architecture; it doesn't remove the need to design it.

That is the mental shift that took me a while: the software constructing the hardware and the behavior of the resulting hardware are two different things.

## In Action;

Let's start with a small example. Suppose I have a list of input signals, and I want to apply the same hardware operation to each one.

In OCaml, a function can be passed to another function as an argument. A function that accepts or returns functions is called a **higher-order function**. Here, `apply_to_all` accepts an operation and uses `List.map` to apply it to every input:

<div class="wide-code">

```ocaml
open Hardcaml
open Signal

let apply_to_all operation inputs =
  Stdlib.List.map operation inputs

let increment_all inputs =
  apply_to_all (fun x -> x +:. 1) inputs

let invert_all inputs =
  apply_to_all (fun x -> ~:x) inputs

let double_all inputs =
  apply_to_all (fun x -> x +: x) inputs
```

</div>

You don't need to memorize the syntax. `fun x -> ...` defines an anonymous function; the three operations describe incrementing a signal, inverting its bits, and adding it to itself. `+:.` adds an integer constant to a signal, while `+:` adds two signals.

The inputs here are Hardcaml `Signal.t` values: descriptions of hardware signals, **not ordinary integers containing their current simulated values**.

During **elaboration**—running the code that constructs the circuit—OCaml walks the list and builds the corresponding operations in Hardcaml's circuit graph. That graph records the operations and their connections.

For eight 32-bit inputs, `increment_all` describes eight independent 32-bit increments. It does **not** describe a processor looping through eight values over eight clock cycles. There is no clock or register in this example: it describes combinational logic, whose outputs depend on its current inputs. The arithmetic results stay at the input width, so overflow is discarded.

The list traversal happens while constructing the circuit. It isn't itself hardware that needs to execute on the FPGA.

For readers familiar with SystemVerilog, a hand-written equivalent of the *incrementing behavior* could look like this:

<div class="wide-code">

```verilog
module vector_increment #(
    parameter int N = 8,
    parameter int W = 32
)(
    input  logic [W-1:0] data_in  [N],
    output logic [W-1:0] data_out [N]
);

    for (genvar i = 0; i < N; i++) begin : gen_inc
        assign data_out[i] = data_in[i] + 1'b1;
    end

endmodule
```

</div>

SystemVerilog can absolutely express this, and its `generate` loop also expands during elaboration rather than iterating over clock cycles. It supports synthesizable functions too. Neither repetition nor arithmetic is the exclusive selling point here.

What interests me is the **way the generator is composed**. In OCaml, I can pass a circuit-building operation around as an ordinary function. The same traversal can build increments, inversions, or something more involved, without modifying the traversal itself.

The little example isn't shorter by some revolutionary amount. It illustrates a mechanism that becomes more interesting as the circuit-building operations grow.

## Why I Want to Use It

The appeal is having the programming language available while constructing hardware.

With Hardcaml, I can think about an operation independently of how I apply it across a collection of signals. Functions can build smaller circuit fragments, and other functions can combine those fragments into a larger design. That lets the organization of the source reflect the ideas used to construct the circuit, rather than only its final wiring.

SystemVerilog has its own tools for abstraction and reuse. My preference here isn't based on it being incapable of describing the same hardware. It's about finding OCaml's function-based approach compelling for writing **reusable circuit generators**.

Testing is another part of the appeal. Hardcaml includes [simulation facilities](https://github.com/janestreet/hardcaml), and the surrounding ecosystem includes [waveform rendering](https://github.com/janestreet/hardcaml_waveterm) and [formal verification tools](https://github.com/janestreet/hardcaml_verify). Verification means checking whether a circuit behaves as intended; it isn't simply the nonsynthesizable remainder of a language.

The circuit description and its tests can live in the same programming environment, even though they do different jobs. Circuit-building code constructs the hardware; test code supplies inputs and checks behavior. That doesn't make verification automatic, but it's a useful way to organize the work.

<!-- EDITORIAL: Add a short example from your own project here: what circuit you were building, what Hardcaml made easier, and one concrete result. Don't claim a productivity or verification benefit you haven't observed. -->

The enthusiasm isn't that I can stop thinking about hardware. It's that I get a richer set of software tools for expressing the hardware I mean.

And that's **AWESOME**.

## Linguistic Quirks and Practical Trade-offs

The obvious question is: **why wouldn't I use it over SystemVerilog?**

First, the learning curve is real. Coming from SystemVerilog and VHDL, I had to learn both unfamiliar OCaml syntax and a different way to organize circuit-building code. Functional programming treats functions as values that can be passed around and combined. That's useful here, but understanding the hardware doesn't mean that style will immediately feel natural.

Second, there's workflow plumbing. One of the things I find unwieldy is the path from an OCaml circuit to a design in Vivado. Emitting RTL is not the same as having a complete FPGA project: the design still needs its top-level connections, clocks, timing constraints, and any device-specific integration. Those responsibilities exist with other HDLs too, but Hardcaml doesn't make them disappear.

<!-- EDITORIAL: Name the particular step that frustrated you in your own Vivado workflow. Distinguish a Hardcaml limitation from project setup or an unfamiliar tool. -->

Third, there's the surrounding team and ecosystem. A generator written in OCaml introduces another language for collaborators to understand and maintain. Existing SystemVerilog modules, test environments, and established workflows can be good reasons not to change.

As a mentor of mine put it:

> "Changing software [in smaller companies] is like turning a speedboat, but changing hardware [anywhere] is like turning a freighter." — Bradley Boccuzzi

Finally, cleaner generator code doesn't guarantee better hardware. The eight independent increments above still describe parallel work; rewriting them with a higher-order function doesn't automatically share resources or add a pipeline. Area, timing, and correctness still depend on the architecture and the downstream tools.

If a project already fits an established SystemVerilog flow, staying there may be the sensible choice. Hardcaml has to earn its place through what it makes easier to express, test, or maintain—not just through being different.

## The Elephant in the Car: Jane Street;

The name that comes up repeatedly around Hardcaml is **[Jane Street](https://www.janestreet.com/)**.

For me, the relevant part isn't its recruiting reputation. It's the investment in hardware tools and their use on real problems. The public [ZPrize projects](https://zprize.hardcaml.com/) are a concrete example, and Andrew Ray's [Signals and Threads interview](https://signalsandthreads.com/programmable-hardware/) gives background on Hardcaml's development and use at Jane Street.

That is a reason to take the tool seriously, not proof that every problem I encounter will already have been solved. Nor does it mean other hardware languages lack production users. It means there are substantial examples to study beyond a neat language demonstration.

The history deserves its own discussion. For this introduction, the important point is that Hardcaml comes with both a way to construct circuits and an ecosystem for doing something useful with them.

## Where This Goes Next

Hardcaml interests me because it lets me organize hardware construction using ordinary OCaml functions and abstractions, while still making explicit architectural decisions.

The cost is learning that environment and fitting it into a hardware workflow. That's a trade-off, not a universal upgrade over SystemVerilog.

This post is the **what and why**. A separate tutorial series can tackle the **how**: building a small circuit, simulating it, and getting the generated RTL into an FPGA project. The distinction to carry into that series is simple: **OCaml runs to construct the circuit; the circuit then has its own hardware behavior.**
