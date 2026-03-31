# The Tactical Edge Implementation Plan

This plan is derived from [SPECS.MD](/Users/giwin/Documents/cloudflareApp/cf_ai_footy/MainIdeas/SPECS.MD) and breaks the product into phased implementation work across every major system.

## Planning Principles

- Build the smallest usable live match experience first.
- Keep the architecture aligned with Cloudflare-native services from the beginning.
- Use mock data before committing to a paid sports feed.
- Ground all AI output in structured state from Durable Objects.
- Add reliability and observability early enough that live match debugging is practical.

## Recommended Stack

- Frontend: `Cloudflare Pages`, `React`, `TypeScript`, `Vite`
- API and orchestration: `Cloudflare Workers`, `TypeScript`
- Stateful match memory: `Durable Objects`
- Event pipelines: `Cloudflare Workflows`
- LLM inference: `Workers AI` with `Llama 3.3`
- Speech input: `Cloudflare AI Speech-to-Text`
- Realtime delivery: `WebSockets` or Cloudflare Realtime-compatible push approach
- Validation and schemas: `Zod`
- Testing: `Vitest`, Worker/Durable Object integration tests, lightweight UI tests
- Deployment and config: `Wrangler`, environment bindings, Cloudflare dashboard
- Monitoring: Cloudflare logs/analytics plus app-side structured logging

## System Breakdown

The product is divided into these implementation tracks:

1. Foundation and project setup
2. Live data ingestion and normalization
3. Match state and rolling memory
4. Event detection and workflows
5. Derived metrics and prediction engine
6. LLM prompt assembly and tactical analysis
7. Voice and chat user interaction
8. Frontend live dashboard
9. Realtime updates
10. Reliability, observability, and security
11. Deployment and scale readiness

## Phase 0: Foundation And Technical Design

### Technologies

- `TypeScript`
- `Wrangler`
- `Cloudflare Workers`
- `Cloudflare Pages`
- `Zod`
- `Vitest`
- GitHub or equivalent source control workflow

### What Will Be Done

- Define the monorepo or repo structure for frontend, worker logic, shared schemas, and docs.
- Create initial TypeScript configuration and common linting/formatting setup.
- Define shared domain models for matches, events, metrics, LLM inputs, and API responses.
- Create environment variable and binding contracts for Workers AI, Durable Objects, speech models, and sports feed credentials.
- Decide how local development will simulate live matches and event streams.
- Create a first architecture decision log for:
  - repo layout
  - mock feed strategy
  - realtime transport choice
  - API naming conventions

### Deliverables

- working Cloudflare project scaffold
- shared types and schema package
- local dev boot instructions
- initial docs for architecture and environments

## Phase 1: Mock Match Engine And Core Backend MVP

### Technologies

- `Cloudflare Workers`
- `Durable Objects`
- `TypeScript`
- `Zod`
- `Vitest`

### What Will Be Done

- Implement a match service Worker with endpoints to create, read, and update match state.
- Build one Durable Object per match keyed by `matchId`.
- Store:
  - metadata
  - score
  - clock
  - recent events
  - recent stats window
  - derived metrics placeholders
- Implement a rolling 10-minute memory window with pruning logic.
- Add a mock event simulation endpoint to inject goals, cards, substitutions, possession swings, and xG updates.
- Normalize all inbound mock events into a single internal event schema.
- Add idempotency protection for duplicate events.

### Deliverables

- `GET /api/matches/:matchId/state`
- `POST /api/internal/matches/:matchId/update`
- mock data generator or simulator
- Durable Object storage model
- integration tests for state updates and window pruning

## Phase 2: Live Data Ingestion Layer

### Technologies

- `Cloudflare Workers`
- scheduled Worker triggers or polling loop patterns
- external sports API provider or mock provider adapter
- `Zod`

### What Will Be Done

- Build a provider adapter interface so the app can support mock data first and a live sports API later.
- Implement polling logic every 1 to 5 seconds for active matches.
- Convert raw provider payloads into normalized internal objects:
  - match metadata
  - scoreline
  - possession
  - xG
  - shots
  - final third entries
  - event feed
- Add retry logic and stale-data handling.
- Introduce rate-limit protections and polling backoff rules.
- Mark data freshness timestamps so downstream systems know whether insights are live or stale.

### Deliverables

- provider adapter abstraction
- normalized ingestion pipeline
- stale feed detection
- retry and backoff behavior
- feed contract tests against mock payloads

## Phase 3: Event Detection And Workflow Orchestration

### Technologies

- `Cloudflare Workers`
- `Cloudflare Workflows`
- `Durable Objects`
- `TypeScript`

### What Will Be Done

- Implement event detection rules for:
  - goals
  - red cards
  - substitutions
  - penalties
  - large xG swing
  - pressure trend flips
- Add event significance thresholds so the system does not recompute on every small stat tick.
- Trigger Workflows only when meaningful state changes occur.
- In each Workflow run:
  - fetch current match state
  - apply latest event package
  - recompute derived metrics
  - request updated AI insight if needed
  - publish state change to clients
- Add retry and failure logging for Workflow runs.

### Deliverables

- event detection module
- Workflow trigger rules
- Workflow execution steps
- retry-safe orchestration behavior
- tests for event significance and trigger suppression

## Phase 4: Metrics And Prediction Engine

### Technologies

- `TypeScript`
- `Durable Objects`
- optionally lightweight statistical heuristics before advanced models

### What Will Be Done

- Implement the rolling momentum score calculation using recent match data.
- Compute additional derived signals:
  - pressure trend
  - possession swing
  - xG differential
  - final third pressure
  - next goal likelihood
  - win probability heuristic
- Define tunable weights for event impacts such as goals, red cards, and substitutions.
- Create confidence labels so the frontend and LLM can communicate uncertainty honestly.
- Version the metric formulas so changes can be tracked and tested over time.

### Deliverables

- metrics engine module
- documented formulas and weights
- regression tests for known match scenarios
- confidence scoring outputs

## Phase 5: Tactical Analysis And LLM Integration

### Technologies

- `Workers AI`
- `Llama 3.3`
- `Cloudflare Workers`
- shared prompt builder utilities

### What Will Be Done

- Implement prompt assembly that combines:
  - score
  - minute
  - recent events
  - metrics
  - user question if present
- Create system instructions that keep answers tactical, concise, and grounded.
- Define separate prompt modes for:
  - automatic match insight refresh
  - user Q&A
  - event-specific interpretation such as substitutions or red cards
- Add output shaping rules so responses stay short and usable during live play.
- Build a fallback rules-based summary path if inference fails or takes too long.
- Store the latest generated insight back into match state.

### Deliverables

- prompt builder
- inference wrapper for Workers AI
- model fallback behavior
- output validation and truncation rules
- test fixtures for common soccer scenarios

## Phase 6: Text Q&A Experience

### Technologies

- `Cloudflare Pages`
- `React`
- `TypeScript`
- `Cloudflare Workers`

### What Will Be Done

- Build a chat-style question panel in the frontend.
- Implement `POST /api/matches/:matchId/question`.
- Send text questions to the Worker, fetch current match context, and return model answers.
- Show recent questions and answers tied to the current match.
- Add loading, timeout, and stale-state indicators.
- Prevent spam by rate limiting and adding basic abuse controls.

### Deliverables

- text chat UI
- question-answer API flow
- session-aware question history for the current match view
- frontend states for loading, error, and stale context

## Phase 7: Voice Input Experience

### Technologies

- `Cloudflare AI Speech-to-Text`
- browser audio APIs
- `Cloudflare Pages`
- `Cloudflare Workers`

### What Will Be Done

- Implement microphone capture in the browser.
- Upload audio clips to a transcription endpoint.
- Convert speech to text and route the transcript into the same question pipeline as chat.
- Handle football-specific terms by adding prompt-side clarification rules and known entity normalization where possible.
- Show the transcript before or alongside the answer so users can verify what was heard.
- Add graceful fallback to text input if transcription fails.

### Deliverables

- record/transcribe UI
- `POST /api/speech/transcribe`
- transcript display and correction path
- voice-specific error and fallback handling

## Phase 8: Live Match Dashboard

### Technologies

- `Cloudflare Pages`
- `React`
- `TypeScript`
- CSS or chosen UI styling layer

### What Will Be Done

- Build the live match screen with:
  - scoreline
  - clock
  - momentum card
  - tactical insight card
  - prediction card
  - event timeline
  - chat and voice panel
- Design the dashboard for second-screen use during matches.
- Make the layout responsive for phone and desktop.
- Clearly differentiate live state from stale state.
- Support empty states for pre-kickoff and disconnected feed scenarios.

### Deliverables

- MVP dashboard UI
- responsive layouts
- match insight presentation components
- stale/live visual states

## Phase 9: Realtime Delivery

### Technologies

- `WebSockets` or equivalent realtime push pattern on Cloudflare
- `Cloudflare Workers`
- `Durable Objects`

### What Will Be Done

- Choose the realtime transport that best fits Cloudflare deployment constraints.
- Push state changes to connected clients when:
  - significant events occur
  - predictions update
  - the latest insight changes
- Minimize payload size by sending changed slices of state where practical.
- Reconnect clients cleanly after transient disconnects.
- Ensure the UI can recover by refetching full state when realtime messages are missed.

### Deliverables

- live push channel
- reconnection and state resync logic
- realtime event contract
- frontend subscription hooks

## Phase 10: Reliability, Security, And Observability

### Technologies

- `Cloudflare Workflows`
- Cloudflare logging/analytics
- structured application logs
- `Wrangler`

### What Will Be Done

- Add structured logs for ingestion, event detection, Workflow runs, model calls, and client question latency.
- Add request tracing with match IDs and event IDs.
- Implement endpoint rate limits and validation guards.
- Protect secrets with Cloudflare bindings.
- Add fallback behaviors for:
  - stale sports feed
  - failed inference
  - failed transcription
  - dropped realtime connections
- Define alerts or dashboard checks for latency spikes and repeated Workflow failures.

### Deliverables

- logging standards
- security guardrails
- latency and failure monitoring plan
- graceful degradation behaviors

## Phase 11: Live Provider Integration And Production Hardening

### Technologies

- chosen sports feed provider
- `Cloudflare Workers`
- `Durable Objects`
- `Cloudflare Workflows`
- `Workers AI`

### What Will Be Done

- Replace or supplement mock data with a live provider adapter.
- Validate provider field coverage for:
  - xG
  - possession
  - event timing
  - substitutions
  - pressure proxies
- Tune polling intervals and thresholds against real match behavior.
- Compare model outputs with real match sequences to reduce hallucination and overreaction.
- Load test concurrent live matches and connected users.
- Harden timeout budgets and fallback behavior for matchday traffic.

### Deliverables

- production provider integration
- tuned thresholds and metric weights
- load test results
- production readiness checklist

## Phase 12: Post-MVP Enhancements

### Technologies

- same core stack with optional analytics or personalization services

### What Will Be Done

- add multiple simultaneous match views
- add favorite club or notification features
- add pre-match and post-match summaries
- improve prediction models with historical calibration
- support richer tactical visualizations if the data provider allows them

### Deliverables

- prioritized enhancement backlog
- calibration and experimentation roadmap

## Workstream View By Product Area

### Frontend

- build dashboard shell
- build insight, prediction, and event timeline components
- build chat panel
- build voice input controls
- add realtime subscription and stale-state handling

### API Layer

- match state endpoints
- question endpoint
- transcription endpoint
- internal ingestion endpoints

### Durable Objects

- per-match state container
- rolling 10-minute memory
- event deduplication
- latest insight storage

### Workflows

- event-triggered recompute pipeline
- retry handling
- publish updated state after recalculation

### AI Layer

- prompt builder
- tactical summary mode
- Q&A mode
- fallback summary mode

### Data Layer

- provider adapter
- normalization
- freshness tracking
- stale-feed handling

### Metrics Layer

- momentum score
- pressure trend
- next goal likelihood
- win probability heuristic

## Suggested Build Order

1. Foundation and schemas
2. Durable Object state and mock ingestion
3. Metrics engine
4. Text Q&A with Workers AI
5. Frontend live dashboard
6. Event-triggered Workflows
7. Voice transcription
8. Realtime push
9. Production data provider
10. Hardening and scale tests

## Milestone Exit Criteria

### MVP Alpha

- one live or simulated match runs end to end
- state updates are visible in the UI
- users can ask text questions and get grounded answers

### MVP Beta

- Workflow-triggered recalculation works for significant events
- momentum and predictions update automatically
- voice input is functional

### Production Candidate

- live sports provider integrated
- stale data and failure handling verified
- latency and scale targets tested under realistic match load

## Major Risks To Resolve Early

- sports provider quality and rate limits
- accuracy and stability of derived metrics
- inference latency during peak traffic
- realtime transport behavior under many concurrent viewers
- transcription quality for player names and football terminology

## Immediate Next Steps

1. Choose the repo structure and scaffold the Cloudflare project.
2. Implement the shared schemas and Durable Object match state.
3. Build the mock match simulator so every later feature can be tested without waiting on a live feed.
4. Implement the metrics engine and question endpoint.
5. Build the first dashboard against simulated live data.
