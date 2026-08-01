# Irrigant-Labs-V1
The New Frontier Model for AgTech


Status: planning stage, no dates, order-based gates. Full manual:
[`BEATNASA.md`](./BEATNASA.md). This file is the short technical map of that
manual: the target, the data, the model plan, and the gates. Read
`BEATNASA.md` for full detail, the outreach list, and sources.

---

## 1 · What this project is

Irrigant runs a four-field irrigation pilot today. This project turns that
pilot into a research lab for field-scale soil-water forecasting, and ships
the research as a product feature.

Two sentences define done:

- **Research claim:** the model predicts soil water at field scale, on
  held-out ground stations, more accurately than NASA's operational SMAP
  Level-4 product, and it forecasts 1 to 7 days ahead. SMAP L4 does not
  forecast at all.
- **Product claim:** a farmer draws a field boundary. No sensor install. The
  model's prediction matches a real probe reading within a stated error
  margin, and it drives an irrigation decision.

## 2 · The target, precisely

The benchmark is SMAP Level-4 Soil Moisture (L4_SM): a 9 km, 3-hourly global
soil-moisture analysis from NASA's Catchment land-surface model, assimilated
with SMAP L-band brightness temperature.

| Metric | Value |
| --- | --- |
| Formal accuracy requirement | ubRMSE ≤ 0.04 m³/m³ |
| Validated surface accuracy | ubRMSE ≈ 0.038–0.039 m³/m³ |
| Validated root-zone accuracy | ubRMSE ≈ 0.029–0.030 m³/m³ |
| Spatial resolution | 9 km |
| Validation sites | ~43 surface, ~17 root-zone reference pixels |

Three ways to win, easiest first:

1. **Forecast skill.** SMAP L4 has no forecast mode. A model that beats
   persistence, climatology, and ERA5-Land extrapolation at 1–7 day horizons
   has no operational competitor.
2. **Resolution.** SMAP L4 runs at 9 km. A pivot field runs at 800 m. Beat
   SMAP L4's accuracy at 100–200 m resolution and the win holds where farmers
   actually operate.
3. **Head-to-head accuracy.** Match or beat 0.038 ubRMSE surface / 0.030
   ubRMSE root zone against the same ground stations SMAP L4 validates on.
   Hardest, because SMAP L4 assimilates the satellite directly. Attempt this
   last.

## 3 · Why the win is reachable

Three precedents support the approach:

- **Weather:** DeepMind's GraphCast and Microsoft's Aurora beat the best
  operational physics weather models. The land-surface layer has not had
  this moment yet.
- **Hydrology:** Kratzert et al. (2018) showed one LSTM, trained across
  hundreds of basins, beats individually calibrated hydrology models. Soil
  moisture has the same problem shape: forcing time series and static
  attributes in, water state out.
- **Soil:** SMAP-HydroBlocks proved 30 m soil moisture from data fusion is
  possible. Planet's VanderSat product sells a commercial 100 m version.
  Nobody has shipped a version with forecasting and irrigation awareness.

## 4 · Data stack

All sources below are free to access.

**Ground truth**

| Source | Scale | Role |
| --- | --- | --- |
| ISMN | ~3,000 sites, global | Evaluation backbone |
| USDA SCAN, USCRN | ~200–114 US sites | Holdout tier |
| SMAP core validation sites | 43 surface / 17 root-zone pixels | Apples-to-apples comparison with NASA |
| Irrigant pilot fields | 4 fields, growing | Ground truth in irrigated crops, the gap public networks miss |

**Satellite inputs**

| Source | Resolution / cadence | Note |
| --- | --- | --- |
| SMAP L3/L4 | 9–36 km, daily/3-hourly | Input and the baseline to beat |
| Copernicus SSM | 1 km, 1–3 day | Sentinel-1 radar retrieval |
| Sentinel-1 GRD | 10–20 m, ~6–12 day | Raw radar backscatter |
| NISAR L3 | 200 m, ~2× per 12 days | Full calibrated release July 2026 |
| HLS | 30 m, 2–3 day | Optical and vegetation state |

**Forcing and statics:** ERA5-Land, gridMET/PRISM, HRRR forecasts,
gSSURGO/POLARIS soil hydraulics, USDA Cropland Data Layer, OpenET, DEM
terrain layers.

**The unobserved forcing:** satellites see the moisture jump from
irrigation, not the pump event itself. Irrigant's product records irrigation
events natively. This is a second proprietary data source, usable both as a
model input and as a standalone detection signal.

## 5 · Evaluation protocol

Non-negotiable rules, in order:

1. Station-level spatial holdout: never train and evaluate on stations
   within 100 km of each other.
2. Temporal holdout: hold out the final year of data from training.
3. Report ubRMSE, bias, Pearson r, and anomaly correlation, per station,
   as a distribution, not a single mean.
4. Compare against five baselines every time: persistence, climatology,
   ERA5-Land, SMAP L4, Copernicus SSM.
5. Report each depth separately. Never blend 0–5 cm and root-zone results.
6. Run negative controls: shuffled labels, input ablations, and a run that
   sees only what SMAP L4 sees.

**IrrigantBench:** a frozen, versioned public benchmark (station set,
forecast tasks, baseline scores, leaderboard). No standard benchmark for
soil-moisture forecasting exists today. Publish this first, before the
model is competitive; it converts every later claim into something a
reviewer can check in one click.

## 6 · Model roadmap

| Version | Scope | Gate |
| --- | --- | --- |
| v0 | Baseline harness, five baselines, reproduce SMAP L4's published ubRMSE independently | Ruler calibrated |
| v1 | One LSTM/GRU/transformer trained across all stations jointly (forcing + statics in, soil moisture out) | Beats persistence and ERA5-Land at 24/48/72h |
| v2 | Spatiotemporal encoder over Sentinel-1, NISAR, HLS, forcing grids at 100–200 m | Beats SMAP L4 ubRMSE on held-out stations |
| v3 | Physics-hybrid: learned model wrapped around a differentiable water balance | Improves out-of-distribution trust on irrigated fields |
| v4 | Forecast conditioning on operational weather, irrigation-event conditioning, per-field probe calibration | 1–14 day product-grade forecasts |

v1 runs on a laptop-class setup. v2 needs a small GPU cluster (weeks on a
few A100s), not planetary-model scale.

## 7 · Compute and funding, in order of use

1. Google TPU Research Cloud, Google Earth Engine, Microsoft Planetary
   Computer: free, no institutional affiliation required, available now.
2. Startup cloud credits (AWS Activate, Google/Microsoft for Startups):
   for storage and serving, available now.
3. NAIRR Pilot compute: needs a sponsoring researcher. Ask for this after
   an outreach contact (Section 9) agrees to sponsor.
4. Science competitions (Regeneron STS, ISEF): non-dilutive money, timed
   for fall submission season.

## 8 · Engineering infrastructure

- One analysis-ready, cloud-optimized Zarr data cube, co-registered across
  satellite, forcing, static, and station-label layers, versioned per layer.
- A flat matchup table (station, timestamp, depth, label, inputs) for
  v0/v1. Gridded patches start only at v2.
- The evaluation harness as a separate, pinned package from the models, so
  a benchmark score stays reproducible.
- Every training run records dataset version, code hash, config, and full
  eval output.
- Trained models export to the existing Helios serving pattern: hash-pinned
  artifact, metadata contract, fail-closed on missing artifact.

## 9 · Outreach and people

No campus, no default advisor. The substitute: a live pilot with real
ground truth, a working eval culture, and a benchmark design that invites
critique. Lead every email with one of those, attach a concrete artifact,
and ask one specific question. Climb the ask ladder: question, call,
protocol critique, compute sponsorship, co-authorship. Full target list
with names, affiliations, and rationale is in `BEATNASA.md` Section 10.

## 10 · Risks and kill criteria

| Risk | Mitigation | Kill signal |
| --- | --- | --- |
| Irrigated fields differ from public (rangeland) ground truth | Pilot flywheel, irrigation-event conditioning, per-field calibration | v2 beats SMAP L4 on natural sites but stays worse than persistence on irrigated fields after calibration |
| Radar sees 0–5 cm; roots drink at 30–90 cm | v3 physics coupling; report depths separately | Persistent root-zone gap after v3 |
| Eval leakage (a station's twin sits in training) | Enforced spatial-block holdout, adversarial review before any public number | Any public number built on unreviewed holdout code |
| Senior-year bandwidth | Cut scope to v1 + benchmark if recruiting stalls | Fall term consumes the research core with no v2 progress |

State every "beat NASA" claim with the product, the sites, the depth, the
metric, and the horizon. An unqualified claim is the only mistake in this
plan that a career this young cannot recover from.

## 11 · Gates

No dates. Order only.

- **G0:** data cube, matchup table, five baselines running. SMAP L4's
  published ubRMSE reproduced independently.
- **G1:** v1 beats persistence and ERA5-Land at 24/48/72h on held-out
  stations.
- **G2:** IrrigantBench public: frozen eval, baselines, leaderboard.
- **G3:** virtual-probe beta live. Backtest against pilot probes published.
- **G4:** v2 beats SMAP L4 ubRMSE on held-out stations, NISAR fusion
  included.
- **G5:** district pilot contracts referencing the benchmark. Paper
  accepted.

---

Sources, the full outreach target list, and the competitive landscape are
in [`BEATNASA.md`](./BEATNASA.md).
