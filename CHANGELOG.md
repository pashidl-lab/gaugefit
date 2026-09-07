# Changelog

## 0.1.0 — 2026-09-07

First release.

### Engine
- Subpixel caliper: linear and annular ROI, resample → orthogonal projection →
  smoothed first derivative → subpixel peak.
- Resampling `nearest` / `bilinear` / `bicubic` / `bspline3` (default), subpixel
  `parabola3` / `gauss3` (default) / `centroid` / `zero_cross`, projection
  `mean` / `median` / `trimmed`.
- Cell integration (`super_u` / `super_v`), ROI tilt estimation, exclusion rectangles.
- Edge pairing as explicit policy, not a hidden heuristic: 4 single-edge and 5 pair
  policies, `PairingError` instead of a plausible wrong number.
- Line Finder and Circle Finder with MSAC + Tukey/Huber IRLS. Individual caliper
  failures are absorbed as outliers and reported, never dropped silently.
- Circle Finder corrects the `−(σ_psf² + W²/12)/2R` curvature bias, backing σ_psf out
  of the measured edge width.
- Warning flags on every result: `NEAR_BOUNDARY`, `EDGES_TOO_CLOSE`, `SATURATED`,
  `LOW_CONTRAST`, `TILTED`, plus finder-level `MANY_CALIPER_FAILURES` / `MANY_OUTLIERS`.

### Two implementations
- A compiled C++ core and a pure-Python reference implementation ship together and are
  held to identical results: worst deviation 1.4e-13 px over 768 edges, 60 fits and 6
  finder runs. `use_native(False)` switches at any time.
- RANSAC is deterministic in both — lexicographic unranking of the combinations, no RNG
  — so two languages see the same subsets. Thread count changes speed only.
- Runtime SIMD dispatch (AVX2+FMA / NEON / scalar) chosen by CPUID, so one wheel runs
  on old and new machines.

### Around it
- Calibration applied to measured points, never by resampling the image;
  `undistort_image` exists only to say so.
- Error budget backed by the Phase 0 bias sweep, shipped as `data/bias_table.csv`:
  your σ, interpolation and SNR look up numbers that were actually measured.
- Recipe: one self-contained JSON with tools, measurements, fixture, calibration and a
  base64 reference image. `load_recipe(...).run(img, pose=...)` is the whole runtime API.
- Dashboard (`gaugefit serve`) for setting tools up and exporting a recipe;
  `fetch_recipe()` pulls it straight from a running dashboard.
- CLI: `demo`, `serve`, `measure`, `budget`, `info`, `check`.
- 90-day evaluation, decided in the compiled core. The pure-Python reference
  implementation is not gated.

### Documentation
- Integration guide, bias characteristics and operations guide, in English and Korean,
  under `docs/`. `tests/test_docs.py` holds them to the code: every name the API reference
  promises must exist, both editions must have the same sections, and the documented flags,
  policies and recipe types must match what the package actually exports.
