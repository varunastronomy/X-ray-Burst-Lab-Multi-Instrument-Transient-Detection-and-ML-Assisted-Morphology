# X-ray-Burst-Lab-Multi-Instrument-Transient-Detection-and-ML-Assisted-Morphology
Developing a Python framework for detecting and characterizing X-ray bursts across instruments. The prototype generates synthetic light curves and detects anomalies in NICER data. Planned work includes multi-mission event ingestion, ML-based detection, burst morphology analysis, and validation against real observations

# X-ray Burst Lab

### Multi-Instrument Transient Detection and ML-Assisted Burst Morphology

X-ray Burst Lab is an ongoing scientific Python project for detecting and characterizing transient events in X-ray observations. Its long-term goal is a shared analysis framework supported by instrument-specific data adapters, with statistical and machine-learning methods for finding candidate intervals and studying burst morphology.

**Current status:** a working prototype for synthetic light-curve generation, real NICER FITS light-curve ingestion, and interpretable statistical anomaly detection. Machine learning, physical classification, burst morphology measurements, and multi-mission event-file ingestion are planned features.

## Research objective

The intended workflow is:

```text
Supported mission event files or extracted light curves
    -> instrument-specific reading and quality checks
    -> standardized time series with exposure and GTIs
    -> anomaly intervals
    -> candidate classification with uncertainty
    -> Type I burst selection and morphology analysis
```

The initial priority is high-recall anomaly detection: finding potentially interesting intervals for further analysis. Multiple anomalies may occur within one observation.

“Multi-instrument” means a common analysis interface with explicit mission support. Different instruments require their own timing, calibration, event-selection, background, and exposure handling. The prototype does not currently accept arbitrary event files.

## Implemented features

### Synthetic observation generation

The simulator creates ten short NICER-like observations per run, with a fixed mixture of individual transients, mixed events, artifacts, and a quiet observation. Random seeds control parameter draws for reproducible examples.

- Time resolution: 0.25 seconds.
- Observation lengths: approximately 0.5–3 kiloseconds.
- Source-template names: 4U 1636-536, 4U 1608-52, and XTE J1810-189.
- Persistent count-rate baselines with smooth variability and changing synthetic backgrounds.
- Poisson-sampled counts and simulated good-time-interval (GTI) gaps.
- Phenomenological injections labeled `type_i`, `type_ii`, `flare`, `magnetar`, `grb`, and `artifact`.

Injection shapes include fast-rise/exponential-decay profiles, recurrent pulses, broad flares, short spikes with tails, multiple pulses, rate steps, and dropouts. These are software-testing templates, not calibrated physical models or validated class distributions. Source names identify simulation presets rather than fits to those sources.

### Separate labels and predictions

Each simulated observation produces:

| Product | Contents |
|---|---|
| CSV | Time, counts, rate, synthetic background rate, GTI validity, and injection labels |
| JSON | Seed, source template, GTIs, injection intervals, and sampled parameters |
| PNG | Diagnostic light-curve plot with injection annotations |

A manifest indexes the observations. A separate candidate table records detector predictions. The detector does not use injection class labels to decide which bins are anomalous.

### Statistical anomaly baseline

The current detector uses a 61-second rolling median baseline and a default deviation threshold of 3. It estimates uncertainties from Poisson rate statistics for simulations and uses supplied rate errors for real FITS light curves.

Adjacent threshold-crossing bins are grouped into candidate intervals. Outputs include start and stop times, peak time, peak deviation significance, and direction (`excess` or `deficit`). Direction is assigned from the strongest deviation within the interval.

These significance values are local standardized deviations; they are not a calibrated observation-wide false-alarm probability. A candidate is a statistical departure, not a physical burst classification.

### Real NICER light-curve ingestion

The FITS reader expects a `RATE` extension containing `TIME`, `RATE`, and `ERROR`. It screens nonfinite values and nonpositive uncertainties, uses positive `FRACEXP` values as a validity filter when available, and applies GTIs when present.

A timing mismatch in the supplied NICER example was addressed by comparing direct GTI alignment with alignment after applying the rate table's `TIMEZERO` offset. The reader selects the alignment covering more bins. This handles the reference example; broader support requires explicit reconciliation of timing metadata.

The analysis writes a JSON candidate report and a diagnostic plot without modifying the input science file.

## Recorded prototype results

### Synthetic smoke test

The existing project documentation records the following result for seed `20260909`:

| Measure | Recorded result |
|---|---:|
| Simulated observations | 10 |
| Injected intervals | 12 |
| Candidate intervals | 23 |
| Injected intervals overlapped by candidates | 10 |
| Missed injections | Both slow flares |

This is a small software smoke test, not a held-out scientific benchmark. Overlapping an injection does not establish accurate onset or tail recovery. The missed flares motivate detection across multiple timescales.

### Real NICER example

The saved analysis report contains 544 valid one-second bins and one candidate at 413–414 seconds in the light curve's stored time coordinates. Its peak deviation is approximately 3.005 and its direction is `deficit`.

There are no upward candidates at the current threshold. The detected deficit is not a burst-like brightening, and the result does not establish that the observation contains no astrophysical event.

### Verification scope

Earlier project records document successful source compilation, generation of ten CSV/JSON/PNG sets, synthetic candidate-table creation, and analysis of the supplied real FITS light curve after the timing correction.

This overview was prepared by reviewing the source code, handoff, status documentation, and saved real-data report. It does not represent a new execution of those tests, a new plot inspection, or independent reproduction of the documented synthetic overlap score. Automated regression coverage and scientific validation remain development tasks.

## Current limitations identified during review

- A single rolling-baseline timescale can absorb broad transients into the estimated background.
- Timing and duration calculations assume approximately uniform bin spacing. Explicit GTI and time-gap boundaries are needed to prevent inappropriate interval merging.
- Adjacent excesses and deficits can be grouped into one candidate because detection currently uses absolute deviations.
- Positive fractional exposure is used for screening; a general per-bin exposure treatment is not yet implemented.
- Very short synthetic profiles require integration over bin widths for faithful count simulation.
- Injection boundaries can extend outside the observed interval or through gaps. Evaluation must define how truncated and unobserved events are scored.
- Threshold-selected intervals need a separate onset/tail recovery stage before morphology measurements.
- No trained ML model, physical candidate classifier, or morphology extractor is implemented in this package.
- Validation on known real bursts and additional instruments is outstanding.

## Development roadmap

1. **Strengthen data handling and tests.** Cover reproducibility, time references, GTIs, gaps, exposure, candidate direction, and invalid inputs.
2. **Improve the statistical benchmark.** Add multiple detection timescales, separate excess/deficit handling, and more robust interval construction.
3. **Define evaluation rules.** Specify event matching, observable injection boundaries, localization errors, and sensitivity versus amplitude and duration.
4. **Expand realistic validation.** Combine controlled injections into representative backgrounds with independent observations containing known bursts.
5. **Scale simulations efficiently.** Generate training examples in batches rather than retaining a large event-file corpus.
6. **Introduce an ML interval detector.** Compare a one-dimensional segmentation model or window classifier against the statistical baseline on untouched test data.
7. **Develop classification and morphology.** Explore uncertain/unknown classifications and measure rise, decay, duration, asymmetry, and peak structure with uncertainties.
8. **Add mission adapters.** Start with supported cleaned/calibrated products and test transfer to a second instrument before expanding coverage.

The provisional target is at least 95% event recall with 1-second boundary tolerance on a clearly defined held-out population. This target has not been achieved or validated. Evaluation should also report precision, false candidates per kilosecond, and performance by event type and observing conditions. Training and test partitions must avoid observation leakage, with source and mission holdouts used to assess generalization.

Morphology will initially describe observed count-rate shapes. Physical interpretation requires consideration of energy band, instrument response, and additional evidence; shape alone will not establish photospheric radius expansion or another physical class.

## Installation and usage

Requires Python 3.10 or later. Dependencies are NumPy, Matplotlib, and Astropy, declared in `pyproject.toml`.

Run these commands from the repository root:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -e .
```

Generate the reference simulations and baseline candidates:

```bash
python3 -m xray_burst_lab --output-dir outputs/smoke_test --seed 20260909
```

Analyze a compatible FITS light curve, replacing the example input name:

```bash
python3 -m xray_burst_lab --input-lc data/example_lightcurve.lc --output-dir outputs/real_example
```

The installed `xray-burst-lab` command provides the same interface. Output filenames are fixed within each destination; use a new output directory to preserve earlier results. Real science data are not required for the synthetic example and may be unavailable for public distribution.

## Repository structure

```text
xray_burst_lab/
├── src/xray_burst_lab/
│   ├── simulation.py       # Synthetic observations and injection labels
│   ├── detector.py         # Statistical baseline and FITS light-curve reader
│   ├── cli.py              # Command-line routing
│   ├── __main__.py         # Module execution
│   └── __init__.py         # Package interface
├── examples/smoke_test/    # Reference synthetic products
├── real/                  # Local science input and saved example results
├── pyproject.toml          # Package metadata and dependencies
├── README.md              # Quick-start documentation
├── PROJECT_STATUS.md      # Development status
├── CODEX_HANDOFF.md        # Implementation handoff
└── github.md              # General project overview
```

## Engineering experience demonstrated

Current work demonstrates modular scientific Python development, FITS table handling, reproducible simulation, labeled dataset construction, uncertainty-based anomaly detection, command-line tooling, diagnostic visualization, and debugging of time-reference inconsistencies.

The next stages develop experience in automated testing, injection–recovery experiments, ML benchmarking, uncertainty-aware classification, and cross-instrument validation. These are planned milestones rather than completed capabilities.
