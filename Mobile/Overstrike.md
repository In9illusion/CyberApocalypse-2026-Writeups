
### Challenge Description
```
Overstrike
The Signet is shattered, and its silence has a shape now: House Vaultrune's quiet trade in re-cut marks, genuine authority lifted and struck anew, each forgery slipped into the Registry so the lie reads as record. Elric goes down under Crownspire, toward the sealed archive Chancellor Veylen Marr keeps, and finds the old vow-stones of the Ash-Vault will not bridge the fire-rift for any mark he carries. They answer only to the true seal. To reach the truth the forgers have filed as fact, Elric must do the very thing that condemns them, and cross on a seal of his own making, one the world itself cannot tell from genuine.
```

In short: Android APK (Godot Engine + C#) → extract & decompile `Overstrike.dll` → recover the secret `TrueSeal` → invert the `Mix` function to forge a valid `CarriedMark` → decrypt `SealedRecord` with SHA-256 keystream XOR → flag.

### Tools & Environment
- Windows + JADX (initial static analysis)
- 7-Zip / unzip for APK extraction
- dnSpy (or ILSpy) for .NET decompilation
- Python 3 (hashlib + struct + modular inverse) for the final crypto steps
- Optional: mono-devel / monodis on Linux if dnSpy is unavailable
- Starting point: only the `Overstrike.apk` file

### 1. Environment Setup & First Round of Pain

The challenge hands you a single APK. No source, no README, just the binary.

First instinct on a complete beginner setup was to open it in JADX and stare at the Java/Kotlin side. The package tree is full of `androidx`, `kotlin`, `kotlinx.coroutines`, `com.godot.game` and `org.godotengine.godot`.  

That last pair is the real giveaway: this is a **Godot** game, not a classic Android app. The interesting logic is not in `classes.dex`.

**Stupid mistakes / pitfalls #1:**
- Spending time reading empty stub `.cs` files under `assets/scripts/` (they are just placeholders)
- Expecting the flag or the seal logic to live in Java
- Forgetting that Godot C# exports ship the real code as compiled DLLs inside `assets/.godot/mono/publish/`

### 2. Locating the Real Code

Once the Godot nature is clear, extract the APK and look under:

```
assets/.godot/mono/publish/arm64/Overstrike.dll
assets/.godot/mono/publish/x86_64/Overstrike.dll
```

(both architectures contain the same game assembly)

Also present (but far less useful):
- `assets.sparsepck` (only 35 KB, almost no strings)
- empty `.cs` stubs
- the usual Godot runtime libraries

`strings` on the DLL already leaked the important names:

```
TrueSeal
WorldSeal
CarriedMark
AshStone
MakeCrownspire
UnsealRegistry
SealedRecord
Mix
TileHash
```

### 3. Decompiling Overstrike.dll

Open the DLL in dnSpy (or ILSpy). The classes of interest are:

- `GameState`
- `Archive`
- `BridgeBuilder`
- `MarkPickup`
- `Main`

The heart of the challenge lives in `GameState`.

### 4. The Core Logic in GameState

```csharp
public const ulong TrueSeal = 15682021040575554950UL;

public ulong CarriedMark;
public ulong WorldSeal;

public override void _Process(double delta)
{
    this.WorldSeal = GameState.Mix(this.CarriedMark);
}

public bool WorldIsAligned
{
    get { return this.WorldSeal == 15682021040575554950UL; }
}

public static ulong Mix(ulong x)
{
    ulong num  = x + 11400714819323198485UL;
    ulong num2 = (num  ^ num  >> 30) * 13787848793156543929UL;
    ulong num3 = (num2 ^ num2 >> 27) * 10723151780598845931UL;
    return num3 ^ num3 >> 31;
}
```

And the decryption routine:

```csharp
public string UnsealRegistry()
{
    // SHA256(CarriedMark) → initial keystream
    // then SHA256(keystream || counter) repeatedly
    // XOR against the static SealedRecord byte array
    // return the resulting printable string
}
```

`SealedRecord` is a fixed 56-byte array baked into the binary.

### 5. What the Game Actually Demands

- The “world” is aligned only when `Mix(CarriedMark) == TrueSeal`.
- The archive / registry only reveals its content when you call `UnsealRegistry()` with a `CarriedMark` that satisfies the above condition.
- In story terms: you must forge a mark that the world itself cannot distinguish from the genuine seal.

So the attack reduces to:

1. Invert the `Mix` function.
2. Obtain the unique preimage `CarriedMark` such that `Mix(CarriedMark) == TrueSeal`.
3. Feed that value into the SHA-256 keystream XOR decryption.

### 6. Inverting Mix

`Mix` is a classic finalizer-style mixer (very similar to SplitMix64). All operations are reversible over 64-bit words:

- The final `x ^ (x >> 31)` is inverted by a short fixed-point iteration.
- Multiplication by the odd constants is inverted with modular multiplicative inverses modulo \(2^{64}\).
- The earlier shifts are inverted the same way.

Python sketch:

```python
def modinv(a, m=2**64):
    return pow(a, -1, m)

def inv_shift_xor(y, shift):
    x = y
    for _ in range(64 // shift + 1):
        x = y ^ (x >> shift)
    return x & 0xFFFFFFFFFFFFFFFF

def unmix(y):
    C1 = 11400714819323198485
    C2 = 13787848793156543929
    C3 = 10723151780598845931
    inv_C2 = modinv(C2)
    inv_C3 = modinv(C3)

    num3 = inv_shift_xor(y, 31)
    t    = (num3 * inv_C3) & 0xFFFFFFFFFFFFFFFF
    num2 = inv_shift_xor(t, 27)
    t2   = (num2 * inv_C2) & 0xFFFFFFFFFFFFFFFF
    num  = inv_shift_xor(t2, 30)
    x    = (num - C1) & 0xFFFFFFFFFFFFFFFF
    return x

CarriedMark = unmix(15682021040575554950)
# → 15549431037298259574
```

Verification: `Mix(15549431037298259574) == TrueSeal` holds.

### 7. Decrypting the Registry

With the correct `CarriedMark` in hand, implement the exact keystream generation from `UnsealRegistry`:

```python
import hashlib, struct

carried = struct.pack('<Q', 15549431037298259574)
key     = hashlib.sha256(carried).digest()

plaintext = bytearray(len(SealedRecord))
i = num = 0
while i < len(SealedRecord):
    block = key + struct.pack('<I', num)
    stream = hashlib.sha256(block).digest()
    for b in stream:
        if i >= len(SealedRecord):
            break
        plaintext[i] = SealedRecord[i] ^ b
        i += 1
    num += 1

flag = ''.join(chr(b) if 32 <= b < 127 else '.' for b in plaintext)
```

And then get result.
### 8. Full Attack Chain Recap

1. Open APK in JADX → notice Godot + empty `.cs` stubs  
2. Extract APK → locate `Overstrike.dll` under `.godot/mono/publish/`  
3. Decompile with dnSpy → read `GameState`  
4. Identify `TrueSeal`, `Mix`, and `UnsealRegistry`  
5. Invert `Mix` to obtain a forged `CarriedMark`  
6. Run the SHA-256 keystream XOR against the static `SealedRecord`  
7. Obtain the flag  

### 9. All the Stupid Things We Hit

| Problem | What happened | Fix |
|---------|---------------|-----|
| Empty `.cs` files in JADX | Thought the source was missing | Real code is the compiled DLL |
| Treating it as pure Java | Wasted time on androidx/kotlin | Look for Godot / mono folders |
| Trying to decrypt with `TrueSeal` directly | Garbage output | Must invert `Mix` first |
| Wrong endianness on the counter | Keystream misaligned | Confirm little-endian (`<I` / `<Q`) |
| Incomplete `SealedRecord` copy | Truncated plaintext | Copy the full 56-byte array |
| Assuming the flag is in resources | Nothing useful in sparsepck | Logic is entirely in the DLL |

### 10. Lessons Learned

1. **Godot C# Android exports hide the real logic in DLLs**, not in the visible `.cs` stubs or the sparse pck.
2. A “true seal” constant that is compared after a mixing function almost always means you need to invert that mixer.
3. The narrative (“forge a seal the world cannot tell from genuine”) maps one-to-one onto finding a preimage under `Mix`.
4. When a decryption routine takes a player-controlled value as the key, the challenge is usually to compute the *correct* value that satisfies a separate integrity check.
5. Always extract the full static byte arrays; partial copies produce misleading garbage.
6. Beginner-friendly tools (dnSpy + a few lines of Python) are more than enough once the right assembly is identified.

### One-sentence path

APK → extract `Overstrike.dll` → decompile `GameState` → invert `Mix(TrueSeal)` to forge `CarriedMark` → SHA-256 keystream XOR on `SealedRecord` → flag.

A clean mobile reverse + lightweight crypto challenge that rewards recognising a Godot C# export and treating the story’s “forged seal” as a concrete preimage problem.