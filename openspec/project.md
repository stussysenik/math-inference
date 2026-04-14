# math-inference

## Purpose

`math-inference` is a single monolith that replaces six prior attempts at building a mathematically verified LLM tutor / solver: `math-explainer-redwood`, `kimi-math-chat`, `math-chat`, `phoenix-math-stream`, `zig-elixir-math-katex`, and `math_reader`. Its purpose is to produce answers to engineering- and contest-level math problems where every section the user sees has been verified by at least one symbolic engine, ideally two via cross-verification, with a full audit trail that can be scrubbed backward and forward.

The non-negotiable architectural invariant: **LLMs are never in the verification path. Symbolic engines are the sole ground truth.** The LLM routes, parses, and narrates; SymPy, Julia, GNU Octave, Lean 4, and Wolfram Alpha prove.

## Tech Stack

- **Backend**: Elixir 1.19 + Erlang OTP 28, Phoenix 1.8, LiveView, LiveSvelte
- **Frontend**: Svelte 5 (runes), Vite, Tailwind CSS v4, SaladUI primitives, GSAP motion, Phosphor icons via `unplugin-icons`, KaTeX
- **LLM provider**: NVIDIA NIM (OpenAI-compatible streaming API, authenticated via `NIM_API_KEY`)
- **Verification engines**: SymPy (Python 3.13 Port daemon), Julia (slice 2+), GNU Octave (slice 2+), Lean 4 (Oban background jobs, slice 2+), Wolfram Alpha (optional, slice 2+)
- **Persistence**: In-memory ETS / Agent for slice 1, Ecto + Postgres 16 for slice 2+
- **Web transport**: LiveView for the web UI, JSON + SSE REST API for native clients, Phoenix Channels for mobile real-time, OpenAPI 3.1 spec via `open_api_spex`
- **Dev environment**: Nix flake pinning all five runtimes, `direnv` with `use flake`
- **Pre-commit**: Lefthook
- **Deployment**: Railway via Nixpacks (slice 3+), single region initially
- **Observability**: Telemetry + Phoenix LiveDashboard; optional Langfuse in a later slice

## Project Conventions

### Code Style

- **Elixir**: `mix format` is the single source of truth. `mix credo --strict` enforces additional style rules. `mix dialyzer` runs in CI.
- **Svelte / TypeScript**: Prettier formats, ESLint lints, `tsc --noEmit` + `svelte-check` type-check. Svelte 5 runes syntax throughout. No stores for local component state — use runes.
- **Python**: Ruff formats and lints. `mypy --strict` type-checks every `priv/python/*.py` file.
- **Julia**: `JuliaFormatter` formats.
- **Lean 4**: `lake build` enforces compilation.
- **Commits**: Conventional Commits (`type(scope): description`) with atomic blocks — one coherent change per commit. PRs with mixed-concern commits get rebased or squashed before merge.

### Architecture Patterns

- **Context-first business logic**: all domain logic lives in `lib/math_inference/*/` contexts (`Inference`, `Verification`, `Sections`). Web-layer modules (`lib/math_inference_web/*`) are thin adapters that parse parameters, call context functions, and shape responses. No business logic in controllers, LiveView handlers, channels, or JSON views.
- **Dual-client transport**: LiveView and the REST API both call the same context functions. There is exactly one place where the orchestration logic lives.
- **Verifier behaviour**: every engine implements `MathInference.Verification.Behaviour` with `verify/1`, `health_check/0`, `capabilities/0`. Adding a new engine means adding one module and registering it in the supervisor — no other module needs to change.
- **Parallel dispatch with kill-on-timeout**: verifiers are called via `Task.async_stream` with bounded concurrency and `:kill_task` on timeout, so slow engines never block fast ones.
- **Cross-verifier gates rendering**: a section does not render until it reaches `:verified`, `:cross_verified`, or `:single_engine`. States `:pending`, `:verifying`, `:verification_mismatch`, `:failed` render status pills but not the mathematical content.
- **Hypertime event log**: every state change appends to an immutable per-session event log so any moment in a generation can be replayed via the scrubber UI. This also provides an audit trail for free.
- **Prompts as first-class artifacts**: all system prompts live in `priv/prompts/*.md` with frontmatter (`name`, `version`, `target_model`, `last_verified_at`) and are loaded via `MathInference.Prompts`.

### Testing Strategy

- **Elixir**: ExUnit tests colocated under `test/math_inference/` and `test/math_inference_web/`. Integration tests exercise full context flows with mocked NIM streams and mocked verifier Ports. `mix test --cover` runs in CI.
- **Svelte**: Vitest unit tests for pure rendering logic on at least `VerificationPill`, `SectionStream`, and `HypertimeScrubber`.
- **Python sidecars**: `pytest` under `priv/python/tests/`.
- **End-to-end**: Playwright in a later slice for browser-driven verification of the full pipeline.
- **Verification-first invariant testing**: every test that exercises the pipeline asserts the zero-unverified-renders guarantee holds — a section's rendered content must match its status.

### Git Workflow

- Main branch: `main`. Feature work on short-lived branches named `feat/<scope>`, `fix/<scope>`, or `chore/<scope>`.
- PRs must pass `mix ci` before merge.
- Force-push to `main` is forbidden. Force-push to personal branches is allowed only before PR is opened.
- Each OpenSpec change proposal maps to one PR; the PR description links to the proposal and tasks files.
- After merge, the OpenSpec change is archived via `openspec archive <change-name>`.

## Domain Context

The project targets **engineering mathematics** — calculus, linear algebra, ordinary differential equations, complex analysis, probability, numerical methods — at roughly the level of Erwin Kreyszig's *Advanced Engineering Mathematics*, and **contest mathematics** at the Putnam and AIME levels. The `math-chat/PROMPTS.md` prompts already encode a "TruthBattle" framing (Orchestrator, Parser, Vision, Tutor, Verifier) with a Socratic recursive-question tutor pattern — that framing is carried forward verbatim.

The target user is a single operator (the project author, a staff growth product designer and math enthusiast) who wants a personal harness for research, study, and eventual shareable content. No multi-tenant features, no billing, no auth in the initial slice. Multi-user support may arrive in a later slice if the harness proves valuable enough to share.

The design language is modelled after Railway's developer-platform aesthetic: obsidian dark backgrounds, 1 px gradient borders, monospace uppercase labels, status pills with subtle glow, outline interaction design, keyboard-first navigation, and `:focus-visible` rings everywhere.

## Important Constraints

- **Zero-cost baseline**: every dependency is OSS or has a free tier accommodating single-user development. Railway starter and NIM free tier keep production at $0 until usage grows.
- **Disk headroom**: the development machine currently has only ~2 GB free. A prerequisite task is to sync `~/Desktop` (280 GB) to Google Drive via `rclone` or the native Google Drive client before running `mix phx.new`, which needs ~10 GB working headroom for `deps/`, `_build/`, `assets/node_modules/`, and language-specific caches.
- **Reproducibility**: development must be reproducible via one command (`nix develop`). Five language runtimes (Elixir, Node, Python, Julia, Lean 4) are too many to install ad hoc.
- **Portability**: the backbone must be consumable by future native iOS (Swift) and Android (Kotlin) clients via an OpenAPI 3.1 contract. Business logic never leaks into the web layer.
- **Auditability**: every proposal, task, and requirement is tracked in `openspec/` so the project history is readable after the fact. This is explicitly why OpenSpec is adopted before any code is written.

## External Dependencies

- **NVIDIA NIM API** — `https://integrate.api.nvidia.com/v1/`; authenticated via `NIM_API_KEY`. Primary inference provider. OpenAI-compatible.
- **Wolfram Alpha** (optional, slice 2+) — authenticated via `WOLFRAM_APP_ID`. Secondary symbolic engine for cross-verification.
- **Langfuse** (optional, slice 2+) — `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`. LLM tracing and cost tracking.
- **Railway** (slice 3+) — production hosting via Nixpacks. Postgres addon for slice 2+ persistence.
- **GitHub** — `stussysenik/math-inference` is the canonical remote. Six prior repositories become archival references after the first slice lands.
- **Google Drive** — offline storage target for the 280 GB `~/Desktop` sync that unblocks Phoenix scaffolding. Managed via the native Google Drive for Desktop app or `rclone`.
