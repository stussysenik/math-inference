## ADDED Requirements

### Requirement: Verifier behaviour contract

The system SHALL define a `MathInference.Verification.Behaviour` module that every engine module MUST implement. The behaviour SHALL declare callbacks for `verify/1` (perform a single verification), `health_check/0` (report whether the engine is alive and responsive), and `capabilities/0` (return the subset of problem kinds the engine supports).

#### Scenario: Engine module implements the behaviour

- **WHEN** a new engine module is added (for example, `MathInference.Verification.SymPy`)
- **THEN** the module SHALL declare `@behaviour MathInference.Verification.Behaviour` and implement all three callbacks, otherwise compilation SHALL emit a `@behaviour` warning

#### Scenario: Health check reports status

- **WHEN** `Verification.health_check_all/0` is called
- **THEN** the system invokes `health_check/0` on every registered engine and returns a map of `%{engine_name => :ok | {:error, reason}}` reflecting each engine's current state

### Requirement: SymPy engine worker

The system SHALL run a persistent Python daemon (`priv/python/sympy_worker.py`) managed by an Elixir `Port` through `MathInference.Verification.SymPy`. The worker SHALL accept line-delimited JSON requests on stdin and emit line-delimited JSON responses on stdout.

#### Scenario: Verify an indefinite integral

- **WHEN** `SymPy.verify/1` is called with `%{kind: :integral, expression: "x^2", expected: "x^3/3 + C"}`
- **THEN** the Port sends a JSON request to the daemon, receives a JSON response indicating the expressions are equivalent up to the constant of integration, and returns `{:ok, %{status: :verified, details: _}}`

#### Scenario: Daemon crash is recovered

- **WHEN** the Python daemon process exits unexpectedly
- **THEN** the supervising GenServer SHALL detect the `:DOWN` message, restart the Port with a fresh Python process, and re-queue the interrupted request exactly once

#### Scenario: Health check pings the daemon

- **WHEN** `SymPy.health_check/0` is called
- **THEN** the Port sends a `{"op":"ping"}` JSON request, expects a `{"op":"pong"}` response within 500 ms, and returns `:ok` on success or `{:error, :unresponsive}` on timeout

### Requirement: Multi-engine parallel dispatcher

The system SHALL dispatch a single verification task to multiple engines in parallel via `Task.async_stream`, bounded by a configurable `max_concurrency` and per-call timeout, and SHALL kill any engine call that exceeds its timeout rather than blocking the dispatcher.

#### Scenario: Two engines respond within timeout

- **WHEN** a dispatcher call fans out to `[:sympy, :octave]` and both engines return successfully within the 30-second timeout
- **THEN** the dispatcher returns `{:ok, %{sympy: result_a, octave: result_b}}`

#### Scenario: One engine times out

- **WHEN** `:sympy` returns in 200 ms but `:octave` exceeds the 30-second timeout
- **THEN** the dispatcher kills the `:octave` task, returns `{:partial, %{sympy: result_a}, [:octave]}`, and emits a `verification_timeout` telemetry event tagged with `:octave`

#### Scenario: All engines fail

- **WHEN** every dispatched engine returns an error
- **THEN** the dispatcher returns `{:error, :all_engines_failed, %{sympy: err_a, octave: err_b}}` and does not crash the caller

### Requirement: Cross-verifier

The system SHALL compare results from two or more engines through `MathInference.Verification.CrossVerifier.compare/1` and return one of `:cross_verified`, `:single_engine`, or `:verification_mismatch`. Comparison SHALL normalize LaTeX strings, apply numeric tolerance for floating-point values, and attempt symbolic simplification before declaring a mismatch.

#### Scenario: Two engines agree

- **WHEN** `CrossVerifier.compare/1` is given `%{sympy: "(x + 1)^2", octave: "x^2 + 2*x + 1"}`
- **THEN** the function normalizes both expressions, determines they are mathematically equivalent, and returns `{:cross_verified, canonical_form}`

#### Scenario: Only one engine responded

- **WHEN** `CrossVerifier.compare/1` is given a map with a single engine result
- **THEN** the function returns `{:single_engine, engine_name, result}` without attempting cross-verification

#### Scenario: Engines disagree

- **WHEN** `CrossVerifier.compare/1` is given `%{sympy: "2", octave: "2.0000001"}` with default numeric tolerance of `1.0e-6`
- **THEN** the function returns `:cross_verified` because the difference is within tolerance

#### Scenario: Engines clearly disagree

- **WHEN** `CrossVerifier.compare/1` is given `%{sympy: "x + 1", octave: "x - 1"}`
- **THEN** the function returns `{:verification_mismatch, details}` and the caller SHALL mark the section as `:verification_mismatch`

### Requirement: Zero unverified renders guarantee

The system SHALL NOT render any mathematical result in the user-facing UI unless its backing section has reached `:verified` or `:cross_verified` status. Sections in `:pending`, `:verifying`, `:verification_mismatch`, or `:failed` states SHALL render a status pill that describes the state, but the mathematical result itself SHALL be masked or visually de-emphasized until verification resolves.

#### Scenario: Pending section renders a placeholder

- **WHEN** a section is in the `:verifying` state and the LiveView attempts to render it
- **THEN** the `SectionStream` component SHALL render a skeleton placeholder with a "verifying…" pill and SHALL NOT render the section's KaTeX content

#### Scenario: Verified section renders the math

- **WHEN** a section transitions to `:verified` or `:cross_verified`
- **THEN** the `SectionStream` component SHALL render the section's KaTeX content fully, replace the status pill with a green checkmark, and trigger a GSAP entrance animation

#### Scenario: Mismatched section renders a warning

- **WHEN** a section transitions to `:verification_mismatch`
- **THEN** the `SectionStream` component SHALL render the section with a red outline, a warning pill showing which engines disagreed, and a click-to-expand diff of the conflicting results

### Requirement: Engine capability catalogue

The system SHALL expose a catalogue of which problem kinds each engine supports (for example: `:integral`, `:derivative`, `:linear_algebra`, `:proof`, `:ode`). The router SHALL consult this catalogue before dispatching so it only hits engines that can handle a given problem.

#### Scenario: Catalogue returns supported kinds

- **WHEN** `Verification.Engines.capabilities(:sympy)` is called
- **THEN** the function returns a sorted list of atoms such as `[:derivative, :factor, :integral, :limit, :simplify, :solve]`

#### Scenario: Router uses the catalogue to skip incompatible engines

- **WHEN** the router receives a `:proof` kind problem and the catalogue indicates only `:lean4` supports `:proof`
- **THEN** the dispatcher fans out only to `:lean4` and does not call SymPy or Octave

### Requirement: Lean4 verification runs as a background job

The system SHALL execute Lean4 proof verification as an Oban background job rather than an inline Port call, because Lean4 cold-start and compilation latency (15–30 seconds) would otherwise block the streaming UI.

#### Scenario: Lean4 job enqueued from router

- **WHEN** the router dispatches a `:proof` verification to `:lean4`
- **THEN** the `Lean4` engine module enqueues an Oban job carrying the proof term, the expected goal, and the section id, and immediately returns `{:queued, job_id}` to the dispatcher

#### Scenario: Lean4 job completes and updates section

- **WHEN** the Oban worker finishes verifying the proof
- **THEN** the worker broadcasts `{:pipeline_stage, id, :proof_verified, result}` to the subscribed LiveView process, the section transitions from `:pending_proof` to `:verified` or `:failed`, and the UI updates the status pill accordingly
