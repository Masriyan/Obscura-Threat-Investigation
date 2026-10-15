# Eight APK Paths, Seven Indonesian Banks: Following Hirudo's Patched Banking App Trail

*An investigation into Hirudo's Android banking RAT infrastructure, a CloudFront distribution hosting bank-specific APK routes, and why replacing the real banking app is more dangerous than simply drawing a fake login screen.*

**(sudo3rs)** · October 15, 2026 · ~23 min read

`#ThreatIntelligence` `#AndroidMalware` `#Hirudo` `#BankingTrojan` `#MalwareAnalysis` `#Indonesia`

---

It started with a path that looked almost too ordinary:

```text
/livin/latest.apk
```

On its own, that path tells you very little. `latest.apk` is a generic filename. But the surrounding infrastructure was not generic. The path appeared in a set of extracted strings associated with a CloudFront distribution that ThreatFabric had just identified as part of **Hirudo**, an Android banking remote-access trojan (RAT). Their October 14, 2026 research described **eight purpose-built replacement banking applications belonging to seven Indonesian banks**. It did not publicly name those seven banks. [1]

The strings in the investigation material contained eight separate download routes. One pointed to `livin`. Another to `mybca`. Then `btn`, `bni`, `bri`, `bsi`, `jak`, and an oddly misspelled `bcamoblie`.

That spelling mistake was interesting. The count was even more interesting.

I mapped the route names against official Indonesian banking products and the infrastructure described in ThreatFabric's research. The result was a strong, internally consistent target reconstruction: **Mandiri, BTN, BCA, BNI, BRI, BSI, and Bank Jakarta**, with two routes apparently belonging to BCA.

But the bigger story is the malware's approach. Hirudo does not have to stop at putting a fake login page over a legitimate banking application. For selected targets, it tries to **uninstall the real bank app and install a modified copy that can lie to its own security controls**.

> *A familiar banking icon is not proof of a genuine banking application. When the application itself has been modified, even its security checks may be answering the wrong questions.*

**Important scope note:** This is a reconstruction of *targeting*, based on supplied extracted strings, publicly documented infrastructure, vendor reverse engineering, and official banking-product references. It is **not evidence that any of the seven banks' backend systems were breached**, nor does it establish the number of infected customers. The eight bank-specific binaries were not individually retrieved and verified as part of this write-up.

---

## 1. The Original Finding: Eight Paths on One Distribution

ThreatFabric's report describes a registry containing **42 financial applications**, but a smaller subset of **eight Indonesian apps across seven banks** has dedicated malicious replacement APKs. It explicitly identifies the following Amazon CloudFront distribution as the host for that replacement set. [1]

```text
d3q4tlm7pw29zr[.]cloudfront[.]net
```

The extracted material supplied for this investigation contained these eight paths:

```text
d3q4tlm7pw29zr[.]cloudfront[.]net/livin/latest[.]apk
d3q4tlm7pw29zr[.]cloudfront[.]net/btn/latest[.]apk
d3q4tlm7pw29zr[.]cloudfront[.]net/mybca/latest[.]apk
d3q4tlm7pw29zr[.]cloudfront[.]net/bcamoblie/latest[.]apk
d3q4tlm7pw29zr[.]cloudfront[.]net/bni/latest[.]apk
d3q4tlm7pw29zr[.]cloudfront[.]net/bri/latest[.]apk
d3q4tlm7pw29zr[.]cloudfront[.]net/bsi/latest[.]apk
d3q4tlm7pw29zr[.]cloudfront[.]net/jak/latest[.]apk
```

The shared host is the important link to the vendor's findings. The directory names then offer potential clues to the target applications.

There are limits to that inference. A path named `/bni/` identifies an intended bank much more confidently than it identifies an exact APK version. Attackers can name files however they want. They can also change the object stored behind `latest.apk` without changing the URL.

And a failed download attempt for the exact `/livin/latest.apk` object means I cannot use that object to claim an independently calculated hash, signing certificate, package ID, or binary diff here.

**What can be said:** the infrastructure matches a vendor-documented malicious distribution, and its route labels align remarkably well with the reported eight-application/seven-bank count.

---

## 2. Reconstructing the Seven Indonesian Banking Targets

The route-to-bank mapping is straightforward in several cases, but it is worth separating **bank-level identification** from **identification of a specific app build**.

| Download route | Bank indicated by the route | Likely targeted banking app | Attribution assessment |
|---|---|---|---|
| `/livin/latest.apk` | Bank Mandiri | Livin' by Mandiri | Very high for bank and app branding |
| `/btn/latest.apk` | Bank Tabungan Negara | balé by BTN / BTN Mobile lineage | High for bank; exact build unverified |
| `/mybca/latest.apk` | Bank Central Asia | myBCA | Very high for bank and app branding |
| `/bcamoblie/latest.apk` | Bank Central Asia | BCA mobile | Very high for bank and app branding; route contains a typo |
| `/bni/latest.apk` | Bank Negara Indonesia | Likely wondr by BNI | High for bank; app build unverified |
| `/bri/latest.apk` | Bank Rakyat Indonesia | Likely BRImo | High for bank; app build unverified |
| `/bsi/latest.apk` | Bank Syariah Indonesia | BYOND by BSI or legacy BSI Mobile | High for bank; exact app uncertain |
| `/jak/latest.apk` | Bank Jakarta | Likely JakOne Mobile | High for bank and product ecosystem |

The attribution levels above reflect how strongly the **route names** match known products. They do **not** measure confidence that each specific binary was obtained, executed, or forensically identified.

### Mandiri: `/livin/`

`livin` strongly matches **Livin' by Mandiri**. There is little branding ambiguity. But this is a mapping between the attacker's pathname and an official banking product, not proof that a genuine Mandiri signing certificate was present in the attacker-hosted APK.

### BTN: `/btn/`

`btn` points to **Bank Tabungan Negara**. The exact client is less certain. BTN introduced **balé by BTN** as its super app in February 2025 and describes it as the successor to BTN Mobile. [3]

A route with an old or generic name can keep serving newer binaries. Without examining the downloaded APK's package information and resources, choosing a particular generation would be speculation.

### BCA: `/mybca/` and `/bcamoblie/`

This pair deserves its own section. Both **myBCA** and **BCA mobile** are real, distinct BCA applications. BCA's official instructions explicitly discuss installing and migrating them separately. [2]

The string `bcamoblie` appears to be an operator-side misspelling of `bcamobile`. I would preserve it exactly as observed in an IOC list, because correcting the typo would break matching against the actual path.

### BNI: `/bni/`

The banking institution is a strong match for **BNI**. Its current banking ecosystem prominently features **wondr by BNI**, launched in July 2024, and BNI subsequently announced migration from the older BNI Mobile Banking service. [4]

That makes wondr the leading app candidate, **not a binary-confirmed result**.

### BRI: `/bri/`

`bri` naturally matches **Bank Rakyat Indonesia**, and **BRImo** is the leading consumer mobile-banking candidate. The app identification still needs confirmation from the replacement binary itself.

### BSI: `/bsi/`

`bsi` matches **Bank Syariah Indonesia**. BSI says **BYOND by BSI** is its primary mobile-banking channel, but it also reported that **BSI Mobile** remained active during the transition in 2026. [5]

So `/bsi/latest.apk` cannot establish which app lineage was repackaged.

### Bank Jakarta: `/jak/`

The `/jak/` label is strongly consistent with **Bank Jakarta's JakOne Mobile** ecosystem. The precise application and release would again require checking the APK itself.

> *A directory name is a useful attribution clue. A verified Android package name and signing certificate would be much stronger evidence.*

---

## 3. The Detail That Makes the Numbers Fit: BCA Appears Twice

ThreatFabric's published count is specific: **eight modified apps, seven banks**. [1]

The extracted paths supply an elegant explanation:

```text
Eight bank-specific routes
         |
         +-- Mandiri       /livin/
         +-- BTN           /btn/
         +-- BCA           /mybca/
         |                  /bcamoblie/   <-- second BCA route
         +-- BNI           /bni/
         +-- BRI           /bri/
         +-- BSI           /bsi/
         +-- Bank Jakarta  /jak/
         |
         v
     8 routes / 7 banks
```

This does not prove that every underlying APK was successfully replaced on a victim device. What it does provide is a **coherent target manifest** that fits the vendor's reported total without inventing an eighth bank.

It is also the main finding that differentiates this route analysis from a general summary of Hirudo. The vendor disclosed the scale and the host; the supplied route list provides names that can be investigated as potential target identities.

---

## 4. The First Stage: A Normal-Looking App With a Fake Update

Before the banking-app replacement, the operator still needs to establish control of the victim's Android device.

ThreatFabric reports a **two-application delivery design**. The initial app presents itself as something that people might download voluntarily: a shopping application, an adult-video application, or a short-drama/video service. Examples described in its report include Shopora, 1V1 X, KoiShort, and ZoyaTV. [1]

The lure may actually display functional content. This matters because a user is less likely to suspect an application that appears to do what its name promises.

Then comes the fake update.

```text
Advertising / download lure
             |
             v
   Shopping or video app
   (apparently functional)
             |
             v
      "Update required"
             |
             v
      Separate RAT APK
             |
             v
  Permissions / Accessibility
             |
             v
       HIRUDO RAT
```

The user is not necessarily exploiting an Android vulnerability. The operation relies heavily on **installation prompts, user trust, and abuse of powerful platform features**.

The distinction between the lure and the payload is important for incident response. Deleting the obvious shopping or entertainment app may not remove the separately installed RAT.

---

## 5. The RAT Sample: What the Metadata Actually Shows

The investigation material includes metadata for a known Hirudo sample that ThreatFabric identifies as **Hirudo RAT v7.8.166**. [1]

```text
SHA-256:
b2270c29b3d0a49a716730d6ef108cffec0b1a4b159dd02e93333fb85e492b99

Package:     com.r9nkvx.qw2mpl
Version:     7.8.166
Size:        approximately 45.94 MB
Target SDK:  36
Platform:    Android
```

That hash belongs to the documented RAT sample, **not to the unverified `/livin/latest.apk` replacement**. Mixing those two artifacts would be a serious analytical error.

The supplied manifest and component inventory includes capabilities associated with device surveillance and remote control:

| Observed capability / component | Why it matters |
|---|---|
| `READ_SMS`, `RECEIVE_SMS` | Potential access to SMS content, including messages used in authentication workflows |
| Accessibility services | UI observation and interaction when enabled and appropriately configured |
| `SYSTEM_ALERT_WINDOW` | Potential overlay-based impersonation when permitted |
| `CAMERA`, `RECORD_AUDIO` | Camera and microphone access |
| `FOREGROUND_SERVICE_MEDIA_PROJECTION` | Screen-capture / sharing workflows |
| `REQUEST_INSTALL_PACKAGES` | Additional APK installation pathways |
| `RECEIVE_BOOT_COMPLETED` | Ability to respond after device restart |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | Attempts to remain active in the background |
| VPN-related service components | Network tunneling or manipulation capabilities |

Several component names give additional context:

```text
com.nirenr.talkman.a11y.ipc.A11yCoreBridgeService
com.nirenr.talkman.a11y.keepalive.A11yScreenOffSilentAudioService
com.nirenr.talkman.Serivce.SimpleScreenShareService
com.nirenr.talkman.keepalive.ResidentForegroundService
com.nirenr.talkman.vpn.TunnelPipeService
com.nirenr.talkman.push.MyFirebaseMessagingService
```

The same supplied metadata includes receiver actions for `BOOT_COMPLETED`, `LOCKED_BOOT_COMPLETED`, `USER_UNLOCKED`, `SMS_RECEIVED`, screen-state changes, package-install events, and connectivity changes.

There is a difference between **capability** and **observed execution**. Android permissions can be denied, restricted, or unavailable to third-party apps. A manifest entry does not prove that a particular function ran on a victim device.

Nevertheless, the combination is consistent with the much broader credential capture, SMS access, screen viewing, camera use, and device control that ThreatFabric describes from reverse engineering. [1]

> *One dangerous Android permission is not a malware verdict. A coordinated set of remote-control components, suspicious installation events, and known malicious infrastructure is a very different story.*

---

## 6. The Replacement Trick: Uninstall the Bank, Install a Lookalike

This is the part that changes the threat model.

A conventional banking overlay tries to fool the user by placing attacker-controlled content over the legitimate app. Hirudo has that kind of credential-theft functionality too, but for a selected set of Indonesian targets it attempts something more invasive: **replace the installed banking application itself**. [1]

A simplified reconstruction looks like this:

```text
Hirudo RAT is active
          |
          v
Observe installed / foreground apps
          |
          v
Match a registered banking target
          |
          v
Check expected app / signing identity
          |
          v
Display deceptive update prompt
          |
          v
Open Android uninstaller for genuine app
          |
          +------------------------------+
          |                              |
          v                              v
  Genuine app removed          Download patched APK
          |                              |
          +---------------+--------------+
                          |
                          v
              Install modified copy
                          |
                          v
               Activate internal hooks
```

Why uninstall first?

Because Android package signing exists for a reason. A replacement APK with the same package identifier but a different signing identity generally cannot be installed as a normal update over the trusted application. ThreatFabric reports that the malicious patched apps keep important identifying attributes while being signed with a **public Android test key**, not the bank's legitimate production key. [1]

So the attacker's workflow becomes:

```text
GENUINE BANKING APP
  package = expected identity
  signer  = bank's trusted certificate
            |
            v
         UNINSTALL
            |
            v
PATCHED BANKING APP
  package = familiar identity
  signer  = different / test certificate
```

The uninstall and installation prompts create a real opportunity to interrupt the attack. But if the victim follows the fake update instructions, the result can be a familiar-looking app running code the bank never approved.

**Same icon does not mean same signer. Same package name does not mean same code.**

---

## 7. Inside the Patched App: How the Security Checks Get Blinded

According to ThreatFabric's reverse engineering, the banking client is not necessarily rebuilt from scratch. Much of the original application remains, while a small startup modification loads an injected hooking engine. The report identifies **YAHFA** for Java method hooking and **Dobby** for native-function hooking. [1]

```text
Modified banking APK
|
+-- Original banking UI
+-- Original business logic
+-- Original network / API logic
+-- Original resources
|
+-- Injected initialization
    |
    +-- Embedded hooking engine
        |
        +-- Java method hooks (YAHFA)
        +-- Native hooks (Dobby)
        +-- Signing-identity manipulation
        +-- Accessibility-state concealment
        +-- Screen-protection bypass
        +-- Integrity / attestation interference
```

That is precisely why this attack is dangerous. A user might still see real menus, familiar transfer screens, and much of the usual app behavior. The injected components operate underneath that appearance.

### Signing certificate checks

Consider an application asking Android which certificate signed its own APK.

```text
Actual modified APK:
    signer = public test key

Application asks:
    "Which certificate signed this app?"
                |
                v
         Injected hook
                |
                v
Application receives:
    bank's expected certificate data
```

ThreatFabric says the hooking engine can substitute expected signing information. The goal is to convince local code that the installed app is still genuine even though its actual APK signature is not. [1]

### Accessibility checks

A banking app may look for active Accessibility services because those services can be abused to observe sensitive screens or automate UI interactions.

```text
Real device state:
    Hirudo Accessibility service = ENABLED

Banking client query:
    "Which Accessibility services are enabled?"
                |
                v
          Hooked result
                |
                v
            []
```

The check may execute, but the response has been manipulated.

### `FLAG_SECURE` protection

Android's `FLAG_SECURE` is commonly used by sensitive applications to limit screenshots and certain display-capture paths. ThreatFabric reports that Hirudo's patched code attempts to suppress this protection, enabling the malware's viewing functionality to capture previously protected banking screens. [1]

```text
Expected:    Sensitive window -> FLAG_SECURE set

Modified:    Sensitive window -> hook changes flags
                                    |
                                    v
                           Capture protection reduced
```

The lesson is broader than any one API: **if hostile code runs inside the process that performs a security check, the check's local result may be hostile too**.

---

## 8. Play Integrity: A Verdict Is Not a Transaction Authorization

Play Integrity is designed to help a backend assess signals about the app and environment involved in a request. Google's documented app-integrity information can include package identity, certificate digest, version, and application-recognition status. [6]

A patched bank app signed with the wrong key should create an application-integrity problem.

But ThreatFabric reports an additional Hirudo technique: some patched builds interfere with key attestation and send Play Integrity requests through an **operator-controlled relay service**. Its analysis also identifies use of a publicly leaked keybox, with at least two associated certificates appearing in an attestation revocation list as compromised. [1]

The reported relay endpoint is:

```text
43[.]225[.]157[.]157:8766/request_token
```

This is not proof that every relay request succeeds, or that every bank backend would accept its result. The point is that the attacker is trying to make app-generated integrity signals less reliable.

The defensive question should not simply be:

```text
"Did the client send an Integrity token?"
```

It should be closer to:

```text
Is the verdict valid and correctly verified server-side?
Does it identify the expected application and certificate?
Is it bound to this user's protected operation?
Does it match the actual request / transaction content?
Is the evidence fresh and consistent with other risk signals?
Are relevant attestation certificates revoked?
```

Google specifically documents `requestHash` for binding a Standard Play Integrity request to relevant action parameters; classic requests use a nonce-based approach. [7]

```text
Transfer requested
      |
      v
Canonicalize protected request data
      |
      v
Compute request digest for requestHash
      |
      v
Obtain Integrity token
      |
      v
Server verifies token and request binding
      |
      +-- app / certificate / version
      +-- action / request digest
      +-- freshness / reuse protection
      +-- behavioral and account-level risk
      |
      v
Server-side authorization decision
```

**Integrity is evidence to evaluate. It should never be the only authorization decision.**

---

## 9. Infrastructure: The CloudFront Host Was Only One Layer

The extracted strings and ThreatFabric's published IOCs describe several infrastructure roles. The important thing is not to flatten all of them into the generic label "C2".

| Infrastructure | Reported or inferred role | Evidence basis |
|---|---|---|
| `d3q4tlm7pw29zr[.]cloudfront[.]net` | Delivery of eight patched Indonesian banking APKs | ThreatFabric + supplied routes |
| `jiekou[.]baojingtextile[.]cn` | Hirudo RAT configuration API | ThreatFabric |
| `134[.]122[.]153[.]136:8085` and `:18085` | HTTP API | ThreatFabric |
| `134[.]122[.]153[.]136:1883` and `:1884` | MQTT service | ThreatFabric |
| `43[.]225[.]157[.]157:8766/request_token` | Play Integrity relay | ThreatFabric |
| `www[.]helloharmonyos[.]com` | Lure registration / channel configuration | ThreatFabric |
| `134[.]122[.]153[.]68:8080` | APK processing panel | ThreatFabric |
| `134[.]122[.]153[.]92:8080` | APK processing panel | ThreatFabric |
| `18[.]139[.]79[.]161` | APK source host / app-management console | ThreatFabric |
| `d3s06yhcwfnwcx[.]cloudfront[.]net` | 1V1 X distribution | ThreatFabric |
| `d21gcldn8gp80l[.]cloudfront[.]net` | KoiShort casino-themed distribution | ThreatFabric |
| `dydjjnnjg7jeb[.]cloudfront[.]net` | KoiShort short-drama distribution | ThreatFabric |

These infrastructure roles come from the vendor report unless explicitly described otherwise. I did not independently scan, authenticate to, or enumerate those hosts. [1]

One detail matters operationally: **Amazon CloudFront is a legitimate CDN**. A malicious object on a particular distribution does not make `cloudfront.net` as a whole malicious. Blocking all of CloudFront would create enormous collateral damage.

The actionable IOC is the **specific malicious distribution**, ideally combined with path, HTTP metadata, app-install telemetry, and the surrounding campaign indicators.

---

## 10. The Configuration API Strings

The supplied memory strings include several paths under the reported configuration domain:

```text
jiekou[.]baojingtextile[.]cn/Appapi/register/appkey/abc123456
jiekou[.]baojingtextile[.]cn/appapi/appInit/appkey/abc123456
jiekou[.]baojingtextile[.]cn/appapi/getSharePid/appkey/abc123456
jiekou[.]baojingtextile[.]cn/appapi/playPage/appkey/abc123456/did/
```

The endpoint names suggest a possible control-plane sequence:

```text
register       -> possible registration
appInit        -> possible configuration/bootstrap
getSharePid    -> possible sharing/campaign identifier
playPage       -> possible page/content configuration
```

**Those meanings are hypotheses derived from names, not confirmed request/response semantics.**

Similarly, `appkey/abc123456` is a hardcoded credential-like string but cannot be classified as an actual authentication secret without protocol analysis. It could be a campaign parameter, placeholder, routing key, or genuine access token.

What makes it useful right now is its potential value as a **clustering marker**. Finding the same unusual API pattern in another sample would provide a promising pivot, especially alongside a matching package family or infrastructure indicator.

---

## 11. String Extraction: Not Every URL Is an IOC

One of the easiest ways to make malware triage noisy is to export every URL from memory or a decompiled APK and call all of them malicious.

The supplied extraction also contains links to ordinary Android ecosystem components and library references, including:

```text
apache.org/licenses/LICENSE-2.0
schemas.android.com/apk/res/android
developer.android.com/training/articles/direct-boot
crashpad.chromium.org
firebase-settings.crashlytics.com
app-measurement.com
android.googlesource.com/toolchain/llvm-project
youtrack.jetbrains.com
github.com/TooTallNate/Java-WebSocket
```

These can originate from bundled open-source licenses, XML namespaces, SDK documentation, crash reporters, Firebase modules, Google measurement libraries, and toolchain dependencies.

```text
STRING FOUND IN APK
         !=
CONNECTION OBSERVED ON DEVICE
         !=
MALICIOUS NETWORK ACTIVITY
```

That distinction is not optional. A legitimate framework URL may be present in both a clean Android application and a malicious repackaged copy.

I would prioritize indicators with **campaign-specific context**: dedicated replacement routes, the documented configuration domain, known RAT package names and hashes, and correlated unexpected banking-app installation events.

---

## 12. Reconstructed Attack Timeline

Combining the published Hirudo behavior with the supplied target routes produces this incident hypothesis:

```text
T0    Victim encounters malicious advertisement / download lure
 |
T1    Victim installs a functional-looking shopping or media app
 |
T2    Lure displays a fake mandatory update
 |
T3    Separate Hirudo RAT APK is installed
 |
T4    Victim grants relevant permissions / Accessibility access
 |
T5    RAT identifies a target banking application
 |
T6    Victim is prompted through removal of the genuine bank app
 |
T7    Bank-specific replacement is retrieved from the CloudFront host
 |
T8    Modified bank APK is installed with a different signer
 |
T9    Injected hooks manipulate local security checks
 |
T10   RAT attempts credential, screen, SMS, and other collection
 |
T11   Operator may attempt fraudulent activity
```

**This is a reconstructed sequence, not an observed timeline from a specific victim phone.** Individual steps can fail, differ by lure, or be stopped by Android protections and user decisions. The ordering reflects the mechanisms described by ThreatFabric. [1]

The critical detection windows are around **T3-T4** (payload installation and permission grants) and **T6-T8** (unusual removal and replacement of a banking app).

---

## 13. Detection Engineering: Focus on the Sequence, Not One Permission

A standalone alert for `READ_SMS` or an enabled Accessibility service will produce false positives. A better analytic combines multiple behaviors that should not normally occur together:

```text
Unknown / side-loaded application installed
                 +
New Accessibility service enabled
                 +
Banking application uninstalled
                 +
Same banking package reinstalled from outside trusted channel
                 +
Signing identity does not match bank-approved certificate
                 +
Contact with known Hirudo infrastructure
                 =
        HIGH-CONFIDENCE INVESTIGATION CANDIDATE
```

For organizations with Mobile Threat Defense, managed Android telemetry, or app-integrity measurements, the strongest checks are:

- **Package identity and signer comparison:** verify the installed bank app against independently maintained official package and signing-certificate allowlists.
- **Installation chronology:** correlate uninstall/reinstall events, installer source, and recently granted Accessibility permissions.
- **Known sample matching:** search for Hirudo package identifiers and hashes where available.
- **Infrastructure correlation:** hunt the exact CloudFront distribution and documented configuration services, not generic CDN traffic.
- **Sensitive capability correlation:** look at SMS access, overlay capability, screen-sharing components, boot receivers, and related app activity in the same time window.
- **Fraud telemetry:** investigate new beneficiaries, abnormal payment velocity, unusual transfers, device re-enrollment, and anomalous authentication recovery.

### Safe static APK verification

If defenders have a **forensically acquired local APK**, Android's `apksigner` can print its signing-certificate information without executing the app:

```bash
apksigner verify --verbose --print-certs suspicious.apk
```

Compare the reported certificate digest against a **trusted official baseline** for the exact package and legitimate signing-key lineage. A package name, filename, screenshot, or icon is not an adequate baseline by itself.

Do not install suspected malicious APKs on a personal device to inspect them.

### Simplified detection logic

The following is platform-neutral pseudocode, **not a production-ready SIEM rule**:

```text
IF installed_package IN official_banking_package_allowlist
AND observed_signer NOT IN approved_signers[installed_package]
THEN
    alert.severity = CRITICAL
    alert.reason = "Banking package has an unapproved signing identity"

IF bank_app_uninstalled AND bank_app_reinstalled_within_short_window
AND (installer_is_untrusted OR new_accessibility_service_enabled)
THEN
    alert.severity = HIGH
    alert.reason = "Suspicious banking application replacement sequence"
```

The certificate mismatch is particularly important because it can survive changes to the malware's domain names, filenames, and visual branding.

---

## 14. IOC Master List: Hirudo and the Patched-Banking Cluster

The following are **historical/research indicators**, not a claim that every endpoint is still active at the time of reading. Network indicators are defanged for safety. Hashes and package names retain their exact form for searching.

### RAT and lure APK hashes

| SHA-256 | Android package | Published role |
|---|---|---|
| `f796eb9ecf516e4f1fafcf89200d9f7d8ddd234f705e63f1bc8fc3ef583f0248` | `com.agbbh.kjdrttt` | Shopping lure / dropper |
| `0097a0406be20544e98e981075427f40789db40fe4a53b1b4523383a945328b7` | `com.onev.vip` | 1V1 X lure / dropper |
| `2e45277acfde3fb704179476b08f428f66a03670bbdbf2e8ec79c0b0333c009b` | `com.koishort.app` | KoiShort lure / dropper |
| `d51f2bd7ffb49c224d8eaec2c8ca1c8ab9fce2fd5fdceea01eb54296c2f47b4d` | `com.zoyatv.app` | ZoyaTV lure / dropper |
| `425cd7072b5e9386ac94e97fe272980b6ef36240df6b5d77b371ab7378004c83` | `com.zk7qpx.mv9wlt` | Hirudo RAT (earlier build) |
| `b2270c29b3d0a49a716730d6ef108cffec0b1a4b159dd02e93333fb85e492b99` | `com.r9nkvx.qw2mpl` | Hirudo RAT v7.8.166 |

These hashes and role attributions are from ThreatFabric's published indicator set and the supplied sample report. [1]

### Campaign-specific infrastructure

| Indicator | Classification |
|---|---|
| `d3q4tlm7pw29zr[.]cloudfront[.]net` | Eight-bank-app replacement distribution |
| `jiekou[.]baojingtextile[.]cn` | RAT configuration API |
| `134[.]122[.]153[.]136:8085` | HTTP API |
| `134[.]122[.]153[.]136:18085` | HTTP API |
| `134[.]122[.]153[.]136:1883` | MQTT service |
| `134[.]122[.]153[.]136:1884` | MQTT service |
| `43[.]225[.]157[.]157:8766/request_token` | Integrity relay |
| `www[.]helloharmonyos[.]com` | Lure configuration |
| `134[.]122[.]153[.]68:8080` | APK-processing panel |
| `134[.]122[.]153[.]92:8080` | APK-processing panel |
| `18[.]139[.]79[.]161` | APK source / management |
| `d3s06yhcwfnwcx[.]cloudfront[.]net` | 1V1 X distribution |
| `d21gcldn8gp80l[.]cloudfront[.]net` | KoiShort casino distribution |
| `dydjjnnjg7jeb[.]cloudfront[.]net` | KoiShort short-drama distribution |

### Path-level targeting pivots

```text
/livin/latest.apk
/btn/latest.apk
/mybca/latest.apk
/bcamoblie/latest.apk
/bni/latest.apk
/bri/latest.apk
/bsi/latest.apk
/jak/latest.apk
```

**Important:** Those strings should be interpreted alongside the specific CloudFront distribution. A generic `/latest.apk` path on an unrelated legitimate domain is not an IOC.

---

## 15. If You Suspect an Android Phone Was Exposed

If a device installed an unfamiliar shopping/video app, accepted an unexpected "update," or suddenly removed and reinstalled a banking app, the safest response is to treat the phone as potentially untrusted until assessed.

**1. Stop sensitive banking activity on that phone.** Use a separately trusted device or the bank's official support channel. Avoid entering credentials, OTPs, or PINs into an app whose provenance is uncertain.

**2. Contact the bank using a known official channel.** Explain the suspected malicious app replacement and ask about account access, transaction review, session/device re-registration, and fraud-protection measures.

**3. Preserve evidence when an investigation is needed.** Record visible app names, installation times, Android security prompts, and suspicious URLs without opening them. In an enterprise environment, let the mobile security or DFIR team collect logs and app artifacts under appropriate procedures.

**4. Review Accessibility and installation permissions.** Look for unfamiliar enabled services and apps allowed to install unknown packages. The absence of an obvious warning is not proof of a clean device.

**5. Recover accounts from a trusted environment.** Revoke sessions and change affected banking, email, or other sensitive credentials as advised by the provider. A device confirmed to be compromised may require a controlled wipe and reinstall from trusted sources, not just deletion of the visible lure app.

**6. Reinstall banking apps only through official distribution channels.** Confirm the legitimate source and bank guidance before restoring access.

If there are unauthorized transactions, preserve supporting evidence and report them promptly through the bank's fraud channel.

---

## 16. What Is Still Unverified

Good threat intelligence is as much about knowing where the evidence ends as knowing what it supports.

For this investigation, these gaps remain:

- **No eight-binary validation:** The actual eight bank-specific APK files were not individually acquired and examined for this article.
- **No verified Livin' replacement hash:** The supplied research notes document an unsuccessful fetch of the exact `/livin/latest.apk` object. The RAT sample hash is not a substitute.
- **No bank-specific binary diff:** Claims about the particular injected classes, signatures, and exact package versions for each of the eight replacements require those binaries.
- **No confirmed affected-customer count:** The existence of operator infrastructure does not tell us how many Indonesian customers were infected.
- **No proof of bank-backend intrusion:** This campaign targets Android endpoints and app trust, not necessarily the banks' internal networks.
- **No verified current endpoint availability:** Vendor observations that routes were active into October 2026 should not be read as a real-time availability check on the day someone reads this article.
- **No confirmed operator identity:** Infrastructure and malware-family attribution do not by themselves establish a real-world person or organization behind the operation.

The next useful research step is not to add more adjectives to the headline. It is to acquire authorized, safely handled samples; calculate independent hashes; verify signer and package metadata; compare code against legitimate builds; and reproduce relevant behavior in an isolated lab.

---

## Closing: The Bank Wasn't Necessarily Hacked. The Banking App May Have Been Replaced.

What stands out about Hirudo is not simply that it steals credentials. Android banking malware has been doing that for years. It is the decision to attack **the application's own trust boundary**.

The operation uses a believable lure to get onto the device, a separate RAT to gain sensitive capabilities, and a bank-specific replacement APK to push past controls implemented inside the legitimate client. The attacker tries to make an altered application report that its signature is correct, that dangerous Accessibility services are not running, and that protected screens remain protected.

The supplied eight-route cluster adds a useful intelligence detail to ThreatFabric's public findings. It gives us a defensible, though still binary-unverified, explanation for **which seven Indonesian banking brands are likely represented by the eight targeted applications**. BCA appears twice. The numbers fit.

None of that should be turned into the false claim that seven banks were breached. A repackaged client on a compromised customer phone and an intrusion into a bank's core infrastructure are fundamentally different incidents.

For defenders, the answer is not to trust a green checkmark that comes from the same client process an attacker can patch. Validate application identity independently. Correlate uninstall/reinstall events. Bind integrity evidence to the protected transaction. Keep behavioral and account-level controls on the server.

> *When the client can be replaced, the client cannot be the final authority on whether it is genuine.*

---

## References and Research Notes

The writing and target reconstruction above are based on the supplied investigation material, with external sources used for corroboration. The underlying sample metadata and string extraction were supplied with the research; they are not represented here as new live captures or as independently retrieved replacement binaries.

1. **ThreatFabric**, [*Hirudo: From Fake Updates to Patched Banking Apps*](https://www.threatfabric.com/blogs/hirudo-from-fake-updates-to-patched-banking-apps), October 14, 2026. Primary reporting on the RAT, eight-app/seven-bank count, hooking engine, distribution, attack chain, and published IOCs.
2. **Bank Central Asia**, [*Cara Memindahkan Aplikasi myBCA dan BCA mobile di Ponsel Baru*](https://www.bca.co.id/id/informasi/edukatips/2025/04/29/02/43/Cara-memindahkan-aplikasi-myBCA-dan-BCA-mobile-di-ponsel-baru). Official confirmation that myBCA and BCA mobile are distinct applications.
3. **Bank Tabungan Negara**, [*BTN Resmi Luncurkan Bale by BTN*](https://www.btn.co.id/id/about/gallery/news/news/listing/2025/02/10/btn-resmi-luncurkan-bale-by-btn), February 10, 2025. Documents the BTN Mobile-to-balé transition.
4. **Bank Negara Indonesia**, [*Pengalihan Layanan BNI Mobile Banking ke wondr by BNI*](https://www.bni.co.id/id-id/beranda/kabar-bni/pengumuman/articleid/24810), June 2, 2025. Documents the migration and app naming.
5. **Bank Syariah Indonesia**, [*BYOND by BSI Kian Diminati Generasi Muda*](https://www.bankbsi.co.id/news-update/berita/byond-by-bsi-kian-diminati-generasi-muda-bsi-targetkan-10-juta-pengguna-di-2026), March 31, 2026. Confirms BYOND and continued presence of BSI Mobile during the transition.
6. **Google Android Developers**, [*Play Integrity API overview*](https://developer.android.com/google/play/integrity/overview). Official description of app/device integrity signals and response interpretation.
7. **Google Android Developers**, [*Make a standard API request*](https://developer.android.com/google/play/integrity/standard). Official guidance for `requestHash` and binding integrity results to requests.
8. **BleepingComputer**, [*Hirudo Android malware replaces banking apps to evade security checks*](https://www.bleepingcomputer.com/news/security/hirudo-android-malware-replaces-banking-apps-to-evade-security-checks/), October 14, 2026. Independent news coverage of the ThreatFabric disclosure.

**Research methodology:** Supplied string and APK-metadata review; comparison against ThreatFabric's published infrastructure and reported reverse engineering; official banking-product cross-references; careful separation of direct observations, vendor findings, and analytical inference.

**Disclosure reminder:** Indicators are provided for defensive threat hunting, incident response, and research. Brand names identify likely *impersonation targets*, not confirmed breaches of the organizations mentioned.
