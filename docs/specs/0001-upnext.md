# 0001: UpNext queue

Status: draft
Package: `internal/tbr`
Related: [0002: Backlog](0002-backlog.md)

## Goal
Package goal (becomes `doc.go`):
Package tbr models a reader's to-be-read queue, enforcing its invariants so callers can rely on it to pick what to read next.

This spec covers `UpNext`: the short, ordered list of books the reader has committed to reading next.

## Scope
- In: `UpNext` and its rules (capacity, duplicates, ordering, removal).
- Out: HTTP, storage, external book APIs, users/auth. `tbr.UpNext` is one user's queue and knows nothing about users.
- Out (→ 0002): the Backlog, filtering, swipe/promote, and what happens to a book after it leaves UpNext.
  `UpNext.Remove` only removes. Deciding where the book goes is the coordinator's job.

## Decisions
<!-- The decisions are settled. Fill in each Why in your own words. If you can't, reopen the decision. -->
<!-- Decision numbers are stable. Never renumber; mark moved or superseded rows instead. -->

| # | Area | Decision | Why |
|---|------|----------|-----|
| 1 | Scope | One `UpNext` = one user's queue. The user→queue mapping lives in a later layer | This package is only concerned with the queue mechanics and rules. User-specific logic is not necessary. |
| 2 | Book | Required: `ID`, `Title`, `Author`. Optional: `Genre`, `Vibe`, `PageCount` | Each Book needs an identifier, along with a title and author. Other fields are useful, though not required. |
| 3 | Page count | `0` means unknown. Negative values are invalid | Books with zero pages technically do not exist. |
| 4 | Capacity | `DefaultCapacity = 5`. `NewUpNext(capacity)` requires capacity > 0 | Capacity must be positive to account for user's queue. |
| 5 | Overflow | `Add` on a full queue returns `ErrUpNextFull`. The queue is unchanged | Never silently lose a book the user chose. |
| 6 | Duplicates | Identity is `Book.ID`. A repeat ID returns `ErrDuplicate` | Prevents duplicate entries on return. Book.ID is opaque; spec 0004 decides where it comes from. |
| 7 | Add check order | validate → duplicate → full | Package should confirm the book exists, check the list for duplicates, then ensure the list can add a new book. |
| 8 | Ordering | `List` returns books in insertion order | Represents the "Up Next" nature desired for package. |
| 9 | Removal | `Remove` of an ID that isn't in the queue returns `ErrNotFound` | Book cannot be removed that does not exist in queue. |
| 10–12 | Filtering | Moved to [0002](0002-backlog.md) | |
| 13 | Concurrency | `UpNext` guards itself with a `sync.Mutex`. `List` returns a copy | Concurrent requests can't break invariants; copies keep callers from mutating or racing on internal state. |
| 14 | Invalid input | `NewUpNext(capacity <= 0)`: return an error | Capacities should be positive. |

## Public API
```go
type Book struct {
	ID, Title, Author string // required
	Genre, Vibe       string // optional
	PageCount         int    // optional; 0 = unknown
}

const DefaultCapacity = 5

var (
	ErrUpNextFull  = errors.New("tbr: up next is full")
	ErrDuplicate   = errors.New("tbr: duplicate book")
	ErrNotFound    = errors.New("tbr: book not found")
	ErrInvalidBook = errors.New("tbr: invalid book")
)

func NewUpNext(capacity int) (*UpNext, error)
func (u *UpNext) Add(b Book) error
func (u *UpNext) Remove(id string) error
func (u *UpNext) List() []Book
```

## Behaviors → tests
<!-- TODO(you): list case names for each table before writing any Go. -->

### TestNewUpNext
- 

### TestUpNext_Add
- 

### TestUpNext_Remove
- 

### TestUpNext_List
- 

### TestUpNext_Concurrent
- 

## Acceptance
- [ ] Every behavior above has a table-driven test in `package tbr_test`
- [ ] `gofmt -l .` prints nothing
- [ ] `go vet ./...` is clean
- [ ] `go test -race -count=1 ./...` passes
