---
title: "A Fake TikTok Shop Served Me a Real TikTok, With a Packer Bolted On"
date: 2026-09-11 20:00:00 +1000
categories: [Malware Analysis, Android]
tags: [android, malware, reverse-engineering, tiktok, packer, infostealer, threat-intel]
description: An AU-targeted fake TikTok Shop handed out a repackaged TikTok APK with a native DEX-packer hidden inside. Here's the full teardown, including the bit I couldn't crack.
toc: true
---

## The short version

I pulled a dodgy "TikTok Shop" lure off the [phishunt.io](https://phishunt.io) feed, poked at it on an isolated REMnux box, and expected the usual: a cheap fake app that asks for your logins and phones home. That's not what I got.

The APK it hands you (`tiktok.apk`) is the **actual TikTok**. Real package, real code, version 44.6.4. Someone took the genuine app, re-signed it with a junk certificate, and bolted a **native DEX-packer** onto the side that decrypts and loads a hidden payload once the app is running. The manifest is clean. Static analysis is clean. Nothing looks wrong until it's live on a phone, which is exactly the point.

I took it apart end to end without ever running it on real hardware, right up to the last locked door: the C2, which sits AES-encrypted inside a fake `.jpg` and won't come out to play in a sandbox. This is the whole walk, including the wall I hit. Every indicator is at the bottom, un-redacted, because that's the only version of threat intel that's actually useful.

> **Where the reporting sits:** landing infrastructure has gone to the ASD/ACSC via ReportCyber, the APK host to AWS abuse, and the iOS redirect to its registrar. IOCs published in full so anyone else chasing this can pivot straight off them.
{: .prompt-info }

---

## Finding it: look before you touch

My process on these is deliberately dull, because dull keeps you out of trouble.

First pass is [urlscan.io](https://urlscan.io) as an **unlisted** scan. urlscan visits from its own infrastructure, so the target logs urlscan's IP and never sees me or my VPN exit. Free look, no attribution.

Second pass is hands-on in REMnux behind a kill-switched VPN, where `curl` *downloads* the source and never runs it. I save the page to the case folder and read it like any other document.

Same four questions every time. What's being impersonated and what gives it away. Where do the creds or the payload actually go. Are there hardcoded keys, tokens or exfil domains sitting in the source. And who's hosting it, because that decides the takedown route.

`au-tiktok-vs[.]com` was two days old, sitting on Alibaba Cloud Hong Kong through an Amazon reseller, wearing a convincing "TikTok Shop" storefront. One of roughly **2,860** near-identical pages in the same batch, so this was a factory, not a one-off.

## The bait and the payload don't live together

The landing page is a Vue single-page app, which changes how you read it. The HTML shell is nearly empty; the logic and the payload URLs get assembled in the JavaScript. `curl` the root and you get almost nothing worth having, so you go straight to the bundle and read the "Try It Now" handler.

It branches on your operating system:

- **Android / desktop** → `hxxps://d55rcvv2c10n6.cloudfront[.]net/tiktok.apk` (payload on AWS CloudFront)
- **iOS** → bounce to `hxxps://tk.1swyc[.]com/SPqdzD`

The setup is tidier than it looks. The throwaway bait page runs on cheap Alibaba infrastructure, but the malware itself comes off **AWS CloudFront**, which is fast, reputable and TLS-clean. Block the bait domain and you've done nothing about the payload host. That's why cleaning this up is a three-party job across Alibaba, AWS and the iOS redirect's registrar, not one polite email.

## The manifest lied to me (sort of)

I pulled the APK down to the bench. An Android APK can't run on Windows anyway, so a volume-mounted work folder is fine for this one, and it's a good habit to build now for the day the sample *is* a Windows `.exe`. Straight to `AndroidManifest.xml`.

This is where it stopped being a boring afternoon. The manifest is the **genuine TikTok**:

- package `com.zhiliaoapp.musically`
- `com.ss.android.ugc.aweme.*` and `com.bytedance.*` components all through it

And it's missing every permission I'd expect from a nasty Android app. No `READ_SMS` or `RECEIVE_SMS`. No Accessibility service. No `REQUEST_INSTALL_PACKAGES`. If you were triaging on permissions alone, you'd file it as "someone sideloading real TikTok" and move on.

That's the trap. The manifest disproved my banker hypothesis, which is a useful thing for a manifest to do, but "no scary permissions declared" is not the same as "safe." It just means the bad code isn't putting its hand up. Two options were left standing: a real TikTok being passed around as an attribution-fraud lure, or a repackaged TikTok with its payload tucked somewhere the manifest never has to mention.

You don't settle that with permissions. You settle it with the signature.

## The certificate doesn't lie

Small gotcha first. `keytool` told me *"Not a signed jar file."* Ignore it. `keytool` only understands the old v1 (JAR) signature scheme and goes blind on the v2/v3 schemes every modern APK uses. `apksigner` isn't packaged on REMnux, so I reached for **androguard** (`APK().get_certificates()` plus `is_signed_v1/v2/v3()`).

Every Android signing cert is self-signed, so issuer-equals-subject means nothing on its own. The question is *whose* cert it is. Real TikTok is signed by ByteDance's key. This one was signed by:

```
CN=TCTDSEK, O=EGUBY, C=DYSZVOVT
SHA-1: 5a80f2b3d22c94dc49dbca91f1c84e38b3bb71ba
(v2 signature scheme only)
```

That's a random-string, auto-generated throwaway. Not ByteDance, not anyone. Put that next to a VirusTotal-unknown hash and MobSF choking on the file (more on that below), and the verdict is in: repackaged and re-signed.

## Diffing against a clean copy finds the payload in seconds

No sense spelunking a 100MB obfuscated app in the dark. I grabbed a **clean TikTok 44.6.4** from APKMirror (whose verified uploads also publish the official signer, which is what let me sanity-check the cert above), extracted both trees, and ran the laziest possible query:

```bash
diff -rq clean/ repack/
```

That coughed up exactly what got injected:

- **Added:** `lib/arm64-v8a/libsakura.so` and `lib/armeabi-v7a/libsakura.so`, a native loader
- **Added:** `assets/config.jpg`, which `file` reports as `data`, high-entropy, no strings. Encrypted payload wearing a photo's file extension.
- **Modified:** `classes21.dex`, `classes24.dex`, `classes28.dex`, `classes32.dex`, `classes49.dex`, hooked to call into the loader
- **Removed:** the Google Play `stamp-cert-sha256`, a dead giveaway that someone's been inside

Two-minute job that would've been an afternoon by hand. The shape is worth filing away too: a native `.so`, a fake-image asset, a handful of tampered `.dex`. That combination shows up again and again.

> A warning I earned the hard way. While I was still deciding how to approach this, MobSF's local static analysis **took off** and pegged ~370% CPU for over an hour, then hung its own web UI. That runaway is itself a signal, it means heavy obfuscation. Kill the container, fall back to `apktool` for the manifest and an online sandbox for behaviour, and don't sit there grinding a packed sample locally waiting for it to finish. It won't.
{: .prompt-warning }

## Sakura: how the payload stays out of sight

The strings and layout of `libsakura.so` lay the whole trick out:

1. `JNI_OnLoad` fires the moment the native lib loads inside TikTok's process.
2. It reads and **decrypts `assets/config.jpg`**.
3. It **reflectively loads a hidden `sakura.*` DEX** into the live TikTok process.
4. That DEX beacons to an **HTTPS C2** (`HttpURLConnection` and `loadUrl` both sitting in the lib).

Now the clean manifest makes sense. The payload runs *inside TikTok's own process* and just borrows TikTok's already-generous permissions: location, `RECORD_AUDIO`, `CAMERA`, `READ_CONTACTS`, `READ_MEDIA_*`, `MEDIA_PROJECTION` screen capture, `SYSTEM_ALERT_WINDOW` overlays. It never has to declare a thing of its own, because its host already asked for everything.

The loader calls itself **"sakura."** One caveat I want to be loud about: **a packer name is not attribution.** Packers get sold and reused across completely unrelated crews. There are "Sakura ransomware" write-ups floating around, and they tell you nothing about what *this* payload does. So I hashed the loader itself and searched VT on that hash specifically, since loaders get recycled even when a full APK is brand new:

```
libsakura.so  SHA-256: 9db6e9bdc70714f9c9ac289aab4802237c0524cd358ff436544815d2d401fb89
              BuildID:  1ff160af78882f5dbbd3e07d39778bdd8b64555f
```

Also unknown to VirusTotal. This is a custom build, not something off the shelf.

## Detonation: a quiet sandbox is not a clean sample

Static had given me everything it was going to, so I sent the APK to **tria.ge** (Recorded Future Triage):

- **Static run** (`260911-2mxpwsfm5w`), score 6/10. Re-confirmed the dangerous permission set and the same random-DN untrusted signer. The malware-config extractor came back **empty**, no known family, which lines up with "custom."
- **Behavioural run** (`260911-2wfsqsfn4t`), score **8/10, "Likely malicious."** Tags: `collection`, `defense_evasion`, `discovery`, `execution`, `persistence`.

The behavioural signatures are where it shows its hand:

- **Collection**: queries *other apps'* accounts (`IAccountManager.getAccountsAsUser`)
- **Discovery**: reads the network operator and active network
- **Persistence**: schedules jobs
- **Defense evasion**: checks for root (`/system/app/Superuser.apk`), reads `/proc/cpuinfo` and `/proc/meminfo`, which is textbook **Virtualization/Sandbox Evasion (MITRE T1633.001)**

And the kicker: **there was no C2 in the network capture.** All ~300 connections were legit TikTok, Google, Facebook, AppsFlyer and Cloudflare, because the real TikTok wrapper carries on as normal. The payload sniffed the sandbox, decided it was being watched, and sat on its hands. MITRE Command-and-Control and Exfiltration cells: empty.

That's not a clean result. That's the malware reading the room and shutting up. A quiet C2 is evasion, not innocence.

## What it is, and the door I couldn't get through

Line it all up. The permission ceiling caps what it *could* do, the behavioural run shows what it *reached for*:

- **Best fit is a modular RAT / infostealer / spyware.** Collection, discovery and persistence, with surveillance-grade reach available for free through TikTok's own permissions (mic, camera, location, contacts, media, screen capture) and overlay potential via `SYSTEM_ALERT_WINDOW`.
- **Not ransomware.** No encryption behaviour, no device-admin, nothing in the Impact column.
- **Not a miner.** Wrong effort-to-target ratio, no mining behaviour anywhere.

What I *didn't* get is the actual C2 address and the exact exfil behaviour. They're behind two locks. First the AES-encrypted `config.jpg` (and it *is* real crypto: `xortool -c 00` found no DEX-magic candidate, so it's not lazy repeating-key XOR). Second the runtime evasion that keeps the payload asleep anywhere it smells analysis.

Two ways to force it, and I'll write up whichever I take next:

- **Static.** Reverse `libsakura.so` in Ghidra, recover the decryption routine, then decrypt `config.jpg` offline. Evasion is irrelevant to a static decrypt, so this is the one I'm leaning towards.
- **Dynamic.** A rooted physical device or a hardened emulator with Frida to dump the DEX after it's decrypted, plus mitmproxy on the beacon. More setup, and you have to beat the anti-analysis checks to get anything.

I'd sooner publish "here's exactly how far I got and where it stopped me" than dress up a half-answer. The wall's the best part of the story anyway.

---

## Indicators of Compromise

**Delivery**

| Type | Value | Notes |
|---|---|---|
| Domain (landing) | `au-tiktok-vs[.]com` | Fake TikTok Shop, Vue SPA; ~2-day-old domain |
| IP | `47.86.105[.]213` | Alibaba Cloud Hong Kong, AS45102 |
| URL (Android payload) | `hxxps://d55rcvv2c10n6.cloudfront[.]net/tiktok.apk` | AWS CloudFront |
| URL (iOS redirect) | `hxxps://tk.1swyc[.]com/SPqdzD` | iOS visitors only |
| Cluster | ~2,860 similar pages | Same campaign |

**The repackaged APK**

| Attribute | Value |
|---|---|
| Package | `com.zhiliaoapp.musically` (TikTok), v44.6.4 |
| Signer DN | `CN=TCTDSEK, O=EGUBY, C=DYSZVOVT` (random throwaway) |
| Signer cert SHA-1 | `5a80f2b3d22c94dc49dbca91f1c84e38b3bb71ba` |
| Signature scheme | v2 only; Play `stamp-cert-sha256` stripped |

**Injected payload**

| File | Detail |
|---|---|
| `lib/{arm64-v8a,armeabi-v7a}/libsakura.so` | Native DEX-packer / loader |
| `libsakura.so` SHA-256 | `9db6e9bdc70714f9c9ac289aab4802237c0524cd358ff436544815d2d401fb89` |
| `libsakura.so` BuildID | `1ff160af78882f5dbbd3e07d39778bdd8b64555f` |
| `assets/config.jpg` | AES-encrypted payload (fake image; high-entropy) |
| Modified DEX | `classes21/24/28/32/49.dex` (bootstrap hooks) |

**Sandbox references**

| Platform | Sample ID |
|---|---|
| tria.ge (static) | `260911-2mxpwsfm5w` (6/10) |
| tria.ge (behavioural) | `260911-2wfsqsfn4t` (8/10) |

## MITRE ATT&CK (Mobile)

- **T1655.001** — Match Legitimate Name or Location (repackaged TikTok)
- **T1407 / T1406** — Dynamic code loading / obfuscated payload (native packer + encrypted asset)
- **T1633.001** — Virtualization/Sandbox Evasion
- **T1417 / T1430 / T1512 / T1429** — Collection & surveillance (accounts, location, camera, audio) *[capability ceiling]*
- **T1637** — Command and Control over HTTPS *[present in the loader, endpoint not yet recovered]*

---

*All of this happened on an isolated REMnux bench behind a kill-switched VPN. Nothing was run on real hardware. If you've seen the same `libsakura.so` build, or you've cracked how `config.jpg` is encrypted, get in touch, I'd love to compare notes.*
