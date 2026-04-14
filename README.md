# math-inference

A Phoenix LiveView + LiveSvelte monolith that produces mathematically verified LLM answers for engineering- and contest-level mathematics. Every rendered section is verified by at least one symbolic engine (SymPy, Julia, GNU Octave, Lean 4, Wolfram Alpha) before it reaches the UI. LLMs route, parse, and narrate; symbolic engines prove.

## Status

This repository is being bootstrapped. The initial change proposal lives under `openspec/changes/add-math-inference-backbone/` — read that first. Implementation begins once disk headroom is available and the change proposal is approved.

## The invariant

**LLMs are never in the verification path.** Symbolic engines are the sole ground truth for mathematical correctness. No section reaches the UI without passing at least one engine; cross-verification across two engines is preferred when the router can identify a second applicable backend.

## Stack

- Phoenix 1.8 + LiveView + LiveSvelte + Svelte 5
- Vite + Tailwind v4 + SaladUI + GSAP + Phosphor icons (via `unplugin-icons`)
- NVIDIA NIM for inference (OpenAI-compatible streaming)
- SymPy, Julia, GNU Octave, Lean 4, optional Wolfram Alpha for verification
- Nix flake for reproducible dev across `aarch64-darwin` and `x86_64-linux`
- Railway for deployment (via Nixpacks)

## Getting started

Installation is driven by the Nix flake. After the first slice lands:

```sh
nix develop
mix setup
mix phx.server
```

Until then, see `openspec/changes/add-math-inference-backbone/tasks.md` for the ordered scaffolding checklist.

## Prior art

This project centralizes six prior attempts into one canonical monolith:

- `stussysenik/math-explainer-redwood` — reference only (RedwoodJS + GraphQL)
- `stussysenik/kimi-math-chat` — reference only (Rails + Hotwire + Rust NIF)
- `stussysenik/math-chat` — prompts ported verbatim into `priv/prompts/`
- `stussysenik/phoenix-math-stream` — reference only (minimal LiveView NIM client)
- `stussysenik/zig-elixir-math-katex` — structural base (Phoenix 1.8 + SymPy + Wolfram + Octave)
- `stussysenik/math_reader` — dropped

See the change proposal design document for the full prior-art synthesis.

## License

TBD.
