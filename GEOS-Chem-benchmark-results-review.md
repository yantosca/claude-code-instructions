# GEOS-Chem Benchmark Results Review (GCClassic and GCHP)

Instructions for reviewing a GEOS-Chem or GCHP version-comparison benchmark
(e.g. "what changed between alpha.19 and alpha.20") and tying the diff
pattern back to the PR that caused it.

## Entry point

Benchmark runs are indexed at `gc-dashboard.org`:

```
https://gc-dashboard.org/difference?primary_key=diff-<ref-run>-<dev-run>
```

`primary_key` examples:
- GCHP: `diff-gchp-c24-1Mon-14.8.0-alpha.19-gchp-c24-1Mon-14.8.0-alpha.20`
- GCClassic: `diff-gcc-4x5-1Mon-14.8.0-alpha.19-gcc-4x5-1Mon-14.8.0-alpha.20`

The dashboard page shows run metadata (status, site, timestamps) and a
"Public Artifacts" section linking to every plot/table for that run. If the
exact filename/timestamp of an artifact is unknown, WebFetch the dashboard
page first to get the exact linked URL rather than guessing one.

## Underlying S3 layout

Artifacts live at `s3://benchmarks-cloud/diff-plots/<period>/<primary_key>/BenchmarkResults/...`.
GCHP paths have an extra `GCHP_version_comparison` segment that GCClassic
paths lack:

- GCHP tables: `.../BenchmarkResults/GCHP_version_comparison/Tables/<name>.txt`
- GCClassic tables: `.../BenchmarkResults/Tables/<name>.txt`
- GCHP category plots: `.../BenchmarkResults/GCHP_version_comparison/<Category>/<Category>_<Level>.pdf`
- GCClassic category plots: `.../BenchmarkResults/<Category>/<Category>_<Level>.pdf`

`<Category>` examples: Aerosols, Bromine, Chlorine, Iodine, Nitrogen,
Oxidants, Primary_Organics, ROy, Secondary_Organic_Aerosols,
Secondary_Organics, Sulfur. `<Level>` examples: Surface, 500hPa,
FullColumn_ZonalMean, Strat_ZonalMean. Fetch these directly with
`curl`/Read rather than asking the user to paste contents — the text
tables run 300-450 lines and the PDFs are a few pages each.

## Step 1: Global mass tables

Start with `GlobalMass_TropStrat_<timestamp>.txt` (Trop+Strat) — it lists
every species' Ref mass, Dev mass, Dev-Ref, and % diff in one pass.

1. Read the header's "N differences" count out of ~400 species. A high
   count (e.g. 389/394) means the change is *broad-but-shallow* (nearly
   everything nudges a little); a low count means *narrow-but-deep* (a
   handful of species jump, the rest sit at exact 0.000000%).
2. Ignore `nan` rows (Ref mass ~1e-16 or smaller — division noise) and
   exact-zero rows (species genuinely unaffected, e.g. H2/N2/O2 in GCHP).
3. Rank by `|% diff|`, not by absolute mass diff — a huge absolute diff on
   a huge-burden species (CO2, CH4) can be a smaller % diff than a tiny
   absolute diff on a trace species.
4. Report both ends: the largest-magnitude % diffs (what's driving the
   change) and the smallest nonzero ones (confirmation nothing broke for
   inert/well-mixed tracers).

## Step 2: Look for a chemical-family pattern, cross-check the PR

Pull the PR's own description (`gh pr view <N> --repo geoschem/geos-chem
--json body`) and check its "Expected changes" section against the pattern
in the largest movers:

- Long-lived, well-mixed, or chemically inert species (CFCs, HCFCs, CH4,
  N2O, CO2, H2, N2, O2) barely moving → consistent with a change to fast
  kinetics/met-field details, not transport or emissions.
- Short-lived radicals, peroxides, organonitrates, RO2-family
  intermediates moving the most → consistent with a change to
  temperature- or humidity-dependent reaction rates.
- Aerosol species with strong thermodynamic equilibrium partitioning
  (e.g. NIT via ISORROPIA) are disproportionately sensitive to T/RH
  changes.
- A PR noting "no diff for GEOS-Chem Classic, diffs for GCHP only" points
  at GCHP-specific meteorology plumbing (`Interfaces/GCHP/`), not shared
  chemistry code.

## Step 3: Read the plot PDFs — the significant differences are the key

The mass table tells you *how much* changed; the category plot PDFs tell
you *where*, which is usually what identifies the mechanism. Each species
panel has 6 maps: Ref, Dev, Difference (Dynamic Range), Difference
(Restricted Range [5%,95%]), Ratio (Dynamic Range), Ratio (Fixed Range).

**The "Restricted Range [5%,95%]" difference map is the one to read** — it
clips the outlier grid cells that dominate and wash out the "Dynamic
Range" panel's color scale. Interpret its *spatial pattern*, not just its
magnitude:

- **One dominant sign nearly everywhere** (e.g. a species increasing
  almost uniformly over all land) → a systematic shift, not noise.
- **Localized to a specific physical regime** (e.g. an anomaly
  concentrated over deserts, or over ocean, or at high latitudes) → a
  mechanistic fingerprint. Match the regime to what physically varies
  there (humidity, temperature, insolation, boundary-layer depth) to
  identify the causing change.
- **Scattered, sign-mixed checkerboard** → a smaller, more buffered/
  downstream response rather than a direct driver.

Example: for PR #3373 (GCHP `State_Met%T`/`SPHU` changed from
timestep-midpoint to post-advection values), the Oxidants/Surface plot
showed OH with the largest fractional restricted-range signal of the
oxidants (~±0.25%), as a negative patch specifically over the
Sahara/Middle East — consistent with OH's O(¹D)+H₂O source being
humidity-sensitive, and pointing straight at the humidity-timing change
rather than an emissions or transport bug.

## Environment note: rendering PDFs

Some sandboxes have no `poppler-utils` and no passwordless `sudo`, so the
PDF-page-rendering step (`pdftoppm`) fails by default. Fix once per
machine via conda (no sudo needed):

```
conda install -n base -c conda-forge -y poppler
ln -sf ~/miniforge3/bin/pdftoppm ~/miniforge3/bin/pdfinfo ~/miniforge3/bin/pdftocairo ~/.local/bin/
```

(Symlinking is needed because conda's `bin/` isn't on `PATH` by default —
only `condabin/` is.) After that, reading a PDF works directly; use a page
range for anything over 10 pages.
