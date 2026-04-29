---
applyTo: "**/*.go"
---

# Go Code Review Guidelines

## Naming

- Local variables favor concision; exported names favor clarity
- Check the codebase for existing terminology before introducing new names
- Packages should have a clear, focused responsibility; name them for what they provide (`auth`, `orders`), not what they contain (`utils`, `helpers`)
- Package names: lowercase, concise, no underscores; multi-word names stay unbroken (`paymentgateway`)
- If utility functions are needed, scope them: `strutil` rather than `utils`
- When a package exports a primary type, name its constructor `New` (`api.New()`, not `api.NewAPI()`)
- Method receivers: short (one to three letters), abbreviate the type, consistent across all methods; no `this` or `self`

## Functions

- Parameter order: `context.Context` first, then dependencies (loggers, stores), then main entities, then ancillary params; `error` last in returns
- Keep cyclomatic complexity low (under 10 for most functions), but note Go's error handling inflates this metric
- Extract helpers for reuse or readability, but let patterns repeat a few times before extracting
- Prefer early returns over nesting to keep code flat
- Define interfaces where they are used, not where they are implemented; an interface should capture exactly the behavior the consumer depends on, nothing more

## Documentation

- All exported names must have doc comments following Go conventions
- Inline comments should not restate what the code already expresses

## Errors

- Messages: lowercase (except proper nouns), no trailing period
- Use "cannot X" when a precondition prevents the operation
- Use "failed to X" when an operation was attempted and did not succeed
- Use `%w` to wrap errors when callers need `errors.Is`/`errors.As`
- Define sentinel errors in a shared package so layers can check error types without importing implementation packages
- Do not prefix messages with package or component names; error wrapping provides that context
- The store layer translates driver errors into domain sentinels (e.g., `pgx.ErrNoRows` to `models.ErrNotFound`). Handlers map sentinels to HTTP status codes via `errors.Is`. Internal details must never leak to clients
- Only `main` should panic; all other code propagates errors

## Concurrency

- Keep goroutine management visible from `main`; launch goroutines in `run()` using `errgroup`
- Background workers: use a `for`/`select` loop listening for work and context cancellation
- Pass `context.Context` through call chains for cancellation and timeouts
- Do not store application data in context values; context is for request-scoped metadata like trace IDs

## Patterns

- Never pass the full `*config.Config` beyond `main`; each subsystem receives only its config slice (e.g., `&cfg.PostgreSQL` to the store, `&cfg.API` to the API layer)
- No ORM or query builder; use raw SQL with `pgx.NamedArgs` (`@name` parameter syntax)
- Use `log/slog` for logging
- Include relevant context as key-value pairs: `slog.Info("order created", "order_id", order.ID)`, `slog.Error("failed to create order", "err", err, "order_id", orderID)`

## Testing

- Name tests `Test<Operation>_<Scenario>_<Outcome>` (e.g., `TestManageSessions_Create_Success`, `TestExchange_SessionNotFound`)
- Use table-driven tests for parameterized cases where appropriate
- Tests live in `<file>_test.go` in a separate `<pkg>_test` package; use `export_test.go` for internal access
- Use testify suites to group related tests and share setup; use `SetupTest` for per-test initialization and `SetupSuite` for expensive one-time setup like starting test containers
- Hand-write mocks with `testify/mock.Mock` embedded, placed in the test file that uses them; no mock generation tools
- Mock at boundaries (between layers, external systems), not within a package; always call `AssertExpectations` to verify mock calls
- Bug fixes must be covered by new tests that fail before the fix and pass after
