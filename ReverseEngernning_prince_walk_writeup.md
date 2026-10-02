# PRINCE WALK: Full Write-up (beginner-friendly)

**Event:** CSSCTF 2026 | **Category:** Reverse Engineering | **Flag format:** `CSSCTF{...}`

> ## Flag
> **`CSSCTF{P12INC3_0R_P1NC3?}`**

---

## 1. TL;DR (the 30-second version)

* We get a terminal game. Our character starts at **(1, 1)** and must reach **(999999, 999999)**. Walking there one step at a time is impossible.
* The game never stores the flag. When you stand on the target square, it **calculates** the flag from your coordinates using a small hidden "mini-computer" (a *virtual machine*, VM).
* That calculation depends only on the two numbers `x` and `y`. So we don't need to play the game. We feed `999999, 999999` into the game's own calculation and read the answer.
* Result: `CSSCTF{P12INC3_0R_P1NC3?}`

---

## 2. Words you'll see (mini glossary)

| Term | Plain meaning |
|---|---|
| **Binary / ELF** | A compiled program. "ELF" is the Linux executable format. |
| **Reverse engineering (RE)** | Working out what a program does without having its source code. |
| **Disassembler** (`objdump`) | A tool that turns the machine code back into readable CPU instructions (assembly). |
| **`strings`** | A tool that lists all readable text hiding inside a binary. |
| **Memory editor (PINCE)** | A Linux tool to find and change numbers inside a running program, like cheating in a game. |
| **Int32** | A whole number stored in 4 bytes. The game keeps your X and Y as Int32s. |
| **Virtual machine (VM)** | A tiny fake CPU written inside a program. It reads a list of instructions ("bytecode") and runs them. |
| **Hash / checksum** | A scrambler that turns data into a fixed-size number. Used here to scramble values and to check the result is correct. |
| **Stripped binary** | A binary with its function names removed, so we only see addresses like `0x2c56`. |

---

## 3. What we were given

* The story: a survey simulation called PRINCE WALK. The avatar starts at `(1, 1)`; the "recovery beacon" is at `(999999, 999999)`.
* One file: `prince_walk`.
* Goal: get the flag in the form `CSSCTF{...}`.

```
$ file prince_walk
prince_walk: ELF 64-bit LSB pie executable, x86-64, dynamically linked, stripped
```

So it's a normal Linux program with function names removed.

---

## 4. The obvious (but wrong) approach: just walk

The map is about a million tiles wide. At one step per key press, you would need nearly two million moves. Nobody is meant to do that, so there must be a trick.

---

## 5. Step-by-step solution

### Step 1: Read the clues hidden in the text

Running `strings prince_walk` shows the program's visible text. One line stands out:

> *"A terminal sandbox for **PINCE** Int32 memory-editing practice."*

That tells us the author's *intended* trick: the player's X and Y are plain 4-byte numbers in memory. With a memory editor like PINCE you can search for them, overwrite both with `999999`, take one step, and arrive instantly.

We didn't need to run the game at all. But the clue also tells us something important: **the position numbers are what matter**.

Also notable: there is **no `CSSCTF{...}` anywhere in the strings**. The flag isn't stored in the file in readable form, so it must be **built at runtime**.

### Step 2: Notice the suspiciously large data table

Looking at the file's sections (`readelf -S prince_walk`), the read-only data (`.rodata`) is about **69 KB**. The game's visible text is only a few KB. The rest, a block starting around address `0x4a80`, is **66,560 bytes of unreadable data**.

A big blob of random-looking data in a CTF binary is a strong hint: it's either the encrypted flag or a "program" for something to run. Here it turns out to be a program.

### Step 3: Find the "you've arrived" check

Using a disassembler (`objdump -d -M intel prince_walk`) we look for where the number **999999** appears. In hex, 999999 is `0xF423F`. Searching the assembly for `0xf423f` leads to one function at address **`0x2c56`**. Its first lines do this:

```
if (x != 999999) -> fail
if (y != 999999) -> fail
otherwise        -> run the big calculation
```

So that function is the "destination reached" handler. We can think of it as:

```
decode(x, y, output_buffer)  ->  success or fail
```

If you're on the right square it fills `output_buffer` with a message (the flag); if not, it fails.

### Step 4: Understand what the calculation does (the VM)

This is the heart of the challenge. Think of it like a **combination safe**:

* The **combination** is your coordinates `(x, y)`.
* Inside is a **long script of instructions** (the 66 KB table).
* The coordinates are mixed into the machine's starting state. If the combination is even slightly wrong, every later step scrambles differently and you get garbage.

Here is what the function does, in plain steps:

1. **Set up 16 "registers"** (16 little number slots) and one "accumulator" by scrambling `x` and `y` with a hash function. Same coordinates → same starting values, every time.
2. **Run 8,320 steps.** For each step number `pc`:
   * Look up an 8-byte entry in the table.
   * Combine it with a hash of the accumulator and `pc` to get two numbers, `r` and `v`.
   * The lowest byte of `r` is the **instruction type** (7 kinds of arithmetic/rotate/hash operations, plus one special "write a byte" instruction). Other bits of `r` pick which registers to use.
   * Do the operation and update the accumulator. Since the accumulator feeds into the next step, **one wrong number early on ruins everything after it**.
3. **The special instruction (opcode `0x83`)** writes a single byte into a 128-byte buffer at a position that is also computed. Over the whole run, all 128 positions get filled exactly once.
4. **Check the answer.** The buffer is laid out as:

   ```
   [ length ][ message bytes ... ][ 4-byte checksum ]
   ```

   The function recomputes an **FNV-1a checksum** of the message and compares it to the stored one. It also requires every character to be printable text. Only if everything matches does it copy the message out and return success.

**Why this matters for us:** the checksum means the function is *self-verifying*. If it returns success, the message is genuinely correct. There is no chance of getting a wrong-but-plausible flag.

### Step 5: Run the calculation ourselves

Since the function only needs `x`, `y` and the static table, we don't have to play the game. Two independent ways:

**Method A: call the game's own machine code.**
We load the binary into memory, point at the function at `0x2c56`, and call it with `(999999, 999999)`. (One internal `memcpy` call is redirected to the normal library version, because we aren't running the full program.)

```
(1, 1)             -> fails (no message)
(999999, 999998)   -> fails (off by one is enough!)
(999999, 999999)   -> Developer:
                      "No you didn't."

                      MISSION COMPLETE

                      CSSCTF{P12INC3_0R_P1NC3?}
```

**Method B: re-implement it in Python.**
We translate the VM into ~40 lines of Python (`prince_solve.py`), read the table directly from the file, and run it. We get exactly the same text and the checksum passes.

Getting the same answer two different ways, plus the built-in checksum, is strong proof that we have the right flag.

---

## 6. The result

```
Developer:
"No you didn't."

MISSION COMPLETE

CSSCTF{P12INC3_0R_P1NC3?}
```

The flag in leetspeak reads **"PRINCE or PINCE?"**. It's a joke on the game's name versus the PINCE memory editor that the author expected players to use. The line `"No you didn't."` is the "developer" teasing you for cheating your way to the end.

**Flag: `CSSCTF{P12INC3_0R_P1NC3?}`**

---

## 7. Two roads to the same answer

| | Intended route | Our route |
|---|---|---|
| Idea | Edit X and Y in memory with PINCE, then step onto the target | Call the "arrival" function directly with the target coordinates |
| Needs | Running the game + a memory editor | Just the file; no game running |
| Why it works | The game trusts the coordinates stored in memory | The flag depends *only* on the coordinates |

Both work for the same reason: **the coordinates are the key**.

---

## 8. Reproduce it yourself

```bash
file prince_walk                      # confirm it's an x86-64 ELF
strings prince_walk | less            # spot the PINCE hint, no flag in plain text
readelf -S prince_walk                # note the oversized .rodata (~69 KB)
objdump -d -M intel prince_walk > dis.txt
grep -n "f423f" dis.txt               # finds the 999999 check -> function 0x2c56
python3 prince_solve.py ./prince_walk # prints the flag
```

Bonus: `python3 prince_solve.py ./prince_walk 1 1` prints "no message", which proves wrong coordinates give nothing.

---

## 9. Lessons learned

1. **If the flag isn't in `strings`, it's computed.** Look for what feeds the computation.
2. **A big unexplained data blob + a hard-coded constant** (`0xF423F` = 999999) is a classic sign of "the reward is derived from the data and the inputs".
3. **Pure functions can be called directly.** If a check depends only on its inputs and static data, you don't need to run the whole program or beat the game. Feed it the right inputs.
4. **Built-in checksums are your friend.** They tell you immediately whether your solve is right.
5. **Hints in flavor text matter.** "PINCE Int32 editing" told us exactly which values (X and Y) were important.

---

## Appendix: technical details for the curious

**Function map (stripped binary, addresses are file virtual addresses)**

| Address | Role |
|---|---|
| `0x2c18` | Integer hash `H(v)` ("lowbias32"): `v ^= v>>16; v *= 0x7FEB352D; v ^= v>>15; v *= 0x846CA68B; v ^= v>>16` |
| `0x2c56` | `decode(x, y, out, outlen)`: the VM and final validation |
| `0x4a80` | Bytecode table: 0x2080 entries × 8 bytes (two u32 per entry) |

**VM initialisation**

```
st[i] = H( ((i+1) * 0x9E3779B9) ^ (x ^ rotl(y, i+1)) )      for i = 0..15
acc   = H( (rotl(y, 19) + x) ^ 0xA4093822 )
```

**Each step (`pc` = 0 .. 0x207F)**

```
idx = (pc*0xD9 + 0xCD6) mod 0x2080
(w0, w1) = table[idx]
r = H((pc*0x9E3779B9) ^ acc) ^ w0
v = H(pc + acc + 0x6A09E667) ^ w1
op = r & 0xFF; d = (r>>8)&15; a = (r>>12)&15; b = (r>>16)&15; s = (r>>20)&31
A = st[a]; B = st[b]
```

| Opcode | Operation |
|---|---|
| `0x8B` | `st[d] = rotl(B,s) + A + v` |
| `0x48` | `st[d] = rotl(B+v, s) ^ A` |
| `0x31` | `st[d] = (v\|1) * ((B+v) ^ A)` |
| `0x71` | `st[d] = A - rotl(B^v, s)` |
| `0xB3` | `st[d] = (A & v) \| (~v & B)` |
| `0x12` | `st[d] = H(A+B+v)` |
| `0xFF` | `st[d] = rotl(A^v, (B^s)&31)` |
| `0x83` | `pos = v>>25; buf[pos] = (v ^ A ^ rotl(B,s)) & 0xFF` (each pos once); `st[d] ^= (pos+byte)*0x45D9F3B` |
| other | abort (fail) |

After the operation: `acc = rotl(st[d] ^ st[a] ^ acc ^ r, 9) + v + pc*0x3C6EF372`

**Final validation:** exactly 128 bytes written; `buf[0]` = length `L` (1..0x7B); message = `buf[1..L]`, every byte printable ASCII or newline; `FNV-1a(message)` (offset basis `0x811C9DC5`, prime `0x1000193`) must equal the little-endian u32 at `buf[L+1..L+4]`.

All of this is implemented in `prince_solve.py`, which reads the table straight from the binary and prints the flag.
