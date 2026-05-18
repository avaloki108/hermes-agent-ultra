# Hermes Rust 100% Parity — AI-Assisted Delivery Plan

> **Goal**: Bring `hermes-agent-rust` to **100% user-perceived feature parity** with `hermes-agent` (Python) within **8 weeks (solo + AI)** or **5 weeks (two-person + AI)**.
>
> **Baseline**: Python `NousResearch/hermes-agent@v2026.4.13`
> **Definition of Done**: Each module passes Python fixture comparison tests + end-to-end integration tests

---

## I. Overall Timeline

```
Week 0  | Infrastructure setup (parity test framework, fixture generation, CI)
Week 1  | P0-A: Filesystem snapshots + process_registry + channel_directory
Week 2  | P0-B: send_message full pipeline + security policy engine
Week 3  | P0-C: Training/evaluation environment (SWE + benchmarks + parsers)
Week 4  | P0-D: voice_mode + mixture_of_agents + TTS/transcription complete
Week 5  | P1-A: Browser matrix + auxiliary_client + context_compressor
Week 6  | P1-B: copilot_acp_client + models_dev + context_references
Week 7  | P2:   CLI command completion + root scripts (batch_runner, etc.)
Week 8  | Integration regression + 17-platform live testing + release
```

---

## II. Infrastructure (Week 0)

### 0.1 Parity Test Framework (Must Be Done First)

Create `crates/hermes-parity-tests/` containing:

```
hermes-parity-tests/
├── fixtures/                    # Input/output recorded from the Python side
│   ├── checkpoint_manager/
│   ├── send_message/
│   └── ...
├── src/
│   ├── harness.rs              # Comparison runner
│   └── recorder.rs             # Python-side recording tool (PyO3 or subprocess)
└── Cargo.toml
```

**Recording script (Python side)** `scripts/record_fixtures.py`:

```python
# Run batch test cases for each Python module and store (input, output, side_effects) as JSON
# Goal: make `cargo test --package hermes-parity-tests` reproducible on the Rust side
```

### 0.2 AI Development Environment Configuration

Add `.cursor/rules/` or `AGENTS.md` to the project root:

```markdown
# Hermes Rust Parity Rules

## General Porting Conventions
1. Any porting task must first read the corresponding Python source file + relevant Rust crate directory structure
2. All public API signatures retain Python naming (snake_case -> Rust snake_case)
3. Use existing `AgentError` / `ToolError` error types within the crate; do not create new ones
4. Use `tracing::{debug,info,warn,error}` for logging; no println!
5. Use tokio runtime for async functions; do not use async-std
6. Tests must assert against fixtures/<module_name>/*.json
7. Each PR ports only one module; commit message format: `parity(<module>): port from python v2026.4.13`

## Prohibited Actions
- Do not modify the crate workspace structure
- Do not introduce new top-level dependencies (must align with versions already in workspace Cargo.toml)
- Do not skip clippy warnings
```

### 0.3 CI Pipeline (`.github/workflows/parity.yml`)

```yaml
jobs:
  parity:
    steps:
      - cargo fmt --check
      - cargo clippy --workspace --all-targets -- -D warnings
      - cargo test --workspace --all-features
      - cargo test --package hermes-parity-tests
```

**Week 0 Deliverables**:
- [x] Parity test framework compiles (`crates/hermes-parity-tests`, `cargo test -p hermes-parity-tests`)
- [x] Python fixtures recorded for at least 3 modules (`anthropic_adapter`x2 files + `hermes_core`x1; golden files cross-validated with `scripts/record_fixtures.py`)
- [x] AI rules file in place (root `AGENTS.md`)
- [x] CI green (`.github/workflows/ci.yml` includes parity steps; equivalent coverage to `cargo test --workspace`)

---

## III. Sprint Detailed Plan

### Week 1 — P0-A: Filesystem and Process Management

#### Day 1-2: `checkpoint_manager` (Python 541 lines -> Rust ~700 lines estimated)

**Dependency**: `git2` crate (use directly if already in workspace, otherwise add it)

**Prompt Template**:

```
Read the complete implementation of @research/hermes-agent/tools/checkpoint_manager.py.

Task: Implement an equivalent Rust version in crates/hermes-tools/src/backends/checkpoint.rs.

Requirements:
1. Use git2-rs to operate a shadow git repository (path ~/.hermes/shadow/<workspace_hash>)
2. Preserve the snapshot/rollback/list/diff interface semantics from Python
3. Automatically create a commit before each turn with message format: `turn-{turn_id}-{timestamp}`
4. Register with ToolRegistry, tool name: `checkpoint`
5. Add fixture comparison tests @crates/hermes-parity-tests/fixtures/checkpoint_manager/*.json

Reference existing implementations:
- @crates/hermes-tools/src/backends/file.rs (learn trait implementation style)
- @crates/hermes-tools/src/registry.rs (learn registration mechanism)
```

**Acceptance Criteria**:
- [ ] `cargo test -p hermes-tools checkpoint` all green
- [ ] Can snapshot a 10MB workspace in <100ms
- [ ] Manual test: modify file -> rollback fully restores it

#### Day 3: `process_registry` (Python 1045 lines -> Rust ~1200 lines)

**Prompt Template**:

```
The current crates/hermes-tools/src/tools/process_registry.rs is only 61 lines (stub).

Task: Fully rewrite it based on @research/hermes-agent/tools/process_registry.py.

Key capabilities to preserve:
1. Background process launching (spawn + PID management)
2. Log capture (stdout/stderr separate ring buffers, default 10MB)
3. Graceful termination (SIGTERM -> timeout -> SIGKILL)
4. List/status query/output retrieval
5. Crash-restart policy (optional)
6. Process persistence to ~/.hermes/processes.json, restored after restart

Use tokio::process, not std::process.
```

#### Day 4-5: `channel_directory` (Python 272 lines + persistence)

**Prompt Template**:

```
Rust current state: crates/hermes-gateway/src/channel_directory.rs has only an in-memory HashMap.

Task: Align with @research/hermes-agent/gateway/channel_directory.py by adding:
1. Persistence to ~/.hermes/channel_directory.json
2. Cross-platform channel resolution ("telegram:12345" / "discord:67890")
3. Load on startup, atomic writes on update (write temp file + rename)
4. Channel alias / nickname system
5. Query interface for integration with send_message_tool

Test: Verify that all channel mappings are restored after a process restart.
```

**Week 1 Deliverables**:
- [ ] `checkpoint` tool functional
- [ ] `process_registry` equivalent to Python
- [ ] `channel_directory` persistence correct
- [ ] Parity test pass rate >= 90%

---

### Week 2 — P0-B: Message Delivery and Security Policy

#### Day 1-3: `send_message_tool` Real Delivery (Python 1043 lines)

**Dependency**: Week 1's `channel_directory`

**Prompt Template**:

```
Read:
- @research/hermes-agent/tools/send_message_tool.py
- @research/hermes-agent/gateway/delivery.py
- @crates/hermes-tools/src/tools/messaging.rs (current stub)
- @crates/hermes-gateway/src/delivery.rs (existing delivery infrastructure)

Task: Refactor messaging.rs so that it:
1. No longer returns status=pending, but actually calls the Gateway for delivery
2. Parses channel_ref (supports alias, platform prefix, bare user_id number)
3. Auto-splits large messages (markdown_split already exists, reuse it)
4. Uploads media attachments (images/audio/files)
5. Retry on failure + fallback (use fallback platform if primary fails)

Do not re-implement an HTTP client inside the tool; always delegate to the corresponding
platform adapter via the GatewayAdapter trait.
```

#### Day 4: `tirith_security` (Python 670 lines)

**Prompt Template**:

```
Read @research/hermes-agent/tools/tirith_security.py to understand its policy model.

Create crates/hermes-tools/src/tirith.rs implementing:
1. Policy DSL parsing (YAML/JSON rules)
2. Pre-check hook before tool calls
3. Tiered violation handling (warn / block / require_approval)
4. Integration with approval.rs

Sample rules must be loadable from ~/.hermes/tirith_rules.yaml.
```

#### Day 5: `website_policy` (Python 282 lines)

**Prompt Template**:

```
Read @research/hermes-agent/tools/website_policy.py.

Implement in crates/hermes-tools/src/website_policy.rs:
1. URL allow/deny lists (by domain/path/regex)
2. Pre-check interface for web_tools.rs and browser.rs
3. Hot-reload of policies (file mtime polling)
```

**Week 2 Deliverables**:
- [ ] send_message can deliver to Telegram + Discord + Slack for real (at least these 3 live tests)
- [ ] tirith + website_policy integrated into the tool pipeline
- [ ] Security policy fixture tests passing

---

### Week 3 — P0-C: Training/Evaluation Environment (Hardest Week)

> Warning: **High-risk week**: Deep Python ecosystem dependency (swe-bench / datasets);
> recommended strategy: "Python subprocess fallback"

#### Day 1-2: Set up `hermes-environments` training environment submodule

**New directory structure**:

```
crates/hermes-environments/src/
├── training/              # New
│   ├── mod.rs
│   ├── base_env.rs       # HermesBaseEnv trait
│   ├── agent_loop.rs     # Training-specific loop
│   ├── tool_context.rs
│   ├── patches.rs
│   └── parsers/          # tool_call_parsers for each model
│       ├── anthropic.rs
│       ├── openai.rs
│       ├── qwen.rs
│       └── ...
```

#### Day 3-4: `hermes_swe_env` + `web_research_env` + `agentic_opd_env`

**Prompt Template**:

```
Read all files under @research/hermes-agent/environments/hermes_swe_env/.

Strategy: hybrid implementation
- Core env loop, tool calls, and trajectory recording in Rust
- SWE-bench dataset loading via `python3 -c "..."` subprocess calls
  (avoid rewriting the datasets library)

Implement in crates/hermes-environments/src/training/swe.rs,
maintaining compatibility with the default.yaml config format.
```

#### Day 5: `benchmarks/` (tblite + terminalbench_2 + yc_bench)

**Prompt Template**:

```
Each benchmark corresponds to a submodule:
- training/benchmarks/tblite.rs
- training/benchmarks/terminalbench.rs
- training/benchmarks/yc_bench.rs

Each implementation:
1. Dataset loading (reuse Python subprocess fallback)
2. Task execution driver
3. Scoring / pass@k calculation
4. Result serialization (bit-exact alignment with Python output format)
```

**Week 3 Deliverables**:
- [ ] Can run a small SWE-bench sample (10 tasks) with results matching Python
- [ ] 3 benchmark runners functional
- [ ] tool_call_parsers cover all models supported by Python

---

### Week 4 — P0-D: Multimedia Tools

#### Day 1-2: `voice_mode` (Python 1016 lines)

**Prompt Template**:

```
Current voice_mode.rs is only a 31-line placeholder.

Read @research/hermes-agent/tools/voice_mode.py in full.

Implementation highlights:
1. Audio recording (cpal crate)
2. VAD (Voice Activity Detection) simple threshold implementation
3. Streaming transcription (connect to existing transcription.rs)
4. Voice command triggering (wake word optional, skip for now)
5. TTS playback (connect to Week 4 TTS)

Frontend WebSocket endpoint: /ws/voice
```

#### Day 3: `mixture_of_agents_tool` (Python 562 lines)

**Prompt Template**:

```
Current mixture_of_agents.rs is only 32 lines.

Read @research/hermes-agent/tools/mixture_of_agents_tool.py.

Implement:
1. Parallel prompt dispatch to multiple providers (tokio::join_all)
2. Aggregation layer (aggregator model) synthesizes final response
3. Configurable aggregator (model / prompt template)
4. Cost summary + latency reporting
```

#### Day 4-5: TTS + Transcription Complete

**Prompt Template (TTS)**:

```
Current tts_premium.rs is just an ElevenLabs queued placeholder.

Read @research/hermes-agent/tools/tts_tool.py (983 lines) + neutts_synth.py (104 lines).

Implement:
1. Multiple backends: ElevenLabs (real HTTP) + OpenAI TTS + neutts (ONNX local inference)
2. Streaming audio output (chunked)
3. Caching (by text hash) to ~/.hermes/tts_cache/

Use the ort crate to load the NeuTTS ONNX model.
Model files load from ~/.hermes/models/neutts/.
```

**Prompt Template (Transcription)**:

```
Current transcription.rs is 107 lines, Whisper API only.

Read @research/hermes-agent/tools/transcription_tools.py (708 lines).

Add:
1. Local Whisper (whisper-rs crate)
2. Streaming transcription (segmented + real-time output)
3. Multi-language support
4. Speaker diarization (optional, skip for v1)
```

**Week 4 Deliverables**:
- [ ] voice_mode can complete the full record -> transcribe -> TTS playback flow
- [ ] mixture_of_agents makes real parallel calls to 3 providers
- [ ] TTS three backends functional

---

### Week 5 — P1-A: Browser and Agent Depth

#### Day 1-3: Browser Matrix

**Prompt Template**:

```
Current backends/browser.rs is only a thin CamoFox CDP wrapper.

Read:
- @research/hermes-agent/tools/browser_camofox.py (592 lines)
- @research/hermes-agent/tools/browser_providers/*.py
- @research/hermes-agent/tools/browser_tool.py (2218 lines)

Refactor backends/browser.rs into a trait + multiple implementations:
- trait BrowserProvider { async fn navigate/click/fill/screenshot/... }
- CamoFoxProvider (full CDP WebSocket driver using chromiumoxide crate)
- BrowserbaseProvider (HTTP API)
- BrowserUseProvider (Python subprocess fallback acceptable)
- FirecrawlProvider (extracted from existing web backend)

browser_tool.rs acts as a unified entry point, selecting the provider based on configuration.
```

#### Day 4: `auxiliary_client` (Python 2261 lines)

**Prompt Template**:

```
Read @research/hermes-agent/agent/auxiliary_client.py (2261 lines).

This module is large; port it in three steps:
1. First scan it fully and list all public functions + responsibilities
2. Implement in crates/hermes-intelligence/src/auxiliary_client.rs
   organized by responsibility region
3. Focus on "auxiliary LLM calls": session_search summaries, title generation,
   insights extraction, review pass

Reuse existing provider.rs / credential_pool.rs; do not reinvent the wheel.
```

#### Day 5: `context_compressor` Completion

**Prompt Template**:

```
Current compression.rs is only 32 lines.

Read @research/hermes-agent/agent/context_compressor.py (738 lines).

Add:
1. Multiple strategies: summarize / truncate / drop_oldest / importance_score
2. Independent compression for tool results (preserve key fields)
3. Separate handling for images/attachments
4. Compression quota management (target tokens / actual tokens)
5. Before/after compression diff logging
```

**Week 5 Deliverables**:
- [ ] Browser 4 providers switchable
- [ ] auxiliary_client full interface coverage
- [ ] context_compressor passes compression quality tests
  (compression ratio >= 60%, no task completion degradation)

---

### Week 6 — P1-B: Remaining Agent Layer

| Day | Module | Python LOC | Rust Target |
|---|---|---|---|
| 1-2 | `copilot_acp_client` | 570 | `hermes-agent/src/copilot_acp.rs` |
| 3 | `models_dev` | 670 | `hermes-intelligence/src/models_dev.rs` |
| 4 | `context_references` | 491 | `hermes-agent/src/context_refs.rs` |
| 5 | Buffer / wrap-up + integration tests | — | — |

**Prompt Template (copilot_acp_client)**:

```
Read @research/hermes-agent/agent/copilot_acp_client.py (570 lines) +
the copilot_acp-related parts of smart_model_routing.

Implement crates/hermes-agent/src/copilot_acp.rs:
1. GitHub Copilot ACP subprocess launch
2. Handshake + session initialization
3. Prompt/completion forwarding
4. Error handling and reconnection logic

Reuse the protocol layer from the hermes-acp crate; do not re-implement JSON-RPC.
```

---

### Week 7 — P2: CLI and Scripts

**Strategy**: CLI commands are mostly boilerplate; AI can batch-produce 3-5 per day.

#### Day 1-2: High-Value CLI Commands

```
Batch task prompt:

Read these files under @research/hermes-agent/hermes_cli/ and port them to Rust in order:
1. nous_subscription.py -> crates/hermes-cli/src/nous_subscription.rs
2. codex_models.py      -> crates/hermes-cli/src/codex_models.rs
3. region.py            -> crates/hermes-cli/src/region.rs
4. memory_setup.py      -> crates/hermes-cli/src/memory_setup.rs
5. runtime_provider.py  -> fill out runtime_providers in crates/hermes-cli/src/app.rs

For each module when complete:
- Register the subcommand in cli.rs
- Add --help text (matching Python)
- Write at least one integration test
```

#### Day 3: CLI Command Patches (Remaining)

```
Batch fill-in:
- status.py        -> expand run_status in main.rs from 80 lines to Python's 465-line equivalent
- gateway.py       -> split the 2510 lines into gateway_cmd.rs submodules
- auth_commands.py -> complete auth.rs
- clipboard.py     -> standalone crate or tui/clipboard.rs module
- plugins_cmd      -> plugins subcommand in commands.rs
```

#### Day 4-5: Root Python Modules

```
Port:
- batch_runner.py (1287)          -> hermes-rl/batch_runner.rs (complete)
- trajectory_compressor.py (1455) -> hermes-rl/trajectory_compressor.rs (complete)
- mini_swe_runner.py (709)        -> hermes-rl/mini_swe_runner.rs (new)
- rl_cli.py (446)                 -> rl subcommand in hermes-cli
- model_tools.py (577)            -> hermes-tools/src/model_tools.rs
- mcp_serve.py (867)              -> verify whether hermes-mcp crate already covers this

Handle ops scripts separately:
- scripts/discord-voice-doctor.rs (Rust version)
- scripts/sample_and_compress.rs
```

**Week 7 Deliverables**:
- [ ] All Python CLI commands have a corresponding `hermes <subcommand>` on the Rust side
- [ ] `hermes --help` output differs from Python `hermes --help` only in copyright/version lines

---

### Week 8 — Integration, Live Testing, Release

#### Day 1-2: 17-Platform Live Testing

For each Gateway platform:
1. Send a "hello" from a test account
2. Verify the bot responds
3. Send an image / audio to verify media delivery
4. Log any issues found

**Prompt Template**:

```
When testing @platform I encountered <specific error>.

The equivalent path on the Python side is <specific function>
in @gateway/platforms/<platform>.py.

The Rust side is at @crates/hermes-gateway/src/platforms/<platform>.rs.

Compare both implementations, find the discrepancy, and fix it.
```

#### Day 3: Performance Regression

```bash
# Compare end-to-end latency between Python and Rust
cargo bench --workspace
python benchmarks/e2e_latency.py

# Goal: Rust is not slower than Python in any scenario; cold start >= 3x faster
```

#### Day 4: Documentation

- Update `README.md` (change "13/13 parity" to "Full parity with Python v2026.4.13")
- Write `MIGRATION.md` (how Python users can migrate)

#### Day 5: Release

- Cut release tag `v1.0.0`
- GitHub Actions auto-builds 6-platform binaries
- Update Homebrew formula
- Install script smoke test

---

## IV. Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Week 3 training environment implementation runs over schedule | High | 1-2 week delay | Prepare Python subprocess fallback strategy in advance |
| Difficult debugging for full browser CDP driver | Medium | 3-5 day delay | Use chromiumoxide crate, which has a mature implementation |
| NeuTTS ONNX model size/licensing issues | Medium | May drop local TTS | Keep only ElevenLabs + OpenAI two cloud backends |
| Cannot obtain accounts for all 17 platforms | Medium | Some platforms can only be automated-tested | Prioritize: TG/DC/Slack must be tested live; others just smoke test |
| AI-generated code enters clippy failure loop | Medium | Extra 0.5-1h per module | Force `#[allow(...)]` explicit declarations in the prompt |

---

## V. Recommended Daily Workflow

```
09:00 - 09:30  Read Python source + corresponding Rust state (no coding, just understand)
09:30 - 10:00  Generate fixtures (run Python recording script)
10:00 - 12:00  First AI prompt round -> get skeleton code
12:00 - 13:00  Lunch break + cargo build + review compiler errors
13:00 - 16:00  Second AI prompt round (with compiler errors) -> fix -> compile successfully
16:00 - 17:30  Run parity tests, fix diffs
17:30 - 18:00  commit + push + update TODO
```

---

## VI. General AI Prompt Templates

### Template 1: Module Porting

```
### Background
I am porting @https://github.com/NousResearch/hermes-agent to Rust (hermes-agent-rust).

### Task
Port <module_name> from @<python_file_absolute_path> to Rust.

### Context (read as needed)
- Python source file: <path>
- Related Rust crate: <path>
- Similar existing implementation (for style reference): <path>
- Error type conventions: @crates/hermes-core/src/error.rs
- Fixture tests: @crates/hermes-parity-tests/fixtures/<module>/

### Constraints
1. Keep Python behavior bit-exact (compare against fixture)
2. Only modify <target_file>; do not touch other files (unless absolutely necessary)
3. Use tracing for logging, not println
4. Add doc comments to all public functions
5. Use thiserror to define errors; do not hand-write Display

### Deliverables
1. Implementation code
2. Unit tests (coverage >= 80%)
3. One-line commit message: `parity(<module>): port from python v2026.4.13`
```

### Template 2: Debug Fix

```
### Current State
@<rust_file> fails on `<test_name>` in @<test_file>:

<compiler error or test output>

### Equivalent Python Implementation
@<python_file> function <function_name> (lines X-Y)

### Task
Compare against the Python logic and fix the Rust implementation.
Only change the necessary lines; leave the rest of the code unchanged.
```

### Template 3: Refactor and Merge

```
### Background
@<rust_file_1> and @<rust_file_2> have duplicated logic
corresponding to Python's <python_function>.

### Task
1. Extract the common function to @<shared_module>
2. Update both call sites to reference it
3. Ensure all existing tests pass (cargo test -p <crate>)
```

---

## VII. Weekly Checkpoints

Before end-of-day every Friday:

- [ ] All modules from this week pass `cargo test`
- [ ] `cargo clippy --workspace --all-targets -- -D warnings` produces no warnings
- [ ] Parity test coverage >= 85%
- [ ] Update the parity progress table in `README.md`
- [ ] Update TODO tracker to GitHub Issues

---

## VIII. Post-Release (Week 9+)

Achieving parity is just the beginning. Future work:

1. **Performance optimization**: Rust should show a 3-10x improvement over Python; quantify and showcase in README
2. **Rust-only enhancements**: Use Rust-ecosystem-exclusive capabilities (zero-copy, SIMD, true parallelism) for optimizations Python cannot achieve
3. **Community maintenance**: Track new versions from the Python mainline and continuously increment parity

---

**Final Advice**:

- Write **today's Prompt Templates** in advance before starting each day and paste them into Cursor; don't improvise on the spot
- Maintain the **AI output -> human review -> compile verification -> test verification** four-step loop; do not skip review
- Week 3 is the most likely to go wrong; add an extra day of buffer in advance
- Rest on weekends; fatigued coding will drop your AI acceleration ratio below 1.5x

**Good luck.**
