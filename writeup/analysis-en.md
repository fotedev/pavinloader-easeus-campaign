# From a cracked "EaseUS Partition Master 2026" to a fileless .NET payload

Incident analysis — fotedev, 2026-10-30. English version; Arabic: analysis-ar.md.

## TL;DR

- On 2026-10-28, a user hunting a cracked "EaseUS Partition Master 2026" on an Arabic software site was routed by an injected download button to a fake file-hosting gateway on `*.m5u1iioax.store` (skins: DropVault, Icecloud, GoFiles, WeShare, ShareBox) and downloaded a ~700–800 MB "archive".
- The archive was not EaseUS: a Ren'Py 8.1.3 game-engine bundle with an unsigned `Setup.exe`. Running it installed two fake driver MSIs, "Killer Network Service 2.6" (Qualcomm) and "PhysX Runtime 7.0" (NVIDIA); neither version exists, and the machine had an AMD GPU and a Realtek NIC.
- Each MSI drops a fake "CatalogIngestor" .NET solution and a bundled `MSBuild.exe`: genuinely Microsoft-signed but modified after signing (Authenticode `HashMismatch`).
- The loader uses MSBuild property functions only, no inline tasks: read a 31.6 MB `Resources.resx`, extract Res1–Res4, strip `p` separators, XOR-6 per nibble, rotate at 142,132, `SoapHexBinary.Parse`, then `AppDomain.CurrentDomain.Load(...).CreateInstance("CloneExtensions.FallbackProvider5")`: a fileless ~72 KB payload in memory.
- Malwarebytes named the loader family Trojan.PavinLoader and quarantined 109 objects on 2026-10-29. Co-infections (Trojan.Glupteba, Trojan.Ranumbot, two adware families) predate or parallel this campaign, not necessarily the same actor.
- Entry point: an injected third-party widget on the source page (its real section: 264 plain short-links buttons); the gateway is a cloaked, per-link, step-gated redirect on an apex ~2 days old.

## The chain

```
farescd.com/software/easeus-partition-master/
   |  injected button "تحميل الملف مع التفعيل" (class file-download-btnn, base64 JS)
   |  anti-DevTools + popup-geometry gate + command channel "mr2127a58" (1 h TTL)
   v
*.m5u1iioax.store  (save-njbg / blob-imrx / upload-atd5 / box-tfdg / mirrorgcve)
   |  skins: DropVault - Icecloud - GoFiles - WeShare - ShareBox
   |  "Your file is ready" -> Download File      [Cloudflare; 404 to scanners]
   v
"EaseUS_Partition_Master_2026..._Archive_free_3295.zip"  (~700-800 MB)
   |  Ren'Py 8.1.3 bundle + unsigned Setup.exe  (NOT EaseUS)
   v
two WiX MSIs: "Killer Network Service 2.6" / "PhysX Runtime 7.0"
   |  custom action type 51+242:  cmd /c start "" <Project>.lnk
   v
fake solution "CatalogIngestor": MSBuild.exe (HashMismatch) + 31.6 MB Resources.resx
   |  property functions: strip 'p' -> XOR-6 -> rotate @142132 -> SoapHexBinary.Parse
   v
AppDomain.CurrentDomain.Load -> CreateInstance("CloneExtensions.FallbackProvider5")
   |  FILELESS ~72 KB payload  ->  Trojan.PavinLoader (dllhost / taskhostw tasks)
   +  co-infections: Glupteba, Ranumbot, adware (not necessarily same actor)
```

## Stage 1: the injected button widget

The source page is an Arabic "activated software" library's download page for EaseUS Partition Master. Its genuine download section holds 264 plain `short-links` buttons, with no malicious markers. The captured Elementor markup also contained three copies of an extra button: class `file-download-btnn`, label "تحميل الملف مع التفعيل", carrying a `data:text/javascript;base64,…` script. Whether it arrived via a compromised ad slot or a third-party injector is not established by the evidence (see Attribution note).

Decoded statically, the script:

1. Uses a `87942;` header and a custom string expander to rebuild the final payload script.
2. Detects open DevTools via a `Function('debugger')` timing trap and a `console.log` getter, aborting when either trips.
3. Renders only in a real tab: `window.top === window` and `outerWidth − innerWidth > 160`, never in a small popup.
4. Reads its command channel from `localStorage['mr2127a58']` and cookies `mr2127a58N…` (`timestamp|command`); commands stay valid one hour (`t − ct < 36e5`); stage-2 commands are base64, char-shift −1, run via `Function(...)`.
5. Ends in a decoy tail: `(()=>{const _kpo=1,_nvz=2})();(()=>{let _xfk=0})()`.

Rendering is conditional on all three gates, which explains the "ads disappeared" effect (see Lessons).

## Stage 2: the fake-hoster gateway

The button leads to subdomains of a fresh apex, `m5u1iioax.store` (`save-njbg`, `blob-imrx`, `upload-atd5`, `box-tfdg`, `mirrorgcve`), each serving one of five fake-hoster skins: DropVault, Icecloud, GoFiles, WeShare, ShareBox, with a "Your file is ready" page, fake trust badges, and a Download File button. The pages listed the archive as uploaded 2026-10-27.

Public scan data (urlscan.io, 2026-10-28) for a sibling subdomain shows the cloaking: Cloudflare fronting (AS13335), a TLS certificate issued 2026-10-24, HTTP 404 for the scanner, and a per-link state blob:

```json
{"id":"bxlz4","tag":"6bnS3H2T:C1","file":"Download Link - Archive Free Software And Tools","redirectStep":0}
```

Referred browsers get the funnel; everything else gets nothing.

## Stage 3: the masquerade package

The delivered ZIP (~700–800 MB, `…Archive_free_3295.zip`) contains a full Ren'Py 8.1.3 engine distribution (Python runtimes, a 145 KB RPA game archive), and no EaseUS binaries at all. Root `Setup.exe` is unsigned and version-less, SHA-256 `7123e151…888d4ab`, later flagged by Malwarebytes as Generic.Malware/Suspicious. Run on 2026-10-28, it staged the loader kit and installed both MSIs within minutes; the genuine EaseUS product was installed afterward the same day.

## Stage 4: the fake driver MSIs

Both installers were built with WiX 3.11 on 2026-10-26 from a near-identical template:

| | "Killer Network Service 2.6" | "PhysX Runtime 7.0" |
|---|---|---|
| Manufacturer | Qualcomm Technologies Inc. | NVIDIA Corporation |
| ProductCode | `{96447065-A735-4A79-BA59-A2679E7BAFEF}` | `{3B086246-E935-41F8-BE45-676EF3138112}` |
| Drop path | `%LOCALAPPDATA%\Qualcomm\Killer Network Service` | `%LOCALAPPDATA%\NVIDIA\PhysX Runtime` |
| Launches | `Antlr3Runtime.lnk` | `MasterDataManager.lnk` |

Their only behavioral payload is a custom-action pair (type 51 + 242, sequence 6601/6602, `NOT REMOVE~="ALL"`, install/repair only, not uninstall) executing `cmd.exe /d /s /c "cd /d <drop dir> && start "" <Project>.lnk"`.

The masquerade collapses under two checks. NVIDIA's PhysX System Software has only ever shipped as 9.x, and Killer suites are Rivet-era 1.x or Intel-era 40.x/50.x: neither "7.0" nor "2.6" exists. And the machine's AMD GPU + Realtek NIC gave neither product any reason to be there.

## Stage 5: the loader inside MSBuild

Each MSI drops a fake developer solution: "CatalogIngestor 2.3.0.0" by "Ingestion Labs", with fabricated `DQ-*` changelog entries, decoy NuGet references, and a benign `Directory.Build.props`. Two files matter:

- **`MSBuild.exe`** (4.8.9032.0): genuine Microsoft certificate (thumbprint `5A858500A0262E237FBA6BFEF80FA39C59ECEE76`) but `Get-AuthenticodeSignature` returns `HashMismatch`: modified after signing.
- **`Resources.resx`** (31,618,579 B): deliberately invalid XML (truncated resheaders), so it is only ever read as raw text. Res1–Res4 carry the payload; Res5–Res127 (~31 MB) are decoy mass.

**`Internal.targets`** (3,671 B) has no `UsingTask`, no `CodeTaskFactory`, no target hooks. Everything runs in MSBuild property functions, evaluated during property evaluation, earlier than any target, and less visible to static tooling:

1. `File.ReadAllText` pulls the whole resx into a property.
2. Four regexes extract blobs by aliases defined in *other* innocuous files (`PackageIcon=Res1`, `NuGetAuditSuppress=Res3` in the `.csproj.user`; `PackageTags=Res2`, `Description=Res4` in `Shared.props`).
3. Concatenated blobs lose their `p` per-byte separators; a two-stage character substitution applies XOR-6 to every hex nibble.
4. `Substring(142132) + Substring(0,142132)` rotates the hex stream.
5. `SoapHexBinary.Parse(...).Value` converts hex to bytes.
6. `AppDomain.CurrentDomain.Load(bytes).CreateInstance("CloneExtensions.FallbackProvider5")` loads the assembly in memory and instantiates the type: the fileless entry point.

From stored blob lengths (216,575 characters), the payload assembly is ≈72 KB: derived; never decoded or executed.

## Persistence and follow-ons

A Malwarebytes threat scan on 2026-10-29 quarantined 109 objects and named the loader family Trojan.PavinLoader: fake scheduled tasks `C:\Windows\System32\Tasks\dllhost` and `…\taskhostw` with logon triggers, plus staging directories disguised under `Microsoft\Protect`, `Crypto`, `Themes`, `Libraries`, `Windows\INetCache` (as `ANTLR3RUNTIME`/`MASTERDATAMANAGER`), `WebCache`, `Notifications`, and `Explorer`.

The same scan surfaced co-infections that predate or parallel this campaign, not necessarily the same actor: Trojan.Glupteba (`EpicNet Inc\CloudNet\cloudnet.exe`), Trojan.Ranumbot (a `%TEMP%\csrss` kit: `tor.exe`, `obfs4proxy`, `i2pd`, SMB scanners), and the adware pair PhoenixInvicta (Chrome) + PUP.Optional.SquareStartPage (Edge). A long history of cracked software on the machine is the plausible common denominator.

## Detection reality check

- **VirusTotal (MSI-1, `fc870a5e…3536c5`): 2/62**: Kaspersky `HEUR:Trojan.WinLNK.Agent.gen` (malicious-shortcut family; fits the `.lnk` launch), label `trojan.winlnk`; behavior tag `checks-usb-bus` (undocumented publicly; our inference: sandbox USB/PnP enumeration, a removable-drive propagation precursor); community score −51.
- **Smadav: 0 detections on 1,401 bundle files**: the installed "additional" engine was blind to the whole package.
- **Microsoft Defender:** no detections or Run-key persistence at the time; the scheduled tasks were the only persistence found, quarantined later by Malwarebytes.

## Timeline (dates only)

| Date | Event |
|---|---|
| 2026-10-24 | Gateway TLS certificate issued; apex ~2 days old at the incident. |
| 2026-10-26 | Both fake MSIs built (WiX package timestamps). |
| 2026-10-27 | Fake-hoster pages list the archive as uploaded. |
| 2026-10-28 | Infection day: sibling gateway serves the funnel publicly; `Setup.exe` runs; both MSIs installed; PavinLoader persistence established; genuine EaseUS installed afterward. |
| 2026-10-29 | Ren'Py engine caches written (bundle launched); Malwarebytes quarantines 109 objects; this analysis. |

## Lessons learned

1. **"The ads disappeared" is gating, not cleanup.** Rendering requires no DevTools, real-tab geometry, and a command younger than one hour in `mr2127a58`; when a skin's campaign stops issuing commands, the frame stays blank.
2. **Version numbers are free IOCs.** "PhysX Runtime 7.0" and "Killer Network Service 2.6" never existed; check installs against vendor lines and the actual hardware (no NVIDIA, no Killer here).
3. **A valid signature is not a verdict.** The loader ran under a genuinely Microsoft-signed binary; verify signature *status* — `HashMismatch` means modified after signing — not certificate presence.
4. **Property functions beat inline tasks — for the attacker.** `CodeTaskFactory`-grepping detections find nothing here; watch for `MSBuild.exe` outside Framework directories, tens-of-MB resx files, and `AppDomain.CurrentDomain.Load` in project files (MITRE ATT&CK T1127.001).
5. **2/62 on VirusTotal is expected, not reassuring.** Per-campaign builds keep prevalence near zero; the −51 community score said more than the engine count.
6. **One weak engine is no engine.** Smadav: 0 findings on 1,401 files; layered behavioral tooling: 109 objects.
7. **Capture the page's genuine widget set before attributing.** The 264 real short-links buttons prove the malicious button was injected: on the page, but not of it.
8. **Fake hosters are a commodity kit.** DropVault/Icecloud-style skins are documented across multiple download scenes; block the pattern: fresh domain, random subdomains, 404-to-scanner, per-link state blobs, not just today's domain.

## Artifacts & IOCs

**SHA-256**

```
fc870a5ed14512ab04816681ce7fdab545e455e8e74f76f394486192f33536c5   MSI-1 "Killer Network Service" (VT 2/62)
55c42ed6c1c610b78e99ade2ac9e47648aafdc2d8112c621b4a1f25c2497866d   MSI-2 "PhysX Runtime"
6bc58e85a42bcf60df74c23de212c882d8fdc5e6a60f20f5f3c5e392da6b0ad0   bundled MSBuild.exe 4.8.9032.0 (MD5 2024ddf63513a9ea309091339a84a656)
30d16380654c4cb4eebd6c6cdc5ce1d13c04f2ad3467fd9e1514c7df1471c993   kit Resources.resx (31.6 MB carrier)
e20c494d8d081353f4d188ba88304028496165467486903338142a94b634ef5a   Internal.targets (loader)
7123e1514b939b165985560057fe3c761440a9fff9783a3b84e861fd2888d4ab   Ren'Py bundle Setup.exe (unsigned)
b1f19e6504e31fbe0e3af05da792cb7fac8a9b7427f0928d6d26cdfe75d3c7bf   delivered ZIP
```

**MSI identities:** UpgradeCodes `{392F3156-6442-4173-B0B8-20E198E37E9D}` / `{E147E7CC-0C80-4705-826C-4D9982F08D37}`; PackageCodes `{6230FAE6-CC8B-4372-8054-09EC986D56AB}` / `{FC482E92-9A99-4FAF-BAB8-7C57E01C0720}`; custom actions `SetCA_/CA_c61e54380f464eee81bcd885b78b865f` and `SetCA_/CA_072d6bab34de4524a75d7d80ce1ea576`.

**Network:** apex `m5u1iioax.store`; subdomains `save-njbg`, `blob-imrx`, `upload-atd5`, `box-tfdg`, `mirrorgcve`, `inspectv2lf`; source page `farescd.com/software/easeus-partition-master/` (injected-widget page); Cloudflare AS13335, 188.114.96.3.

**Host artifacts:** `%LOCALAPPDATA%\Qualcomm\Killer Network Service\`; `%LOCALAPPDATA%\NVIDIA\PhysX Runtime\`; `C:\Windows\System32\Tasks\dllhost` and `…\taskhostw`; staging under `AppData\{Roaming,Local}\Microsoft\{Protect, Crypto, Themes, Libraries, Windows\INetCache, WebCache, Notifications, Explorer}`; `%TEMP%\csrss\` (Ranumbot); `%APPDATA%\EpicNet Inc\CloudNet\` (Glupteba); Chrome extension `iaddfgegjgjelgkanamleadckkpnjpjc`; Edge extension `hnjlcpngjhdoehbjfkmhjagmmojcimnc`.

**Strings & file names:** `CloneExtensions.FallbackProvider5`; rotation offset `142132`; XOR-6 nibble map; channel key `mr2127a58`; script header `87942;`; NuGet slug `iqbydwfv`; camouflage "Ingestion Labs" / "CatalogIngestor" 2.3.0.0 / `DQ-*` tickets; `Res1`–`Res127` (aliases `PackageIcon=Res1`, `PackageTags=Res2`, `NuGetAuditSuppress=Res3`, `Description=Res4`); files `build.cmd`, `Antlr3Runtime.lnk`, `MasterDataManager.lnk`, `CatalogIngestor.csproj`, `Internal.targets`.

## Attribution note

The malicious button was an injected third-party widget observed on the source page; the site's genuine downloads are plain short-links, and no public report ties the site itself to malware distribution. The DropVault/Icecloud-style kit operates identically across multiple download scenes — shared commodity infrastructure, not proof of a single actor.

All hashes and rules published in this repository.
