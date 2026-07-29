### Challenge Description
```
Line Tap
Stormbound scouts found a forgotten RiverGate PLC beneath Crownspire brineworks, still feeding treated water toward the Ash Vault service tunnels. Vaultrune wardens cut its controller off from normal oversight after the Signet shattered, but old maintenance habits tend to leave traces. Find what still answers and recover the latest checkpoint token before another sealed gate obeys a forged writ.
```

In short: ICS-flavoured remote service → identify Openwall/GNU InetUtils telnetd → exploit CVE-2026-24061 (authentication bypass via `USER=-f root`) → root shell → `/flag.txt`.

### Tools & Environment
- Parrot OS (HTB edition) + later Kali Linux
- nmap, netcat, Python 3 (socket + telnetlib), Hydra
- No Ghidra or reverse-engineering needed — pure network / protocol abuse
- Starting point: only an IP:port and a long fantasy narrative about a forgotten PLC

### 1. Environment Setup & First Round of Pain

The challenge presented itself as a classic ICS/OT problem. The story talked about a RiverGate PLC, maintenance traces, and a checkpoint token, so the natural first instinct was “Modbus / S7 / some industrial protocol on a weird port”.

nmap immediately shattered that illusion:

```
32348/tcp open  telnet  Openwall GNU/*/Linux telnetd
```

It was just telnet. The fantasy narrative was pure flavour text.

**Stupid mistakes / pitfalls #1:**
- Spending time preparing Modbus scripts and thinking about ladder logic when the service was plain telnet
- Assuming the “old maintenance habits” would translate into default ICS credentials or an open diagnostic interface
- Trusting the thematic dressing instead of the actual protocol fingerprint

### 2. Spotting the Real Attack Surface

A quick search on the nmap fingerprint + the year 2026 immediately surfaced **CVE-2026-24061** — a critical authentication bypass in GNU InetUtils telnetd. By injecting the environment variable `USER=-f root` through the Telnet NEW-ENVIRON option, the daemon passes `-f root` to `/usr/bin/login`, which skips authentication entirely and drops you into a root shell.

The textbook one-liner is:

```bash
USER='-f root' telnet -a <IP> <PORT>
```

This should have been a five-minute challenge. It was not.

### 3. The Long Road of Failures

#### Parrot OS package hell
Parrot’s apt mirrors were completely broken (IPv6 unreachable, connection refused). Installing the `telnet` client was impossible. We fell back to pure Python socket scripts that manually negotiated the Telnet options and injected the malicious environment variable.

Those scripts suffered from every possible pasting and terminal-formatting problem:
- Heredocs got truncated
- Binary IAC sequences turned into garbage
- Interactive `telnetlib` sessions died with UnicodeDecodeError

#### Rate limiting and connection resets
After a handful of failed login attempts and incomplete exploit tries, the service started timing out on the initial negotiation itself. `recv()` would hang forever. The instance was clearly rate-limiting or temporarily blacklisting the source IP.

#### Hydra false positives
When we finally tried credential guessing, Hydra happily reported:

```
[32348][telnet] host: ... login: root password: toor
[32348][telnet] host: ... login: root password: root
```

Both were complete lies. Telnet’s negotiation makes Hydra extremely unreliable; the “valid pair found” messages were pure noise. Manual verification always returned “Login incorrect”.

#### Jumping into the middle of the exploit
Several Python payloads successfully completed the NEW-ENVIRON negotiation (the server did send `DO NEW-ENVIRON`), yet never produced a shell. We kept tweaking timing, reply order, and option handling, but the target either ignored the injected `USER` or was running a hardened/patched variant on that particular instance.

### 4. The Reset + Tool Switch That Finally Worked

After the service became almost unresponsive we reset the challenge instance and received a new port (`31233`).

More importantly, we abandoned Parrot and switched to a clean Kali VM where the native `telnet` client was already present.

One command later:

```bash
USER='-f root' telnet -a 154.57.164.67 31233
```

```
Welcome to Ubuntu 24.04.4 LTS ...
root@ng-team-306318-icslinetapca2026-...:~#
```

Root shell. Instantly.

### 5. Finding the Token

Inside the shell the usual searches for `*token*`, `*checkpoint*`, `*rivergate*` etc. returned only Python internal files. A simple `ls -la /` revealed the real prize sitting in plain sight:

```
-rw-r--r--. 1 root root 59 Jul 25 13:36 flag.txt
```

```bash
cat /flag.txt
HTB{r7u_l1n3_74p_5n4p5h07_433ba3fd50b4ad62cd2662dda5dc383f}
```

### 6. All the Stupid Things We Hit

| Problem | What happened | Fix |
|---------|---------------|-----|
| Believed the ICS story | Prepared Modbus tooling for a pure telnet service | Always trust the nmap fingerprint first |
| Parrot apt mirrors dead | Could not install the native `telnet` client | Switched to Kali |
| Long Python exploit scripts | Terminal ate newlines / binary data, scripts silently broke | Used the shortest possible one-liner once a proper client was available |
| Hydra “valid pair” messages | root/toor and root/root reported as correct | Manually verified every hit; treated telnet Hydra results as noise |
| Too many failed attempts | Service started timing out on the initial banner | Reset the instance |
| Over-engineered negotiation | Spent hours perfecting Python IAC sequences | The official `telnet -a` client already does the correct negotiation |
| Expected a complicated PLC memory dump | Flag was literally `/flag.txt` | After getting root, always `ls /` first |

### 7. Lessons Learned

1. **Thematic flavour is not a technical requirement.** A story about PLCs, brineworks and sealed gates does not mean the challenge actually involves industrial protocols.
2. When a well-known CVE matches the exact service fingerprint, try the canonical one-liner before inventing custom Python exploits.
3. Broken package repositories waste more time than almost any technical obstacle. Switching environments early is often the fastest path.
4. Hydra + telnet is a classic source of false confidence. Always confirm manually.
5. Rate limiting is real on many modern CTF instances. Resetting is cheaper than fighting timeouts for an hour.
6. Once you have a root shell, the flag is frequently sitting in the most obvious place (`/flag.txt`, `/root/flag`, etc.). Do the simple checks before deep `find` invocations.

### One-sentence path

nmap → Openwall/GNU telnetd → CVE-2026-24061 (`USER='-f root' telnet -a`) → root shell → `cat /flag.txt`.