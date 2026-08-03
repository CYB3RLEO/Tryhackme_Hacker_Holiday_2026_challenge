# TryHackMe: Hacker Holidays — Day 06 Walkthrough

**Room:** Hacker Holidays, Day 06 (The Byte Lotus Hotel)
**Category:** OSINT
**Difficulty:** Easy–Medium

No target IP for this one — no nmap, no shells. Day 06 is a pure open-source intelligence challenge built around a leaked chat screenshot.

> ⚠️ Note: unlike the other write-ups in this repo, I no longer have the original screenshot or full room text on hand for this one, so this is written from memory of the solve path. The methodology below is accurate; treat exact wording of the in-room clue as approximate.

---

## 1. The Starting Point — A Chat Screenshot

The room provides a screenshot of a chat conversation between hotel guests/staff. One participant, in the course of the conversation, mentions that she uses a particular platform — one whose name starts with **"G"** — specifically to **scrub her presence off the web**. She adds that if anyone genuinely needs to reach her, they can do so at a specific **email address**, which she includes in the message.

That's the entire lead: a platform name starting with "G," described as a privacy/cleanup tool, plus a real email address.

## 2. Working Out the Platform

"Starts with G" plus "used to clean up your online footprint" points at a specific, real, and slightly counter-intuitive answer: **[Gravatar](https://gravatar.com/)**.

Gravatar (short for **Globally Recognized Avatar**) isn't a privacy-cleaning tool in the way it's being described in-character — the in-universe framing is a bit of misdirection — but it's a real, long-standing service (owned by Automattic/WordPress) that lets you attach a single global profile (avatar, bio, links) to your **email address**, which then follows you across any site that supports Gravatar integration (WordPress comments, forums, etc.).

The key mechanic that matters for OSINT purposes: **Gravatar identifies profiles by a hash of the associated email address**, not by a chosen username. That single detail is what makes this solvable — if you have someone's email, you can compute the same hash Gravatar uses and go straight to their profile, even if you don't know their display name or handle.

## 3. Finding the Profile

Gravatar profiles are reachable directly via a hash of the email address in the URL, e.g.:
```
https://gravatar.com/<hash>
```

Historically this hash was MD5; Gravatar's own documentation and site now also reference SHA-256 for newer profile URLs, so it's worth trying both if a straight lookup doesn't resolve. In practice, the fastest path is simpler and doesn't require computing anything by hand:

1. Go to **gravatar.com**.
2. Use the on-site profile lookup / search with the **exact email address** from the chat screenshot.
3. This resolves directly to the associated public profile — no manual hashing required, since Gravatar's own site does the lookup for you.

This is the step that resolves the "clean all her traces... except one corner of the internet" framing from the briefing: Gravatar profiles are usually not something people think of as "findable by email" the way a Twitter/Instagram handle is, which is exactly why it was overlooked in-universe.

## 4. The Profile — and a Base64 Surprise

The profile itself belongs to a persona named **"Lambo"** (displayed as *Lam-boh · Byte Lotus Hotel*), and includes a message directed squarely at whoever found the profile:

> "Funny thing about email hashes, they follow you places you didn't expect. Glad you found the right corner of the internet! Here is your prize: `VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0<redacted>M2R9`"

That long string is **Base64-encoded** — recognizable by the character set (uppercase, lowercase, digits, no unusual symbols) and the `=` padding conventions Base64 typically uses (though this particular string doesn't need padding).

Decode it with any standard tool:
```bash
echo "VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0<redacted>M2R9" | base64 -d
```

This resolves directly to the flag in TryHackMe's usual `THM{...}` format.

---

## Summary of the Chain

| Stage | Technique | Result |
|---|---|---|
| Lead | Chat screenshot mentions a "G" platform used for privacy + an email address | Identified the platform as **Gravatar** |
| Pivot | Gravatar profiles are keyed by a hash of the email address, not a username | Looked up the profile directly via Gravatar's own email search |
| Payoff | Profile bio contained a Base64-encoded string | Decoded with `base64 -d` → flag |

**Key takeaway:** this room is a good reminder that "cleaning your traces off the internet" and "using a service that indexes you by your email" are not the same thing — Gravatar's whole design is built around a *stable, cross-site identifier* derived from your email, which is the opposite of anonymity. The in-character mistake (using Gravatar as a "clean corner of the internet") is the entire vulnerability.

*(Flag intentionally omitted from this writeup — look the profile up yourself to retrieve it.)*
