# What makes MateHunter work

MateHunter began as a king-danger evaluator for Stockfish: an evaluation that
ignores material and scores how exposed the defending king is. It does find far
more forced mates than stock Stockfish. The measurements below show that the
king-danger terms are not why. The gain comes from replacing NNUE's evaluation
with **the correction history Stockfish learns during the search**.

**Correction: these results are in-sample for MateHunter.** Its options and
profile were chosen on matetrack, which contains 6,526 of ChestUCI's 6,545
positions. On generated positions outside both corpora the advantage is far
smaller and depends on the clock: at 5 seconds the recommended profile finds
about 12% more mates than stock Stockfish 19 at mate in 10 and 11 (+39/-20,
repeated as +42/-20), against 74% more on ChestUCI; at 1 second it is no better
than stock, and the full king-danger evaluator does better; at equal nodes it is
never ahead, because it searches about 2.7 times as many nodes per second.
`README.md` gives the numbers. What follows describes MateHunter on ChestUCI.

Every number here is from ChestUCI, with one thread, every mate option set
explicitly, and results paired position by position. "+A/−B" counts positions only the first arm solved against
positions only the second solved; p is a two-sided sign test on those counts.

## The result

| positions | budget | stock Stockfish 19 | full king-danger evaluator | recommended profile |
|---|---|---|---|---|
| mate in 14 or more, 953 | 10M nodes | 343 | 586 | 590 |
| mate in 10 to 13, 1,524 | 10M nodes | 757 | 1,176 | 1,225 |
| mate in 14 or more, 953 | 5 seconds | 324 | 592 | 606 |
| mate in 10 to 13, 1,524 | 5 seconds | 715 | 1,195 | 1,244 |

The recommended profile is `MateEval=true`, `MateEvalNull=true`,
`MateMode=false`, and it is the default. Against stock it is +283/−36 on the
deeper set and +520/−52 on the shallower one at 10M nodes. With every mate
option off, MateHunter searched exactly as stock did over all 953 deeper
positions: 343 against 343, no position solved by one and not the other, and
the same `bench` node count.

## Against Huntsman 1

[Huntsman 1](https://github.com/joergoster/Stockfish-old/releases/tag/h1), a
Stockfish fork with a `MateSearch` option, ran on the same positions and budget
with `MateSearch=true`, `Threads=1` and `Hash=256`.

| positions | Huntsman 1 | full king-danger evaluator | recommended profile |
|---|---|---|---|
| mate in 14 or more, 953 | 542 | 586 | 590 |
| mate in 10 to 13, 1,524 | 1,101 | 1,176 | 1,225 |

On reported mates the recommended profile is +131/−83 against Huntsman on the
deeper set (p = 0.0013) and +239/−115 on the shallower one (p < 0.0001).

A reported mate is a claim, not a proof. Every mate only one of the two engines
reported, and a seeded sample of 100 that both reported, was re-proved with
[MateProver](https://github.com/ScottMoore0/mateprover) 0.2.0
(`--direct-depth --no-portfolio` at the claimed length, 20M nodes).

| claims | proved | refuted | not settled in 20M nodes |
|---|---|---|---|
| recommended profile only, 370 | 220 | 0 | 150 |
| Huntsman only, 198 | 97 | 0 | 101 |
| both, sample of 100 (MateHunter / Huntsman) | 74 / 73 | 0 / 0 | 26 / 27 |

Counting proved claims only, the recommended profile is +220/−97 against
Huntsman (p < 0.0001):

| mate in | proved, MateHunter only | proved, Huntsman only | p |
|---|---|---|---|
| 10 to 13 | 177 | 73 | < 0.0001 |
| 14 to 17 | 31 | 8 | 0.0003 |
| 18 to 21 | 9 | 9 | 1.0 |
| 22 or more | 3 | 7 | 0.34 |

1. **MateHunter is clearly ahead from mate in 10 to 17**, on reported and
   proved claims alike.
2. **From mate in 18 the two are level.** From mate in 26 Huntsman reports
   slightly more mates (+6/−11 at mate in 26 to 30, +11/−17 at 31 or more), not
   significantly, and too few claims that deep are settled within the proof
   budget to say more.
3. **No claim by either engine was refuted**, and the share not settled is about
   the same for both, so re-proving favours neither.

The full king-danger evaluator gives the same picture: +193/−105 on proved
claims.

## The constant evaluation is not constant

`MateEvalNull` makes the king-danger evaluator return 0. That matched the full
evaluator on the deeper set (590 against 586, +68/−64, p = 0.79) and beat it on
the shallower set (1,225 against 1,176, +145/−96, p = 0.002). Under a 5-second
clock it beat it again (1,244 against 1,195, +136/−87, p = 0.0013).

A search that sees a zero evaluation everywhere should not be a strong mate
finder, so the natural question is what the search sees instead. Stockfish adds
a **correction** to every static evaluation: a value learned during the search
from how search results differ from the static evaluation, keyed by pawn
structure, minor pieces, non-pawn material and the preceding moves. With the
evaluation at 0, that correction is the whole of what the search treats as its
evaluation.

## Finding which part of the search carries it

`MateEvalOff` switches off one consumer of the static evaluation per bit. Each
switch was run twice at 10M nodes: under the recommended profile (against the
recommended profile) and under NNUE (against the all-off control). Switches that
moved anything were re-run on the shallower set.

| switched off | recommended, 14+ | recommended, 10–13 | NNUE, 14+ | NNUE, 10–13 |
|---|---|---|---|---|
| nothing | 590 | 1,225 | 343 | 757 |
| correction history in the static evaluation only | **169** (+5/−426) | **732** (+45/−538) | 293 | 710 |
| correction history everywhere | 159 (+8/−439) | 677 (+48/−596) | 300 | 689 |
| quiet-move futility pruning | 556 (p = 0.003) | 1,181 (p = 0.002) | 288 | 673 |
| improving flags | 542 | 1,194 (p = 0.037) | 314 | 737 |
| aspiration windows | 567 (p = 0.07) | not run | 309 | 719 |
| each of seven others | 570–587, none p < 0.05 | not run | 323–334 | not run |

The seven others are eval-difference move ordering, ProbCut, capture futility,
move-count pruning, the LMR evaluation term, quiescence futility and history
bonus scaling.

1. **Take the correction out of the static evaluation and the profile falls
   below stock** on both sets: 590 to 169, and 1,225 to 732. A genuinely flat
   evaluation is worse than NNUE.
2. **Removing the correction from the evaluation costs almost as much as
   removing correction history everywhere**, so its other uses, in margins and
   reductions, are not what matters.
3. **No single consumer is misled by NNUE on its own.** Switching any one off
   under NNUE recovers none of the gain; every NNUE arm is at or below the
   control.
4. **Quiet-move futility pruning and the improving flags help** under the
   learned evaluation, on both sets.

The reading, that inside a forced-mate search the search's own results are a
better guide than a network trained to predict game outcomes, is an
interpretation of these numbers, not a separate measurement.

## Things it does not explain, and ruled out on the way

- **Razoring, futility and null-move pruning** are not the mechanism: switching
  all three off under NNUE gains nothing (330 against 343 on the deeper set,
  +73/−86, p = 0.34; 730 against 757 on the shallower one).
- **Routing king danger to one part of the search only** does not reproduce the
  gain (main search only 343, quiescence only 317), but those arms put two
  evaluation scales into one search and cannot be read on their own.
- **Escape squares alone** beat the full evaluator on both sets, but beat the
  constant evaluation on the deeper set only; that did not replicate.

## Where it is weak

- **Very deep mates.** At mate in 31 or more it solves about a fifth: 35 of 169
  with the full evaluator at 10M nodes, 34 with the recommended profile under a
  5-second clock.
- **Tablebase endgames where the defender keeps a piece.** On 526 positions from
  3- and 4-piece tablebases, stock Stockfish 19 solved 317 and MateHunter 283
  (stock +64/−30); on rook against rook, 96 against 77. There NNUE's knowledge of
  the material is what decides the line.

## Reproducing

Build as described in `README.md`, check `bench` against stock, then run each
arm with these options (every other mate option at its default of off):

| arm | options |
|---|---|
| stock-equivalent control | `MateEval=false`, `MateEvalNull=false` |
| full king-danger evaluator | `MateEval=true`, `MateEvalNull=false` |
| recommended profile | `MateEval=true`, `MateEvalNull=true` |
| a consumer switched off | add `MateEvalOff=N`: 1 improving, 2 eval-difference ordering, 4 ProbCut, 8 capture futility, 16 quiet futility, 32 move-count pruning, 64 LMR evaluation term, 128 quiescence futility, 256 correction history, 512 aspiration windows, 1024 history bonus scaling, 2048 correction history in the static evaluation only |
| Huntsman 1 | `MateSearch=true` |

Every search was `go mate N nodes 10000000` (or `movetime 5000`) with
`Threads=1` and `Hash=256`, where N is the position's stated mate length, and a
position counts as solved when the engine reports `score mate` between 1 and N.
A claim is re-proved with `mateprover -z N --direct-depth --no-portfolio
--threads 1 --node-limit 20000000`, where N is the claimed length.
