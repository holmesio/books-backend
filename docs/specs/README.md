# Specs

Each spec is the contract for one unit of work, usually one PR. Read a spec before touching the code it covers.

## Product
**For readers paralysed by too many books: narrow the pile to a few you'll actually read, keep the rest tidy, and
never turn reading into a score.**

Lifecycle: **Backlog (30) → Reading (5) → finished or DNF.** Swiping happens when a Reading slot opens.

The core loop is **triage**, not discovery. Other apps help readers grow a TBR (swipe-discovery apps) or store one
(Goodreads, StoryGraph, Hardcover). This app helps them shrink it.

Principles. Check every feature against these; they are reasons to say no:
1. **Constraints are the product.** Backlog and Reading are deliberately capped. Limits are features, not missing functionality.
2. **No gamification.** No streaks, yearly goals, badges or reading stats. Reading for reading's sake.
3. **No social layer.** No feeds, followers, likes or public reviews.
4. **Start small and stay small.** No bulk import. Readers add books one at a time via search, so the TBR never
   starts out overwhelming. Swiping 30 books is triage; swiping 300 is a chore.

## Roadmap

| Spec | What | Depends on | Status |
|------|------|------------|--------|
| [0001](0001-reading.md) | Reading: capped set of current reads, pure domain rules | none | draft |
| [0002](0002-backlog.md) | Backlog + shelf aggregate (start, finish/DNF, history): pure domain | 0001 | stub |
| 0003 | Postgres persistence of user state (Backlog, Reading, history) | 0001, 0002 | not started |
| 0004 | Catalog: search an external book API, cached in Postgres (manual search is how books get in) | 0003 | not started |
| 0005 | HTTP API for the mobile client | all above | not started |

Rejected: a separate UpNext queue between Backlog and Reading (one list too many for an anti-overwhelm app);
a "pause" state (a stalled book must be finished or DNF'd). Note: with Reading at 5 the cap nudges less; see the stale-reading question in 0002.

Missing: an **auth** spec, and a decision on **what client** consumes the API (mobile, thin web, CLI).

Rejected (for now): community vibe/pacing checks, i.e. aggregating readers' post-read tags to show on search results.
It needs many readers tagging the same books (no value at small scale), StoryGraph already shows community mood and pace, and
it would require loosening principle 3. It also can't live in `tbr` (cross-user, see 0001 #1). **Reopen if** the app
has enough active readers; if so, count only post-read tags (never predictions) and show a book's aggregate only after N readers.

Rejected: bulk import from Goodreads/StoryGraph/Hardcover. It conflicts with principles 1 and 4 and the 30-book Backlog cap.

Status values: `not started` → `stub` → `draft` → `accepted` (decisions frozen) → `done` (acceptance green).

## Architecture rules
- Import arrows point inward: `postgres`, `catalog` and `http` may import `tbr`. **`tbr` imports only the standard library.**
- Interfaces are declared by the package that *uses* them, not the one that implements them.
- The database is the source of truth for user state. In-memory locks protect one process only. Cross-request
  invariants (capacity, duplicates) must also be enforced in Postgres. See 0003.

## Settled across specs
- `Book.ID` is an opaque, non-empty string to the domain. Its source (planned: the external API's **work-level** ID,
  namespaced, e.g. `ol:OL45883W`) is decided in 0004. ISBN identifies an edition, not a work, so it is optional
  metadata only, never identity.

## Conventions
- File name: `NNNN-<slug>.md`. Numbers are never reused.
- Every spec has: Goal (one sentence) · Scope / Out of scope · Decisions table (choice + Why) · Public API ·
  Behaviors → test case names · Acceptance.
- Decision numbers are stable. Never renumber; mark rows as moved or superseded and link to where they went.
- If a decision looks wrong during implementation, stop and raise it. Don't silently diverge from the spec.
- Test case names are written in the spec before any test code exists.
