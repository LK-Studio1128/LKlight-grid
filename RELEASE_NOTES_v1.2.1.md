# LKlight-grid v1.2.1

**Nucleic-acid scoring safety patch for the grid CPU engine.** This patch
prevents unsupported scorers from returning a successful-looking constant zero
score on protein–DNA / RNA inputs.

## Fixed

- Nucleic-acid inputs are validated before scoring. Only `dna` and `ddna` are
  accepted; incompatible methods fail clearly with exit code 2.
- Worker-thread failures are reflected in the process exit status.
- The 1AZP grid DNA regression test now uses a relative tolerance consistent
  with the 0.5 Å interpolated grid path while preserving the expected energy.

## Validation

- `cargo test --release --lib`: **34 passed**.
- Real protein–DNA regression (`MYB46-ALL.pdb` + `proCDH653-1137.pdb`,
  `score ... dna`): **-5186.346373**, not zero.
- A 1 swarm × 20 glowworms × 20 steps DNA docking produced a varying ranked
  score range (37.41–49.24) on Linux; macOS reproduced
  `-5186.346373`.

## Binaries

- `LKlight-grid-v1.2.1-linux-x86_64` — Linux x86-64
- `LKlight-grid-v1.2.1-mac-arm64` — macOS Apple Silicon
- `LKlight-grid-v1.2.1-win64.exe` — Windows x64
