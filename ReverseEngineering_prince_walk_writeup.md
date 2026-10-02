# Prince Walk - CTF Writeup

**Category:** Reverse Engineering

**Flag:** `CSSCTF{P12INC3_0R_P1NC3?}`

## Challenge Overview

The challenge presents itself as a terminal "survey simulation". The player's avatar starts at coordinates `(1, 1)` and the stated goal is to reach `(999999, 999999)`. Walking there one tile at a time with WASD, with trees obstructing the path, is obviously not a realistic way to actually finish the game. The binary's own strings make the intended solution path fairly obvious:

```
A terminal sandbox for PINCE Int32 memory-editing practice.
```

PINCE is a well known Linux GUI memory editor and debugger, similar in spirit to Cheat Engine. The player's X and Y coordinates are stored as plain 32-bit integers, so the "intended" solve is to attach PINCE, scan memory for those two values, overwrite both with 999999, and then just step the avatar onto the goal tile.

I took a different route and solved it entirely statically, without ever running the actual game.

## Recon

The binary, `prince_walk`, is a 64-bit, little-endian, position independent, stripped x86-64 executable linked only against libc. A few things stood out while poking around it:

- There is no flag string anywhere in the binary, which immediately rules out a simple strings grep solve.
- The `.rodata` section is roughly 69 KB, far larger than what the UI text alone would need. That strongly suggested a large embedded data table was hiding in there, and indeed there is one starting at `0x4a80`, consisting of 0x2080 entries of 8 bytes each.
- Terrain appears to be hashed from coordinates, with a small tile table at `0x4450` holding characters like `......,,TT~#`, plus some joke "landmark" messages and achievements tied to specific special coordinates.
- The binary calls `prctl`, which is used here to restrict debugger attachment. The game prints something along the lines of "Attach opt-in unavailable; OS debugger rules apply", which is a none-too-subtle nudge against just attaching a conventional debugger directly.

## The Destination Check

The function responsible for validating the destination, located at `0x2c56`, begins with a very direct comparison:

```asm
cmp x, 0xf423f    ; 999999
jne fail
cmp y, 0xf423f
je  run_vm
```

`0xf423f` in hex is exactly 999999 in decimal, so this function only proceeds past the initial check when both x and y equal 999999 precisely. Critically, the flag itself is not stored anywhere as a string, it is computed at runtime by a small virtual machine that only runs once this coordinate check passes.

## The Virtual Machine

The VM's internal state and execution model break down into a few pieces:

**State.** There are 16 32-bit registers, initialized as:

```
st[i] = H(((i+1) * 0x9E3779B9) ^ (x ^ rotl(y, i+1)))
```

along with a single accumulator:

```
acc = H((rotl(y, 19) + x) ^ 0xA4093822)
```

where `H` is a function referred to in the binary as "lowbias32", a well known integer hash finalizer, implemented at `0x2c18`.

**Program.** The VM then executes 0x2080 steps. At each step `pc`, it reads an 8-byte entry `(w0, w1)` from the big data table at `0x4a80`, at index:

```
(pc * 0xD9 + 0xCD6) mod 0x2080
```

From there it derives two working values:

```
r = H(pc * 0x9E3779B9 ^ acc) ^ w0
v = H(pc + acc + 0x6A09E667) ^ w1
```

**Decode.** The low byte of `r` is the opcode, with additional bit fields of `r` providing the operands `d`, `a`, `b`, and `s`. There are seven valid opcodes (`0x8B`, `0x48`, `0x31`, `0x71`, `0xB3`, `0x12`, `0xFF`) covering register arithmetic, rotation, and further hashing operations. Any opcode outside that set causes the VM to abort immediately.

**Output operation (0x83).** This particular opcode writes a single byte into a 128-byte output buffer. The byte value is:

```
(v ^ st[a] ^ rotl(st[b], s)) & 0xFF
```

and it is written at index `v >> 25`. Each slot in the buffer may only ever be written once, so a given run either fills in all 128 slots correctly or it doesn't produce a usable result at all.

**Avalanche effect.** After every single step, the accumulator `acc` is updated based on the current register state. This means the entire 0x2080 step run is one long avalanche, any difference in the initial x or y values cascades through every subsequent step and produces complete garbage output rather than anything resembling a valid message. This is precisely why brute forcing or guessing at intermediate values would not work, only the exact correct coordinates produce a coherent result.

## Final Validation

After all 0x2080 steps complete, the code checks that all 128 output buffer bytes have actually been written. The buffer's layout is:

```
[length byte][message bytes, length of them][FNV-1a hash of the message, as a 4-byte little-endian u32]
```

The decoded message must consist of printable ASCII or newline characters, and the FNV-1a hash stored at the end must match a freshly computed FNV-1a hash of the message bytes. Only once both of those checks pass does the function return the message as successfully decoded. This checksum is effectively a self-authentication step: the VM's output only validates as correct when the decode genuinely succeeds with the right coordinates, which rules out any chance of a false positive from a near-miss computation.

## Solving

Since the entire flag generation depends only on the two input coordinates and the static data already embedded in the binary, the cleanest solve was simply to call the function directly with the target coordinates rather than trying to reconstruct or reverse engineer the VM's logic into an independent decoder by hand.

### Approach 1: Running the Binary's Own Code

I mapped the ELF image into executable memory directly, taking care that RIP-relative references into the data table still resolved correctly since the mapping preserved the binary's internal layout. The single external call to `memcpy@plt` was redirected to the real libc implementation, and then the destination check function at `0x2c56` was called directly with the target coordinates and a 256-byte output buffer.

```
(1, 1)              -> failed
(999999, 999998)     -> failed
(999999, 999999)     -> Developer:
                        "No you didn't."

                        MISSION COMPLETE

                        CSSCTF{P12INC3_0R_P1NC3?}
```

Testing a near miss coordinate like `(999999, 999998)` deliberately confirmed that the check really was exact, not off by one or otherwise lenient, and that the avalanche behavior in the VM meant anything short of the precise target coordinates produced a failed result rather than a partially correct one.

### Approach 2: Independent Pure Python Port

As a second, fully independent confirmation, I wrote a pure Python reimplementation of the VM (`prince_solve.py`), reading the data table directly out of the binary file rather than relying on any runtime execution of the original code at all. Running it against the real target coordinates produced an identical result:

```
$ python3 prince_solve.py ./prince_walk
Developer:
"No you didn't."

MISSION COMPLETE

CSSCTF{P12INC3_0R_P1NC3?}
```

Having two completely separate implementations, one driven by actually executing the binary's own machine code and one a from-scratch Python port, agree exactly on both the decoded message and the flag was strong confirmation that the VM logic had been understood correctly rather than just happening to produce a plausible looking string.

## Final Flag

```
CSSCTF{P12INC3_0R_P1NC3?}
```

## Key Takeaways

**The "intended" solve was a runtime memory edit, but a static solve was entirely possible.** Since the flag generation depended only on two input integers and data already present in the binary, there was no actual need to run the game, attach a debugger, or fight the binary's own anti-debugging checks at all.

**A suspiciously large `.rodata` section is worth investigating on its own.** 69 KB of read-only data for a text based game was a strong signal that something beyond UI strings was embedded there, which turned out to be the VM's instruction table.

**Self-authenticating output, via an embedded checksum like FNV-1a, is a strong design choice for this kind of challenge.** It meant that a correct decode could be trusted immediately, without needing to separately judge whether a given output string looked plausible as a flag.

**Testing a near-miss input is a cheap but valuable sanity check.** Confirming that `(999999, 999998)` failed cleanly, rather than producing a partially correct result, validated the understanding that this was a true all-or-nothing avalanche construction rather than something that could be approximated.

**Independent reimplementation is a strong way to validate a reverse engineering result.** Getting matching output from both directly executing the original binary code and a separate from-scratch Python port left very little room for the result being a coincidence or a misunderstanding of the VM's actual logic.
