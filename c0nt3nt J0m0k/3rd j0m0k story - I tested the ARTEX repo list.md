# A Tool That Breached Korean Banks Got Pulled From GitHub. I Went and Tested Everything Left Behind.

*An audit of the ARTEX-named repository ecosystem - 251 repos collected, 9 flagged entries verified, one leaked production database, one pre-auth RCE, and a fork wave that refuses to stop.*

**(sudo3rs)** · November 10, 2026 · ~25 min read

`#ThreatIntelligence` `#ARTEX` `#AIPentest` `#SupplyChain` `#OSINT` `#GitHub`

---

This one started with a tasking list, not an instinct. A list of ARTEX-named repositories landed in my lap - the kind of list you get when you search GitHub for `artex` after the news breaks and export everything that comes back. ~130 entries, raw, no classification, no verdicts. Just names.

If you've been anywhere near threat intel since October, you know the name. ARTEX is the open-source agentic pentest platform a Chinese developer under the handle **Autumn-27** first published on **July 26, 2026** under AGPL-3.0 - the project that won Baidu's "agent+" attack-defense challenge, then spent late September punching through South Korean financial institutions before anyone fully understood what it was.

```
2026-07-26 ─ ARTEX first public release (Autumn-27, AGPL-3.0)
   │         "Baidu agent+ attack-defense challenge champion project"
   ▼
2026-08/09 ─ slow fork drip: 1 repo every few days
   ▼
2026-09-30+ ─ first breaches in the SK financial sector
   │         (loan-progress inquiry service + employee mobile
   │          work-support system as initial access vectors)
   ▼
2026-10-06 ─ WSJ: "Hackers Use Chinese AI Tool to Hit South Korean Banks"
   │         ~68,000 people's data exposed (Shinhan ~25k customers,
   │         Yegaram ~40k, KB Kookmin, Hana, BNK Busan)
   ▼
2026-10-07 ─ CrowdStrike blog: unattributed, financially motivated actor,
   │         Chinese speaker (moderate confidence) - "YY", @YY520CN
   ▼
2026-10-08 ─ Autumn-27 pulls the repo. Goes closed source.
   │         32 ARTEX-named repos created on GitHub THIS DAY alone
   ▼
2026-10-09 ─ 21 more. 91 in the first week (my dataset).
   ▼
2026-11-07 ─ GitGuardian: ARTEX family = 1,428 services on Shodan
   ▼
2026-11-10 ─ this audit: 1,032 name-matches, 251 collected,
              9 flagged entries re-verified one by one
```

The interesting part of that timeline isn't the top end. It's the bottom. The tool is gone from its origin, but the *name* is now a namespace - and namespaces on GitHub are searchable, rankable, and squattable. That's what this investigation is about: what actually lives inside the `artex` search results today, one month after the takedown.

> *"When a tool gets pulled, the threat doesn't shrink. It redistributes - into every repo that still carries the name."*

---

## 1. Background - What ARTEX Actually Is

For anyone coming in cold: ARTEX is not a model. It's an orchestration layer - a Go backend with an embedded Next.js frontend over PostgreSQL, driving multiple LLM agents that plan and execute a full penetration test chain autonomously: recon, fingerprinting, vulnerability matching, verification, exploitation, lateral movement, reporting. Human-in-the-loop approvals for sensitive operations. MITM traffic recording. An asset coverage graph. It calls external LLMs (Anthropic, OpenAI, whatever you point it at) and external tools, rather than being intelligent itself.

In the Korean bank intrusions, the operator ran it with **DeepSeek v4.1-flash** as the primary backend, supplemented by Zhipu's GLM-5.3 and xAI's Grok 4.6, with Claude Code used in a separate role - routing through roughly nine proxies, a Hong Kong server as primary infrastructure, and a second box at `38.244.50[.]120` hosting the ARTEX instance aimed at the Korean targets. The actor used an LLM API reseller, `xcai.pro`, to provision model access. AhnLab's Security Intelligence Center counted about **600 IPs worldwide hosting ARTEX instances** in the incident window.

One caveat that matters for intellectual honesty: the link between ARTEX and the bank breaches rests substantially on the operator's *own session logs*, per the CSA research note - not on independent victim-side confirmation. Keep that in mind when you read the more breathless coverage. The session logs also gave us the detail I can't stop thinking about: the operator asked Claude to **draft a security-researcher résumé describing the intrusions he had just committed**. Ambition, meet operational security.

Sources for this section, so you don't have to take my word for any of it: [CrowdStrike](https://www.crowdstrike.com/en-us/blog/unknown-threat-actor-uses-artex-to-target-south-korean-finance) (Oct 7), [WSJ](https://www.wsj.com/world/asia/hackers-use-chinese-ai-tool-to-hit-south-korean-banks-exposing-new-risk-5d4d3885) (Oct 6), [CSA research note](https://labs.cloudsecurityalliance.org/research/csa-research-note-artex-agentic-pentest-bank-breaches-202610) (Oct 9), [GitGuardian](https://blog.gitguardian.com/ai-pentesting-tools-beyond-artex) (Nov 9).

---

## 2. Scope & Method

Collection window: **2026-11-10, 15:20–15:32 UTC**. Everything below was re-verified live - no cached claims.

**Step 1 - enumeration.** Six GitHub search API queries, rate-limit spaced:

```
q=artex in:name                       (best match, page 1)   → total_count 1,032
q=artex in:name   sort=stars          (page 1)              → overlap merge
q=artex in:name,description           (page 1)              → 1,087
q=artex autonomous pentest            (context sweep)        → mirrors & translations
q=ARTEX mirror backup                 (lineage sweep)
q=artex in:name                       (page 2)
```

Merged and deduplicated by `full_name`: **251 unique repositories**, 11,467 combined stars, 11,413 combined forks. Yes - the fork count nearly equals the star count. Hold that thought; it comes back in §3.

**Step 2 - verification.** Every entry from the original tasking list that *didn't* show up in the search index got verified individually: `git ls-remote` for the refs, HTTP status on the web UI for the account/repo state. Empty-tree repos (0 commits) and single-commit repos were confirmed by cloning and reading the actual git history - because a 200 on the web UI tells you a repo exists, not that it contains anything.

**Step 3 - static analysis.** Shallow clones of every repo that had real content, scanned against 21 patterns (API keys, tokens, webhooks, eval/exec primitives, long base64, credential storage markers, raw.githubusercontent payload fetchers, pipe-to-shell). Pattern hits then read by hand in context - a regex says "here", a human says "what does this actually do".

```
  tasking list          GitHub Search API (6 queries)
      │                          │
      ▼                          ▼
  dedupe by full_name ──▶ 251 repo snapshot
      │                          │
      │                 git ls-remote + HTTP status
      │                 (entries absent from the index)
      ▼                          ▼
  9 flagged entries ──▶ verified one by one ◀── clone & read
      │                          │
      ▼                          ▼
  verdicts table        21-pattern static scan
      │                          │
      └──────────▶ this report
```

All IOCs below are defanged per repo convention. Leaked credentials are published truncated - the full values are sitting in public repos, go read them there; I'm not in the business of re-distributing working keys.

---

## 3. What the List Actually Looks Like

The creation-date histogram is the whole story in one picture. This is every repo in the snapshot, bucketed by the day its GitHub account created it:

```
2026-07-24 | #                                              1
2026-07-28 | #                                              1   ← tx1ee, 2 days post-release
2026-08-15 | #                                              1
2026-08-23 | #                                              1
2026-08-31 | #                                              1
2026-09-22 | #                                              1   ← panda0x123/artex-ymh
2026-09-24 | #                                              1
2026-10-02 | #                                              1
2026-10-03 | #                                              1
2026-10-06 | ##                                             2
2026-10-07 | #####                                          5
2026-10-08 | ################################            32   ← TAKEDOWN DAY
2026-10-09 | #####################                        21
2026-10-10 | ##########                                    10
2026-10-11 | #########                                      9
2026-10-12 | ########                                       8
2026-10-13 | ####                                           4
2026-10-14 | ####                                           4
2026-10-15 | ###                                           3
2026-10-16 | ###                                           3
2026-10-17 | ###                                           3
2026-10-18 | ###                                           3
2026-10-19 | #####                                          5
2026-10-20 | ###                                           3
2026-10-21 | ####                                           4
2026-10-22 | #                                              1
2026-10-23 | #########                                      9
2026-10-24 | ####                                           4
2026-10-25 | #                                              1
2026-10-27 | #                                              1
2026-10-28 | #                                              1
2026-10-29 | ##                                            2
2026-10-30 | ###                                           3
2026-10-31 | ###                                           3
2026-11-01 | #                                              1
2026-11-03 | #                                              1
2026-11-04 | ##                                            2
2026-11-05 | ##                                            2
2026-11-06 | ####                                           4
2026-11-07 | ###                                           3
2026-11-08 | #                                              1
2026-11-09 | #                                              1
2026-11-10 | #                                              1   ← failsafe-security/artex, today
```

32 repos on takedown day. 91 in the first week. GitGuardian counted 102 forks *plus* ARTEX-named repos in that same window across their vantage point. And a month later it still hasn't stopped - one to four new repos a day, every day, including the morning I wrote this.

Classification of the 251-repo snapshot:

```
"artex" search namespace - 1,032 matches, 251 collected
│
├── ARTEX orbit ─────────────── 129  (51%)
│   ├── lineage / mirror ──────  27  (源码备份, "backup of v0.3.15",
│   │                                mirrors via mhtsec, offline archives)
│   └── 2nd-dev / translation ─ 102  (EN/KO/zh-TW ports, reworks, forks
│                                     of forks, "one-click getshell" builds)
├── name-collateral ──────────── 84  (33%)
│   └── games, agencies, contract repos, an Ethereum engine, a 2025
│       web-appsec platform that already owned the name, a 2014 Java
│       project, Roblox frameworks, SQLite tooling
├── unclassified ──────────────── 38  (15%)
│   └── empty descriptions, mostly Go, created post-07-26 -
│       almost certainly silent mirrors; nothing to classify *from*
└── tasking-list squats ────────── 9  (verified individually, §4)
```

Two structural observations before the deep dives.

**The fork line is top-heavy and weird.** The major distribution lines:

| Repo | Stars | Forks | Created |
|---|---:|---:|---|
| `jiwoochris/artex-global` | 3,026 | 2,775 | 2026-10-03 |
| `Hinln/ARTEX` | 1,471 | 1,995 | 2026-10-08 |
| `tx1ee/ARTEX` | 6 | 978 | 2026-07-28 |
| `RuoJi6/ARTEX` | 1,550 | 918 | 2026-10-09 |
| `cskwork/scopeweaver` | 956 | 766 | 2026-10-02 |
| `BaaRai/ARTEX` | 1,961 | 751 | 2026-10-08 |
| `imxv/ARTEXX` | 1 | 628 | 2026-10-09 |
| `panda0x123/artex-ymh` | 81 | 562 | 2026-09-22 |
| `TengHu/autonomous-pentest-agent` | 0 | 422 | 2026-10-08 |
| `bdg9412/artex-ko-plus` | 267 | 244 | 2026-10-09 |

`tx1ee/ARTEX`: **6 stars, 978 forks.** `imxv/ARTEXX`: 1 star, 628 forks. `TengHu/autonomous-pentest-agent`: 0 stars, 422 forks. That inversion - forks massively outpacing stars - is the deployment signature. People don't fork these repos to contribute; they fork to *run* them. Every fork line is a probable deployment, and every deployment is a console somebody has to secure. GitGuardian's Shodan work says a lot of people don't: 1,428 services in the ARTEX family answering on the open internet as of Nov 7, many unauthenticated.

**The name has generations.** `IlyassCODEX/ARTEX` - 260 stars, created **2025-07-16** - is a completely different, older project: an AI-driven *web application* security analysis platform. It owned the ARTEX name a full year before Autumn-27's tool existed. Further back: `VitorPeixoto/ArtexSavior` (2014, Java), `va109/Artex` (2017, smart contracts), `nodeGame/artex` (2019, an art exhibition game). When you search `artex` today you are searching a multi-generational name collision, and ~33% of what you get back has zero relationship to the tool that made the name dangerous.

> *"A fork wave is a deployment census. Read the fork counts and you're reading how many copies of the thing are probably running somewhere."*

---

## 4. The Flagged Entries - Verified One by One

The tasking list came with entries that looked off. Every one of them got the same treatment: `git ls-remote`, HTTP status, clone, read the history. Verdicts:

| Entry | State (2026-11-10) | Verdict |
|---|---|---|
| `iTzKwenx/ArtexBOT` | 200, 1 commit | **Name-squat.** Entire repo, verbatim, complete: `# ArtexBOT` - 10 bytes, one heading. Nothing else, ever. |
| `rushmills/ArtexSync-Dashboard` | 200, **0 commits** | **Placeholder squat.** No branches, no history, no description. Exists to own the name. |
| `sametkaracan18/ArtexBOT` | 200, **0 commits** | **Placeholder squat.** Description carries a link to `alfanzok.glitch.me` - which serves HTTP **410 Gone**. Empty repo, dead lure link. |
| `ExrionLaChaplin/artex-altyapi` | 200, **0 commits** | **Placeholder squat.** "bot altyapı" (bot base) - nothing in it. |
| `ArtexPycak/ArtexDDoS` | 200, **0 commits** | **Name-squat.** The account is alive; the repo is a shell. No DDoS code ever landed in it. |
| `jweny/ZDA` | 200, **0 commits** | Exists, empty. Listed in the tasking snapshot; no content to assess. |
| `zw99sbm554-commits/ARTEX` | **404** | **Gone.** A `-commits`-suffix account (the mirror-account pattern GitHub accumulates); both of its ARTEX repos have vanished while the account itself stays up (200). Classic churn pattern. |
| `zw99sbm554-commits/artex` | **404** | Same. |
| `iTzKwenx/ArtexSync-Dashboard` / `ArtexPycak/ArtexBOT` | **404** | Never resolved under these owners - the surviving twins under `rushmills`/`ArtexPycak` (above) are what the names actually pointed at. |

That table is the quiet finding of this whole investigation. Not one flagged entry contained malware. They're *worse than that in a way*: they're proof that the ARTEX name is now trusted enough to squat on and cheap enough to squat on with an empty repo. Five of nine are zero-commit shells. The DDoS-branded one has no code at all. The `-commits` mirror account deleted its copies the moment they'd served whatever purpose they had.

Nobody plants empty repos named after an infamy-of-the-month for fun. They do it because search ranking is inventory, and inventory gets filled later - or the repo exists only long enough to be indexed, then gets dropped. That's the lifecycle visible in the 404s.

---

## 5. Deep Dive A - `vlady-slav/-ArtexBot-`: A Stray Telegram Token Riding a Hot Name

The one repo in the flagged set with actual code. `main.py`, 209 lines, Python, aiogram (Telegram bot framework). A to-do list / reminder bot with dice rolls and support-ticket forwarding.

Riding the name: confirmed - it has nothing to do with ARTEX the pentest platform, or even with "Artex" as any brand. The name was free, the code was lying around, the two got introduced.

The problem is what's hardcoded in the middle of it:

```python
bot = Bot(token='6627524866:AAFEXPYuJRwRs2c2B_CYeYHsBgIWVgLaBqM')
...
OWNER_CHAT_ID = '5956033204'
```

A live-format Telegram Bot API token and the owner's chat ID, committed in plaintext. Everything users type into the "Support" flow - their messages, their usernames, their IDs - gets forwarded to `5956033204`. And anyone who clones this repo runs a bot whose token is also in *their* copy, which means anyone can hijack the bot out from under them: receive its messages, send as it, drain its groups.

Verdict: **not malware, but a live credential leak with a support-tap attached.** The token is defanged-truncated in the IOC table below; the full value is where I found it.

---

## 6. Deep Dive B - `afwanaditya/RoboArtex-WhatsAppBot`: The One That Actually Leaks People

This is the finding I'd flag highest in the whole set, and it has nothing to do with pentest tooling - which is exactly the point of testing a *name list* instead of a *tool list*.

`RoboArtex-WhatsAppBot` presents as an Indonesian WhatsApp multi-function bot ("RoboArtex" branding). Under the hood it's a republish of the `neoxr/neoxr-bot` base (`package.json` still says `"nolep-bot 2.2.2"`, author field still says Wildan Izzudin, repository field still points at `neoxr/neoxr-bot`) - a Baileys-based bot with a plugin system, and an `>eval` owner command that does `eval()` on arbitrary text plus `execSync('git pull')` for updates. Risky-by-design, standard for this family.

What it shipped *with* is the issue. Two files, `database.bak` and `database.json` - identical, 17,104,991 bytes each - a production database, committed to a public repo:

```
database.bak (17 MB, JSON) ── afwanaditya/RoboArtex-WhatsAppBot
│
├── users[]      36,849 records (36,641 carry +62 = Indonesian MSISDNs)
│    jid, afk, banned, limit, premium, point, lastseen, hit
│    e.g. 628952xxxxxxx@s[.]whatsapp[.]net (masked - these are real
│    people's phone numbers)
├── chats[]       3,536 records (jid, chat counts, lastseen, command ts)
├── groups[]        231 records (group jids, antidelete/antilink settings)
├── statistic{}  257,175 total command hits (stiker/ping/api/yt/tt/ig/qc…)
├── setting{}     runtime config - owners, prefixes, chatbot flags
└── sticker/captcha/bots{}
```

36,849 user records - real WhatsApp JIDs (which *are* phone numbers), ban status, premium flags, points, activity timestamps - plus the group graph of 231 communities and per-chat metadata. That's a 36.8k-person PII leak sitting in a repo whose only relationship to "Artex" is the letters in its name. Nobody republishing a bot base should ever ship the production DB with it; whoever ran this instance had 36.8k users and committed their identities to GitHub.

And the credentials bundled in the same tree:

| Secret | Location | Note |
|---|---|---|
| OpenAI key `sk-uaHAOk…Vq4h` | `lib/dalle.js` | hardcoded, image-gen plugin |
| OpenAI key `sk-Wrvn8n…bon9` | `plugins/special/ai.js` | hardcoded, AI chat plugin |
| MongoDB Atlas URI `mongodb+srv://neoxrbot:•••@cluster0[.]kontol[.]mongodb[.]net/…` | config | live-format creds, cluster name is operator's choice |
| Telegram bot token `891038791:AAHW…uzPD4` | `plugins/download/stele.js` | sticker-downloader plugin |
| trace.moe `access_secret` `1C8eU…` | `plugins/utilities/whatanime.js` | anime scene search |

Five live-format credentials, one 17MB population database, and an `>eval` primitive - published under an Artex-branded name, in the exact part of the search results where a user looking for "ARTEX" tooling lands when they scroll far enough. That's the supply-chain risk of a hot name in one exhibit: you're not testing what the repo *says* it is, you're testing what it *ships*.

Verdict: **PII leak + credential bundle, name-riding.** If you have users in Indonesia in your dataset, the JID patterns above are what the leak looks like - the masked format is deliberate; I won't republish 36k phone numbers to make a point.

---

## 7. Deep Dive C - `therustymate/ARTXploit`: The Tool That Breached Banks Is Itself Breachable

The most technically interesting entry in the entire list isn't a fork. It's an exploit - for ARTEX itself.

`ARTXploit` is a public, working pre-auth RCE against **ARTEX v0.3.14** (commit `b55ceb1fdd84a813d77de09a06af83d323a81f85`), documented with a full reproduction environment (KVM, Ubuntu 26.04.1, PostgreSQL 18.6). The chain:

```
attacker ──(flood /api/auth/status, ~10k concurrent)──▶ ARTEX v0.3.14
    │
    ▼
PostgreSQL max_connections (100 default) exhausted
    │
    ▼
auth.go's DB connection FAILS
    │
    ▼  …and the failure is handled as
       "no password has been configured yet"        ← FAIL-OPEN
    │
    ▼
accept + store a new password → re-trigger /setup
    │
    ▼
chain into /api/tools/custom/test
    │
    ▼
RCE as the service user
```

The root cause is that the auth logic treats *database unavailability* as *unconfigured state*, and the door swings open. The race window is timing-sensitive - the author is explicit that success varies with hardware and load - but it works, it's pre-auth, and there's a YouTube demo. It was fixed in a later upstream commit, which means: **every fork that split off before the fix, and every deployment still running v0.3.14-era code, is an unauthenticated RCE target.**

Now put that next to the fork census from §3 - 978-fork lines, deployment-signature inversions, 1,428 ARTEX-family services visible on Shodan - and the exposure picture inverts. The population running ARTEX consoles isn't just a threat vector. It's a *victim* population: pre-auth RCE, default web UI on `:8787`, and a culture of fork-to-deploy with no patch discipline. The demo instance referenced in the exploit README, `artex-demo[.]vercel[.]app`, was still serving when I checked (HTTP 307, edge cache age ~24 days).

> *"The tool that walked itself into bank networks ships with a fail-open front door. Every unpatched console is both an attack platform and a sitting target - usually at the same IP."*

---

## 8. The Threat Model of a 1,000-Repo Namespace

Step back and assemble it. What can an attacker actually do with the state of `artex` on GitHub today?

```
                    ┌────────────────────────────────────┐
                    │  search: artex → 1,032 repos       │
                    └────────────────┬───────────────────┘
                                     │
     ┌───────────────┬───────────────┼────────────────┬─────────────────┐
     ▼               ▼               ▼                ▼                 ▼
 LEGIT LINEAGE   2ND-DEV FAMILY   COLLATERAL      SQUATS         DEAD NAMES
 mirrors of      translations,    33% noise -     empty shells   404'd after
 the pulled       reworks,        dilutes the     that own        being indexed
 upstream -       "getshell"      search results  ranking for     (commits-
 trusted by       builds, EN/     for defenders   later fill-     account pattern)
 association      KO ports                          in
     │               │               │                │                 │
     │               │               │                └────────┬────────┘
     │               │               │                         ▼
     │               │               │              T1195.002 - the namespace
     │               │               │              becomes supply-chain
     │               │               │              terrain: deploy-from-
     │               │               │              search-result is the norm
     ▼               ▼               ▼
 the ARTEX-family consoles themselves
 (1,428 on Shodan · many unauthenticated · pre-auth RCE pre-fix)
     │
     ▼
 attacker lands on either side of the console:
 as OPERATOR (provision an agent platform cheaply -
              the CrowdStrike intrusion model), or
 as INTRUDER (ARTXploit the unpatched console).
```

The specific mechanics an operator gets from this namespace, all verified in this investigation:

1. **Search poisoning by volume.** With 1,032 name-matches, anything an attacker publishes is 1 entry among a crowd - and 33% of the crowd is unrelated noise that further buries defender triage.
2. **Zero-commit inventory.** Empty repos that own ranking positions (`ArtexBOT`, `ArtexSync-Dashboard`, `ArtexDDoS`, `artex-altyapi`). Content can be pushed at any chosen moment, or the repo exists purely to be indexed and vanish (§4's 404s).
3. **Credential bundles in branded wrappers.** The RoboArtex exhibit: an Artex-named repo that ships 36.8k users' data and five live-format secrets to whoever clones it (§6).
4. **Weaponized trust by lineage.** `TengHu/autonomous-pentest-agent` - 422 forks, 0 stars, created on takedown day, described as a "private mirror… via mhtsec/ARTEX backup." Fork-to-deploy means operators inherit other people's code review, or the lack of it. `wpsec/ARTEX-local`'s description literally opens with a claim that the project was audited and contains no backdoor - that's the kind of assurance a namespace like this manufactures on demand.

MITRE mapping - only what the evidence here actually supports:

```
T1195.002  Compromise Software Supply Chain   republished bases, bundled secrets,
                                            squat-deploy from search results
T1036.005   Masquerading: Match Legitimate    ArtexBOT/ArtexSync/RoboArtex -
            Name or Location                  names chosen to ride ARTEX
T1190      Exploit Public-Facing Application  ARTXploit chain vs unpatched
                                            ARTEX consoles (v0.3.14 era)
T1005 / T1539  Data from Local System /       the RoboArtex DB - 36,849 user
            Steal Web Session Cookie          records, exposed (if harvested)
T1583.003  Acquire Infrastructure: VPS        operator infrastructure per
T1588.007  Obtain Capabilities: AI            CrowdStrike (cite, not my data)
T1090      Connection Proxy                   9-proxy routing, HK + 38.244.50[.]120
```

---

## 9. Defensive Recommendations

**If you run an ARTEX-family console** (any lineage - RuoJi6, Hinln, artex-global, scopeweaver, artex-ymh, the whole census):

1. **Know your build's commit.** If it predates the fail-open fix, you are running a pre-auth RCE. Rebuild from a post-fix commit; verify against a known-good lineage (Hinln v0.3.15-era or RuoJi6's maintained line) by diff, not by README claims.
2. **Bind loopback.** Default `127.0.0.1:8787` exists for a reason. If you must expose it, put it behind an authenticated reverse proxy - and know that "the setup token protects first-run" is not an auth model for day 30.
3. **Alert on the flood signature.** ARTXploit's precondition is a request storm against `/api/auth/status`. Rate-spike alerting on that endpoint is your tripwire: it fires before the race resolves, which is the only time you get to react.
4. **Treat the console as production infrastructure.** It executes tools, holds LLM API keys, records MITM traffic of everything it tests. Its compromise is full-chain: platform + model spend + captured traffic.

**If you're triaging the namespace** (SOC / threat intel):

5. **Monitor for the squat pattern, not the malware.** The artifacts to watch are zero-commit repos matching infamy-of-the-month names, `-commits`-suffix accounts whose repos vanish, and description-borne links to dead hosting (the `alfanzok.glitch.me` 410 pattern). Those are *pre-loading* indicators; the payload comes later or never.
6. **Don't trust names; trust lineage.** In a 1,000-repo namespace, the unit of analysis is the commit graph. Check what a "translation" actually changed before you mirror it - a real localization touches strings; a tampered one touches `auth`, `tools`, or anything that runs at startup.
7. **Sweep your egress for the console signature.** GitGuardian's fingerprinting queries are public; the 1,428-service count on Nov 7 is a baseline that will drift upward. Fingerprint ports, `Server` headers, and the `/api/auth/status` shape.

**If you publish bot bases / templates** (the §6 lesson, generalized):

8. **The repo is the distribution.** Strip databases, session states, and `.bak` files before publishing. `database.bak` should never appear in a public tree - 17MB of committed production data is how "open source" becomes "doxxing with extra steps."

---

## 10. IOC Master List

All defanged per repo convention. Credential values truncated; full values remain in the referenced public repos.

| Type | Indicator | Context |
|---|---|---|
| Repo (squat, 1 commit) | `iTzKwenx/ArtexBOT` | entire content: `# ArtexBOT` (10 bytes) |
| Repo (squat, 0 commits) | `rushmills/ArtexSync-Dashboard` | placeholder |
| Repo (squat, 0 commits) | `sametkaracan18/ArtexBOT` | desc links dead Glitch page |
| Repo (squat, 0 commits) | `ExrionLaChaplin/artex-altyapi` | "bot altyapı" placeholder |
| Repo (squat, 0 commits) | `ArtexPycak/ArtexDDoS` | empty; account active |
| Repo (empty) | `jweny/ZDA` | exists, 0 commits |
| Repo (deleted) | `zw99sbm554-commits/ARTEX`, `zw99sbm554-commits/artex` | 404; account survives |
| Repo (leak) | `afwanaditya/RoboArtex-WhatsAppBot` | neoxr base + production DB |
| Repo (exploit) | `therustymate/ARTXploit` | pre-auth RCE PoC, v0.3.14 |
| Account (mirror-bot pattern) | `zw99sbm554-commits` | `-commits` suffix, repos purged |
| Telegram token | `6627524866:AAFE…aBqM` | vlady-slav/-ArtexBot-, plaintext |
| Chat ID | `5956033204` | same repo, support-tap recipient |
| OpenAI key | `sk-uaHAOk…Vq4h` | RoboArtex, `lib/dalle.js` |
| OpenAI key | `sk-Wrvn8n…bon9` | RoboArtex, `plugins/special/ai.js` |
| MongoDB URI | `mongodb+srv://neoxrbot:•••@cluster0[.]kontol[.]mongodb[].net` | RoboArtex config |
| Telegram token | `891038791:AAHW…uzPD4` | RoboArtex, `stele.js` |
| API secret | `1C8eU…` (trace.moe) | RoboArtex, `whatanime.js` |
| Commit | `b55ceb1fdd84a813d77de09a06af83d323a81f85` | ARTEX v0.3.14 - fail-open era |
| Commit | `c791d9c134e7…` | iTzKwenx/ArtexBOT "Initial commit" |
| Commit | `6567459` | RoboArtex "Update neoxrApi.js" |
| URL (live console) | `artex-demo[.]vercel[.]app` | HTTP 307, cache age ~24d |
| URL (counter badge) | `profile-counter[.]glitch[.]me/{neoxr}/count.svg` | RoboArtex README |
| URL (dead lure) | `alfanzok[.]glitch[.]me` | HTTP 410 Gone |
| JID pattern | `62xxxxxxxxxxx@s[.]whatsapp[.]net` ×36,849 | RoboArtex `database.bak` |
| IP (ARTEX instance) | `38.244.50[.]120` | CrowdStrike, SK campaign |
| IPs (proxy pool ×9) | `101.53.80[.]20`, `205.214.59[.]31`, `124.155.252[.]63`, `154.201.79[.]246`, `23.248.249[.]90`, `23.158.220[.]98`, `103.248.148[.]84`, `203.160.133[.]172`, `209.209.85[.]38` | CrowdStrike, SK campaign |
| Telegram (actor) | `@YY520CN` | CrowdStrike attribution |
| LLM reseller | `xcai.pro` | CrowdStrike, SK campaign |

---

## 11. The Full List - Appendix A

Every repository in the snapshot, 251 entries, sorted by class then name. Collected 2026-11-10; classes are heuristic (rules stated in §2–§3), "unclassified" = nothing to classify from (empty description, Go, post-release creation - the silent-mirror profile).

<details>
<summary><strong>Open the full 251-repo table</strong></summary>

| Repo | Created | ★ | Forks | Class | Description |
|---|---|---:|---:|---|---|
| `a01632401/artex` | 2019-11-27 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `Abhishekbabu052/Artex` | 2026-04-09 | 0 | 0 | name-collateral (unrelated) | Artex is an Android app that allows users to schedule tasks with a specific date and time, and receive noti... |
| `admin1i/https---apps.facebook.com-artexas--by_` | 2013-08-05 | 5 | 3 | name-collateral (unrelated) | Apache License                            Version 2.0, January 2004                         http://www.apac... |
| `agunggm/Artex` | 2018-07-24 | 0 | 0 | name-collateral (unrelated) | Augmented Reality Live Texturing |
| `AlexandraMol/artex` | 2022-03-06 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `AmirhosseinAzimyzadeh/Artex` | 2019-11-06 | 1 | 0 | name-collateral (unrelated) | create and edit 3D Object based on .Obj (wavefront) in java |
| `Anindya91/artext` | 2015-09-12 | 2 | 0 | name-collateral (unrelated) | Article Extractor |
| `anishsuwal/artex` | 2022-06-27 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `AntonioSilva00/artex` | 2024-02-08 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `artex-agency/artex-debug` | 2025-02-03 | 1 | 0 | name-collateral (unrelated) | A lightweight, extensible PHP debugging tool with error handling, performance profiling, and a powerful UI ... |
| `artex-is/ALRM-BusinessCentral` | 2020-11-24 | 2 | 0 | name-collateral (unrelated) | *(no description)* |
| `artex-is/ALRM-VSCode` | 2020-11-24 | 2 | 0 | name-collateral (unrelated) | *(no description)* |
| `Artex012/Artex` | 2023-04-14 | 0 | 0 | name-collateral (unrelated) | Artex |
| `artexcargo/artex` | 2018-12-17 | 0 | 0 | name-collateral (unrelated) | database |
| `Artexxx/Artexxx.github.io` | 2023-12-30 | 8 | 0 | name-collateral (unrelated) | https://artexxx.github.io/ |
| `artexyc/artexyc` | 2024-11-18 | 1 | 2 | name-collateral (unrelated) | Config files for my GitHub profile. |
| `basit-ali/Artex` | 2011-08-14 | 0 | 0 | name-collateral (unrelated) | Financial Website |
| `boothxll/artex` | 2025-05-18 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `Clobera/Artexico` | 2023-03-23 | 1 | 0 | name-collateral (unrelated) | *(no description)* |
| `Cloudy-Development/artex-core` | 2024-08-15 | 5 | 0 | name-collateral (unrelated) | Just an incomplete server core for Spigot 1.8 |
| `CODAME/artex-open` | 2026-03-22 | 3 | 2 | name-collateral (unrelated) | *(no description)* |
| `codeReleaser/Artex` | 2020-08-05 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `davipatti/artEx` | 2014-09-19 | 0 | 0 | name-collateral (unrelated) | Artificial extinction scripts |
| `deikike-ux/ARTEX` | 2025-11-28 | 0 | 0 | name-collateral (unrelated) | Mi primer proyecto, con este nuevo codigo |
| `dhkamp/artex` | 2023-05-24 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `Dimitrof04/ArtexDesktopApps` | 2026-08-23 | 1 | 0 | name-collateral (unrelated) | Apps From Artex Desktop (all) |
| `Dimitrof04/ArtexDesktopHyprland` | 2026-06-19 | 1 | 0 | name-collateral (unrelated) | *(no description)* |
| `Diogo0602x/Artex` | 2022-06-08 | 2 | 0 | name-collateral (unrelated) | Site para a marca Artex Creative Studio |
| `enlianzer/Artex` | 2017-03-13 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `fandiks32/artex` | 2015-06-13 | 0 | 2 | name-collateral (unrelated) | Augmented Reality Text Translation |
| `fgaim/artext` | 2018-09-29 | 6 | 0 | name-collateral (unrelated) | Probabilistic Noising of Natural Language |
| `G340G/artex` | 2025-03-21 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `Gabbasov98/artex` | 2024-05-12 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `Garfild13-06/Artex` | 2025-03-17 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `hsusueusnjejeje-creator/artex` | 2026-01-18 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `humanai-foundation/ArtExtract` | 2024-07-21 | 12 | 14 | name-collateral (unrelated) | *(no description)* |
| `IlyassCODEX/ARTEX` | 2025-07-16 | 260 | 152 | name-collateral (unrelated) | ARTEX is an advanced web application security analysis platform that automates reconnaissance, vulnerabilit... |
| `irongirl101/artex` | 2026-07-08 | 1 | 1 | name-collateral (unrelated) | art exchange app/website or smth idk yet |
| `iTzKwenx/Artex` | 2020-05-09 | 0 | 0 | name-collateral (unrelated) | Artex Moderasyon Botu |
| `jluo117/ARTextBook` | 2019-09-07 | 2 | 0 | name-collateral (unrelated) | *(no description)* |
| `jluo117/ARTextBook3.0` | 2019-09-08 | 1 | 0 | name-collateral (unrelated) | *(no description)* |
| `JMencius/Artex` | 2024-09-14 | 2 | 1 | name-collateral (unrelated) | Extension of the Artex pipeline |
| `Kagiso31/artex` | 2024-01-05 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `kdh3543/ArtexVision` | 2021-12-09 | 1 | 3 | name-collateral (unrelated) | *(no description)* |
| `lailamariel-01/ARTEX` | 2024-10-29 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `Le-ArteX/MeetGenius` | 2026-04-24 | 2 | 0 | name-collateral (unrelated) | MeetGenius eliminates manual note-taking by using AI to capture and structure meeting conversations in real... |
| `LuisAP/Artex` | 2018-10-10 | 0 | 0 | name-collateral (unrelated) | Proyecto artex |
| `madneal/artext` | 2016-09-27 | 1 | 0 | name-collateral (unrelated) | It is the the reading notes of the method of programming |
| `mantranit/artex` | 2015-09-01 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `markzhong88/ARText` | 2017-08-28 | 81 | 19 | name-collateral (unrelated) | ARText by ARKit |
| `martin-lentschat/ARTEXT4LOD` | 2020-05-12 | 1 | 0 | name-collateral (unrelated) | *(no description)* |
| `maxappdevelopement/artex` | 2019-06-09 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `medban/artex` | 2025-03-13 | 0 | 0 | name-collateral (unrelated) | sté textile |
| `Midohajhouj/ArtExt` | 2025-03-19 | 5 | 2 | name-collateral (unrelated) | ArtExt is a Python-based command-line tool that allows you to generate customizable ASCII art banners |
| `Muhammadsolih-xacer/ARTEX` | 2024-05-07 | 0 | 0 | name-collateral (unrelated) | aaaaaaaaaa |
| `natsukisubarujmk-create/artex` | 2026-03-12 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `njligames/ARTextsMVP` | 2017-09-25 | 1 | 1 | name-collateral (unrelated) | *(no description)* |
| `nnuvi/artex` | 2025-08-29 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `nodeGame/artex` | 2019-10-28 | 0 | 0 | name-collateral (unrelated) | Art Exhibition Game |
| `NotKisoMomo/ArTex` | 2026-05-09 | 1 | 0 | name-collateral (unrelated) | ArTex is a full-stack game framework for Roblox. Define your services, managers, and controllers once -- Ar... |
| `Othernet-Project/artexin` | 2014-06-11 | 8 | 2 | name-collateral (unrelated) | Article Extraction and Indexing for Outernet |
| `piumsudhara/Artex` | 2018-06-09 | 0 | 0 | name-collateral (unrelated) | A Software for Service Center Created using Java |
| `plasticgaming99/artex` | 2023-11-08 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `racytech/artex` | 2025-12-03 | 1 | 0 | name-collateral (unrelated) | A single-threaded, in-memory Ethereum execution engine written in C |
| `RajKumar210mmxx/artex` | 2024-04-21 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `Rmaain/artex` | 2025-01-15 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `S-Harabara/Artex` | 2019-08-20 | 0 | 0 | name-collateral (unrelated) | Artex Manufacturing |
| `saadakhtr29/Artex-Tech` | 2025-03-20 | 4 | 1 | name-collateral (unrelated) | Artex Tech builds scalable web, mobile, AI, and Web3 solutions. We help startups and enterprises design, bu... |
| `sambytemail-max/artex` | 2026-07-24 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `SamuelAkintomide/ARTEx` | 2025-02-12 | 0 | 0 | name-collateral (unrelated) | Final Year Project |
| `ScottKjr3347/Short_Queries_ArtEx-OtherTools` | 2025-03-09 | 3 | 0 | name-collateral (unrelated) | Short Photos.sqlite queries for quick use in ArtEx and other SQLite tools |
| `SeherD/Artex` | 2021-11-16 | 0 | 1 | name-collateral (unrelated) | *(no description)* |
| `shakty/artex` | 2016-03-24 | 0 | 1 | name-collateral (unrelated) | Art Exhibition Game |
| `STARIONGROUP/Artex` | 2026-03-05 | 0 | 0 | name-collateral (unrelated) | simple package repository that supports nuget, maven and kpar |
| `tien198/artex` | 2026-06-23 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `va109/Artex` | 2017-09-13 | 0 | 0 | name-collateral (unrelated) | Artex smart-contracts |
| `valerie-butron97/ARTEX` | 2019-10-18 | 1 | 0 | name-collateral (unrelated) | *(no description)* |
| `VercelGitt/Artex` | 2025-04-10 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `VitorPeixoto/ArtexSavior` | 2014-04-18 | 1 | 0 | name-collateral (unrelated) | Project ArtexSavior Repository |
| `vnmslf/artex` | 2024-05-09 | 0 | 0 | name-collateral (unrelated) | artex official |
| `waelarous/Artex` | 2021-06-04 | 0 | 0 | name-collateral (unrelated) | *(no description)* |
| `wheelywonder/Artex` | 2016-12-12 | 0 | 0 | name-collateral (unrelated) | Part of my BSc Honours Project, an application which artificially textures a touchscreen and mimics the tex... |
| `yonas44/Artex-conference-website` | 2022-10-17 | 7 | 0 | name-collateral (unrelated) | A conference website to display event program, featured speakers along with date of the event. Built with H... |
| `zestlark/artex` | 2023-11-11 | 0 | 1 | name-collateral (unrelated) | https://zestlark.github.io/artex/  Artex is a futuristic webOS design that maximizes efficiency, allowing y... |
| `01NGVI-QINGYI/ARIA-X` | 2026-10-23 | 6 | 4 | fork family / 2nd-dev | Local-first state-space reasoning engine for security research. Fuses ARTEX, Cairn & PentAGI. Python, zero ... |
| `551763510/artex` | 2026-11-07 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `AFOGSHEEP/artex-zh` | 2026-10-18 | 2 | 1 | fork family / 2nd-dev | ARTEX 中文版 -- LLM 多智能体自主渗透测试系统（jiwoochris/artex-ko 的中文发行，附防御与检测资料） |
| `alkjd/ARTEX` | 2026-10-08 | 1 | 0 | fork family / 2nd-dev | *(no description)* |
| `AntiFox-AI/antifox` | 2026-10-09 | 331 | 181 | fork family / 2nd-dev | AI 自主渗透测试平台 · 多智能体自主渗透测试（二次开发自 ARTEX） |
| `aqoonraadiye-arch/ARTEX-en` | 2026-10-31 | 0 | 1 | fork family / 2nd-dev | ARTEX autonomous pentest system - full English fork (private) |
| `Ariember/ARTEX` | 2026-10-16 | 3 | 3 | fork family / 2nd-dev | AI 自主渗透测试系统（Go + Next.js），由 fraternity-z 独立维护，保留上游历史与 AGPL-3.0 许可。 |
| `azrielnan/artex-ko-main` | 2026-10-15 | 111 | 5 | fork family / 2nd-dev | 面向已获授权安全测试与本地研究的 AI 多 Agent 安全测试平台。 |
| `BaaRai/ARTEX` | 2026-10-08 | 1961 | 751 | fork family / 2nd-dev | ARTEX - AI autonomous penetration testing system (English translation) |
| `bdg9412/artex-ko-plus` | 2026-10-09 | 267 | 244 | fork family / 2nd-dev | ARTEX 한국어판 ·AI 기반 침해사고 대응 기능 추가 (upstream: Autumn-27/ARTEX, AGPL-3.0) |
| `blitzcolo/ARTEX` | 2026-10-08 | 0 | 1 | fork family / 2nd-dev | ARTEX - AI autonomous pentest system, English UI build (upstream: Autumn-27/ARTEX) |
| `bluechips-zhao/DITING` | 2026-10-11 | 122 | 17 | fork family / 2nd-dev | 谛听 DITING：LLM 驱动的自主渗透测试工作台，Go + Next.js + PostgreSQL，基于 ARTEX（AGPL-3.0） |
| `bookSiYi1412/ARTEX` | 2026-10-12 | 21 | 16 | fork family / 2nd-dev | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `buuzzy/ARTEX` | 2026-10-20 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `bXZb/ARTEX` | 2026-10-10 | 0 | 1 | fork family / 2nd-dev | *(no description)* |
| `chesskuo/ARTEX` | 2026-10-11 | 1 | 3 | fork family / 2nd-dev | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 ### special thanks: https://github.com/mhtsec/ARTEX ### |
| `chrgu000/artex` | 2026-10-08 | 2 | 1 | fork family / 2nd-dev | ARTEX：默认中文的自主渗透测试系统（由韩文版还原） |
| `Clever-rain/dsh-artex-plugin` | 2026-10-21 | 2 | 1 | fork family / 2nd-dev | ARTEX-style dual-graph planner/worker orchestration for DeepSeek Harness (DSH plugin) |
| `cnlnn/ARTEX` | 2026-10-08 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `Comet0721/artex` | 2026-10-19 | 0 | 0 | fork family / 2nd-dev | ARTEX 0.3.15 |
| `cskwork/scopeweaver` | 2026-10-02 | 956 | 766 | fork family / 2nd-dev | ScopeWeaver: English and Korean localization of ARTEX, with preserved AGPL-3.0 attribution. |
| `D4rk-3ch0/artex-c` | 2026-10-15 | 9 | 5 | fork family / 2nd-dev | AI 自主渗透测试系统，基于 ARTEX v0.3.15 二次开发。保留原有主体框架，增强蜜罐识别、出口审查、范围强制（RoE）与反测绘门控，引入思维链提示词框架。AGPL-3.0 · 仅限授权环境使用 |
| `DFveloper/artex` | 2026-10-11 | 0 | 0 | fork family / 2nd-dev | ARTEX · AI 자율 침투 테스트 프레임워크 |
| `doraemonbd903/ARTEX` | 2026-10-23 | 0 | 0 | fork family / 2nd-dev | ARTEX - AI autonomous penetration testing system (English translation) |
| `dunvu0/ARTEX` | 2026-10-29 | 0 | 0 | fork family / 2nd-dev | RuoJi6/ARTEX, but english |
| `dx86683-lab/ARTEX` | 2026-11-06 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `Ernket/CodeScan-Artex` | 2026-10-14 | 293 | 154 | fork family / 2nd-dev | 二开artex的代码审计平台 / 新增本地反编译java依赖库 / 多账号管理 |
| `Error336070/ARTEX` | 2026-10-11 | 1 | 2 | fork family / 2nd-dev | The ARTEX agentic penetration testing framework, ported to English with extra modifications planned for fur... |
| `failsafe-security/artex` | 2026-11-10 | 67 | 0 | fork family / 2nd-dev | Source-built, locally bound FailSafe distribution of ARTEX (v0.3.15) |
| `FelipeR-UFBA/ARTEX` | 2026-10-24 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `fightingFirefox/ARTEX` | 2026-10-08 | 0 | 0 | fork family / 2nd-dev | back-artex |
| `froiibn/ARTEX-Desktop` | 2026-10-14 | 3 | 0 | fork family / 2nd-dev | AI-assisted penetration testing desktop app that helps users find vulnerabilities and improve the security ... |
| `futureboop/artex` | 2026-11-04 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `guidie/ARTEX` | 2026-10-12 | 0 | 1 | fork family / 2nd-dev | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `hater-us/Artex-update` | 2026-08-31 | 3 | 0 | fork family / 2nd-dev | 黑板架构的自动化渗透Agent |
| `hctinno/ARTEX` | 2026-10-07 | 0 | 0 | fork family / 2nd-dev | artex-ko |
| `hmruu/ARTEX-Magisk` | 2026-10-20 | 4 | 0 | fork family / 2nd-dev | 一个在手机运行ARTEX的方案 |
| `horizonl/ARTEX` | 2026-10-08 | 41 | 96 | fork family / 2nd-dev | ARTEX - AI autonomous pentest system, English UI build (upstream: Autumn-27/ARTEX) |
| `hunmeng2012/artex-hm` | 2026-11-04 | 10 | 3 | fork family / 2nd-dev | ARTEX 0.3.15 二开版 / 拒答治理 · 内网渗透功能页 · 攻击方法论技能包 · 渐变主题 / 交流群 1109964876 · 破甲模型 www.hunmeng.icu / AGPL-3.0 / 仅限... |
| `HuuDoe/artex-fork` | 2026-10-30 | 0 | 0 | fork family / 2nd-dev | ARTEX autonomous pentesting fork - CloakBrowser, Devin SWE-2/Windsurf integration, traffic capture fixes |
| `IcebergN2026/artex-ld` | 2026-10-27 | 36 | 34 | fork family / 2nd-dev | artex 流程优化及落地，适配内网渗透/审核intent/漏洞挖掘/代码审计等多场景，可自定义编排。 |
| `ijlalhaider313/ARTEX` | 2026-10-11 | 0 | 0 | fork family / 2nd-dev | ARTEX - autonomous pentest system (English translation, vendored localized deps, SpecterOps pentest skills ... |
| `ILoveScratch2/ARTEX` | 2026-10-08 | 0 | 0 | fork family / 2nd-dev | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `imxv/ARTEXX` | 2026-10-09 | 1 | 628 | fork family / 2nd-dev | Autumn-27/ARTEX |
| `infinity-avenir/ARTEX` | 2026-10-25 | 0 | 0 | fork family / 2nd-dev | Autumn-27/ARTEX 持续维护开源版本 |
| `ivthnight/Artex` | 2026-10-23 | 0 | 0 | fork family / 2nd-dev | Artex but in more languages |
| `Jason-Guillaume/ARTEX` | 2026-10-23 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `javaht/ARTEX` | 2026-10-09 | 0 | 0 | fork family / 2nd-dev | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `jaykwok/ARTEX` | 2026-10-19 | 45 | 31 | fork family / 2nd-dev | ARTEX ：AI 自主渗透测试系统（Go + Next.js），基于 Autumn-27/ARTEX v0.3.15（e6ec569，AGPL-3.0）。原版 v0.3.15 下载说明与恢复指引见 README。 |
| `jicine1360-prog/roy-vaccine-artex-shield` | 2026-11-06 | 0 | 0 | fork family / 2nd-dev | Roy Vaccine - ARTEX Shield: 3-layer defense against autonomous AI pentest agents (MIT + AGPL components). B... |
| `jiwoochris/artex-global` | 2026-10-03 | 3026 | 2775 | fork family / 2nd-dev | Global edition of ARTEX - an LLM multi-agent autonomous penetration-testing framework. English default, swi... |
| `Josehoho/ARTEX` | 2026-11-05 | 1 | 0 | fork family / 2nd-dev | AI渗透开源项目ARTEX |
| `lkokyong-lab/ARTEX` | 2026-10-08 | 0 | 0 | fork family / 2nd-dev | ARTEX is an advanced web application security analysis platform that automates reconnaissance, vulnerabilit... |
| `LLLLLLiulei/ARTEX` | 2026-10-30 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `logkiller/artex-deep-hunter` | 2026-10-24 | 1 | 1 | fork family / 2nd-dev | ARTEX Deep Hunter v0.4 - autonomous multi-agent pentest framework: hunt-class close-out gate, campaign bran... |
| `luojiaping/ARTEX` | 2026-10-11 | 0 | 0 | fork family / 2nd-dev | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `luoxi0615/ARTEX` | 2026-10-17 | 2 | 0 | fork family / 2nd-dev | *(no description)* |
| `milantgh/artex` | 2026-10-23 | 0 | 0 | fork family / 2nd-dev | add cve detection to artex |
| `Min313/ARTEX` | 2026-10-19 | 0 | 0 | fork family / 2nd-dev | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `MMhh001/ARTEX` | 2026-10-23 | 1 | 0 | fork family / 2nd-dev | *(no description)* |
| `MoeclubM/ARTEX` | 2026-10-16 | 0 | 0 | fork family / 2nd-dev | AI 自主渗透测试系统 |
| `MoLeft/ARTEX` | 2026-10-31 | 3 | 1 | fork family / 2nd-dev | Autumn-27/ARTEX 持续维护开源版本 |
| `momo121138/ARTEX` | 2026-10-24 | 1 | 0 | fork family / 2nd-dev | AI渗透开源项目ARTEX |
| `neronain/ARTEX` | 2026-10-09 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `nina27owo/ARTEX-Controller` | 2026-10-28 | 2 | 0 | fork family / 2nd-dev | 本地多机 ARTEX 控制台：SSH 隧道统一管理多台目标机，支持任务下发、暂停/继续、提问主 Agent 与报告导出 |
| `Nixon-H/ARTEX-en` | 2026-10-12 | 0 | 1 | fork family / 2nd-dev | ARTEX AI autonomous pentest system - full English translation (AGPL-3.0; translated from mhtsec/ARTEX, see ... |
| `OpenWizzy/ARTEX` | 2026-10-11 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `panda0x123/artex-ymh` | 2026-09-22 | 81 | 562 | fork family / 2nd-dev | ARTEX 二次开发版:外网突破→内网纵深全流程自主渗透平台(上游 Autumn-27/ARTEX,AGPL-3.0) |
| `ParkSnoopy/ARTEX` | 2026-10-12 | 0 | 0 | fork family / 2nd-dev | ARTEX from Autumn-27/ARTEX |
| `pfgq/artex` | 2026-10-21 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `r1k0t3k1/ARTEX` | 2026-11-07 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `RC-CHN/ARTEX` | 2026-10-09 | 9 | 5 | fork family / 2nd-dev | *(no description)* |
| `rizkrisk/KSIR-ARTEX` | 2026-10-29 | 0 | 3 | fork family / 2nd-dev | ARTEX - AI autonomous pentesting system. Fork melhorado: 34 skills, CVE database 2.223 entradas, Cloudflare... |
| `RuoJi6/ARTEX` | 2026-10-09 | 1550 | 918 | fork family / 2nd-dev | Autumn-27/ARTEX 持续维护开源版本 |
| `sebastian93921/ARTIFEX` | 2026-10-09 | 35 | 56 | fork family / 2nd-dev | English and Traditional Chinese version of enhanced ARTEX |
| `segfo/artex` | 2026-10-20 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `shantanu561993/ARTEX` | 2026-11-06 | 1 | 1 | fork family / 2nd-dev | *(no description)* |
| `sharenjun/ARTEX-main` | 2026-10-08 | 158 | 79 | fork family / 2nd-dev | 可以提交issue和优化思路，会维护 |
| `shingo-katashi/artex` | 2026-10-07 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `shiwei404/ARTEX` | 2026-11-05 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `Solihov-Y/ARTEX` | 2026-10-08 | 1 | 0 | fork family / 2nd-dev | AI-powered security testing platform |
| `spierenburg/ARTEX` | 2026-11-07 | 0 | 0 | fork family / 2nd-dev | ARTEX - AI autonomous pentest system, English UI build (upstream: Autumn-27/ARTEX) |
| `TATHPE/ARTEX-Endfield` | 2026-10-13 | 7 | 1 | fork family / 2nd-dev | AI 自主渗透测试系统（Go 后端 + Next.js 前端）· 终末地（Arknights: Endfield）主题版 现已整合REA |
| `Team-intN18-SoybeanSeclab/prtseek` | 2026-11-09 | 35 | 15 | fork family / 2nd-dev | 基于ARTEX的CTF Agent |
| `therustymate/ARTXploit` | 2026-10-06 | 6 | 7 | fork family / 2nd-dev | ARTEX v0.3.14 - Pre-Auth Remote Code Execution via Fail-Open Authentication |
| `trongkyKMA/ARTEX-en` | 2026-10-30 | 0 | 0 | fork family / 2nd-dev | ARTEX autonomous pentest system - full English fork (private) |
| `trycatch219/ARTEX2` | 2026-10-21 | 1 | 2 | fork family / 2nd-dev | *(no description)* |
| `unvbld/ARTEX` | 2026-10-09 | 1 | 0 | fork family / 2nd-dev | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `vitamin1123/ARTEX` | 2026-10-09 | 0 | 0 | fork family / 2nd-dev | ARTEX project |
| `vv-get-ti/ARTEX` | 2026-10-07 | 2 | 2 | fork family / 2nd-dev | ARTEX 私有副本；上游：https://github.com/Autumn-27/ARTEX |
| `wangzunxiang/ARTEX` | 2026-10-22 | 0 | 0 | fork family / 2nd-dev | ARTEX -- 授权渗透测试智能体平台（私有） |
| `webb-w/ARTEX` | 2026-10-14 | 0 | 0 | fork family / 2nd-dev | *(no description)* |
| `willsurpass/WSARTEX` | 2026-10-17 | 63 | 43 | fork family / 2nd-dev | 基于 ARTEX 的 AI 自主渗透测试系统，支持多 Agent 协作、资产探索与漏洞管理，提供本地部署面板。 |
| `wnduddld0513/ARTEX` | 2026-10-18 | 0 | 0 | fork family / 2nd-dev | ARTEX，AI 自主渗透测试系统。百度 agent+ 攻防挑战赛冠军项目。 |
| `wufufu770/reArtex` | 2026-10-12 | 16 | 13 | fork family / 2nd-dev | 对Artex的二次开发 |
| `Xiao1an/ARTEX-God` | 2026-10-21 | 88 | 77 | fork family / 2nd-dev | ARTEX二开增强版-优化敏感操作拦截，增加一键getshell、长链路后渗透、内网横向、逻辑漏洞增强、卫星等IOT协议测试适配等... |
| `xsywl945/artex-ymh-mirror` | 2026-10-17 | 2 | 3 | fork family / 2nd-dev | ARTEX 二次开发版:外网突破→内网纵深全流程自主渗透平台(上游 Autumn-27/ARTEX,AGPL-3.0) |
| `Y0neQnine/ARTEX` | 2026-10-23 | 2 | 1 | fork family / 2nd-dev | ARTEX-二开 |
| `yhaolc/ARTEX` | 2026-10-08 | 0 | 5 | fork family / 2nd-dev | Korean reference fork of Autumn-27/ARTEX |
| `YoungUsing/ARTEX` | 2026-10-08 | 1 | 1 | fork family / 2nd-dev | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `zhuzhuzihan/ARTEX` | 2026-10-09 | 0 | 0 | fork family / 2nd-dev | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `zseek/ARTEX` | 2026-10-09 | 0 | 1 | fork family / 2nd-dev | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `200210xsb/artex-full` | 2026-11-03 | 30 | 17 | lineage/backup | ARTEX 自用衍生版（非官方）：AI 自主渗透测试系统，衍生自 ARTEX（原作者 Autumn-27，AGPL-3.0） |
| `3185263343/ARTEX` | 2026-10-08 | 11 | 13 | lineage/backup | AI 自主渗透测试系统（mhtsec/ARTEX 备份）/ Go + Next.js |
| `acheong08/artex` | 2026-10-09 | 5 | 6 | lineage/backup | A mirror of the now deleted Chinese pentesting tool used against korea |
| `ating2634-boop/ARTEX-0.4.1-research` | 2026-11-08 | 2 | 0 | lineage/backup | Research archive of the ARTEX 0.4.1 Linux amd64 release package |
| `bear20252026/ARTEX-release` | 2026-10-10 | 5 | 1 | lineage/backup | ARTEX v0.3.12 precompiled binaries (prebuilt releases) - built from bear20252026/ARTEX (source: mhtsec/ARTE... |
| `Core-RCE/ARTEX-original-version` | 2026-10-13 | 11 | 7 | lineage/backup | ARTEX 官方闭源前的开源源码备份，保留原始代码，未经二次开发，遵循 AGPL-3.0 开源协议 / Archive of the original open-source ARTEX codebase befo... |
| `daofaziran1/ARTEX` | 2026-10-09 | 0 | 0 | lineage/backup | ARTEX 源码备份：基于 Autumn-27/ARTEX v0.3.15，保留原始提交历史与 AGPL-3.0 许可证。 |
| `EncSpark/ARTEX` | 2026-10-23 | 0 | 0 | lineage/backup | ARTEX 源码备份：基于 Autumn-27/ARTEX v0.3.15，保留原始提交历史与 AGPL-3.0 许可证。 |
| `fanfan20001206/ARTEX` | 2026-10-13 | 2 | 1 | lineage/backup | ARTEX backup |
| `furs0x/ARTEX` | 2026-10-24 | 0 | 0 | lineage/backup | ARTEX - AI autonomous penetration testing system (unofficial community archive, based on v0.3.15) |
| `Hinln/ARTEX` | 2026-10-08 | 1471 | 1995 | lineage/backup | ARTEX 源码备份：基于 Autumn-27/ARTEX v0.3.15，保留原始提交历史与 AGPL-3.0 许可证。 |
| `jacklilyhello/ARTEX` | 2026-10-08 | 7 | 1 | lineage/backup | ARTEX 源码备份：基于 Autumn-27/ARTEX v0.3.15，保留原始提交历史与 AGPL-3.0 许可证。 |
| `Mai-xiyu/ARTEX` | 2026-10-10 | 0 | 0 | lineage/backup | ARTEX 源码备份：基于 Autumn-27/ARTEX v0.3.15，保留原始提交历史与 AGPL-3.0 许可证。 |
| `nekofinley/ARTEX` | 2026-10-19 | 0 | 0 | lineage/backup | ARTEX 源码镜像：基于 Hinln/ARTEX (Autumn-27/ARTEX) v0.3.15，保留原始提交历史与 AGPL-3.0 许可证。 |
| `nu11b1ade/ARTEX` | 2026-10-31 | 0 | 0 | lineage/backup | ARTEX 二次开发版本：基于 RuoJi6/ARTEX v0.3.25 迁移，Norma 使用 nu11b1ade/Norma v0.4.3，Docker 镜像 nullblade/artex。 |
| `orangetangerine/ARTEX` | 2026-10-10 | 0 | 0 | lineage/backup | ARTEX 源码备份：基于 Autumn-27/ARTEX v0.3.15，保留原始提交历史与 AGPL-3.0 许可证。 |
| `QioBerDev/ARTEX` | 2026-10-13 | 0 | 0 | lineage/backup | ARTEX - AI自主渗透测试系统，个人学习备份仓库 |
| `RayMorTwinkle/ART-bak` | 2026-10-10 | 0 | 0 | lineage/backup | ARTEX 最终版纯源码备份 - LLM 多 Agent 驱动的自主渗透测试系统（Go + Next.js + PostgreSQL，双图架构 · MITM 流量留痕 · 人在环路拦截审批）。Final sourc... |
| `recall704/ARTEX` | 2026-10-09 | 0 | 0 | lineage/backup | ARTEX 源码备份：基于 Autumn-27/ARTEX v0.3.15，保留原始提交历史与 AGPL-3.0 许可证。 |
| `richard1230/artex-backup` | 2026-10-08 | 4 | 0 | lineage/backup | ARTEX v0.3.14 离线备份（上游 Autumn-27/ARTEX 已于 2026-10 从 GitHub 删除） |
| `s1916/autonomous-pentest-agent` | 2026-10-09 | 0 | 1 | lineage/backup | Private mirror of ARTEX (upstream Autumn-27/ARTEX, via mhtsec/ARTEX backup); AGPL-3.0 |
| `TengHu/autonomous-pentest-agent` | 2026-10-08 | 0 | 422 | lineage/backup | Private mirror of ARTEX (upstream Autumn-27/ARTEX, via mhtsec/ARTEX backup); AGPL-3.0 |
| `thalysonz/ARTEX` | 2026-10-10 | 2 | 1 | lineage/backup | ARTEX 源码备份：基于 Autumn-27/ARTEX v0.3.15，保留原始提交历史与 AGPL-3.0 许可证。 |
| `wpsec/ARTEX-local` | 2026-10-08 | 5 | 8 | lineage/backup | （经审计本项目不存在后门-main分支保持不变，只修复漏洞-迭代版分支为：artex-iteration）AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目  作者已停止维护项目；本仓库保存最后一版... |
| `xc179/ARTEX` | 2026-11-06 | 0 | 0 | lineage/backup | ARTEX 源码备份：基于 Autumn-27/ARTEX v0.3.15，保留原始提交历史与 AGPL-3.0 许可证。 |
| `XXXXXQ-0206/focus-agent` | 2026-10-12 | 38 | 29 | lineage/backup | 基于 Autumn-27/ARTEX 的 AGPL-3.0 自主安全测试平台：planner/worker、多图态势、范围审批、会话与内网编排；仅供源码学习和本地隔离验证。两套前端：类 Codex 风格客户端（开发... |
| `yes-chen/ARTEX` | 2026-10-08 | 14 | 23 | lineage/backup | ARTEX 源码（非 fork 镜像）：上游 Autumn-27/ARTEX 已删除，源码取自 Hinln/ARTEX v0.3.15，AGPL-3.0 |
| `abdo6400/artex` | 2026-10-09 | 0 | 0 | unclassified | *(no description)* |
| `ADENH/artex-skill` | 2026-10-10 | 0 | 0 | unclassified | Agent Skill that distils an autonomous penetration-testing methodology into a portable prompt+file package:... |
| `chainsawteam/ARTEX` | 2026-10-08 | 1 | 2 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `correctree/artex` | 2026-08-15 | 0 | 0 | unclassified | *(no description)* |
| `D3-3109/ARTEX` | 2026-10-08 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `dandelion1234/ARTEX` | 2026-10-11 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `dlytwx/ARTEX` | 2026-10-09 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `dolapo0090/Artex` | 2026-10-19 | 0 | 0 | unclassified | my new launched |
| `fanky357/ARTEX` | 2026-10-09 | 0 | 0 | unclassified | *(no description)* |
| `FoxHMM/ARTEX` | 2026-10-08 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `grubwithu/ARTEX` | 2026-10-08 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `HAMAN23333/ARTEX` | 2026-10-08 | 1 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `ioriyameng/ARTEX` | 2026-10-23 | 0 | 0 | unclassified | a faithful preservation of history and a continuation (｡•̀ᴗ-)✧ |
| `Kilnui/ARTEX` | 2026-10-07 | 1 | 5 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `kishwordulal2005/ARTEX-original-eng` | 2026-11-01 | 0 | 0 | unclassified | Autonomous Pentest Agent - LLM multi-agent autonomous penetration testing system (Go backend + Next.js UI).... |
| `LangfordKuo/ARTEX` | 2026-10-15 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `Linstone114514/ARTEX` | 2026-10-08 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `lxy1044/ARTEX` | 2026-10-08 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `LyCecilion/ARTEX` | 2026-10-08 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `mathusuthan/ARTEX` | 2026-10-06 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `misakano7545/ARTEX` | 2026-10-08 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `MooMeACow/ARTEX` | 2026-10-07 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `palopao/ArteX` | 2026-09-24 | 0 | 0 | unclassified | *(no description)* |
| `ptintwin/artex` | 2026-10-11 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `PurrfectlyFine/ARTEX` | 2026-10-12 | 0 | 0 | unclassified | AI 自主渗透测试系统 |
| `python-watchdog/ARTEX` | 2026-10-09 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `singualow/ARTEX` | 2026-10-16 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `snowfishgo/artex-en` | 2026-10-08 | 6 | 145 | unclassified | *(no description)* |
| `ttkkaq/ARTEX` | 2026-10-12 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `tx1ee/ARTEX` | 2026-07-28 | 6 | 978 | unclassified | AI 自主渗透测试系统 |
| `User-krbtgt/ARTEX` | 2026-10-08 | 0 | 2 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `wangjingtiankl/ARTEX` | 2026-10-08 | 4 | 14 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `wehnes/ARTEX` | 2026-10-10 | 0 | 0 | unclassified | *(no description)* |
| `xxmdd/ARTEX` | 2026-10-14 | 0 | 0 | unclassified | *(no description)* |
| `yji0728/ARTEX` | 2026-10-10 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `ymmmmmmmm/ARTEX` | 2026-10-10 | 0 | 0 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |
| `znaukitai/ArtexAI_BadPiggies` | 2026-10-18 | 1 | 0 | unclassified | *(no description)* |
| `ztx888/ARTEX` | 2026-10-09 | 8 | 11 | unclassified | AI 自主渗透测试系统 / 百度“agent+”攻防挑战赛冠军项目 |

</details>

---

## Closing

The tasking asked me to *test a list*. The honest result is that the list stopped being the interesting part about a third of the way in. The list is a census - and censuses tell you about populations.

Three populations live in the `artex` namespace now. The **operators**, fork-to-deploying an agentic attack platform by the hundred, most of whom will never read a line of the Go they're running, some fraction of whom are a `/setup` token away from being someone else's RCE. The **squatters**, who've figured out that a hot name plus a zero-commit repo costs nothing and owns a search result - and who delete faster than anyone can index them. And the **collateral**, a third of the namespace: art exhibitions, Ethereum engines, a 2014 Java project, a Roblox framework - all of them now one scroll away from a tool that breached banks, in a search result that makes no distinction between them.

And somewhere outside the namespace there's still the operator who walked into financial institutions with a rented agent, a proxy pool, and a résumé he'd asked his other assistant to write. The session logs gave him up. The résumé draft is the detail I keep coming back to - he wanted a *document* of it. He got one. Just not the kind he meant.

The forks are still being created at one to four a day. I'll re-run the census when the curve flattens - or when the next name goes hot, whichever comes first.

- (sudo3rs)
 | November 10, 2026

---

`#ThreatIntelligence` `#ARTEX` `#AIPentest` `#SupplyChain` `#OSINT` `#GitHub` `#MITRE` `#CyberSecurity` `#ThreatHunting`
