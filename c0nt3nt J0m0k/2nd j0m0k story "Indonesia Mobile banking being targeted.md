# Hirudo Android Banking RAT: Deep Research Report on the `livin/latest.apk` Cluster

## Executive assessment

The data you supplied provides the missing link needed to reconstruct, with **high confidence, the seven Indonesian banks targeted by Hirudo's patched-application replacement operation**.

ThreatFabric independently reports that Hirudo maintains a registry of 42 financial applications but has **purpose-built replacement APKs for exactly eight Indonesian banking apps belonging to seven banks**. ThreatFabric also identifies `d3q4tlm7pw29zr[.]cloudfront[.]net` specifically as the download host for those eight patched banking applications. citeturn17search0 The infrastructure and behavior described in the ThreatFabric report are also reproduced in the material supplied with this investigation. fileciteturn0file2

Your extracted strings expose the eight bank-specific paths on that host:

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

Those names produce an unusually clean result:

**Bank Mandiri, Bank Tabungan Negara, Bank Central Asia, Bank Negara Indonesia, Bank Rakyat Indonesia, Bank Syariah Indonesia, and Bank Jakarta/JakOne.**

The reason eight applications correspond to seven banks is almost certainly **BCA**, for which the attacker infrastructure contains two independent routes: `mybca` and the misspelled `bcamoblie`, corresponding to **myBCA** and **BCA mobile**. BCA itself confirms that these are distinct BCA mobile-banking applications and that myBCA was introduced as a complement to BCA mobile. citeturn15search1turn15search10turn15search12

This is a **targeting attribution**, not evidence that the banks' backend systems were compromised. Hirudo's documented technique attacks the **customer's Android device and a modified copy of the banking client**, attempting to defeat security controls from inside the replacement application. citeturn17search0

One important research limitation: I attempted to retrieve the exact supplied `livin/latest.apk` path directly, but the retrieval environment returned a cache/fetch failure. Therefore, I am **not claiming to have independently calculated the SHA-256, signing certificate, package name, or binary diff of that particular Livin replacement APK** in this session. citeturn18view0 The attribution below instead combines the route names you extracted, ThreatFabric's independently documented infrastructure, the known Hirudo RAT sample metadata, and official bank sources.

**Overall assessment: Critical / high-confidence active banking-malware infrastructure.** ThreatFabric reported the Hirudo research on October 14, 2026 and stated that some associated download routes remained active into October 2026. citeturn17search0turn17search1

## Target reconstruction: the Indonesian banks

The eight route names are especially valuable because ThreatFabric's public report describes the **number** of affected banks and applications but does not publicly enumerate their identities. The routes effectively provide a target manifest.

| Attacker-controlled route | Attributed bank | Likely legitimate application being impersonated/repacked | Confidence |
|---|---|---|---|
| `.../livin/latest.apk` | **Bank Mandiri** | **Livin' by Mandiri** | **Very high** |
| `.../btn/latest.apk` | **Bank Tabungan Negara (BTN)** | Most likely **balé by BTN / former BTN Mobile** | **High bank / medium-high app** |
| `.../mybca/latest.apk` | **Bank Central Asia (BCA)** | **myBCA** | **Very high** |
| `.../bcamoblie/latest.apk` | **Bank Central Asia (BCA)** | **BCA mobile** | **Very high** |
| `.../bni/latest.apk` | **Bank Negara Indonesia (BNI)** | Most likely **wondr by BNI**; exact build remains unverified | **High bank / medium app** |
| `.../bri/latest.apk` | **Bank Rakyat Indonesia (BRI)** | Most likely **BRImo** | **High bank / high app** |
| `.../bsi/latest.apk` | **Bank Syariah Indonesia (BSI)** | Most likely **BYOND by BSI**; legacy BSI Mobile is another possibility | **High bank / medium app** |
| `.../jak/latest.apk` | **Bank Jakarta / JakOne ecosystem** | **JakOne Mobile** | **Very high** |

### Bank Mandiri

`livin` is effectively unambiguous. Bank Mandiri's own website identifies **Livin' by Mandiri** as its financial super app and directs Android users to official application stores for installation. citeturn16search2turn16search6

This makes:

```text
/livin/latest.apk
        ↓
Bank Mandiri
        ↓
Livin' by Mandiri
```

a **very-high-confidence attribution**.

The crucial distinction is that an APK obtained from the Hirudo CloudFront path would not be an official Mandiri distribution merely because it resembles Livin'. ThreatFabric's analysis says these replacements retain the target application's identity characteristics where useful but are modified and re-signed by the attacker. citeturn17search0

### Bank Tabungan Negara

`btn/latest.apk` strongly identifies **PT Bank Tabungan Negara (Persero) Tbk**. The only ambiguity is which generation of its mobile client was used as the attacker's patching base.

BTN officially renamed/replaced **BTN Mobile** with **balé by BTN**, launching balé as its super app in February 2025. BTN explicitly says it was “previously known as BTN Mobile,” and by March 2026 described balé as an important primary digital platform. citeturn14search0turn14search2

Therefore:

```text
/btn/latest.apk
        ↓
Bank Tabungan Negara
        ↓
likely balé by BTN / BTN Mobile code lineage
```

The **bank identification is high confidence**. The exact APK generation cannot be established from the filename alone.

### Bank Central Asia

This is the most informative pair:

```text
/mybca/latest.apk
/bcamoblie/latest.apk
```

BCA's official material explicitly identifies both **myBCA** and **BCA mobile** as BCA applications, including a current September 2026 notice discussing myBCA functionality exposed to BCA mobile users. citeturn15search3turn15search8

The typo `bcamoblie` instead of `bcamobile` is likely an operator-side naming mistake. What matters analytically is that there are **two separate BCA-related download objects**.

That explains ThreatFabric's otherwise curious statement:

> eight applications, seven banks

without needing to invent an eighth institution. ThreatFabric independently gives exactly that count. citeturn17search0

The resulting model is:

```text
Bank Central Asia
   ├── myBCA        → /mybca/latest.apk
   └── BCA mobile   → /bcamoblie/latest.apk
```

This is one of the strongest findings in the supplied dataset.

### Bank Negara Indonesia

`bni/latest.apk` points directly to **Bank Negara Indonesia**.

By 2026, BNI's primary modern mobile application is **wondr by BNI**. BNI states that wondr launched on July 5, 2024 and identifies it as a mobile-banking application. citeturn14search5turn14search6

However, I would not label the malicious file specifically as a patched wondr APK until its package name and resources are obtained. A malicious infrastructure operator could preserve a generic route such as `/bni/` while replacing the underlying target build over time.

So the defensible conclusion is:

**BNI is confirmed at high confidence; wondr by BNI is the leading candidate for the exact application.**

### Bank Rakyat Indonesia

The `bri` identifier corresponds naturally to **Bank Rakyat Indonesia**. BRI's official material identifies **BRImo** as its super app and reports it as its major digital customer platform. citeturn15search6turn15search16

Thus:

```text
/bri/latest.apk
       ↓
BRI
       ↓
very likely BRImo
```

The bank attribution is high confidence and the application attribution is also strong, although a binary-level confirmation would still require the APK's original application ID, resources, version code, and signing history.

### Bank Syariah Indonesia

`bsi/latest.apk` points to **Bank Syariah Indonesia**.

BSI's current main application is **BYOND by BSI**. In March 2026 BSI described BYOND as a primary mobile-banking channel and confirmed that an older **BSI Mobile** application also remained in use during the transition period. citeturn15search0 By August 2026, BSI continued to promote BYOND as its current mobile-banking channel. citeturn15search17

Consequently, there are two plausible bases for the attacker APK:

```text
/bsi/latest.apk
       ↓
Bank Syariah Indonesia
       ├── BYOND by BSI       ← most likely in 2026
       └── legacy BSI Mobile  ← cannot yet be excluded
```

The bank identification is high confidence; the specific app remains a medium-confidence attribution until the binary is inspected.

### Bank Jakarta and JakOne Mobile

The `/jak/latest.apk` route is strongly consistent with **JakOne Mobile**. Bank Jakarta's current official site lists JakOne Mobile as its digital-banking smartphone application and directs customers to obtain it from Google Play or Apple's App Store. citeturn16search14turn16search16

Therefore the likely target is:

```text
/jak/latest.apk
       ↓
Bank Jakarta
       ↓
JakOne Mobile
```

This completes the seven-bank set and makes the route list fit ThreatFabric's **eight-app / seven-bank** statement exactly. citeturn17search0

### Reconstructed target set

The best-supported list is therefore:

**Bank Mandiri → BTN → BCA → BNI → BRI → BSI → Bank Jakarta/JakOne.**

BCA accounts for two of the eight patched applications.

Again, this should be worded operationally as **“banks whose Android applications were targeted for malicious replacement”**, not “banks that were hacked.”

## Artifact and RAT sample analysis

The accompanying Hirudo sample metadata provides useful independent evidence for understanding what sits behind those patched banking APKs.

The known sample:

```text
SHA-256
b2270c29b3d0a49a716730d6ef108cffec0b1a4b159dd02e93333fb85e492b99

Package
com.r9nkvx.qw2mpl

Displayed version
7.8.166

Size
45.94 MB

Target SDK
36
```

is classified in the supplied sample information as an Android executable and contains a broad set of components and permissions consistent with a fully interactive mobile RAT. fileciteturn0file0 ThreatFabric identifies that same hash and package as **Hirudo RAT v7.8.166**. citeturn17search0

### The permission profile is coherent, not random

Several declared capabilities are particularly revealing:

| Android capability observed | Security significance |
|---|---|
| `READ_SMS`, `RECEIVE_SMS` | Read incoming or stored SMS, relevant to OTP/message theft |
| `CAMERA` | Camera access |
| `RECORD_AUDIO` | Microphone access |
| `SYSTEM_ALERT_WINDOW` | Ability to draw over other applications when authorized |
| Accessibility service components | Can inspect or interact with foreground UI when enabled |
| `FOREGROUND_SERVICE_MEDIA_PROJECTION` | Supports persistent screen-sharing/capture workflows |
| `REQUEST_INSTALL_PACKAGES` | Relevant to installation of additional APKs |
| `RECEIVE_BOOT_COMPLETED` | Enables restart/persistence logic after reboot |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | Helps maintain long-running background behavior |
| `MANAGE_EXTERNAL_STORAGE` | Broad filesystem access subject to Android restrictions |
| VPN-related components | Supports traffic manipulation or tunneling behavior |

These declarations appear together in the supplied sample metadata. fileciteturn0file0 ThreatFabric independently reports SMS collection, front- and rear-camera sessions, screen viewing via WebRTC, credential theft, lock-screen capture, application installation, and device takeover functionality in Hirudo. citeturn17search0

Android's Accessibility framework is especially important because an enabled service can obtain the root node of the active window if configured for window-content retrieval; Android also supports retrieving interactive window information through the Accessibility APIs. citeturn19search0turn19search1 In banking-malware context, this creates the technical foundation for detecting what app is on screen, reading UI state, and programmatically assisting interactions.

The APK metadata goes beyond permissions. It exposes components such as:

```text
com.nirenr.talkman.a11y.ipc.A11yCoreBridgeService
com.nirenr.talkman.a11y.keepalive.A11yScreenOffSilentAudioService
com.nirenr.talkman.Serivce.SimpleScreenShareService
com.nirenr.talkman.keepalive.ResidentForegroundService
com.nirenr.talkman.vpn.TunnelPipeService
com.nirenr.talkman.push.MyFirebaseMessagingService
```

along with receivers for events including:

```text
BOOT_COMPLETED
LOCKED_BOOT_COMPLETED
USER_UNLOCKED
SMS_RECEIVED
SCREEN_ON
SCREEN_OFF
PACKAGE_ADDED
CONNECTIVITY_CHANGE
```

These are directly present in the supplied APK metadata. fileciteturn0file0 Taken together with ThreatFabric's reverse engineering, they provide a much stronger behavioral picture than any individual permission would.

A technical caveat is important: **a permission being declared in the manifest does not automatically mean Android will grant it**. Several sensitive or privileged Android permissions are restricted by platform policy or signing level. Therefore, the manifest is evidence of intended capability or code requirements, while runtime execution must still be confirmed separately.

### What the application is designed to accomplish

The observed feature set can be conceptually divided into five layers:

```text
                 HIRUDO RAT
                     │
        ┌────────────┼────────────┐
        │            │            │
  Device control   Collection   Persistence
        │            │            │
 Accessibility      SMS          Boot
 Screen control     Camera       Foreground svc
 VPN/tunneling      Audio        alarms
        │
        ▼
 Banking-app discovery
        │
        ▼
 Genuine bank app removal
        │
        ▼
 Patched bank APK installation
        │
        ▼
 In-process security bypass
```

ThreatFabric's reverse engineering supports this broader architecture and describes Hirudo as combining credential phishing, remote device control, screen and camera access, SMS theft, and replacement of selected banking applications. citeturn17search0

## Attack chain and patched-app mechanics

Hirudo is more sophisticated than a conventional Android banking overlay because the operator does not stop at placing a fake login window above the bank application.

Instead, for the high-value Indonesian targets, the attacker attempts to **replace the real bank APK**.

### Initial compromise

ThreatFabric observed lure applications themed around shopping, adult video, and short-drama content. The lure remains functional enough to appear legitimate while presenting what looks like an application update. That “update” actually installs the separate Hirudo RAT payload. citeturn17search0

The simplified chain is:

```text
Malicious advertisement / lure
            │
            ▼
 Functional-looking lure APK
            │
      "Update required"
            │
            ▼
      Hirudo RAT APK
            │
   Accessibility enabled
            │
            ▼
  Device takeover capability
```

The malicious logic is therefore split between at least two applications, reducing the amount of obviously malicious functionality that needs to exist in the initial lure. ThreatFabric explicitly describes this as a two-app architecture. citeturn17search0

### Target detection

Once operational, the RAT maintains information about financial applications and monitors device activity. ThreatFabric reports **42 registered financial applications**, with the specialized replacement mechanism implemented for eight Indonesian applications. citeturn17search0

For an eligible banking target, Hirudo uses Accessibility visibility and stored certificate information to identify the genuine application. citeturn17search0

This means the malware is not blindly replacing random applications. It has a target-selection layer resembling:

```text
foreground / installed application
               │
               ▼
     Is package in target registry?
               │
          yes  ▼
   Validate expected identity
               │
               ▼
Is dedicated replacement available?
               │
          yes  ▼
     Trigger replacement flow
```

### Why the legitimate app must be removed

This is one of the most important technical details in the campaign.

ThreatFabric found that the malicious replacement preserves the bank application's original package name and version characteristics but is **signed using an Android test key rather than the genuine bank's signing key**. citeturn17search0

Android's package update model prevents an APK signed by a different key from silently replacing an installed application with the same application ID. Android requires the update's signing certificate to match the installed application's signing certificate, barring valid signing-key rotation mechanisms. citeturn16search0turn16search1

So Hirudo cannot simply perform:

```text
official.bank.apk
        │
 attacker APK "update"
        ▼
```

because the signatures do not match.

It must instead perform:

```text
GENUINE BANK APP
 official certificate
        │
        ▼
     uninstall
        │
        ▼
ATTACKER-PATCHED BANK APP
 same package identity
 different/test signing key
```

ThreatFabric reports that Hirudo opens the Android uninstaller for the original app and downloads the target-specific replacement in parallel. citeturn17search0

This also exposes a natural defensive chokepoint: **unexpected uninstall/reinstall activity involving a financial application should be considered highly suspicious**, particularly if immediately preceded by Accessibility-driven prompts or followed by an installation from outside the official store.

### Why the replacement still looks authentic

ThreatFabric's reverse engineering found that the attackers do not rebuild the banking client from scratch. Instead, most of the original application code is preserved.

A small initialization modification loads a bundled hooking engine. citeturn17search0

The conceptual structure becomes:

```text
Original bank APK
│
├── Original UI
├── Original network logic
├── Original business logic
├── Original bank APIs
│
└── injected startup logic
       │
       ▼
 malicious hooking engine
       │
       ├── Java hooks
       ├── Native hooks
       ├── signature spoofing
       ├── Accessibility hiding
       ├── screen-capture bypass
       └── integrity manipulation
```

This architecture is much harder for a victim to visually identify than a crude clone: the victim may effectively be looking at substantial amounts of genuine bank application code.

### Runtime hooking

ThreatFabric identifies **YAHFA** for Java method replacement and **Dobby** for native-function hooking inside the patched application. citeturn17search0

The significance of hooking is that the attacker does not necessarily have to remove every security check. It can allow a check to execute but intercept the function that supplies the answer.

Conceptually:

```text
Bank app:
"Which certificate signed me?"

Normal Android:
"Bank's production certificate"

Patched app:
        │
        └── hook intercepts query
                    │
                    ▼
         "Bank's production certificate"
         even though APK is attacker-signed
```

The same model is used against several classes of security control. citeturn17search0

### Accessibility concealment

A banking application may check whether suspicious Accessibility services are enabled.

Hirudo's patched copy hooks this query so that the application receives an empty or attacker-selected response rather than the real state. citeturn17search0

Conceptually:

```text
Actual device state:
Hirudo Accessibility Service = ENABLED

Bank application asks Android:
"What Accessibility services are active?"

Hook:
intercepts return value

Bank application sees:
[]
```

This is an important lesson for mobile-security design: **a client-side check that runs inside an attacker-modified process can itself be modified**.

### Screen protection bypass

Banking apps frequently use `FLAG_SECURE` to restrict screenshots and display capture. Android documents `FLAG_SECURE` as a mechanism intended to prevent screenshots and prevent protected windows from being shown on insecure displays; Google explicitly notes its usefulness for sensitive apps such as banking software. citeturn19search13

ThreatFabric says Hirudo's injected engine strips this protection so that banking screens become available to its remote viewing functionality. citeturn17search0

The effective transition is:

```text
Bank application:
Window.FLAG_SECURE = enabled
              │
              ▼
         malicious hook
              │
              ▼
FLAG_SECURE removed / suppressed
              │
              ▼
screen becomes capturable
```

### Play Integrity manipulation

Play Integrity is supposed to give the backend signals about the application and device executing a sensitive request. Google's `appIntegrity` verdict can include:

```text
packageName
certificateSha256Digest
versionCode
appRecognitionVerdict
```

and `PLAY_RECOGNIZED` means the application and certificate correspond to a version distributed through Google Play, whereas `UNRECOGNIZED_VERSION` indicates a package or certificate mismatch. citeturn19search9

That should make an attacker-test-signed banking APK problematic.

Hirudo responds with a more elaborate mechanism. ThreatFabric reports that the modified applications contain logic for key-attestation manipulation and that Play Integrity requests in some patched apps are **relayed to an operator-controlled service**, rather than trusted exclusively as locally generated evidence. citeturn17search0

ThreatFabric also reports a publicly leaked keybox and notes that at least two associated certificates appeared on Google's attestation revocation list as compromised. citeturn17search0

This means that the defensive question should not simply be:

```text
"Did I receive an Integrity token?"
```

but:

```text
Was this token generated for:
  this exact session?
  this exact action?
  this exact request body?
  this package?
  this certificate?
  this device?
  at this point in time?
```

Google recommends binding Standard Play Integrity requests to the protected operation through `requestHash`; classic requests use a nonce for analogous anti-replay/tampering properties. citeturn17search2

That is directly relevant to Hirudo's relay technique.

## Infrastructure and string triage

Your string list contains two very different categories of URLs: **meaningful Hirudo infrastructure** and **ordinary third-party library artifacts**.

Treating every URL extracted from an APK as an IOC would create a large number of false positives.

### High-value Hirudo infrastructure

The strongest indicators from your material are:

| Indicator | Interpretation | Confidence |
|---|---|---|
| `d3q4tlm7pw29zr[.]cloudfront[.]net` | Dedicated host for eight patched Indonesian banking apps | **Very high** |
| `/livin/latest.apk` | Bank Mandiri replacement route | **Very high attribution** |
| `/btn/latest.apk` | BTN replacement route | **High attribution** |
| `/mybca/latest.apk` | myBCA replacement route | **Very high attribution** |
| `/bcamoblie/latest.apk` | BCA mobile replacement route | **Very high attribution** |
| `/bni/latest.apk` | BNI replacement route | **High attribution** |
| `/bri/latest.apk` | BRI replacement route | **High attribution** |
| `/bsi/latest.apk` | BSI replacement route | **High attribution** |
| `/jak/latest.apk` | JakOne replacement route | **Very high attribution** |
| `jiekou[.]baojingtextile[.]cn` | Hirudo RAT configuration infrastructure | **Very high** |
| `134[.]122[.]153[.]136:8085` | Hirudo HTTP API | **Very high** |
| `134[.]122[.]153[.]136:18085` | Hirudo HTTP API | **Very high** |
| `134[.]122[.]153[.]136:1883` | MQTT infrastructure | **Very high** |
| `134[.]122[.]153[.]136:1884` | MQTT infrastructure | **Very high** |
| `43[.]225[.]157[.]157:8766/request_token` | Reported Play Integrity relay | **Very high** |

ThreatFabric independently identifies the same CloudFront distribution as the eight-banking-app host and `jiekou[.]baojingtextile[.]cn` as the RAT configuration API, along with the HTTP, MQTT, and Integrity-relay infrastructure shown above. citeturn17search0

The CloudFront hostname deserves an important qualification: **Amazon CloudFront is a legitimate CDN service**. The IOC is the particular distribution and malicious objects hosted through it, not CloudFront as a technology. ThreatFabric specifically describes Hirudo as abusing legitimate advertising and cloud delivery infrastructure. citeturn17search0

### The `jiekou.baojingtextile.cn` API strings

Your memory extraction contains:

```text
jiekou[.]baojingtextile[.]cn/Appapi/register/appkey/abc123456

jiekou[.]baojingtextile[.]cn/appapi/appInit/appkey/abc123456

jiekou[.]baojingtextile[.]cn/appapi/getSharePid/appkey/abc123456

jiekou[.]baojingtextile[.]cn/appapi/playPage/appkey/abc123456/did/
```

ThreatFabric's independent classification of this domain as Hirudo's **RAT configuration API** makes these strings materially important. citeturn17search0

The endpoint names suggest a logical workflow:

```text
register
   │
   ├── likely client/device registration
   │
appInit
   │
   ├── likely application/config bootstrap
   │
getSharePid
   │
   ├── likely campaign / sharing / attribution data
   │
playPage
   │
   └── likely page/content configuration
```

Those functional meanings are **analytical inferences from endpoint naming**, not confirmed protocol definitions.

Likewise, the literal:

```text
appkey/abc123456
```

is clearly a hardcoded credential-like/configuration value, but static extraction alone cannot establish whether it is a genuine authorization secret, tenant identifier, placeholder, or merely an API-routing parameter.

It should nevertheless be retained in threat-hunting records because it may provide a useful clustering primitive across related APKs.

### Infrastructure beyond your string dump

ThreatFabric also attributes the following infrastructure to the same operation:

```text
www[.]helloharmonyos[.]com
    lure registration / channel configuration

134[.]122[.]153[.]68:8080
134[.]122[.]153[.]92:8080
    APK-processing panels

18[.]139[.]79[.]161
    APK source host / app-management console

d3s06yhcwfnwcx[.]cloudfront[.]net
    1V1 X distribution

d21gcldn8gp80l[.]cloudfront[.]net
    KoiShort casino route

dydjjnnjg7jeb[.]cloudfront[.]net
    KoiShort short-drama route
```

These are documented by ThreatFabric as parts of the broader Hirudo infrastructure rather than inferred solely from your sample. citeturn17search0

### Strings that should not be treated as Hirudo IOCs

A substantial part of your extraction is normal Android or embedded-library residue.

Examples include:

```text
http://www.apache.org/licenses/LICENSE-2.0
http://www.apache.org/licenses/
http://www.eclipse.org/legal/epl-v10.html

https://developer.android.com/training/articles/direct-boot
http://schemas.android.com/apk/res/android
http://schemas.android.com/apk/res-auto

https://crashpad.chromium.org/
https://crashpad.chromium.org/bug/new
https://crashpad.chromium.org/truncated

https://firebase-settings.crashlytics.com/...

https://app-measurement.com/a
https://app-measurement.com/s
https://app-measurement.com/s/d

https://android.googlesource.com/toolchain/llvm-project
https://youtrack.jetbrains.com/issue/KT-55980
https://github.com/TooTallNate/Java-WebSocket/...
```

These categories map naturally to:

| String category | Typical origin |
|---|---|
| Apache/Eclipse license links | Bundled open-source license metadata |
| `schemas.android.com` | Android XML/resource namespace |
| `developer.android.com` | Android library/documentation references |
| Crashpad | Crash reporting code |
| Firebase/Crashlytics | Google/Firebase telemetry and crash reporting |
| `app-measurement.com` | Google application measurement libraries |
| LLVM/JetBrains links | Toolchain or Kotlin library metadata |
| Java-WebSocket wiki | Bundled websocket library reference |

Their mere presence inside memory or an APK does **not** make those domains malicious.

A useful analyst rule is:

```text
STRING PRESENT
     ≠
NETWORK CONNECTION OBSERVED
     ≠
MALICIOUS INFRASTRUCTURE
```

For malware triage, indicators should be weighted according to context.

In this dataset:

```text
Apache / Android / Firebase / Crashpad
             ↓
     mostly library residue

jiekou.baojingtextile.cn
             ↓
 documented Hirudo infrastructure

d3q4tlm7pw29zr.cloudfront.net
             ↓
 documented patched-bank APK host
```

That distinction materially improves detection quality.

## Detection and defensive priorities

The defensive challenge with Hirudo is that once the genuine banking client has been replaced, **checks performed exclusively inside that client process can no longer be assumed trustworthy**. ThreatFabric reaches the same conclusion from its reverse engineering. citeturn17search0

### Highest-value endpoint indicators

For mobile threat hunting, the strongest combination is not simply the presence of one dangerous permission. It is the clustering of several events:

```text
unknown/lure APK installed
        +
Accessibility service enabled
        +
notification or screen-sharing capability
        +
banking app unexpectedly removed
        +
banking app reinstalled outside Play Store
        +
certificate/signing identity changes
        +
traffic to Hirudo infrastructure
```

The known Hirudo RAT sample directly exposes Accessibility, SMS, camera, microphone, screen-sharing, APK-installation, persistence, and VPN-related components consistent with this chain. fileciteturn0file0 ThreatFabric independently documents the same functional behavior. citeturn17search0

For EDR, MTD, MDM, or telemetry pipelines, high-value indicators include:

```text
SHA-256:
b2270c29b3d0a49a716730d6ef108cffec0b1a4b159dd02e93333fb85e492b99

Package:
com.r9nkvx.qw2mpl

Related Hirudo package:
com.zk7qpx.mv9wlt
```

ThreatFabric identifies these respectively as Hirudo v7.8.166 and an earlier July build. citeturn17search0 The supplied sample report independently provides the detailed metadata for the first package. fileciteturn0file0

Additional ThreatFabric-reported hashes worth hunting are:

```text
f796eb9ecf516e4f1fafcf89200d9f7d8ddd234f705e63f1bc8fc3ef583f0248
  com.agbbh.kjdrttt
  Shopping lure/dropper

0097a0406be20544e98e981075427f40789db40fe4a53b1b4523383a945328b7
  com.onev.vip
  1V1 VIP lure/dropper

2e45277acfde3fb704179476b08f428f66a03670bbdbf2e8ec79c0b0333c009b
  com.koishort.app
  KoiShort lure/dropper

d51f2bd7ffb49c224d8eaec2c8ca1c8ab9fce2fd5fdceea01eb54296c2f47b4d
  com.zoyatv.app
  ZoyaTV lure/dropper

425cd7072b5e9386ac94e97fe272980b6ef36240df6b5d77b371ab7378004c83
  com.zk7qpx.mv9wlt
  Hirudo RAT

b2270c29b3d0a49a716730d6ef108cffec0b1a4b159dd02e93333fb85e492b99
  com.r9nkvx.qw2mpl
  Hirudo RAT v7.8.166
```

These are published as Hirudo indicators by ThreatFabric. citeturn17search0

### Certificate monitoring is particularly valuable

Because Android requires application updates to maintain compatible signing identities, a bank app that suddenly has the expected package name but **a different signing certificate** represents a high-value anomaly. Android explicitly requires an application's update certificate to match the installed app's certificate or an authorized rotation chain. citeturn16search0

For managed or security-instrumented devices, a strong rule is conceptually:

```text
IF
    package_name == official_bank_package
AND
    signing_certificate != approved_bank_certificate
THEN
    severity = CRITICAL
```

That is more durable than relying only on application label, icon, filename, or version string.

### Bank-side Play Integrity enforcement

Google's application-integrity verdict exposes package identity, certificate digest, and version information, and `PLAY_RECOGNIZED` is specifically associated with an app/certificate combination known to Google Play. citeturn19search9

However, given Hirudo's reported relay/manipulation attempts, the server should bind the integrity result to the actual protected transaction rather than treating it as an independent green flag.

A robust logical model is:

```text
Client requests:
"add beneficiary X / transfer amount Y"

             │
             ▼
Generate Integrity request bound to:
session_id
user_id
beneficiary
amount
transaction identifier
fresh timestamp/challenge

             │
             ▼
Backend verifies:
Integrity response
+ requestHash binding
+ package identity
+ certificate digest
+ version
+ freshness
+ behavioral risk
+ account risk
```

Google recommends using `requestHash` to bind the protected request data into Standard Play Integrity checks. citeturn17search2

The lesson is broader than Hirudo: **an integrity verdict should be one signal in authorization, not the authorization decision itself**.

### Transaction-level controls remain the final trust boundary

ThreatFabric recommends layered behavioral defenses and server-side risk controls because local security controls can be altered when the banking client itself has been patched. citeturn17search0

The strongest controls are therefore those the modified APK cannot simply return `true` for:

```text
login risk evaluation
device re-enrollment checks
credential-recovery controls
new-beneficiary risk
payment velocity
unusual transfer destination
behavioral deviation
step-up authentication
server-side transaction authorization
```

A client can lie about:

```text
"Accessibility is disabled"
"screen capture is blocked"
"my signature is correct"
"integrity passed"
```

but it cannot independently force a correctly designed bank backend to authorize a payment.

### Practical incident-hunting sequence

For a suspected Android device, the most defensible investigation sequence is to establish **identity and chronology**, rather than immediately opening the suspicious APK.

First determine whether any official banking application underwent an unexpected uninstall/reinstall event. Then compare package IDs and signing certificates to trusted bank-distributed versions. Look for the known Hirudo packages and suspicious Accessibility services, followed by network records involving the dedicated CloudFront distribution or `jiekou[.]baojingtextile[.]cn`. Finally, correlate SMS access, screen-sharing services, overlay privileges, camera/microphone use, and boot persistence around the same timestamp. The known sample's manifest and service inventory provide concrete hunting pivots for those behaviors. fileciteturn0file0

The resulting forensic hypothesis would be:

```text
T0  lure installed
 │
T1  fake update shown
 │
T2  Hirudo RAT installed
 │
T3  Accessibility / notification access enabled
 │
T4  target bank app observed
 │
T5  genuine bank app removed
 │
T6  bank-specific APK downloaded from d3q4...
 │
T7  attacker-signed patched bank client installed
 │
T8  injected hooks activated
 │
T9  credential / PIN / screen / SMS collection
 │
T10 fraudulent banking activity
```

That sequence closely matches the replacement and fraud workflow documented by ThreatFabric. citeturn17search0

## Overall intelligence conclusion

The supplied strings materially improve the public understanding of the Hirudo campaign because they appear to expose the **bank-specific object names behind the CloudFront distribution that ThreatFabric publicly describes only as hosting “eight Indonesian banking apps across seven banks.”** citeturn17search0

The resulting attribution is:

| Bank | Target evidence | Assessment |
|---|---|---|
| **Bank Mandiri** | `/livin/latest.apk` + official Livin' branding | **Confirmed with very high confidence** |
| **Bank Tabungan Negara** | `/btn/latest.apk` | **Confirmed with high confidence** |
| **Bank Central Asia** | `/mybca/latest.apk` and `/bcamoblie/latest.apk` | **Confirmed with very high confidence; two apps** |
| **Bank Negara Indonesia** | `/bni/latest.apk` | **Confirmed with high confidence** |
| **Bank Rakyat Indonesia** | `/bri/latest.apk` | **Confirmed with high confidence** |
| **Bank Syariah Indonesia** | `/bsi/latest.apk` | **Confirmed with high confidence** |
| **Bank Jakarta / JakOne ecosystem** | `/jak/latest.apk` + official JakOne branding | **Confirmed with very high confidence** |

The **eight-app/seven-bank count matches exactly** because BCA is represented twice by myBCA and BCA mobile. Official BCA materials confirm that both applications coexist as BCA digital-banking products. citeturn15search3turn15search12

Technically, the campaign represents a significant escalation from ordinary Android overlays. Hirudo first obtains broad control of the victim device through an Accessibility-driven RAT, then selectively removes a genuine banking application and replaces it with a modified copy. The attacker retains much of the real application's functionality while injecting hooks designed to falsify signature checks, conceal Accessibility, defeat screen-capture protection, and interfere with integrity/attestation logic. citeturn17search0

The infrastructure in your memory extraction strongly supports that analysis. The `d3q4tlm7pw29zr[.]cloudfront[.]net` routes correspond to ThreatFabric's documented patched-bank distribution host, while `jiekou[.]baojingtextile[.]cn` independently matches its documented Hirudo configuration API. citeturn17search0 The unrelated Crashpad, Firebase, Android, Apache, Eclipse, Kotlin, LLVM, and WebSocket URLs should be treated as ordinary dependency or framework artifacts rather than campaign IOCs unless runtime evidence demonstrates otherwise.

Most importantly, **none of this implies that Mandiri, BTN, BCA, BNI, BRI, BSI, or Bank Jakarta's core banking infrastructure was breached**. The demonstrated attack model compromises the customer's Android environment and abuses repackaged versions of legitimate mobile applications. The security boundary that remains hardest for this malware to forge is therefore the **bank's server-side authorization and fraud-control layer**, particularly when application identity, certificate state, transaction-bound Play Integrity signals, and behavioral risk are evaluated together. citeturn17search0turn17search2turn19search9
