## ADDED Requirements

### Requirement: Context-first business logic rule

The system SHALL place all business logic inside Phoenix contexts (`MathInference.Inference`, `MathInference.Verification`, `MathInference.Sections`) and SHALL NOT embed business logic inside LiveView modules, controllers, JSON views, or channel handlers. Web-layer modules SHALL be thin adapters that call context functions.

#### Scenario: LiveView and REST API share the same function

- **WHEN** both `HarnessLive` (LiveView) and `InferenceController` (REST API) need to start a new inference session
- **THEN** both SHALL call exactly the same function `MathInference.Inference.run/1` rather than duplicating the orchestration logic in each web-layer module

#### Scenario: Web-layer module contains no business logic

- **WHEN** a reviewer inspects any file under `lib/math_inference_web/`
- **THEN** no file in that directory SHALL contain domain logic (no direct verifier dispatch, no direct NIM API calls, no direct section persistence) — only parameter parsing, context calls, and response shaping

### Requirement: LiveView web client

The system SHALL provide a LiveView web client at the root route `/` implemented by `MathInferenceWeb.HarnessLive`. The LiveView SHALL subscribe to its session's PubSub topic, consume `{:pipeline_stage, id, stage, payload}` messages, and reflect them in the UI via LiveSvelte components.

#### Scenario: User submits a prompt

- **WHEN** the user types a prompt into the composer and presses Cmd+Enter
- **THEN** the LiveView calls `MathInference.Inference.run/1`, attaches the returned `session_id` to its assigns, subscribes to the session PubSub topic, and begins rendering streaming sections

#### Scenario: LiveView handles session crash gracefully

- **WHEN** the underlying session GenServer crashes and the LiveView receives `{:pipeline_stage, _, :crashed, reason}`
- **THEN** the LiveView SHALL render a "session interrupted — retry?" banner without itself crashing, and SHALL preserve the already-rendered sections

### Requirement: JSON REST API

The system SHALL expose a REST API under `/api/v1/` covering `POST /api/v1/inference` (start a session), `GET /api/v1/sessions/:id` (fetch a session's current sections), and `GET /api/v1/sessions/:id/events` (stream the session's event log as Server-Sent Events).

#### Scenario: REST client starts a session

- **WHEN** a client POSTs `{"prompt": "integrate x^2 dx"}` to `/api/v1/inference` with a valid API key header
- **THEN** the system calls `MathInference.Inference.run/1`, returns HTTP 202 Accepted with body `{"session_id": "...", "status": "streaming"}`, and begins processing the session in the background

#### Scenario: REST client fetches session state

- **WHEN** a client GETs `/api/v1/sessions/:id` for an active session
- **THEN** the response body SHALL contain the current list of sections, each with its status, `latex_source`, `engine_results`, and timestamps, serialized per the OpenAPI schema

#### Scenario: REST client streams events via SSE

- **WHEN** a client GETs `/api/v1/sessions/:id/events` with `Accept: text/event-stream`
- **THEN** the system SHALL respond with `Content-Type: text/event-stream` and emit one SSE event per Hypertime event as it is appended to the log, until the client disconnects or the session terminates

### Requirement: OpenAPI 3.1 specification

The system SHALL generate an OpenAPI 3.1 specification for every REST endpoint via `OpenApiSpex`. The spec SHALL be served at `/api/openapi.json` and `/api/openapi.yaml` and SHALL be committed to the repository at `priv/openapi.json` for offline consumption by client code generators.

#### Scenario: Spec is served at runtime

- **WHEN** a client GETs `/api/openapi.json`
- **THEN** the response is a valid OpenAPI 3.1 JSON document describing every endpoint under `/api/v1/`

#### Scenario: Spec is committed to the repository

- **WHEN** the repository is cloned fresh
- **THEN** the file `priv/openapi.json` SHALL exist and SHALL match the runtime-served spec byte-for-byte after `mix openapi.generate` runs

#### Scenario: CI fails on spec drift

- **WHEN** the CI pipeline runs `mix openapi.check`
- **THEN** CI SHALL fail if the runtime-generated spec differs from the committed `priv/openapi.json`, ensuring the committed spec is never stale

### Requirement: Native client generation targets

The system SHALL document the commands required to generate Swift and Kotlin clients from the OpenAPI spec using `openapi-generator-cli`, so that future iOS and Android applications can consume the API without hand-writing client code. The generated clients SHALL NOT be committed to this repository; they belong in their own client repositories.

#### Scenario: Swift client generation documented

- **WHEN** a contributor wants to generate a Swift client
- **THEN** `AGENTS.md` SHALL contain a documented invocation such as `openapi-generator-cli generate -i priv/openapi.json -g swift6 -o ../math-inference-ios/MathInferenceKit` with notes on which generator flags to use

#### Scenario: Kotlin client generation documented

- **WHEN** a contributor wants to generate a Kotlin client
- **THEN** `AGENTS.md` SHALL contain a documented invocation for `-g kotlin` with notes on which generator flags to use

### Requirement: Phoenix channels for realtime mobile streaming

The system SHALL expose a Phoenix Channel on the topic `session:<session_id>` that streams the same `pipeline_stage` and `hypertime_event` messages the LiveView consumes, so that native mobile clients can use `SwiftPhoenixClient` or `PhoenixKotlin` to receive low-latency updates over WebSocket.

#### Scenario: Mobile client joins a session channel

- **WHEN** a native client connects to the socket at `/socket` and joins the topic `session:<session_id>` with a valid token
- **THEN** the channel accepts the join, begins pushing `pipeline_stage` and `hypertime_event` messages, and returns the current session state as the join reply

#### Scenario: Unauthorized channel join is rejected

- **WHEN** a client joins a session channel without a valid token or for a session it does not own
- **THEN** the channel SHALL reject the join with `{:error, %{reason: "unauthorized"}}`
