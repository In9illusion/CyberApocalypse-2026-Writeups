### Challenge Description
```
False Ferry
Lysa Harrowmere reaches the lower city ferry piers while Stormbound soldiers wait for the morning boat. They are supposed to cross the river and guard the east road before Vaultrune's next patrol moves through. The route board says the boat goes to the east road landing, but the crew roster sends it to a dock controlled by Vaultrune. If Lysa warns the soldiers openly, Vaultrune's men can claim she started a fight at the pier. If she confronts the ferry master, his guards can tear down the roster and post the correct one. Lysa has one job: find the earlier crossing list, prove who changed the dock, and get the soldiers onto the right boat before Vaultrune cuts the road.
```

In short: restricted IAM user → Systems Manager Parameter Store namespace exploration → AssumeRole (with ExternalId) → versioned S3 object.

Example target: `154.57.164.69:31422` (AWS API port) + `154.57.164.69:30472` (briefing port). Instances expire, so expect to respawn more than once.

### Tools & Environment
- AWS CLI v2
- Custom endpoint (LocalStack-style simulated AWS)
- Starting credentials from the briefing page:
  - IAM User: `coalition-ferry-clerk`
  - Access Key ID: `AKIA7DCW6G0C6O69Q6B4`
  - Secret Access Key: `eRQvottBDvF3T8h92vVGiZkxF0jMm3C6GtjeMypl`
  - Region: `us-east-1`
- Key hint on the page: Crossing batch metadata lives in Systems Manager under `/ferry/crossing/`. **You must catalog the namespace before reading any parameter value.** And of course… “Break the seal.”

### 1. Environment Setup & First Round of Pain

The briefing page (port 30472) gives the credentials and is very clear:

```
Point AWS_ENDPOINT_URL at the instance IP and the AWS API port from your instance card, not the briefing port in the address bar.
```

Correct setup:

```bash
export AWS_ENDPOINT_URL=http://154.57.164.69:31422
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=AKIA7DCW6G0C6O69Q6B4
export AWS_SECRET_ACCESS_KEY=eRQvottBDvF3T8h92vVGiZkxF0jMm3C6GtjeMypl
unset AWS_SESSION_TOKEN
```

Verify:

```bash
aws sts get-caller-identity
```

**Stupid mistakes / pitfalls #1:**
- Hitting the API port without credentials → `AccessDeniedException: User is not authorized to perform: MissingAuthentication`
- Accidentally pointing `AWS_ENDPOINT_URL` at the briefing port (30472) → completely wrong service
- Docker instance timing out mid-solve → ports die, everything resets, forced respawn

### 2. Actually Following the Prompt: Catalog First

The challenge forces you to catalog before reading values. So we did:

```bash
aws ssm describe-parameters \
  --parameter-filters "Key=Name,Option=BeginsWith,Values=/ferry/crossing"

# or the cleaner version that only shows metadata
aws ssm get-parameters-by-path \
  --path /ferry/crossing/ \
  --recursive \
  --query 'Parameters[*].[Name,Type,LastModifiedDate,Version]' \
  --output table
```

Successfully listed:
- `/ferry/crossing/live-crossing-id`
- `/ferry/crossing/CROSSING-7A3F`
- `/ferry/crossing/CROSSING-DRAFT-8D40`
- `/ferry/crossing/CROSSING-CLOSED-5E22`
- several `/ferry/crossing/CROSSING-VOID-*`

All Version 1, Type String.

**Stupid mistakes / pitfalls #2:**
- Jumping straight to `GetParameterHistory` → AccessDenied (no history permission)
- Trying `iam get-user`, `list-attached-user-policies`, `list-user-policies` → all denied. The clerk has almost zero ability to look at itself.

### 3. Reading the Live Pointer and the Real Parameter

First the live one:

```bash
aws ssm get-parameter --name /ferry/crossing/live-crossing-id
```

Value: `CROSSING-7A3F`

Then the important one:

```bash
aws ssm get-parameter --name /ferry/crossing/CROSSING-7A3F
```

This returned the gold JSON:

```json
{
  "crossing_id": "CROSSING-7A3F",
  "status": "AUTHORIZED",
  "issuer": "stormbound-coalition-ferry-office",
  "scanner_role_arn": "arn:aws:iam::584729103648:role/ferry-crossing-scanner",
  "scanner_external_id": "ferry-crossing-scanner-7a3f",
  "manifest_bucket": "ferry-crossing-manifest",
  "manifest_object_key": "manifests/morning-crossing-order.txt",
  "manifest_version_id": "39660ba0-5fb0-48b0-b54b-8041e5fa3555",
  "record_type": "crossing_manifest"
}
```

I also checked the DRAFT / CLOSED / VOID series. Confirmed they were just third-party-archive decoys (no role_arn, different object keys).

**Stupid mistakes / pitfalls #3 (important):**
- After the initial catalog, a full `get-parameters-by-path` (without the restrictive query) started returning AccessDenied.
- Permissions seem to allow listing/describing first, but then tighten when you try to bulk-read values. Had to switch to individual `get-parameter` calls.
- This perfectly matches the “Catalog the namespace before you read any parameter value” design.

### 4. AssumeRole – Breaking the Seal

Used the Role + ExternalId from the JSON:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::584729103648:role/ferry-crossing-scanner \
  --role-session-name ferry-scanner \
  --external-id ferry-crossing-scanner-7a3f
```

Got temporary credentials. Overwrote the environment variables and verified:

```bash
export AWS_ACCESS_KEY_ID=ASIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

aws sts get-caller-identity
```

Now showing:
```
arn:aws:sts::584729103648:assumed-role/ferry-crossing-scanner/ferry-scanner
```

**Key realization:** The ExternalId is mandatory. Without it the assume-role fails. Classic controlled role assumption, and the direct mapping of “Break the seal.”

### 5. Reading the Versioned S3 Manifest

```bash
aws s3api get-object \
  --bucket ferry-crossing-manifest \
  --key manifests/morning-crossing-order.txt \
  --version-id 39660ba0-5fb0-48b0-b54b-8041e5fa3555 \
  morning-crossing-order.txt

cat morning-crossing-order.txt
```

The file contains the Crossing Release Record and the flag.

Extra check on versions:

```bash
aws s3api list-object-versions \
  --bucket ferry-crossing-manifest \
  --prefix manifests/morning-crossing-order.txt
```

Multiple VersionIds appear (the “earlier crossing list” vs the currently authorized one), but the flag lives in the exact version the parameter handed us.

### 6. Full Attack Chain Recap

1. Clerk credentials + correct Endpoint → STS identity check  
2. **Forced catalog** of `/ferry/crossing/` (describe / get-parameters-by-path with query)  
3. Read `live-crossing-id` → points to `CROSSING-7A3F`  
4. Single `get-parameter` on `CROSSING-7A3F` → Role ARN + ExternalId + S3 Bucket/Key/VersionId  
5. `sts assume-role` (with ExternalId) → switch to `ferry-crossing-scanner`  
6. `s3api get-object` with the exact VersionId → final record + flag  

### 7. All the Stupid Things We Hit

| Problem | What happened | Fix |
|---------|---------------|-----|
| Port confusion | Hit briefing port or forgot Endpoint | Always use the API port + `AWS_ENDPOINT_URL` |
| Early history read | `GetParameterHistory` AccessDenied | Drop it, stay on current version |
| Early IAM introspection | Every IAM call denied | Stop trying to look at yourself, focus on SSM + STS |
| Bulk read tightened | Later `GetParametersByPath` denied | Switch to single `get-parameter` |
| Forgot ExternalId | assume-role fails | Must include `--external-id` |
| Ignored VersionId | Might grab the wrong object version | Use the exact VersionId from the parameter |
| Instance timeout | Connection refused | Respawn the Docker |

### 8. Lessons Learned

1. **Follow the stated order.** “Catalog before you read” is not flavor text — it’s part of the permission design.
2. **Permissions can tighten mid-challenge.** What works for listing may stop working for full value reads. Be ready to drop to the smallest possible API.
3. **ExternalId is a common role-protection key** in CTFs. Don’t skip it.
4. **S3 Versioning is the classic way to hide historical data.** The parameter even hands you the correct VersionId.
5. **Narrative maps cleanly to technique.** “Earlier crossing list,” “who changed the dock,” and “Break the seal” line up with S3 versions, issuer/LastModified, and AssumeRole + ExternalId.
6. **Cloud challenge environment details matter a lot.** Endpoint, Region, credential overwrite order, clearing SessionToken — any of these wrong and everything after it dies.
7. **Decoy parameters are normal.** The VOID / DRAFT / CLOSED series exist to waste time. Only the live → AUTHORIZED record actually matters.

### One-sentence path

Clerk → Catalog SSM `/ferry/crossing/` → read live + CROSSING-7A3F → grab scanner role + external_id + exact S3 version → AssumeRole → read the specified version of the manifest → flag.

Clean AWS permission-chain + versioned-object challenge. Rewards reading the prompt carefully and understanding STS / S3 versioning.