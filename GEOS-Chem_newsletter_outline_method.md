# GEOS-Chem Newsletter Outline Method

Instructions for drafting an outline for the *next* GEOS-Chem newsletter
issue on request. This documents the *method*, not a snapshot of results —
re-fetch and re-diff against whatever the actual latest issue is each time,
rather than reusing an old outline.

## Sources (restrict to these unless told otherwise)

1. **Most recent newsletter issue** — currently hosted as a PDF on Google
   Drive (get the current link from the user; don't guess or reuse an old
   one, the link changes per issue).
2. **The wiki page for the GEOS-Chem version currently "in development"**
   (e.g. `https://wiki.seas.harvard.edu/geos-chem/index.php?title=GEOS-Chem_X.Y.Z`)
   — the full list of updates/fixes for that version.
3. **https://geoschem.github.io/** (main site) — for upcoming meeting
   announcements; follow one level deep into a linked meeting sub-page if
   the homepage references one, but no further afield than that.
4. **Google Scholar publications lookup** — see
   [[GEOS-Chem_publications_lookup]] for the documented method.

## Retrieving the Google Drive newsletter PDF

- The `/view` URL only serves a viewer shell (thumbnail + sign-in prompt)
  — no real content comes back from fetching it directly.
- The `.../uc?export=download&id=<id>` URL 303-redirects to
  `drive.usercontent.google.com/download?...`; a fetch tool that follows
  redirects will download the actual PDF this way — but its auto-summary
  of the PDF can still be poor or lossy.
- These newsletters are Google Docs exports with subsetted, glyph-encoded
  fonts, so plain text extraction from the raw PDF bytes can come out
  garbled. If an auto-summary looks incomplete/garbled, read the raw PDF
  bytes directly and decode the embedded font ToUnicode CMaps to recover
  verbatim text rather than trusting a lossy summary.

## Method

1. Fetch the most recent newsletter and extract: issue number/date, its
   section structure (historically ~5 numbered sections: Featured News;
   Versions In Development; Model Development Priorities; Previous
   Version Releases; Recent Publications), and — critically — every
   specific news item/version detail/meeting/publication it already
   covered, so the new outline doesn't repeat it as "new."
2. Fetch the in-development version's wiki page for its current full
   feature/fix list, and diff it against what the last newsletter already
   reported for that same version. Only the delta is "new since last
   issue" — the wiki page accumulates everything merged so far, most of
   which was likely already announced.
3. Fetch geoschem.github.io for upcoming meeting announcements; diff
   against what the last newsletter already announced.
4. Run the publications lookup method (see
   [[GEOS-Chem_publications_lookup]]) and explicitly cross-check candidate
   papers against the last issue's actual (already-published) list —
   Scholar's "most recent" results can overlap almost completely with
   "already reported" if not deduped against the real prior-issue text.
5. Flag, rather than fabricate, anything that needs information outside
   these four sources (e.g. a just-concluded meeting's actual outcomes/
   recap, or Steering Committee decisions) — surface these as open
   questions for the user/GCST, don't invent plausible-sounding content.

## Format conventions observed in past issues

- Numbered top-level sections with nested bullets.
- Inline GitHub PR references in parentheses, e.g. "(geos-chem PR #3218)".
- Bracketed citations "[Author et al., Year]" when citing a paper inline
  (separate from the numbered Recent Publications list at the end).
- Sign-off naming the current GCST members by first name.

**Why this matters:** skipping the diff-against-prior-issue step produces
an outline that re-lists things readers already saw. This happened on a
first pass — 5 of 6 initial publication candidates, and several
"new-sounding" version-update items, turned out to already be in the prior
issue once its actual text was checked.

**How to apply:** always fetch and read the previous issue in full *before*
drafting anything else from the other three sources, and treat its actual
content (not memory or assumption) as the source of truth for "already
covered."
