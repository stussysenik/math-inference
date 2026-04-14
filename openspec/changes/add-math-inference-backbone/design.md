## Context

Six prior repositories owned by the project author (`math-explainer-redwood`, `kimi-math-chat`, `math-chat`, `phoenix-math-stream`, `zig-elixir-math-katex`, `math_reader`) have independently explored the same problem — producing mathematically verified LLM answers for engineering- and contest-level math — on incompatible stacks (RedwoodJS + GraphQL + Prisma, Rails + Hotwire + Rust NIFs, FastAPI + Python, Phoenix LiveView, and a minimal Python script). Survey findings (captured alongside this proposal) show that all non-trivial repos converge on the same architecture: NLP router → parallel multi-engine dispatcher → cross-verifier → verification-gated renderer. The strongest structural base is `zig-elixir-math-katex` (32 commits, Phoenix 1.8 + LiveView + SymPy Port + Wolfram + Octave engines). The strongest semantic base is `math-chat/PROMPTS.md` (five full system prompts: Orchestrator / Parser / Vision / Tutor / Verifier). The remaining four repos are reference-only.

Current state of the target directory (`/Users/s3nik/Desktop/math-inference`): empty except for the OpenSpec scaffold being authored in this change. Development host is an Apple Silicon macOS 26.3 machine. Elixir 1.19.5 / OTP 28, Nix 3.15.1 (Determinate Nix), Node 24.8, Python 3.12, Julia, `rclone`, `rsync`, and `gh` are installed. Lean4 is not yet installed and will arrive via the Nix flake. The NVIDIA NIM API key is available for authenticated use of hosted models. Deployment target is Railway; the operator prefers local-first iteration until the harness runs end-to-end.

Constraints carried from the author's instructions and the survey:
- Backend must be in Elixir; additional languages (Python, Julia, Lean4, Octave) arrive only as verification sidecars.
- Business logic must live in Phoenix contexts, not in the web layer, so that native iOS (Swift) and Android (Kotlin) clients can consume the same functions via a portable REST/SSE API.
- LLMs are never in the verification path. Symbolic engines are the sole ground truth. No section reaches the UI without passing at least one engine.
- Design language follows Railway: dark-first, outline interaction, `:focus-visible` rings, GSAP motion under 300 ms, Phosphor icons via `unplugin-icons`.
- Dev environment must be reproducible via a single `nix develop`.
- `$0` baseline: every dependency OSS or free-tier; Railway starter + NIM free tier.

## Goals / Non-Goals

**Goals:**

- Establish one canonical monolith that replaces all six prior repositories.
- Guarantee that every rendered section has been verified by at least one symbolic engine, and ideally two (cross-verification).
- Make the verification pipeline observable end-to-end: every tool call, every engine response, every latency number surfaced in the UI via a Hypertime scrubber.
- Keep the backend portable: LiveView is one client among many, and the OpenAPI spec is the contract shared with future iOS / Android / desktop clients.
- Ship a reproducible `nix develop` dev shell that installs all five language runtimes (Elixir, Node, Python, Julia, Lean4) pinned to exact versions.
- Produce work that is auditable after the fact — every proposal, every task, every requirement tracked in `openspec/`.

**Non-Goals:**

- No authentication, no multi-tenant access controls, no rate limiting for slice 1. The harness is a single-user research tool until scope expands.
- No iOS / Android clients are implemented in this change. Only the OpenAPI surface needed to unblock them is scaffolded.
- No fine-tuned or self-hosted models. NVIDIA NIM is the sole inference provider for slice 1.
- No Wolfram Alpha integration required for slice 1 — it is supported as an optional engine but not wired for the first vertical slice.
- No image OCR / vision input in slice 1. The `Vision` prompt is ported but not wired into an upload UI until a later slice.
- No production deployment in slice 1. Railway deployment is explicitly slice 3 scope.
- No Ecto persistence in slice 1. The section store is in-memory (`:ets` or `Agent`) until slice 2 adds Postgres.

## Decisions

### D1: Elixir / Phoenix 1.8 + LiveView as the backbone

**Decision**: Use Phoenix 1.8 + LiveView + LiveSvelte as the core stack. Deploy to Railway eventually via Nixpacks.

**Alternatives considered:**
- **Next.js 16 + AI SDK v6**: excellent LLM ecosystem, but loses BEAM fault tolerance and OTP supervision that the author values for crash-safe verifier pools. The survey shows `phoenix-math-stream` and `zig-elixir-math-katex` both already use LiveView successfully.
- **Rails 8 + Hotwire**: `kimi-math-chat` explored this; the Rust NIF integration proved brittle and Hotwire's streaming story is less elegant than LiveView for the Hypertime scrubber use case.
- **RedwoodJS + React**: `math-explainer-redwood` used this; the Redwood GraphQL layer added complexity that did not pay for itself, and React hydration is an unnecessary cost for a server-rendered math harness.
- **Loco.rs / Iced**: Rust-first options, rejected because the Elixir ecosystem for LLM tool-calling and streaming is stronger today and because the math verification sidecars (Python, Julia, Lean4) would still require a subprocess bridge regardless of the web framework.

**Rationale**: LiveView delivers the streaming UX the author wants ("as fast as RSC / Remix") without client-side JS hydration, and OTP supervisors provide crash-safe verifier pools for free. LiveSvelte covers the bidirectional interactive widgets (KaTeX editor, parameter explorer, Hypertime scrubber) where LiveView round-trips would feel laggy. The survey confirms the author has Phoenix familiarity.

### D2: Dual-client context pattern (LiveView + REST API) for portability

**Decision**: Place all business logic inside Phoenix contexts (`MathInference.Inference`, `MathInference.Verification`, etc.) and expose both a LiveView client and a `/api/v1/*` JSON+SSE REST API through `OpenApiSpex`. Both clients call the same context functions.

**Alternatives considered:**
- **LiveView only**: simplest, but forecloses native iOS / Android / desktop clients. The author explicitly requested portability.
- **GraphQL via Absinthe**: mature in Elixir, but GraphQL subscriptions over WebSocket duplicate LiveView channels, and the author does not need GraphQL's flexible queries for this use case.
- **gRPC via `grpc-elixir`**: strong typing and streaming, but gRPC adds operational complexity on Railway (HTTP/2 + tooling), and iOS / Android clients are easier to generate from OpenAPI than from `.proto`.

**Rationale**: REST + SSE is the lowest-friction portable surface. OpenAPI 3.1 spec → `openapi-generator-cli` produces Swift and Kotlin clients automatically. The same Elixir context functions back both the web and native clients, so there is one place where business logic lives.

### D3: Multi-engine verification with parallel dispatch and cross-verification

**Decision**: Implement a `MathInference.Verification.Behaviour` with per-engine modules (`SymPy`, `Julia`, `Octave`, `Lean4`, `Wolfram`). Dispatch via `Task.async_stream` with per-call timeouts and `:kill_task` on timeout. A `CrossVerifier` compares results across engines and returns `:cross_verified | :single_engine | :verification_mismatch`.

**Alternatives considered:**
- **Single-engine (SymPy only)**: simpler, but the survey shows every non-trivial prior repo used at least two engines. Single-engine verification cannot catch engine-specific bugs.
- **Sequential dispatch**: easier to reason about, but blocks on the slowest engine unnecessarily. `Task.async_stream` with `max_concurrency` matching the verifier pool size is the idiomatic OTP pattern.
- **LLM self-verification (reflection loop)**: rejected outright. LLMs are not in the verification path.

**Rationale**: Parallel dispatch + cross-verification is the pattern `zig-elixir-math-katex` already proves at small scale, and it matches the architectural lesson from all six prior repos. `Task.async_stream` gives us timeouts, kill-on-deadline, and structured concurrency in a handful of lines. The cross-verifier is the critical invariant enforcer: if two engines disagree, the section is marked `:verification_mismatch` and does not render without a user override.

### D4: Prompt library ported verbatim from `math-chat/PROMPTS.md`

**Decision**: Port the five prompts (`Orchestrator`, `Parser`, `Vision`, `Tutor`, `Verifier`) into `priv/prompts/*.md` and expose them via a `MathInference.Prompts` module that loads, versions, and templates them.

**Alternatives considered:**
- **Write new prompts from scratch**: wastes prior iteration work. Survey confirms `math-chat/PROMPTS.md` is the most mature prompt set across all six repos.
- **Inline prompts in Elixir module attributes**: works for simple cases but makes prompt iteration harder (no syntax highlighting, no version control granularity).

**Rationale**: `priv/prompts/*.md` keeps prompts as first-class artifacts that can be diffed, reviewed, and versioned independently of code. A thin `MathInference.Prompts` wrapper provides ergonomic loading (`Prompts.get(:orchestrator, version: 2)`).

### D5: LiveSvelte from day one, not deferred

**Decision**: Install LiveSvelte and Svelte 5 in slice 1. Ship at least `KatexBlock`, `VerificationPill`, `SectionStream`, and `PromptComposer` as Svelte components. Route bidirectional state through LiveSvelte props.

**Alternatives considered:**
- **Pure Heex in slice 1, LiveSvelte in slice 2**: faster to ship slice 1, but forces a second refactor when we add interactive widgets. The author explicitly asked to "go to the spicy stuff immediately."
- **React via LiveReact**: LiveReact exists but the Svelte 5 runes story is cleaner for the kind of fine-grained reactivity the Hypertime scrubber needs.

**Rationale**: The Hypertime scrubber, the parameter explorer, and the KaTeX editor all benefit from 60 fps client-side reactivity. Svelte 5 runes + LiveSvelte props is the lightest-weight way to get there without React's hydration tax.

### D6: GSAP for motion, Phosphor for icons (via unplugin-icons)

**Decision**: Use GSAP (now free under the GSAP-Standard license post-Webflow acquisition) for motion including ScrollTrigger and Flip. Use `unplugin-icons` + `@iconify-json/ph` for Phosphor icons, giving zero-runtime SVG imports and access to the full Iconify catalog if another set is ever needed.

**Alternatives considered:**
- **Motion One (4kb)**: smaller, cheaper, but lacks ScrollTrigger and Flip, which are load-bearing for the Hypertime scrubber and section reordering.
- **Framer Motion**: React-only; not applicable inside Svelte.
- **Lucide icons**: cleaner stroke weights than some alternatives but only one weight; Phosphor offers six weights (thin/light/regular/bold/fill/duotone) which matters for signaling verification state.

**Rationale**: GSAP's ScrollTrigger + Flip are exactly the primitives the Hypertime scrubber needs. `unplugin-icons` compiles icons to inline SVG at build time, so bundle cost is literally zero for unused icons.

### D7: Nix flake for reproducibility

**Decision**: Ship a `flake.nix` in slice 1 that pins Elixir 1.19 / OTP 28, Node 24, Python 3.13 + SymPy + NumPy, Julia, GNU Octave, Lean4, and Postgres 16. Use `flake-utils.lib.eachDefaultSystem` to target `aarch64-darwin` (local) and `x86_64-linux` (Railway build).

**Alternatives considered:**
- **asdf / mise**: lighter, but cannot pin system libraries (Lean4's dependencies, Octave's BLAS) and cannot provide reproducible builds across macOS and Linux.
- **Docker Compose**: works for services but adds runtime overhead for interactive development on macOS.

**Rationale**: With five language runtimes, onboarding without Nix is "good luck installing all of this correctly." One `nix develop` eliminates an entire class of environment drift bugs, and the flake outputs double as the Railway Nixpacks contract.

### D8: Railway (not Fly.io) for production

**Decision**: Deploy to Railway in slice 3 via Nixpacks auto-detection. Single-region initially.

**Alternatives considered:**
- **Fly.io**: better for BEAM multi-region clustering, but the author explicitly requested Railway.
- **Self-hosted on a VPS**: unnecessary operational burden for a solo harness.

**Rationale**: Railway is simpler, the author prefers it, the Nix flake gives us compatible build reproducibility, and single-region is acceptable for a solo research tool.

### D9: Verified-section data model with Hypertime event log

**Decision**: Every generated section is persisted with a status lifecycle (`pending → verifying → verified | mismatch | failed`), its engine verification results, and an append-only event log that records each LLM token, tool call, verifier invocation, and result. The Hypertime scrubber replays this log on the client.

**Alternatives considered:**
- **Ephemeral sections (no log)**: simpler, but loses the ability to audit, replay, or share a generation.
- **Full transcript replay without structured events**: harder to scrub at section granularity.

**Rationale**: The Hypertime scrubber is a headline UX feature for the author. Storing it as a structured event log makes it straightforward to implement and also gives us an audit trail for free — which is itself a requirement the author stated explicitly ("so that way it's audited and done in a professional matter").

## Risks / Trade-offs

**[Risk] Lean4 cold-start latency (15–30 s) blocks the UI** → Mitigation: Lean4 runs as an Oban background job, not inline. The section renders with a `:pending` proof badge that resolves to `:verified` or `:failed` when the job completes. User sees answer stream immediately; proof certifies asynchronously.

**[Risk] Multiple language runtimes inflate Docker image size for Railway** → Mitigation: Use a multi-stage Dockerfile with separate builder and runtime images. Lean4 is the heaviest single component (~1 GB); we can split it into a dedicated sidecar container if image size becomes a deploy-time blocker.

**[Risk] Cross-verifier false positives when engines disagree on notation** → Mitigation: Implement `results_match?/2` with numeric tolerance and symbolic simplification (sympify both sides before comparing). The `zig-elixir-math-katex` repo has a TODO in this exact function — we inherit and fix it.

**[Risk] NVIDIA NIM rate limiting on the free tier** → Mitigation: Cache prompts + responses by content hash, log every call via Langfuse (free tier), and surface rate-limit errors in the UI with retry-after guidance.

**[Risk] 280 GB of existing data on `~/Desktop` blocks slice 1 scaffolding** → Mitigation: this change includes a follow-up task to `rclone` Desktop contents to Google Drive before running `mix phx.new`. OpenSpec proposal work fits in the current 2 GB free headroom.

**[Risk] Prompts go stale as NIM models update** → Mitigation: `priv/prompts/*.md` files carry frontmatter with `version`, `target_model`, and `last_verified_at` fields. `MathInference.Prompts.get/2` surfaces warnings when a prompt hasn't been revalidated in N days.

**[Risk] LiveSvelte + Vite + Phoenix dev-server watcher complexity** → Mitigation: use `phx_vite` (community package) which wires Vite into Phoenix's watcher config correctly. Document the setup in `AGENTS.md`.

**[Risk] OpenAPI spec drift between Elixir contexts and native clients** → Mitigation: CI job that fails if `open_api_spex` generates a spec that differs from the committed `priv/openapi.json`. Native client generators re-run on every spec change.

## Migration Plan

Because this is a greenfield project with no users and no existing deployments, migration is limited to repository consolidation:

1. **Freeze prior repositories**. Add an archival notice to the README of each of the six prior repositories pointing to `stussysenik/math-inference` as the canonical continuation.
2. **Port prompts first**. Copy `math-chat/PROMPTS.md` into `priv/prompts/*.md` with frontmatter. Validate each prompt renders via `MathInference.Prompts` before any verifier code is written.
3. **Port the verification skeleton**. Translate `zig-elixir-math-katex/lib/math_viz/pipeline/tool_dispatcher.ex` and `cross_verifier.ex` into `MathInference.Verification.*`. Keep the same message protocol (`{:pipeline_stage, id, stage, payload}`).
4. **Port the first two engines** (SymPy and Octave). Julia and Lean4 arrive in slice 2.
5. **Retire prior repositories**. Once slice 1 is green and pushed, mark `math-explainer-redwood`, `kimi-math-chat`, `math-chat`, `phoenix-math-stream`, `zig-elixir-math-katex`, and `math_reader` as `Archived` on GitHub.

Rollback strategy: this change is additive and creates no dependency on existing infrastructure. Rollback is `rm -rf openspec/ && git checkout -- .` — the six prior repositories remain intact and operable.

## Open Questions

- **Should the cross-verifier require two engines to pass before rendering, or allow single-engine verification with a warning pill?** Leaning toward single-engine-with-warning for slice 1, promote to required-two-engines in slice 2 once more engines are wired.
- **Does the Hypertime scrubber need to persist across sessions (durable event log in Postgres) or is ephemeral in-memory sufficient for slice 1?** Leaning toward ephemeral for slice 1, durable in slice 2 alongside Ecto.
- **Are we committing to LiveSvelte long-term, or is it a stopgap until LiveView gets better client-side reactivity primitives?** Leaning toward long-term commitment; LiveSvelte is mature and actively maintained.
- **Which Langfuse features do we actually need in slice 1?** Leaning toward none — add Langfuse in slice 2 alongside persistence.
