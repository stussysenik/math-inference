## 1. Environment and Reproducibility

- [ ] 1.1 Write `flake.nix` pinning Elixir 1.19 / OTP 28, Node 24, Python 3.13 + SymPy + NumPy, Julia, GNU Octave, Lean 4, and Postgres 16 via `flake-utils.lib.eachDefaultSystem`
- [ ] 1.2 Add `.envrc` with `use flake` so `direnv` auto-enters the dev shell
- [ ] 1.3 Verify `nix flake check` passes on `aarch64-darwin`
- [ ] 1.4 Document `nix develop` as the canonical entry point in `AGENTS.md`
- [ ] 1.5 Add `.gitignore` covering `_build/`, `deps/`, `.elixir_ls/`, `assets/node_modules/`, `priv/static/assets/`, `.DS_Store`, and `.env`

## 2. Phoenix Scaffold

- [ ] 2.1 Run `mix archive.install hex phx_new --force` inside the Nix shell
- [ ] 2.2 Run `mix phx.new . --app math_inference --module MathInference --no-ecto --install` in the cleared working directory (after disk cleanup)
- [ ] 2.3 Verify `mix compile` and `mix phx.server` succeed on the fresh scaffold
- [ ] 2.4 Replace esbuild with Vite via `phx_vite`, wire `assets/vite.config.js`, update `config/dev.exs` watcher block
- [ ] 2.5 Install `live_svelte` Hex dep and `svelte@5` + `@sveltejs/vite-plugin-svelte` Node deps
- [ ] 2.6 Install Tailwind v4, configure `assets/css/app.css` with `@import "tailwindcss"` and a `@theme` block for Railway-aesthetic tokens
- [ ] 2.7 Install SaladUI primitives via the documented install path and verify one primitive renders
- [ ] 2.8 Install GSAP (`gsap`) and `unplugin-icons` + `@iconify-json/ph`, verify a Phosphor icon imports cleanly in a Svelte component

## 3. Conventions and Quality Gates

- [ ] 3.1 Write `AGENTS.md` covering Architecture, Context-First Rule, Adding a Verifier Engine, Adding a LiveSvelte Component, Adding a REST Endpoint, Commit Style, PR Checklist, Running Locally, and `mix ci` Reference
- [ ] 3.2 Configure `.credo.exs` with strict ruleset
- [ ] 3.3 Configure `.dialyzer_ignore.exs` and add `dialyxir` dep
- [ ] 3.4 Write `assets/eslint.config.js` (flat config), `assets/prettier.config.js`, and `assets/.prettierrc`
- [ ] 3.5 Configure `assets/svelte.config.js` and `assets/tsconfig.json` with strict settings
- [ ] 3.6 Write `priv/python/ruff.toml` with strict lint rules
- [ ] 3.7 Write `priv/python/mypy.ini` with `strict = true`
- [ ] 3.8 Write `lefthook.yml` with pre-commit hooks for `mix format`, `mix credo`, `prettier --check`, `ruff check`, bounded to staged files
- [ ] 3.9 Write `lib/mix/tasks/ci.ex` exposing the unified `mix ci` task
- [ ] 3.10 Verify `mix ci` passes on the empty scaffold

## 4. Prompt Library

- [ ] 4.1 Port `math-chat/PROMPTS.md` prompts into `priv/prompts/orchestrator.md`, `parser.md`, `vision.md`, `tutor.md`, `verifier.md` with frontmatter (`name`, `version`, `target_model`, `last_verified_at`)
- [ ] 4.2 Implement `MathInference.Prompts` module with `get/2` loader
- [ ] 4.3 Implement `MathInference.Prompts.render/2` with `{{variable}}` substitution and `MissingBindingError`
- [ ] 4.4 Add stale-prompt telemetry event when `last_verified_at` is older than 30 days
- [ ] 4.5 Write ExUnit tests for `Prompts.get/2` and `Prompts.render/2` covering happy path and error cases

## 5. Inference Core

- [ ] 5.1 Implement `MathInference.Inference.NimClient.stream/2` using `Req` with SSE streaming
- [ ] 5.2 Implement `MathInference.Inference.StreamParser.parse_chunk/2` as a pure function with unit tests
- [ ] 5.3 Implement `MathInference.Inference.Router` with a tool catalogue and `route/2` dispatcher
- [ ] 5.4 Implement `MathInference.Inference.Session` GenServer with PubSub broadcast on every state transition
- [ ] 5.5 Implement `MathInference.Inference.run/1` context function as the single entry point for both LiveView and REST clients
- [ ] 5.6 Wire `Session` into the application supervision tree under a `DynamicSupervisor`
- [ ] 5.7 Write integration test: happy-path session from prompt submission through one verified section using a mocked NIM stream

## 6. Verification Pipeline

- [ ] 6.1 Define `MathInference.Verification.Behaviour` with `verify/1`, `health_check/0`, `capabilities/0` callbacks
- [ ] 6.2 Write `priv/python/sympy_worker.py` — persistent stdin/stdout JSON daemon, type-hinted under mypy strict
- [ ] 6.3 Implement `MathInference.Verification.SymPy` Port wrapper with crash-recovery GenServer
- [ ] 6.4 Implement `MathInference.Verification.Dispatcher.run/2` using `Task.async_stream` with kill-on-timeout
- [ ] 6.5 Implement `MathInference.Verification.CrossVerifier.compare/1` with numeric tolerance and symbolic simplification
- [ ] 6.6 Wire Poolboy pool for the SymPy Port workers
- [ ] 6.7 Emit telemetry events (`verification_started`, `verification_finished`, `verification_timeout`, `cross_verification_mismatch`)
- [ ] 6.8 Write tests: single-engine happy path, timeout path, mismatch path, dispatcher-all-fail path
- [ ] 6.9 Expand `results_match?/2` (inherited TODO from `zig-elixir-math-katex`) with numeric tolerance + sympify-both-sides comparison
- [ ] 6.10 Scaffold placeholder modules for `Julia`, `Octave`, `Lean4`, `Wolfram` (documented as slice 2+ implementation)

## 7. Sections and Hypertime

- [ ] 7.1 Define `MathInference.Sections.Section` struct with all required fields
- [ ] 7.2 Implement `Sections.transition/2` with the status lifecycle state machine
- [ ] 7.3 Implement `MathInference.Sections.EventLog` as an append-only log stored in ETS
- [ ] 7.4 Implement `Sections.replay_to/2` that reconstructs section state at an arbitrary timestamp
- [ ] 7.5 Implement `MathInference.Sections.Store` (in-memory for slice 1) with `put/1`, `get/1`, `list_for_session/1`
- [ ] 7.6 Write tests: valid transitions, invalid transitions, replay happy path, replay to pre-session timestamp

## 8. Transport Layer

- [ ] 8.1 Define `MathInferenceWeb.Router` with scopes for `/` (LiveView) and `/api/v1` (REST)
- [ ] 8.2 Implement `MathInferenceWeb.HarnessLive` LiveView with PubSub subscription and LiveSvelte integration
- [ ] 8.3 Implement `MathInferenceWeb.InferenceController` with `POST /api/v1/inference` calling `Inference.run/1`
- [ ] 8.4 Implement `MathInferenceWeb.SessionController` with `GET /api/v1/sessions/:id` returning JSON
- [ ] 8.5 Implement SSE event stream for `GET /api/v1/sessions/:id/events` via `Plug.Conn.chunk/2`
- [ ] 8.6 Add `open_api_spex` dep, define `MathInferenceWeb.ApiSpec` and per-controller operation specs
- [ ] 8.7 Serve `/api/openapi.json` and commit `priv/openapi.json` to the repo
- [ ] 8.8 Write `mix openapi.generate` and `mix openapi.check` Mix tasks
- [ ] 8.9 Wire `mix openapi.check` into `mix ci`
- [ ] 8.10 Implement `MathInferenceWeb.SessionChannel` for mobile Phoenix Channel consumption

## 9. Frontend Components

- [ ] 9.1 Write `assets/svelte/KatexBlock.svelte` with copy-LaTeX button and KaTeX rendering
- [ ] 9.2 Write `assets/svelte/VerificationPill.svelte` with GSAP 180ms state transitions
- [ ] 9.3 Write `assets/svelte/SectionStream.svelte` with staggered entrance animations respecting `prefers-reduced-motion`
- [ ] 9.4 Write `assets/svelte/PromptComposer.svelte` with Cmd+Enter submit handling and slash-command scaffolding
- [ ] 9.5 Write `assets/svelte/ToolCallTimeline.svelte` with terminal-style trace rendering
- [ ] 9.6 Write `assets/svelte/HypertimeScrubber.svelte` with GSAP ScrollTrigger pointer-drag
- [ ] 9.7 Wire LiveSvelte props in `HarnessLive` to pass sections and hypertime events to components
- [ ] 9.8 Implement global command palette (Cmd+K) via a shared Svelte component
- [ ] 9.9 Ensure every interactive element has a `:focus-visible` ring defined in `app.css`
- [ ] 9.10 Write Vitest unit tests for the pure rendering logic in at least `VerificationPill` and `SectionStream`

## 10. First Vertical Slice Verification

- [ ] 10.1 Start `mix phx.server` inside `nix develop`
- [ ] 10.2 Submit prompt "Compute the integral of x² from 0 to 1" in the browser
- [ ] 10.3 Verify NIM stream arrives, router dispatches to SymPy, verifier returns, section transitions to `:verified`, KaTeX renders in the browser
- [ ] 10.4 Verify Hypertime scrubber replays the generation backward and forward
- [ ] 10.5 POST the same prompt to `/api/v1/inference` via `curl`, verify JSON response matches OpenAPI schema
- [ ] 10.6 GET `/api/v1/sessions/:id/events` via `curl -N`, verify SSE events stream in real time
- [ ] 10.7 Capture a screenshot of a `:verified` section rendered in the browser and attach it to the PR description

## 11. Repository Hygiene and Remote

- [ ] 11.1 `git init` and set `main` as the default branch
- [ ] 11.2 Configure `git` user if not already set (repo-local)
- [ ] 11.3 Stage `openspec/`, `flake.nix`, `.gitignore`, and `.envrc` for the first commit
- [ ] 11.4 Commit `feat(openspec): scaffold math-inference backbone proposal and specs`
- [ ] 11.5 Create private GitHub repo `stussysenik/math-inference` via `gh repo create`
- [ ] 11.6 Push `main` with upstream tracking (`git push -u origin main`)
- [ ] 11.7 Add archival notices to the six prior repositories' READMEs pointing to `stussysenik/math-inference`

## 12. Disk Cleanup Prerequisite

- [ ] 12.1 Enumerate top-level items in `~/Desktop` (280 GB) via `du -sh`
- [ ] 12.2 Mount Google Drive locally (install Google Drive for Desktop or configure `rclone config`)
- [ ] 12.3 Choose sync strategy: Google Drive for Desktop (native stream) OR `rclone copy` to a Google Drive remote
- [ ] 12.4 Dry-run `rclone copy --dry-run ~/Desktop gdrive:Desktop-Archive-2026-04-15` to verify the plan
- [ ] 12.5 Run the copy with progress, verify byte counts match on the remote before any local deletion
- [ ] 12.6 Delete local Desktop items that are confirmed synced — user approval required per item category
- [ ] 12.7 Run the safe-cleanup batch (Xcode DerivedData, iOS DeviceSupport, CoreSimulator unavailable, npm cache, pnpm store, Homebrew cleanup) with user approval
- [ ] 12.8 Verify `df -h /System/Volumes/Data` shows at least 20 GB free before proceeding to section 2

## 13. Validation Gate

- [ ] 13.1 Run `openspec validate add-math-inference-backbone --strict --no-interactive` and fix all issues
- [ ] 13.2 Run `mix ci` end-to-end after section 10 is green
- [ ] 13.3 Take a final screenshot of the fully working harness and attach it to the change archival note
- [ ] 13.4 Archive this change via `openspec archive add-math-inference-backbone` once slice 1 is merged to main
