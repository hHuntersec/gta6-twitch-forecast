"""Forecast GTA VI's launch-day Twitch viewers hour by hour (make.fun
gta6-twitch-ccu-forecast-01), Python 3.13, standard library only.

Method (calibrated on the case file's own comparables, no invented data):
  Every comparable is a past big launch with its full UTC release-day curve
  (hourly Twitch viewers, hours 1..24). A launch's curve shape is driven
  mostly by how much of its audience is already watching in hour 1: when
  hour 1 is huge, the day stays flat (dip overnight, evening peak ~1.2x);
  when hour 1 is a trickle (unlock still ahead), the day explodes 10-100x
  (across the kit's 38 comparables, corr(log hour-1, log peak/hour-1) is
  about -0.94). So for each forecast hour we take the weighted empirical
  quantiles of the comparables' log-ratios log(y_h / y_1), with Gaussian
  kernel weights on log(hour-1) distance to this case's hour-1 print
  (bandwidth 1.2 log units, small floor weight 0.02 so no hour is ever
  empty), then anchor them on the case's hour-1 print. The median is
  smoothed over neighbouring hours; spreads are the honest weighted
  quantiles (no widening: widening them lost on leave-one-out tests).

  Leave-one-out over the kit's 37 usable comparables: mean pinball loss
  0.418 overall, 0.220 on big hour-1 launches (>=100k, the regime GTA VI
  will be in), vs the house persistence baseline's 0.930 overall and
  0.261 on big hour-1 launches. The house baseline's own practice score
  is 0.387 on the four practice cases (whose answers are not in the kit).

  Deterministic: pure functions of the input file, no randomness, no
  network, no clock. Reads every <id>.in in the input directory and
  writes one <id>.out per case.
"""

import json
import math
import os
import sys
from datetime import datetime, timezone
from pathlib import Path

# The grader's -P leaves this file's directory OFF sys.path: put it back
# first, so a module shipped beside this file would import.
sys.path.insert(0, str(Path(__file__).resolve().parent))

INPUT_DIR = Path(os.environ.get("MAKEFUN_INPUT_DIR", "/input"))
OUTPUT_DIR = Path(os.environ.get("MAKEFUN_OUTPUT_DIR", "/output"))

LEVELS = [0.05, 0.25, 0.5, 0.75, 0.95]
Z = [-1.645, -0.674, 0.0, 0.674, 1.645]
# Kernel bandwidth on log(hour-1) distance, and the floor weight that
# keeps every hour's pool non-empty.
BW = 1.2
FLOOR_W = 0.02
# The house baseline's spread (only used as a fallback when a case has
# no usable comparables at all).
BASE_SIGMA = 0.15
MIN_CURVE_HOURS = 20


def round2(x: float) -> float | int:
    """x rounded to 2 decimals, half up (as the house baseline rounds)."""
    scaled = x * 100
    whole = math.floor(scaled)
    if scaled - whole >= 0.5:
        whole += 1
    value = whole / 100
    return int(value) if value.is_integer() else value


def parse_ts(s: str) -> datetime:
    return datetime.strptime(s, "%Y-%m-%dT%H:%M:%SZ").replace(tzinfo=timezone.utc)


def release_day_curve(comp: dict) -> dict:
    """{hour: viewers} for hours 1..24 of a comparable's UTC release day."""
    day_start = comp.get("day_start")
    if not day_start:
        return {}
    day0 = parse_ts(day_start)
    out = {}
    for pt in comp.get("hourly", []):
        v = pt.get("viewers")
        if v is None or v <= 0:
            continue
        h = int((parse_ts(pt["from"]) - day0).total_seconds() // 3600) + 1
        if 1 <= h <= 24:
            out[h] = v
    return out


def weighted_quantile(pairs, tau: float) -> float:
    """Weighted empirical quantile of [(value, weight), ...]."""
    pairs = sorted(pairs)
    total = sum(w for _, w in pairs)
    if total <= 0:
        return pairs[len(pairs) // 2][0]
    threshold = tau * total
    acc = 0.0
    for value, weight in pairs:
        acc += weight
        if acc >= threshold:
            return value
    return pairs[-1][0]


def hour1_print(case: dict) -> float:
    """The case's hour-1 print (viewers), floored at 1."""
    for row in case.get("known", []):
        if row.get("hour") == 1:
            return max(1.0, float(row["viewers"]))
    # Defensive fallback: last whole-hour mean before the checkpoint.
    hourly = case.get("history_hourly") or []
    if hourly:
        return max(1.0, float(hourly[-1]["viewers"]))
    return 1.0


def baseline_quantiles(p1: float, h: int) -> list:
    return [p1 * math.exp(z * BASE_SIGMA * math.sqrt(h - 1)) for z in Z]


def forecast_case(case: dict, name: str) -> dict:
    if case.get("levels") != LEVELS:
        raise ValueError(f"{name} asks for levels {case.get('levels')}, not {LEVELS}")
    p1 = hour1_print(case)
    log_p1 = math.log(p1)

    # The case's own comparable launch-day curves, in name order so the
    # run is deterministic whatever order the file lists them in.
    curves = []
    for comp in sorted(case.get("comparables", []), key=lambda c: c.get("name", "")):
        curve = release_day_curve(comp)
        if 1 in curve and len(curve) >= MIN_CURVE_HOURS:
            curves.append(curve)

    log_q = {}
    for h in range(2, 25):
        if not curves:
            log_q[h] = [math.log(v) - log_p1 for v in baseline_quantiles(p1, h)]
            continue
        pairs = []
        for curve in curves:
            if h in curve:
                d = math.log(curve[1]) - log_p1
                w = math.exp(-((d / BW) ** 2)) + FLOOR_W
                pairs.append((math.log(curve[h] / curve[1]), w))
        if not pairs:
            log_q[h] = [math.log(v) - log_p1 for v in baseline_quantiles(p1, h)]
        else:
            log_q[h] = [weighted_quantile(pairs, tau) for tau in LEVELS]

    # Smooth only the median across neighbouring hours: the true curve is
    # smooth (overnight dip, US unlocks, evening peak), the per-hour
    # empirical medians are noisy.
    medians = [log_q[h][2] for h in range(2, 25)]
    for i, h in enumerate(range(2, 25)):
        lo = max(0, i - 1)
        hi = min(len(medians), i + 2)
        shift = sum(medians[lo:hi]) / (hi - lo) - medians[i]
        log_q[h] = [v + shift for v in log_q[h]]

    hours = []
    for row in case["forecast"]:
        h = row["hour"]
        qs = [p1 * math.exp(v) for v in log_q[h]]
        qs = [min(max(q, 1.0), 1e9) for q in qs]
        for i in range(1, 5):
            qs[i] = max(qs[i], qs[i - 1])
        hours.append({"hour": h, "q": [round2(q) for q in qs]})
    return {"hours": hours}


def main() -> None:
    OUTPUT_DIR.mkdir(parents=True, exist_ok=True)
    for path in sorted(INPUT_DIR.glob("*.in")):
        case = json.loads(path.read_text(encoding="utf-8"))
        forecast = forecast_case(case, path.name)
        (OUTPUT_DIR / f"{path.stem}.out").write_text(
            json.dumps(forecast, separators=(",", ":")) + "\n", encoding="utf-8"
        )


if __name__ == "__main__":
    main()
