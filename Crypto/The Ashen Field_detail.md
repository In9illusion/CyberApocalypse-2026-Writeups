> (P.S: This writeup include more detail about cryptosystem, especially why public keys degenerate into linearity)
> 
> All formulas are based on standard KaTeX, and relevant background links have been supplemented :)
### Challenge Description
```
The Ashen Field Deep beneath the Ash-Vault rests a relic the Cinderbound once swore could preserve truth without ever revealing it, a public rite built from equations, guarded only by the silent calculations its keepers left behind before they vanished. Unravel those old protections and the words sealed inside may finally be read, along with whatever authority the realm hoped they'd bury for good.
```
**In short:**  
Multivariate public-key encryption over \(\mathbb{F}_2\) whose public polynomials contain **only univariate** terms \(x_i^4\) and \(x_i^2\).  
Because every element of \(\mathbb{F}_2\) satisfies \(x^2 = x\), these polynomials collapse to linear forms.  
The encryption therefore reduces to a linear system over \(\mathrm{GF}(2)\).  
Solving the system recovers the 137-bit message `KEY`, which is then used as the AES key to decrypt the flag.

### Tools & Environment
- Python 3 + `pycryptodome` + `numpy` (or pure-Python GF(2) Gaussian elimination)
- Optional: SageMath
- Files: `source.sage`, `output.txt`

---

### 1. First Look at the Public Key

The file `output.txt` begins with an enormous tuple of 137 polynomials.  
Every single one of them has the following shape:
```

x1^4 + x2^4 + x4^4 + … + x97^2 + x103^2 + … + 1

text

````
**Critical observation:**  
There are **no cross terms** whatsoever (no \(x_i x_j\), no \(x_i^2 x_j\), etc.).  
Only pure fourth powers, pure squares, and an optional constant.

---

### 2. Why the High Powers Collapse (the Algebraic Heart)

The underlying ring is the Boolean ring \(\mathbb{F}_2[x_1,\dots,x_n]/(x_i^2-x_i)\).  
In any Boolean ring the identity

\[
x^2 = x
\]

holds for every element (see [Boolean ring – Wikipedia](https://en.wikipedia.org/wiki/Boolean_ring)).

Consequently, for any bit \(a\in\{0,1\}\):

\[
\begin{align*}
a^2 &= a,\\
a^4 &= (a^2)^2 = a^2 = a.
\end{align*}
\]

Therefore each public polynomial

\[
\sum_{i\in S} x_i^4 + \sum_{j\in T} x_j^2 + b
\]

evaluates to the **linear** form

\[
\sum_{i\in S\cup T} x_i + b \pmod{2}.
\]

(The coefficient of \(x_k\) is 1 if \(k\) appears an odd number of times among the fourth-power and square terms.)

Encryption is consequently nothing more than a matrix-vector product over \(\mathrm{GF}(2)\):

\[
\mathbf{c} = M\mathbf{x} + \mathbf{b} \pmod{2}.
\]

This is the decisive vulnerability: the designers believed that raising a linear form to the power \(F^{2q}+F^q+1\) (with \(q=2\)) would produce high-degree multivariate polynomials.  
Because the input variables live in a Boolean ring and the published public key happened to contain only univariate monomials, the whole system remained linear.

---

### 3. Extracting the Linear System

We parse each polynomial with regular expressions:

```python
import re

def parse_poly(p: str):
    coeffs = [0] * 137
    for idx in re.findall(r'x(\d+)\^4', p):
        coeffs[int(idx)-1] ^= 1
    for idx in re.findall(r'x(\d+)\^2', p):
        coeffs[int(idx)-1] ^= 1
    const = 1 if p.strip().endswith('+ 1') else 0
    return coeffs, const
````

This produces:

- a (137\times 137) matrix (M) over (\mathrm{GF}(2)),
- a constant vector (\mathbf{b}).

The right-hand side of the linear system is simply

[ \mathbf{rhs} = \mathbf{c} \oplus \mathbf{b}. ]

---

### 4. Solving over (\mathrm{GF}(2))

Full Gaussian elimination yields

[ \operatorname{rank}(M) = 135 \qquad\Rightarrow\qquad \text{nullity} = 2. ]

There are therefore exactly four solutions. We compute one particular solution (free variables set to 0) together with a basis of the null space, then enumerate the four affine combinations.

---

### 5. Reconstructing the Integer KEY

Sage’s Integer.bits() returns bits from least-significant to most-significant. The encryption routine concatenates the bits in exactly the same order, so

[ \mathrm{KEY} = \sum_{i=0}^{136} x_i \cdot 2^i. ]

We discard any candidate whose bit length is not exactly 137 (the highest bit must be 1).

---

### 6. AES Decryption

For each remaining candidate:

Python

```
AES_KEY = hashlib.sha256(str(KEY).encode()).digest()
pt = unpad(AES.new(AES_KEY, AES.MODE_ECB).decrypt(enc_flag), 16)
```

Exactly one candidate produces a correctly padded plaintext — the flag.

---

### 7. Implementation Notes & Pitfalls

|Problem|What happened|Fix|
|---|---|---|
|Treating the system as a genuine MQ instance|Gröbner / XL attempts ran out of memory|Inspect the actual monomials first|
|Forgetting (x^4=x) on bits|Kept unnecessary high-degree terms|Reduce modulo (x_i^2-x_i) (or just evaluate)|
|Wrong endianness when rebuilding KEY|AES key never matched|Match Sage’s little-endian .bits()|
|Assuming full rank|Missed the 4-solution space|Always compute the nullity|
|Launching heavy polynomial machinery before parsing|Hours wasted|Look at the printed public key with your eyes|

---

### 8. Lessons Learned

1. A construction that _looks_ like Hidden Field Equations (HFE) can still collapse to linear algebra when the public polynomials are univariate.
2. The Boolean-ring identity (x^2=x) is not a curiosity — it turns every higher power into a linear term as soon as the inputs are bits.
3. Always extract the longest contiguous known structure (here the complete list of monomials) before launching heavy algebraic attacks.
4. A rank deficiency of only two still leaves a trivial enumeration; never assume “under-determined ⇒ hard”.
5. The narrative flavour of the challenge (“equations guarded only by silent calculations”) maps directly onto the cryptographic reality that a seemingly high-degree public key was in fact linear.

---

### One-sentence path

Public-key polynomials contain only (x_i^4) and (x_i^2) → reduce to linear system over (\mathrm{GF}(2)) → solve (4 candidates) → recover KEY → AES decrypt.

**Flag:** HTB{e1th3r_gr0bn3r_0r_v4r13ty___1t_st1ll_w0rks!th4nks_f4l4y_f0r_y0ur_4tt4ck_0n_HFE}