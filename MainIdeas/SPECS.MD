# The Tactical Edge

Implementation spec for a real-time AI soccer analyst built on Cloudflare.

Reference for Cloudflare Agents:
[https://developers.cloudflare.com/agents/](https://developers.cloudflare.com/agents/)

## Product Summary

The Tactical Edge is a live companion app for Premier League matches. It ingests live match statistics, tracks short-term momentum, reacts to key match events, and gives fans fast tactical analysis in natural language through chat and voice.

The core user experience is:

1. A live match dashboard updates in near real time.
2. The system detects important events like goals, red cards, and substitutions.
3. An LLM recalculates tactical interpretation and likely match outcomes.
4. Users ask follow-up questions by voice or text and receive context-aware answers grounded in the last 10 minutes of match state.

## Product Goals

- Explain what is happening tactically, not just display raw stats.
- React to major events within seconds.
- Keep answers grounded in structured signals such as xG swing, possession trend, field tilt, and recent event sequence.
- Support both passive consumption of insights and interactive Q&A.
- Run primarily on Cloudflare services for edge performance and operational simplicity.

## Non-Goals For MVP

- Full historical analytics across entire seasons.
- Coverage for every football league from day one.
- Personalized betting advice.
- Full video analysis or frame-by-frame computer vision.
- Social features such as friends, comments, or sharing.

## Primary Users

- Premier League fans following live matches.
- Fantasy football players who want quick tactical context.
- Analysts and creators who want a fast second-screen companion.

## Core User Stories

- As a fan, I want to see who has momentum right now so I can understand the flow of the match.
- As a fan, I want the app to explain the impact of a substitution, red card, or tactical switch in plain English.
- As a user, I want to ask a voice question during a live match and get a fast, relevant answer.
- As a user, I want predictions to update when the match state materially changes.
- As an operator, I want the system to degrade gracefully if the sports feed or model is delayed.

## High-Level Architecture

```text
[Live Sports Data API]
          |
          v
[Cloudflare Worker Poller]
          |
          v
[Event Detection + Normalization]
          |
          +--> [Cloudflare Workflow]
          |         |
          |         v
          |   [Metric Recalculation]
          |         |
          |         v
          +--> [Durable Object: Match State]
                    |
                    +--> [Workers AI: Llama 3.3]
                    |
                    +--> [Realtime Push / API Response]
                              |
                              v
                     [Cloudflare Pages Frontend]
                              |
                              v
                    [Voice + Chat User Interface]
```

## Cloudflare Service Mapping

- `Cloudflare Workers`: API endpoints, polling, orchestration, normalization, event detection, client responses.
- `Cloudflare Workflows`: reliable event-triggered recalculation pipeline for significant match moments.
- `Durable Objects`: per-match state container and rolling 10-minute memory.
- `Workers AI`: Llama 3.3 for tactical analysis and Q&A.
- `Cloudflare AI Speech-to-Text`: converts voice questions into text.
- `Cloudflare Pages`: frontend app hosting.
- `Realtime/WebSockets`: live client updates for event feed, momentum, and predictions.

## Match Lifecycle

### 1. Match Initialization

- Create a Durable Object instance keyed by `matchId`.
- Seed initial metadata:
  - teams
  - kickoff time
  - competition
  - lineups
  - formation if available
- Start polling or subscribe to feed updates.

### 2. Live Ingestion

- Worker fetches live data every 1 to 5 seconds.
- Raw provider payload is normalized into the app schema.
- Duplicate events are ignored using provider event IDs and timestamps.

### 3. Event Detection

Significant events include:

- goal
- red card
- substitution
- penalty awarded
- formation change if feed supports it
- sharp xG swing
- sustained pressure spike

### 4. Workflow Trigger

When a significant event is detected:

1. Persist event and fresh stats to Durable Object.
2. Trigger Workflow for recalculation.
3. Recompute derived metrics.
4. Generate updated tactical summary and predictions.
5. Push changes to connected clients.

### 5. User Question Flow

```text
User voice/text
  -> Speech-to-Text if needed
  -> Worker question endpoint
  -> Durable Object context fetch
  -> Prompt assembly
  -> Workers AI inference
  -> Answer returned to UI
```

## Functional Requirements

### 1. Tactical Analysis Layer

The LLM must:

- summarize the current tactical state of the match
- explain momentum shifts
- assess how substitutions affect midfield control, pressing shape, or defensive stability
- answer contextual questions using recent match state
- produce concise predictions such as:
  - current momentum edge
  - next goal likelihood
  - likely tactical adjustment
  - match control trend

The LLM should not:

- invent unseen events
- make claims unsupported by state
- present betting or certainty language

### 2. Live Event Processing

The backend must:

- continuously ingest match data
- detect significant state changes
- trigger recalculation only when needed
- support multiple concurrent matches
- tolerate upstream feed retries and temporary failures

### 3. Voice And Chat UX

The frontend must:

- support push-to-talk or tap-to-record voice input
- support text chat fallback
- show the latest insight, event timeline, and momentum indicator
- render streaming or fast-turn responses when possible
- work well on mobile and desktop

### 4. Match Memory

The app must maintain:

- rolling 10-minute event memory
- recent stat history
- derived metrics
- last generated insights
- last user query context when helpful

## Durable Object State Design

One Durable Object should represent one live match.

Example state shape:

```json
{
  "matchId": "pl-2026-ars-che-001",
  "competition": "Premier League",
  "clock": {
    "minute": 63,
    "stoppage": 2,
    "phase": "SECOND_HALF"
  },
  "teams": {
    "home": "Arsenal",
    "away": "Chelsea"
  },
  "score": {
    "home": 2,
    "away": 1
  },
  "recentEvents": [
    {
      "id": "evt-1882",
      "minute": 58,
      "type": "substitution",
      "team": "Chelsea",
      "summary": "Defensive midfielder replaced by attacking midfielder"
    }
  ],
  "statsWindow": {
    "possessionHomeLast10": 62,
    "possessionAwayLast10": 38,
    "xgHomeLast10": 0.74,
    "xgAwayLast10": 0.11,
    "shotsHomeLast10": 5,
    "shotsAwayLast10": 1,
    "finalThirdEntriesHomeLast10": 13,
    "finalThirdEntriesAwayLast10": 4
  },
  "derived": {
    "momentumScore": 0.72,
    "pressureTrend": "HOME_ASCENDING",
    "nextGoalLikelihood": {
      "home": 0.61,
      "away": 0.19,
      "noneSoon": 0.20
    },
    "winProbability": {
      "home": 0.68,
      "draw": 0.20,
      "away": 0.12
    }
  },
  "latestInsight": {
    "headline": "Arsenal are pinning Chelsea deeper after the midfield change.",
    "generatedAt": "2026-03-31T20:15:10Z"
  }
}
```

## Derived Metrics

### Momentum Score

Purpose:
Provide a compact signal from `-1` to `+1` representing which side currently has territorial and chance-generation control.

Suggested inputs:

- xG differential over last 10 minutes
- possession swing over last 10 minutes
- shots and shots on target in recent window
- final third entries
- touches in penalty area if available
- major event weights:
  - goal
  - red card
  - dangerous substitution pattern

Interpretation:

- `-1.0` to `-0.4`: away team strongly on top
- `-0.39` to `0.39`: balanced
- `0.4` to `1.0`: home team strongly on top

### Pressure Trend

Categorical label for narration:

- `HOME_ASCENDING`
- `AWAY_ASCENDING`
- `BALANCED`
- `CHAOTIC_AFTER_EVENT`

### Next Goal Likelihood

Probability distribution that can be surfaced in UI and used by prompts.

## Prompting Strategy

The model should receive:

- current score and match clock
- last 10 minutes of meaningful events
- recent trend metrics
- any specific user question
- a clear instruction to stay grounded in provided data

Prompting rules:

- prefer structured context over raw prose
- explicitly say when evidence is mixed
- use concise tactical language
- answer in football terms the user understands
- avoid overclaiming from weak signals

Example LLM input payload:

```json
{
  "question": "How will the substitution of a midfielder affect the defensive line?",
  "match_state": {
    "score": "2-1",
    "minute": 63,
    "recent_events": [
      "58': Chelsea replaced a defensive midfielder with an attacking midfielder",
      "61': Arsenal recorded two shots from central areas",
      "62': Arsenal possession in last 10 minutes rose to 62%"
    ],
    "derived": {
      "momentum_score": 0.72,
      "pressure_trend": "HOME_ASCENDING",
      "next_goal_likelihood_home": 0.61
    }
  }
}
```

Expected answer style:

- 2 to 5 sentences
- tactical and specific
- grounded in recent evidence
- mention uncertainty when appropriate

## API Surface

### `GET /api/matches/:matchId/state`

Returns the latest normalized match state and derived metrics.

### `POST /api/matches/:matchId/question`

Accepts:

```json
{
  "question": "How will this substitution affect midfield control?",
  "conversationId": "optional-session-id"
}
```

Returns:

```json
{
  "answer": "The change makes Chelsea more aggressive between the lines, but it also reduces protection in front of the back four.",
  "sources": [
    "momentumScore",
    "recentEvents",
    "statsWindow"
  ],
  "generatedAt": "2026-03-31T20:15:12Z"
}
```

### `POST /api/speech/transcribe`

Accepts recorded audio, returns transcription text for the question flow.

### `POST /api/internal/matches/:matchId/update`

Internal endpoint for normalized ingestion updates from the poller.

## Event Trigger Rules

Trigger the Workflow when any of the following happens:

- score changes
- red card occurs
- substitution occurs
- xG delta crosses configured threshold
- possession trend flips sharply
- pressure trend changes after a sustained window

Do not trigger a full recomputation for every minor stat refresh if no meaningful shift occurred.

## Frontend Requirements

The frontend should include:

- live score and match clock
- momentum indicator
- rolling event timeline
- current AI tactical insight card
- prediction card
- voice input control
- chat panel with recent questions and answers

UX expectations:

- first useful content should appear quickly
- event reactions should feel nearly live
- answers should be readable while second-screen viewing a match

## Reliability Requirements

- Polling latency target: 1 to 5 seconds
- Significant event reaction target: under 2 seconds
- Voice transcription target: under 1 second when possible
- End-to-end user question response target: under 2 seconds
- LLM inference target: under 500 ms when feasible at edge
- Graceful fallback when:
  - sports API is stale
  - speech transcription fails
  - LLM inference is unavailable

Fallback behavior:

- show last known valid insight
- label stale data clearly
- allow text fallback if voice fails
- return a limited rules-based summary if model inference fails

## Security And Safety

- Keep provider secrets in Cloudflare environment bindings.
- Rate limit public question endpoints.
- Validate internal event payloads before state updates.
- Sanitize user speech transcripts before prompt assembly.
- Log prompt inputs and outputs carefully without storing unnecessary personal data.
- Avoid medical, legal, or gambling-style advice framing.

## MVP Build Plan

### Phase 1

- single live match support
- mock or sandbox sports data provider
- manual event simulation endpoint
- basic Durable Object state
- text-only question flow
- simple tactical summary generation

### Phase 2

- real sports data feed
- workflow-driven recalculation
- momentum and next-goal models
- voice transcription
- live dashboard updates

### Phase 3

- multi-match support
- richer event classification
- better caching and observability
- personalization and favorite clubs if desired

## Open Questions

- Which sports data provider will be used first for development?
- Will predictions be heuristic, model-based, or hybrid in MVP?
- Is the frontend intended as a standalone Pages app or embedded widget?
- Should the app support only live matches or pre-match and post-match views later?

## Success Metrics

- Users receive updated tactical insight within seconds of major events.
- Answers remain consistent with recent match context.
- The app stays responsive during concurrent live matches.
- Users can ask natural questions about substitutions and momentum and get useful answers quickly.
