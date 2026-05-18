# Hermes Rust Parity Rules

Follow these conventions for all porting and parity-related changes (consistent with `PARITY_PLAN.md` Week 0).

## General Porting Conventions

1. Any porting task must begin by reading the corresponding Python source file (`research/hermes-agent`) and the relevant Rust crate directory structure.
2. Public API names must match the Python side (`snake_case`).
3. Prefer using existing `AgentError` / `ToolError` types within each crate; avoid creating parallel error hierarchies.
4. Use `tracing::{debug,info,warn,error}` for logging; avoid ungated `println!` (except for CLI user-facing output).
5. Use **tokio** for async code; do not use async-std.
6. Comparable behavior should be provided as golden fixtures via `crates/hermes-parity-tests/fixtures/<module>/*.json`; when adding new cases, also update `scripts/record_fixtures.py` (if applicable).
7. Each PR should port only one module; suggested commit message: `parity(<module>): port from python …`

## Prohibited Actions

- Do not arbitrarily change the workspace member structure (adding a new crate requires a clear justification and an update to the root `Cargo.toml`).
- New top-level dependencies must align with the version policy already in the root `Cargo.toml`.
- Before merging, eliminate any new `clippy` warnings as much as possible (the goal is `-D warnings` across the entire repo; current CI may still allow pre-existing warnings).

## Parity Tests

```bash
cargo test -p hermes-parity-tests
```

- Module status is tracked in `crates/hermes-parity-tests/fixtures/registry.json`.
- Content in `fixtures/pending/` is not included in `run_all_active_fixtures` by default.

Recording fixtures from the Python side:

```bash
python3 scripts/record_fixtures.py
```

When the Python repository is unavailable, the script still outputs **checkpoint directory hashes** (consistent with the shadow directory naming algorithm used by `checkpoint_manager`).

## Evaluation Result Persistence

`hermes-eval` uses [`JsonReporter`](crates/hermes-eval/src/reporter.rs) to write [`RunRecord`](crates/hermes-eval/src/result.rs) as JSON; baseline comparison / Parquet output can be built on top of this.

### Real Agent Rollout (Non-Noop)

Build the same [`hermes_agent::AgentLoop`](crates/hermes-agent) used by `hermes-cli`, enable the crate feature **`agent-loop`**, and pass [`AgentLoopRollout`](crates/hermes-eval/src/agent_rollout.rs) as the [`TaskRollout`](crates/hermes-eval/src/runner.rs) to [`Runner::run`](crates/hermes-eval/src/runner.rs):

`cargo build -p hermes-eval --features agent-loop`
