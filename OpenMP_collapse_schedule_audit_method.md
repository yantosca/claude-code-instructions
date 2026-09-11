# OpenMP COLLAPSE/SCHEDULE Audit Method

Instructions for auditing a Fortran/OpenMP codebase to find `!$OMP DO`/
`!$OMP PARALLEL DO` loops that could take a `COLLAPSE(n)` clause, and for
checking whether existing `SCHEDULE` clauses fit each loop's workload.
This documents the *method*, not a snapshot of results — re-run it fresh
against the current state of the codebase each time; don't reuse an old
audit's verdicts.

This is a research/reporting task by default: produce a list, don't edit
code or commit anything unless explicitly asked to. The user typically
wants to verify and apply any actual changes themselves, module by
module.

## Scoping before you start

1. Confirm exactly which directories/submodules are in scope. Large
   scientific Fortran codebases often bundle multiple logically-separate
   components (e.g. a chemistry model plus a separately-maintained
   emissions/coupling library) — the user may want only one audited now
   and the rest later, or specific subcomponents excluded entirely.
   Don't assume; ask or confirm if it's unclear which subfolders count.
   An exclusion from a *prior* audit (e.g. "we excluded module X per
   request last time") does not automatically carry forward — re-confirm
   it explicitly for this run rather than silently inheriting it.
2. Don't assume the directory layout matches a prior audit, even on a
   supposedly-related branch — top-level components get reorganized
   between branches/versions (e.g. a subsystem moving out of a shared
   "core" directory into its own top-level folder). List the actual
   top-level directories fresh (`ls -d */`) and re-derive scope from
   that, rather than reusing a remembered file/directory list.
3. Get a size estimate per subdirectory before diving in:
   `grep -rliE '!\$OMP' <dir>` for file counts, and
   `grep -rciE '!\$OMP PARALLEL' <dir>` summed per directory for a rough
   directive-count estimate. This tells you where the bulk of the work
   is and whether a single pass or parallel batches make sense.
4. A directory showing OMP-tagged files but zero `PARALLEL` matches
   often means something structurally different is going on (e.g. only
   `THREADPRIVATE` declarations for solver state, with the actual
   parallel loops living elsewhere) — verify what's actually there
   before assuming it's in scope for this kind of audit; report plainly
   if there's genuinely nothing to evaluate rather than forcing findings.

## Splitting the work

For anything beyond a handful of files, delegate to parallel
general-purpose agents rather than doing it all in one context — this is
an *analytical* task (reading loop bodies for correctness, not just
pattern-matching), so use general-purpose/analysis-capable agents, not a
fast read-only search agent that isn't meant for open-ended analysis.

- Balance batches by **directive count, not file count** — a handful of
  files can dominate the directive total. A simple greedy approach
  (sort files by directive count descending, assign each to whichever
  batch currently has the smaller running total) gives good balance
  without needing to be exact.
- Give each batch agent the exact file list you assigned it, but also
  tell it to double-check for files with matching directives that
  weren't on its list (or its sibling batch's list) — pre-computed
  splits from a rough grep count can be slightly off, and files get
  missed at the boundary otherwise.
- After all batches report, reconcile: sum "directives reviewed" across
  agents and compare to your original grep-based estimate. Chase down
  any gap before treating the audit as complete. The most common
  explanation for a gap is a **disabled directive** the grep estimate
  counted but that isn't actually live OpenMP — a directive commented
  out with an extra `!` (`!!$OMP ...`, or an indented `!    !$OMP ...`).
  Ask each batch agent to explicitly list any such disabled directives
  it finds in its assigned files (file:line, one-line reason) so gaps
  reconcile cleanly instead of looking like a miscount.
- If an agent's own sub-task grows large enough to need further
  splitting, that's fine, but make sure whatever it produces gets
  reported all the way back up — watch for a sub-agent's result arriving
  as its own separate notification if it couldn't reach its intended
  parent (e.g. an addressing failure between agent layers); don't lose
  or silently drop that data when compiling the final list.

## COLLAPSE eligibility criteria

`COLLAPSE(n)` requires **n perfectly-nested loops** immediately under the
directive:
- No executable statements between the loop headers themselves (only the
  `DO` statements — declarations/comments are fine).
- Each outer loop's body must be *only* the next inner loop — nothing
  between the inner loop's end and the outer loop's end either.
- Loop bounds may be affine functions of outer indices (OpenMP allows
  non-rectangular loop nests in `COLLAPSE`), but flag these for extra
  scrutiny rather than treating them as automatically safe.

**Flag as NOT collapsible, with the specific reason, when:**
- Code (assignments, calls, IF-guards) sits between loop headers or
  between an inner loop's end and its enclosing loop's end.
- A `CYCLE`/`EXIT`/`GOTO`/`RETURN` would behave differently once the
  loops are flattened into one iteration space — especially an
  **unlabeled EXIT inside what otherwise looks like a clean nest**; this
  is an easy one to miss and a real correctness hazard if collapsed
  mechanically.
- A loop-carried dependency exists on what would become one of the
  collapsed dimensions (e.g. a running accumulator like `total(i,j) =
  total(i,j) + ...` inside the innermost loop) — the *other* dimensions
  may still be safely collapsible, just not the one carrying the
  dependency.
- There's genuinely only one loop under the directive (nothing to
  collapse).
- A GOTO-based search pattern with loop-carried search state (e.g. a
  starting index that persists across outer iterations) — common in
  hand-written interpolation/remapping code; this also usually means the
  outer loop's body isn't "just the inner loop," compounding the issue.

**Flag as "possible but needs manual verification"** when the nest looks
structurally clean but something else warrants a second look: a called
subroutine/function whose side effects on shared state aren't visible
from the file alone, an OpenMP `REDUCTION` clause present, non-rectangular
bounds, or a directive that's the target of a conditional compilation
(`#ifdef`) making its real shape hard to assess from one build path.
Also watch for a directive whose clauses look like they were **copied
from a different, similar loop nearby** — check that the loop actually
matches the clause (a mismatched copy-paste can produce something that
isn't valid OpenMP for the loop it now decorates; that's a latent bug
worth flagging on its own, separate from any tuning recommendation).

## Data-race check (independent of COLLAPSE/SCHEDULE)

For every directive, also check that every variable *written* inside the
loop body is actually safe under `DEFAULT(SHARED)` — this is a distinct
check from loop-nest shape and surfaces real bugs at roughly the same
rate as EXIT/accumulator hazards do:
- A scalar or small array written inside the loop (directly, or via an
  `INTENT(OUT)`/`INTENT(INOUT)` argument to a called subroutine) but
  **missing from the `PRIVATE` clause** is a data race, even if every
  thread happens to write the same value (still undefined behavior, and
  wasteful). This is easy to miss because the write often isn't in the
  loop nest's own executable statements — it's inside a called
  subroutine, so you have to check what that subroutine's output
  arguments are, not just what the loop body itself assigns.
- A running total/counter updated via `x = x + ...` (or similar) on a
  shared, non-reduction variable is a race that needs either
  `REDUCTION(+:x)` or removal if the accumulated value is actually dead
  code (check whether it's ever read after the loop — dead accumulators
  turn up surprisingly often and are also just wasted compute).
- Flag these prominently in "notable findings," the same as EXIT/
  copy-paste-clause bugs — they're correctness issues independent of
  whatever COLLAPSE/SCHEDULE verdict the directive gets.

## SCHEDULE evaluation

Only recommend a change where the loop body's cost profile clearly
mismatches its current clause:
- **`DYNAMIC`/`GUIDED` on a cheap, uniform-cost body** (simple arithmetic,
  no conditional expensive calls) → suggest **STATIC** — the scheduling
  overhead isn't earning its keep.
- **`STATIC`/no clause on a body with real per-iteration cost variance**
  (a `CYCLE`/`EXIT` that skips most iterations under some condition,
  conditionally-called expensive solvers, day/night or land/ocean
  branches, variable-length inner search loops) → suggest
  **DYNAMIC or GUIDED**.
- Otherwise, if you can't tell the actual cost variance from static
  reading alone, say so explicitly ("no change / needs profiling to
  decide") rather than guessing a direction with unearned confidence.

## Output format

One consolidated list, grouped by directory then file, with one entry per
directive: file:line, current clauses, one-line loop-structure summary,
COLLAPSE verdict + reason, SCHEDULE verdict + reason. Pull out a short
"notable findings" section up front for anything that looks like a
pre-existing bug or correctness risk rather than a tuning opportunity —
those are worth surfacing prominently, separate from the bulk tuning
list, since they matter regardless of whether the user ever applies any
of the COLLAPSE/SCHEDULE suggestions.

State each verdict as an unambiguous **action**, not just a confidence
label — the user reading this report wants to know exactly what to type,
not just how sure you are. Prefer phrasing like "ADD COLLAPSE(3)",
"CHANGE SCHEDULE(DYNAMIC) → SCHEDULE(STATIC)", "No change — already
COLLAPSE(2), correct", or "POSSIBLE COLLAPSE(2) but NEEDS MANUAL
VERIFICATION — confirm `<specific thing>`" over vague language like
"could be collapsed" or "consider revisiting the schedule." If a change
needs a caveat or a check before applying, put the caveat in the same
line as the action, not as separate prose the reader has to reconcile
with the verdict themselves.

**Why this matters:** real findings from applying this method (kept
generic here on purpose) include an unlabeled `EXIT` inside an
otherwise-clean-looking nest, a sequential accumulator on what would
have been the innermost collapsed dimension, a directive whose clauses
had clearly been copied from a different loop and no longer matched the
loop they decorated, and — a distinct and recurring category — a shared
scalar written inside the loop (often via a called subroutine's output
argument) that was missing from the `PRIVATE` clause, a genuine data
race invisible from the directive text alone. All of these would have
been missed by pattern-matching on the directive text alone — they only
surface from actually reading the loop body (and, for the last one,
checking what any called subroutines write into).
