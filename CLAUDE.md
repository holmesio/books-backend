# books-backend

Go backend for a Books API (future mobile client). Built spec-first: specs in `docs/specs/` are the contract.

## Working agreement
- This is a learning project. Default to **navigator** mode: explain idioms, ask questions, and suggest.
  Write implementation code only when explicitly asked.
- Before touching a package, read its spec. Decisions marked in a spec are settled. If one seems wrong, raise it;
  don't silently change it.
- Tests come before implementation. Table-driven, black-box (`package <name>_test`), one table per behavior.

## Layout
- Start at `docs/specs/README.md` for the roadmap and architecture rules.
- `internal/<domain>/`: domain packages, named for what they provide (`tbr`, not `models`/`utils`).
  Domain packages import only the stdlib. Storage and HTTP depend on the domain, never the reverse.
- `cmd/` only once there's a binary to build.
- Specs: `docs/specs/NNNN-<slug>.md`, numbered in order of creation.

## Go conventions
- Errors: exported sentinel vars (`var ErrX = errors.New("pkg: ...")`), checked with `errors.Is`.
  Add context by wrapping with `%w`.
- Never return internal slices or maps from a type that holds a mutex. Return copies.
- No new dependencies without discussing them first. Prefer the standard library.

## Done means
```
gofmt -l .            # prints nothing
go vet ./...
go test -race -count=1 ./...
```
