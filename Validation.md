# Validation

## YARA rules (`yara/pavinloader_kit.yar`)

- Engines: **yara-python 4.5.4** and **YARA-X 1.22.1**, identical results.
- Positives: **8 unique incident samples** (10 scan instances; the kit staging copy and one MSI each existed twice byte-identically):

| sample | matching rule(s) |
|---|---|
| `Internal.targets` (loader) | `PavinLoader_MSBuild_Loader_InternalTargets`, `PavinLoader_CatalogIngestor_Kit_Metadata` |
| `CatalogIngestor.csproj.user` / `.Shared.props` | `PavinLoader_CatalogIngestor_Kit_Metadata` |
| `Resources.resx` (31.6 MB carrier) | `PavinLoader_Encoded_Resx_Payload` |
| MSI-1 "Killer Network Service" | `PavinLoader_MSI_Killer_Network_Service_Masquerade` |
| MSI-2 "PhysX Runtime" | `PavinLoader_MSI_PhysX_Runtime_Masquerade` |
| injected button page (`virus-buttons.txt`) | `PavinLoader_Web_FakeDownload_Button` |

- False-positive scan: **1,236 benign files, 938.8 MB, 0 matches** across three corpora:
  Windows `System32` (796 files), `Program Files` + `Program Files (x86)` (400), and the
  YARA project's own test fixtures (40).

## Sigma rules (`sigma/`)

- `sigma-cli` `sigma check` (pySigma 2.0.2): **0 errors, 0 validation issues** (after
  migrating tactic tags to the 2026 MITRE naming: `defense-impairment`,
  `privilege-escalation`).
- `yamllint` (repository config): clean.
- Logsource validation (`tests/test_logsource.py` logic): clean.

## Upstream CI

- VirusTotal/yara AppVeyor (MSVC x86/x64 Debug/Release + Cygwin): **build succeeded** on
  both PRs (#2258, #2259).
- Full GitHub Actions matrix (ASAN/UBSAN, MinGW, `make check` incl. the extended
  `test-dotnet` assertion): **6/6 jobs green on both branches** (run on the fork;
  upstream runs on maintainer approval).

## Known limitations

- The `.lnk` -> `build.cmd` -> bundled `MSBuild.exe` chain is asserted from the MSI
  tables and file set, not executed.
- No upstream regression test covers `yr_compiler_load_atom_quality_table()`; one can be
  added on request.
- Positive-sample testing is inherently circular (the rules were derived from these
  samples); the benign-corpus scan above is the generalization check.
- YARA `guids[0]` equals the MVID for toolchain-produced assemblies; the module does not
  resolve the Module table's `Mvid` heap index. A strict index-based variant is available.
