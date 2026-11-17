[
# GTA VI launch-day Twitch forecast

Entry for the [make.fun bounty "Forecast GTA VI's launch-day Twitch viewers hour by hour"](https://make.fun/bounty/gta6-twitch-ccu-forecast-01).

## What it does

`main.py` (Python 3.13, standard library only) reads a case file — everything
recorded up to the end of hour 1 of GTA VI's release day (19 Nov 2026, UTC) —
and forecasts, for each of hours 2–24, five quantiles (5%, 25%, 50%, 75%, 95%)
of that hour's Twitch viewer count. It runs offline in the sandbox: no
network, deterministic, ~0.15 s per case.

## Method

Calibrated on the case file's own comparables (past big launches' full
release-day curves, read at runtime — no invented data):

- A launch's curve shape is driven mostly by how much of its audience is
  already watching in hour 1: huge hour-1 launches stay flat (dip overnight,
  evening peak ~1.2x); small hour-1 launches explode 10–100x (across the
  kit's 38 comparables, corr(log hour-1, log peak/hour-1) is about −0.94).
- For each forecast hour, weighted empirical quantiles of the comparables'
  log-ratios log(y_h / y_1), with Gaussian kernel weights on log(hour-1)
  distance to the case's hour-1 print (bandwidth 1.2, floor weight 0.02),
  anchored on the case's hour-1 print. Median smoothed over neighbouring
  hours; spreads are the honest weighted quantiles.
- Falls back to the house baseline formula if a case has no usable
  comparables.

Leave-one-out over the kit's 37 usable comparables: mean pinball loss 0.418
overall, 0.220 on big hour-1 launches (>=100k, the regime GTA VI will be in),
vs the house persistence baseline's 0.930 overall and 0.261 on big hour-1
launches.

## Run

```sh
MAKEFUN_INPUT_DIR=<input-dir> MAKEFUN_OUTPUT_DIR=<output-dir> python3 main.py
```

Reads every `<id>.in` in the input directory and writes one `<id>.out` per case.
