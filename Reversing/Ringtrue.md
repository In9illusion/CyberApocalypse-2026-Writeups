### Challenge Description
```
Ringtrue
House Eastreach found one of the broken Signet's shards. Instead of melting it down, they put it to work. The shard still holds a bit of the dragon Astrael's power, so Eastreach built a small device around it and taught it to listen for one voice. Play the right eight tones and the shard rings true, the vow sealed inside opens, and it can carry a king's command again. Play anything else and nothing happens. Sera can't guess the tones. There are eight of them, each can be any value, and far too many to try one by one. But Eastreach was careless and left the device open to read, with everything it was taught still written plainly inside. That carelessness is the whole opening she needs. Find the one voice it was built to obey, and the shard answers to her instead of Eastreach.
```

In short: Linux ELF binary → reverse the 8-8-8-8 int8 MLP in Ghidra + GDB → extract weights/biases and target `ECHO_S` → invert the network (Z3) to recover the unique 8-tone preimage → input the tones → flag.

### Tools & Environment
- Linux (Ubuntu) + Ghidra (auto-analysis + decompiler)
- GDB for live inspection of globals and symbols
- Python 3 (z3-solver for exact integer solving; optional numpy for verification)
- Starting point: only the `ringtrue` ELF binary

### 1. Environment Setup & First Round of Pain

The challenge hands you a single stripped ELF. No source, no README, just the binary and a long narrative about tones and a dragon shard.

First instinct was to throw it into Ghidra, hit Auto-Analyze, and start hunting strings. The Defined Strings window returned almost nothing useful for “tone”, “ring”, “true”, “shard” or “voice”. The only interesting format string was:

```
%d %d %d %d %d %d %d %d
```

**Stupid mistakes / pitfalls #1:**
- Expecting a classic “compare against a hard-coded string/array of 8 numbers” and spending too long staring at `.rodata`
- Filtering Defined Strings for story keywords and concluding the binary had no useful strings
- Forgetting that modern CTF “plainly written inside” often means a whole neural-network weight matrix, not a simple constant array

### 2. Locating the Real Logic

After the format-string XREF led to `main`, the decompiler revealed the real structure:

- A long boot-sequence of fake kernel messages that mention “npu: resonance-core”, “MLP 8-8-8-8, int8, leaky”, “weights symmetric int8”
- An 8-integer `sscanf`
- Three successive calls to a function named `dense`
- A comparison of the final 8-long output against a global called `ECHO_S`
- On exact match → SHA-style key derivation from the original tones and decryption of a sealed vow (the flag)

The network is therefore the entire “device” the story talks about.

### 3. Decompiling the Core

Open the binary in Ghidra, locate `main` and the helper `dense`.

```c
// simplified from decompiler
void dense(long W, long B, long in, long out)
{
    for (int i = 0; i < 8; i++) {
        long sum = *(int *)(B + i*4);
        for (int j = 0; j < 8; j++)
            sum += (char)*(W + j) * *(long *)(in + j*8);
        *(long *)(out + i*8) = sum;
        W += 8;                     // next row
    }
}
```

Activation after the first two layers:

```c
if (val < 0) val *= 2;          // the “leaky” mentioned in the boot log
```

Final check:

```c
bool aligned = true;
for (int i = 0; i < 8; i++)
    if (local_378[i] != ECHO_S[i])
        aligned = false;
```

### 4. Extracting the Model

All parameters live as globals. In GDB:

```
x/64xb &L0_W
x/32xb &L0_B
x/64xb &L1_W
x/32xb &L1_B
x/64xb &L2_W
x/32xb &L2_B
x/8dg  &ECHO_S
```

Yields (after parsing little-endian int8 matrices and int32 biases):

```
L0_W (8×8 int8), L0_B (8× int32)
L1_W (8×8 int8), L1_B (8× int32)
L2_W (8×8 int8), L2_B (8× int32)
ECHO_S = [1542223, 574187, -2694563, -3518303,
          383776, 576877, 2637871, -2518822]
```

### 5. What the Device Actually Demands

- The shard “rings true” only when the three-layer network maps the eight input tones exactly onto `ECHO_S`.
- The sealed vow is then decrypted with a keystream derived from those same tones.
- In story terms: you must play the single voice the device was taught, the one the world itself cannot distinguish from genuine.

So the attack reduces to:

1. Model the exact forward pass (dense + leaky activation).
2. Solve for the unique 8-tuple of integers that produces `ECHO_S`.
3. Feed that tuple to the running binary.

### 6. Inverting the Network

Because every operation is integer and the activation is piecewise-linear, the problem is a pure integer constraint system. Z3 solves it in milliseconds:

```python
from z3 import *

# \ldots load the six matrices/vectors exactly as extracted \ldots

x = [Int(f'x{i}') for i in range(8)]
s = Solver()
for xi in x:
    s.add(xi >= -200, xi <= 200)          # generous but finite range

def dense(W, B, inp):
    return [B[i] + sum(W[i][j]*inp[j] for j in range(8)) for i in range(8)]

def act(v):
    return [If(v[i] < 0, v[i]*2, v[i]) for i in range(8)]

h = act(dense(L0_W, L0_B, x))
h = act(dense(L1_W, L1_B, h))
out = dense(L2_W, L2_B, h)

for i in range(8):
    s.add(out[i] == ECHO_S[i])

assert s.check() == sat
tones = [s.model()[xi].as_long() for xi in x]
# → [83, 97, 108, 116, 67, 114, 119, 110]
```

Verification: feeding the recovered tones through a pure-Python forward pass reproduces `ECHO_S` exactly.

(The numbers are also the ASCII bytes of `SaltCrwn` – a nice thematic nod to the CTF title.)

### 7. Obtaining the Flag

```bash
./ringtrue
# \ldots boot messages \ldots
  attune> 83 97 108 116 67 114 119 110
```

The device prints the victory banner and the decrypted vow (the flag).

### 8. Full Attack Chain Recap

1. Open ELF in Ghidra → notice Godot-style boot log mentioning “MLP 8-8-8-8”
2. Follow the `%d %d %d %d %d %d %d %d` XREF into `main`
3. Decompile `dense` and the activation
4. Dump `L0_W/B`, `L1_W/B`, `L2_W/B`, `ECHO_S` with GDB
5. Encode the exact network in Z3 and solve for the preimage
6. Input the eight recovered integers
7. Flag

### 9. All the Stupid Things We Hit

| Problem | What happened | Fix |
|---------|---------------|-----|
| Empty / irrelevant Defined Strings | Thought the secret was a simple constant array | The secret is an entire neural net |
| Only partial bias dumps (`x/8xb`) | Incomplete model, wrong answers | Always dump the full 32-byte bias arrays |
| Wrong matrix layout assumption | Random search never hit | Decompile `dense` – it is clearly row-major `W @ x + B` |
| Trying continuous optimisation first | Landed near but never exact | Switch to exact integer solver (Z3) |
| Forgetting the activation multiplies negatives by 2 | Model mismatch | Copy the decompiled `if (val < 0) val *= 2` literally |
| Looking for the flag inside the binary | Nothing useful | The flag is decrypted only at runtime after a correct input |

### 10. Lessons Learned

1. **A “device that listens for one voice” is often a neural network** whose weights are left in plain sight.
2. When the comparison target is the *output* of a mixer / network, the challenge is almost always to invert that function.
3. The narrative (“play the right eight tones”) maps one-to-one onto finding a preimage under the MLP.
4. Integer networks with simple piecewise-linear activations are exact constraint problems; Z3 is usually faster and more reliable than random search or gradient methods.
5. Always extract the complete static arrays (weights *and* biases); partial copies produce silently wrong models.
6. Beginner-friendly tools (Ghidra decompiler + a few dozen lines of Python/Z3) are more than enough once the right globals are identified.

### One-sentence path

ELF → Ghidra `main` + `dense` → GDB dump of the three-layer int8 MLP and `ECHO_S` → Z3 invert → eight tones → flag.