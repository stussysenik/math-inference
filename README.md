# math-inference

A Phoenix LiveView + LiveSvelte monolith that produces **mathematically verified** LLM answers for engineering- and contest-level mathematics. Every rendered section is verified by at least one symbolic engine (SymPy, Julia, GNU Octave, Lean 4, Wolfram Alpha) before it reaches the UI. LLMs route, parse, and narrate; symbolic engines prove.

> **The non-negotiable invariant:** LLMs are never in the verification path. Symbolic engines are the sole ground truth for mathematical correctness. No section reaches the UI without passing at least one engine; cross-verification across two engines is preferred when the router can identify a second applicable backend.

## Status

This repository is in the bootstrap phase. The code scaffold has not yet been generated — the initial OpenSpec change proposal is the source of truth until slice 1 lands. Read the proposal first, then the tasks checklist:

- [`openspec/changes/add-math-inference-backbone/proposal.md`](./openspec/changes/add-math-inference-backbone/proposal.md) — what we're building and why
- [`openspec/changes/add-math-inference-backbone/design.md`](./openspec/changes/add-math-inference-backbone/design.md) — 9 architectural decisions with alternatives considered
- [`openspec/changes/add-math-inference-backbone/tasks.md`](./openspec/changes/add-math-inference-backbone/tasks.md) — 97-task implementation checklist
- [`openspec/project.md`](./openspec/project.md) — canonical project context for contributors and agents

## Documentation

- **[VISION.md](./VISION.md)** — why this project exists, the architectural convergence across six prior attempts, the verification invariant, the slice plan, and success criteria.
- **[TECHSTACK.md](./TECHSTACK.md)** — full stack breakdown with rationale for every choice, alternatives considered, and the lifecycle of a verified generation.
- **[openspec/](./openspec/)** — change proposals, capability specs, and project conventions.

## Stack at a glance

| Layer | Choice |
| --- | --- |
| Backend | Elixir 1.19 / OTP 28, Phoenix 1.8, LiveView, LiveSvelte |
| Frontend | Svelte 5 (runes), Vite, Tailwind v4, SaladUI, GSAP, Phosphor icons, KaTeX |
| LLM | NVIDIA NIM (OpenAI-compatible streaming) |
| Verification | SymPy (slice 1), Julia, GNU Octave, Lean 4, Wolfram Alpha (slice 2+) |
| Transport | LiveView + JSON/SSE REST + Phoenix Channels, OpenAPI 3.1 via `open_api_spex` |
| Persistence | ETS / Agent (slice 1), Ecto + Postgres 16 (slice 2+) |
| Dev environment | Nix flake pinning Elixir, Node, Python, Julia, Lean 4, Postgres |
| Pre-commit | Lefthook |
| Deployment | Railway via Nixpacks (slice 3+) |

See [TECHSTACK.md](./TECHSTACK.md) for the full breakdown.

## Getting started

Once slice 1 lands (tracked in `tasks.md`), the canonical entry point is:

```sh
nix develop
mix setup
mix phx.server
```

Until then, the repo carries only the OpenSpec scaffold. Implementation begins after the disk-headroom prerequisite (task group 12) completes.

## Prior art

This project centralizes six prior attempts into one canonical monolith:

| Repository | Role |
| --- | --- |
| [`stussysenik/math-explainer-redwood`](https://github.com/stussysenik/math-explainer-redwood) | reference only (RedwoodJS + GraphQL) |
| [`stussysenik/kimi-math-chat`](https://github.com/stussysenik/kimi-math-chat) | reference only (Rails + Hotwire + Rust NIF) |
| [`stussysenik/math-chat`](https://github.com/stussysenik/math-chat) | prompts ported verbatim into `priv/prompts/` |
| [`stussysenik/phoenix-math-stream`](https://github.com/stussysenik/phoenix-math-stream) | reference only (minimal LiveView NIM client) |
| [`stussysenik/zig-elixir-math-katex`](https://github.com/stussysenik/zig-elixir-math-katex) | structural base (Phoenix 1.8 + SymPy + Wolfram + Octave) |
| [`stussysenik/math_reader`](https://github.com/stussysenik/math_reader) | dropped |

See [VISION.md](./VISION.md#prior-art-and-the-architectural-convergence) for the full synthesis.

## Contributing

This is a single-operator research harness in the bootstrap phase. Contribution guidelines will land with `AGENTS.md` in task group 3.

## License

TBD.
