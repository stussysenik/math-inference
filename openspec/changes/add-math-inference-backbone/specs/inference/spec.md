## ADDED Requirements

### Requirement: NVIDIA NIM streaming client

The system SHALL provide an Elixir HTTP client (`MathInference.Inference.NimClient`) that streams completions from the NVIDIA NIM API using Server-Sent Events. The client SHALL authenticate using the `NIM_API_KEY` environment variable, SHALL target the OpenAI-compatible NIM endpoint by default, and SHALL emit each token to the calling process as an Elixir message in the form `{:nim_token, session_id, token}` as it arrives.

#### Scenario: Successful token stream

- **WHEN** a caller invokes `NimClient.stream/2` with a prompt and a session id, and the NIM API returns a normal SSE stream
- **THEN** the calling process receives one `{:nim_token, session_id, token}` message per SSE `data:` line, in order, followed by exactly one `{:nim_done, session_id}` message when the stream terminates normally

#### Scenario: Missing API key

- **WHEN** `NimClient.stream/2` is called and the `NIM_API_KEY` environment variable is not set
- **THEN** the call SHALL return `{:error, :missing_api_key}` without making a network request

#### Scenario: Upstream rate limit

- **WHEN** NIM responds with HTTP 429 Too Many Requests
- **THEN** the calling process receives `{:nim_error, session_id, {:rate_limited, retry_after_seconds}}` and no further token messages for that session

#### Scenario: Stream interrupted mid-generation

- **WHEN** the SSE connection drops after one or more tokens have been delivered
- **THEN** the calling process receives `{:nim_error, session_id, :stream_interrupted}` and the partial token buffer SHALL be preserved in the session state so it can be resumed or inspected

### Requirement: Tool-calling router

The system SHALL route each generated section to the correct symbolic verifier by emitting a structured tool call via the NIM model's OpenAI-compatible `tools` parameter. The router (`MathInference.Inference.Router`) SHALL expose a catalogue of verifier tools — one per installed engine — and SHALL treat the model's chosen tool as the section's verification target.

#### Scenario: Router picks SymPy for algebraic simplification

- **WHEN** the NIM model emits a tool call to `sympy_check` with arguments `%{"expression" => "x^2 + 2x + 1", "expected" => "(x + 1)^2"}`
- **THEN** the router dispatches the tool call to `MathInference.Verification.SymPy.verify/1` with the same arguments and attaches the result to the corresponding section

#### Scenario: Router handles unknown tool name

- **WHEN** the NIM model emits a tool call with a name not in the router's catalogue (for example, a tool the server removed in a newer revision)
- **THEN** the router SHALL return `{:error, {:unknown_tool, tool_name}}`, emit a warning telemetry event, and mark the section as `:verification_failed` without crashing the session

#### Scenario: Router returns the catalogue

- **WHEN** the inference session begins and requests the current tool catalogue
- **THEN** the router SHALL return a list of JSON-schema tool definitions covering every verifier engine that reports itself healthy at router-start time

### Requirement: Per-session orchestration GenServer

The system SHALL run one `MathInference.Inference.Session` GenServer per active inference session. The GenServer SHALL own the NIM stream, maintain the section buffer, dispatch tool calls to the router, collect verifier results, and broadcast state transitions to the subscribed LiveView process via Phoenix PubSub.

#### Scenario: Session lifecycle

- **WHEN** a LiveView process calls `Session.start_link/1` with a session id and a prompt
- **THEN** a new GenServer process is started, registered under the session id, subscribes to its own PubSub topic, begins streaming NIM tokens, and emits `{:pipeline_stage, id, :streaming, _}` messages as state changes

#### Scenario: Session crash does not take down the app

- **WHEN** the session GenServer crashes mid-generation (for example, a verifier Port dies)
- **THEN** its supervisor SHALL restart the process under a `:transient` restart strategy, the LiveView SHALL receive `{:pipeline_stage, id, :crashed, reason}`, and the user sees a "re-verifying…" pill rather than a broken page

#### Scenario: Session emits hypertime events

- **WHEN** any state transition occurs inside the session (token received, tool call emitted, verifier result received, section status changed)
- **THEN** the session SHALL append a structured event to its Hypertime event log and broadcast `{:hypertime_event, id, event}` to subscribers

### Requirement: SSE stream parser

The system SHALL parse the NIM Server-Sent Events stream into discrete tokens, tool call deltas, and stream termination markers via a dedicated parser module (`MathInference.Inference.StreamParser`). The parser SHALL be a pure function over binary input so it can be unit-tested without network IO.

#### Scenario: Parse a single data frame

- **WHEN** `StreamParser.parse_chunk/2` receives a chunk containing `data: {"choices":[{"delta":{"content":"integral"}}]}\n\n`
- **THEN** the parser returns `{:ok, [{:token, "integral"}], remaining_buffer}`

#### Scenario: Parse a tool-call delta

- **WHEN** the chunk contains a `tool_calls` delta with partial JSON arguments split across two SSE frames
- **THEN** the parser accumulates the partial arguments across frames and emits a single `{:tool_call, name, full_arguments}` event only once the arguments are fully assembled

#### Scenario: Handle the terminating [DONE] marker

- **WHEN** the chunk contains the literal `data: [DONE]\n\n`
- **THEN** the parser returns `{:ok, [:done], remaining_buffer}` and leaves no partial state in the buffer

### Requirement: Versioned prompt library

The system SHALL ship five prompts ported verbatim from `math-chat/PROMPTS.md` — `orchestrator`, `parser`, `vision`, `tutor`, `verifier` — as files under `priv/prompts/*.md`. Each prompt file SHALL carry frontmatter declaring `name`, `version`, `target_model`, and `last_verified_at`. The system SHALL expose these prompts through a `MathInference.Prompts` module.

#### Scenario: Load a prompt by name

- **WHEN** a caller invokes `MathInference.Prompts.get(:orchestrator)`
- **THEN** the module returns the current version of the orchestrator prompt with its frontmatter metadata attached

#### Scenario: Load a specific version

- **WHEN** a caller invokes `MathInference.Prompts.get(:tutor, version: 2)`
- **THEN** the module returns version 2 of the tutor prompt if it exists, otherwise `{:error, :version_not_found}`

#### Scenario: Warn on stale prompts

- **WHEN** a prompt's `last_verified_at` is older than 30 days
- **THEN** the module emits a `prompt_stale` telemetry event with the prompt name, version, and age in days, but still returns the prompt content

### Requirement: Prompt templating

The system SHALL allow prompts to include named placeholders in the form `{{variable_name}}` and SHALL substitute them at load time when callers pass a `bindings` map. Missing bindings SHALL raise a descriptive error rather than leaving the placeholder unresolved.

#### Scenario: Substitute bindings

- **WHEN** a caller invokes `MathInference.Prompts.render(:parser, bindings: %{"problem" => "integrate x^2 dx"})`
- **THEN** the returned string has every `{{problem}}` occurrence replaced with `"integrate x^2 dx"`

#### Scenario: Missing binding raises

- **WHEN** a prompt contains `{{expected}}` but the bindings map does not contain an `"expected"` key
- **THEN** `Prompts.render/2` raises `MathInference.Prompts.MissingBindingError` with the prompt name and the missing key
