# MateHunter

A Stockfish 19 fork for finding forced mates, version 0.1.0; see `CHANGELOG.md`.
It is distributed as a patch
against the Stockfish 19 release: 6 files and 27,470 bytes of diff, which is
smaller than a binary and shows exactly what was changed.

## Building

The base is the Stockfish 19 release: tag `sf_19`, commit
`edb0d9db6731067ec50ce619ff372b463bc4dd5d`, whose `src/` directory has the git
tree hash `15c9967c8add24a07ef4a34522cb6fb5c71ca580`. The patch paths are
relative to `src/`:

```
git clone --branch sf_19 --depth 1 https://github.com/official-stockfish/Stockfish.git
cd Stockfish
git rev-parse HEAD:src
cd src
patch -p1 < /path/to/matehunter.patch
make -j build ARCH=x86-64-avx2
```

`git rev-parse HEAD:src` must print `15c9967c8add24a07ef4a34522cb6fb5c71ca580`.
`make` validates the NNUE network, and fetches it if it is missing; the patch
does not change it.

**Check the build before trusting any result.** With `MateEval` off, MateHunter
searches exactly as stock Stockfish 19 does, so

```
printf "setoption name MateEval value false\nbench\nquit\n" | ./stockfish
```

must report `Nodes searched` of **2497913**, the bench recorded in the
Stockfish 19 release commit. With its defaults, `bench` reports **5314178**.

## Options

| option | default | effect |
| --- | --- | --- |
| `MateEval` | `true` | replace NNUE's static evaluation with the mate evaluation |
| `MateEvalNull` | `true` | with `MateEval`, fix the static evaluation at 0, so the only evaluation the search sees is the correction history Stockfish learns during the search. With it off, `MateEval` uses a king-danger evaluator that ignores material |

The defaults, `MateEval=true` and `MateEvalNull=true` with every other option
at its default, are the recommended profile. Scripts should still set every
option they rely on explicitly.

### For reproducing the measurements

These options exist so the results in `FINDINGS.md` can be reproduced. None is
part of the recommended profile, and at their defaults they change nothing.

| option | effect |
| --- | --- |
| `MateEscapeW`, `MateCheckW`, `MateExposeW`, `MateDefendW`, `MateAttackW`, `MateNearW`, `MateScale` | the king-danger evaluator's weights; they matter only with `MateEvalNull=false` |
| `MateEvalMain`, `MateEvalQS` | route the mate evaluation to one consumer of the static evaluation only: the main search, or quiescence. `MateEval` overrides both |
| `MateMode` | switch off razoring, futility and null-move pruning, and relieve one ply of reduction for checks |
| `MateNoRazor`, `MateNoFutility`, `MateNoNull` | the three prunings `MateMode` switches off, separately |
| `MateEvalOff` | bitmask switching off one consumer of the static evaluation per bit, listed in `engine.cpp` and `FINDINGS.md`; 0 in normal use |

## What it does, measured

Against stock Stockfish 19 on ChestUCI (neither engine tuned on it), one thread,
paired, with every mate option set explicitly:

| positions | budget | Stockfish 19 | full king-danger evaluator | recommended profile |
| --- | --- | --- | --- | --- |
| d14+, 953 | 10M nodes | 343 | 586 | 590 |
| d10-13, 1,524 | 10M nodes | 757 | 1,176 | 1,225 |
| d14+, 953 | 5 s | 324 | 592 | 606 |
| d10-13, 1,524 | 5 s | 715 | 1,195 | 1,244 |

With every mate option off the fork matched stock exactly: 343 = 343, zero
discordant.

Against Huntsman 1, a Stockfish fork with a `MateSearch` option, on the same
positions at 10M nodes: 590 against 542 at d14+ and 1,225 against 1,101 at
d10-13. Every mate only one of the two engines reported was re-proved with
MateProver, and none was refuted. On proved claims MateHunter is +220/−97
(p < 0.0001): ahead from mate in 10 to 17, level from mate in 18.

**Why, replicated at both depth ranges.** The gain comes from replacing NNUE's
evaluation with the correction history Stockfish learns inside the current
search. With `MateEvalNull` the static evaluation is 0 plus that correction
term. Remove the term from the static evaluation (`MateEvalOff=2048`) and the
profile falls below stock: 169 at d14+, 732 at d10-13. A genuinely flat
evaluation is worse than NNUE. The king-danger terms add nothing over the
recommended profile, and switching off any single consumer of the evaluation
under NNUE recovers none of the gain. The measurements, the mechanism study and
how to reproduce them are in `FINDINGS.md`.

## Provenance

This patch is a port of the Stockfish 18 fork to Stockfish 19. The king-danger
evaluator was carried over unchanged from the Stockfish 18 fork, not rewritten.

The measurements in `FINDINGS.md` were made with a development build that also
carried experimental options since removed from the patch: move-ordering,
move-restriction and learned-evaluation experiments, none of them part of any
profile measured here. Removing them does not change the search. Under 13
option settings covering every option that remains, `bench` reports the same
node count for the development build and for this patch (depth 9 for the two
settings that switch pruning off under NNUE, the default depth otherwise).

| | |
| --- | --- |
| Stockfish base | tag `sf_19`, commit `edb0d9db6731067ec50ce619ff372b463bc4dd5d` |
| patch sha256 | `d14cb5c22a513c8eb98c2323b3c0c7b35f92a670d86e8a6036c840c7ef8a821a` |
| stock Stockfish 19 bench | 2497913 nodes |
| MateHunter, `MateEval=false`, bench | 2497913 nodes |
| MateHunter, defaults, bench | 5314178 nodes |

## Licence

GPL-3, inherited from Stockfish; see `COPYING.txt`. This patch is a derivative
work of Stockfish and carries the same terms, which is why it lives apart from
MateProver and MateBench, both MIT.
