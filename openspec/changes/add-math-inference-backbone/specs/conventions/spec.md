## ADDED Requirements

### Requirement: AGENTS.md at repository root

The system SHALL ship an `AGENTS.md` file at the repository root documenting the internal code conventions, the context-first architecture rule, how to add a new verifier engine, how to add a new LiveSvelte component, how to add a new REST endpoint and regenerate the OpenAPI spec, the commit style (conventional commits, atomic blocks), and the PR checklist. Future contributors — human or agent — SHALL be able to onboard from this file alone.

#### Scenario: AGENTS.md exists and is non-trivial

- **WHEN** a fresh contributor clones the repository
- **THEN** they SHALL find an `AGENTS.md` file at the root containing at minimum the nine sections: Architecture, Context-First Rule, Adding a Verifier Engine, Adding a LiveSvelte Component, Adding a REST Endpoint, Commit Style, PR Checklist, Running Locally, and `mix ci` Reference

#### Scenario: AGENTS.md reflects current conventions

- **WHEN** a convention changes (for example, the `mix ci` task adds a new subcommand)
- **THEN** `AGENTS.md` SHALL be updated in the same change set that introduces the convention change — AGENTS.md SHALL NOT drift from reality

### Requirement: Unified mix ci task

The system SHALL expose a single `mix ci` Mix task at the repository root that runs the full quality gate in one command: `mix format --check-formatted`, `mix credo --strict`, `mix dialyzer`, `mix test --cover`, `cd assets && pnpm run lint && pnpm run check && pnpm run test && cd ..`, `ruff check priv/python`, `ruff format --check priv/python`, `mypy --strict priv/python`, `pytest priv/python`, `julia --project=priv/julia -e 'using JuliaFormatter; JuliaFormatter.format("priv/julia", verbose=false) || error("format failed")'`, `lake -C priv/lean4 build`, and `nix flake check`.

#### Scenario: mix ci passes on a clean repository

- **WHEN** a contributor runs `mix ci` on a freshly cloned and formatted repository
- **THEN** the command SHALL exit with status 0

#### Scenario: mix ci fails on formatting drift

- **WHEN** a file is modified to violate `mix format` rules
- **THEN** `mix ci` SHALL exit with a non-zero status and print the diff that `mix format --check-formatted` detected

#### Scenario: mix ci fails on a single language failure

- **WHEN** any subcommand in the chain fails (for example, `eslint` reports an error)
- **THEN** `mix ci` SHALL exit with the failing subcommand's exit code without running the subsequent subcommands, unless `MIX_CI_CONTINUE=1` is set to surface all failures at once

### Requirement: Lefthook pre-commit hooks

The system SHALL use Lefthook for pre-commit enforcement. Pre-commit hooks SHALL run `mix format --check-formatted`, `mix credo`, `prettier --check` on staged files, `ruff check` on staged Python files, and the relevant formatter checks for any staged file — bounded to staged files for speed.

#### Scenario: Commit with formatting errors is blocked

- **WHEN** a contributor attempts `git commit` with a staged `.ex` file that fails `mix format --check-formatted`
- **THEN** Lefthook SHALL block the commit and report the failing file

#### Scenario: Staged-file scoping keeps hooks fast

- **WHEN** a contributor commits a single-file change
- **THEN** Lefthook SHALL run checks only on that staged file and SHALL complete in under 5 seconds on a warm machine

### Requirement: Nix flake for reproducible development

The system SHALL ship a `flake.nix` at the repository root that pins Elixir 1.19 / Erlang OTP 28, Node.js 24, Python 3.13 with SymPy and NumPy, Julia, GNU Octave, Lean 4, and Postgres 16. The flake SHALL use `flake-utils.lib.eachDefaultSystem` to target both `aarch64-darwin` and `x86_64-linux`.

#### Scenario: nix develop produces a working shell

- **WHEN** a contributor with Nix installed runs `nix develop` in the repository root
- **THEN** the resulting shell SHALL make `elixir`, `mix`, `node`, `python3`, `julia`, `octave`, `lean`, and `psql` available on the PATH at the pinned versions

#### Scenario: nix flake check passes

- **WHEN** CI runs `nix flake check`
- **THEN** the command SHALL exit with status 0, verifying that all flake outputs evaluate cleanly

#### Scenario: direnv integration works

- **WHEN** a contributor has `direnv` installed and the repository contains a `.envrc` with `use flake`
- **THEN** changing into the repository directory SHALL automatically enter the Nix dev shell without requiring a manual `nix develop` invocation

### Requirement: Per-language static analysis

The system SHALL enforce strict static analysis on every language in the monolith: `dialyzer` for Elixir (strict mode), `tsc --noEmit` + `svelte-check` for TypeScript and Svelte, `mypy --strict` for Python, `JuliaFormatter` for Julia formatting, and `lake build` for Lean 4.

#### Scenario: Dialyzer catches a type mismatch

- **WHEN** an Elixir function is annotated with `@spec foo(integer()) :: integer()` but returns a string
- **THEN** `mix dialyzer` SHALL report the mismatch and `mix ci` SHALL fail

#### Scenario: mypy catches a missing type hint

- **WHEN** a Python function in `priv/python/sympy_worker.py` is missing type hints
- **THEN** `mypy --strict` SHALL report the missing annotation and `mix ci` SHALL fail

### Requirement: Conventional commits and atomic block policy

The system SHALL require all commits on the main branch to follow the Conventional Commits specification (`<type>(<scope>): <description>`) and to represent a single atomic block of change. Pull requests that contain mixed-concern commits SHALL be rebased or squashed before merge.

#### Scenario: Commit message follows the format

- **WHEN** a contributor commits a new verifier engine with the message `feat(verification): add Julia engine worker`
- **THEN** the commit message SHALL pass conventional-commits validation in CI

#### Scenario: Atomic block rule rejects mixed commits

- **WHEN** a commit contains both a new feature in `lib/math_inference/verification/` and an unrelated formatting change in `assets/css/app.css`
- **THEN** a reviewer SHALL request the contributor split the commit into two atomic blocks before merge

### Requirement: Prettier + ESLint for JS / TS / Svelte

The system SHALL configure Prettier as the single source of truth for JS/TS/Svelte formatting and ESLint as the linter for logical errors and style violations not covered by Prettier. Configuration files SHALL be committed at `assets/.prettierrc`, `assets/prettier.config.js`, and `assets/eslint.config.js`.

#### Scenario: Prettier formats Svelte components

- **WHEN** a contributor runs `pnpm run format` in `assets/`
- **THEN** Prettier SHALL format every `.svelte`, `.ts`, `.js`, `.json`, and `.md` file in the directory according to the committed configuration

#### Scenario: ESLint catches unused imports

- **WHEN** a Svelte component imports a function it does not use
- **THEN** ESLint SHALL report the unused import and `mix ci` SHALL fail

### Requirement: Ruff + mypy for Python sidecars

The system SHALL use Ruff for Python formatting and linting and mypy (strict mode) for Python static type checking, applied to every file under `priv/python/`.

#### Scenario: Ruff formats and lints the SymPy worker

- **WHEN** a contributor runs `ruff format priv/python && ruff check priv/python`
- **THEN** the commands SHALL exit 0 on a clean repository and SHALL apply or report violations on dirty files

#### Scenario: mypy strict catches missing annotations

- **WHEN** the SymPy worker is missing a return-type annotation on a function
- **THEN** `mypy --strict priv/python` SHALL report the missing annotation with a clear error message
