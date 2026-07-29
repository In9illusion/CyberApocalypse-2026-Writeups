### Challenge Description
```
The Forged Signet
A long time ago the Brine Signet was the one thing nobody could fake; it was how the Crown proved a seal was real. Then it broke, and everything went bad. Every seal is still checked against a secret called the First Mark, a hidden string `s` that lives inside the verifier. The verifier has a flaw it was never meant to have: if two inputs differ by exactly the Mark, it cannot tell them apart, so `f(x)` and `f(x XOR s)` always look the same to it. The Registry hid the verifier behind a quantum oracle and swore the Mark was safe. It is not.
```

In short: restricted quantum oracle → Simon’s algorithm → recover the secret period \(s\) → forge the seal.

Example target: `154.57.164.73:31691` (HTTP API). Instances expire fast, so expect to respawn more than once.

### Tools & Environment
- AWS-style? No. Pure quantum CTF with a custom HTTP frontend.
- Starting point: only a Docker instance (IP + port). No downloadable files.
- Key tools we actually used:
  - `nmap -sV`
  - `curl`
  - Python `requests` + homemade GF(2) linear algebra
- Region / credentials: none. Just the public HTTP endpoint.

### 1. Environment Setup & First Round of Pain

The challenge only gives you a Docker IP:PORT. No files, no README, nothing.

First attempts looked like this:

```bash
nc 154.57.164.73 31691
```

…and it just sat there. Forever.

`timeout 6 nc -v ...` reported “Connection succeeded!” but the server never spoke. Sending `help`, `flag`, `\n`, random junk — all timed out.

**Stupid mistakes / pitfalls #1:**
- Treating it as a classic TCP text service
- Forgetting that many modern HTB quantum challenges hide behind HTTP
- VPN interference (GlobalProtect was silently dropping packets after the TCP handshake)

The breakthrough came from the most basic recon command:

```bash
nmap -sV -Pn -p 31691 154.57.164.73
```

Result:

```
31691/tcp open  http    Gunicorn
```

It was an HTTP service the whole time. Classic.

### 2. Discovering the Real Interface

Once we knew it was HTTP:

```bash
curl -s http://154.57.164.73:31691/
```

A beautiful frontend appeared (“The Forged Signet · Resonance Oracle”) with 64 qubits and the promise \(f(x) = f(x \oplus s)\) written right on the page.

The real gold was the JavaScript:

```bash
curl -s http://154.57.164.73:31691/app.js
```

From `app.js` we extracted the actual API surface:

| Method | Path          | Purpose                                     |
| ------ | ------------- | ------------------------------------------- |
| GET    | `/api/oracle` | Datasheet (n, circuit description, serial)  |
| POST   | `/api/run`    | Run the quantum circuit, return samples     |
| POST   | `/api/forge`  | Submit the recovered \(s\) and get the flag |

`/api/run` body format:

```json
{
  "layer": "H",          // or a list of 64 individual gates
  "shots": 128           // 1–256
}
```

### 3. What the Circuit Actually Is

Calling `/api/oracle` gave us the exact circuit description:

```
each run: H^n . U_f . measure&discard(output) . L . measure(input) ; you choose the single-qubit layer L
```

In other words:

1. Apply \(H^{\otimes n}\) to the input register
2. Query the oracle \(U_f\)
3. Measure & discard the output register
4. Apply the layer \(L\) you chose (can be different gates per qubit)
5. Measure the input register

When you set `layer = "H"` (or a list of 64 `"H"`s), this is **exactly** the standard Simon circuit.

### 4. Simon’s Algorithm – The Math That Matters

This is a pure Simon’s problem.

**Promise**: there exists a secret non-zero string \(s \in \{0,1\}^n\) such that

$f(x) = f(y) \quad \Leftrightarrow \quad x \oplus y \in \{0^n, s\}$

**Standard quantum attack**:

1. Prepare $(\lvert 0\rangle^{\otimes n})$ and apply $(H^{\otimes n})$
2. Query \($U_f$\)
3. Measure the output register → the input register collapses to
   $\frac{1}{\sqrt{2}}\big(\lvert x\rangle + \lvert x \oplus s\rangle\big)$

4. Apply \(H^{\otimes n}\) again
5. Measure → obtain a string \(y\) satisfying
   $y \cdot s \equiv 0 \pmod{2}$


Repeat until you have $(n-1)$ linearly independent equations over \($\mathrm{GF}(2)$\), then solve the system. The unique non-zero solution is $(s)$.

We confirmed the theory against:
- Wikipedia – Simon’s problem
- Qiskit Textbook (official “Simon’s Algorithm” chapter)
- Nielsen & Chuang

### 5. Collecting Samples & Solving

We ran multiple rounds of:

```bash
curl -s -X POST http://IP:PORT/api/run \
  -H "Content-Type: application/json" \
  -d '{"layer":"H","shots":256}'
```

Filtered out the all-zero strings, turned the rest into \(\mathrm{GF}(2)\) vectors, and ran Gaussian elimination to extract the 1-dimensional nullspace.

A short Python script did the whole loop: collect → deduplicate → solve → submit to `/api/forge`.

### 6. Full Attack Chain Recap

1. Spawn Docker → get IP:PORT  
2. `nmap -sV` reveals Gunicorn (HTTP)  
3. Fetch `/` and especially `/app.js` → discover the three API endpoints  
4. `GET /api/oracle` → confirm \(n=64\) and the exact circuit  
5. Repeatedly `POST /api/run` with `layer=H` → collect vectors orthogonal to \(s\)  
6. GF(2) linear algebra → recover \(s\)  
7. `POST /api/forge` with the 64-bit string → flag  

### 7. All the Stupid Things We Hit

| Problem                        | What happened                                      | Fix                                      |
|--------------------------------|----------------------------------------------------|------------------------------------------|
| nc hangs forever               | Server is HTTP, not a text protocol                | Always start with `nmap -sV`             |
| VPN interference               | TCP handshake succeeds, data never arrives         | Disable GlobalProtect / switch network   |
| Instance expires mid-solve     | Connection refused                                 | Respawn and update IP:PORT               |
| Not enough independent vectors | Nullspace dimension > 1                            | Collect more rounds (6–10 × 256 shots)   |
| Forgetting to drop the zero vector | Pollutes the matrix                             | Explicitly filter strings containing no `1` |
| Trying fancy gates too early   | Unnecessary complexity                             | Just use `layer=H` — it is already Simon |

### 8. Lessons Learned

1. **nmap -sV is non-negotiable.** Half the “stuck” feeling in modern HTB challenges comes from guessing the wrong protocol.
2. Frontend JavaScript is often the real documentation. `app.js` told us everything the challenge authors wanted us to know.
3. Simon’s algorithm is still one of the cleanest quantum CTF primitives: collect hyperplanes orthogonal to \(s\), solve the linear system.
4. At \(n=64\) you need both volume of samples and decent linear-algebra hygiene.
5. The narrative (“First Mark”, “Break the seal”, “Resonance”) maps almost 1-to-1 onto the technical steps.

### One-sentence path

HTTP discovery → `/api/oracle` confirms Simon’s problem \(n=64\) → repeated `layer=H` queries → GF(2) nullspace → `/api/forge` → flag.

Clean, elegant quantum period-finding challenge that rewards proper recon and a solid understanding of the underlying linear algebra over $(mathrm{GF}(2))$.