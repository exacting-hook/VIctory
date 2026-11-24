# RTX 4070 Ti on Windows 11: test results

A data point for NVIDIA on Windows, where the README's numbers come from one AMD RX 6700 XT. It covers
`cargo test`, the shader census with host pipelines, and five runs of the title. Every number below is
measured on this one machine; nothing here comes from the title's files except counts.

| | |
|---|---|
| Commit | `8e79c80` (2026-11-24) for the tests, the census and the last two runs; `d2a4bae` for the first three runs |
| GPU | NVIDIA GeForce RTX 4070 Ti (Ada, 12 GB), driver 591.86 |
| CPU, RAM | AMD Ryzen 9 5900X, 32 GB |
| OS | Windows 11 Pro N 25H2, build 26200 |
| Toolchain | Rust 1.98.1 (`rust-toolchain.toml`), MSVC; DXIL validator from Windows SDK 10.0.26100 |

## `cargo test --workspace`

311 passed, 0 failed, 6 ignored, over 23 test binaries. No game data and no `victory.toml`.

## Shader census: `victory translate --validate --pipelines`

26,599 containers (614 duplicates skipped), 26,148 programs; all 26,148 translated and structured;
25,502 have every resource traced.

| Stage | Programs | DXIL emitted | Validated | Host pipelines created |
|---|---:|---:|---:|---:|
| Vertex and pixel | 14,013 | 12,236 | 12,160 | 12,236 |
| Compute | 2,581 | 1,440 | 1,417 | **1,121** |
| Mesh | 8,506 | 8,163 | 8,101 | 8,163 |

**319 compute pipelines are refused by the driver** with `The parameter is incorrect.` (`E_INVALIDARG`):
every refusal in the census is a compute pipeline, and no vertex, pixel or mesh pipeline is refused. This is
the same error #468 reports for 114 compute pipelines on an RTX 4060 Laptop while the world loads; here it
is measured over the whole census rather than at run time. Which descriptor or root-signature property
the driver rejects is not established; a D3D12 debug-layer run of one refused program would say.

The largest refusals in the census output (VIctory's own messages):

| Count | Reason |
|---:|---|
| 653 | emit: a 64-bit image atomic |
| 604 | emit: a scalar memory read as data |
| 388 | emit: a descriptor read only as data |
| 319 | pipeline: The parameter is incorrect. |
| 298 | resources: a descriptor neither loaded nor built |
| 288 | emit: a select on an untracked mask |
| 272 | emit: a lane mask holding lanes elsewhere read as data |
| 248 | emit: a lane mask read as data |
| 247 | resources: an address in an SGPR that is no root parameter |
| 207 | I/O: an interpolation with computed barycentrics |

The DXIL validator refuses 161 emitted programs: 53 mesh programs whose vertex plus primitive output
exceeds 32,768 bytes, 85 over signatures (an interpolation mode that differs from another element packed
into the same row, or an input or output signature part that does not match the module), and 23 with an
operation incompatible with the shader stage. Those are validator rules, not driver behaviour, so they
should read the same on any vendor.

## Runs of the title

Four runs of the release build and one of the diagnostics build, the longest about 9 minutes, all in the
title's first screens and menus:

- Boot to first frame in about 2 s; menus at a steady 29.9-30.0 fps (33.3 ms/frame) with 3-53 draws per
  frame and no draw dropped for a missing pipeline. GPU time per frame was 1.1-11 ms in 144 of 166
  five-second samples, 11-28 ms in 21 and 128.9 ms in one.
- Pipelines saved on the first run were loaded on the next (`host pipeline library loaded`), so later
  boots translate nothing for these screens.
- Saves work: the title created its save container on the third run.
- No crash, no `crashes.log`, no device removal in any run. Every run ended by closing the window
  (`title did not finish suspending in time; ending the title`, exit code 0).

Warnings logged on every run, from the host side:

- `host view heap capped at 1000000` for a requested 1,245,184 descriptors (`E_INVALIDARG` at the
  requested size).
- `no host texture` (`E_INVALIDARG`) for two 2560 x 1440 textures, DXGI formats 86 (flags 5) and 115
  (flags 1).
- `method not implemented` for device slots `0x218` and `0x1c8`.

The world was not reached. A diagnostics-build run with `VICTORY_KEYS` pressing Enter every 5 s from 15 s
to 150 s stayed on the start screen, so the NVIDIA world hangs in #314, #325, #496, #619 and #625 are not
confirmed or ruled out on this card.

AI-assisted: Claude Code built both profiles, ran the tests and the census, and wrote this file from their
output.
