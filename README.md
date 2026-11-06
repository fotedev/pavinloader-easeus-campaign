# PavinLoader / "EaseUS Partition Master 2026" fake-crack campaign

**Author:** [fotedev](https://github.com/fotedev) · **First seen:** 2026-10-28 · **Published:** 2026-10-29
**Status:** public detection pack — hashes, network IOCs, YARA & Sigma rules, bilingual analysis.
**No malware binaries are hosted in this repository** — only hashes, patterns, and analysis.

A commodity fake-crack download flow delivered a fileless MSBuild loader (Malwarebytes: **Trojan.PavinLoader**) to a Windows 10 machine. The chain: Arabic software site → **injected download button** → fake file-hoster gateway on a 2-day-old `.store` apex (Cloudflare-cloaked, 5 rotating skins) → a **Ren'Py 8.1.3 engine bundle masquerading as EaseUS Partition Master** (unsigned `Setup.exe`) → two WiX-built **fake driver MSIs** ("Killer Network Service 2.6" / "PhysX Runtime 7.0" — neither version exists) → bundled **MSBuild.exe (genuine Microsoft signature, modified after signing — HashMismatch)** → MSBuild **property functions** (no inline tasks) decode a ~72 KB .NET payload from a 31.6 MB `Resources.resx` and execute it **fully in memory**.

## Contents

| Path | What it is |
|---|---|
| [`writeup/analysis-en.md`](writeup/analysis-en.md) | Full technical write-up (English) |
| [`writeup/analysis-ar.md`](writeup/analysis-ar.md) | نفس التحليل بالعربية |
| [`iocs/iocs.csv`](iocs/iocs.csv) · [`iocs/iocs.json`](iocs/iocs.json) | Structured IOC pack (hashes, domains, MSI product codes, loader strings) |
| [`yara/pavinloader_kit.yar`](yara/pavinloader_kit.yar) | YARA rules for the loader kit, fake MSIs, and injected button script (tested, see test report) |
| [`sigma/`](sigma/) | Sigma rules: MSBuild-from-user-paths, `MSBUILDENABLEALLPROPERTYFUNCTIONS`, fake task names, msiexec→cmd→.lnk |
| [`Validation.md`](Validation.md) | Test matrix and false-positive scan results |

## Key IOCs (subset — full list in `iocs/`)

```
fc870a5ed14512ab04816681ce7fdab545e455e8e74f76f394486192f33536c5   MSI "Killer Network Service"
55c42ed6c1c610b78e99ade2ac9e47648aafdc2d8112c621b4a1f25c2497866d   MSI "PhysX Runtime"
6bc58e85a42bcf60df74c23de212c882d8fdc5e6a60f20f5f3c5e392da6b0ad0   tampered MSBuild.exe 4.8.9032.0
7123e1514b939b165985560057fe3c761440a9fff9783a3b84e861fd2888d4ab   Ren'Py fake "EaseUS" Setup.exe
*.m5u1iioax.store                                                  fake-hoster gateway (Cloudflare)
CloneExtensions.FallbackProvider5                                  in-memory payload type
MR2127A58 (case-insensitive)                                       injected button command-channel key
```

## Why this matters

- **2/62 on VirusTotal** at detection time: fresh, per-campaign, cloaked infrastructure keeps samples below vendor thresholds — IOC sharing is the practical defense.
- The loader avoids inline C# tasks entirely, riding **property functions** that evaluate earlier than any `<Target>` hook — a blind spot for rules that only look for `CodeTaskFactory`.
- The "ads disappeared / gray screen" behavior of fake-hoster campaigns is **by design**: the button script renders only with a fresh command in its channel (1-hour TTL).

## Rule Validation

- YARA rules: verified on yara-python 4.5.4 and YARA-X 1.22.1 against incident samples;
  benign-corpus scan of 1,236 files (System32, Program Files, YARA test fixtures) with 0
  matches.
- Sigma rules: `sigma check` 0 issues, yamllint clean, logsource validation clean.
- Details and per-sample matrix: [`Validation.md`](Validation.md).

## Attribution & license

- Incident analysis, rules and write-up: **fotedev**, 2026-10-29.
- Licensing map ([LICENSE](LICENSE)): **CC0-1.0** for IOC data · **DRL-1.1** for YARA/Sigma rules (matching the community rule repos) · MIT for the rest. No warranty; verify before blocking.

## Contact

Open an issue, or via GitHub [@fotedev](https://github.com/fotedev).
