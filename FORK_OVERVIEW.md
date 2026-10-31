# Fork overview: henningms/vrp vs reinterpretcat/vrp

Snapshot taken 2026-10-31.

- **upstream**: `reinterpretcat/vrp` master at `e49bee0e` (2026-09-21)
- **fork**: `henningms/vrp` master at `1291d3fe` (v1.43.0, 2026-10-22)

The fork is **177 commits ahead** of upstream (about 155 not counting merge commits) and **0 behind**. Upstream has
been merged into the fork twice (2026-06-23 and 2026-09-24), so the fork contains all of upstream. The diff
touches 258 files, with about 28k lines added and 1.5k removed, most of them tests.

Commit hashes below refer to the fork's history. Reproduce the overview with:

```sh
git log --reverse --no-merges --format='%h %ad %s' --date=short upstream/master..origin/master
git diff --stat upstream/master origin/master
```

## Upstream contribution policy

Upstream's README says: *"open source, limited contribution … Please submit an issue or discussion if you have ideas
for improvement."* It is a one-person hobby project ("permanently experimental"). So the way into upstream is an
**issue or discussion first**, with a PR only if the maintainer wants one.

One fork PR was opened upstream: reinterpretcat/vrp#189 (tag-based LIFO). Henning closed it himself a minute after
opening it, so it was never reviewed or rejected.

## Changes, grouped

### A. New constraints and objectives

| Feature | Key commits | Notes |
|---|---|---|
| Job preferences (preferred / acceptable / avoid vehicles, weight) | d9fdb26e, c5aa5d63 | **Has no effect on the solution**: the objective never reaches the main goal. henningms/vrp#18 replaces it with `preferredSkills` |
| LIFO loading, separate stacks per tag | 104927db, 5d114035, 1a4c9610, 5af9d6bb | |
| Fixed activity order inside a multi job | d4cd9bf5 | |
| `requestedTime` on a job plus a matching objective | 12d41905 | |
| Max ride duration (hard constraint) | 8dd86ae7, 7b965897, a69a06c2 | |
| Solo riding | 168fae8d, 2f2b8916, 07f60376 | |
| Site mixing / site visits objectives, site inference | bcbab7f8, 5ccf0e02 | PRs #8, #9 |
| Minimize opposite-side arrivals (place `curbSide`) | 8cd0def8, 027caa0f, cecbce5a | PR #15 |
| `skills.oneOf` as a list of groups | dceb244c, a125b802, 2f949e44 | Changes upstream semantics. A flat list still parses as one group after 2f949e44. PRs #10, #11 |

### B. Vehicles and capacity

- **Configurable capacity**: several mutually exclusive capacity configurations per vehicle, e.g. seats vs.
  wheelchair spaces (67273e02, 5c19f06b, edf09bcd; PRs #1, #12).
- **Capacity layouts in the solution output**, with a solution checker for them (f200c9ce … 8abc4fd7; PR #13).
- **Per-run vehicle cost (`perRun`)**, plus a shared "tour runs" enabler (a5c822b3 … a90aa436; PR #14).
- **Flexible routes**: required stops and via waypoints on a shift (e8cfc0e2, fec79eb5, 94a97226; PR #5).

### C. Time and transport modelling

- **Ferry-aware transport**: vehicle-on-ferry crossings against a timetable, FIFO departure and arrival variants,
  `ferryLegs` per tour, and checker support (3ac56c87 … 21b4ec4d; PR #6). This is the largest single feature.
- **Stop duration**: pull-in/parking time charged once per arrival at a location, reported per stop and checked
  (a2813a32 … 73b6a722; PR #16).
- **Duration offsets on matrix cells**, keyed by departure time (782cc0b6 … 6f7a8695; PR #17).

### D. Insertion feasibility API (vrp-pragmatic)

- `FeasibilityContext` with `check_job` / `accept_job` / `to_solution_json`: checks whether a booking can be inserted
  in about 90µs, instead of re-running the solver (168fae8d, ef0640ca).
- Follow-ups: idle (registry) vehicles (726a98b4), candidate coordinates (e4d0dc5a), candidate stop durations
  (fa6ba665), and a fix for open routes corrupting the start schedule (ac62d957).

### E. Solver heuristics and performance

- New operators and selectors: `UnassignedRepair`, solo-aware construction, `UnassignedJobSelector`,
  `AnyFeasibleResultSelector`, `SoloAwareResultSelector`, an exhaustive route selector
  (4924fe2b, 4359290e, 1d6ec4e4).
- Construction tuning on `VrpConfigBuilder`: construction job cap, `initial_max_size`, a choice of initial
  construction heuristic (SISR blinks, sampled variants, …) (0b9aafa2, 4cbb71d0, 07be84ca).
- Bounded infeasible diversification, a bounded dynamic recreate portfolio, limits on exhaustive recreates, and no
  redundant multi suffix scans (7ced5c1d, 3261f189, 2f9ad923, 60f50c5d).
- Job index build about 3x faster (e1ac5c99), and the job neighbourhood built lazily (4155c158).
- CLI `analyze` ride-quality report (07be84ca).

### F. Core bug fixes

- **Pickup-delivery capacity checked per run** (d9828463). Touches upstream's `capacity.rs`; see below.
- Time-aware distance read at the pull-in time (e890d14d), loop bounds in LIFO and solo riding (1a4c9610, 2f2b8916),
  multi-dimension demand in LIFO (5af9d6bb), delivery waiting in max ride duration (7b965897). These are all in
  fork-only code.

### G. Fork plumbing

- CI: upstream's audit, coverage, discovery, experiments, maturin and publish workflows are replaced by `ci.yaml` and
  `release.yaml` (eb644886).
- Version bumps from 1.27.0 to 1.43.0, each with a CHANGELOG entry.
- Sync aids: fork-only error codes moved to E9xxx and constraint codes to 1001+, so they stop colliding with
  upstream's on every sync (75b35456). The LKH options removed upstream are cleaned up (75c4ae33). `--parallelism`
  is kept as a deprecated flag (585aa42d).

## What could go upstream

The fork's code has drifted far from upstream, so cherry-picking won't work. Anything we send upstream should be
**recreated on top of `upstream/master`**. Once upstream accepts something, we rebase the fork onto it.

### Strong candidates

1. **Pickup-delivery capacity per run** (d9828463). Upstream adds a pickup-delivery job's load to the peak load of
   the whole rest of the tour, as if the rider stayed on board to the end. That rejects valid insertions, e.g. a
   rider placed before a later run that fills the vehicle. The core fix is in `capacity.rs`; the `ConfigurableLoad`
   part is fork-only. Next step: a minimal repro on plain upstream, then an issue.
2. **Job index build** (e1ac5c99). Upstream has since removed the per-pair `Vec` allocations on its own (4f38e1f0),
   which was a large part of the 3x gain. What upstream still lacks: `select_nth_unstable_by` instead of a full sort,
   cloning only the K kept neighbours, and deduplicating vehicle start locations. This needs re-measuring on
   upstream before we propose it.
3. **Lazy job neighbourhood** (4155c158). A real gain for anyone building a `Problem` without solving it, but it
   changes the `Jobs::new` signature.

### Worth a discussion

General-purpose, but large or opinionated:

- `VrpConfigBuilder` construction settings and faster blink insertion
- `oneOf` groups, proposed as an additive change
- max ride duration and per-run cost, which are common in passenger transport (DRT)
- the feasibility API
- stop duration and duration offsets

### Probably not

Too tied to our domain, or fork-only by nature:

- ferry crossings, site mixing, curb side, solo riding, LIFO tags, capacity layouts
- the ride-quality CLI
- CI and release workflows, version bumps, error-code ranges

## Housekeeping

- `Cargo.toml`: the workspace version is `1.43.0`, but the internal `vrp-*` dependencies still say
  `version = "1.27.0"`.
- `docs/superpowers/plans/2026-10-15-capacity-layout-output.md` (about 1.5k lines) and
  `docs/superpowers/specs/2026-10-15-capacity-layout-output-design.md` are AI planning notes committed to master.
- Job preferences are silently ignored by the main objective (henningms/vrp#18).
- reinterpretcat/vrp#190 by kristinl ("tags constraint") is open upstream, and a reply there suggests using skills
  instead. It may be worth checking whether it is related to our work.
