# gaugefit

Subpixel caliper metrology for machine vision — **measure a width, a gap, a diameter,
a straightness**, and know how wrong the answer can be.

[![PyPI](https://img.shields.io/pypi/v/gaugefit.svg?cacheSeconds=3600)](https://pypi.org/project/gaugefit/)
[![Python](https://img.shields.io/pypi/pyversions/gaugefit.svg?cacheSeconds=3600)](https://pypi.org/project/gaugefit/)
[![Platform](https://img.shields.io/badge/platform-linux%20%7C%20macOS%20%7C%20Windows-lightgrey.svg)](https://pypi.org/project/gaugefit/#files)
[![License](https://img.shields.io/badge/license-Proprietary-red.svg)](#license)

Put a rectangle across an edge. gaugefit resamples the strip, projects it into a 1-D
profile, differentiates, and reports where the edge is to a few thousandths of a pixel —
with the flags that say when not to trust it.

This is the Cognex Caliper / HALCON `measure_pos` family of tool: a linear or annular
ROI, edge pairing policies, Line and Circle finders that absorb individual caliper
failures as outliers, distortion and scale calibration, and a fixturing transform so a
recipe follows the part instead of the frame.

**A C++ core behind a Python API that needs only numpy.** No OpenCV. A caliper measurement
takes about 58 µs; a 28-caliper circle finder with robust fitting takes 0.16 ms.

```bash
pip install gaugefit
gaugefit demo            # runs on a bundled sample, nothing else needed
gaugefit serve           # the dashboard: set it up, test it, export a recipe
```

---

## Two implementations, the same numbers

The package ships **both** a compiled core and a pure-Python reference implementation,
and they are held to producing identical results — not similar ones.

| what | cases compared | worst deviation |
|---|---|---|
| linear caliper (4 interp × 4 subpixel × 3 projection × 3 polarity × …) | 768 edges | **1.4e-13 px** |
| annular caliper | 16 | 7.1e-15 |
| line / circle fitting, robust, with outliers | 60 | 1.1e-14 |
| Line / Circle finder end to end | 6 | 2.3e-13 |

That is what makes the reference implementation useful rather than decorative: it is the
specification, it is readable, and any disagreement is a bug in one of them.

```python
import gaugefit as gauge

gauge.engine_info()        # {'engine': 'native', 'simd': 'AVX2+FMA', 'threads': 32, ...}
gauge.use_native(False)    # fall back to the reference implementation at any time
```

RANSAC is deterministic in both: samples come from lexicographic unranking of the
combinations, not from a random number generator, so two runs — and two languages —
see the same subsets. Thread count changes the speed and nothing else.

---

## Measure something

```python
import gaugefit as gauge

img = gauge.imread("part.png")

# one caliper: put it across the edge, 90 px long, 20 px wide
cal = gauge.Caliper(center=(320, 240), angle_deg=0, length=90, width=20)
res = cal.measure(img)
for e in res.edges:
    print(e.t, e.point, e.polarity, e.contrast)

# a width: pick the outermost opposite-polarity pair
pair = cal.measure_pair(img, policy="first_last", pair_polarity="opposite")
print(pair.width)

# a line, from 15 calipers, with the bad ones thrown out
line = gauge.LineFinder(p0=(100, 80), p1=(100, 400), n=15, length=40, width=10).run(img)
print(line.shape, line.rms, line.span)      # span is straightness

# a circle, from 24 annular calipers, curvature-corrected
circ = gauge.CircleFinder(center=(300, 300), r_nominal=61, n=24, r_span=30).run(img)
print(circ.shape.center, circ.shape.radius, circ.span)   # span is roundness
```

Every result carries flags (`NEAR_BOUNDARY`, `EDGES_TOO_CLOSE`, `SATURATED`,
`LOW_CONTRAST`, `TILTED`) and a diagnostic dict — the total edge width, the lens PSF σ
backed out of it, the ROI tilt. Those are what let a line stop and say *why* rather than
returning a plausible wrong number.

---

## Defaults chosen by measurement, not by taste

Before the engine was written, a harness rendered edges with **analytically
anti-aliased** coverage — the ground truth is known to machine precision — and swept
subpixel phase, edge angle, projection width, noise, curvature and adjacent-edge spacing.
The defaults come out of that table:

- **`subpixel="gauss3"`** (log-parabola on three samples) beats `parabola3` by roughly
  **350×** on phase-dependent bias: 0.00009 px peak-to-peak vs 0.032 px at σ = 1.5.
- **`interp="bspline3"`** is the only resampler that matters. `bicubic` is
  indistinguishable from `bilinear` (0.031 vs 0.030 px); B-spline is **10× better**
  (0.0030 px).
- An **axis-aligned edge is the worst case**, not a benign one. Tilt it 5° and bias drops
  by an order of magnitude.
- Curvature bias is `−(σ_psf² + W²/12) / 2R` — it comes from the 2-D blur and the
  projection width, and is **independent of the caliper σ**. The circle finder backs
  σ_psf out of the measured edge width and corrects for it.
- Adjacent edges interfere out to **5σ**, not the 3σ usually quoted.
- Keep `W·tanΔ < 3σ`, where Δ is ROI misalignment: that product, not either factor, is
  what smears the projection.

The full tables, with the conditions attached, are in [`docs/bias.md`](https://github.com/pashidl-lab/gaugefit/blob/main/docs/bias.md) — including what happens when you ignore them.

---

## The dashboard

```bash
gaugefit serve
```

Set the tools up on a real image, watch the profile and the derivative under the ROI you
are dragging, add measurements, and export a **recipe** — one self-contained JSON file
holding the tools, the measurements, the fixture, the calibration and a reference image.
Then, from your own application:

```python
import gaugefit as gauge

recipe = gauge.load_recipe("part.gaugefit.json")     # or gauge.fetch_recipe() from the dashboard
out = recipe.run(img, pose=match)                # pose from geofit, or None
print(out.values, out.units, out.flags)
```

Nothing about the recipe is tied to the dashboard: it is a file, and `run()` is the whole
runtime API.

---

## Install

```bash
pip install gaugefit
```

Python 3.10 – 3.14 on Windows x86-64, Linux x86-64 / aarch64, macOS arm64.
The only runtime dependency is **numpy**.

PNG and PGM need nothing else. Other formats (JPEG, TIFF, …) go through Pillow if it is
installed:

```bash
pip install "gaugefit[image]"
```

The wheel is tagged `py3-none-<platform>`: it does not link against the CPython ABI, so
one file covers every supported Python and keeps working when a new one is released.
The AVX2 path is selected at runtime by CPUID, so the same wheel runs on older machines.

---

## Distortion is applied to points, not to images

There is no `undistort_image()`. Undistorting an image means resampling it, and
resampling destroys exactly the subpixel information the caliper is there to read. So
lens distortion and the pixel-to-world scale are applied to the **measured coordinates**,
after the fact:

```python
from gaugefit.calib import calibrate_from_dot_grid

cal = calibrate_from_dot_grid(target_img, rows=9, cols=11, pitch=2.0)
# or, when a single scale factor is enough:
cal = gauge.Calibration.from_scale(px_per_unit=41.7, unit="mm")

world = cal.to_world([e.point for e in res.edges])
print(cal.distance(world[0], world[1]), cal.unit)
```

`gaugefit.calib.distortion.undistort_image` exists only to raise `NotImplementedError`
with that explanation.

---

## Uncertainty

```python
from gaugefit.quality import caliper_budget

caliper_budget(sigma=1.5, psf=0.8, angle=0, width=20, snr_db=35)
# -> per-term contributions and a combined 1σ, in pixels
```

A repeatability number and an accuracy number are different claims, and the budget keeps
them apart: noise-driven scatter, phase-dependent estimator bias, resampling bias,
curvature and projection-width terms are reported separately, so a spec argument can be
settled by pointing at a row.

---

## Documentation

| | |
|---|---|
| [Integration guide](https://github.com/pashidl-lab/gaugefit/blob/main/docs/integration.md) | putting gaugefit inside your application ([한국어](https://github.com/pashidl-lab/gaugefit/blob/main/docs/integration.ko.md)) |
| [Bias characteristics](https://github.com/pashidl-lab/gaugefit/blob/main/docs/bias.md) | what it gets wrong, measured, with conditions ([한국어](https://github.com/pashidl-lab/gaugefit/blob/main/docs/bias.ko.md)) |

Everything, including PDF editions: https://github.com/pashidl-lab/gaugefit

---

## Licence

gaugefit is distributed for **evaluation**: 90 days from first use on each machine.
See [LICENSE](https://github.com/pashidl-lab/gaugefit/blob/main/LICENSE).

```bash
gaugefit check       # licence / evaluation status, and where the key would go
```

For a licence key, write to **pashidl.lab@gmail.com**. A key is a signed statement
verified offline — there is no server and nothing is transmitted anywhere. Install it
with `GAUGEFIT_LICENSE` or by saving it to `~/.gaugefit/license`.

The evaluation limit applies to the compiled core. The pure-Python reference
implementation is not gated — it ships readable in the wheel, and pretending otherwise
would be theatre.

---

## See also

**[geofit](https://pypi.org/project/geofit/)** — rotation- and scale-invariant shape
matching from the same authors. geofit finds the part, gaugefit measures it, and
`gauge.Fixture.from_geofit(match)` connects the two so a recipe follows the part around the
frame.
