# Default Go fixer policy

Version: 1

This policy guides the write-capable fixer after the judges have produced a
cited deduction plan. It is never supplied to judges, never creates a
deduction, and never changes a score.

The version line above is provenance, not a compatibility gate: it must match
`^Version:[ \t]*([0-9]+)[ \t]*$` on its own line, and the first such line in the
file wins. A reader that cannot parse it stops rather than reviewing against an
unidentified policy. Nothing branches on the number. In fix mode it travels
with the result as `fixPolicySource` so an edit can be traced to the conventions
that guided its implementation. A fork replacing this file keeps the line and
increments it.

## Structure and contracts

- Prefer flat packages; a package needs multiple consumers or a real swap-out
  story. Keep one dominant type per file and avoid `utils.go`/`helpers.go` bags.
- Keep interfaces small and define them at the point of use. Constructors
  return concrete types when there is one implementation.
- Constructors validate. Exported identifiers have doc comments beginning with
  their name. Discriminator enums use a `Type` suffix, not `Kind`.
- Prefer the standard library and boring code. `const` and `var` groups use
  trailing aligned comments rather than one doc comment per line.
- Once a constructor needs more than three options, prefer functional options
  over growing positional or boolean parameter lists.

## Errors, inputs, and security

- Errors are values: wrap with context using `%w`, classify with
  `errors.Is`/`errors.As`, and never use `panic` or `log.Fatal` in libraries.
- Bound untrusted bytes before parsing them. Invalid state is rejected at
  construction, not deferred downstream.
- Prefer standard-library cryptography. Compare secrets in constant time, use
  parameterized queries, and never put secrets in logs.

## Concurrency and operations

- Propagate `context.Context`; do not introduce `context.Background()` inside a
  call chain. External calls carry deadlines.
- Put `defer mu.Unlock()` immediately after `mu.Lock()`.
- Goroutines have an owner and shutdown path. Queues, worker pools, fan-out,
  retries, and buffers have explicit bounds; retries are capped and backed off.
- Dependencies are passed explicitly. Avoid package-global mutable state and
  real work in `init()`.
- Cross-node operations are idempotent or guarded by a version, CAS, lease, or
  equivalent recovery mechanism. Do not use wall-clock time for ordering.

## Tests, performance, and diagnostics

- Use the standard `testing` package, never testify. Keep tests, benchmarks, and fuzz cases in
  `foo_test.go` beside `foo.go`; do not create separate benchmark/fuzz files.
- Optimize after measuring. Performance claims need a benchmark or profile;
  interesting numbers belong in the change record.
- Reuse hot-path buffers where measurement warrants it. A saturated system
  sheds or bounds work instead of growing without limit.
- Long-lived work preserves context and useful profiler attribution. Prefer
  named goroutine entry points and profiler-visible blocking primitives.
