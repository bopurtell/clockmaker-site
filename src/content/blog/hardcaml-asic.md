---
title: 'Introducing hardcaml_asic: Camels like chips'
description: 'An introduction to an OCaml-native toolkit for describing ASIC targets, preparing hardware for implementation, and managing the path from Hardcaml to silicon.'
pubDate: '2026-11-25'
---

<!--
WORKING DRAFT — revise in your own voice before publication.
The publication date is provisional. This site's collection does not have a
draft flag, so keep this out of a production deployment until ready.

Editorial scope: a release introduction, not an installation tutorial.
The reader should leave understanding why the library exists, what it adds
to Hardcaml, and what they can actually do with the first release.
-->

In my [previous post](/blog/hardcaml/), I talked a bit about why I like Hardcaml, and why I find the idea of constructing hardware in a functional programming language so compelling.

Towards the end of that post, I mentioned one of the less pleasant parts of working with Hardcaml: what happens *after* you've generated your Verilog?

Generating a circuit is one thing. Getting that circuit onto a piece of silicon is another entirely. FPGAs are some code and a handful of **xdc** files; ASIC is an entire `nother camel.

I'd like to introduce a project I've been working on, **[hardcaml_asic](https://github.com/bopurtell/hardcaml_asic)**, an open-source OCaml library and command-line toolkit for taking Hardcaml designs into an ASIC implementation flow.

This is an in dependent project, and in no way affiliated with Jane Street.

## So, I have my Verilog. Now what?

<!-- AUTHOR: A short personal anecdote here would be stronger than a generic
explanation. What was the first design or experience that made you want to
write hardcaml_asic? What were you doing manually that felt unnecessary? -->

Suppose I've written a little circuit in Hardcaml, simulated it, and emitted its Verilog. At this point I have a description of digital hardware, but not yet a complete description of the chip I want to manufacture.

Among other things, I still need to specify:

- What fabrication technology I'm targeting.
- How my design connects to the outside world.
- What clocks and timing constraints the implementation must satisfy.
- Which technology-specific hardware resources I intend to use.
- How I'm going to run the physical design tools, and keep track of their results.

An ASIC (*application-specific integrated circuit*) is different from an FPGA in that we aren't configuring an existing array of programmable logic. The implementation tools have to work out which cells to use, where to physically place them, and how to connect them on the chip.

None of that is a deficiency in Hardcaml. Describing a circuit and implementing it are different problems.

What I find interesting, though, is that I've just spent all this time using OCaml to express my hardware, only to step out of that environment to describe much of the information necessary to implement it. Can I bring some of that information into the same program?

## A chip as an OCaml value

The central abstraction in `hardcaml_asic` is the **chip target**.

A Hardcaml circuit describes the hardware I want. A chip target describes the circuit alongside the implementation intent needed to turn it into an ASIC.

In particular, the library distinguishes three things:

- **Harness:** the interface and geometry within which my design must fit.
- **Technology:** the process technology, standard cells, and available hard macros.
- **Flow:** the collection of tools and associated methodology that will implement the design.

For the first release, these aren't arbitrary combinations. The supported stack is [Tiny Tapeout](https://tinytapeout.com/), the IHP SG13G2 CMOS5L technology variant, and [LibreLane](https://librelane.readthedocs.io/).

The distinction is still important. The meaning of a clock constraint, for example, doesn't belong exclusively to a particular physical design program.

## In Action;

Let's look at a deliberately uninteresting circuit: an eight-bit register that captures the Tiny Tapeout input bus. When the design is enabled, we expose that registered value on the output bus.

There's nothing particularly ASIC-specific about the register itself.

<div class="wide-code">

```ocaml
open! Core
open! Hardcaml
open! Hardcaml_asic

module Observable = struct
  module I = Tt_cmos5l.I
  module O = Tt_cmos5l.O

  let name = "tt_um_asic_observable"

  let create _ (i : _ I.t) =
    let observed =
      Signal.reg
        (Signal.Reg_spec.create
           ~clock:i.clk ~clear:(Signal.(~:) i.rst_n) ())
        i.ui_in
    in
    { O.uo_out = Signal.mux2 i.ena observed (Signal.zero 8)
    ; uio_out = Signal.zero 8
    ; uio_oe = Signal.zero 8
    }
  ;;
end
```

</div>

The library supplies the Tiny Tapeout input and output interfaces, but the logic inside `create` is still ordinary Hardcaml. I decide what my register does, how it resets, and what the output enable means.

The more interesting part is the declaration surrounding it. Here is an excerpt from the [observable example](https://github.com/bopurtell/hardcaml_asic/blob/oxcaml/examples/tt_bundle_example.ml) shipped with the library:

<div class="wide-code">

```ocaml
Chip_target.create
  ~name:"asic_observable"
  ~metadata
  ~design:(module Observable)
  ~target:
    { harness = Tiny_tapeout { tiles = T6x4 }
    ; technology = Technology.ihp_sg13cmos5l
    }
  ~flow:(Flow.librelane
           ~overrides:(Tt_cmos5l.overrides ()) Hardening)
  ~clocks:
    [ { Clock.port = "clk"
      ; period = Time_float.Span.of_sec (1. /. 48e6)
      }
    ]
  ~timing
  ~pinout
  ~policy:(Resource_policy.create ~default:Flops []
           |> Or_error.ok_exn)
  ()
|> Or_error.ok_exn
```

</div>

I've left out the definitions of `metadata`, `timing`, and `pinout` for brevity. The complete example provides them.

What we've expressed here is a circuit targeting a 6-by-4 Tiny Tapeout tile allocation, a particular IHP technology, a LibreLane hardening flow, and a requested clock frequency of 48 MHz. In addition, we've declared how memory resources should be implemented, even though this particular circuit doesn't use any.

All of these are inputs to implementation. Declaring a 48 MHz clock does not, of course, mean my resulting design will actually meet timing at 48 MHz.

## On the subject of memory;

One part of the library that I think is particularly interesting is the handling of hardware resources.

Let's say I have a memory in my design. There are several ways I might want to represent it:

- A behavioral model for simulation.
- An implementation constructed from flip-flops.
- A technology-specific SRAM macro.

An SRAM macro is a predesigned physical memory block. Unlike constructing a large array of registers from standard cells, using one means incorporating an already characterized piece of hardware into the design.

I don't necessarily want to rewrite the rest of my circuit whenever I change how its memory is realized.

`hardcaml_asic` provides `Single_port_ram` as a common memory abstraction and a `Resource_policy` to specify how resources may be implemented.

For instance, these two policies express different physical intentions:

```ocaml
let flops =
  Resource_policy.create ~default:Flops []
  |> Or_error.ok_exn
;;

let exact =
  Resource_policy.create ~default:Exact []
  |> Or_error.ok_exn
;;
```

The former requests a flop implementation. The latter requires an exact supported macro match for requested resources, rather than silently falling back to flops.

The key distinction is that the memory's architectural behavior remains a contract. Changing its implementation is not permission to change its width, depth, or read latency to fit whatever happens to be available in the technology.

For this release, the supported physical SRAM macro is the stock IHP 256-by-16 single-port macro, under a restricted physical profile. This isn't a general-purpose memory compiler.

<!-- AUTHOR: Optional figure showing the same logical memory branching into
behavioral simulation, flop-based RTL, and a hard macro. Save actual area,
timing, and memory comparisons for the silicon series. -->

## Where OCaml stops and the tools begin

An important design decision in `hardcaml_asic` is to separate the generation of implementation inputs from the execution of the implementation tools.

<p>
<strong>
Hardcaml circuit + chip target →<br />
<span style="display: inline-block; padding-left: 2em;">
elaboration and target resolution →<br />
immutable implementation bundle →<br />
synthesis and physical implementation →<br />
reports and retained results
</span>
</strong>
</p>

The **bundle** is the handoff between those two environments. It contains generated RTL, timing constraints (SDC), LibreLane configuration, Tiny Tapeout metadata, and the declared source inputs, together with a manifest identifying the build.

You don't need a PDK (process design kit), Docker, or a running EDA tool to generate that bundle.

If you have the library's OCaml dependencies installed, the observable example can be run from the repository with:

```sh
DEMO=$(mktemp -d)
dune exec examples/tt_bundle_example.exe -- observable "$DEMO/bundle"
cat "$DEMO/bundle/manifest.json"
```

There is now a concrete set of implementation inputs that can be inspected before asking any physical design tool to do work.

<!-- AUTHOR: Insert a real bundle directory listing or a cropped manifest
here, ideally followed by a single real layout image as a teaser. Do not
invent output; capture it using the final release revision. -->

The bundle has an identity derived from its contents and recorded inputs. If I change my design or its declared configuration, I should generate another bundle rather than edit the old one.

This also separates a *build* from a *run*. I can execute the same set of implementation inputs more than once and retain the results of each attempt independently. That distinction becomes quite useful when investigating timing failures or comparing implementation choices.

The other half of the project is the installed command-line tooling. It manages the external flow, exposes physical stage boundaries and supported checkpoints, and provides reports from retained results.

It doesn't replace Yosys, OpenROAD, or LibreLane. Those tools are still responsible for doing the actual implementation work.

## What can I actually do with it?

As of the first release, the supported scope is intentionally narrow:

| | Support |
| --- | --- |
| Harness | Tiny Tapeout 6×4 |
| Technology | IHP `ihp-sg13cmos5l` |
| Physical flow | LibreLane Classic |
| Memory | Behavioral, explicit flops, exact stock 256×16 SRAM |
| Workflow | Bundle generation, implementation, supported checkpoints, reporting |

The hardware library and command-line toolkit can be used independently of any hardware workbench. The tested OCaml environment uses OxCaml, and actual physical execution requires additional tools and technology collateral.

There are limitations, particularly around macro implementation profiles and restarting physical stages. Other process technologies and commercial flow adapters aren't supported in this release.

And perhaps most importantly: a completed implementation is not, by itself, a signoff verdict. There is still quite a bit of electrical and functional verification to do before one ought to manufacture a chip.

## Proxime

This post has been about expressing the intent of an ASIC project and where that fits into the Hardcaml ecosystem.

The next thing I want to explore is the hardware itself. In a separate **Hardcaml to Silicon** series, I'd like to walk through the process of writing and simulating a circuit, implementing it with open-source tools, looking at the resulting layout, and working through some of the architectural decisions that actually affect silicon area and timing.

For those interested in trying the library in the meantime:

- [hardcaml_asic source and quick start](https://github.com/bopurtell/hardcaml_asic)
- [Documentation](https://github.com/bopurtell/hardcaml_asic/tree/oxcaml/docs)
- [Standalone FIFO and SRAM examples](https://github.com/bopurtell/hardcaml_asic_example/tree/oxcaml)

<!-- AUTHOR: Add the immutable release/tag link once published. -->
