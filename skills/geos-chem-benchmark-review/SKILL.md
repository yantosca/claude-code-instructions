---
name: geos-chem-benchmark-review
description: Review a GEOS-Chem Classic or GCHP version-comparison benchmark and trace a difference back to the PR that caused it. Use when asked what changed between two GEOS-Chem or GCHP versions or benchmark runs (e.g. 14.8.0-alpha.19 vs alpha.20), to interpret benchmark difference plots, global mass tables, or gc-dashboard.org artifacts, or to explain why a species mass or concentration shifted between two runs.
---

# GEOS-Chem Benchmark Results Review (GCClassic and GCHP)

Instructions for reviewing a GEOS-Chem or GCHP version-comparison benchmark
(e.g. "what changed between alpha.19 and alpha.20") and tying the diff
pattern back to the PR that caused it.

## Step 0: Find out what is in the release

One alpha tag can bundle several PRs. Before attributing a diff, list them from a GEOS-Chem source checkout and attribute the benchmark to the whole set, not just the PR the user names:

```
git log --oneline <prev-tag>..<tag>     # e.g. 14.9.0-alpha.4..14.9.0-alpha.5
git diff <prev-tag>..<tag> -- KPP/fullchem/*.eqn KPP/fullchem/fullchem_*.F90
```

Example: 14.9.0-alpha.5 contained both PR #3409 (HMS rate update) and PR #3430 (which built on it).

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

### Listing artifacts without the dashboard

The bucket is publicly listable, which gives exact filenames (table timestamps, GCHP-only or GCC-only tables) without parsing the dashboard page:

```
curl -s "https://benchmarks-cloud.s3.amazonaws.com/?list-type=2&prefix=diff-plots/1Mon/<primary_key>/BenchmarkResults/" \
  | grep -o '<Key>[^<]*</Key>' | sed 's/<\/\?Key>//g'
```

Files are then at `https://benchmarks-cloud.s3.amazonaws.com/<key>`. Results are capped at 1000 keys per listing, so narrow `prefix` to `.../Tables/` if the list looks truncated. Download into a fresh scratch directory.

Useful tables beyond the mass table: `OH_metrics.txt` (global mean OH, MCF and CH4 lifetimes) and `Summary.txt` (which diagnostic collections differ), both present for GCClassic; `Benchmark_Timers_*.txt` and `Diagnostic_Sanity_Check.txt` for both models. Use the mass table at the *end* of the run (`GlobalMass_TropStrat_20190801T00.txt` for a July 1-month benchmark), not the initial one.

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

Parsing recipe: rows end in a `*` flag column, so the % diff is field 6, not `$NF`. Rank by magnitude, skipping `nan`:

```
awk '$2==":" && $6!="nan" {v=$6+0; a=(v<0?-v:v); if(a>0) print a, $0}' GlobalMass_TropStrat_*.txt \
  | sort -gr | head -25 | cut -d' ' -f2-
```

Swap `sort -gr` for `sort -g` to see the smallest nonzero diffs. Values around 1e-14 % are round-off and confirm unrelated species (dust, BC) did not move.

Production/loss tracers (`PSO4MP`, `LHMSMP`, `PHMSAQ`, `LHMSAQ`, `PSO4AQ`, ...) are accumulated diagnostic species, not real burdens. A large % change in one is evidence of a flux shifting between pathways, which is often the most direct readout of what a kinetics change did.

**Cross-model check.** When both GCClassic and GCHP benchmarks exist, compare their tables side by side. Matching % diffs (e.g. HMS −24.4% vs −24.3%) and matching spatial patterns confirm a real, model-independent effect. Timer changes are much noisier: a ±10% swing in one model's chemistry timer that the other model does not show is node noise, not a speedup.

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

Example: for 14.9.0-alpha.5 (PRs #3409 + #3430, HMS + SO2 oxidation rate now using a constant aqueous OH of 6.3e-15 M), the Sulfur/Surface HMS page showed a decrease everywhere. The absolute drop was largest where the burden is (E. China, India, E. US), but the Dev/Ref ratio bottomed out at ~0.4–0.6 over the Sahara and Middle East, the driest regions. That fingerprint pointed at the new rate being applied with no aerosol-liquid-water or RH gating (and no day/night cycle). Reading the old code also showed the original bug: the old rate used `hydroxide = 10**(pH-14)`, the OH⁻ *ion*, as if it were the OH radical. SO4 rose +1.2% globally, matching the PR's "~1%" expectation, with the usual NH4 up / NIT down / NH3 down ISORROPIA response.

### Finding the right PDF page

Each category PDF has one species per page. Map pages to species before rendering, then render only the pages you need at low resolution:

```
for p in $(seq 1 $(pdfinfo f.pdf | awk '/Pages/{print $2}')); do
  echo "p$p: $(pdftotext -f $p -l $p f.pdf - | grep -o 'SpeciesConc[A-Za-z]*_[A-Za-z0-9]*' | sort -u)"
done
pdftoppm -f 3 -l 3 -r 60 -png f.pdf out     # -r 60 is enough to read the maps
```

## Environment note: rendering PDFs

On Harvard Cannon, `pdftoppm`, `pdftotext`, and `pdfinfo` are already in `/usr/bin`, so the workaround below is only needed on other machines.

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
