---
paths:
  - "**/*.go"
---

# Go Coding Guidelines

## Error Handling

- If the caller needs to distinguish between multiple cases → sentinel error (`var ErrXxx = errors.New(...)`) or custom type, must support `errors.Is` / `errors.As`
- If a simple message is enough → `errors.New` or `fmt.Errorf`
- If the caller needs specific fields → custom type suffixed with `Error` (e.g. `ValidationError`)
- Sentinel errors prefixed with `Err` (or `err` if unexported), declared at the top of the file, concise text without "error" / "failure" / "failed"
- Default propagation: `fmt.Errorf("context: %w", err)` — use `%v` only if the caller must never access the original error
- Propagation context must be concise: no "failed to", "error", "failure" — it is implicit in returning an error
- Handle each error exactly once: no log + return on the same error — choose one or the other

```go
// Good
var ErrNotFound = errors.New("not found")

if err != nil {
    return fmt.Errorf("fetch user: %w", err)
}

// Bad
if err != nil {
    log.Error("failed to fetch user", zap.Error(err))
    return fmt.Errorf("failed to fetch user: %w", err) // handled twice
}
```

## Logging (uber-go/zap)

- Use `zap.L()` exclusively — never `zap.S()` (performance: ~22% faster, no string allocation)
- Messages in lowercase, no trailing punctuation
- Static description — never interpolate variables in the message, use typed fields instead
- Accumulate context before logging rather than logging step by step
- Log levels:
  - `Debug`: dev only, never critical in production
  - `Info`: normal flow, expected operations
  - `Warn`: potentially problematic but recoverable situation
  - `Error`: must be addressed, Datadog alert expected
- Expected business error → `Info` or `Warn`
- Common/frequent error → Prometheus counter, no log
- Abnormal error → `Error` + Datadog alert configured
- Use a `hint` field when the probable cause is known

```go
// Good
zap.L().Info("user authenticated",
    zap.String("user_id", userID),
    zap.Duration("duration", elapsed),
)

// Bad
zap.S().Infof("user %s authenticated in %v", userID, elapsed)
zap.L().Info(fmt.Sprintf("user %s authenticated", userID)) // variable in the message
```

## Tracing (OpenTelemetry)

- Do not trace every function — only key operations (I/O, external calls, critical business logic)
- Span names and attribute keys in `snake_case`
- Span names without call hierarchy: not `login.get_user`, just `get_user`
- `span.RecordError` and `span.SetStatus` only in the function that actually handles the error — not at every propagation level
- Log the error once, at the level where context is richest
- Use the parent span to add attributes when a dedicated span is not necessary
- No large or sensitive data in attributes (PII, tokens, passwords)

```go
// Good
ctx, span := tracer.Start(ctx, "get_user")
defer span.End()

span.SetAttributes(attribute.String("user_id", userID))

if err != nil {
    span.RecordError(err)
    span.SetStatus(codes.Error, err.Error())
    return fmt.Errorf("get user: %w", err)
}

// Bad
ctx, span := tracer.Start(ctx, "login.get_user") // hierarchy in the name
span.SetAttributes(attribute.String("password", password)) // sensitive data
```
