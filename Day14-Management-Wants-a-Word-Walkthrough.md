# TryHackMe: Management Wants a Word — Walkthrough

**Room:** Management Wants a Word (Hacker Holidays — Day 14, The Byte Lotus Hotel)
**Category:** Forensics — Windows / DPAPI / Cryptography
**Difficulty:** Hard

It was always her. It was never a bug; it was the business model.

---

## 1. Reading the Room Briefing First

> "Housekeeping found a guest's laptop left behind after an early checkout, Room 214, registered to a 'Vera.' IT pulled a full triage before wiping it for the next guest. Hunt down the artifacts scattered across her machine and figure out how they fit together. Somewhere in that trail is a password she never meant to leave behind. Follow it, and it'll open a door to something she was keeping very quiet."

Plus @0xMia's hint:

> "apparently a browser will remember things for you that you never told anyone else 💀 not every hidden file needs a password cracker, some of them just need a really good memory"

The key phrase is **"a really good memory, not a password cracker."** That's a direct pointer away from brute-forcing and toward **Windows DPAPI** (Data Protection API) — the mechanism Windows and Chrome use to encrypt saved credentials. DPAPI secrets aren't meant to be cracked; they're meant to be *unlocked* by walking a specific chain of artifacts that are all sitting right there on disk if you know where to look. This room is a pure offline forensics exercise — no live target machine, just a KAPE triage collection to investigate.

## 2. Getting Oriented in the KAPE Output

The provided ZIP extracts to a standard **KAPE** (Kroll Artifact Parser and Extractor) triage collection — a folder structure mirroring a live Windows filesystem, rooted at `KAPE/C/`:

```
KAPE/C/
├── Users/
│   ├── Default/
│   └── vera/
│       ├── AppData/
│       │   ├── Local/Google/Chrome For Testing/User Data/...
│       │   └── Roaming/Microsoft/Protect/S-1-5-21-.../
│       ├── Documents/backup
│       └── NTUSER.DAT
└── Windows/
    ├── ServiceProfiles/
    └── System32/
```

A quick `tree -a` over the `vera` profile shows a full Chrome user profile, a DPAPI "Protect" folder (this is the giveaway — that folder specifically stores DPAPI **masterkey** files, one per user, used to decrypt everything DPAPI-protects for that account), and a mysterious 100MB file at `Documents\backup` with no extension.

Deeper in `KAPE/C/Windows/System32/config/`, the classic Windows registry hives are present: `SAM`, `SYSTEM`, `SECURITY`. This is the full ingredient list for a DPAPI-decryption chain, laid out for us to assemble.

## 3. Understanding the DPAPI Chain Before Touching Anything

It's worth mapping the whole plan before running commands, since DPAPI decryption has several dependent stages:

1. **Registry hives (`SAM`/`SYSTEM`/`SECURITY`)** → can contain LSA Secrets, including things like stored service credentials or autologon passwords, in the `SECURITY` hive.
2. **A user's password or NTLM hash** → needed to decrypt that user's DPAPI **masterkey** file.
3. **The masterkey file itself** (`AppData\Roaming\Microsoft\Protect\<SID>\<GUID>`) → once decrypted, yields the raw AES key DPAPI actually uses.
4. **Chrome's `Local State` file** → contains an `os_crypt.encrypted_key` field: Chrome's own AES key, itself wrapped in a DPAPI blob, decryptable using the masterkey from step 3.
5. **Chrome's `Login Data` SQLite database** → contains saved logins; each `password_value` is encrypted with the AES key from step 4.

Every step depends on the one before it — this is exactly the "trail of artifacts scattered across her machine" the briefing describes.

## 4. Step 1 — LSA Secrets: Finding the Password Directly

Rather than immediately cracking a hash, the smarter first move is checking whether the `SECURITY` hive's **LSA Secrets** contain anything readable outright — Windows sometimes stores things like autologon passwords here in recoverable form, precisely so the OS itself can read them without prompting the user.

Using Impacket's `secretsdump.py` (pulled directly from Impacket's GitHub, since the standalone example scripts aren't always bundled with the distro package):
```bash
wget https://raw.githubusercontent.com/fortra/impacket/master/examples/secretsdump.py -O ~/secretsdump.py
python3 ~/secretsdump.py -sam SAM -system SYSTEM -security SECURITY LOCAL
```

The SAM hash extraction failed with a version-mismatch error against this particular Impacket install (a script/library API drift issue, not a data problem), but that turned out not to matter — the **LSA Secrets** section succeeded and immediately surfaced:
```
[*] DefaultPassword
(Unknown User):minivera
```

`DefaultPassword` is a genuine Windows registry value used for **autologon** configuration — when a machine is set to log a user in automatically at boot without a password prompt, the plaintext password has to be stored *somewhere* for Windows to use, and that somewhere is this LSA secret. This is "a really good memory" made completely literal: no cracking needed, the password (`minivera`) is stored in cleartext in the registry, and standard tooling reads it directly.

## 5. Step 2 — Decrypting the DPAPI Masterkey

With a plaintext password in hand, we can decrypt vera's DPAPI masterkey file directly — no NTLM hash needed at all, since DPAPI supports deriving the decryption key straight from a user's password + SID.

Using `dpapick3` (a Python library purpose-built for DPAPI artifact decryption):
```bash
pip install dpapick3 --break-system-packages
```

```python
from dpapick3 import masterkey

with open(masterkey_file_path, "rb") as f:
    mkf = masterkey.MasterKeyFile(f.read())

mkf.decryptWithPassword(sid, "minivera")

print(mkf.masterkey.key.hex())
```

(`sid` here is the folder name the masterkey file lives under — `S-1-5-21-2529683458-431225740-1723070931-1000` — which is vera's Windows user SID, always embedded in the DPAPI folder path for exactly this purpose.)

This decrypted successfully, producing the raw masterkey (a long hex string) — the actual cryptographic key DPAPI uses under the hood, now fully recovered.

**Small hiccup worth noting:** the first attempt called a `.get_key()` method that doesn't exist on this library version; inspecting the object with `dir(mkf.masterkey)` showed the real attribute is simply `.key`. When working with a library whose exact API you're unsure of, `dir()`/`help()` on the object is faster than guessing method names repeatedly.

## 6. Step 3 — Unwrapping Chrome's AES Key

Chrome's `Local State` file (JSON) contains an `os_crypt.encrypted_key` field — Chrome's actual encryption key, itself protected by DPAPI:
```bash
python3 -c "
import json
with open('Local State') as f:
    print(json.load(f)['os_crypt'])
"
```

The `encrypted_key` value is base64-encoded, and Chrome prepends a literal `DPAPI` ASCII prefix to it before the real DPAPI blob starts — that prefix has to be stripped first:

```python
import base64
from dpapick3 import blob

raw = base64.b64decode(encrypted_key_b64)
assert raw[:5] == b"DPAPI"
dpapi_blob = raw[5:]

b = blob.DPAPIBlob(dpapi_blob)
b.decrypt(masterkey=bytes.fromhex(masterkey_hex))

print(b.cleartext.hex())  # Chrome's actual AES key
```

This produced Chrome's real AES key in hex — the key that protects every saved password and cookie in this browser profile.

## 7. Step 4 — Decrypting the Saved Chrome Password

`Login Data` is a SQLite database; copying it out to a writable location first (SQLite can be finicky about locking on read-only source files) and querying the `logins` table:

```bash
cp "Login Data" /tmp/LoginData.db
sqlite3 /tmp/LoginData.db "SELECT origin_url, username_value, hex(password_value) FROM logins;"
```

```
http://bytelotus.thm:8080/|VeraSecretVault|763130C88A72A64F...
```

A saved login for a suggestively-named account, `VeraSecretVault`. Chrome encrypts `password_value` using **AES-256-GCM**, in a specific format: a 3-byte version prefix (`v10` or `v20`), a 12-byte nonce, the ciphertext, and a 16-byte authentication tag appended at the end:

```python
from Crypto.Cipher import AES

data = bytes.fromhex(encrypted_hex)
prefix, nonce, ciphertext, tag = data[:3], data[3:15], data[15:-16], data[-16:]

cipher = AES.new(key, AES.MODE_GCM, nonce=nonce)
plaintext = cipher.decrypt_and_verify(ciphertext, tag)
```

This decrypted cleanly (`v10` prefix, as expected for locally-encrypted Chrome data) to reveal the saved password: a purpose-built-looking string, clearly meant to be *the* password the briefing promised — "a password she never meant to leave behind."

## 8. Dead End — There's No Live Host

`bytelotus.thm:8080` looks like a real target worth attacking directly, and it was tempting to go hunting for an IP to point at it. But this room has **no live machine at all** — it's a pure static-artifact investigation. The URL is in-world flavor context only (it tells the story of what the password was originally *for*), not something reachable. The real next step is realizing the password must unlock something we **already have on disk** — and the one remaining unexplained artifact is the 100MB, header-less, high-entropy `Documents\backup` file spotted right at the start.

**Lesson worth keeping:** in a pure-artifact forensics room, any URL, hostname, or IP encountered in browser history/config files is scene-setting, not infrastructure to attack — the puzzle lives entirely in the files you've been given.

## 9. Step 5 — Opening the Vault

A 100MB file, pure random-looking bytes, no file signature, sitting in a folder called `Documents`, tied to a saved login literally named `VeraSecretVault` — strongly suggestive of an encrypted container, most likely **VeraCrypt** given the thematic naming ("Vera" the character, "Vault" in the credential name).

The dedicated `veracrypt` package wasn't available to install directly, but `cryptsetup` (standard on most Linux distributions, including Kali) has built-in support for mounting VeraCrypt volumes via its `tcrypt` mode — no separate VeraCrypt installation needed:

```bash
sudo cryptsetup open --type tcrypt --veracrypt "Documents/backup" vera_vault
# passphrase prompt: the decrypted Chrome password
```

```bash
sudo mkdir -p /mnt/vera_vault
sudo mount /dev/mapper/vera_vault /mnt/vera_vault
```

It mounted cleanly, revealing:
```
$RECYCLE.BIN/
secret_financial_documents/
  ├── important_invoice_byte_lotus.pdf
  └── transactions_q3.csv
System Volume Information/
```

## 10. Reading the Evidence

`transactions_q3.csv` lists a set of ordinary-looking transactions, except one line stands out immediately:
```
2026-07-12,TXN-10531,Internal Adjustment,Image asset correction,0.00,Archived
```

A $0.00 "image asset correction" is an obvious plant — it's telling us to look closely at an *image*, and specifically that the image has been "corrected" (tampered with) somehow.

`important_invoice_byte_lotus.pdf` initially looked empty via `pdftotext` — no extractable text — which is the signature of a PDF whose content is a **rendered image**, not real text, so it needs to be viewed visually rather than text-extracted:

```bash
pdftoppm -png -r 150 important_invoice_byte_lotus.pdf /tmp/invoice
```

Opening the rendered PNG directly showed the flag **visibly present in the invoice image itself** — hidden in plain sight within what looked like an ordinary financial document, tying directly back to the CSV's own hint about a tampered "image asset."

---

## Summary of the Chain

| Stage | Technique | Result |
|---|---|---|
| Recon | Recognized the KAPE folder layout and identified the DPAPI `Protect` folder, registry hives, and Chrome profile | Mapped out the full DPAPI decryption chain needed |
| LSA Secrets | `secretsdump.py` against `SAM`/`SYSTEM`/`SECURITY` | Recovered a plaintext `DefaultPassword` (`minivera`) — a Windows autologon credential, no cracking required |
| Masterkey | `dpapick3.masterkey.MasterKeyFile.decryptWithPassword(sid, password)` | Decrypted vera's DPAPI masterkey directly from her password + SID |
| Chrome key unwrap | Stripped the `DPAPI` prefix from `Local State`'s `os_crypt.encrypted_key`, decrypted the remaining blob with the masterkey | Recovered Chrome's real AES-256 encryption key |
| Saved password | Queried `Login Data` (SQLite) for `logins`, decrypted `password_value` with AES-256-GCM using the recovered key | Recovered a saved plaintext password for account `VeraSecretVault` |
| Dead end (recognized, not chased) | `bytelotus.thm:8080` in the saved login — no live host exists in this room | Correctly identified as flavor context, not an attack target |
| Container | `cryptsetup open --type tcrypt --veracrypt` on the unexplained `Documents\backup` file, using the recovered password | Mounted a VeraCrypt volume containing hidden financial documents |
| Payoff | A suspicious "image asset correction" line in a CSV pointed at a PDF invoice; rendered to PNG since it held no extractable text | Flag visible directly in the rendered invoice image |

**Key takeaway:** this room is a complete, realistic walkthrough of a Windows DPAPI credential-recovery chain — the exact technique real forensic investigators and red teamers use to recover saved browser credentials from a disk image, without ever needing a password cracker. Every stage depends on the one before it (registry secret → masterkey → Chrome key → saved password → encrypted container → hidden document), and the room rewards patiently following that dependency chain rather than attacking any single artifact in isolation. The closing twist — a "financial" document turning out to be the actual hiding place for the flag, flagged by a suspiciously-worded $0.00 line in an otherwise unremarkable spreadsheet — is a good reminder to read every artifact's *content* closely, not just its filename or type.

*(Flag intentionally omitted from this writeup — walk the chain yourself against your own copy of the artifacts to recover it.)*
