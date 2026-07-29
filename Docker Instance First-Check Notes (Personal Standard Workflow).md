### 1. Standard Process (Do this every single time)

```bash
# 1. Click Spawn on the challenge page and grab the IP + port
# Example: 154.57.164.xx:xxxxx

# 2. First thing – always run nmap to see what the service actually is
nmap -sV -Pn -p <PORT> <IP>

# 3. Choose your connection method based on the service type
```

### 2. Common Cases

| What nmap shows              | Connection type     | Recommended tools              |
|-----------------------------|---------------------|--------------------------------|
| http / Gunicorn / nginx     | Web service         | curl or Python requests        |
| just "open" (no http)       | Plain TCP           | nc / ncat / pwntools           |
| ssh                         | SSH                 | ssh user@IP -p <PORT>          |
| mysql / redis / etc.        | Protocol-specific   | Corresponding client commands  |

### 3. Useful Command Templates

**A. Plain TCP text service (most common for non-web)**

```bash
nc <IP> <PORT>
# or
ncat <IP> <PORT>
```

**B. Want verbose errors + timeout**

```bash
timeout 5 nc -v <IP> <PORT>
```

**C. HTTP service (like many web / API challenges)**

```bash
# Homepage
curl -v http://<IP>:<PORT>/

# API endpoint
curl -s http://<IP>:<PORT>/api/xxx | python3 -m json.tool

# POST request
curl -s -X POST http://<IP>:<PORT>/api/run \
  -H "Content-Type: application/json" \
  -d '{"key":"value"}'
```

**D. Python interaction (recommended for scripting)**

```python
import socket

s = socket.socket()
s.settimeout(5)
s.connect(("<IP>", <PORT>))
print(s.recv(4096).decode())
# continue interacting...
```

Or with pwntools (more powerful):

```python
from pwn import *

r = remote("<IP>", <PORT>)
print(r.recv())
r.sendline(b"xxx")
r.interactive()
```

### 4. Why it didn’t connect at first on some challenges

nmap showed **HTTP (Gunicorn)**.  
That means it is **not** a plain TCP text protocol.  
Sending raw text with `nc` does nothing — the server expects proper HTTP requests.  
You must use `curl` or `requests`.

### 5. Habit to build

Every time you spawn a Docker instance, the **first** command should always be:

```bash
nmap -sV -Pn -p <PORT> <IP>
```

Only after you see the service type do you decide whether to use `nc`, `curl`, `ssh`, or something else.

### Correct syntax examples

Wrong way people often type:
```bash
nmap -sV -Pn -p 154.57.164.67:30663   # ← nmap does not accept IP:PORT like this
```

Correct way:

```bash
nmap -sV -Pn -p 30663 154.57.164.67
```

- `-p 30663` → the port goes here  
- `154.57.164.67` → the IP goes at the end

### Useful variants

```bash
# Basic (recommended)
nmap -sV -Pn -p 30378 154.57.164.72

# Faster
nmap -sV -Pn -T4 -p 30378 154.57.164.72

# More verbose
nmap -sV -Pn -p 30439 154.57.164.75 -vv
```

---

**Quick reminder for myself**  
Always start with nmap.  
Knowing whether it’s HTTP, raw TCP, SSH, or something else saves a lot of “why isn’t it responding” time and helps understand what the challenge is actually exposing.