# Live pit strategy (Strategy tab)

The Live page's **Strategy** tab answers one question during a race: *if this
car pitted now, where would it rejoin?* It is a deterministic estimate from the
current timing screen, not a forecast.

Code: `src/f1dash/processing/pit_strategy.py` (the calculation, pure) and
`src/f1dash/ui/strategy.py` (the panel). Tests: `tests/test_pit_strategy.py`,
several against `tests/fixtures/strategy/sao_paulo_2026_race.json`, a
trimmed recording of the live feed during the 2026 São Paulo Grand Prix.

## What it shows

- **The selected car now**: racing number, position, current tyre and its age,
  pit stops, gap to the leader and to the cars either side.
- **Pit loss**: a slider (15–35 s, widened if the estimate falls outside)
  starting at the estimate below, and a button that restores the estimate.
- **The rejoin**: the position before and after the stop, and the projected
  order around the rejoin point with each car's gap to the leader and its
  distance to the car that pitted. The car that pits is outlined.
- **Margins**: how much more pit loss loses the next place, and how much less
  gains one. A tie goes to the car already there, so the place is lost at
  exactly the first margin and gained only past the second. A margin is not
  shown when that neighbour has no timed gap, when the place is a range, or
  when gaining would need a stop costing less than zero.
- **Scenarios**: the same calculation at the chosen loss and 2 s either side,
  for a quicker or slower stop. These are scenarios, not a confidence interval.
- The time since the feed's last update, and a warning past 30 s (the Live
  page's own threshold) that the order is from then, not now.

## How it works

1. **One time axis.** Every running car gets a time behind the leader. On the
   lead lap that is `GapToLeader`. A lapped car's `GapToLeader` reads `1 L`,
   but its `IntervalToPositionAhead` is still in seconds, so the axis continues
   by adding intervals down the order (recorded live: P16 read `1 L` with an
   interval of `+24.236` to P15 at `+63.785`, so 88.0 s).
2. **The stop.** The selected car's time grows by the pit loss. Every other
   car keeps its time. The rejoin position is one more than the number of cars
   at or below the new time.
3. **Unknown places stay open.** A car with neither a gap nor an interval to a
   car with one is only known to sit between its neighbours in the order. If
   the rejoin point falls in that window, the panel shows a range (`P2–P3`)
   rather than picking one.

## Pit-loss estimate

The feed's `PitLaneTimeCollection` announces each stop's pit lane time (entry
to exit, stationary time included) and deletes it about 20 s later, so the
adapter keeps every announcement (`SignalRLiveAdapter.pit_history`). With three
or more stops between 12 and 45 s (shorter or longer is a drive-through, a
repair or a red-flag wait), the estimate is their median. Otherwise it is the
circuit's figure from `processing/pit_loss.PIT_LOSS_SEEDS`, otherwise 21 s.
Stops are counted from when the server connected to the feed. No topic is
added to the subscription and no request is made.

## Assumptions

- Every other car keeps its current gap: unchanged relative pace through the
  stop (no undercut or overcut), and no one else pits.
- The pit loss is the pit lane time. It slightly overstates the true loss,
  which is the pit lane time minus the time to drive the same stretch of track.
  Adjust the slider where the two differ. For reference, the seven green-flag
  stops announced during the recorded 2026 São Paulo race took 23.3–30.7 s in
  the pit lane (median 24.5 s), above the 21.0 s circuit seed.
- Positions are classification by time behind the leader at the timing line.
  A car that rejoins next to a lapped car on the road is not racing it.

## Limits (V1)

- **Safety car, VSC and red flag are unsupported.** A stop under them costs
  much less than under green and is not modelled, so the panel says so instead
  of predicting.
- **A car in the pit lane** has no trustworthy gap: mid-stop, the screen can
  still show its pre-stop gap (recorded live: VER `IN PIT` at P3 with 11.6 s,
  less than P2's 12.1 s). It keeps its place in the order but is placed only
  between its neighbours, and its own prediction is unavailable until it is
  back on track.
- Retired cars are left out of the order. Practice and qualifying are not
  supported.
- Gaps update per car as each passes a timing point, so the screen is briefly
  out of order during overtakes and stops. Gaps more than 0.5 s out of order
  are flagged.
- A neighbour within 1 s of the rejoin point makes the call "too close": a
  slow stop or a quick out-lap swaps them.

Out of scope for V1: tyre degradation, undercut and overcut pace, other cars'
stops, safety-car probability and full-race projections.
