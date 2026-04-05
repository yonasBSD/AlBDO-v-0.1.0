# AlBDO Professionalization Plan

> **Scope:** `B:\albedo-pre-release` — the full workspace (`dom-render-compiler`, `crates/albedo-node`, `crates/albedo-server`)
>
> **Goal:** Bring the codebase to the standard of a well-maintained, production-grade open-source Rust project — before the public release and GSoC submissions.

---

## Table of Contents

1. [Rustfmt — Enforced Formatting](#1-rustfmt--enforced-formatting)
2. [Clippy — Enforced Linting](#2-clippy--enforced-linting)
3. [Workspace Cargo.toml — Canonical Structure](#3-workspace-cargotoml--canonical-structure)
4. [Dependency Hygiene](#4-dependency-hygiene)
5. [Error Handling Standards](#5-error-handling-standards)
6. [Documentation Standards](#6-documentation-standards)
7. [Testing Infrastructure](#7-testing-infrastructure)
8. [CI/CD Pipeline — GitHub Actions](#8-cicd-pipeline--github-actions)
9. [Pre-commit Hooks](#9-pre-commit-hooks)
10. [Changelog & Versioning](#10-changelog--versioning)
11. [Repository Hygiene](#11-repository-hygiene)
12. [Code Organization Conventions](#12-code-organization-conventions)
13. [Execution Order](#13-execution-order)

---

## 1. Rustfmt — Enforced Formatting

**Create `rustfmt.toml` at the workspace root.**

```toml
edition = "2021"
max_width = 100
tab_spaces = 4
newline_style = "Unix"
use_small_heuristics = "Default"
reorder_imports = true
reorder_modules = true
remove_nested_parens = true
merge_derives = true
use_field_init_shorthand = true
force_explicit_abi = true
normalize_comments = true
wrap_comments = true
comment_width = 100
```

**Rules:**
- Every contributor runs `cargo fmt --all` before committing. No exceptions.
- CI fails if `cargo fmt --all -- --check` exits non-zero.
- Never manually align struct fields or match arms — rustfmt owns that.
- Do not suppress `#[rustfmt::skip]` without a comment explaining why.

---

## 2. Clippy — Enforced Linting

**Create `.clippy.toml` at the workspace root:**

```toml
avoid-breaking-exported-api = true
msrv = "1.75.0"
```

**Add a `[workspace.metadata.clippy]` deny block in `Cargo.toml`**, or use a shared `deny` attribute in each crate root:

```rust
// At the top of every lib.rs / main.rs in the workspace:
#![deny(clippy::all)]
#![warn(clippy::pedantic)]
#![warn(clippy::nursery)]
#![allow(clippy::module_name_repetitions)] // too noisy for this codebase structure
#![allow(clippy::must_use_candidate)]      // revisit before 1.0
```

**Specific lints to explicitly deny (add to lib.rs files):**

```rust
#![deny(
    clippy::unwrap_used,           // use expect() or proper error propagation
    clippy::expect_used,           // allowed only with descriptive message
    clippy::panic,                 // no panics in library code
    clippy::indexing_slicing,      // use .get() with bounds checks
    clippy::integer_arithmetic,    // prefer checked/saturating arithmetic
    clippy::as_conversions,        // use From/Into or explicit casts
    clippy::shadow_unrelated,      // no variable shadowing of unrelated types
    clippy::todo,                  // no TODO stubs in committed code
)]
```

**Run command during development:**
```sh
cargo clippy --workspace --all-targets --all-features -- -D warnings
```


---

## 3. Workspace Cargo.toml — Canonical Structure

The root `Cargo.toml` must be the single source of truth for **all** shared dependency versions. Nothing version-pins in member crates — they inherit from `[workspace.dependencies]`.

**Target structure:**

```toml
[workspace]
resolver = "2"
members = [
    ".",                        # dom-render-compiler (root crate)
    "crates/albedo-node",
    "crates/albedo-server",
]

[workspace.package]
version      = "0.1.0-alpha.1"
edition      = "2021"
rust-version = "1.75.0"        # MSRV — match .clippy.toml
authors      = ["Bishal <you@domain>", "Pinaki Pritam Singha"]
license      = "MIT OR Apache-2.0"
repository   = "https://github.com/your-org/albedo"
homepage     = "https://albedo.dev"
keywords     = ["jsx", "tsx", "compiler", "ssr", "runtime"]
categories   = ["web-programming", "compilers", "development-tools"]

[workspace.dependencies]
# Async runtime
tokio       = { version = "1",  features = ["full"] }
axum        = { version = "0.8" }
# Serialization
serde       = { version = "1",  features = ["derive"] }
serde_json  = { version = "1" }
# Parsing
swc_core    = { version = "0.103" }
# Concurrency
dashmap     = { version = "6" }
rayon       = { version = "1" }
# Error handling
thiserror   = { version = "2" }
anyhow      = { version = "1" }
# Logging / tracing
tracing     = { version = "0.1" }
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
# Testing
criterion   = { version = "0.5", features = ["html_reports"] }

[workspace.lints.rust]
unsafe_code       = "forbid"
missing_docs      = "warn"
unused_imports    = "deny"
dead_code         = "warn"
unused_variables  = "warn"

[workspace.lints.clippy]
all     = "deny"
pedantic = "warn"
```

**Rules for member `Cargo.toml` files:**

```toml
[package]
name.workspace    = true   # ← inherit from workspace where applicable
version.workspace = true
edition.workspace = true

[dependencies]
tokio.workspace = true     # ← always inherit version, never re-pin
serde.workspace = true
thiserror.workspace = true
```

No member crate should ever specify a version string for a dependency that is in `[workspace.dependencies]`.

---

## 4. Dependency Hygiene

### 4a. `cargo-deny` — License & Security Policy

Install and init:
```sh
cargo install cargo-deny --locked
cargo deny init
```

**`deny.toml` (place at workspace root):**

```toml
[licenses]
allow = ["MIT", "Apache-2.0", "Apache-2.0 WITH LLVM-exception", "ISC", "Unicode-DFS-2016"]
deny  = ["GPL-2.0", "GPL-3.0", "AGPL-3.0"]
copyleft = "warn"

[bans]
multiple-versions = "warn"
wildcards         = "deny"   # no wildcard version specs

[advisories]
vulnerability = "deny"
unmaintained  = "warn"
yanked        = "deny"

[sources]
unknown-registry = "deny"
unknown-git      = "deny"
```

**Run:** `cargo deny check` — runs in CI on every push.

### 4b. `cargo-audit` — CVE Scanning

```sh
cargo install cargo-audit --locked
cargo audit
```

Run weekly in CI via a scheduled GitHub Action.

### 4c. `cargo-udeps` — Remove Unused Dependencies

```sh
cargo install cargo-udeps --locked
cargo +nightly udeps --workspace
```

Run before each release to catch bloat.

### 4d. `cargo update` Discipline

- Pin `Cargo.lock` in version control (this is a binary/application workspace, not a library).
- Run `cargo update` periodically and review diffs in `Cargo.lock` before merging.


---

## 5. Error Handling Standards

### 5a. Library crates (`dom-render-compiler`, `albedo-node`)

Use `thiserror` for all public-facing error types. Every module that can fail gets its own typed `Error` enum. No `Box<dyn Error>` in public APIs.

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum CompilerError {
    #[error("parse failed at {file}:{line}: {reason}")]
    ParseFailure { file: String, line: u32, reason: String },

    #[error("cycle detected in component graph: {0}")]
    CycleDetected(String),

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
}
```

### 5b. Binary / CLI crate (`albedo`, `albedo-bench`)

Use `anyhow` for top-level glue code and CLI error reporting. Wrap library errors with `.context()`:

```rust
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let manifest = compiler.compile()
        .context("failed to compile project")?;
    Ok(())
}
```

### 5c. Banned patterns

| Pattern | Why banned | Alternative |
|---------|-----------|-------------|
| `.unwrap()` | Panics in prod | `.expect("reason")` or `?` |
| `.expect()` without message | Undebuggable | Always include a message |
| `panic!()` in lib code | Breaks callers | Return `Err(...)` |
| `eprintln!` for errors | Not structured | `tracing::error!` |
| `Box<dyn Error>` in pub API | Loses type info | Typed `thiserror` enum |

### 5d. The `tracing` contract

- Every async entry point: `#[tracing::instrument]`
- `tracing::debug!` for internal state, `tracing::info!` for user-visible events
- `tracing::error!` for recoverable failures, `tracing::warn!` for degraded paths
- Never use `println!` / `eprintln!` in library or server code

---

## 6. Documentation Standards

### 6a. Crate-level docs

Every `lib.rs` must begin with a module-level doc comment:

```rust
//! # albedo-server
//!
//! Axum-based HTTP runtime for AlBDO compiled JSX/TSX applications.
//!
//! ## Quick start
//! ```rust,no_run
//! use albedo_server::AlbedoServer;
//! # async fn run() {
//! AlbedoServer::builder().port(3000).build().serve().await.unwrap();
//! # }
//! ```
```

### 6b. Public API docs — mandatory

Every `pub` struct, enum, trait, and function **must** have a doc comment. The `missing_docs = "warn"` lint enforces this at the workspace level. Before release, escalate to `"deny"`.

```rust
/// Represents the compiled render manifest for a JSX/TSX project.
///
/// Produced by [`RenderCompiler::compile`] and consumed by [`albedo_server`].
///
/// # Fields
/// The manifest encodes per-route component graphs, hydration modes, and
/// effect lattice tier assignments.
pub struct RenderManifestV2 { ... }
```

### 6c. `rustdoc` lints — add to every `lib.rs`

```rust
#![warn(rustdoc::broken_intra_doc_links)]
#![warn(rustdoc::missing_crate_level_docs)]
#![warn(rustdoc::invalid_codeblock_attributes)]
```

### 6d. README.md at workspace root

Must include:
- One-line pitch
- Architecture diagram (reference `docs/architecture.md`)
- Quickstart (`albedo init`, `albedo dev`, `albedo build`)
- Compatibility matrix (OS × Node version × NAPI target)
- Links to GSoC proposal, landing page, pitch deck

---

## 7. Testing Infrastructure

### 7a. `cargo-nextest` — replace `cargo test`

```sh
cargo install cargo-nextest --locked
cargo nextest run --workspace
```

Benefits: parallel by default, up to 60% faster, richer output, per-test timeouts.

Create `.config/nextest.toml` at workspace root:

```toml
[profile.default]
test-threads = "num-cpus"
retries      = 0

[profile.ci]
test-threads = 4
retries      = 1
fail-fast    = true
```

### 7b. Test organization rules

- **Unit tests** — in the same file as the code they test, in `#[cfg(test)] mod tests { ... }`
- **Integration tests** — in `tests/` at the crate root
- **Benchmarks** — in `benches/` using `criterion`; run via `albedo-bench` binary
- **Doc tests** — every public function with an example must have a working `# Example` block

### 7c. Coverage (optional, pre-release)

```sh
cargo install cargo-tarpaulin --locked
cargo tarpaulin --workspace --out Html
```

Target: 60%+ line coverage on `dom-render-compiler` core modules before public release.


---

## 8. CI/CD Pipeline — GitHub Actions

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main, dev]
  pull_request:

env:
  RUSTFLAGS: "-Dwarnings"
  CARGO_TERM_COLOR: always

jobs:
  fmt:
    name: Formatting
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt
      - run: cargo fmt --all -- --check

  clippy:
    name: Clippy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo clippy --workspace --all-targets --all-features -- -D warnings

  test:
    name: Tests
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - uses: taiki-e/install-action@nextest
      - run: cargo nextest run --workspace --profile ci

  deny:
    name: Dependency Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: EmbarkStudios/cargo-deny-action@v1

  docs:
    name: Docs
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: RUSTDOCFLAGS="-Dwarnings" cargo doc --workspace --no-deps
```

**Separate weekly security workflow** (`.github/workflows/audit.yml`):

```yaml
name: Security Audit
on:
  schedule:
    - cron: '0 9 * * 1'   # Every Monday 9am UTC
  workflow_dispatch:

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: rustsec/audit-check@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

---

## 9. Pre-commit Hooks

Use `cargo-husky` **or** a hand-rolled `.git/hooks/pre-commit` script. The hand-rolled approach is more transparent for a project of this maturity.

Create `.hooks/pre-commit` (and symlink or install via `git config core.hooksPath .hooks`):

```sh
#!/usr/bin/env bash
set -e

echo "→ cargo fmt check"
cargo fmt --all -- --check

echo "→ cargo clippy"
cargo clippy --workspace --all-targets -- -D warnings

echo "→ cargo test (fast)"
cargo nextest run --workspace --profile default -q

echo "✓ All checks passed."
```

**Install for the team (run once after clone):**

```sh
git config core.hooksPath .hooks
chmod +x .hooks/pre-commit
```

Document this in `CONTRIBUTING.md` — it is mandatory for all contributors.

---

## 10. Changelog & Versioning

### 10a. Keep a CHANGELOG

Follow [Keep a Changelog](https://keepachangelog.com) format. File: `CHANGELOG.md` at workspace root.

```markdown
# Changelog

All notable changes to AlBDO are documented here.
Format: Keep a Changelog — https://keepachangelog.com/en/1.1.0/
Versioning: Semantic Versioning — https://semver.org/

## [Unreleased]

### Added
### Changed
### Fixed
### Removed
```

Every PR **must** include a CHANGELOG entry. Enforce this in the PR template.

### 10b. Semantic Versioning contract

- `0.x.y-alpha.z` until public release
- BREAKING changes bump the minor version while in `0.x`
- Use `cargo-release` for version bumps: `cargo install cargo-release --locked`

### 10c. Git tag convention

```
v0.1.0-alpha.1
v0.1.0-alpha.2
v0.1.0          ← first public release
```

---

## 11. Repository Hygiene

### 11a. `.gitignore`

Ensure these are excluded:

```
/target
**/*.rs.bk
.env
.env.local
*.log
dist/
.DS_Store
Thumbs.db
```

### 11b. `.editorconfig`

```ini
root = true

[*]
indent_style  = space
indent_size   = 4
end_of_line   = lf
charset       = utf-8
trim_trailing_whitespace = true
insert_final_newline     = true

[*.md]
trim_trailing_whitespace = false

[*.toml]
indent_size = 2
```

### 11c. Required root files

| File | Status | Notes |
|------|--------|-------|
| `README.md` | Must have | Pitch + quickstart + compat matrix |
| `CHANGELOG.md` | Must have | Keep a Changelog format |
| `CONTRIBUTING.md` | Must have | Hook setup, PR flow, coding standards |
| `LICENSE-MIT` | Must have | Dual-license |
| `LICENSE-APACHE` | Must have | Dual-license |
| `SECURITY.md` | Should have | Vulnerability disclosure policy |
| `rustfmt.toml` | Must have | Formatting config |
| `.clippy.toml` | Must have | Lint config |
| `deny.toml` | Must have | Dep audit config |
| `.editorconfig` | Must have | Cross-editor consistency |
| `.github/workflows/ci.yml` | Must have | CI pipeline |


---

## 12. Code Organization Conventions

### 12a. Module file layout (per crate)

```
src/
  lib.rs          ← crate root; #![deny(...)], pub use re-exports only
  error.rs        ← single crate-level Error enum using thiserror
  types.rs        ← shared data types with no logic (Component, Tier, etc.)
  config.rs       ← configuration structs
  <domain>/
    mod.rs        ← module re-exports
    core.rs       ← primary logic
    tests.rs      ← unit tests (or inline #[cfg(test)])
```

### 12b. Import ordering (enforced by rustfmt)

Rustfmt with `reorder_imports = true` will enforce this automatically:

```rust
// 1. std
use std::collections::HashMap;

// 2. external crates
use dashmap::DashMap;
use serde::{Deserialize, Serialize};

// 3. internal crate (super / crate)
use crate::types::ComponentId;
use super::graph::ComponentGraph;
```

Do not use glob imports (`use foo::*`) except in `prelude` modules and test files.

### 12c. Naming conventions

| Item | Convention | Example |
|------|-----------|---------|
| Types / Traits | `UpperCamelCase` | `RenderManifestV2` |
| Functions / methods | `snake_case` | `compile_component` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_EFFECT_TIER` |
| Lifetime params | short lowercase | `'a`, `'buf` |
| Feature flags | `kebab-case` | `feature = "napi"` |
| Files / modules | `snake_case` | `bundler/emit.rs` |

### 12d. Unsafe code policy

`unsafe_code = "forbid"` at the workspace level. Any unsafe block requires:
1. A `// SAFETY:` comment explaining the invariant being upheld
2. A corresponding issue or PR discussion documenting why safe alternatives were rejected
3. Review by both Bishal and Pinaki before merge

### 12e. Feature flags discipline

Define features intentionally. No `default = ["everything"]` — be additive and explicit:

```toml
[features]
default = []
napi    = ["dep:napi", "dep:napi-derive"]     # Node.js bridge
bench   = ["dep:criterion"]                    # benchmarking harness
full    = ["napi", "bench"]
```

---

## 13. Execution Order

Work through these phases in sequence. Each phase is a PR or a local commit batch.

| Phase | Task | Priority | Effort |
|-------|------|----------|--------|
| **P0** | Add `rustfmt.toml` + run `cargo fmt --all` | 🔴 Critical | 30 min |
| **P0** | Add `.clippy.toml` + fix all `cargo clippy -D warnings` | 🔴 Critical | 2–4 hrs |
| **P0** | Consolidate `[workspace.dependencies]` — remove member-level version pins | 🔴 Critical | 1 hr |
| **P0** | Add `[workspace.lints]` block | 🔴 Critical | 15 min |
| **P1** | Add `deny.toml` + run `cargo deny check` | 🟠 High | 1 hr |
| **P1** | Replace `cargo test` with `cargo nextest` + add `.config/nextest.toml` | 🟠 High | 30 min |
| **P1** | Replace `Box<dyn Error>` and `.unwrap()` violations with `thiserror` / `anyhow` | 🟠 High | 3–6 hrs |
| **P1** | Add `tracing` instrumentation to all async entry points | 🟠 High | 2 hrs |
| **P2** | Write GitHub Actions CI (`fmt` + `clippy` + `test` + `deny` + `docs`) | 🟡 Medium | 1 hr |
| **P2** | Install pre-commit hook (`.hooks/pre-commit`) + document in CONTRIBUTING.md | 🟡 Medium | 30 min |
| **P2** | Add/complete doc comments on all `pub` items in `dom-render-compiler` | 🟡 Medium | 3–5 hrs |
| **P3** | Create `CHANGELOG.md` + backfill unreleased section | 🟢 Normal | 1 hr |
| **P3** | Add `.editorconfig`, `SECURITY.md`, dual `LICENSE-*` files | 🟢 Normal | 30 min |
| **P3** | Run `cargo-udeps`, remove dead dependencies | 🟢 Normal | 1 hr |
| **P4** | Set `missing_docs = "deny"` (after all docs are written) | 🔵 Polish | 30 min |
| **P4** | Set up weekly `cargo audit` GitHub Action | 🔵 Polish | 30 min |
| **P4** | Run `cargo tarpaulin` and document coverage baseline | 🔵 Polish | 1 hr |

---

## Tools Summary

```sh
# Install all tooling in one go:
rustup component add rustfmt clippy
cargo install cargo-deny    --locked
cargo install cargo-audit   --locked
cargo install cargo-nextest --locked
cargo install cargo-udeps   --locked
cargo install cargo-release --locked
cargo install cargo-tarpaulin --locked  # optional, pre-release
```

---

Pinaki and i will get started on this for now and follow thisup throughout the all 4 feature development followed up with the release of V0.2.0

