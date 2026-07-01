# SUSE Longhorn Storage Sizing Calculator

## What this is

Single-file browser app for sizing Longhorn Kubernetes storage clusters.
Calibrated to official Longhorn v1.12 benchmarks (EC2 m5zn.2xlarge, EBS gp3, kbench).

## How to run

Open `index.html` directly in a browser, or:

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

No install, build, or server required.

## Architecture

Everything lives in `index.html` (~2000 lines): HTML structure, embedded CSS, embedded JS.
**Do not split into separate files or add build tooling.**

Two modes:
- **Forward**: hardware inputs → performance predictions
- **Reverse**: target IOPS/throughput → recommended hardware configs (top 3 options)

State persists in the URL hash (base64-encoded JSON). Exports: shareable URL, PDF (print CSS), JSON, CSV.

## Key constants — do not change without a benchmark citation

These are calibrated to Longhorn v1.12 official benchmark data:

- Replica read multipliers (random I/O): 3r = 2.20×, 2r = 1.65×, 1r = 0.50×
- Replica write multipliers (random I/O): 3r = 0.115×, 2r = 0.115×, 1r = 0.29×
- Locality boost: strict-local = 1.30× read throughput (vs 1.00× disabled)
- V2 (SPDK) engine: ~1.8× IOPS, ~0.22× latency vs V1
- Base latency: 750 µs (SSD, V1, 3-replica, best-effort, 1 volume)
- Contention onset: ~50 volumes per 3-node equivalent
- IOPS ceiling scales linearly with node count; bounded by disk IOPS and network bandwidth

## Coding conventions

- Vanilla JS — no frameworks, no TypeScript
- CSS variables for theming; SUSE brand palette: bg `#0d1520`, accent green `#73BA25` (Pantone 368 C)
- All inputs trigger real-time recalculation via `change`/`input` events
- Tooltip pattern: `(?)` hover elements on each parameter

## Git workflow

Feature branches → PR. Conventional commit messages: `feat:`, `fix:`, `chore:`, `docs:`.
