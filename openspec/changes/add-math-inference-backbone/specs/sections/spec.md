## ADDED Requirements

### Requirement: Verified section schema

The system SHALL model every streamed answer fragment as a `MathInference.Sections.Section` struct with the fields `id`, `session_id`, `index`, `latex_source`, `rendered_katex`, `status`, `engines_attempted`, `engine_results`, `proof_payload`, `cost_cents`, `latency_ms`, `created_at`, and `updated_at`.

#### Scenario: New section created on tool call

- **WHEN** the router emits a tool call for a new section
- **THEN** the system constructs a `Section` struct with `status: :pending`, the section `index` assigned from a monotonic counter per session, and the `created_at` timestamp set to the current UTC time

#### Scenario: Section update preserves history

- **WHEN** an existing section transitions from `:verifying` to `:verified`
- **THEN** the system SHALL emit an append-only Hypertime event capturing the old status, new status, and timestamp, rather than overwriting the section in place without trace

### Requirement: Section status lifecycle

The system SHALL constrain every section to the following status transitions: `:pending → :verifying → :verified | :cross_verified | :single_engine | :verification_mismatch | :failed` and `:pending → :failed` (direct failure without a verify attempt). Transitions SHALL be enforced by a `Sections.transition/2` function that rejects invalid moves.

#### Scenario: Valid forward transition

- **WHEN** `Sections.transition(section, :verifying)` is called on a section whose current status is `:pending`
- **THEN** the function returns `{:ok, updated_section}` with `status: :verifying` and appends a lifecycle event

#### Scenario: Invalid backward transition is rejected

- **WHEN** `Sections.transition(section, :pending)` is called on a section whose current status is `:verified`
- **THEN** the function returns `{:error, {:invalid_transition, :verified, :pending}}` and does not mutate the section

#### Scenario: Direct failure path

- **WHEN** the NIM stream errors out before the verifier runs and `Sections.transition(section, :failed)` is called on a `:pending` section
- **THEN** the function returns `{:ok, updated_section}` with `status: :failed` and the event log records that no verifier ran

### Requirement: Proof attachments

The system SHALL attach verifier output to a section as a `%{engine: atom, result: term, raw_output: binary, latency_ms: integer}` map under the `engine_results` field. When multiple engines verify the same section, the system SHALL collect all their results and expose them as a list ordered by engine latency.

#### Scenario: Single-engine result attached

- **WHEN** a section receives a single successful SymPy result
- **THEN** `section.engine_results` contains one map `%{engine: :sympy, result: {:ok, ...}, raw_output: "...", latency_ms: 120}`

#### Scenario: Multi-engine results attached and ordered

- **WHEN** a section receives successful results from both `:sympy` (200 ms) and `:octave` (450 ms)
- **THEN** `section.engine_results` is a list with the SymPy map first and the Octave map second, ordered by `latency_ms`

### Requirement: Hypertime event log

The system SHALL maintain a per-session append-only event log that records every significant event in chronological order: NIM token received, tool call emitted, verifier started, verifier result received, section status changed. Each event SHALL be a `%{kind: atom, section_id: binary | nil, payload: term, timestamp: DateTime.t()}` map.

#### Scenario: Token events captured

- **WHEN** a NIM token arrives during generation
- **THEN** the system appends `%{kind: :nim_token, section_id: nil, payload: %{text: token, delta_index: n}, timestamp: now}` to the session's event log

#### Scenario: Verifier events captured

- **WHEN** a verifier finishes a call
- **THEN** the system appends one `:verifier_started` event at dispatch time and one `:verifier_finished` event at completion time, both carrying the section id, engine name, and a correlation id so they can be paired

#### Scenario: Event log is append-only

- **WHEN** any caller attempts to mutate or delete an event in the log
- **THEN** the mutation SHALL be rejected — events are immutable once appended, and the only supported operations are `append/2`, `replay_from/2`, and `snapshot_at/2`

### Requirement: Hypertime replay

The system SHALL support replaying the event log up to a caller-specified timestamp via `Sections.replay_to/2`, returning the exact section state at that moment in time. This powers the Hypertime scrubber UI on the client.

#### Scenario: Replay to mid-generation

- **WHEN** `Sections.replay_to(session_id, timestamp)` is called with a timestamp halfway through a completed generation
- **THEN** the function returns the sections as they existed at that timestamp, including their in-progress statuses and partial content

#### Scenario: Replay to a time before the session started

- **WHEN** `replay_to/2` is called with a timestamp that predates the session's first event
- **THEN** the function returns `{:ok, []}` — no sections existed yet

#### Scenario: Replay to the current moment

- **WHEN** `replay_to/2` is called with `DateTime.utc_now/0`
- **THEN** the function returns the current state, which SHALL be identical to `Sections.list_for_session(session_id)`

### Requirement: In-memory section store for slice 1

The system SHALL persist sections and event logs in memory (via `:ets` or an `Agent`) for the initial slice, without Ecto or Postgres. The in-memory store SHALL expose the same functional interface that the future Ecto-backed store will use, so swapping implementations in slice 2 requires no caller changes.

#### Scenario: Section stored and retrieved

- **WHEN** `Sections.Store.put(section)` is called followed by `Sections.Store.get(section.id)`
- **THEN** the returned section is structurally equal to the one stored

#### Scenario: Store survives LiveView disconnect

- **WHEN** a LiveView process disconnects and reconnects during an active session
- **THEN** the reconnected LiveView can call `Sections.list_for_session(session_id)` and receive the same sections that were in the store before disconnect — the store lives in a separate supervised process

#### Scenario: Store does not survive application restart

- **WHEN** the Elixir application restarts (cold boot)
- **THEN** the in-memory store starts empty — this is acceptable for slice 1 and will be addressed in slice 2 when Ecto is added
