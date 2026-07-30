### Challenge Description
```
The Compressed Truth
ASHVAULT was never the destination. It was the key. The operational brief recovered from ASHVAULT's search index pointed deeper — to Veylen Marr's inner node, the machine where the Shard Reference custody chains sleep behind his personal credentials. CROWQUILL used what the brief called an unverified password — lifted through a Suncourt intermediary — to authenticate as vmarr and walk through the inner door. No forced entry. No broken locks. Just a stolen name presented at the right threshold, and the machine believed it. What the operative found inside was worse than Cassian had hoped for. The Shard Reference 7 custody chain was there — a full record of where the Greywater Fragment had moved since the night the Signet shattered, whose hands had touched it, and where it now rested. But CROWQUILL did not stop at documents. They came hunting for something deeper: the master store of secrets that Marr kept locked behind a single password — a vault within the vault, holding the keys to every shard the Registry had ever catalogued. To reach it, the operative brought a specialized tool onto the machine, one built not to break a lock but to lift the key from a hand already holding it. With the keys in hand and the records copied, CROWQUILL staged everything — compressed it, sealed it, prepared it for a journey through channels that do not ask questions. The source files were wiped. The tools were removed. The operative disconnected before the custody window closed. No wonder Cassian's men moved when they did. The Registry's inner vault had been standing on the assumption that the right credentials made a man trustworthy. It was a comfortable assumption. It was wrong. The machine has been imaged and handed to you. The files are gone and the archive has left — but the registry remembers the hands that packed it. 7-Zip does not forget which folders were opened, which paths were browsed, and where the staging began. Find the trail. Recover what was taken. Understand what Cassian now holds — and what it means for every shard still in the Registry's keeping. The archive left. Its shadow stayed.
```

In short: Windows disk image → identify the correct user hive (`vmarr`) → recover the dirty NTUSER.DAT → parse 7-Zip’s own registry history (FolderHistory, CopyHistory, ArcHistory, PathHistory, PanelPath0) → reconstruct the entire post-exploitation sequence into seven discrete flags.

### Tools & Environment
- Python 3 + `regipy` + `python-registry`
- Eric Zimmerman’s Registry Explorer (excellent for visual verification)
- Provided image tree containing `C:\Users\` (cyberjunkie, Default, vmarr) and `C:\Windows\System32\Config\`
- Starting point: only the DEFAULT hive files at first

### 1. Wrong Hive, Wrong User, and the Initial Confusion

We were first given the three files from `C:\Windows\System32\Config\` (DEFAULT + LOG1 + LOG2).  
They were classic dirty hives, so we recovered them and started hunting for 7-Zip artefacts. The result was almost completely empty — no useful ArcHistory, almost no ShellBags, nothing related to shards or staging.

We then tried the `cyberjunkie` user hive. Same story: no `Software\7-Zip` key of interest, only a lonely `D:\setup.exe` in UserAssist, and a handful of system applications.

Only after re-reading the scenario (“authenticate as vmarr”) did we realise the obvious: the operator had logged in as `vmarr`. The correct files were:

```
C:\Users\vmarr\NTUSER.DAT
C:\Users\vmarr\ntuser.dat.LOG1
```

**Stupid mistake #1**  
Wasting significant time on the DEFAULT and cyberjunkie hives simply because those were the files that had been uploaded first.

### 2. Dirty Hive Recovery – Non-Negotiable

The `vmarr` hive was dirty (primary sequence number 188, secondary 187).

```python
from regipy.recovery import apply_transaction_logs

apply_transaction_logs(
    "NTUSER.DAT",
    "ntuser.dat.LOG1",
    restored_hive_path="vmarr_NTUSER_recovered"
)
```

After recovery the sequence numbers matched and the critical keys finally appeared under:

```
Software\7-Zip
├── Compression\ArcHistory
├── Extraction\PathHistory
└── FM
    ├── FolderHistory
    ├── CopyHistory
    └── PanelPath0
```

Without applying the transaction log, many of the most recent values would have been missing or inconsistent. This is standard registry forensics hygiene and should never be skipped.

### 3. Reading the 7-Zip History Keys

Once the recovered hive was open, the entire operation became visible in a single place. 7-Zip keeps its own excellent history; there was no need to dig deeply into ShellBags or UserAssist for the core answers.

| Key | Content | What it answered |
|-----|---------|------------------|
| `Extraction\PathHistory` | `C:\Users\vmarr\AppData\Local\Temp\writ\KeeFarce\` | Tool name |
| `Extraction` key LastWrite timestamp | 2026-06-18 13:15:15 | Tool extraction time |
| `FM\FolderHistory` (deepest path inside a .zip) | `...\oath_records_cinderbound_vol2\saltoaths_secretive\` | Deepest folder enumerated inside the archive |
| `FM\CopyHistory` | `C:\Users\Public\Music\saltwork\` | Staging / collection location |
| `Compression\ArcHistory` | `C:\Users\Public\Pictures\shardchain.tar` | Final archive prepared for exfiltration |
| `FM\FolderHistory` | `...\shard_storage\ShardKeepass_FirstMark\` | Location of the master key store |
| `FM\PanelPath0` + newest FolderHistory entry | `C:\Users\vmarr\Desktop\working\` | Last folder 7-Zip was focused on |

### 4. The Staging Mistake (and How We Corrected It)

We initially submitted `C:\Users\vmarr\Desktop\working\` as the staging location because it was both `PanelPath0` and the most recent entry in FolderHistory.

That was wrong.

The real collection point lived in **CopyHistory**:

```
c:\users\public\music\saltwork\
```

CopyHistory records actual file-copy operations — precisely the action described by “files pulled from their places and gathered where the operative controls”. Desktop\working was only the last place the 7-Zip window happened to be looking.

This single mis-mapping cost us a rejected submission and forced a careful re-examination of the difference between browsing history and copy history.

### 5. Final Flag Set

```
1. KeeFarce
2. 2026-06-18 13:15:15
3. saltoaths_secretive
4. C:\Users\Public\Music\saltwork\
5. shardchain.tar
6. C:\Users\vmarr\Documents\Registry\shard_storage\ShardKeepass_FirstMark\
7. C:\Users\vmarr\Desktop\working\
```

### 6. All the Stupid Things We Hit

| Problem | What happened | Fix |
|---------|---------------|-----|
| Started with DEFAULT / cyberjunkie | No useful 7-Zip history | Re-read the scenario — the operator authenticated as `vmarr` |
| Skipped dirty-hive recovery | Recent keys missing or inconsistent | Always compare primary vs secondary sequence numbers first |
| Trusted PanelPath0 for staging | Wrong answer submitted | Distinguish browsing (FolderHistory / PanelPath0) from actual collection (CopyHistory) |
| Over-focused on ShellBags & UserAssist | Wasted time on nearly empty keys | 7-Zip keeps its own high-quality history under `Software\7-Zip` |
| Assumed path case would be consistent | Had to normalise later | Registry stores mixed case; match exactly what is written |

### 7. Lessons Learned

1. Flavour text is not decoration. The single word “vmarr” told us which hive to open.
2. Dirty hive recovery is mandatory. Sequence numbers exist for a reason; ignoring them loses recent activity.
3. Different 7-Zip history keys answer different questions:
   - PathHistory / Extraction → tool drop location and time
   - CopyHistory → real staging / collection
   - FolderHistory → browsing depth and interesting paths
   - ArcHistory → final archive name
   - PanelPath0 → last UI state
4. When two paths look equally plausible, prefer the one that matches the *action* described in the question (copy versus browse).
5. Keep a clean, reproducible analysis path. Once you start guessing, you introduce your own state errors.

### One-sentence path

Recover the dirty `vmarr` NTUSER.DAT → open `Software\7-Zip` → read Extraction, FM and Compression histories in the correct semantic order → map each artefact to the corresponding question → flag.