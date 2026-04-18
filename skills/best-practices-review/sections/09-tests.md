---
section: "Tests"
number: 9
category: actionable
rules:
  - leave-testing-to-the-test-function
  - designing-extensible-validation-apis (acceptance-testing, writing-an-acceptance-test)
  - use-real-transports
  - t-error-vs-t-fatal
  - error-handling-in-test-helpers
  - dont-call-t-fatal-from-separate-goroutines
  - use-field-names-in-struct-literals
  - keep-setup-code-scoped (when-to-use-custom-testmain, amortizing-common-test-setup)
sources:
  - https://google.github.io/styleguide/go/best-practices#tests
---

# Section 09 — Tests (Best Practices)

## What this section covers

Eight pragmatic test-design rules — where assertion logic lives, how to design extensible validation APIs (acceptance testing), why to use real transports against test backends, when `t.Error` vs `t.Fatal`, error handling in test helpers, the goroutine-`t.Fatal` trap, table-driven test struct literal style, and how to scope setup code (with `TestMain` / `sync.Once` patterns).

> [!IMPORTANT]
> This section pairs tightly with `decisions-review:07-test-failures.md` (which covers `cmp.Equal`/`cmp.Diff`, got-before-want, full-structure comparisons, etc.). Decisions covers *what failure messages look like*; Best Practices covers *how the tests are structured*.

## #leave-testing-to-the-test-function

> Go distinguishes between "test helpers" and "assertion helpers":
>
> Test helpers are functions that do setup or cleanup tasks. All failures that occur in test helpers are expected to be failures of the environment (not from the code under test) — for example when a test database cannot be started because there are no more free ports on this machine. For functions like these, calling `t.Helper` is often appropriate to mark them as a test helper.
>
> Assertion helpers are functions that check the correctness of a system and fail the test if an expectation is not met. **Assertion helpers are not considered idiomatic in Go.**

> The purpose of a test is to report pass/fail conditions of the code under test. The ideal place to fail a test is within the `Test` function itself.

> If many separate test cases require the same validation logic, arrange the test in one of the following ways instead of using assertion helpers or complex validation functions: Inline the logic (both the validation and the failure) in the `Test` function, even if it is repetitive.

### Recommended pattern

```go
// Good:
func polygonCmp() cmp.Option {
    return cmp.Options{
        cmp.Transformer("polygon", func(p *s2.Polygon) []*s2.Loop { return p.Loops() }),
        cmp.Transformer("loop", func(l *s2.Loop) []s2.Point { return l.Vertices() }),
        cmpopts.EquateApprox(0.00000001, 0),
        cmpopts.EquateEmpty(),
    }
}

func TestFenceposts(t *testing.T) {
    got := Fencepost(tomsDiner, 1*meter)
    if diff := cmp.Diff(want, got, polygonCmp()); diff != "" {
        t.Errorf("Fencepost(tomsDiner, 1m) returned unexpected "+
            "diff (-want+got):\n%v", diff)
    }
}
```

The pattern: a *helper that returns `cmp.Options`* (no `t` involved) is fine. A *helper that calls `t.Errorf` itself* is the assertion-helper anti-pattern.

### Smells

- Helper named `assertX(t, got, want)` / `expectX(...)` / `requireX(...)` that calls `t.Errorf` internally — push the `if/t.Errorf` back into the `Test` function.
- Custom matcher / DSL (`expect(x).ToEqual(y)`) — same anti-pattern; use plain `if` + `cmp.Equal` + `t.Errorf`.
- `t.Helper()` chains 3+ levels deep — strong sign an assertion library is emerging.
- Repeated validation logic across tests refactored into an assertion helper — instead, factor the *comparison config* (e.g., `polygonCmp()`) and inline the call.

(See also `decisions-review:07-test-failures.md#assertion-libraries` — same anti-pattern, slightly different framing.)

---

## #designing-extensible-validation-apis — Designing extensible validation APIs

### Acceptance testing

> Such testing is referred to as **acceptance testing**. The premise of this kind of testing is that the person using the test does not know every last detail of what goes on in the test; they just hand the inputs over to the testing facility to do the work.

The doc references [`testing/fstest.TestFS`](https://pkg.go.dev/testing/fstest) as an example: a generic library validates that a user-supplied `fs.FS` upholds the basic `io/fs` contract. The user-implementer just hands their implementation over; they don't need to know every contract detail.

### Writing an acceptance test

Pattern for building one of these (e.g., for a `chess.Player` interface):

> Create a new package for the validation behavior, customarily named by appending the word `test` to the package name (for example, `chesstest`).
>
> Create the function that performs the validation by accepting the implementation under test as an argument and exercises it.

```go
// ExercisePlayer tests a Player implementation in a single turn on a board.
// The board itself is spot checked for sensibility and correctness.
//
// It returns a nil error if the player makes a correct move in the context
// of the provided board. Otherwise ExercisePlayer returns one of this
// package's errors to indicate how and why the player failed the
// validation.
func ExercisePlayer(b *chess.Board, p chess.Player) error
```

#### Failure reporting modes

**Fail fast**:

```go
for color, army := range b.Armies {
    // The king should never leave the board, because the game ends at checkmate.
    if army.King == nil {
        return &MissingPieceError{Color: color, Piece: chess.King}
    }
}
```

**Aggregate**:

```go
var badMoves []error

move := p.Move()
if putsOwnKingIntoCheck(b, move) {
    badMoves = append(badMoves, PutsSelfIntoCheckError{Move: move})
}

if len(badMoves) > 0 {
    return SimulationError{BadMoves: badMoves}
}
return nil
```

#### How a user calls it

```go
// Good:
package deepblue_test

import (
    "chesstest"
    "deepblue"
)

func TestAcceptance(t *testing.T) {
    player := deepblue.New()
    err := chesstest.ExerciseGame(t, chesstest.SimpleGame, player)
    if err != nil {
        t.Errorf("Deep Blue player failed acceptance test: %v", err)
    }
}
```

> The acceptance test should honor the keep going guidance by not calling `t.Fatal` unless the test detects a broken invariant in the system being exercised.

### Smells

- Validation library that calls `t.Errorf` / `t.Fatal` directly inside its exercise function — instead, return errors and let the caller decide how to surface them.
- Validation library named without the `test` suffix (e.g., `chessvalidator` instead of `chesstest`).
- `Exercise*` function with hidden test setup that the caller can't override.
- No structured error types returned — only string errors — making it impossible for the caller to programmatically distinguish failures.

---

## #use-real-transports — Use real transports

> When testing component integrations, especially where HTTP or RPC are used as the underlying transport between the components, prefer using the real underlying transport to connect to the test version of the backend.

The pattern: real client + test backend, not mock client + real backend.

> For example, suppose the code you want to test (sometimes referred to as "system under test" or SUT) interacts with a backend that implements the long running operations API. To test your SUT, use a real `OperationsClient` that is connected to a test double (e.g., a mock, stub, or fake) of the `OperationsServer`.

> This is recommended over hand-implementing the client, due to the complexity of imitating client behavior correctly. By using the production client with a test-specific server, you ensure your test is using as much of the real code as possible.

> Where possible, use a testing library provided by the authors of the service under test.

### Pattern

| Layer | In production | In test |
|---|---|---|
| Client | Real production client | **Same real production client** |
| Transport | HTTP / gRPC | **Same real HTTP / gRPC** (in-process or loopback) |
| Server | Real production server | Test double (stub / fake / mock server) |

### Smells

- Hand-rolled "fake client" wrapping the production interface — usually a mistake; use the real client against an in-process test server.
- HTTP test using `httptest.NewServer` + production client → ✅ correct pattern.
- gRPC test that bypasses the gRPC transport with a function call → loses transport-layer behaviour (timeouts, headers, deadlines).
- Test using `mock.NewClient` from an external mocking library when the real client could connect to a `httptest`/`bufconn` test server.
- Author's own service under test offers a `*test` library that goes unused — usually the right tool.

### When NOT to apply

- Pure unit tests of a function that takes a typed dependency interface — fine to inject a tiny stub directly.
- Tests of the client itself (where the test backend's real-world quirks would just be noise).

---

## #t-error-vs-t-fatal — `t.Error` vs. `t.Fatal`

> As discussed in decisions, tests should generally not abort at the first encountered problem.

> However, some situations require that the test not proceed. Calling `t.Fatal` is appropriate when some piece of test setup fails, especially in test setup helpers, without which you cannot run the rest of the test.

### Inside table-driven tests

> In a table-driven test, `t.Fatal` is appropriate for failures that set up the whole test function before the test loop. Failures that affect a single entry in the test table, which make it impossible to continue with that entry, should be reported as follows:
>
> - If you're not using `t.Run` subtests, use `t.Error` followed by a `continue` statement to move on to the next table entry.
> - If you're using subtests (and you're inside a call to `t.Run`), use `t.Fatal`, which ends the current subtest and allows your test case to progress to the next subtest.

Quick decision tree:

| Situation | Use |
|---|---|
| Test setup that, if it fails, makes all subsequent test code meaningless (`db, err := openDB()`, `if err != nil { ... }`) | `t.Fatal` |
| Value mismatch where subsequent assertions still meaningfully run | `t.Error` |
| Inside a `t.Run` subtest, single-case failure that prevents this case from completing | `t.Fatal` (only stops this subtest) |
| In a flat for-loop over table cases (no `t.Run`), single-case failure | `t.Error` + `continue` |

> **Warning:** It is not always safe to call `t.Fatal` and similar functions. (See `#dont-call-t-fatal-from-separate-goroutines`.)

### Smells

- `t.Fatal` for value mismatches in flat test bodies — should be `t.Error`.
- `t.Error` followed by code that dereferences a result that may be `nil` — should be `t.Fatal` (otherwise you crash inside the test).
- Flat for-loop over table cases using `t.Fatal` — only the first failure is reported per run; either switch to `t.Error` + `continue` or use `t.Run` subtests.
- `t.Fatal` inside a subtest goroutine spawned by `t.Run` — see `#dont-call-t-fatal-from-separate-goroutines`.

(See also `decisions-review:07-test-failures.md#keep-going` for the broader principle.)

---

## #error-handling-in-test-helpers — Error handling in test helpers

> This section discusses test helpers in the sense Go uses the term: functions that perform test setup and cleanup, not common assertion facilities.

> Operations performed by a test helper sometimes fail. For example, setting up a directory with files involves I/O, which can fail. When test helpers fail, their failure often signifies that the test cannot continue, since a setup precondition failed. When this happens, **prefer calling one of the `Fatal` functions in the helper.**

### Good

```go
func mustAddGameAssets(t *testing.T, dir string) {
    t.Helper()
    if err := os.WriteFile(path.Join(dir, "pak0.pak"), pak0, 0644); err != nil {
        t.Fatalf("Setup failed: could not write pak0 asset: %v", err)
    }
    if err := os.WriteFile(path.Join(dir, "pak1.pak"), pak1, 0644); err != nil {
        t.Fatalf("Setup failed: could not write pak1 asset: %v", err)
    }
}
```

The combination: `t.Helper()` so the failure attribution lands in the caller's line, `t.Fatalf` for the actual failure (because there's no point continuing without the asset), and a descriptive failure message.

> The failure message should include a description of what happened. This is important, as you may be providing a testing API to many users, especially as the number of error-producing steps in the helper increases. When the test fails, the user should know where, and why.

### Smells

- Helper that returns `error` to the caller, requiring every caller to write `if err := setup(t); err != nil { t.Fatal(err) }` — push the `t.Fatal` into the helper.
- Helper missing `t.Helper()` — failure attribution lands inside the helper, not at the caller.
- `t.Fatalf("setup failed")` (no detail) — should describe what step failed and why.
- Helper that calls `t.Fatal` from a goroutine (see `#dont-call-t-fatal-from-separate-goroutines`).

---

## #dont-call-t-fatal-from-separate-goroutines — Don't call `t.Fatal` from separate goroutines

> As [documented in package testing](https://pkg.go.dev/testing#T), it is incorrect to call `t.FailNow`, `t.Fatal`, etc. from any goroutine but the one running the Test function (or the subtest).

> If your test starts new goroutines, they must not call these functions from inside these goroutines.

### Good — `t.Errorf` from inside the goroutine

```go
func TestRevEngine(t *testing.T) {
    engine, err := Start()
    if err != nil {
        t.Fatalf("Engine failed to start: %v", err)
    }

    num := 11
    var wg sync.WaitGroup
    wg.Add(num)
    for i := 0; i < num; i++ {
        go func() {
            defer wg.Done()
            if err := engine.Vroom(); err != nil {
                // This cannot be t.Fatalf.
                t.Errorf("No vroom left on engine: %v", err)
                return
            }
            if rpm := engine.Tachometer(); rpm > 1e6 {
                t.Errorf("Inconceivable engine rate: %d", rpm)
            }
        }()
    }
    wg.Wait()

    if seen := engine.NumVrooms(); seen != num {
        t.Errorf("engine.NumVrooms() = %d, want %d", seen, num)
    }
}
```

The pattern: inside spawned goroutines, use `t.Errorf` (or send errors back via a channel) and `return` from the goroutine on failure. Aggregate / observe outside.

> Adding `t.Parallel` to a test or subtest does not make it unsafe to call `t.Fatal`.

### Smells

- `go func() { ... t.Fatal(err) ... }()` — undefined behaviour; switch to `t.Errorf` + `return`.
- Subtest spawning goroutines that call the parent test's `t.Fatal` — same problem.
- `t.FailNow()` / `t.SkipNow()` from a non-Test goroutine — same trap.
- Using a `chan error` to ferry failures back to the main test goroutine without an aggregation step — partial fix; may swallow late errors.

---

## #use-field-names-in-struct-literals — Use field names in struct literals

> In table-driven tests, prefer to specify field names when initializing test case struct literals. This is helpful when the test cases cover a large amount of vertical space (e.g. more than 20-30 lines), when there are adjacent fields with the same type, and also when you wish to omit fields which have the zero value.

### Good

```go
func TestStrJoin(t *testing.T) {
    tests := []struct {
        slice     []string
        separator string
        skipEmpty bool
        want      string
    }{
        {
            slice:     []string{"a", "b", ""},
            separator: ",",
            want:      "a,b,",
        },
        {
            slice:     []string{"a", "b", ""},
            separator: ",",
            skipEmpty: true,
            want:      "a,b",
        },
        // ...
    }
    // ...
}
```

Two sibling cases differ only in `skipEmpty` — one omits it (zero default `false`), the other specifies `true`. Without field names, the second case's `true` would be a mystery positional value.

### Smells

- Table-driven test with positional struct literal: `{[]string{"a"}, ",", false, "a"}` — switch to named fields, especially when the table is more than ~20 lines.
- Adjacent fields of the same type without names — easy to swap silently.
- All cases explicitly specifying every field including zero defaults — drop the zero-defaults to make the per-case differences pop.
- Named-field cases mixed with positional in the same table — pick one.

---

## #keep-setup-code-scoped — Keep setup code scoped to specific tests

> Where possible, setup of resources and dependencies should be as closely scoped to specific test cases as possible.

### Good — explicit per-test setup

```go
// mustLoadDataset loads a data set for the tests.
func mustLoadDataset(t *testing.T) []byte {
    t.Helper()
    data, err := os.ReadFile("path/to/your/project/testdata/dataset")
    if err != nil {
        t.Fatalf("Could not load dataset: %v", err)
    }
    return data
}

func TestParseData(t *testing.T) {
    data := mustLoadDataset(t)
    // ...
}

func TestListContents(t *testing.T) {
    data := mustLoadDataset(t)
    // ...
}

func TestRegression682831(t *testing.T) {
    // doesn't load the dataset — doesn't need it
    if got, want := guessOS("zpc79.example.com"), "grhat"; got != want {
        t.Errorf(`guessOS("zpc79.example.com") = %q, want %q`, got, want)
    }
}
```

`TestRegression682831` doesn't call `mustLoadDataset` — so when you run `go test -run TestRegression682831`, you don't pay the dataset-loading cost.

### Bad — package-level setup runs even for unrelated tests

```go
// Bad:
var dataset []byte

func init() {
    dataset = mustLoadDataset()
}

func TestRegression682831(t *testing.T) {
    // This still pays the init cost even when running in isolation.
}
```

> A user may wish to run a function in isolation of the others and should not be penalized by these factors.

### #when-to-use-custom-testmain — When to use a custom `TestMain` entrypoint

> If **all tests in the package** require common setup and the **setup requires teardown**, you can use a custom testmain entrypoint. This can happen if the resource the test cases require is especially expensive to setup, and the cost should be amortized.

> Using a custom `TestMain` **should not be your first choice** due the amount of care that should be taken for correct use. Consider first whether the solution in the *amortizing common test setup* section or an ordinary test helper is sufficient for your needs.

```go
// Good:
var db *sql.DB

func TestInsert(t *testing.T) { /* omitted */ }
func TestSelect(t *testing.T) { /* omitted */ }
func TestUpdate(t *testing.T) { /* omitted */ }
func TestDelete(t *testing.T) { /* omitted */ }

// runMain sets up the test dependencies and eventually executes the tests.
// It is defined as a separate function to enable the setup stages to clearly
// defer their teardown steps.
func runMain(ctx context.Context, m *testing.M) (code int, err error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    d, err := setupDatabase(ctx)
    if err != nil {
        return 0, err
    }
    defer d.Close()
    db = d

    return m.Run(), nil
}

func TestMain(m *testing.M) {
    code, err := runMain(context.Background(), m)
    if err != nil {
        log.Fatal(err)
    }
    // NOTE: defer statements do not run past here due to os.Exit
    //       terminating the process.
    os.Exit(code)
}
```

Pattern keys: extract `runMain` so teardown can `defer`; `TestMain` calls `os.Exit` after `runMain` returns.

> Ideally a test case is hermetic between invocations of itself and between other test cases.

> At the very least, ensure that individual test cases reset any global state they have modified if they have done so (for instance, if the tests are working with an external database).

### #amortizing-common-test-setup — Amortizing common test setup

> Using a `sync.Once` may be appropriate, though not required, if all of the following are true about the common setup:
>
> - It is expensive.
> - It only applies to some tests.
> - It does not require teardown.

```go
// Good:
var dataset struct {
    once sync.Once
    data []byte
    err  error
}

func mustLoadDataset(t *testing.T) []byte {
    t.Helper()
    dataset.once.Do(func() {
        data, err := os.ReadFile("path/to/your/project/testdata/dataset")
        dataset.data = data
        dataset.err = err
    })
    if err := dataset.err; err != nil {
        t.Fatalf("Could not load dataset: %v", err)
    }
    return dataset.data
}
```

> The reason that common teardown is tricky is there is no uniform place to register cleanup routines. If the setup function (in this case `mustLoadDataset`) relies on a context, `sync.Once` may be problematic. This is because the second of two racing calls to the setup function would need to wait for the first call to finish before returning. This period of waiting cannot be easily made to respect the context's cancellation.

### Smells (across `keep-setup-code-scoped` + sub-rules)

- `init()` doing test setup that not all tests need — push into a per-test helper.
- Package-level `var x = expensiveSetup()` that runs at import time even when only one unrelated test is selected.
- `TestMain` used when a single per-test helper would suffice.
- `TestMain` that calls `m.Run()` directly with no `runMain` wrapper — `defer` for teardown won't run.
- `TestMain` ending with `m.Run()` (returning the code) but not calling `os.Exit` — Go ≥ 1.15 actually does call `os.Exit` for you if you don't, but the explicit form is the documented pattern.
- `sync.Once` setup that needs a context — Best Practices flags this as problematic; use a different mechanism.
- Tests that mutate global state and don't restore it — leak between test cases.

---

## Consolidated review checklist (8 sub-rules)

### Test logic placement

- [ ] No assertion helpers (`assertX(t, ...)`, `expectX(...)`) — push `if` + `t.Errorf` into the `Test` function.
- [ ] Repeated comparison config factored as a `cmp.Options`-returning helper, not as an assertion helper.
- [ ] Custom matcher / DSL APIs absent (`expect(x).ToBe(y)`-style).

### Acceptance / extensibility

- [ ] Validation libraries return errors; they do **not** call `t.Errorf`/`t.Fatal` themselves.
- [ ] Validation library packages named `<pkg>test`.
- [ ] Acceptance-test failures use structured error types so callers can interpret them.

### Transports

- [ ] Tests of integrations use the *real* client connected to a test backend, not a hand-written mock client.
- [ ] HTTP integration tests use `httptest.NewServer`; gRPC uses `bufconn` or in-process server.

### Failure mode

- [ ] `t.Fatal` only when continuing is meaningless (setup failure, nil-deref risk).
- [ ] Flat for-loop over table cases uses `t.Error + continue`, not `t.Fatal`.
- [ ] `t.Run` subtest cases use `t.Fatal` (ends the subtest only).
- [ ] No `t.Fatal` (or `t.FailNow`/`t.SkipNow`) called from a goroutine other than the Test function's own.

### Helpers

- [ ] Test setup helpers call `t.Helper()` and `t.Fatalf` internally rather than returning `error`.
- [ ] Helper failure messages name the step that failed and why.

### Test data

- [ ] Table-driven test struct literals use field names (especially when >20 lines or with adjacent same-type fields).
- [ ] Zero-default fields omitted from per-case literals.

### Setup scope

- [ ] No `init()` doing setup that some tests don't need.
- [ ] No package-level `var = expensiveSetup()` for the same reason.
- [ ] `TestMain` reserved for setups that need teardown AND apply to all tests.
- [ ] When `TestMain` is used: `runMain` wrapper for `defer`; `os.Exit` outside.
- [ ] `sync.Once` used only when setup is expensive, applies to some tests, and doesn't need teardown.

## Suggested finding phrasing (advisory)

- "Test calls helper `assertEqualUsers(t, got, want)` that internally calls `t.Errorf` — Best Practices §leave-testing-to-the-test-function: extract the comparison into a `cmp.Options`-returning helper, keep the `if` + `t.Errorf` in the test."
- "`chesstest.ExercisePlayer(t, p)` calls `t.Fatal` internally — Best Practices §designing-extensible-validation-apis: return structured errors instead; let the caller's test function decide how to surface."
- "Test injects hand-written `mockClient` for HTTP backend — Best Practices §use-real-transports: use the real client + `httptest.NewServer` instead."
- "`t.Fatalf(\"got %v, want %v\", got, want)` for value mismatch in flat test body — Best Practices §t-error-vs-t-fatal: use `t.Errorf`; subsequent assertions should still run."
- "`go func() { if err != nil { t.Fatal(err) } }()` — Best Practices §dont-call-t-fatal-from-separate-goroutines: undefined behavior; use `t.Errorf` + `return` from the goroutine."
- "Helper `setupDB(t *testing.T) (*sql.DB, error)` returns error — Best Practices §error-handling-in-test-helpers: push the `t.Fatalf` into the helper; rename to `mustSetupDB(t)`."
- "Table-driven test uses positional struct literals: `{[]string{\"a\"}, \",\", false, \"a\"}` — Best Practices §use-field-names-in-struct-literals: use named fields; positional swaps are silent bugs."
- "`var dataset = mustLoadDataset()` at package level — Best Practices §keep-setup-code-scoped: pushes the cost onto every test in the package, even unrelated ones; load per-test inside the relevant `Test*` functions."
- "`TestMain` used to do setup that only `TestSelect` needs — Best Practices: scope to the specific test; `TestMain` is for shared setup that ALL tests need AND requires teardown."

## Sources

- <https://google.github.io/styleguide/go/best-practices#tests> — Google Go Best Practices, Tests section (the canonical text quoted above)
- <https://google.github.io/styleguide/go/decisions#useful-test-failures> — Decisions, Useful test failures (foundational rules — sibling skill `decisions-review:07-test-failures.md`)
- <https://pkg.go.dev/testing> — Go `testing` package (`t.Helper`, `t.Fatal`, `TestMain`, etc.)
- <https://pkg.go.dev/testing/fstest> — `testing/fstest` (referenced for `TestFS` acceptance pattern)
- <https://pkg.go.dev/net/http/httptest> — `httptest` (referenced for real-transport tests)
- <https://pkg.go.dev/google.golang.org/grpc/test/bufconn> — `bufconn` (in-process gRPC for `#use-real-transports`)
