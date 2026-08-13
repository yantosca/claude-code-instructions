# GEOS-Chem Publications Lookup

Instructions for pulling new GEOS-Chem-related publications on request. This
documents the *method*, not a snapshot of results — regenerate the list fresh
each time rather than reusing an old output.

## Source

The official aggregator is the Google Scholar profile "GEOS-Chem publications":
https://scholar.google.com/citations?user=ho-sNj4AAAAJ (verified homepage:
geoschem.github.io). This is curated/community-maintained, not a raw keyword
search, so it's the right source when asked for "new GEOS-Chem publications."

## Method that works

1. Fetch `https://scholar.google.com/citations?user=ho-sNj4AAAAJ&sortby=pubdate&cstart=0&pagesize=100`
   via WebFetch, sorted newest-first. Ask explicitly for an exhaustive list
   (title + citation-page URL) restricted to the target year(s) — the
   summarizing model will silently truncate/abbreviate a 100-entry page if
   not told to be exhaustive.
2. Limitation: this profile view shows **year only**, not month/day, per
   entry. It cannot support a precise "since \<specific date\>" cutoff within
   a year — flag this caveat rather than guessing a boundary. AMS/EGU
   conference abstracts cluster near the start of a year, which can help
   bound the estimate but isn't a substitute for exact dates.
3. For DOIs: **do not** re-scrape Scholar's individual citation pages
   one-by-one (slow, redundant, riskier for Scholar rate-limiting/CAPTCHA).
   Instead query the free Crossref API directly
   (`https://api.crossref.org/works?query.bibliographic=<title>`) via a
   Python/curl script — no auth needed, include a `User-Agent` with a
   `mailto:`.
   - Watch for two failure modes in the top match: (a) ACS-journal
     supplementary-info records (DOI suffix `.sNNN`) often outscore the real
     article because titles are identical — strip the suffix and verify the
     base DOI resolves via `api.crossref.org/works/<doi>` before using it;
     (b) AMS Annual Meeting abstracts usually have **no** Crossref DOI at
     all — report "no DOI found" rather than accepting a low-score/
     mismatched-title match (score well below ~50 with a mismatched title
     is the tell).
4. When asked to include links, write the final table with markdown
   hyperlinks: DOI linked to `https://doi.org/<doi>`, and a "Scholar entry"
   column linked to the citation-page URL for each entry.

## Automation stance

Default to a one-time, on-demand pull rather than proposing a reusable
script or scheduled/recurring job, unless explicitly asked for automation.

**Why:** Google Scholar has no official API, actively CAPTCHA-blocks
repeated automated requests (observed firsthand mid-session on a plain
date-sorted search), and its ToS prohibits scraping. A recurring job would
need a paid API (e.g. SerpApi) or proxy setup to be reliable — more
infrastructure than the ad-hoc use case has called for. When given the
choice, one-time-pull-only was preferred over both a reusable script and
scheduled automation.

**How to apply:** When asked for GEOS-Chem publication updates again, just
perform the pull interactively and report results — don't default to
suggesting a script or cron job. Only build automation if asked.
