# Hacker Holidays — TryHackMe Walkthroughs 🏖️

Walkthroughs for **Hacker Holidays**, TryHackMe's Byte Lotus Hotel-themed event series.

Each room ties into an overarching story at the Byte Lotus resort — a jukebox that takes a little more than song requests, a poolside portal that never forgets a session, a stranger who cleaned up their tracks everywhere except one corner of the internet. The rooms are solvable on their own, but reading the room briefing closely is almost always part of the puzzle.

## 📍 Where to find each day

| Day | Room | Category | Writeup |
|---|---|---|---|
| 1–4 | *(various)* | — | Posted on Medium: [@cyb3rleo](https://medium.com/@cyb3rleo) |
| 5 | Beach Bar | Boot2Root — Web / Deserialization | [`Day05-Beach-Bar-Walkthrough.md`](./Day05-Beach-Bar-Walkthrough.md) |
| 6 | *(OSINT room)* | OSINT | [`Day06-OSINT-Walkthrough.md`](./Day06-OSINT-Walkthrough.md) |
| 7 | Do Not Disturb | Boot2Root — Web | [`Day07-Do-Not-Disturb-Walkthrough.md`](./Day07-Do-Not-Disturb-Walkthrough.md) |
| 8 | Towel on the Sunbed | Web Exploitation — Business Logic Abuse | [`Day08-Towel-On-The-Sunbed-Walkthrough.md`](./Day08-Towel-On-The-Sunbed-Walkthrough.md) |
| 9 | CryptoCabana | ☁️ Cloud — Azure Storage / Key Vault | [`Day09-CryptoCabana-Walkthrough.md`](./Day09-CryptoCabana-Walkthrough.md) |
| 10 | The Hollow Shell | Web — Zip Slip / RCE | [`Day10-The-Hollow-Shell-Walkthrough.md`](./Day10-The-Hollow-Shell-Walkthrough.md) |
| 11 | Infinity Pool | Boot2Root — Internal Pivoting | [`Day11-Infinity-Pool-Walkthrough.md`](./Day11-Infinity-Pool-Walkthrough.md) |
| 12 | After Hours | Forensics — Windows / WMI / Reverse Engineering | [`Day12-After-Hours-Walkthrough.md`](./Day12-After-Hours-Walkthrough.md) |

Medium has been having formatting/publishing issues, so more recent days are being kept here on GitHub instead — this repo will keep growing as later days are solved.

## 🧭 How these writeups are structured

Each walkthrough tries to do more than list commands — it explains the *reasoning*:
- What the room briefing hints at, and how to read it
- Dead ends and red herrings that were tried first, and why they didn't pan out
- The actual vulnerability chain, explained step by step
- Full command breakdowns, not just copy-paste blocks

**Flags are intentionally redacted or omitted** in every writeup here. The goal is to teach the methodology, not hand out answers — run the steps yourself against your own instance to get the real flags.

## ⚠️ Disclaimer

These writeups are for **educational purposes only**, based on TryHackMe rooms designed for legal, sandboxed practice. Do not apply any of these techniques against systems you don't own or don't have explicit permission to test.

## 🙋 About

Written up by [@cyb3rleo](https://medium.com/@cyb3rleo). Feedback, corrections, and PRs welcome — if you spot a cleaner path through any of these boxes, open an issue.
