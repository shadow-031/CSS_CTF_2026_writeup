# Silicon Snare - CTF Writeup

**Category:** Hardware / Reverse Engineering

**Flag:** `CSSCTF{01010000010000010101001101010011}`

This writeup is written for beginners, so it spends extra time explaining the basic ideas (logic gates, how to read a schematic, what a wire crossing actually means) before getting into the solve itself. If you already know digital logic well, you can skim the early sections.

## 1. Challenge Overview

The challenge gives you a schematic of an "optical routing matrix", which is just a fancy name for a diagram of logic gates connected together with wires. The story is that this circuit is a fail-safe system for a relay station, and there is one special wire in the middle of the diagram called `OVERRIDE`.

The goal of the challenge is simple to state even if it is not simple to solve by eye:

> Find the 32-bit input pattern that makes `OVERRIDE` output a logic 1 (true).

The circuit has 32 inputs, labeled `I00` through `I31`. The challenge tells you these inputs are arranged around the outside of the diagram in a circle, read clockwise starting from `I00`.

The single most important hint given in the challenge is this line:

> "The only thing to follow is the wire."

That line is telling you directly that this is not a cryptography puzzle and not a programming puzzle. It is a tracing puzzle. You are meant to physically follow wires through the diagram, gate by gate, until you work out what value each input needs to have.

## 2. Background: What Is a Logic Gate?

If you have not worked with digital logic before, here is the short version. A logic gate is a small component that takes one or two input signals, each of which is either `0` (false, off) or `1` (true, on), and produces one output signal that is also `0` or `1`. The rule for how the output is determined from the inputs is called the gate's Boolean function.

This challenge's schematic uses four kinds of gates, each with its own symbol in the diagram and its own rule, explained in a legend on the schematic itself.

### Flow gate (AND)

```
Y = A AND B
```

The output `Y` is `1` only when both `A` and `B` are `1`. If either input is `0`, the output is `0`. Think of it like a rule that says "both of these need to be true for this to be true."

### Negate gate (NOT)

```
Y = NOT A
```

This gate has only one input. It simply flips whatever comes in. A `0` becomes a `1`, and a `1` becomes a `0`.

### Merge gate (XOR)

```
Y = A XOR B
```

The output is `1` when the two inputs are different from each other (one is `0` and the other is `1`), and `0` when they are the same (both `0` or both `1`).

### Shift gate

This one is a bit more unusual because it has two outputs instead of one:

```
Y0 = A OR B
Y1 = A AND NOT B
```

The schematic distinguishes which input is `A` and which is `B` by shape, the pointed end of the gate symbol is `A`, and the flat end is `B`. `Y0` is a simple OR, meaning it is `1` if either input is `1`. `Y1` is `1` only when `A` is `1` and `B` is `0` at the same time.

Knowing just these four rules is really the entire "theory" needed for this challenge. Everything from here on is just carefully applying these rules while tracing wires.

## 3. The Biggest Trap: Wire Crossings Are Not Connections

This is the single most important thing to understand before attempting to trace this schematic, and it is also exactly where most people would go wrong.

In a lot of diagrams, if two lines cross on the page, you might assume they are connected, like two roads meeting at an intersection. In proper schematic drawing convention, and in this challenge specifically, that assumption is wrong.

Two wires crossing on the page are, by default, NOT electrically connected. They are just two separate wires that happen to be drawn overlapping each other visually because the diagram is two-dimensional and the actual circuit is not. Think of it like two pieces of string laid across a table without touching, one just happens to pass over the other.

The schematic only considers two wires to be truly joined (or a single wire to be split into two) at a specific marked point, called a node, which is shown as a small solid dot where the lines meet. If there is no dot, the lines crossing at that spot are unrelated to each other and you must keep following your original wire straight through the crossing as if the other line was not there.

To summarize the rule:

```
wire crossing (no dot)
    -> keep following the original wire, ignore the other line entirely

connection dot (filled circle)
    -> the wires actually do join together or split apart here
```

Because this distinction can be hard to see clearly in a flattened image (like a PNG or JPG), it helps enormously to open the actual SVG file if one is provided. An SVG is a vector file, meaning you can zoom in as far as you want without the lines turning into blurry pixels, which makes it much easier to tell a genuine connection dot apart from two lines that simply overlap at that zoom level.

## 4. Strategy: Work Backwards From OVERRIDE

With 32 separate inputs and a large tangle of gates between them and the center, trying to trace every single input forward from the outside in, hoping it eventually reaches the middle correctly, is slow and error prone.

A much more manageable approach is to work backwards starting from the thing you actually care about, the `OVERRIDE` node itself, and ask at each step: "what does the gate feeding into this need as input in order to produce the value I want?"

So the process looks like this:

```
OVERRIDE
   ^
   | (what value does this gate need, to output a 1?)
previous gate
   ^
   | (and what does THIS gate need, to produce that needed value?)
previous gate(s)
   ^
   |
   ...
   ^
   |
I00 through I31
```

At each gate along the way, you substitute in its Boolean rule from section 2, and figure out backwards what input combination is required. For example, if you find that a Flow (AND) gate needs to output `1`, you immediately know both of its inputs must individually be `1`, since AND only outputs `1` when both inputs are `1`. If you find a Negate (NOT) gate needs to output `1`, you know its single input must be `0`, since NOT just flips the value.

Doing this repeatedly, gate by gate, turns the physical wiring diagram into what is called a Boolean dependency tree, basically a plain logical description of exactly what each input needs to be, built up purely by walking backwards through the gates one at a time.

## 5. Following the Wires to the Inputs

The 32 inputs are laid out in a ring around the outside of the schematic, like numbers around a clock face, going clockwise starting at `I00`:

```
             I00
              ^
       I31          I01
   I30                  I02
 I29                      I03
 ...
             I16
```

Starting from `OVERRIDE` and working backwards as described in section 4, I traced each wire outward, carefully following the correct line through every crossing (ignoring crossings with no dot, respecting actual connection dots), through every gate (applying the correct Boolean rule for that gate type), until each path eventually terminated at one of the 32 labeled inputs around the edge.

Doing this for every single one of the 32 inputs, the full set of required values worked out to:

| Input | Value | | Input | Value | | Input | Value | | Input | Value |
|-------|-------|-|-------|-------|-|-------|-------|-|-------|-------|
| I00   | 0     | | I08   | 0     | | I16   | 0     | | I24   | 0     |
| I01   | 1     | | I09   | 1     | | I17   | 1     | | I25   | 1     |
| I02   | 0     | | I10   | 0     | | I18   | 0     | | I26   | 0     |
| I03   | 1     | | I11   | 0     | | I19   | 1     | | I27   | 1     |
| I04   | 0     | | I12   | 0     | | I20   | 0     | | I28   | 0     |
| I05   | 0     | | I13   | 0     | | I21   | 0     | | I29   | 0     |
| I06   | 0     | | I14   | 0     | | I22   | 1     | | I30   | 1     |
| I07   | 0     | | I15   | 1     | | I23   | 1     | | I31   | 1     |

So the inputs that need to be `1` are:

```
I01, I03, I09, I15, I17, I19, I22, I23, I25, I27, I30, I31
```

Every other input (all 20 of the remaining ones) needs to be `0`.

## 6. Building the 32-Bit Flag String

The challenge states that `I00` is the first bit of the final answer, and the bits are read in order from `I00` through `I31`. So the last step is just lining up the 32 values in order and reading them off as one long binary string.

Breaking it into groups of 8 bits to make it easier to check by eye:

```
I00 I01 I02 I03 I04 I05 I06 I07
 0   1   0   1   0   0   0   0

I08 I09 I10 I11 I12 I13 I14 I15
 0   1   0   0   0   0   0   1

I16 I17 I18 I19 I20 I21 I22 I23
 0   1   0   1   0   0   1   1

I24 I25 I26 I27 I28 I29 I30 I31
 0   1   0   1   0   0   1   1
```

Reading each group of 8 left to right and joining them all together:

```
01010000
01000001
01010011
01010011
```

gives the full 32-bit string:

```
01010000010000010101001101010011
```

That is exactly 32 characters long, which matches the challenge's stated flag format.

## 7. Final Flag

Wrapping that 32-bit string in the challenge's flag format gives:

```
CSSCTF{01010000010000010101001101010011}
```

## Key Takeaways

**This challenge is really a careful reading exercise, not an algebra problem.** The four gate rules (AND, NOT, XOR, and the two-output shift gate) are all simple on their own. The actual difficulty is entirely in tracing which wire goes where without making a mistake.

**Never assume a wire crossing is a connection.** This is the single trap built into the schematic. Two lines crossing on the page are just passing over each other visually unless there is an explicit connection dot marking a real junction. Treating every crossing as a connection will silently wire up the circuit incorrectly and produce a wrong answer that still looks plausible.

**Work backwards from the thing you want, not forwards from everything you have.** Starting at the single `OVERRIDE` output and tracing backwards toward only the inputs that actually matter is far less error-prone than trying to simulate all 32 inputs forward through the entire tangle of gates at once.

**Use a vector file (SVG) over a flattened image whenever one is available.** Zooming into a raster image eventually just gives you blurry pixels, while an SVG stays crisp at any zoom level, which makes the difference between a real connection dot and an incidental crossing much easier to see clearly.

**Read the input ordering exactly as specified.** The challenge explicitly defines `I00` as the first bit and the ordering as clockwise, so the final answer is just a matter of lining up the 32 traced values in that exact order and concatenating them, with no further transformation needed.
