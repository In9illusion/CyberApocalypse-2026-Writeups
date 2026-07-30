### Challenge Description
```
Hollow Courier
Stormbound engineers reached the lower Crownspire brineworks after alarms echoed through the Ash Vault tunnels. The old seal is still guarded by a forgotten HMI and interlock PLC. The PLC exposes a Modbus/TCP control surface alongside the HMI, so players must recover enough process and ladder-logic context to operate it safely. Regain enough control to stop the unstable automatic cycle, manipulate the brine process into the seal-ready state, and trigger the final seal alarm that reveals the checkpoint token in the alarm table.
```

In reality this was a **Secure Coding** challenge.  
You are given a running Flask application behind a Caddy reverse proxy, a developer Git account, and the task of identifying and fixing a security flaw without breaking normal behaviour. The final flag is only released after a clean Pull Request passes both Soft Score and Hard Score checks.

### Tools & Environment
- Git + the supplied developer credentials
- Local Python 3 + pytest
- Docker (optional, for closer-to-prod testing)
- Browser for the `/pulls` review page and the final `/flag` endpoint
- Starting point: only the challenge instance URL and the Git clone command

### 1. Environment Setup & First Round of Confusion

The flavour text screamed ICS / Modbus / PLC.  
My first instinct was therefore to nmap the given port, look for Modbus/TCP, and start writing register-read scripts.

**Stupid mistakes / pitfalls #1:**
- Treating the narrative as literal technical guidance
- Spending time on nmap when the port was filtered / the real interface was a web + Git workflow
- Not immediately reading the challenge landing page that clearly stated “Secure Coding Challenge”

Once the landing page was opened, the real task became obvious: clone the repository, find the security issue, fix it cleanly, and push a PR.

### 2. Locating the Real Code

```bash
git clone http://htb_developer:HTBDeveloperPassword@[IP]:[PORT]/git/core_application.git
cd core_application
git checkout -b developer
```

Repository layout:

```
checkpoint/
├── app/               # Flask application
│   ├── __init__.py
│   ├── gate.py
│   ├── db.py
│   ├── pages.py
│   ├── staff.py
│   └── ...
├── conf/Caddyfile
├── tests/
├── seed.sql
└── ...
```

The interesting files were immediately visible: `app/__init__.py`, `app/gate.py`, and later `app/db.py`.

### 3. Architecture Overview

- Flask application listening on 5000
- Caddy reverse-proxy in front (port 8000)
- Staff authentication via session
- Most routes protected by `@staff.require_staff()`
- The most sensitive endpoint `/app/gate/decree` additionally protected by `gate.require_internal()`

`gate.py` contained the trust decision:

```python
INTERNAL_NETWORKS = (
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("127.0.0.2/32"),   # watch relay loopback alias
)

def is_internal_request() -> bool:
    origin = ipaddress.ip_address(request.remote_addr or "")
    return any(origin in network for network in INTERNAL_NETWORKS)
```

### 4. The Core Vulnerability

In `app/__init__.py`:

```python
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=2, x_proto=1, x_host=1)
```

Caddy is the **only** reverse proxy.  
Setting `x_for=2` tells Werkzeug to trust *two* hops of `X-Forwarded-For`.  

An external attacker can therefore send:

```
X-Forwarded-For: 127.0.0.2
```

and `request.remote_addr` becomes the forged internal address.  
This completely bypasses `require_internal()` and allows an unauthenticated caller to hit the decree endpoint — the exact “forge a seal the world cannot tell from genuine” action the story described.

### 5. The Correct Fix

Two principles had to be respected simultaneously:

1. Reduce the trusted hop count to the real number of proxies (`x_for=1`).
2. **Do not** shrink `INTERNAL_NETWORKS`. The private ranges and the special `127.0.0.2` address are required for legitimate internal traffic.

The minimal, correct change is therefore only:

```python
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1, x_host=1)
```

### 6. The Test-Suite Nightmares

After the ProxyFix change, a single test started failing when the full suite was run, while the same test passed in isolation:

```
test_staff_noise_routes_have_workflows
assert b">17<" in supplies.data   # inventory quantity never became 17
```

Root cause: the test fixture does

```python
os.environ["GATE_DB"] = tempfile_path
reload(app_pkg)          # only reloads the top-level package
```

while the original `db.py` evaluated the path once at import time:

```python
DB_PATH = os.getenv("GATE_DB", ...)
```

Under a full test run the module-level constant became stale, the wrong database was used, and the supply update was lost.  

The defensive (and correct) engineering change was to resolve the path on every connection:

```python
def _db_path() -> str:
    return os.getenv("GATE_DB", os.path.join(os.path.dirname(__file__), "..", "gate.db"))
```

### 7. Submission & Bot Feedback Loop

Several iterations were required:

| Attempt | What went wrong | Bot reaction |
|---------|-----------------|--------------|
| 1 | Correct ProxyFix + incomplete/duplicate `init_db` | Soft-score rejection – “duplicate function definition” |
| 2 | Heredoc commands partially written into source files | Syntax errors, collection failures |
| 3 | Clean three-file patch (`__init__.py`, `db.py`, `gate.py`) | Soft score accepted → Hard score testing started |
| Final | Same clean patch | “HARD CORE TESTING HAS SUCCESSFULLY PASSED! GO GET THAT FLAG!” |

The final commit message that satisfied both scores:

```
fix(security): correct ProxyFix hop count to prevent IP spoofing

- Change ProxyFix x_for from 2 to 1 to match the single trusted Caddy reverse-proxy hop.
- Prevent untrusted client X-Forwarded-For headers from controlling the reconstructed client IP.
- Resolve GATE_DB dynamically at connection time so the application remains correct under the test fixture’s reload pattern.
- Preserve the original INTERNAL_NETWORKS allowlist for legitimate internal traffic.
```

### 8. All the Stupid Things We Hit

| Problem | What happened | Fix |
|---------|---------------|-----|
| Believed the ICS/Modbus flavour text | Wasted time on nmap and protocol analysis | Read the actual challenge landing page |
| Shrinking INTERNAL_NETWORKS | Would have broken legitimate internal traffic | Left the allow-list untouched |
| Module-level `DB_PATH` | Full test suite failed while isolated tests passed | Made path resolution dynamic |
| Duplicate `init_db` definition | Soft-score rejection | Clean single definition only |
| Broken `cat > file << 'EOF'` commands | Python files contained shell syntax | Re-wrote the files from scratch with complete heredocs |
| Leaving debug prints in `pages.py` | Extra noise in the patch | `git checkout -- app/pages.py` before final commit |

### 9. Lessons Learned

1. **Narrative flavour is not the technical specification.**  
   Always read the concrete challenge instructions first.

2. **Trusted-proxy hop count must exactly match the real deployment.**  
   One extra trusted hop is enough to turn `X-Forwarded-For` into a privilege-escalation primitive.

3. **Business allow-lists and trust boundaries are different things.**  
   Fixing the trust boundary must not destroy the legitimate traffic the application was designed to accept.

4. **Test fixtures that rely on `reload` + environment variables are fragile.**  
   Application code that evaluates configuration at import time will bite you. Prefer late binding.

5. **Soft Score cares about cleanliness as much as correctness.**  
   Duplicate functions, leftover debug code, or unrelated changes will be rejected even if the security fix is right.

6. **Operational discipline matters.**  
   A single mangled heredoc can cost more time than the actual vulnerability analysis.

### One-sentence path

Clone the developer repository → notice `ProxyFix(x_for=2)` behind a single Caddy hop → reduce it to `x_for=1` while preserving the original internal-network allow-list → make database path resolution dynamic so the reload-based test fixture stays correct → push a clean PR → collect the flag.

A clean secure-coding exercise that rewards precise understanding of reverse-proxy trust boundaries and refuses to accept anything less than a production-grade patch.