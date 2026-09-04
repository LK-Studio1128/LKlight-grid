# LKlight-grid

**Grid-accelerated CPU docking engine — LKlight-grid v1.2.0.** Full-featured molecular
docking for protein–nucleic-acid and protein–protein complexes, distributed as
single-file, portable binaries for macOS, Windows and Linux.

LKlight is a high-performance Rust reimplementation of
[LightDock](https://github.com/bioinsilico/LightDock)'s GSO (glowworm swarm
optimisation) docking protocol. This project ships the **CPU grid (fast)
variant**: the scoring bottleneck of the original all-pairs implementation — an
all-atom pairwise scan over a 30 Å cutoff — is replaced by a
**≤ 10 Å cell-list near-term + 10–30 Å receptor far-field grid lookup** that is
numerically equivalent for solution ranking.

A sister project, **[LKlight-GPU](https://github.com/LK-Studio1128/LKlight-GPU)**,
provides the CUDA-batched build (Linux + Windows) that re-uses the same
source and drops back to this CPU grid path automatically when no NVIDIA GPU
is present.

---

## Features

- **12 scoring functions** behind one `Score` trait, exposed identically across
  every command: `dfire`, `dfire2`, `dna`, `ddna`, `mj3h`, `pydock`, `cpydock`,
  `sd`, `pisa`, `sipper`, `tobi`, `vdw`.
- **GSO search**: glowworm swarm intelligence with luciferin-mediated neighbour
  selection, adaptive vision ranges, spatial-hash neighbour search (≥ 64
  glowworms) and a fully parallel movement phase.
- **Grid acceleration on all all-atom scorers**: `dna`, `vdw`, `pydock` and
  `cpydock` are all near/far split through one reusable receptor cell list
  (`src/nearcell.rs`); the 8 knowledge-based scorers are lookup-table based and
  fast at any size by construction.
- **Advanced docking controls preserved**: ANM flexibility (normal modes),
  interface restraints, membrane penalty, seeding, output throttling — bit
  compatible with the reference semantics.
- **Verification tooling**: `tools/clash_analyze.py` (pose clash/contact
  audit), `tools/scan_all.py`, `tools/run_parallel.py` (run S swarms with P
  concurrent processes).

## Accuracy & numerical contract

| Guarantee | Value |
|---|---|
| `vdw` grid vs original | bit-identical (0.000 on every tested pose — no far term) |
| `dna` / `pydock` / `cpydock` vs original | far-field interpolation only: ≤ 0.5 % on bound poses, ≤ 12 energy units absolute |
| Solution agreement vs original (same seed) | top-5 poses within ±2 Å: 100 % overlap |
| Ranking correlation vs original | Spearman ≥ 0.9996 |
| GPU build vs this CPU build | < 1e-5 (f32 rounding); see LKlight-GPU |

Since **v1.2.0** the far-field 10–30 Å grid is built at **0.5 Å spacing**
(near-reference resolution) instead of 1.0 Å; on 1AZP the worst-case per-pose
absolute deviation vs. the reference engine drops from ~9.6 to ~6.9 energy
units (~1.4×) with no change to the near-field exact path. Grid build is also
~1.55× faster via shell-band scan pruning, keeping results bit-identical.

The original all-pairs path (`energy_exact`) is retained in the source for
verification and used automatically by ANM runs (each pose deforms the
receptor, so no static field can be cached — a documented design choice, not a
feature gap). Restraint/membrane runs keep full grid acceleration by
collecting interface flags inside the cell near pass.

## Quick start

```bash
# macOS
BIN=release_bin/LKlight-mac-arm64
# Windows
BIN=release_bin/LKlight-win64.exe
# Linux
BIN=release_bin/LKlight-linux-x64
chmod +x $BIN        # (unix)

# 1) Prepare: 6 swarms × 20 glowworms (typical: 25-50 swarms × 200 glow × 100 steps)
$BIN setup receptor.pdb ligand.pdb dna -s 6 -g 20 --seed 42 --noxt --now

# 2) Run every swarm (serial, or parallel — see below)
for i in 0 1 2 3 4 5; do $BIN run setup.json initial_positions_$i.dat 100 dna; done
python3 tools/run_parallel.py $BIN 100 dna 6 6      # equivalent, 6 concurrent

# 3) Rank + export poses
$BIN rank 6 100
for i in 0 1 2 3 4 5; do $BIN generate lightdock_rec.pdb lightdock_lig.pdb swarm_$i/gso_100.out 20; done
```

Swap `dna` for any other scoring function in every command. Useful flags:
`--noxt` (input already has hydrogens), `--now` (drop waters),
`--restraints FILE`, `--seed N` (reproducibility), ANM: `--anm --anm-rec N --anm-lig N`.

## Releases (portable, no toolchain needed)

| File | Platform | Runtime | Notes |
|---|---|---|---|
| `release_bin/LKlight-mac-arm64` | macOS arm64 (+Rosetta x86) | system only | copy & run |
| `release_bin/LKlight-win64.exe` | Windows 10/11 x64 | system UCRT | native MSVC build (Windows Server 2022) |
| `release_bin/LKlight-linux-x64` | Linux x86-64 | **none (static-pie)** | fully static, runs on any glibc/musl host |

Verify a binary in seconds:
```bash
$BIN score <rec.pdb> <lig.pdb> dna --tx 1 --ty 2 --tz 3
# -> Score (DNA): -1xxxx.xxxxx
```

## Building from source

```bash
cargo build --release        # needs Rust stable; binary at target/release/lklight
cargo test --release         # 34 tests incl. grid-vs-original consistency
```

## Performance (RNA system: 8 218 rec + 12 625 lig atoms; 1 swarm × 20 glow × 100 steps)

| Scorer | CPU grid | vs original |
|---|---|---|
| vdw | ~1.0 s (server) | **48×** |
| dna | ~3 s (Mac) | **19×** |
| pydock | ~7.6 s (server) | **15×** |
| cpydock | ~9.7 s (server) | **12×** |

Full benchmark: `PERF_COMPARE_20260902.md`. Deployment notes: `DEPLOY_PLAYBOOK.md`.
Engine-by-engine write-ups: `docs/engines/`. Chinese readme: `README.zh-CN.md`.

## Acceptance runs (2026-09-03, real machines)

| Platform | Hardware | Engine | 1 swarm × 20 glow × 100 steps (1AZP) |
|---|---|---|---|
| macOS | Apple Silicon | grid | 0.45 s |
| Windows | Server 2022 x64 | grid (MSVC) | passed (setup/run/rank, 40 solutions) |
| Linux | 12-core | grid | 1.17 s |

Raw logs: `tests/acceptance/` (in LKlight-GPU) and per-platform files above.

## Known boundaries (not bugs)

- ANM runs on the original all-pairs path by design (per-pose receptor deformation
  invalidates any static field cache).
- Ligands longer than a few hundred Å against a small receptor are geometrically
  pathological for rigid docking; trim the ligand to the binding domain first.
- This project is the CPU grid variant — for NVIDIA-accelerated runs see
  **LKlight-GPU**.

## License

See `LICENSE` / `NOTICE`. LKlight is an independent Rust implementation of the
LightDock docking protocol (https://github.com/bioinsilico/LightDock), which is
released under its own open-source licence.
