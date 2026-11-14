# polymarket-faa-attribution-02 — per-market realized-volatility quantile forecaster

Forecasts, minute by minute and 4 hours ahead, the Yes price of every
trading market of Polymarket's FAA outage attribution family (whether the
U.S. government officially attributes the November 13, 2026 FAA outage to
Iran, to a foreign actor, to a hack, or to an AI sandbox escape), as five
quantiles (0.05 / 0.25 / 0.5 / 0.75 / 0.95) per market per minute.

Entry: `main.py` (Python 3.13, standard library only, no network, fully
deterministic). See `build-log.md` for the calibration notes and the full
practice table.

## Approach

A per-market realized-volatility forecaster around a persistence centre,
calibrated on the sibling competition polymarket-nvidia-ceo-forecast-01 and
re-verified on this kit's practice cases:

- **Centre:** the market's last price `x` at the checkpoint. No drift, no
  mean reversion, no centre smoothing — every variant of those tested made
  the practice pinball loss worse.
- **Spread:** `q_tau(h) = clamp(x + z_tau * sigma_m * h**0.40, 0, 1)`,
  rounded to 6 decimals, where `z_tau` is the standard normal quantile at
  the five levels.
- **sigma_m:** the winsorized RMS of the market's 1-minute returns over
  the last 120 minutes (returns clipped at +/-0.05), per square-root
  minute. Floored at `max(0.0005, 0.01 * x)` and capped at `0.008`. The
  short 120-minute window matters twice over here: the four FAA markets
  were created on 2026-11-13 and opened with violent repricing (Iran
  0.49 -> 0.275 in one minute; AI sandbox escape 0.215 -> 0.02 in one
  minute), and a long window would carry those launch-regime jumps into
  every final case, while the last-120-minute RMS forgets them within
  2 hours.
- **Mild asymmetry:** upper quantiles (z > 0) scaled by 0.9, lower
  quantiles (z < 0) by 1.1 — the practice hit rates showed the upper band
  too wide and the lower band too narrow.
- **Short-history safety:** a market with fewer than 120 prices uses all
  the prices it has; with 0–1 prices sigma falls back to the floor, and
  an empty history defaults the centre to 0.5. Nothing assumes a 72-hour
  history, which matters because the FAA markets are only ~16 hours old
  at the deadline.

## How it runs

```
MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out python3 main.py
```

Reads every `<id>.in` in the input directory and writes one
`<id>.out` per case: `{"forecasts": [{"id": ..., "q": [[...5 numbers...]
x 240 rows]}, ...]}`. Runtime is ~0.1–0.15 s per case (limit 8 s);
two runs on the same input produce byte-identical output.

## Practice results

Pinball loss (percentage points) on this kit's practice cases p00..p04,
ground truth taken from each case's successor case; p05 has no successor
in the kit, so it is not scored:

| case | this entry | house baseline |
|------|-----------|----------------|
| p00  | 0.446088  | 0.536138 |
| p01  | 0.297476  | 0.380352 |
| p02  | 0.344662  | 0.408408 |
| p03  | 0.239656  | 0.368599 |
| p04  | 0.193297  | 0.298794 |
| **mean** | **0.296418** | **0.392518** |

Beats the house baseline on every practice case — a 24.5% improvement
on the mean. Caveat, from the rules: the practice is cut from a different
event ("Next CEO of NVIDIA?"), so its score is no guide to the final;
the model is per-market and event-agnostic, so it transfers directly to
the 4 FAA markets.

## Files

- `main.py` — the entry.
- `LICENSE` — MIT.
- `build-log.md` — build log: calibration, per-case practice table,
  validation runs.
- `README.md` — this file.
