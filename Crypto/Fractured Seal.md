### Challenge Description
```
Fractured Seal
One of the Registry's oldest key-scrolls survived the fall of Crownspire, though time and fire spared only fragments of its writing, and most in the vault dismissed it as useless. Caldrin didn't. She always said a seal doesn't have to be whole to still remember the door it once opened.
```

In short: Damaged RSA private key PEM → extract almost-complete modulus `n` and high bits of prime `p` → Coppersmith attack to recover the missing low bits of `p` → factor `n` → decrypt `flag.enc`.

### Tools & Environment
- Python 3 + `pycryptodome`
- SageMath (for `small_roots`)
- Optional: `openssl asn1parse` / `asn1tools` for inspecting the broken PEM
- Starting point: only three files — `encrypt.py`, `flag.enc`, `fractured_seal.pem`

### 1. Environment Setup & First Round of Pain

The challenge gives you a short Python encryption script, a 256-byte ciphertext, and a heavily corrupted RSA private key in PEM format.

The PEM is littered with asterisks and a Japanese kaomoji, so the first instinct is usually “maybe I can just replace the `*` with zeros / A’s and import the key”. That fails immediately.

**Stupid mistakes / pitfalls #1:**
- Treating the `*` as literal characters that can be filled in arbitrarily
- Spending time trying to repair the PEM with `openssl rsa -in fractured_seal.pem -check`
- Forgetting that the interesting part is the *structure* of the remaining base64, not the missing characters themselves

### 2. Understanding the Encryption

`encrypt.py` is completely standard:

```python
p = getPrime(1024)
q = getPrime(1024)
n = p * q
e = 0x10001
d = pow(e, -1, (p-1)*(q-1))

m = bytes_to_long(open('flag.txt', 'rb').read())
open('seal.pem', 'wb').write(RSA.construct((n, e, d)).export_key())
open('flag.enc', 'wb').write(long_to_bytes(pow(m, e, n)))
```

2048-bit RSA, no padding, classic textbook encryption. The only non-standard part is that we are given a *fractured* version of the private key instead of the intact `seal.pem`.

### 3. Recovering Partial `n`

The beginning of the PEM is intact:

```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAjK59ahXlX7a+oF+jt5icukpGeNXXgQO4D3jPeLaGAupJm6a6
9PnWCf0W3QDhmCPxLYWSr1C4DvxP23UlvP8Lcfu+w/oIS0jDHkfnv+m1Qku/ii9w
...
7HFFfXYNoKjnqgp+LOmLwQGQeO2Yo1Hb8SCVd***************************
```

Decoding the known base64 prefix yields the DER header and almost the entire modulus. The INTEGER encoding of `n` is 257 bytes long (leading `00` + 256-byte value). We recover the high 255 bytes of `n`; only the final byte is missing (256 candidates).

### 4. Recovering High Bits of `p`

Further down in the PEM a second intact fragment appears:

```
****************************************************2msCgYEAwLGx
cJ7/YCgq9GPDS16cHPNZEmYrbSX+atzUBBO2jLYg0QbXfitTIHfU+55DqIxFQOcu
+CahrPQQROoZAAIPg0LdaGd+3R3/ri**********************************
```

Base64-decoding this fragment reveals the classic ASN.1 marker for a 1024-bit prime:

```
02 81 81 00 c0 b1 b1 70 9e ff 60 28 2a f4 ...
```

We obtain the high 73 bytes (584 bits) of one of the primes.  
584 > 512 (= \(n^{1/4}\)), which is exactly the threshold needed for a practical Coppersmith attack.

### 5. The Attack – Factoring with Known High Bits

The classic result (Coppersmith 1996 / Howgrave-Graham) states that if we know more than a quarter of the bits of a factor of `n`, we can recover the remaining bits in polynomial time.

Concrete steps:

1. For every possible last byte of `n` (0…255) construct a candidate modulus.
2. Form the monic polynomial  

   $f(x) = p_{\text{high}} \cdot 2^{440} + x$
   
3. Call Sage’s `small_roots` with  
   - `X = 2^{440}`  
   - `beta = 0.5`  
   - a modest `epsilon` (0.03 works well)
4. Check whether the recovered root produces a factor of `n`.

Once `p` is known, `q = n // p` and the rest is ordinary RSA decryption.

### 6. Implementation (Sage)

```sage
from Crypto.Util.number import bytes_to_long, long_to_bytes, inverse

n_high255 = bytes.fromhex("8cae7d6a15e55fb6…dbf12095")   # 255 bytes
p_high    = bytes_to_long(bytes.fromhex("c0b1b1709eff60…d1dffae"))  # 73 bytes

k = 440
X = 2**k
e = 0x10001

for last in range(256):
    n = (bytes_to_long(n_high255) << 8) | last
    if n.bit_length() != 2048:
        continue

    R.<x> = Zmod(n)[]
    f = p_high * (1 << k) + x
    roots = f.small_roots(X=X, beta=0.5, epsilon=0.03)

    if roots:
        p = p_high * (1 << k) + int(roots[0])
        if 1 < p < n and n % p == 0:
            q = n // p
            break

phi = (p-1)*(q-1)
d   = inverse(e, phi)
c   = bytes_to_long(open("flag.enc","rb").read())
print(long_to_bytes(pow(c, d, n)))
```

### 7. All the Stupid Things We Hit

| Problem | What happened | Fix |
|---------|---------------|-----|
| Trying to “repair” the PEM with random characters | Invalid key / import errors | Extract only the known base64 fragments |
| Looking for the flag inside the kaomoji or the `*` runs | Nothing useful | The art is just flavour |
| Using `TrueSeal`-style thinking (direct comparison) | N/A | This is a pure factoring problem |
| Running Sage directly on a Windows-mounted path (`/mnt/c/…`) | `PermissionError` on temporary files | Copy files into the WSL home directory first |
| Missing `pycryptodome` inside the Sage conda environment | `ModuleNotFoundError: Crypto` | `pip install pycryptodome` after `conda activate sage` |
| Incomplete copy of the high bits of `p` | Coppersmith fails | Make sure the full 73-byte fragment is used |

### 8. Lessons Learned

1. A damaged RSA private-key PEM still leaks structured ASN.1 data; the remaining fragments are often enough for a Coppersmith attack.
2. Knowing more than \(n^{1/4}\) high bits of a factor is the classic threshold that turns an otherwise hard factoring problem into a practical lattice attack.
3. The narrative (“a seal doesn’t have to be whole”) maps directly onto the cryptographic reality that a partial key can still open the door.
4. Tooling friction (WSL paths, conda environments, missing Python packages) frequently costs more time than the actual crypto.
5. Always extract the *longest contiguous* known byte sequences from a corrupted key; even a few dozen extra bits can make the difference between a working and a non-working Coppersmith instance.

### One-sentence path

Broken PEM → recover almost-complete `n` + 584 high bits of `p` → Coppersmith → factor → decrypt `flag.enc`.