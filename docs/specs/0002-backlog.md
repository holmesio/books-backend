# 0002: Backlog and the shelf

Status: stub. Don't start until 0001 is green.
Package: `internal/tbr`
Depends on: [0001: Reading](0001-reading.md)

## Goal
<!-- TODO(you): one sentence for this spec. -->

## Product context
- The Backlog is the TBR in its pure form: up to 30 books the reader might read.
- Lifecycle: **Backlog (30) → Reading (5) → finished or DNF**. There's no pause; the only ways out of Reading are finish or DNF.
- The Reading cap is the nudge: to start something new with all slots full, the reader must finish or DNF a book first.
- DNF is a legitimate, guilt-free outcome, not a failure.
- Mobile swipe UX. Swiping happens **when a Reading slot opens**:
  - Right swipe starts reading the book (Backlog → Reading).
  - Left swipe passes: the book stays in the Backlog and its pass count goes up.
  - The X button deletes the book from the Backlog.
  - After repeated passes, the app asks: "You passed this book N times. Would you like to remove it from the TBR?"

## Decisions (proposed; Why lines are TODO(you))
| # | Area | Decision | Why |
|---|------|----------|-----|
| 15 | Backlog cap | Hard cap of 30 books (`DefaultBacklogCapacity = 30`, raised from 15). Adding a 31st returns an error | 15 felt too tight in practice; 30 still keeps the TBR triage-sized. TODO(you): refine |
| 16 | Left swipe | A pass *is* domain state: it increments the book's pass count for this reader | TODO |
| 17 | Stale books | A book is stale when passes ≥ `StalePassThreshold` (3). The domain reports staleness; the prompt text is UI | TODO |
| 18 | Keep after prompt | Choosing **Keep** resets the book's pass count to 0 | TODO |
| 19 | Pass count home | The Backlog holds `Entry{Book, Passes}`. `Book` stays pure catalog data. The reader stays implicit (#1) | TODO |
| 20 | Threshold owner | The domain owns `StalePassThreshold` and reports staleness. Clients only choose the wording | TODO |
| 22 | Outcomes | A book leaving Reading is either finished or DNF. The domain keeps a minimal record: `Book`, outcome, date | TODO |
| 23 | Time | The domain never calls `time.Now()`. Callers pass the time in (e.g. `Finish(id, at time.Time)`) | TODO |
| 24 | Reader tags | **Superseded by #32, then #37** (tags on `Entry` only) | |
| 25 | Tag values | Moved to [0001](0001-reading.md) (Reading needs `Tags` first) | |
| 26 | Vibe limits | Moved to [0001](0001-reading.md) | |
| 27 | Genre vs vibe | Independent dimensions. No genre→vibe rules in the domain; genre-based vibe suggestions are a UI concern | TODO |
| 28 | Post-read tags | **Superseded by #31** | |
| 31 | Post-read tags | **Superseded by #33** (was two tag sets: prediction on Entry, actual on Record) | |
| 32 | Tags home | **Superseded by #37** (was a Shelf-owned tags map) | |
| 37 | Tags travel | Tags are held by whatever holds the book: `Entry` (Backlog), `Item` (Reading, 0001 #34), `Record` (history). `Start`, `Finish` and `DNF` carry them across with `Tags.Clone()`. Tags can be set when adding to the Backlog. `Shelf.SetTags(id, t)` finds the book and delegates | No separate store to keep in sync, so deleting a book deletes its tags wherever it is. `Backlog.Filter` stays self-contained. TODO(you): refine |
| 33 | Tag lifecycle | Tags are editable at any time, wherever the book is: Backlog, Reading or history (finished *and* DNF). There's a single set; the prediction is not kept separately. On finish/DNF the UI shows the current tags: "Here's how you predicted it; update if your mind changed" | Helps the reader reflect on how a book felt versus what they expected, including why they DNF'd it. A quick check, not a review. One set means no prediction-accuracy score is possible (principle 2). TODO(you): refine |

```go
// Vibe, Pacing, Tags (with Validate and Clone) and Item are defined in 0001.

const (
	DefaultBacklogCapacity = 30
	StalePassThreshold     = 3
)

// Entry is one book as it sits in this reader's backlog.
type Entry struct {
	Book   Book
	Tags   Tags
	Passes int
}

func (b *Backlog) Add(bk Book, t Tags) error // tags optional: Tags{} is valid

// Stale is derived from Passes rather than stored, so it can never disagree with Passes.
func (e Entry) Stale() bool { return e.Passes >= StalePassThreshold }

type Outcome int

const (
	Finished Outcome = iota + 1 // start at 1 so the zero value means "unset"
	DidNotFinish
)

type Record struct {
	Book    Book
	Tags    Tags
	Outcome Outcome
	At      time.Time
}

// Shelf is the aggregate: one reader's Backlog, Reading and history under one lock.
type Shelf struct {
	mu      sync.Mutex
	backlog *Backlog
	reading *Reading // 0001
	history []Record
}

// SetTags finds the book in Backlog, Reading or history and updates its tags there.
// ErrNotFound if the book isn't anywhere on the shelf.
func (s *Shelf) SetTags(id string, t Tags) error
```

## Open questions
- [ ] Cross-list invariant: is a book in at most one of Backlog or Reading at a time?
- [ ] Who owns that invariant? Separate locks can't make "remove from one, add to the other" atomic.
      Candidate: an aggregate (e.g. `Shelf`) holding Backlog, Reading and history under one mutex, exposing `Start(id)`, `Finish(id, at)`, `DNF(id, at)`.
- [ ] `Start` when Reading is full: return `ErrReadingFull`, and the book stays in the Backlog?
- [ ] What happens to a book's pass count when it's started? (It leaves the Backlog; does anything carry over?)
- [ ] Re-adding a book that's in the history: allow, warn, or block? (Especially a DNF. Should the app say "you DNF'd this before"?)
- [ ] History is unbounded. Does that conflict with principle 4 ("stay small")? Does anything ever read it besides the duplicate check above?
- [ ] Guardrail for principle 2: the history must never power stats, counts or streaks. Write that into the spec explicitly?
- [ ] Stale *reading* nudge (reopened by raising Reading to 5): the cap now rarely forces a finish/DNF decision.
      Detect books in Reading untouched for N days and ask "finish or let it go?"? That needs `startedAt`/`lastTouched` and decision #23 (time passed in).
- [ ] The vibe list itself: which vibes, and how many? (TODO(you): draft 8–12)
- [ ] Consistency: switch `Outcome` from `iota` to string-backed like `Vibe`/`Pacing`, since it's also stored and sent as JSON?
- [ ] Is `Genre` really single-valued? Catalog APIs usually return several subjects per book.
- [ ] Test cases to include once this spec is drafted: tags survive `Start`; tags survive `Finish`; tags survive `DNF`;
      `Shelf.SetTags` works in each of the three places; a moved book's tags don't alias the old copy (#37, 0001 #36).
- [ ] Idea, not a decision: **DNF pattern hint**. If the reader DNF'd books tagged e.g. dark + slow-burn, point out Backlog
      books with the same tags ("you've DNF'd books like this before"). Triage-aligned and uses only the reader's own data.
      Open: how many DNFs make a pattern, and how to word it without judging. Possibly its own spec.

## Filtering
Purpose: when a Reading slot opens, narrow the swipe deck by mood ("something fast-paced and funny").

| # | Area | Decision | Why |
|---|------|----------|-----|
| 10 | Filter | AND across the fields that are set. `MaxPages` is a ceiling. `Filter{}` returns everything | Goal is to search across backlog to build TBR. TODO(you): restate with the slot-opens purpose |
| 11 | Filter + unknown pages | Books with `PageCount == 0` are excluded when `MaxPages` is set | Books with a page count of 0 technically do not exist. |
| 12 | Matching | **Superseded by #25:** vibes and pacing come from a fixed list, so they're validated, not fuzzy-matched. Genre matching TBD with 0004 | |
| 29 | Vibe match | Several vibes in a filter match **any** (OR) within the vibe field; still AND across fields | TODO |
| 30 | Filter + untagged | Entries with no pacing are excluded when `Pacing` is set; entries with no vibes are excluded when `Vibes` is set (same rule as #11) | TODO |

```go
// Filter: a zero-valued field is unconstrained.
type Filter struct {
	Genre    string
	Vibes    []Vibe // match any
	Pacing   Pacing
	MaxPages int
}

func (b *Backlog) Filter(f Filter) []Entry // each Entry's Tags cloned
```
