# LKlight-grid v1.2.0

**0.5 Å far-field grid (reference-level accuracy) + ~1.55× faster grid setup.**

## Added

- **Far-field grid at 0.5 Å reference resolution (new default).** The 10–30 Å
  receptor potential grid is now built at 0.5 Å spacing instead of 1.0 Å, and
  the box half-width is derived as `spread = ceil(FIELD_RMAX / spacing) + 2`
  (the previous hard-coded ±32 cells assumed 1 Å and would have truncated the
  field at 0.5 Å). The exact ≤ 10 Å near-field path is unchanged. On the BM5
  1AZP complex the worst-case per-pose absolute far-field deviation vs. the
  reference LKlight engine drops **9.60 → 6.94** energy units (~1.4×).

## Changed

- **Shell-band box scan for faster grid setup (bit-identical).**
  `ReceptorField::build()` prunes `(y,z)` rows already outside the shell and
  visits only the contiguous run of `x` slabs that can intersect the 10–30 Å
  band (±1-cell margin). Every written cell still passes the same
  `d² > r_min² && d² ≤ r_max²` test, so the written-cell set and order — and
  therefore the `f32` field — are bit-identical to v1.1.0. Grid build + first
  score: 1.57 s → 1.02 s at 0.5 Å (~1.55×).

## Accuracy contract (unchanged semantics)

- `vdw` grid vs original: bit-identical (no far term).
- `dna` / `pydock` / `cpydock` vs original: far-field interpolation only,
  now with the residual absolute deviation roughly halved by the 0.5 Å grid.
- Solution agreement (same seed) and Spearman ranking unchanged.

## Downloads

| Platform | File | Binary |
|---|---|---|
| macOS Apple Silicon | `LKlight-grid-v1.2.0-mac-arm64.zip` | aarch64, native |
| Linux x86-64 | `LKlight-grid-v1.2.0-linux-x86_64.tar.gz` | musl static-PIE |
| Windows x64 | `LKlight-grid-v1.2.0-win-x64.zip` | PE32+ console |

Each archive contains the binary plus `README.md`, `LICENSE` (GPL-3.0),
`NOTICE` and `CHANGELOG.md`.

## Citation & DOI

Zenodo DOI from v1.1.0 is retained (same DOI is used for this patch-minor
release on the GitHub side). See `NOTICE` for LightDock attribution.
