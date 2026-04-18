---
section: "Useful test failures"
number: 7
category: actionable
rules:
  - assertion-libraries
  - identify-the-function
  - identify-the-input
  - got-before-want
  - full-structure-comparisons
  - compare-stable-results
  - keep-going
  - types-of-equality
  - level-of-detail
sources:
  - https://google.github.io/styleguide/go/decisions#useful-test-failures
---

# Section 07 — Useful test failures

## What this section covers

Nine concrete decisions on how Go test failures should be written so that **a single run produces enough information to diagnose the bug** — rather than failing fast, hiding context, or depending on assertion-library magic.

> [!IMPORTANT]
> The single most-violated rule in this section is `#assertion-libraries`. Codebases that import `testify` or similar are doing exactly what this section forbids — flag it, but understand the project may have a long-standing `#10 Local consistency` deviation (Style Guide). The Style Guide explicitly names assertion libraries as an *invalid* local-style scope.

## #assertion-libraries — Assertion libraries

> Do not create "assertion libraries" as helpers for testing.

Assertion libraries combine validation and failure-message production within tests. The Style Decisions warn against them because they either stop tests prematurely or omit relevant information about what succeeded.

### Bad

```go
assert.IsNotNil(t, "obj", obj)
assert.StringEq(t, "obj.Type", obj.Type, "blogPost")
```

### Good

```go
if !cmp.Equal(got, want) {
    t.Errorf("Blog post = %v, want = %v", got, want)
}
```

The pattern: write the conditional + `t.Errorf` (or `t.Fatalf`) explicitly. Use `cmp.Equal`/`cmp.Diff` for the comparison. The failure message you write is the diagnostic the developer reads at 2am — making it custom to the function and inputs is the whole point.

### Smells

- Any import of `github.com/stretchr/testify`, `github.com/onsi/gomega`, `github.com/franela/goblin`, or similar.
- Project-local `assert` / `require` / `expect` helper packages.
- `t.Helper()` chains 3+ levels deep — the repeated indirection signals an emerging in-house assertion library.
- Test functions that contain only `assert.X(...)` lines and no `t.Errorf`/`t.Fatalf` — switch to explicit `if + t.Errorf`.
- Custom matchers (`expect(x).ToBeNil()`-style fluent APIs).

---

## #identify-the-function — Identify the function

> In most tests, failure messages should include the name of the function that failed, even though it seems obvious from the name of the test function.

Recommended format:

```
YourFunc(%v) = %v, want %v
```

— rather than just `got %v, want %v`.

The reason: when a test fails in CI, the message is often surfaced out of context (Slack notification, email, log search). Knowing *which function* the assertion was about, without going to the source, saves time.

### Smells

- `t.Errorf("got %v, want %v", got, want)` with no function name.
- `t.Errorf("expected true")` / `t.Errorf("not equal")` — opaque, no function, no inputs.
- `t.Errorf("FAIL")` or `t.Errorf("test failed")` — content-free.
- A failure message that names a *helper* function instead of the function under test.

---

## #identify-the-input — Identify the input

> In most tests, failure messages should include the function inputs if they are short.

For lengthy or opaque inputs, name the test cases descriptively (e.g., in a table-driven test) and print that **case name** within error messages.

```
YourFunc(%v) = %v, want %v   ← short input directly in message
case "RFC3339Nano with timezone" failed: YourFunc = %v, want %v   ← long input → case name
```

### Smells

- `t.Errorf("Parse(...) = ...")` with `...` literally in the format — not interpolating actual inputs.
- Table-driven test cases with no `name` field, so failure messages can't identify which row failed.
- Multi-thousand-character JSON literal pasted into the failure message — too long; print a label or hash instead.
- Subtests using `t.Run("", func(...))` (empty name) — the case is unidentifiable.

---

## #got-before-want — Got before want

> Test outputs should include the actual value that the function returned before printing the value that was expected. A standard format for printing test outputs is `YourFunc(%v) = %v, want %v`. Where you would write "actual" and "expected", prefer using the words "got" and "want", respectively.

> For diffs, directionality is less apparent, and as such it is important to include a key to aid in interpreting the failure.

The standard message order: **`got` then `want`**. And for diffs (which can go either direction):

```go
if diff := cmp.Diff(want, got); diff != "" {
    t.Errorf("MyFunc(%v) mismatch (-want +got):\n%s", input, diff)
}
```

The `(-want +got)` legend is the "key to aid in interpreting the failure" the doc names.

### Smells

- `t.Errorf("want %v, got %v", want, got)` — wrong order.
- "actual" / "expected" terminology — use "got" / "want".
- `cmp.Diff` calls without the `(-want +got)` (or `(-got +want)`) legend in the message — readers can't tell which side is which.
- Inconsistent ordering across test files in the same package — pick one.

---

## #full-structure-comparisons — Full structure comparisons

> If your function returns a struct (or any data type with multiple fields such as slices, arrays, and maps), avoid writing test code that performs a hand-coded field-by-field comparison of the struct. Instead, construct the data that you're expecting your function to return, and compare directly using a deep comparison.

### Bad — hand-coded field-by-field

```go
val, multi, tail, err := strconv.UnquoteChar(`\"Fran & Freddie's Diner\"`, '"')
if err != nil {
    t.Fatalf(...)
}
if val != `"` {
    t.Errorf(...)
}
if multi {
    t.Errorf(...)
}
if tail != `Fran & Freddie's Diner"` {
    t.Errorf(...)
}
```

### Good — deep comparison

```go
got, err := MyFunc(input)
if err != nil {
    t.Fatalf("MyFunc(%v) returned error: %v", input, err)
}
want := Result{Val: `"`, Multi: false, Tail: `Fran & Freddie's Diner"`}
if diff := cmp.Diff(want, got); diff != "" {
    t.Errorf("MyFunc(%v) mismatch (-want +got):\n%s", input, diff)
}
```

The deep-comparison version: (a) is shorter, (b) reports *all* differences in one shot (instead of bailing on the first), (c) survives field additions to the struct without test changes.

### Smells

- A test that asserts `if x.A != ... ; if x.B != ... ; if x.C != ...` for a fixed-shape return value.
- Hand-rolled `reflect.DeepEqual` instead of `cmp.Equal` (`reflect.DeepEqual` doesn't handle protos / time / unexported fields well).
- A test that ignores some struct fields by checking only some of them — this *might* be intentional; if so, use `cmp.Diff` with `cmpopts.IgnoreFields` so it's explicit.
- Tests that pass when fields are added to the struct (because the test never noticed them) — full-structure comparison flags this immediately.

---

## #compare-stable-results — Compare stable results

> Avoid comparing results that may depend on output stability of a package that you do not own. Instead, the test should compare on semantically relevant information that is stable and resistant to changes in dependencies.

> For functionality that returns a formatted string or serialized bytes, it is generally not safe to assume that the output is stable.

Examples of *unstable* outputs:

- `proto.MarshalTextString` — protobuf text format (whitespace, field order, version-specific).
- `json.Marshal` — field order is technically stable since Go 1.12, but encoding details can change (HTML escaping, etc.).
- `fmt.Sprintf("%v", structVal)` — Go's default formatter; can change between versions.
- `time.Time.String()` — locale-sensitive in some Go versions.
- Map iteration order in any printed form.
- Stack trace strings.

Test against the **structured value**, not its rendered form.

### Smells

- Test compares two `proto.MarshalTextString(...)` outputs as strings.
- Test compares two `json.Marshal(...)` byte slices instead of comparing the unmarshalled structs.
- Test asserts `fmt.Sprintf("%v", got) == "Result{A:1 B:2}"` — fragile to formatting changes.
- Test relies on map keys being printed in any particular order.
- Golden file tests that re-serialize before comparison instead of comparing structured values directly.

---

## #keep-going — Keep going

> Tests should keep going for as long as possible, even after a failure, in order to print out all of the failed checks in a single run.

> Prefer calling `t.Error` over `t.Fatal` for reporting a mismatch.

```go
// Good:
gotMean, gotVariance, err := MyDistribution(input)
if err != nil {
    t.Fatalf("MyDistribution(%v) returned unexpected error: %v", input, err)
}
if diff := cmp.Diff(wantMean, gotMean); diff != "" {
    t.Errorf("MyDistribution(%v) returned unexpected difference in mean value (-want +got):\n%s", input, diff)
}
```

> Calling `t.Fatal` is primarily useful for reporting an unexpected condition (such as an error or output mismatch) when subsequent failures would be meaningless.

The rule: `t.Fatal` only when continuing would be **meaningless** (e.g., the function under test returned `nil, err` so dereferencing the result would panic). Otherwise `t.Error`, so all assertions report.

### Smells

- `t.Fatalf` used for *value* mismatches when subsequent assertions could still meaningfully run.
- A test with 6 assertions, the first 5 are `t.Fatalf` — only the first failure is ever reported per run, so cycle time multiplies.
- `t.Fatalf` followed by code that would only be reached on success — fine, that's the legitimate use.
- A subtest pattern (`t.Run`) where `t.Fatal` aborts the parent — should be `t.Errorf` (or use `t.Run` for isolation).

---

## #types-of-equality — Equality comparison and diffs

> Use [`cmp.Equal`](https://pkg.go.dev/github.com/google/go-cmp/cmp#Equal) for equality comparison and [`cmp.Diff`](https://pkg.go.dev/github.com/google/go-cmp/cmp#Diff) to obtain a human-readable diff between objects.

> `cmp` may not know how to compare certain types without options like `protocmp.Transform`.

> Prefer using `cmp` for new code.

The rule, simply: **use `cmp` everywhere**, in the canonical patterns:

```go
// Equality check
if !cmp.Equal(got, want) {
    t.Errorf("MyFunc(%v) = %v, want %v", input, got, want)
}

// Diff for richer messages
if diff := cmp.Diff(want, got); diff != "" {
    t.Errorf("MyFunc(%v) mismatch (-want +got):\n%s", input, diff)
}

// With options for protos
if diff := cmp.Diff(want, got, protocmp.Transform()); diff != "" {
    t.Errorf(...)
}
```

### Smells

- `reflect.DeepEqual` in test code — switch to `cmp.Equal`.
- `==` used to compare structs that have private fields, slices, maps, or pointers.
- Comparing protos with `==` or `reflect.DeepEqual` (silently broken — must use `protocmp.Transform`).
- Comparing `time.Time` with `==` (compares wall + monotonic + location; usually want `time.Equal()` or `cmpopts.EquateApproxTime`).
- Custom equality helpers (`func equalUsers(a, b User) bool { … }`) re-implementing what `cmp` already provides.

---

## #level-of-detail — Level of detail

> The conventional failure message, which is suitable for most Go tests, is `YourFunc(%v) = %v, want %v`.

> Tests performing complex interactions should describe the interactions too.

Two tiers:

1. **Simple test** → conventional one-line message: `MyFunc(%v) = %v, want %v`.
2. **Complex / multi-step interaction** → message includes interaction context: `"after starting server with %v and posting %v, status = %v, want %v"`.

The principle: someone debugging a 2am failure should learn from the message what the test was *trying* to do, not just that some comparison failed.

### Smells

- Trivial tests with novella-length failure messages.
- Multi-step integration tests with single-word failure messages (`t.Errorf("FAIL")`).
- Failure message that omits the *step* of the interaction that failed (just says "got X want Y" without "after step 3...").
- Stable but unhelpful messages like `t.Errorf("test failed at line %d", line)` — line number is in the test runner output already; instead describe what's wrong.
- Repeated boilerplate prefix in a test (`"in scenario A, "...`, `"in scenario A, "...`) instead of a `t.Run("scenario A", ...)` subtest.

---

## Consolidated review checklist (all 9 rules)

- [ ] No assertion-library helpers (testify, gomega, project-local `assert` packages).
- [ ] Failure messages name the function under test.
- [ ] Failure messages include the input (or, for long inputs, the named test case).
- [ ] Failure messages put `got` before `want`; use the words `got` / `want`.
- [ ] Diff messages include a `(-want +got)` (or `(-got +want)`) legend.
- [ ] Struct/slice/map comparisons use `cmp.Equal` / `cmp.Diff`, not field-by-field `if` chains or `reflect.DeepEqual`.
- [ ] Tests don't compare unstable serialized output (proto text, JSON bytes, `fmt.Sprintf("%v")`) — they compare structured values.
- [ ] `t.Fatal` reserved for cases where continuing is meaningless; `t.Error` used for value mismatches.
- [ ] Proto comparisons use `protocmp.Transform`; time comparisons use `time.Equal()` or `cmpopts.EquateApproxTime`.
- [ ] Failure-message detail is proportional to the test's complexity.

## Suggested finding phrasing

- "`assert.Equal(t, want, got)` from `testify` — Decisions §assertion-libraries: replace with `if !cmp.Equal(got, want) { t.Errorf(\"MyFunc(...) = %v, want %v\", got, want) }`."
- "`t.Errorf(\"got %v, want %v\", got, want)` omits the function under test — Decisions §identify-the-function: include the name (`\"MyFunc(%v) = %v, want %v\"`)."
- "Subtest case name is empty (`t.Run(\"\", ...)`) — Decisions §identify-the-input: cases must be named so failures can identify the row."
- "`t.Errorf(\"want %v, got %v\", want, got)` — Decisions §got-before-want: order is got, then want."
- "`if x.Foo != \"\" { ... } if x.Bar != 0 { ... }` testing return struct — Decisions §full-structure-comparisons: use `cmp.Diff(want, got)` instead."
- "Test compares `proto.MarshalTextString(p1) == proto.MarshalTextString(p2)` — Decisions §compare-stable-results + §types-of-equality: compare protos with `cmp.Diff(want, got, protocmp.Transform())` instead of serialized strings."
- "`t.Fatalf(\"x = %v, want %v\", got, want)` for a value mismatch — Decisions §keep-going: use `t.Errorf` so subsequent assertions also run."
- "`reflect.DeepEqual(got, want)` in test — Decisions §types-of-equality: prefer `cmp.Equal` (handles unexported fields, protos, time correctly)."
- "Multi-step integration test with `t.Errorf(\"FAIL\")` — Decisions §level-of-detail: describe which step failed and what the expected interaction was."

## Sources

- <https://google.github.io/styleguide/go/decisions#useful-test-failures> — Google Go Style Decisions, Useful test failures section (the canonical text quoted above)
- <https://pkg.go.dev/github.com/google/go-cmp/cmp> — `go-cmp` (`cmp.Equal`, `cmp.Diff`)
- <https://pkg.go.dev/google.golang.org/protobuf/testing/protocmp> — `protocmp.Transform` (required for proto comparisons)
- <https://pkg.go.dev/github.com/google/go-cmp/cmp/cmpopts> — `cmpopts` (`IgnoreFields`, `EquateApproxTime`, etc.)
