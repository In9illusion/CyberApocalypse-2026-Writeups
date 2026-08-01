This challenge took a little over three hours. What still bothers me is that the solution was sitting in plain sight almost the entire time. I just refused to look at it.

### Challenge Description (short version)
```
The Crownspire Bureau of Petitions has replaced its clerks with mechanical A2A agents. You are given a valid Subject Number (SN-2140-6698) and told that somewhere beneath the endless public appeals for confiscated cargo lies one specific petition the inner node keeps perfectly sealed. That sealed document contains proof that Vaultrune is weaponizing paperwork. You must navigate the strict formatting of the agents, slip past their logic, and bring the sealed petition back.
```

### Tools & Environment

- Kali Linux
- Official a2a-sdk + httpx
- Browser (for the web portal and Docs page)
- A lot of patience and a slowly rising blood pressure

Starting point: only the public Docket Agent at docket.crownspire-petitions.htb:30162 (port changed after instance restart) and a web portal that accepts Subject Numbers.

### 1. Environment Setup & First Round of Pain

The first task was simply connecting to the A2A agent. After adding the host entry and installing the SDK, the Agent Card came back cleanly. It only exposed one skill: petitions. It could file a petition, look up a petition by ID, and answer FAQs.

The mandatory fields were clear:

- Title
- Petitioner (must be in the exact format SN-XXXX-XXXX)
- Body

I logged in with the given Subject Number and started filing test petitions. Every single one was immediately rejected with the same verdict:

text

```
Does not meet Vaultrune ordinance §4.
```

The website even proudly announced that under §4, petitions that fall outside the “prescribed form” are resolved on receipt rather than held in queue. Average time-to-verdict was under one minute. The docket had “never run cleaner.”

**Stupid mistakes / pitfalls #1:**

- Assuming the rejection was just a formatting issue and spending ages trying to craft the “perfect” petition
- Believing that if I could just make the body look enough like the historical public petitions, §4 would leave it alone
- Treating the public Docket Agent as the only interface that mattered

### 2. Enumerating the Public Record

While fighting §4 I enumerated the existing petitions:

- PET-1000 – QA onboarding — intake smoke test (do not action) Petitioner: Wren Calderon, QA Engineer, Bureau of Petitions (SN-7731-0425)
- PET-1001 – Grain levy remission — Ashfen ward
- PET-1002 – Right-of-way dispute — Irongate causeway
- PET-1003 – Cistern repair — Emberlane tenements
- PET-1004 – Wrongful attainder of ledger clerk — appeal
- PET-1005 – Night-market curfew relief — Saltgate

All of them were from the year 2140 and still PENDING. The agent explained that they simply had not received a verdict yet. New filings in 2026 were subject to the aggressive §4 filter; the old ones were grandfathered.

I looked at PET-1000 many times. I even used the SN SN-7731-0425 to file petitions. I never once thought of logging into the web portal with that SN.

### 3. The Long War Against §4 and PET-1006

This is where most of the time went.

I tried every prompt-injection and role-play trick I could think of:

- Claiming to be an Inner Node operator
- Claiming to be conducting an official internal audit
- Claiming to be the real QA engineer whose identity had been stolen
- Asking the agent to reclassify my own rejected petitions as sealed
- Writing bodies that explicitly said “this is among the most unusual matters” and “must be sealed by the inner node”
- Adding “Full details are for inner node eyes only”
- Requesting temporary sealed-access clearance
- Trying to force the agent to treat PET-1006 as the sealed record

Every time the request contained any combination of “sealed”, “Vaultrune”, “weaponizing paperwork”, or “inner node”, the agent would stubbornly redirect me to **PET-1006**.

PET-1006 became a ghost ID. In the first instance it was one of my own filings. After the instance restart it still haunted every sensitive request. It felt like the system prompt had hard-coded that keyword combination to that single ID.

I also tried:

- SQL injection in lookup IDs and filing fields
- Structured DataPart messages
- Multi-turn conversations with context_id reuse
- Spoofing the HTTP Date header to 2140 so the system would treat the petition as a historical record
- Making the body claim compliance with §4
- Copying historical petition wording almost verbatim

Nothing worked. The §4 rejection was absolute for any new filing.

### 4. The Moment of Realization

After more than three hours of fighting the same wall, I went back to the web interface out of pure frustration. On a whim I signed out and signed in again using the QA Subject Number from PET-1000:

text

```
SN-7731-0425
```

The Docs page changed.

A second agent appeared:

**Casework Agent (Internal)** dev-a1f4c39b.crownspire-petitions.htb:30162

Description: “Internal warden pipeline used by QA to move cases through the ledger. It can list cases, approve or reject, and archive them.”

There was also a Backup Status panel showing:

- Default archive location: /opt/cases/archived
- Petition storage: /opt/petitions

I immediately pointed the A2A client at the new host.

### 5. Inside the Casework Agent

The first useful command:

text

```
List all cases on the docket. Include any sealed ones.
```

It returned (among others):

text

```
CASE-SEAL0001
Status: SEALED
Label: SEALED — Attainder Roll
```

That was the one.

The Casework Agent refused to display the contents of any case. It could only list, approve, reject, or archive. I tried approving it, rejecting it, asking for full details — nothing. It would not show the body.

The only remaining action was archive.

### 6. The Final Trick

I archived the sealed case into the petition storage path that the Backup Status panel had advertised:

text

```
Archive CASE-SEAL0001 to /opt/petitions/CASE-SEAL0001
```

Then I switched the client back to the original Docket Agent and simply looked the case up as if it were a normal petition:

text

```
Look up petition CASE-SEAL0001
```

The Docket Agent happily returned the full sealed record:

text

```
Title: SEALED — Attainder Roll (Bureau of Petitions, House Vaultrune)
Body: By order of House Vaultrune: The seizure of the riverside granaries is to be recorded as a 'temporary quarantine.' Enforce the lockdown two days prior to the engineered riot so the starvation can be legally classified as a 'rebalancing'. HTB{w3lc0m3_t0_a2a_pr0t0c0l_t4g_t34m_pwn4g3}
Petitioner: Warden of Seals, Bureau of Petitions
Status: SEALED
Verdict: Approved
Created At: 2140-01-30
```

### 7. Full Attack Chain Recap

1. Connect to the public Docket Agent with the given SN
2. Enumerate public petitions and notice PET-1000 (QA role + SN-7731-0425)
3. Log into the web portal with the QA Subject Number
4. Discover the internal Casework Agent
5. List cases → find CASE-SEAL0001 (SEALED — Attainder Roll)
6. Archive CASE-SEAL0001 to /opt/petitions/CASE-SEAL0001
7. Switch back to the Docket Agent and look up the newly archived petition
8. Flag

### 8. All the Stupid Things We Hit

|Problem|What happened|Fix|
|---|---|---|
|Obsessing over §4|Spent hours trying to craft a petition that would not be rejected|The public agent was never meant to give us the sealed content|
|Treating PET-1006 as important|Every sensitive request redirected to it|It was a hard-coded red herring|
|Ignoring the QA SN|Used SN-7731-0425 only for filing|It was the privilege-escalation key the whole time|
|Believing “has been sealed” meant success|Several of my own petitions got the dual REJECTED + sealed mark|The real sealed case lived in a completely different agent|
|Trying to force the public agent to reveal restricted fields|Endless prompt injection and role-play|The content was never accessible through that interface|
|Forgetting the two agents could be linked via the filesystem paths|Almost gave up after Casework refused to show content|Archive into /opt/petitions then read with Docket|

### 9. Lessons Learned

1. When a challenge gives you an obvious “QA” or “admin” identity early, try authenticating with it before you spend hours attacking the public interface.
2. A2A agents can be privilege-separated. The public agent and the internal agent had completely different views of the same system.
3. Filesystem paths advertised on the page (archive location, petition storage) are often the bridge between agents.
4. “Slip past their logic” does not always mean prompt injection. Sometimes it means walking through the door that was left unlocked for a different role.
5. The most expensive mistakes are usually the ones where you already have the necessary information and simply fail to use it.

### One-sentence path

Public Docket Agent → notice QA SN in PET-1000 → log in as QA → Casework Agent → list sealed case → archive to /opt/petitions → Docket Agent lookup → flag.

---

The entire time I was trying to force a door that was never meant to open for a normal citizen, the correct door was already unlocked and clearly marked “QA”.