# Review of `main` 95d8925..1b95859 (the random-number refactor, its documentation, the `--eps` default)

Reviewed 2026-10-10 on the `pepopepopepo` fork, branch `audit/main` (cut from 1b95859; see `FORK.md` at
the root for how the fork's branches relate). Round 2 reviewed 95d8925..fe9113c (sections 1-4), round 3
1b95859 (section 5) and the level-11 benchmark against the paper (section 6). Environment: Linux, gcc
16.2.1, deal.II 9.7.1 (built against boost 1.92), doxygen 1.15. Every `path:line` below was checked on the
1b95859 tree (1b95859 changes two lines of `include/rmo/option.h` and no line count, so the round-2
citations stand); every number comes from a command quoted in the last section, "Reproduction". This
branch adds only this file.

## 0. Verdicts

| commit | verdict | one line |
|---|---|---|
| 0b393c4 "refactor random number generation" | sound; take as is | builds, 5 of 5 tests pass, bit-identical output for the default start and for `--initial random` with the same seed; four small follow-ups (section 1) |
| fe9113c "document random.h" | sound | doxygen parses the file without a warning; one stale help string outside the file (C3) |
| `staging/md/random.md` on `proto/doc` (not part of main) | now a merged-work note | the plan this commit implements, in a different shape; reviewed on `audit/proto-doc`, round 2 |
| 1b95859 "decrease --eps default to 2e-8" | sound; one stale help string | the default now matches `study.sh`; the `--eps` help text names the wrong norm (B5, section 5) |
| level-11 benchmark at eps 2e-8 (section 6) | the paper's Table 8 claim holds, larger here | 32 of 40 multilevel configurations faster than single-level; frobenius/differential_frobenius slower at every depth (E3) |
| `main_coarse` exit status (section 6) | a gap | exits 0 after a stalled line search or at the iteration cap (E5) |

Round 1 (`audit/proto-doc`, `review_proto_doc.md`): C5 (unseeded `energy.cc`) is closed by 0b393c4 in
upstream's own shape. C2, C6, C7 (`verify_solutions.py`, `plot_solution.py`) are still open on main;
`fix/verify-solutions` carries the two one-line fixes rebased onto fe9113c.

## 1. 0b393c4: the refactor

- A1. **Builds and passes.** gcc 16.2.1, `cmake --preset release`, `cmake --build --preset release`:
  no warning from the changed files (the only warning in the build is the pre-existing
  `staging/test/lumping.cc:88` `-Wmissing-braces`, untouched here). `ctest`: 5 of 5 in 2.5 s with one
  thread.
- A2. **Output is bit-identical to 95d8925.** Two checks, both with the 95d8925 binary kept from the
  previous build and the fe9113c one:
  - default start (`--initial constant`, the benchmarked path): level 8, `--multilevel 8,7` mass/mass,
    two runs each: iterations 43, final energy 30.50350270428, 132 inner iterations, 22 coarse steps,
    identical;
  - `--initial random --seed 42`: level 6 single-level, 40 iterations: the whole convergence table
    (42 rows, energy printed to 17 digits) is identical between the two binaries, so `Engine(seed)`
    followed by one `UniformDistribution(0.5, 1.5)` draws the sequence `NumberGenerator::seed(seed)`
    plus `unifrnd(0.5, 1.5)` drew, as the comment at `random.h:46` claims. `--seed 43` changes the
    start (energy at iteration 0: 67.7478 vs 68.0683), so the option still reaches the draw.
- C1. **`rand_sparse_vec_idx` stays with no caller** (`random.h:220`; `grep -rn rand_sparse_vec_idx`
  finds only its definition) and keeps `<algorithm>`, `<numeric>` and `<vector>` included for it.
  The plan on `proto/doc` (phase 4) said delete it or rewrite it with `std::ranges::sample`; neither
  happened. Delete it, or give it a test.
- C2. **One engine per `initial_value()` call.** `model.h:99-104` builds `Engine engine(options.seed)`
  inside the `RANDOM` case, so every call returns the same `x0`. In `main_coarse.cc` the single-level
  (`:390`) and multilevel (`:415`) calls are in exclusive branches (`:384`, `:410`) and `src/*.cc` draw
  nowhere else, so today one call happens per process and the behaviour equals the old one. A future
  caller that draws twice (a restart, a second experiment in one process) gets the same start twice
  without being told; a sentence in the `initial_value()` docstring (`model.h:83-86`) would say so.
- C3. **Stale help string.** `option.h:281-282`: `--seed` is "seed of the random number generator";
  there is no single generator any more (`option_types.h:163` was updated, the help text was not).
  Also unchanged: `option.h:308`'s "(see --seed)", which still reads fine.
- C4. **`energy.cc` seeds with `default_seed`, not `--seed`.** `energy.cc:64` is the one consumer that
  cannot take `GPE_Options::seed`: its `main()` takes no arguments (`energy.cc:85`). Consistent with the
  test having no options; stated here so nobody reports it as the next C5. `gradient.cc:264` uses
  `config.gpe.seed`, as the plan says.
- C5. **Design points that hold** (no action): `DistributionBase` is non-copyable and, with the copy
  constructor user-declared, not movable either, so no slicing and no engine duplication
  (`random.h:70-72`); the engine is a non-const reference, so a temporary `Engine(42)` does not bind
  (`random.h:65`); `Distribution` keeps the `std::` distribution between calls (`random.h:201`), which
  matters for `std::normal_distribution`'s cached second value; the parameter checks implement exactly
  the preconditions of [rand.dist.norm.normal], [rand.dist.uni.real], [rand.dist.uni.int]
  (`random.h:108-146`); `fill(std::span<Element>)` takes a `dealii::Vector<double>` directly because
  its iterators are raw pointers (`model.h:102` compiles, A1). `make_checked(Params... params)` takes
  the parameters by value, so `UniformDistribution(engine, 0, 1)` with `int` arguments converts to
  `double` silently: harmless.
- C6. **The removed `seed()` calls in `main.cc`, `main_bench.cc`, `main_matrix.cc`, `arpack.cc`** are
  safe: at fe9113c none of `src/*.cc` calls `random_point`, `random_tangent_vector` or a distribution;
  their only draw was `initial_value()`, which now seeds itself (C2).

## 2. fe9113c: the documentation

- B1. **Doxygen parses it cleanly.** A generated default `Doxyfile` over `include/` (Reproduction) gives
  two warnings, both "Problems running latex" for the formula images (no LaTeX on this machine), none
  about `random.h`. `EXTRACT_ALL = NO` and `WARN_IF_UNDOCUMENTED = NO`, so this says the markup is
  well-formed, not that everything is documented.
- B2. **The file comment's claims hold:** the engine equivalence (A2), "`dealii::Vector<double>`
  converts to `std::span`" (A1), the standard citations (C5). "Not thread-safe" matches the code: the
  engine is shared by reference and `std::mt19937` is not synchronised.
- B3. **"The engine of the former NumberGenerator"** (`random.h:46`) is the only remaining mention of
  the old class in the tree (`grep -rn NumberGenerator` over `include`, `src`, `staging`). Fine as
  history; it will read as a dangling reference once nobody remembers the singleton. The plan note on
  `proto/doc` describes it, which is where that history belongs.

## 3. `random.md` on `proto/doc`, against this implementation

The note (883713e, 941d577, written with Claude Opus 5.5) proposes a `Random` wrapper class plus either
a `std::variant` of parameter structs (B) or a `RandomBase` hierarchy (E), in six phases. 0b393c4
implements E without the wrapper: `Engine` is `std::mt19937` itself, `DistributionBase<T>` /
`Distribution<Std>` are the hierarchy, kernels take `DistributionBase<double>&`. Phases 1-3 are one
commit (bit-identical anyway, A2); phase 4's choice for `rand_sparse_vec_idx` was not made (C1);
phase 5's `GPE_Options::initial_dist` and `--initial-dist` do not exist; phase 6 (Boost distributions)
was not done. The note's "How `util/random.h` works today" describes the pre-0b393c4 tree; its
thirteen `path:line` citations all resolve on `proto/doc`'s base c869c8a and none on main. Its "TODO
in `random.h`" that "asks whether mean and standard deviation should become constructor arguments"
(section "Distribution parameters stay per call") exists on no commit: `random.h` carried two TODOs
until 6e2ca79 removed them on 2026-10-07 ("use a set-based approach if `s` is much smaller than `n`"
and "allow a user-specified seed"), neither about distribution parameters, and none since. The full
comparison and the status header live on `audit/proto-doc` (round 2 of `review_proto_doc.md`).

## 4. What this branch changes, what is left to upstream

This branch: this file and `FORK.md`. Left to upstream: C1 (delete or test `rand_sparse_vec_idx`), C2
(one sentence in the docstring), C3 (the help string), B5 (the `--eps` help string, round 3), the
`verify_solutions.py` fixes of round 1 (`fix/verify-solutions`, two commits, cherry-pickable), and from
section 6: E3 (frobenius/differential_frobenius), E4 (the line-search rejections), E5 (the exit status).

## 5. 1b95859: the `--eps` default (round 3)

- B4. **The default now matches `study.sh`.** `option.h:199` sets `--eps` to 2e-8 (it was 1e-4), the
  value `study.sh` passes since c0f35ad; the only other change is the file's last newline. It matters for
  every run that relies on the default: `ropt/fas.h:256-257` switches the coarse condition off for the rest
  of the run once `norm_level <= eps`. A level-11 run with the Frobenius metric and transport at depth 4
  (`--multilevel 11,10,9,8`) takes 71 fine iterations with 3 coarse steps at 1e-4, and 12 with 6 at 2e-8
  (Reproduction; the deterministic counts of the measured runs in section 6).
- B5. **The help text names the wrong norm.** `option.h:200` calls `--eps` the "minimum norm of restricted
  gradient"; `fas.h:256` compares it with `norm_level`, the norm of the current level's gradient
  (`state.x_grad`, `fas.h:250`), not with `norm_coarse`, the restricted gradient's (`state.x_grad_restr`,
  `fas.h:251`), and in the coarse condition's metric (`cond_norm`), so it is not a residual tolerance
  either. Suggested: "stop coarse corrections once the level's gradient norm (in the coarse-condition metric)
  is at or below this".

## 6. Level-11 benchmark at eps 2e-8, against the paper's Table 8

Measured 2026-10-10 on the reviewer's machine: one pinned core of a 16-core desktop, performance governor,
`OMP_NUM_THREADS=1 DEAL_II_NUM_THREADS=1`, the binary built at fe9113c (1b95859 changes only the default
of `--eps`, which every run passes). An earlier section 6, published on this branch on the morning of
2026-10-10, was measured at the old default `--eps 1e-4` and counted duplicate transports as separate
algorithms; it is withdrawn and replaced by this one.

Setup: `study.sh`'s arguments (`$COMMON --eps 2e-8`, Reproduction). Single-level: `--levels 11 --metric
none --max-iter 1000 --tol-residual 1e-8`. Multilevel: five distinct (metric, transport) pairs at depths 2-5
(`--multilevel 11,10[,9[,8[,7]]] --max-iter 150 --tol-residual 1e-8`): mass/adjoint_differential (Version
III), mass/mass (IV), mass/differential (VI), frobenius/frobenius (IV), frobenius/differential_frobenius
(the Frobenius counterpart of VI) (`option_types.h:166-175`). `study.sh`'s other pairs are duplicates:
`option_types.h:173` calls ADJOINT_RESTRICTION_FROBENIUS "same as FROBENIUS", and in an eps 1e-4 run the
same day adjoint_differential_frobenius gave frobenius's counts and mass adjoint_restriction gave
adjoint_differential's, energies equal to the last printed digit. Every configuration ran with consistent
mass and with `--mass-lumping`, once per round, five rounds (n = 5). Time: the `elapsed` column at the
first fine-level row with residual <= 1e-8, what `staging/plot_times_table.py` reads. A difference to
single-level counts when it is over max(5 %, the larger interquartile range); every row below passes that
bar. Solver time against single-level, median of five:

| pair (paper's version) | consistent mass, depth 2 / 3 / 4 / 5 | lumped mass, depth 2 / 3 / 4 / 5 |
|---|---|---|
| single-level (EARGD) | 77.44 s, 130 iterations | 12.35 s, 131 iterations |
| mass/adjoint_differential (III) | -37.6 / -66.4 / -68.4 / -67.2 % | -20.0 / -62.9 / -81.1 / -85.2 % |
| mass/mass (IV) | -39.3 / -64.5 / -67.0 / -59.7 % | -20.7 / -57.2 / -71.1 / -76.9 % |
| mass/differential (VI) | -39.0 / -63.9 / -66.1 / -59.5 % | -17.7 / -56.1 / -69.5 / -75.9 % |
| frobenius/frobenius (IV) | -44.0 / -71.7 / -84.8 / -89.5 % | -37.6 / -71.2 / -84.7 / -89.6 % |
| frobenius/differential_frobenius | +25.8 / +39.5 / +48.2 / +50.0 % | +44.4 / +46.6 / +45.3 / +42.7 % |

- E1. **Table 8 holds and is exceeded.** Table 8 (single-level 237 s; 2-, 3-, 4-level -27, -39, -39 %;
  mass metric, Version III, residual 1e-8) against mass/adjoint_differential with consistent mass
  (`study.sh`'s default): -37.6 / -66.4 / -68.4 %. Depth 3 and 4 are within 6 % of each other, as in the
  paper; depth 5 adds nothing. Absolute times depend on machine and revision; the paper names neither.
- E2. **Every configuration converges at level 11**, each to its condition's single-level energy
  (30.3547061313 consistent, 30.3485204681 lumped; equal within 1e-9 relative), and the five rounds give
  identical iteration counts, energies, residuals, coarse steps and inner-iteration sums. Lumping changes
  the discrete problem (the energies differ by 2.0e-4 relative), so the two columns are never compared
  with each other.
- E3. **frobenius/differential_frobenius is slower than single-level at every depth**, in both conditions:
  134-137 fine iterations against single-level's 130-131, plus 26-37 coarse steps. Its coarse corrections
  do not reduce the fine work.
- E4. **With consistent mass, the mass-metric pairs gain nothing past depth 4** (mass/mass and
  mass/differential: 24 fine iterations at depth 4, 28 at depth 5). Only those runs, and
  mass/adjoint_differential at depths 4-5, print `Warning: Armijo line search hit max iterations (3)`
  (`ropt/descent.h:76`) followed by `-> Step rejected by line search.` (`ropt/solver.h:112`), 9-33 times
  per run; no lumped run prints it. Not investigated.
- E5. **At level 10 some configurations fail, and the exit status does not say so.** One pass over all of
  them (2026-10-10, three re-run for this section): with consistent mass, frobenius/differential_frobenius
  at depths 2-5 ends at the 150-iteration cap (residual 1.2-1.5e-8), mass/mass, mass/differential and
  mass/adjoint_differential at depth 2 stop early (2.5-2.8e-8), mass/mass at depth 3 stops after one fine
  iteration at residual 0.133 ("no progress possible (line search stalled), stopping early",
  `ropt/fas.h:172, 317, 367`, `ropt/solver.h:232`), and mass/mass and mass/differential at depth 5
  converge to energy 30.36218 against single-level's 30.36182 (1.2e-5 relative), another stationary point;
  with lumped mass, frobenius/differential_frobenius at depths 2-5 hits the cap (1.2e-8), mass/mass at depth
  5 stops after one iteration, and five configurations at depths 4-5 reach other states. Every one exits
  0: `main` calls `run_experiment` (`src/main_coarse.cc:475`), checks no outcome, and ends with the implicit
  `return 0`. A non-zero exit when the tolerance is not reached would let `study.sh` and similar scripts
  notice. None of these failures occurs at level 11 (E2).
- E6. **Setup outside `elapsed`** (mesh, assembly before the first iteration) is 1.7-3.6 s per run, median
  2.8 s. Negligible for the consistent-mass times; for the fast lumped runs it is most of the process
  (frobenius/frobenius depth 5 lumped: 1.28 s solver time, 4.08 s for the whole process).
- E7. **Memory:** single-level holds the most (peak RSS 2.9 GB consistent, 2.8 GB lumped, against 1.1-2.5 GB
  for the multilevel runs). Cause UNVERIFIED.

## 7. Reproduction

```
# build and tests (one thread, as benchmarked; round 2 at fe9113c, the same result at 1b95859)
cmake --preset release && cmake --build --preset release
OMP_NUM_THREADS=1 DEAL_II_NUM_THREADS=1 ctest --test-dir build/release        # 100% tests passed out of 5

# A2, --initial random: the same table from both binaries (OLD = a 95d8925 main_coarse, NEW = this build)
ARGS="--potential optical_lattice --boundary dirichlet --radius 11 --line-search --kappa 0.5 --eps 1e-4
      --beta 1000 --levels 6 --metric none --max-iter 40 --tol-residual 1e-8 --initial random --seed 42"
$OLD $ARGS > old.org; $NEW $ARGS > new.org
diff <(grep '^|' old.org | cut -d'|' -f2-10) <(grep '^|' new.org | cut -d'|' -f2-10) | wc -l   # 0
$NEW ${ARGS/--seed 42/--seed 43} | grep '^| 0 '                                              # energy 6.8068e+01

# C1, C6
grep -rn 'rand_sparse_vec_idx' include src staging          # only include/rmo/util/random.h:220
grep -rn 'random_point\|random_tangent_vector\|Distribution\|Engine' src/*.cc    # empty

# B1
doxygen -g Doxyfile && printf 'INPUT = include\nRECURSIVE = YES\nQUIET = YES\nGENERATE_LATEX = NO\n' >> Doxyfile
doxygen Doxyfile 2>&1 | grep random.h                        # empty

# section 3: the TODOs random.h ever had (5 commits change the count; the texts at the last two that had one)
git log --all --oneline -S'TODO' -- include/rmo/util/random.h    # 6e2ca79 ff88cea 9aafb68 6921a20 c421fcc
for c in ff88cea 9aafb68 6e2ca79 c869c8a; do git show $c:include/rmo/util/random.h | grep -c TODO; done   # 1 2 0 0

# round 3, B4 and B5
git diff --stat fe9113c 1b95859                  # include/rmo/option.h | 4 ++--
sed -n 199,200p include/rmo/option.h             # default_value(2e-8), "minimum norm of restricted gradient"
sed -n 248,257p include/rmo/ropt/fas.h           # norm_level from state.x_grad, compared with options_fas.eps
COMMON="--potential optical_lattice --boundary dirichlet --radius 11 --line-search --kappa 0.5 --beta 1000"
for e in 1e-4 2e-8; do
  OMP_NUM_THREADS=1 DEAL_II_NUM_THREADS=1 build/release/main_coarse $COMMON --eps $e --multilevel 11,10,9,8 \
      --max-iter 150 --tol-residual 1e-8 --metric frobenius --transport frobenius > ff-$e.org
  awk -F'|' '/^\| [0-9]/ {n=$4+0; if ($6 ~ /\*/) k++} END {print n, "iterations,", k+0, "coarse steps"}' ff-$e.org
done                                             # 71 iterations, 3 coarse steps / 12 iterations, 6 coarse steps

# section 6: one configuration's time and counts (pinned, one thread); sl's table has no gradient columns
OMP_NUM_THREADS=1 DEAL_II_NUM_THREADS=1 taskset -c 1 build/release/main_coarse $COMMON --eps 2e-8 \
    --multilevel 11,10,9 --max-iter 150 --tol-residual 1e-8 --metric mass --transport adjoint_differential > t.org
awk -F'|' '/^\| [0-9]/ && $8+0 <= 1e-8 {print "iterations", $4+0, "elapsed", $14+0, "s"; exit}' t.org   # 26, ~26 s
OMP_NUM_THREADS=1 DEAL_II_NUM_THREADS=1 taskset -c 1 build/release/main_coarse $COMMON --eps 2e-8 \
    --levels 11 --metric none --max-iter 1000 --tol-residual 1e-8 > sl.org
awk -F'|' '/^\| [0-9]/ && $6+0 <= 1e-8 {print "iterations", $2+0, "elapsed", $12+0, "s"; exit}' sl.org  # 130, ~78 s
build/release/main_coarse $COMMON --eps 2e-8 --multilevel 11,10,9,8,7 --max-iter 150 --tol-residual 1e-8 \
    --metric mass --transport mass > e.org 2> e.err; grep -c 'Step rejected' e.err    # E4: 33

# E5 at level 10, all three exit 0
M="--max-iter 150 --tol-residual 1e-8"
build/release/main_coarse $COMMON --eps 2e-8 --multilevel 10,9,8 $M --metric mass --transport mass > a.org 2> a.err
echo $?; grep -c 'stopping early' a.err; grep '^| [0-9]' a.org | tail -1     # 0; 2; iteration 1, residual 1.3280e-01
build/release/main_coarse $COMMON --eps 2e-8 --multilevel 10,9 $M --metric frobenius \
    --transport differential_frobenius > b.org; echo $?; grep '^| [0-9]' b.org | tail -1   # 0; iteration 150
build/release/main_coarse $COMMON --eps 2e-8 --multilevel 10,9,8,7,6 $M --metric mass --transport mass > c.org
build/release/main_coarse $COMMON --eps 2e-8 --levels 10 --metric none --max-iter 1000 --tol-residual 1e-8 > d.org
grep '^| [0-9]' c.org | tail -1; grep '^| [0-9]' d.org | tail -1    # energy 3.0362182477536770e+01 / ...38373e+01
```
