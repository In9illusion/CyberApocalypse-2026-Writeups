### Challenge Description
```
The Ashen Field
Deep beneath the Ash-Vault rests a relic the Cinderbound once swore could preserve truth without ever revealing it, a public rite built from equations, guarded only by the silent calculations its keepers left behind before they vanished. Unravel those old protections and the words sealed inside may finally be read, along with whatever authority the realm hoped they'd bury for good.
```

In short: Multivariate public-key encryption over $(\mathbb{F}_2)$ whose public polynomials turned out to contain **only univariate** $(x_i^4)$ and $(x_i^2)$ terms → collapse to a linear system over $(\mathrm{GF}(2))$ → solve for the message bits → recover the AES key → decrypt `flag.enc`.

### Tools & Environment
- Python 3 + `pycryptodome` + `numpy` (or pure-Python Gaussian elimination)
- Optional: SageMath (for nicer polynomial handling, but not required)
- Starting point: `source.sage`, `output.txt` (huge public-key polynomials + ciphertext + AES ciphertext)

### 1. Environment Setup & First Round of Pain

We are given a Sage script that builds a fancy-looking multivariate public key over ($mathrm{GF}(2)$) with ($N=137$) variables, encrypts a random 137-bit integer `KEY`, then AES-ECB-encrypts the flag under $(mathrm{SHA\text{-}256}(mathrm{str}(mathrm{KEY})))$.

The output file is a single enormous line of polynomials that look like:

$(x1^4 + x2^4 + x4^4 + \ldots + x133^2 + x137^2 + 1, \ldots )$

followed by a 137-bit ciphertext vector and a hex AES blob.

**Stupid mistakes / pitfalls #1:**
- Trying to import the polynomials into Sage as a full multivariate system and run Gröbner bases (instant memory death)
- Assuming the system is genuinely quadratic/cubic and reaching for XL / hybrid attacks
- Spending time re-implementing the exact ring quotient \(R/(x_i^2-x_i)\) before noticing the actual monomials present in the public key
- Forgetting that evaluation is performed on bits, so higher powers collapse

### 2. Understanding the Encryption

`source.sage` does roughly the following:

```python
# simplified
R = PolynomialRing(GF(2), N, 'x')
# \ldots build an affine transformation S, embed into a univariate polynomial over an extension,
# raise to F^{2q}+F^q+1, transform again by T \ldots
PK = vector of N polynomials in the N variables
encrypted_key = PK(msg_bits)
```

The construction is clearly inspired by **Hidden Field Equations (HFE)**-style schemes (the flag even thanks “f4l4y” for the attack on HFE).  
In a real HFE the public key would be dense quadratic (or higher) forms.  
Here the concrete public key that was printed contains **no cross terms whatsoever**.

### 3. The Crucial Observation

Every polynomial in the public key is of the pure form

$\sum_{i\in S} x_i^4 + \sum_{j\in T} x_j^2 + b,\qquad b\in\{0,1\}.$

Over \(\mathbb{F}_2\) the ring of Boolean functions is a **Boolean ring**: every element satisfies the identity
$x^2 = x$

(see [Boolean ring – Wikipedia](https://en.wikipedia.org/wiki/Boolean_ring)).  
Consequently for any bit \(a\in\{0,1\}\)
$a^4 = (a^2)^2 = a^2 = a$

Thus each public polynomial collapses to a **linear** form:

$PK_i(\mathbf{x}) = \sum_j c_{ij}\,x_j + b_i \in \mathbb{F}_2.$


Encryption is therefore nothing more than the matrix-vector product

$\mathbf{c} = M\mathbf{x} + \mathbf{b} \pmod{2}.$

### 4. Recovering the Linear System

Parse the gigantic string with a few regular expressions:

```python
import re
def parse_poly(p):
    coeffs = [0]*137
    for idx in re.findall(r'x(\d+)\^4', p):
        coeffs[int(idx)-1] ^= 1
    for idx in re.findall(r'x(\d+)\^2', p):
        coeffs[int(idx)-1] ^= 1
    const = 1 if p.strip().endswith('+ 1') else 0
    return coeffs, const
```

This yields a \(137\times 137\) matrix \(M\) over \(\mathrm{GF}(2)\) and a constant vector \(\mathbf{b}\).  
The right-hand side is simply \(\mathbf{rhs} = \mathbf{c}\oplus\mathbf{b}\).

### 5. Solving the Linear System

Gaussian elimination (full row reduction) shows

$\operatorname{rank}(M) = 135 \implies \text{nullity}=2.$

There are therefore exactly four candidate solutions.  
A particular solution can be obtained by setting the two free variables to zero; the remaining three solutions are obtained by adding the null-space basis vectors.

### 6. Reconstructing the Integer KEY & Decrypting

Sage’s `Integer.bits()` returns bits **least-significant first**, and the encryption code pads / concatenates in exactly that order.  
Hence

$\mathrm{KEY} = \sum_{i=0}^{136} x_i\cdot 2^i.$

Only candidates with `KEY.bit_length() == 137` are kept.  
For each surviving candidate:

```python
AES_KEY = hashlib.sha256(str(KEY).encode()).digest()
pt = unpad(AES.new(AES_KEY, AES.MODE_ECB).decrypt(enc_flag), 16)
```

Exactly one of them decrypts cleanly and yields the flag.

### 7. Implementation Sketch (pure Python)

```python
# after building A (matrix) and rhs
x0, null_basis = gf2_solve_with_nullspace(A, rhs)   # rank-135 particular + 2 basis vectors

from itertools import product
for coeffs in product([0,1], repeat=2):
    x = x0.copy()
    for c,v in zip(coeffs, null_basis):
        if c: x = (x + v) % 2
    KEY = sum(int(x[i])<<i for i in range(137))
    if KEY.bit_length() != 137: continue
    # AES decrypt as above \ldots
```

### 8. All the Stupid Things We Hit

| Problem | What happened | Fix |
|---------|---------------|-----|
| Treating the system as a genuine MQ instance | Gröbner / XL attempts exploded | Inspect the actual monomials first |
| Forgetting \(x^4=x\) on bits | Kept high-degree terms unnecessarily | Reduce modulo \(x_i^2-x_i\) (or just evaluate) |
| Wrong bit order when reconstructing KEY | AES key never matched | Match Sage’s little-endian `.bits()` |
| Assuming the matrix was full rank | Missed the 4-solution affine space | Always compute nullity |
| Running heavy lattice / polynomial code before parsing | Wasted hours | Look at the printed public key with your eyes |

### 9. Lessons Learned

1. Even a construction that *looks* like HFE (or any multivariate scheme) can collapse to linear algebra if the public polynomials happen to be univariate.
2. Over \(\mathbb{F}_2\) the identity \(x^2=x\) is not a curiosity—it turns every power into a linear term when the input is a bit.
3. Always extract the *longest contiguous known structure* (here the complete list of monomials) before launching heavy machinery.
4. A rank deficiency of only two still leaves a trivial enumeration; never assume “the system is under-determined so it must be hard”.
5. The narrative flavour (“a seal doesn’t have to be whole”, “equations guarded only by silent calculations”) mapped directly onto the cryptographic reality that a seemingly high-degree public key was in fact linear.

### One-sentence path

Broken-looking multivariate PK → notice only $(x_i^4) & (x_i^2)$ terms → reduce to linear system over $(mathrm{GF}(2))$ → solve (4 candidates) → recover KEY → AES decrypt.

**Flag:** `HTB{e1th3r_gr0bn3r_0r_v4r13ty___1t_st1ll_w0rks!th4nks_f4l4y_f0r_y0ur_4tt4ck_0n_HFE}`