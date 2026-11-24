---
name: vtune-hotspot-comparison
description: Compare two Intel VTune hotspots profiles (ref vs dev) of GEOS-Chem Classic, GCHP, or HEMCO to say which OpenMP parallel-loop changes sped the model up or slowed it down. Use when asked to compare, summarize, or interpret VTune hotspot reports such as r000hs.hotspots.by-func.csv or r000hs.hotspots.by-line.csv, to explain libgomp spin time (gomp_simple_barrier_wait, gomp_team_barrier_wait_end, gomp_team_end, do_spin), or to reconcile VTune CPU time with gcclassic_timers.json wall-clock timers after a COLLAPSE or SCHEDULE optimization.
---

# VTune Hotspots Comparison Method

Instructions for comparing two VTune `hotspots` results, a reference build and a development build, and attributing the timing differences to specific OpenMP parallel loops. This documents the *method*, not a snapshot of results: re-run it fresh on each pair of profiles and do not reuse an earlier comparison's verdicts.

This is a research and reporting task. Do not edit model code, rebuild, or submit jobs unless asked; the user usually recompiles and reruns themselves.

## 1. Confirm the two runs are comparable before reading any numbers

A wrong conclusion here usually comes from a hidden difference between the runs rather than from the profile itself. Check each of these and report any mismatch up front:

- **Submodules.** Run `git submodule status` in each run directory's `CodeDir`. The user may say "only `src/GEOS-Chem` differs", but HEMCO, Cloud-J, or HETP can be on a different commit too. A leading `+` means the checkout differs from the superproject's recorded commit. If HEMCO differs, HEMCO loop changes are mixed in with the GeosCore ones, and every HEMCO-side result must be labeled as such.
- **Build.** Diff the compiler, flag, `CMAKE_BUILD_TYPE`, and `OMP` lines of the two `build/CMakeCache.txt` files.
- **Run setup.** Diff the run scripts (`#SBATCH -c`, partition list, `--exclusive`) and check that `geoschem_config.yml` has the same dates and resolution. Profile runs and timing-only runs in the same tree are often different lengths (1 day vs 1 hour), so do not mix their numbers.
- **Node and timing.** The node names appear in the slurm log, in VTune's `systemcollector-<pid>-<node>` lines. The modification time of `gcclassic_timers.json` gives each run's end time. Two jobs that overlap in time, or that run one after the other against the same input files, see different file-system cache states, which moves HEMCO's I/O-dominated timer by tens of seconds.

## 2. Load and aggregate the CSVs

VTune's `-format csv` output is tab-separated despite the extension. `by-func` keys on `Function`, `Module`, and `Source File`; `by-line` keys on `Source File` and `Source Line`. Line numbers shift between ref and dev, so match the hot lines by reading the code, not by line number.

```python
import csv, re
from collections import defaultdict
COLS = ["CPU Time", "CPU Time:Effective Time", "CPU Time:Spin Time", "CPU Time:Overhead Time"]

def load(path):
    rows = defaultdict(lambda: [0.0] * 4)
    with open(path) as f:
        for r in csv.DictReader(f, delimiter="\t"):
            k = (r["Function"], r["Module"], r["Source File"])
            rows[k] = [a + float(r[c] or 0) for a, c in zip(rows[k], COLS)]
    return rows

def base(fn):
    # gfortran outlines each parallel region as <routine>._omp_fn.N
    return re.sub(r"\._omp_fn\.\d+$", "", fn)
```

Report the totals for the four columns first, then the change in CPU time per `Module`, per source file (`gcclassic` module only), and per routine with `base()` applied.

## 3. Separate compute from waiting

- **Effective time** in `gcclassic` is the real work. If it is unchanged within about 1%, the loop changes did not reduce the computation, and any change in wall time comes from scheduling, imbalance, or I/O.
- **Spin time** is all charged to `libgomp.so.1`, so the flat CSVs cannot say which loop caused it. In libgomp, `gomp_simple_barrier_wait` and `do_spin` are worker threads docked between parallel regions or spinning while the master runs serial code. `gomp_team_barrier_wait_end` and `gomp_team_end` are threads waiting at the end of a region. `gomp_iter_dynamic_next` and `gomp_iter_guided_next` are the dispatch cost of `DYNAMIC` and `GUIDED` schedules; add them up before and after, because moving loops from `DYNAMIC` to `GUIDED` just moves that cost from one function to the other.
- Worker spin is CPU time and costs no wall time on its own. **Master-thread spin costs wall time directly**, because the master cannot proceed until the region ends.

## 4. Attribute master-thread spin to call sites (no rerun needed)

Generate per-thread and master-thread call-tree reports from the existing result directories:

```bash
vtune -report hotspots -r <dir>/r000hs -group-by thread -format csv -csv-delimiter tab -report-output thread.csv
vtune -report top-down -r <dir>/r000hs -filter "thread=gcclassic (TID: <master tid>)" -format csv -csv-delimiter tab -report-output topdown_master.csv
```

The master is the thread named `gcclassic (TID: …)`; the others are `[OpenMP worker]`. From `thread.csv`, compare master CPU, Effective, and Spin, and the worker mean, min, and max. Wall time minus master CPU is time the master spent off CPU, which is mostly I/O.

In `topdown_master.csv`, the depth of each frame is its count of leading spaces, and the columns hold percentages of that thread's total. For every row with a non-zero `CPU Time:Spin Time:Self`, walk up to the nearest ancestor whose `Module` is `gcclassic`, convert the percentage to seconds using the master's spin total, and add it up per (caller, libgomp leaf). The callers whose spin grows from ref to dev are the regressed regions. Usually a few of them map onto the GEOS-Chem timer categories that slowed down.

## 5. Set a noise floor before claiming anything

Find routines whose source did not change in the diff, such as the KPP `kppdecomp`/`kppsolve` routines, `unitconv_mod`, or whatever the diff did not touch, and note how much their CPU time and their timers moved. Do not report a gain or loss smaller than that as caused by the loop changes. A single profile pair on two different nodes commonly shows 5–20% swings in unchanged code.

If a timing-only pair of runs exists, check whether each timer moves the same way there. A change that repeats in both pairs is much stronger evidence than one pair alone.

## 6. Patterns to recognize

- **A whole-routine symbol replaces `_omp_fn` symbols.** For example, `__vdiff_mod_MOD_vdiff._omp_fn.0/1` disappear and the plain `vdiff` symbol grows. This means nested (orphaned) parallel regions were removed. With nested parallelism off, those regions ran on one thread anyway, so removing them saves CPU time even though no thread count changed.
- **Adding only `SCHEDULE(STATIC)`** to a loop that had no schedule clause compiles to the same code under libgomp, whose default is static. Any timing change at such a loop is noise.
- **`COLLAPSE` on a latitude-only loop.** At 4x5 there are 46 J values for 48 threads, so before the change some threads sat idle and the work was unbalanced. Collapsing gives real wall-time gains even when total CPU time rises, because the idle threads now do work. The Cloud-J photolysis loop is the typical case.
- **Many short parallel regions in a row.** For example, a `PARALLEL DO` inside a `DO L` or `DO N` loop. Here the fork/join cost dominates, and `COLLAPSE` or `GUIDED` can make it worse through extra index arithmetic, dispatch cost, and false sharing where chunks split a row. The fix is to hoist the parallel region outside the outer loop, putting the independent column or species loop inside it. Swapping the schedule only tunes a loop that should not be structured that way.
- **CPU time down but wall time up** in the same component means the change is in I/O or spin, not in the loop's computation.

## 7. Report

Lead with the bottom line: the change in wall time, how much of it is compute and how much is waiting, and which differences you cannot attribute because of the issues found in step 1. Then list:

- Improvements you can attribute, each tied to a loop change and its evidence.
- Regressions you can attribute, with the call site, the loop change, and a suggested fix.
- Differences you cannot attribute: noise, I/O, or confounded submodules.

End with the reruns needed to firm up the conclusions: matched submodules, 3 or more repeats per side on the same node with alternating run order, and the step-4 reports.

For auditing the loops themselves before or after profiling, see the `openmp-collapse-schedule-audit` skill.
