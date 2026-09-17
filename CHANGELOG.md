# Changelog

Versions follow [semantic versioning](https://semver.org). The version lives in
`engine_version_info()` in `misc.cpp`, where the patch sets it, and the engine
reports it as its UCI `id name`, followed by the Stockfish release it derives
from.

The UCI options are the interface. An option may be *added* without a major
bump; an existing option's meaning or default will not change without one.

## Documentation correction - 2026-09-17

No change to the patch or the engine.

- **The 2026-09-16 correction below compared the engines at equal nodes
  only**, and its "no advantage" holds only there. The recommended profile
  searches about 2.7 times as many nodes per second as stock Stockfish 19, so at
  equal time it has more search.
- **At equal time, out of sample:** at 5 seconds it is ahead of stock at mate in
  10 and 11 (180 against 161, +39/-20, p = 0.018; repeated, +42/-20) and at mate
  in 9 (+86/-40), a much smaller margin than on ChestUCI. At 1 second it is level
  at mate in 6 to 8 and behind at 9 to 11 (+100/-137), where the full king-danger
  evaluator does better.

## Documentation correction - 2026-09-16

No change to the patch or the engine.

- **The ChestUCI results are in-sample.** `README.md`, `FINDINGS.md` and the
  0.1.0 entry below describe ChestUCI as a corpus neither engine was tuned on.
  MateHunter's profile was chosen on matetrack, which contains 6,526 of
  ChestUCI's 6,545 positions.
- **Out of sample there is no advantage over stock Stockfish 19** on the
  positions measured: on 972 held-out generated mates in 1 to 9, level at 10M
  nodes (963 each, +5/-5) and behind at 100k nodes (815 against 872) and 10k
  nodes (642 against 767). Huntsman 1 solved more than either at every budget.
  Measured one mate length at a time on further generated positions, it is
  not ahead at any depth from mate in 1 to 11: +25/-37 at mate in 10 and 11
  together at 10M nodes. Mate in 12 and deeper is untested.
- The 0.1.0 entry is left as released.

## 0.1.0 - 2026-09-14

**First release: a patch against Stockfish 19** (tag `sf_19`, commit
`edb0d9db6731067ec50ce619ff372b463bc4dd5d`) that makes Stockfish a forced-mate
finder. The engine reports itself as `MateHunter 0.1.0 (Stockfish 19)`.

- **The recommended profile is the default:** `MateEval=true` and
  `MateEvalNull=true`. The static evaluation is 0, so the only evaluation the
  search sees is the correction history Stockfish learns during the search.
- **The options that reproduce `FINDINGS.md` are included:** the king-danger
  evaluator and its weights, `MateEvalMain` and `MateEvalQS`, `MateMode` and its
  three pruning switches, and `MateEvalOff`. At their defaults they change
  nothing.
- **Against stock Stockfish 19** on ChestUCI at 10M nodes, one thread: 590
  against 343 at mate in 14 or more, and 1,225 against 757 at mate in 10 to 13.
- **Against Huntsman 1** on the same positions: 590 against 542, and 1,225
  against 1,101. On claims re-proved by MateProver, +220/−97 (p < 0.0001),
  ahead from mate in 10 to 17 and level from mate in 18. No claim by either
  engine was refuted.
- **Bench:** 2497913 nodes with `MateEval=false`, identical to stock Stockfish
  19; 5314178 with the defaults.
