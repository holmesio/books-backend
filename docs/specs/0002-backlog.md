# 0002: Backlog and promotion

Status: stub. Don't start until 0001 is green.
Package: `internal/tbr`
Depends on: [0001: UpNext](0001-upnext.md)

## Goal
<!-- TODO(you): one sentence for this spec. -->

## Product context
- The Backlog is the TBR in its pure form: every book the reader might elect to read, now or later.
- UpNext exists to work the Backlog down.
- Mobile swipe UX:
  - Right swipe promotes a book from the Backlog to UpNext.
  - Left swipe passes: the book stays in the Backlog and its pass count goes up.
  - The X button deletes the book from the Backlog.
  - After repeated passes, the app asks: "You passed this book N times. Would you like to remove it from the TBR?"

## Decisions (proposed; Why lines are TODO(you))
| # | Area | Decision | Why |
|---|------|----------|-----|
| 15 | Backlog cap | Hard cap of 15 books (`DefaultBacklogCapacity = 15`). Adding a 16th returns an error | TODO |
| 16 | Left swipe | A pass *is* domain state: it increments the book's pass count for this reader | TODO |
| 17 | Stale books | A book is stale when passes ≥ `StalePassThreshold` (3). The domain reports staleness; the prompt text is UI | TODO |
| 18 | Keep after prompt | Choosing **Keep** resets the book's pass count to 0 | TODO |
| 19 | Pass count home | The Backlog holds `Entry{Book, Passes}`. `Book` stays pure catalog data. The reader stays implicit (#1) | TODO |
| 20 | Threshold owner | The domain owns `StalePassThreshold` and reports staleness. Clients only choose the wording | TODO |

```go
const (
	DefaultBacklogCapacity = 15
	StalePassThreshold     = 3
)

// Entry is one book as it sits in this reader's backlog.
type Entry struct {
	Book   Book
	Passes int
}

// Stale is derived from Passes rather than stored, so it can never disagree with Passes.
func (e Entry) Stale() bool { return e.Passes >= StalePassThreshold }
```

## Open questions
- [ ] When a book leaves UpNext, where does it go? (Finished → out of the TBR, maybe a future "Read" shelf? Demoted → back to the Backlog?)
- [ ] Cross-list invariant: is a book always in exactly one of Backlog or UpNext? Can it be in both?
- [ ] Who owns that invariant? Separate locks can't make "remove from one, add to the other" atomic.
      Candidate: an aggregate (e.g. `Shelf`) that holds both lists under one mutex and exposes `Promote(id)`.
- [ ] Promote when UpNext is full: return `ErrUpNextFull`, and the book stays in the Backlog?
- [ ] Is a Backlog ID that's already in UpNext a duplicate?
- [ ] Does promoting a book and then demoting it back keep its pass count?
- [ ] Is 5 the right UpNext default for book paralysis, or is 3 better?

## Filtering (moved from 0001; Why lines carried over)
<!-- With a cap of 15, is filtering still worth it? Revisit before accepting. -->
| # | Area | Decision | Why |
|---|------|----------|-----|
| 10 | Filter | AND across the fields that are set. `MaxPages` is a ceiling. `Filter{}` returns everything | Goal is to search across backlog to build TBR. |
| 11 | Filter + unknown pages | Books with `PageCount == 0` are excluded when `MaxPages` is set | Books with a page count of 0 technically do not exist. |
| 12 | Matching | Genre and vibe compare case-insensitively | Helps ensure genres are returned regardless of casing. |

```go
// Filter: a zero-valued field is unconstrained.
type Filter struct {
	Genre, Vibe string
	MaxPages    int
}

func (b *Backlog) Filter(f Filter) []Book // returns a copy
```
