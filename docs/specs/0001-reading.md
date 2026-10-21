# 0001: Reading

Status: draft
Package: `internal/tbr`
Related: [0002: Backlog and the shelf](0002-backlog.md)

## Goal
Package goal (becomes `doc.go`):
Package tbr models a reader's reading journey from TBR to Done, enforcing its invariants so callers can rely on it to pick what to read next.

This spec covers `Reading`, the small, capped set of books the reader is reading right now, and the reader's `Tags`
(vibes and pacing), which travel with each book through its lifecycle.

## Scope
- In: `Reading` and its rules (capacity, duplicates, ordering, removal, tag edits). `Tags` and their validation.
- Out: HTTP, storage, external book APIs, users/auth. `tbr.Reading` is one user's set and knows nothing about users.
- Out (→ 0002): the Backlog, swiping and starting a book, finish/DNF outcomes and their history.
  `Reading.Remove` only removes. Whether that was a finish or a DNF is the coordinator's job.

## Decisions
<!-- The decisions are settled. Fill in each Why in your own words. If you can't, reopen the decision. -->
<!-- Decision numbers are stable. Never renumber; mark moved or superseded rows instead. -->

| # | Area | Decision | Why |
|---|------|----------|-----|
| 1 | Scope | One `Reading` = one user's set. The user→set mapping lives in a later layer | This package is only concerned with the list mechanics and rules. User-specific logic is not necessary. |
| 2 | Book | Required: `ID`, `Title`, `Author`. Optional: `Genre`, `PageCount`. Reader tags are not on `Book` (see #34) | Each Book needs an identifier, along with a title and author. Other fields are useful, though not required. |
| 3 | Page count | `0` means unknown. Negative values are invalid | Books with zero pages technically do not exist. |
| 4 | Capacity | `DefaultCapacity = 5`. `NewReading(capacity)` requires capacity > 0 | Capacity must be positive. 5 gives readers flexibility while still being a hard cap (raised from 3 after 15/3 felt too tight in practice). |
| 5 | Overflow | `Add` on a full set returns `ErrReadingFull`. The set is unchanged | Never silently lose a book the user chose. |
| 6 | Duplicates | Identity is `Book.ID`. A repeat ID returns `ErrDuplicate` | Prevents duplicate entries on return. Book.ID is opaque; spec 0004 decides where it comes from. |
| 7 | Add check order | validate (book, then tags) → duplicate → full | Package should confirm the book is well-formed, check the list for duplicates, then ensure the list can add a new book. |
| 8 | Ordering | `List` returns items in insertion order (the order they were started) | Order is deterministic for tests and UI. |
| 9 | Removal | `Remove` of an ID that isn't in the set returns `ErrNotFound` | Book cannot be removed that does not exist in Reading set. |
| 10–12 | Filtering | Moved to [0002](0002-backlog.md) | |
| 13 | Concurrency | `Reading` guards itself with a `sync.Mutex`. `List` returns a copy | Concurrent requests can't break invariants; copies keep callers from mutating or racing on internal state. |
| 14 | Invalid input | `NewReading(capacity <= 0)`: return an error | Capacities should be positive. |
| 21 | UpNext dropped | `Reading` replaces `UpNext`. The lifecycle is Backlog → Reading → finished/DNF | One list too many for an app meant to reduce overwhelm. |
| 25 | Tag values | (moved from 0002) A fixed list of string-backed constants, also enumerated by `Vibes()` / `Pacings()` (copies of unexported lists). Unknown values return `ErrInvalidVibe` / `ErrInvalidPacing`. Adding a vibe is a code change in two places, guarded by a test | Filtering on tags only works if everyone uses the same words. |
| 26 | Vibe limits | (moved from 0002) At most `MaxVibes = 3` per book. Too many returns `ErrTooManyVibes`; a repeated vibe returns `ErrDuplicateVibe` (not `ErrDuplicate`, which means a duplicate book) | The limitation encourages the user to think strategically about how they see a book. |
| 34 | Tags travel | A book's `Tags` are held by whatever holds the book. In Reading that's `Item{Book, Tags}`. There's no separate tag store, so removing a book removes its tags | Tags are how the reader sees *this book*, so they go wherever the book goes. Cleanup is automatic. |
| 35 | Tag edits | `SetTags(id, t)` replaces a book's tags while it's in Reading. Same validation as `Add`. `ErrNotFound` if absent | Readers can update their vibe check at any time (0002 #33) |
| 36 | No aliasing | `Tags` contains a slice, so it is cloned at every boundary: stored as `t.Clone()` on the way in (`Add`, `SetTags`), returned as clones on the way out (`List`) | Otherwise a caller holding the slice could change stored tags after validation. `-race` can't catch this; only tests can |
| 38 | Tag validate order | validate (tag) → duplicate → count | Package should confirm that a tag is well-formed and not an unknown value, check the tags for duplicate vibes, then check the vibe count. |

## Public API
```go
type Book struct {
	ID, Title, Author string // required
	Genre             string // optional; catalog data
	PageCount         int    // optional; 0 = unknown
}

type Vibe string

// Grouped by the question a reader asks: "what do I feel like reading right now?"
const (
	// Light
	Cozy      Vibe = "cozy"
	Funny     Vibe = "funny"
	Hopeful   Vibe = "hopeful"
	Romantic  Vibe = "romantic"
	Whimsical Vibe = "whimsical"
	Nostalgic Vibe = "nostalgic"

	// Energy
	Adventurous Vibe = "adventurous"
	Tense       Vibe = "tense"

	// Atmosphere
	Mysterious Vibe = "mysterious"
	Eerie      Vibe = "eerie"

	// Mind
	ThoughtProvoking Vibe = "thought-provoking"

	// Heavy
	Heartbreaking Vibe = "heartbreaking"
	Dark          Vibe = "dark"
	Gritty        Vibe = "gritty"
)

type Pacing string // "" = not tagged yet

const (
	SlowBurn  Pacing = "slow-burn"
	Medium    Pacing = "medium"
	FastPaced Pacing = "fast-paced"
)

// Go can't iterate constants, so the allowed values are also listed. Unexported, because Go has no constant
// slices and an exported var could be modified by any importer. A new constant must be added here too.
var (
	allVibes = []Vibe{
		Cozy, Funny, Hopeful, Romantic, Whimsical, Nostalgic,
		Adventurous, Tense,
		Mysterious, Eerie,
		ThoughtProvoking,
		Heartbreaking, Dark, Gritty,
	}
	allPacings = []Pacing{SlowBurn, Medium, FastPaced}
)

func Vibes() []Vibe     // returns a copy; used by Validate and by clients listing the options
func Pacings() []Pacing // returns a copy

// Tags are this reader's vibes and pacing for one book. The zero value (no tags) is valid.
// If Tags gains another reference type (map, pointer, nested slice), Clone must be updated too.
type Tags struct {
	Vibes  []Vibe
	Pacing Pacing
}

func (t Tags) Validate() error // #25, #26
func (t Tags) Clone() Tags     // new backing array for Vibes

// Item is one book as it sits in Reading.
type Item struct {
	Book Book
	Tags Tags
}

const (
	DefaultCapacity = 5
	MaxVibes        = 3
)

var (
	ErrReadingFull   = errors.New("tbr: reading is full")
	ErrDuplicate     = errors.New("tbr: duplicate book")
	ErrNotFound      = errors.New("tbr: book not found")
	ErrInvalidBook   = errors.New("tbr: invalid book")
	ErrInvalidVibe   = errors.New("tbr: invalid vibe")
	ErrDuplicateVibe = errors.New("tbr: duplicate vibe")
	ErrInvalidPacing = errors.New("tbr: invalid pacing")
	ErrTooManyVibes  = errors.New("tbr: too many vibes")
)

func NewReading(capacity int) (*Reading, error)
func (r *Reading) Add(b Book, t Tags) error
func (r *Reading) SetTags(id string, t Tags) error
func (r *Reading) Remove(id string) error
func (r *Reading) List() []Item // each Item's Tags cloned
```

## Behaviors → tests
<!-- Name each table's cases just before writing that table (see CLAUDE.md: small red→green loops). -->

### TestVibesAndPacings
- 
<!-- Include: every constant appears in Vibes()/Pacings(); mutating a returned slice doesn't affect the next call. -->

### TestTags_Validate
- every vibe in Vibes() → ok
- every pacing in Pacings() → ok
- empty tags → ok (zero value is valid)
- three distinct vibes → ok
- single vibe → ok
- four vibes → fail (ErrTooManyVibes)
- duplicate vibes → fail (ErrDuplicateVibe)
- misspelled vibe → fail (ErrInvalidVibe)
- vibe with wrong case → fail (ErrInvalidVibe)
- invalid pacing → fail (ErrInvalidPacing)
- four vibes, one misspelled → fail (ErrInvalidVibe)
- four vibes, one repeated → fail (ErrDuplicateVibe)

### TestTags_Clone
- empty tags → clone equals Tags{}
- nonempty tags → clone equals original
- modify original's vibes after cloning → clone unchanged

### TestNewReading
- 

### TestReading_Add
- 
<!-- Include: mutating the caller's Vibes slice after Add doesn't change the stored tags (#36). -->

### TestReading_SetTags
- 

### TestReading_Remove
- 

### TestReading_List
- 
<!-- Include: mutating a returned Item's Vibes doesn't change what the next List returns (#36). -->

### TestReading_Concurrent
- 

## Acceptance
- [ ] Every behavior above has a table-driven test in `package tbr_test`
- [ ] `gofmt -l .` prints nothing
- [ ] `go vet ./...` is clean
- [ ] `go test -race -count=1 ./...` passes
