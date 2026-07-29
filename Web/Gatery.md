### Challenge Description
```
Gatery
Lysa Harrowmere reaches Crownspire with proof that a trusted castle informant is selling patrol routes to the enemy. The information is being used to ambush messengers, delay supplies, and keep Stormbound’s allies divided. The only person who can act on the proof is inside the castle for a closed council, but Lysa’s name has been removed from the entry list and the guards have orders to admit no unscheduled visitors. If she waits, the council ends and the traitor disappears with the next route packet. If she speaks openly at the gate, the proof is seized before it reaches the right hands. Lysa must trick the guarded passage, get inside, and place the evidence with the one ally who can expose the leak before the enemy moves again.
```

In short: Web challenge (Nginx + Vite/React SPA + Bun/Elysia backend) → identify signed-session logic → exploit Elysia cookie signature validation bypass (when `secrets` is supplied as an array) → set `session=inside` → retrieve flag.

### Tools & Environment
- Parrot OS (HTB edition) on a VM, source files originally on the Windows host
- nmap, curl, browser, Python (occasionally)
- Source code provided by the challenge
- Starting point: only an IP:port, a long fantasy narrative, and the full source tree

### 1. First Contact & Architecture Confusion

The challenge handed over both a remote instance and the complete source. The story talked about tricking a guarded passage and placing evidence inside a castle, so the mental model was “some form of authentication or entry-list bypass”.

nmap showed only one open port:

```
30378/tcp open  http  nginx 1.28.3
```

curl on the root path returned a classic Vite-built SPA:

```html
<title>Gatery</title>
<script type="module" ... src="/assets/js/index-BcDa1MFc.js"></script>
...
<div id="root"></div>
```

At this stage the difference between frontend routes and real backend endpoints was still fuzzy. Accessing `/flag` sometimes appeared to return plain text (“Gate locked / Music on”) and sometimes the same SPA HTML, depending on the client. This produced unnecessary confusion about whether a real file existed or whether the frontend was simply rendering a locked-gate view.

**Stupid mistakes / pitfalls #1:**
- Spending time trying to interpret inconsistent responses to `/flag` instead of immediately reading the Nginx configuration
- Not yet internalising that `try_files $uri $uri/ /index.html;` turns almost every unknown path into a frontend route
- Treating the fantasy narrative as a possible technical hint rather than pure flavour

### 2. Reading the Source (the correct direction)

Once the source tree was transferred into the VM, the important files became clear:

- `config/nginx.conf` – `/api/` is reverse-proxied to `127.0.0.1:3000`, everything else falls back to the SPA
- `Dockerfile` – flag is copied to `/flag.txt` inside the container; the backend is a Bun process managed by supervisord
- `app/index.ts` – the entire authentication and gate logic

The backend (Elysia) implements a simple state machine with a signed cookie named `session`:

- Successful login → `session = "admin"`
- Calling `/api/gate/enter` while holding a valid session → `session = "inside"`
- `/api/flag` only succeeds when `session.value === "inside"`

The admin password is generated with `randomBytes` on every container start and never printed. Normal login is therefore impossible.

### 3. The Long Road of Partial Understanding

Several dead ends appeared while trying to understand how to reach the `"inside"` state:

- Looking for client-side-only checks in the React code (the real gate is server-side)
- Wondering whether the random password could be recovered or whether a default account existed
- Experimenting with normal login flows and watching the 401/403 responses
- Not immediately questioning the cookie configuration itself

The critical line in `index.ts` was:

```ts
const app = new Elysia({
  cookie: {
    secrets: [sessionSecret],   // array
    sign: [sessionCookie]
  }
})
```

At the time the significance of passing an **array** to `secrets` was completely unknown. The difference between a string secret and an array (rotation path) had never been encountered before.

### 4. The Actual Vulnerability

Research into Elysia’s cookie handling (version 1.4.18) revealed a signature-validation bug that appears when `secrets` is supplied as an array. The internal `decoded` flag is not correctly set to `false` when every secret fails to verify, so an **unsigned** cookie is still accepted.

Consequently the one-liner becomes:

```bash
curl -s -X POST http://154.57.164.72:30378/api/flag \
  -H "Cookie: session=inside" \
  -H "Content-Type: application/json" \
  -d '{}'
```

The backend treats the cookie as valid, sees the value `"inside"`, and returns the flag.

### 5. All the Stupid Things That Happened

| Problem | What happened | Fix |
|---------|---------------|-----|
| SPA fallback confusion | `/flag` sometimes looked like a real file, sometimes like HTML | Read `nginx.conf` first; understand `try_files` |
| Inconsistent tool vs curl output | Different clients produced different responses for the same path | Stick to one client (curl) and ignore tool artefacts |
| File transfer friction | Source lived on Windows, work happened inside Parrot VM | Simple HTTP server on the host + wget, or shared folder |
| Over-focus on frontend | Spent time examining React routes and assets | Real authority checks live only in the Elysia backend |
| Random password rabbit hole | Briefly considered whether the password could be recovered | Realised login was intentionally impossible; the bypass had to be elsewhere |
| Ignorance of Elysia cookie internals | Did not know that `secrets: [..]` triggers a different code path | Searched for the exact configuration + known issues in that version |
| Expecting a complex multi-step exploit | Assumed the gate would require several carefully ordered API calls | A single forged cookie was enough |

### 6. Lessons Learned

1. **Flavour text is not a protocol specification.** A story about entry lists and guarded passages does not mean the vulnerability lives in the frontend UI.
2. Always map the real request flow (what sets the session, what the final check actually requires) before inventing attack paths.
3. Framework-specific configuration details matter. A single pair of brackets (`secrets: [secret]` versus `secrets: secret`) can change the security properties of cookie handling.
4. When a service uses signed cookies, the verification logic itself is part of the attack surface, not only the values that are signed.
5. Transferring files between host and VM wastes less time once a reliable method (temporary HTTP server, shared folder, etc.) is established early.
6. After the architecture is understood, the shortest path is usually the correct one; elaborate multi-stage exploits are often unnecessary.

### One-sentence path

nmap → nginx + SPA → read nginx.conf & index.ts → notice `secrets: [sessionSecret]` → Elysia cookie signature bypass → `Cookie: session=inside` → `POST /api/flag` → flag.