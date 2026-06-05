# AGENTS.md

## Project Context

This is a Rust-based project. Prefer idiomatic Rust, clear ownership boundaries, explicit error handling, and changes that preserve compile-time safety.

## Directory Guidance

- Before inspecting files in any directory, first read that directory's `AGENTS.md` file when present.
- Treat each directory-level `AGENTS.md` as the summary of that directory's purpose, important modules, local conventions, and current maintenance notes.
- After updating source code in a directory, update that directory's `AGENTS.md` so it remains accurate.
- If a source directory does not yet have an `AGENTS.md`, create one when your change meaningfully affects that directory's structure, behavior, or conventions.

## Planning

- Use `docs/plan/` for planning work.
- Before starting substantial work, inspect `docs/plan/` to understand the current project plan, active decisions, and any known sequencing constraints.
- When creating or updating plans, keep them concrete: include scope, assumptions, implementation steps, validation steps, and unresolved questions.
- Keep plans current as implementation changes the shape of the work.

## Rust Engineering Standards

- Keep functions small, focused, and testable.
- Prefer extracting pure helper functions when behavior can be tested independently.
- Avoid large functions that mix parsing, validation, state mutation, I/O, and presentation.
- Keep module boundaries clear and avoid broad cross-module coupling.
- Prefer explicit types and domain-specific structs/enums over loosely structured data.
- Use `Result` and meaningful error types for recoverable failures.
- Avoid panics in library or core logic unless the invariant is internal, documented, and truly unrecoverable.
- Add or update tests for changed behavior. Prefer focused unit tests for small functions and integration tests for cross-module behavior.

## Review Before Commit

- Before committing code changes, run an adversarial review using a sub-agent with the `caveman` skill.
- The adversarial review should look for correctness bugs, brittle assumptions, missing tests, unclear ownership, poor error handling, unnecessary complexity, and behavior that will be hard to maintain.
- Address review findings before committing, or document why a finding is intentionally deferred.

## Validation

- Run the narrowest relevant Rust checks for the change, then broader checks when the change affects shared behavior.
- Prefer, as applicable:
  - `cargo fmt --check`
  - `cargo clippy --all-targets --all-features`
  - `cargo test --all-features`
- If a check cannot be run, record the reason and any residual risk.
