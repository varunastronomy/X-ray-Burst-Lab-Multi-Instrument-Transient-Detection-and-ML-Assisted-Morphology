# X-ray-Burst-Lab-Multi-Instrument-Transient-Detection-and-ML-Assisted-Morphology
Developing a Python framework for detecting and characterizing X-ray bursts across instruments. The prototype generates synthetic light curves and detects anomalies in NICER data. Planned work includes multi-mission event ingestion, ML-based detection, burst morphology analysis, and validation against real observations

# X-ray Burst Lab: Project Status

## Purpose

This project is being built to find anomalous intervals in high-energy X-ray
observations and, later, classify candidates as Type I X-ray bursts, Type II
bursts, accretion flares, magnetar-like bursts, GRB-like events, or
instrumental artifacts. The eventual science workflow is:

```text
Mission data -> common light curve -> anomaly detector -> candidate intervals
             -> candidate classifier -> Type I burst morphology analysis
```

The first implementation is intentionally smaller than that end goal. It is a
working prototype for simulation, labeling, light-curve ingestion, and
statistical candidate discovery.

## Decisions Made So Far

The current design reflects these choices:

- Initial reference instrument: NICER/XTI.
- Initial input representation: light curves only, rather than photon energies
  or event lists.
- Primary simulation time bin: 0.25 s.
- Initial source templates: 4U 1636-536, 4U 1608-52, and XTE J1810-189.
- Initial simulation profiles: transparent phenomenological shapes, not
  physical spectral or instrument-response simulations.
- Initial observation lengths: short NICER-like snapshots of 0.5-3 ks.
- Realistic observing effects already represented in simulation: GTI gaps and
  changing background rate.
- Initial detector objective: high recall, so it is preferable to flag extra
  candidates rather than miss transient events.
- Long-term evaluation target: at least 95% recall on a held-out synthetic
  dataset, with interval boundaries within 1 s.

## What Has Been Implemented

### 1. Synthetic light-curve generator

`src/xray_burst_lab/simulation.py` creates short, NICER-like count-rate light
curves at 0.25 s resolution. It randomly selects a source template, persistent
rate, variability, background level, GTI gaps, and transient parameters.

The first smoke-test dataset contains ten observations:

| Observation type | Number |
|---|---:|
| One Type I-like transient | 1 |
| One Type II-like recurrent transient | 1 |
| One accretion-flare-like transient | 1 |
| One magnetar-like short transient | 1 |
| One GRB-like multi-pulse transient | 1 |
| Type I plus artifact | 1 |
| Flare plus artifact plus GRB-like event | 1 |
| Artifact only | 2 |
| Quiet observation | 1 |

The synthetic transient shapes are useful for testing software flow, labels,
and model interfaces. They must not yet be interpreted as physically faithful
representations of real sources.

### 2. Labels and inspection products

Each simulated observation has three products:

- CSV light curve: time, counts, rate, background estimate, GTI flag, and
  per-bin active labels.
- JSON sidecar: source template, seed, exact GTI boundaries, injection start
  and stop times, and all sampled injection parameters.
- PNG diagnostic plot: observed rate with GTIs and injected intervals shown.

This gives readable files for manual inspection and precise ground truth for
future model training and interval-localization evaluation.

### 3. Statistical baseline detector

`src/xray_burst_lab/detector.py` contains the first detector. It uses a
61-second rolling median as a local baseline, evaluates statistically large
upward and downward departures, groups adjacent significant bins, and records
the interval, peak time, significance, and direction (`excess` or `deficit`).

This is not machine learning. It is a simple, explainable benchmark that
checks whether the data pipeline, labels, and candidate reporting work end to
end. It is deliberately permissive because the future workflow prioritizes
recall.

### 4. Real NICER FITS light-curve support

The code can now read an OGIP-style FITS light curve with a `RATE` extension.
For NICER it reads:

- `TIME`: light-curve bin times.
- `RATE`: net count rate.
- `ERROR`: uncertainty of each net rate bin.
- `FRACEXP`: fractional exposure for filtering unusable bins.
- `GTI`: valid observing intervals.
- `TIMEZERO`: the FITS time-reference offset.

The `TIMEZERO` handling was important for the supplied NICER file: its rate
times are relative (`2-545 s`), while GTI times are absolute mission times.
The adapter checks direct and TIMEZERO-shifted coordinates and applies the one
that overlaps the actual light-curve bins.

## Synthetic Smoke-Test Result

The generated ten-observation example occupies approximately 3.6 MB. With the
fixed seed `20260909`, its statistical baseline produced 23 candidate
intervals and overlapped 10 of the 12 injected intervals.

It recovered Type I-like, Type II-like, magnetar-like, GRB-like, and artifact
injections. It missed the two slow flare injections. That behavior is useful:
it identifies a concrete next improvement, multi-timescale baseline and
detector features for slower changes.

This result does **not** demonstrate 95% recall. Ten observations are not a
large enough independent test population. The 95% target belongs to a future,
large, held-out synthetic dataset and a learned detector.

## Real NICER Test: Net_lc_100_800.lc

The supplied real file is a NICER/XTI FITS **light curve**, not a Level 1 event
file. It has 544 valid one-second bins, spanning 2-545 s. Its median net rate
is about 92.4 counts/s and its median quoted uncertainty is about 10.2
counts/s.

The baseline detector found one interval:

| Time interval | Significance | Direction | Interpretation |
|---|---:|---|---|
| 413-414 s | 3.005 sigma | Deficit | A one-second downward fluctuation, not a burst-like brightening |

There were no upward excess candidates at the current 3-sigma threshold.
Therefore, this file does not contain a burst-like candidate according to the
current baseline. This is a statement about this one light curve and this
simple detector, not a proof that the observation has no astrophysical event.

The real-data plot and machine-readable report are in `real/results/`.

## What "Working Properly" Means Here

Verified:

- The simulator creates the requested ten-observation dataset.
- CSV, JSON, and PNG products are created for every simulation.
- The baseline runs on synthetic light curves and writes candidates.
- The FITS adapter reads the supplied NICER file without modifying it.
- GTI and TIMEZERO reference handling was checked against the FITS headers.
- All 544 real light-curve bins are now correctly considered valid.
- A diagnostic plot and JSON report are written for the real observation.

Not yet verified or implemented:

- Performance on a large synthetic test set.
- The 95% recall requirement.
- A machine-learning detector.
- Classifying detected candidates.
- Type I burst morphology measurements.
- Photon-energy features or cooling-tail behavior.
- NICER event-file simulation or Level 1 event ingestion.
- Event-file adapters for AstroSat, RXTE, NuSTAR, XRISM, HXMT, and IXPE.
- Validation against known real bursts or published catalogs.

## Current File Guide

| Path | Role |
|---|---|
| `src/xray_burst_lab/simulation.py` | Generates synthetic light curves and labels. |
| `src/xray_burst_lab/detector.py` | Statistical detector and real FITS light-curve reader. |
| `src/xray_burst_lab/cli.py` | Command-line entry point. |
| `examples/smoke_test/` | Ten generated synthetic examples. |
| `real/Net_lc_100_800.lc` | Supplied real NICER light curve. |
| `real/results/real_lightcurve_report.json` | Machine-readable real-data result. |
| `real/results/real_lightcurve_candidates.png` | Plot of the real light curve and candidate interval. |

## Commands

Generate the reproducible ten-observation simulation set:

```bash
cd /home/varun/PhD/AI\&ML/xray_burst_lab
PYTHONPATH=src python3 -m xray_burst_lab \
  --output-dir outputs/smoke_test \
  --seed 20260909
```

Analyze the supplied real NICER light curve:

```bash
cd /home/varun/PhD/AI\&ML/xray_burst_lab
PYTHONPATH=src python3 -m xray_burst_lab \
  --input-lc real/Net_lc_100_800.lc \
  --output-dir real/results
```

Using the same seed recreates exactly the same simulations. Changing the seed
creates a different randomised set of light curves.

## Recommended Next Work

1. Add multi-timescale features so broad flares are detected reliably.
2. Generate a larger, balanced synthetic corpus without storing full event
   files until storage allows it.
3. Split that corpus into training, validation, and held-out test sets.
4. Define interval-overlap scoring and evaluate the 95% recall / 1 s boundary
   objective.
5. Train an ML interval detector and compare it to this statistical baseline.
6. Add a second-stage candidate classifier.
7. Move to event-file generation and calibrated mission-specific adapters.
8. Validate with real observations containing independently known events.
