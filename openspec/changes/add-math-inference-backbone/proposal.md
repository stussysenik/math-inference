## Why

Six prior attempts at building a math inference / math explanation tool (`math-explainer-redwood`, `kimi-math-chat`, `math-chat`, `phoenix-math-stream`, `zig-elixir-math-katex`, `math_reader`) have independently converged on the same architectural shape: an NLP router fans prompts out to a parallel multi-engine dispatcher, a cross-verifier compares engine results, and only verified sections reach the user. None of them are finished, none are the single source of truth, and the best ideas are scattered across incompatible stacks (RedwoodJS, Rails, FastAPI, Phoenix, Python). This change centralizes those learnings into one auditable monolith so further work compounds instead of forking, and establishes a portable backbone that web, iOS, and Android clients can share.

## What Changes

- **NEW**: Phoenix 1.8 + LiveView + LiveSvelte monolith at `math_inference` that replaces all six prior repositories as the canonical implementation.
- **NEW**: Multi-engine symbolic verification pipeline (SymPy, Julia, GNU Octave, Lean4, optional Wolfram Alpha) with parallel dispatch via `Task.async_stream` and a cross-verifier that gates rendering.
- **NEW**: NVIDIA NIM streaming tool-call router that decides which verifier to invoke per generated section, modeled after the `zig-elixir-math-katex/lib/math_viz/pipeline/tool_dispatcher.ex` reference.
- **NEW**: Five-prompt library (`Orchestrator`, `Parser`, `Vision`, `Tutor`, `Verifier`) ported verbatim from `math-chat/PROMPTS.md` into `priv/prompts/`, accessed via a `MathInference.Prompts` module.
- **NEW**: Verified-section data model with a status lifecycle (`pending → verifying → verified | mismatch | failed`), proof attachments, and a Hypertime replay log so any moment in a generation can be scrubbed back.
- **NEW**: Dual-client transport — LiveView for the web UI and a JSON + SSE REST API documented by an OpenAPI 3.1 spec (via `OpenApiSpex`), with both clients calling the same Elixir context functions as the single source of truth.
- **NEW**: LiveSvelte frontend components (`KatexBlock`, `VerificationPill`, `SectionStream`, `PromptComposer`, `ToolCallTimeline`, `HypertimeScrubber`) wired with GSAP motion, Phosphor icons via `unplugin-icons`, Tailwind v4 + SaladUI primitives, and a Railway-aesthetic token layer with outline interaction design and `:focus-visible` rings on every interactive element.
- **NEW**: Reproducible dev environment via a Nix flake that pins Elixir 1.19 / OTP 28, Node 24, Python 3.13 + SymPy, Julia, GNU Octave, Lean4, and Postgres 16.
- **NEW**: Code quality pipeline — one `mix ci` target that runs `mix format --check-formatted`, `mix credo --strict`, `mix dialyzer`, `mix test --cover`, `eslint`, `prettier --check`, `svelte-check`, `tsc --noEmit`, `ruff check`, `ruff format --check`, `mypy --strict`, `pytest`, `JuliaFormatter`, `lake build`, and `nix flake check`. Pre-commit enforcement via Lefthook.
- **NEW**: `AGENTS.md` at repo root documenting internal code conventions so future contributors (human or agent) can onboard from the file alone.
- **NEW**: Railway deployment path via Nixpacks (slice 3), local-first development until the harness is working end-to-end.

## Capabilities

### New Capabilities

- `inference`: LLM orchestration including the NVIDIA NIM streaming client, the tool-calling router that decides which verifier to invoke per section, the per-session orchestration GenServer, SSE stream parsing, and the versioned prompt library (`Orchestrator` / `Parser` / `Vision` / `Tutor` / `Verifier`).
- `verification`: Multi-engine symbolic verification — a shared `Verifier` behaviour, per-engine workers (`SymPy`, `Julia`, `Octave`, `Lean4`, `Wolfram`) running as supervised `Port`s or Oban jobs, a parallel dispatcher with per-call timeouts, and a cross-verifier that compares engine results and returns `:cross_verified | :single_engine | :verification_mismatch`.
- `sections`: The verified answer section data model — section schema, status lifecycle, proof attachment, Hypertime event log, and persistence.
- `transport`: Dual-client transport layer — LiveView channels for the web UI, JSON + SSE REST endpoints for native clients, the OpenAPI 3.1 spec generated from Elixir via `OpenApiSpex`, and the rule that business logic lives in contexts and never in the web layer.
- `frontend`: LiveSvelte component suite, GSAP motion system, Phosphor icons via `unplugin-icons`, Tailwind v4 + SaladUI + custom Railway-aesthetic token layer, outline interaction design, `:focus-visible` ring system, `prefers-reduced-motion` handling, and the Hypertime scrubber UI.
- `conventions`: Code quality gates — `AGENTS.md`, Nix flake, Lefthook pre-commit hooks, the unified `mix ci` target, per-language formatters/linters/static-analyzers, and conventional-commits + atomic-block commit policy.

### Modified Capabilities

_None — this is a greenfield project with no existing specs._

## Impact

- **New repository**: `stussysenik/math-inference`, pushed to GitHub as the canonical implementation.
- **Retired repositories**: `math-explainer-redwood`, `kimi-math-chat`, `math-chat`, `phoenix-math-stream`, `zig-elixir-math-katex`, `math_reader` become reference-only archives. None of them continue development after this change lands.
- **New runtime dependencies**: Elixir 1.19 / OTP 28, Node 24, Python 3.13 (SymPy, NumPy), Julia, GNU Octave, Lean4, Postgres 16 (slice 2+), Nix (dev-time only).
- **New hex dependencies**: `phoenix ~> 1.8`, `phoenix_live_view ~> 1.0`, `live_svelte`, `req`, `langchain` (or equivalent), `oban`, `open_api_spex`, `poolboy`, `credo`, `dialyxir`, `ex_machina` (tests).
- **New JS dependencies**: `svelte@5`, `vite`, `tailwindcss@4`, `gsap`, `unplugin-icons`, `@iconify-json/ph`, `eslint`, `prettier`, `typescript`, `svelte-check`.
- **New Python dependencies** (verifier sidecars): `sympy`, `numpy`, `ruff`, `mypy`, `pytest`.
- **External services**: NVIDIA NIM API (`NIM_API_KEY`), optional Wolfram Alpha (`WOLFRAM_APP_ID`), optional Langfuse (`LANGFUSE_*`), Railway (slice 3+) for production hosting.
- **Zero-cost baseline**: every dependency above is OSS or has a free tier that accommodates single-user development. Railway starter tier + NIM free tier keep the production deployment at $0 until usage grows.
- **Non-negotiable invariant**: LLMs are never in the verification path. Symbolic engines are the sole ground truth for mathematical correctness. No section reaches the UI without passing at least one engine; cross-verification across two engines is preferred when the router can identify a second applicable backend.
