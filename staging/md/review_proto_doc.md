# Review of `proto/doc` (e77e64f) against `main` (95d8925)

Reviewed 2026-10-10 on the `pepopepopepo` fork, branch `audit/proto-doc`. Environment: Linux,
gcc 16.2.1, deal.II 9.7.1 (package rebuilt against boost 1.92), Python 3.14.7, scipy 1.18.1,
pybind11 3.1.0. Every `path:line` below was checked on the e77e64f tree; every number comes from a
command quoted in section 7. The four docs under `staging/md/` get a status header and minimal
corrections on this branch (one commit per file); nothing else in the tree is touched.

Round 2 (2026-10-10): the branch was rebased onto `proto/doc` 6cfcb9f and `main` moved to fe9113c; the
delta is reviewed in section 8, the round-1 sections are unchanged. `FORK.md` at the root says how this
branch relates to the fork's other branches.

## 0. Verdicts

| doc | verdict | one line |
|---|---|---|
| the branch | rebase or merge `main` first | lacks 95d8925 "gcc fixes"; `staging/test/lumping.cc:103` does not compile with gcc 16 |
| `m_matrix.md` | keep as the record of PR #18, section 3 obsolete | all seven result rows reproduce with `check_m_matrix.py`; `tex/taussky_theorem.tex` is missing from the repository |
| `verify_solution.md` | keep as the record of PR #19, section 3 obsolete | `verify_solutions.py` has three one-line defects that stop it on a fresh checkout; re-run numbers in section 3 |
| `pybind11_interface.md` | sound design | two renamed programs, one citation that was never true in this tree |
| `tangent_vector_base.md` | sound plan, one design question | citations hold; the type check alone does not need the virtual vector base |

Cross-cutting: none of the four docs says which commit it describes, and two of them were out of date
within two days of merge (section 1). A status line per doc (added here) and a rule that a merge with
changes updates the note would have caught both.

## 1. The branch

- A1. `proto/doc` forked at c869c8a and does not contain 95d8925 ("gcc fixes": `Point& p` parameter
  names in `gpe.h:57,71`, `.extra = {}` in `solver.h:127`, `V{}` in `lumping.cc:103`).
  `git diff origin/main origin/proto/doc -- include staging/test` is exactly the reverse of that commit.
  With gcc 16.2.1 the tree fails to build:
  ```
  staging/test/lumping.cc:103:34: error: uninitialized const member in 'const class rmo::gpe::potential::Square<2>'
  ```
  (`main_coarse` itself compiles). Merge or rebase before anything else lands on the branch.
- A2. The notes describe `proto/m-matrix` (1 commit on efc7292) and `proto/verify-solution` (5 commits on
  efc7292) as they were on 2026-10-07. Both were merged on 2026-10-08/09 with changes the notes do not
  reflect: PR #18 (d4cdc49) merged the M-matrix check, then 3bda041 replaced the Boost Graph program by
  `staging/check_m_matrix.py` and 17d64c7 removed `main_m_matrix`; PR #19 (c869c8a) merged the
  verification, then 35a2c34 and 2d7ec8b folded `main_export` into `src/main_matrix.cc` and split the
  export into `M`, `S`, `M_V`. The commit hashes in `verify_solution.md`'s table (c731b4b .. 7ff57e5) do
  not exist on `main` (the history was rewritten; the surviving commits are 6e2ca79, 103bbf9, 536aae8,
  2c27603).
- A3. `tex/taussky_theorem.tex` is cited by `m_matrix.md` (section 2) and by the docstring of
  `staging/check_m_matrix.py`. It was never committed on any branch (`git log --all -- 'tex/*'` is
  empty); `.gitignore` ignores `doc/`, not `tex/`, so it was simply not added. The 12 references the note
  says were verified live only in that file.

## 2. `m_matrix.md`

- B1. Section 3 describes `staging/main_m_matrix.cc` with the Boost Graph Library and the ctests
  `m_matrix_q1_2d`, `m_matrix_p1_2d`, `m_matrix_q1_3d`. On `main` the program and the ctests are gone
  (17d64c7); `add_test` in `CMakeLists.txt` lists `arpack`, `gradient`, `energy`, `operator`, `lumping`.
  Section 6's first open issue ("remove `test_m_matrix`") is done; what is open instead is that no ctest
  covers the M-matrix property any more.
- B2. The results table (section 4) was re-measured with `main_matrix --matrix-market` and
  `check_m_matrix.py` on `main` 95d8925 (commands in section 7). All seven rows reproduce:

  | case | doc | re-measured |
  |---|---|---|
  | 2D Q1, S, level 5 | holds, 56 of 225 strict, 1 component | holds, 56 of 225, 1 component |
  | 2D P1 simplex, S, level 4 | holds, min A^-1 entry 7.8e-4 | holds, 7.80e-4 over 3 columns |
  | 3D Q1, S, level 3 | holds | holds, 26 of 27 strict |
  | 2D P1, lumped A0, levels 2-5 | holds | holds on 25, 81, 289, 1089 DoFs |
  | 3D P1, lumped A0 | fails, 268 positive entries | fails, 8 positive entries at level 2 (13 free DoFs); the doc's level is not stated |
  | 2D Q1, consistent A0 | fails, 1464 positive entries | fails, 1464 positive, 24 rows not dominant |
  | 2D Q1, S, Neumann | fails only the strict row | fails only the strict row, 0 of 289 |

- B3. Theory: Taussky (1949), the corollary via the Jacobi splitting, P1 opposite angles <= pi, Q1
  aspect ratio <= sqrt 2, non-obtuse tetrahedra are the standard statements. The Hauck, Liang,
  Peterseim reference exists as arXiv:2405.17090 (2024, "Positivity preserving finite element method
  for the Gross-Pitaevskii ground state"); the "Numer. Math. 158, 2026" volume is not in the arXiv record
  and was not checked.
- B4. Section 5, "the M-matrix property is necessary but not sufficient for non-negative iterates": it
  is sufficient for the exact energy-adaptive step and neither necessary nor sufficient for the
  computed iterates; reworded.

## 3. `verify_solution.md`

- C1. Section 3 describes `staging/main_export.cc` (`--matrices`, `_A0.mtx`, `_coords.bin`,
  `_weights.bin`, `_constrained.bin`). On `main`: `main_matrix --matrix-market` writes
  `matrix_lvl<L>_dim<d>_{M,S,MV}.mtx` (`main_matrix.cc:106-112`); `verify_solutions.py` forms
  `A0 = S + M_V` (line 192), takes constrained DoFs from the rows of `S` without off-diagonal entries
  (`constrained_dofs`, line 176), lumped weights from the diagonal of the lumped `M` (line 202), and
  coordinates from `main_coarse --output-bin` (line 173).
- C2. `verify_solutions.py:371`: `--build-dir` defaults to `cmake-build-debug-deal.ii`, an IDE
  directory; the repository ships `CMakePresets.json` with `build/release`.
- C6. `verify_solutions.py:63` imports `load_coords`, `load_solution` from `staging/plot_solution.py`,
  whose line 51 imports `seaborn` at module scope: `--no-plots` still needs seaborn, and on a machine
  without it the script stops before the first solver run (`ModuleNotFoundError: No module named
  'seaborn'`).
- C7. A relative `--build-dir` fails: every solver call runs with `cwd=case_dir` (line 78) and the path
  is never resolved (`FileNotFoundError: ... PosixPath('build/release/main_matrix')`).
- C5. `NumberGenerator` is seeded from `config.gpe.seed` in `staging/test/arpack.cc:81` and
  `gradient.cc:72`; `energy.cc:67` draws random points (`ellipsoid::random_point`) without seeding, so a
  failure there is not reproducible. (`lumping` and `operator` draw no random numbers; an earlier
  version of this line named them too.)
- C4. Section 6: Neumann, V = 0, beta = 0 makes the energy-adaptive matrix singular and CG diverges with
  a non-constant start; no guard exists. One `AssertThrow` in the option check that rejects
  `--boundary neumann --potential zero --beta 0` with the energy-adaptive metric would turn a divergence
  into a message; not applied here (upstream's call).
- C3. Re-run of section 5 on `main` 95d8925 (seed 42, `--jobs 16`, `OMP_NUM_THREADS=1`,
  `--no-plots`, seaborn stubbed, absolute `--build-dir`): **48 of 54 cases pass**, the same six fail,
  `zero` with beta = 10000, consistent and lumped, levels 5 to 7, all at the 2000-iteration cap with
  residuals 1.1e-6 .. 2.9e-4 (the note had 4e-7 .. 2e-4 with seed 0). Normalization error <= 4.4e-16
  everywhere; `optical_lattice` lumped at level 5 (h = 0.5 at radius 8) gives E = 1.330, the aliasing
  value the note attributes to h = 1. The `positive` column is false for 12 consistent cases (down to
  -0.18 for the lattice at level 5), as the note says and as the M-matrix results predict; it is not a
  required check for the consistent variant. So the note's numbers stand; only its text about the
  tooling is stale.

## 4. `pybind11_interface.md`

- D1. "The `-openmp-simd` flag from `CMakeLists.txt:110` (parsed as `-o penmp-simd`)": no such flag
  exists anywhere in this repository's history (`git log -S openmp-simd -- CMakeLists.txt` is empty).
  Note that deal.II's own `DEAL_II_CXX_FLAGS` do carry `-fopenmp-simd` (visible in every compile line);
  the remark must have come from a local configuration. Replaced by a note.
- D2. `main_export` became `main_matrix` (section 3 "Discretization replaces", section 5 mapping,
  section 6 step 5). The build notes (MacPorts Python 3.14, pybind11 not installed) describe one
  machine; on Arch, pybind11 3.1.0 is a package.
- D3. The design holds up: facades over the reference-holding internals are the right cut; the
  `IterationObserver<CycleInfo>` trampoline works with `PYBIND11_OVERRIDE` (it takes the GIL itself);
  `deal_II_exceptions::disable_abort_on_exception()` exists (`deal.II/base/exceptions.h:979`). One
  precision added: `Assert` is compiled out of Release builds, so a Release module only ever raises
  from `AssertThrow`; the abort case is Debug-only. The 6 single-level plus 12 FAS instantiations are
  the compile cost the note already names.

## 5. `tangent_vector_base.md`

- E3. Citations hold on e77e64f: `fas.h:274` (`retract_inv(zk, state.y)` in place), `gpe/residual.h:151`
  (the one call of `MetricBase::apply`), the argument orders of `ManifoldBase` and
  `VectorTransportBase`, `CycleInfo::min_value` and `extra` (`solver.h:65,69`), the deleted rvalue
  constructors (`solver.h:171-172`). `IterationBase` lives in `ropt/oracle.h:74` (there is no
  `iteration.h`).
- E1. `ropt/manifold.h:22` already declares `struct TangentVector { Vector<double> data; };` under the
  TODO the plan answers; the plan does not mention it. Noted as the stub the new type replaces.
- E2. "deal.II appears to have deprecated `VectorSpaceVector` (not verified)": 9.7.1 ships
  `lac/vector_space_vector.h` with no deprecation marker. Corrected; it does not change the design.
- E4. Which swaps the proposed types catch: of the three fixed in July 2026, `update_parameters(phi,
  w_proj)` (5bd06e6) and `apply_metric(grad_tilt, u)` (a2768bb) mix a point or ambient vector with a
  tangent vector and would not compile; `retract_inv_diff_by_norm_adjoint(M, phi, zeta, ...)` swaps two
  points and still compiles, as do `retract_inv(z, x, out)` and `retract_inv_diff(x, zeta, ...)` in the
  plan's own table. Added as a bullet, with the two ways to cover the rest (a base/trial point
  distinction, or `assert_tangent` at those call sites).
- E5. The plan carries two goals, backend switchability (`VectorBase`, `DealiiVector<V>`, `as_dealii`,
  `clone()` per copy) and the point/tangent type check (`TangentVector`), and the second does not need
  the first: a value type over `Vector<double>` gives the same guarantees with phases 3-6 only and no
  virtual dispatch or heap allocation per copy, and the plan's last paragraph already defers the second
  backend. Added as section 5 of the note; recommendation: `TangentVector` first.
- E6. Phase 7's bit-identical check against `data/org` (253 tables) needs both sides run with one
  thread (`DEAL_II_NUM_THREADS=1 OMP_NUM_THREADS=1`), else threaded assembly changes the last digits.
  Added. `min()` as a problem hook rather than a `VectorBase` member: agreed.

## 6. What this branch changes, what is left to upstream

Changed here (one commit per file, after this review): the status header and the corrections above
in the four notes. Left to upstream: merging `main` into the branch (A1); committing `tex/` (A3); the
three one-line fixes in `verify_solutions.py` (C2, C6, C7) and the seed in `energy.cc` (C5); the
singular-matrix guard (C4); a ctest or a `verify`-style script that keeps the M-matrix property
covered (B1).

## 7. Reproduction

```
# A1
git worktree add -b audit/proto-doc <dir> origin/proto/doc && git -C <dir> submodule update --init fmt
cmake -S <dir> -B <build> -G Ninja -DCMAKE_BUILD_TYPE=Release && ninja -C <build> main_coarse test_lumping

# S1 on main 95d8925 (deal.II 9.7.1 rebuilt against boost 1.92): 5 of 5 ctest passed, 4.70 s
cmake --preset release && cmake --build --preset release && OMP_NUM_THREADS=1 ctest --preset release

# B2, one row (2D Q1, S, level 5); the others change --levels/--multilevel, --mesh simplex,
# --dimension 3, --mass-lumping, --boundary neumann, and --matrix A0
build/release/main_matrix --boundary dirichlet --radius 8 --matrix-market --levels 5
python -I staging/check_m_matrix.py matrix_lvl5_dim2 --matrix S [--inverse-columns 3]

# C3 (seaborn stubbed on sys.path, absolute build dir)
OMP_NUM_THREADS=1 DEAL_II_NUM_THREADS=1 python -I staging/verify_solutions.py \
    --build-dir "$PWD/build/release" --out-dir <out> --jobs 16 --no-plots
```

## 8. Round 2: `proto/doc` 6cfcb9f against `main` fe9113c (2026-10-10)

Three new upstream commits on the branch since e77e64f: 883713e and 941d577 add `random.md` (a plan to
replace the `NumberGenerator` singleton, written with Claude Opus 5.5), 6cfcb9f reflows
`tangent_vector_base.md` to 100 columns with no change of content. Nothing of round 1 was taken. The
six round-1 commits above were replayed onto the reflowed text (our additions reflowed to match).

| doc | verdict | one line |
|---|---|---|
| the branch | still rebase or merge `main` first | 3 behind `main` (95d8925, 0b393c4, fe9113c); `lumping.cc` untouched since, so A1 stands |
| `random.md` | now a merged-work note; status header added | implemented the same day by 0b393c4 on `main`, in a different shape |
| `tangent_vector_base.md` | unchanged in content | round-1 findings and section 5 stand |

- F1. **`random.md` and its implementation.** The note proposes a `Random` wrapper around the engine plus
  either a `std::variant` of parameter structs (B) or a `RandomBase` hierarchy (E), in six phases, and
  recommends combining B and E. `main` 0b393c4 implements E without the wrapper: `Engine` is
  `std::mt19937` itself (`random.h:49`), `DistributionBase<T>` / `Distribution<Std>` are the hierarchy
  (`random.h:61`, `:171`), kernels take `DistributionBase<double>&` (`manifold.h:28`, `metric.h:83`,
  `:115`, `:161`). Not done: `GPE_Options::initial_dist` and `--initial-dist` (phase 5), the choice for
  `rand_sparse_vec_idx` (phase 4: kept, still no caller), the Boost distributions (phase 6). Phases 1-3
  landed as one commit; the sequence is bit-identical to the old one anyway (`review_main_fe9113c.md`
  A2 on `audit/main`: the same `--initial random --seed 42` table from a 95d8925 and an fe9113c binary).
  The "tag before phase 3" caveat is moot: no run under `data/org` used `--initial random`.
- F2. **Citations.** All thirteen `path:line` references in "How `util/random.h` works today" resolve on
  this branch's base c869c8a (quoted in section 9) and none on `main` after 0b393c4. The "TODO in
  `random.h`" about mean and standard deviation as constructor arguments exists at no commit:
  `random.h` carried two TODOs ("use a set-based approach if `s` is much smaller than `n`", "allow a
  user-specified seed") until 6e2ca79 removed them on 2026-10-07, before this branch's base.
- F3. **The note's standard-library claims hold.** The preconditions it cites ([rand.dist.norm.normal],
  [rand.dist.uni.real], [rand.dist.uni.int]) are the ones `random.h:108-146` checks; its libstdc++
  observation (`_GLIBCXX_ASSERTIONS` only) matches the header's behaviour. Not re-run here.
- F4. **Same pattern as A2 of round 1.** A design note merged into `proto/doc` the day its implementation
  landed on `main`, without naming the implementing commit; the status header carries it. The round-1
  suggestion stands: a merge with changes updates the note, or the note names its commit.

## 9. Reproduction, round 2

```
git fetch origin && git merge-base origin/main origin/proto/doc            # c869c8a
git log --oneline origin/proto/doc ^origin/main | wc -l                    # 5 (2 of round 1 + 3 new)
git log --oneline origin/main ^origin/proto/doc                            # fe9113c 0b393c4 95d8925
git diff --stat e77e64f 6cfcb9f                                            # random.md +374, tangent_vector_base.md +81 -69
git diff e77e64f 6cfcb9f -- staging/md/tangent_vector_base.md | grep '^[-+]' | grep -v '^[-+][-+]' \
    | sed 's/^[-+]//' | tr -s ' \n' ' ' | wc -w                             # a reflow: the same words
# F2
for ref in include/rmo/gpe/manifold.h:32 include/rmo/gpe/metric.h:90 include/rmo/gpe/model.h:100 \
           src/main_coarse.cc:464 staging/test/gradient.cc:72; do
    f=${ref%%:*}; l=${ref##*:}; git show c869c8a:$f | sed -n ${l}p; done   # each a NumberGenerator call
git log --all --oneline -S'TODO' -- include/rmo/util/random.h               # 6e2ca79 ff88cea 9aafb68 ...
git show 9aafb68:include/rmo/util/random.h | grep TODO                     # the two texts quoted in F2
git show c869c8a:include/rmo/util/random.h | grep -c TODO                  # 0
# A1 stands
git log --oneline 6cfcb9f -- staging/test/lumping.cc | head -1             # older than 95d8925
```
