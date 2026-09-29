# SIH26072: thunderstorm and lightning nowcasting for IMD (submission 30 Sep 2026)

The code is done; the remaining work is running it and presenting results.
- **Follow `STEPS.md` in order.** Background is in `README.md` and `docs/PLAN.md`.
- **Don't refactor or add features without asking.**

## Problem
- SIH26072 (MoES / IMD): nowcast thunderstorms and lightning from radar, satellite, lightning and NWP data.
- We forecast **0–6 h**. IMD's operational nowcasts cover 3 h, and radar extrapolation loses to NWP after ~2 h.
- It must run on Indian data.

## Design
### Training data
- **SEVIR** (`s3://sevir`): NEXRAD VIL, GOES-16 IR 6.9/10.7 µm and GLM lightning, co-registered.
- **HRRR** for NWP. The field valid at H comes from the run initialised at H−fxx (no leakage).
- **7,500 events**, split by date: train < 2019-01-01 ≤ val < 2019-06-01 ≤ test.

### Tier 1 (0–3 h): `model/fusion.py`
- **Inputs:** 30 min of history (7 × 5 min). **Outputs:** 18 steps × 10 min.
- **Architecture:** one stem per source, a U-Net, and NWP cross-attention plus FiLM.
- **Modality dropout** p = 0.15.
- **Outputs:** VIL at 2 km; lightning probability at 8 km.

### Tier 2 (lead hours 1–6): `extended.py`
- HRRR f02–f07 post-processing on a 16 km grid, trained against GLM and VIL.
- Baselines: raw HRRR LTNG and REFC.

### Seamless product
- `scripts/crossover.py` scores both tiers per lead hour at 16 km.
- The product uses tier 1 up to the switch hour and tier 2 after it.

### India
**No open Indian lightning archive exists:** IITM ILLN is on request, and FY-4A LMI and GLM don't cover India.

**Training-time adaptation**, applied to 50% of samples:
- `insat_like`: IR 4 km, WV 8 km, 15/30-min scans.
- `gfs_like`: no LTNG/UH, ~24 km.
- `--india-mode` scores a model on those inputs.

**Adapters in `india/`:**
- INSAT-3DR/3DS L1B (MOSDAC)
- **GK2A from NOAA's open bucket, no login**, verified
- GFS, verified
- IMD dBZ
- strike CSV and ISS-LIS

**India CLI:** `python -m nowcast.india.run --city X --time T --gk2a --tier1 ... --tier2 ... --switch-hour N`

## Hardware
- **Primary (4060 Ti 16 GB):** data build, overnight tier 1 (`configs/dev.yaml`), tier 2, evaluation, India runs.
- **Burst (2× 5090, 3 h, 16 GB SSD):** `scripts/burst.sh all`.
  - Everything is kept in `/dev/shm`.
  - torch must match the primary's version exactly, with cu128.
  - If the burst fails, use the overnight model.

## Verified facts (don't change without re-verifying)
- SEVIR rows are **south-up** (`ROW0_NORTH = False`).
- Lightning x/y are in 48-grid pixels. Binning is (T_k − 5 min, T_k].
- The catalog's `minute_offsets` has typos; `frame_offsets()` repairs them.
- HRRR sfc has no `RH:700 mb` → use 700 mb dewpoint depression.
- GFS lacks LTNG, MXUPHL and VUCSH/VVCSH. Shear = V500 − V10m.
- `--resume` keeps the checkpoint's horizon (`data.t_in / t_out / out_step`).
- The INSAT reader works on real MOSDAC `3RIMG_L1B_STD` files (checked 29 Sep 2026).
  - It agrees with GK2A on the same tile: correlation 0.87 (IR) and 0.84 (WV); INSAT reads ~2.7 K warmer.
  - `Acquisition_Start_Time` is ~35 s after the nominal time in the file name.

## Not yet verified or done
- The ISS-LIS converter on real files.
- The pySTEPS baseline (cut).
- Verification against Indian lightning: ILDN data has been requested.

## Rules
- **Cut order if short on time:** pySTEPS → tier-2 hours 5–6 → radar-only control → burst.
- **Never cut:** the scorecard, one India run, and the honest label.
- **Never commit** `shards/ work/ runs/ ext/ *.pt`. Run `pytest -q` (38 pass) before committing.
- **Report numbers as measured.** Compare models at equal `samples`.
- **Always say:** "trained on US GLM lightning, adapted to INSAT/GFS, not yet verified against Indian lightning observations".
- **India maps** use Survey of India boundaries (official depiction), never Natural Earth.
  - The file is `data/boundaries/india_states.geojson`: SoI OVSF/1M/7 state boundaries, reprojected to lat/lon.
  - It is committed (derived, simplified, attributed); the raw SoI zip is not. Bhuvan only serves boundaries as images (its WFS is disabled).

## Team workflow
- Commit with `/commit-and-push`. It asks which branch to push to and shows the message first.
- Commit messages are short and plain. No `Co-Authored-By`, and no mention of Claude, AI or the model.
- The website is built in `web/` on its own branch (`romir-web`). Web branches never edit main-owned files (everything outside `web/`).
- `site/` is the static data for the site: `cases.json`, plus the `public/data/` output of `scripts/export_site.py`. Don't touch it without prior permission.
