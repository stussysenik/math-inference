# Vision

## Problem

Large language models have become plausible at generating mathematics and unreliable at being correct at it. They confabulate symbolic manipulations, miss sign errors, hallucinate integrals, and present wrong answers with the same confident prose as right ones. For engineering- and contest-level mathematics — calculus, linear algebra, ODEs, complex analysis, probability, numerical methods — plausibility is not good enough. The answer is either provable or worthless.

The standard workarounds fail on their own terms:

- **Chain-of-thought prompting** is still the model checking itself, and self-checking is the problem, not the solution.
- **Reflection / critique loops** are two LLMs in a circle. Same failure mode, twice.
- **Fine-tuning** raises the ceiling on plausibility without changing the fact that you cannot tell from the output whether a given step is right.

The fix is not to make the model better at checking itself. It is to take the model out of the verification path entirely.

## The invariant

**LLMs are never in the verification path. Symbolic engines are the sole ground truth for mathematical correctness.**

The LLM routes, parses, and narrates. SymPy, Julia, GNU Octave, Lean 4, and Wolfram Alpha prove. No section reaches the UI without passing at least one engine. Cross-verification across two engines is preferred when the router can identify a second applicable backend. A section that fails verification is rendered as a status pill, not as content — the reader never sees unverified mathematics.

This invariant is load-bearing. Everything else in the architecture — the streaming router, the parallel dispatcher, the cross-verifier, the verified-section lifecycle, the Hypertime event log, the dual-client transport — exists to enforce it cheaply and observably.

## Prior art and the architectural convergence

Before this project, six independent attempts explored the same problem on incompatible stacks:

| Repository | Stack | Outcome |
| --- | --- | --- |
| `math-explainer-redwood` | RedwoodJS + GraphQL + Prisma + React | Too much framework, not enough domain. |
| `kimi-math-chat` | Rails 8 + Hotwire + Rust NIF | Rust NIF integration brittle; Hotwire streaming weaker than LiveView. |
| `math-chat` | FastAPI + Python | Weakest at UX, strongest at prompt design — the five-prompt library lives here. |
| `phoenix-math-stream` | Phoenix + LiveView | Minimal NIM streaming client; no verification layer. |
| `zig-elixir-math-katex` | Phoenix 1.8 + LiveView + SymPy Port + Wolfram + Octave | Most complete structural base. 32 commits. |
| `math_reader` | Python script | Dropped. |

None of these are finished. None are the single source of truth. And yet every non-trivial one converged on the same architecture:

```
NLP router → parallel multi-engine dispatcher → cross-verifier → verification-gated renderer
```

That convergence is the signal. Six independent attempts did not accidentally arrive at the same shape. The shape is what the problem demands. The waste is that the best ideas are scattered across incompatible stacks — the strongest structural base (`zig-elixir-math-katex`) is in Phoenix, the strongest semantic base (`math-chat/PROMPTS.md`) is in Python, and neither depends on the other.

`math-inference` is the consolidation. One monolith, one context, one audit trail.

## Goals

- **Establish one canonical monolith** that replaces all six prior repositories.
- **Guarantee zero unverified renders**: every section the user sees has been proven by at least one symbolic engine, ideally two via cross-verification.
- **Make the pipeline observable end-to-end**: every LLM token, tool call, engine response, and latency number is captured in a per-session event log and surfaced through a Hypertime scrubber UI.
- **Keep the backbone portable**: LiveView is one client among many. Business logic lives in Phoenix contexts so future native iOS (Swift) and Android (Kotlin) clients consume the same functions through a JSON + SSE REST API documented by an OpenAPI 3.1 spec.
- **Ship a reproducible dev environment**: a single `nix develop` command installs and pins all five language runtimes (Elixir, Node, Python, Julia, Lean 4) so onboarding does not depend on machine drift.
- **Make the work auditable**: every proposal, task, and requirement is tracked in `openspec/` so the project history is readable after the fact. This is why OpenSpec is adopted before any code is written.

## Non-Goals

- **No authentication, no multi-tenant access controls, no rate limiting** for slice 1. The harness is a single-operator research tool until scope expands.
- **No iOS / Android clients** in this change. Only the OpenAPI surface needed to unblock them is scaffolded.
- **No fine-tuned or self-hosted models.** NVIDIA NIM is the sole inference provider for slice 1.
- **No image OCR or vision input** in slice 1. The `Vision` prompt is ported but not wired into an upload UI.
- **No production deployment** in slice 1. Railway ships in slice 3.
- **No Ecto persistence** in slice 1. The section store is in-memory (ETS / Agent) until slice 2.

## The Hypertime audit trail

"Hypertime" is a headline UX feature and an auditability primitive in one. Every state change in a generation — LLM token, tool call, engine dispatch, verification result, section transition — appends to an immutable per-session event log. The Hypertime scrubber replays this log forward and backward on the client so any moment in a generation can be re-experienced.

Storing the event log structurally (rather than as raw transcript text) has two consequences:

1. **Section-granular replay**: you can scrub to "right before the third section verified" without re-running the generation.
2. **Audit trail for free**: every claim the system made about a section's verification status is reconstructable from the log. If a section rendered as `:verified`, the log shows which engine proved it, what its output was, and how long it took.

This is not a nice-to-have. It is the mechanism by which the verification invariant becomes inspectable rather than just asserted.

## Portability through the dual-client context pattern

All business logic lives inside Phoenix contexts (`MathInference.Inference`, `MathInference.Verification`, `MathInference.Sections`). The web layer — LiveView handlers, controllers, channels, JSON views — contains zero business logic. It parses parameters, calls context functions, and shapes responses.

This is the lever that keeps the backbone portable:

- **LiveView** consumes contexts directly.
- **The REST + SSE API** consumes the same contexts.
- **An OpenAPI 3.1 spec** is generated from the Elixir controllers via `open_api_spex`.
- **Native iOS and Android clients** (slice 3+) are generated from the OpenAPI spec via `openapi-generator-cli`. They call the same contexts the web UI does.

One place where orchestration lives. Multiple clients consuming it. Backends drift apart when business logic leaks into transport; this architecture forbids the leak by construction.

## Slice plan

The work is delivered in three slices, gated by end-to-end verification at each boundary.

- **Slice 1 — Harness (current).** Disk cleanup prerequisite → Nix flake → Phoenix scaffold → Vite + LiveSvelte + Tailwind v4 + SaladUI + GSAP + Phosphor → prompt library → NIM streaming client → SymPy verifier → parallel dispatcher → cross-verifier → sections lifecycle → in-memory store → LiveView harness → REST + SSE API → OpenAPI spec → Phoenix Channels stub. First vertical slice: submit "Compute the integral of x² from 0 to 1" in the browser, verify NIM streams, SymPy verifies, section transitions to `:verified`, KaTeX renders, Hypertime replays. Same flow via `curl` for the REST path.
- **Slice 2 — Durability and more engines.** Ecto + Postgres persistence for sections and event logs. Julia, GNU Octave, and Lean 4 (via Oban) as additional verification engines. Wolfram Alpha integration as an optional engine. Cross-verifier promoted from "single engine with warning" to "require two engines".
- **Slice 3 — Shipping.** Railway deployment via Nixpacks. Langfuse tracing. Native iOS and Android clients generated from the OpenAPI spec. Multi-user support if the harness proves valuable enough to share.

Each slice is gated by its own end-to-end verification. Slice 1 is not done when the code compiles; slice 1 is done when the vertical slice passes through a real browser.

## Success criteria

- **Zero unverified renders.** Tests assert that the rendered content of a section always matches its verification status.
- **Auditability by construction.** For any generation, the Hypertime log can reconstruct why each section entered its current state.
- **One entry point for all clients.** LiveView, REST, and (future) native clients consume the same context functions. No business logic in the web layer.
- **Reproducibility.** `nix develop && mix setup && mix phx.server` works on a fresh Apple Silicon machine with nothing installed beyond Nix.
- **Portability.** The OpenAPI 3.1 spec is committed and diffed in CI so native clients can be regenerated deterministically.
- **Auditability of the work itself.** Every decision is tracked in `openspec/`. Future contributors (human or agent) can read the project history in chronological order and understand what changed and why.

## Open questions

Carried forward from the design document and resolved as slice 1 progresses:

1. Does the cross-verifier require two engines for slice 1, or allow single-engine with a warning pill? **Leaning** toward single-engine-with-warning for slice 1, promote to required-two in slice 2 when more engines are wired.
2. Is the Hypertime event log durable across sessions in slice 1, or ephemeral? **Leaning** toward ephemeral for slice 1, durable in slice 2 alongside Ecto.
3. Is LiveSvelte a long-term commitment or a stopgap? **Leaning** toward long-term — Svelte 5 runes plus LiveSvelte props is the lightest-weight path to 60 fps client-side reactivity for the Hypertime scrubber.
4. Which Langfuse features do we actually need in slice 1? **Leaning** toward none — defer to slice 2 alongside persistence.

## The short version

Six attempts. One architecture. One invariant. One monolith.

LLMs route. Engines prove. Hypertime audits.
