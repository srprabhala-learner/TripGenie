# TripGenie — Product & Engineering Spec (v1, LOCKED)

> Status: **v1 scope locked**. Anything marked `[v2]` is explicitly out of scope for v1 and parked.
> Owner: PRABHALA · Build style: solo, incremental, milestone-by-milestone, each milestone demoable.

---

## 0. One-line definition

TripGenie is a production-grade agentic trip planner for India that takes a traveller's
(or group's) intent, constraints and budget, and returns a **grounded, schedule-feasible,
budget-checked, continuously re-plannable itinerary** — with real bookable options surfaced
but **never auto-booked**.

---

## 1. Locked v1 decisions

| Dimension | Decision | Notes |
|---|---|---|
| Users | Single traveller / family / group | One **trip owner** is the decision maker; other members contribute constraints only |
| Geographic scope | **India-only** trips | Abroad + visa flow is `[v2]` (see §4.9) |
| Booking | **Plan + recommend only** | Deep-links handed to human; agent never transacts |
| Data | **Real APIs** (no mocks in prod path) | Mocks allowed only behind a `FAKE_PROVIDERS=1` test flag |
| Stack | **Python + LangGraph** | FastAPI backend, Postgres + pgvector, Redis |
| UI | **Real web UI** | React/Next.js chat + itinerary canvas |
| Money | INR only, no payment processing in v1 | Budget *tracking* only, not payment orchestration |
| Language | English (v1) | Hindi/regional `[v2]` |

### Non-goals for v1
- No payment capture, no PNR issuance, no inventory holds.
- No international destinations, no visa/passport engine (design hooks only).
- No native mobile app.
- No live GPS tracking of the traveller.

---

## 2. Personas & primary jobs-to-be-done

| Persona | JTBD |
|---|---|
| Solo traveller | "I have 4 days and ₹25k, give me a spiritual trip out of Hyderabad." |
| Family trip owner | "2 adults, 2 kids (6, 11), elderly parent with limited mobility, Dec 20–27, beach." |
| Group organiser | "6 friends, split budget, 2 vegetarians, 1 wants trekking, 1 doesn't — reconcile it." |
| Road-tripper | "Bangalore → Goa via Hampi over 5 days, self-drive, suggest route + stays + food." |
| In-trip traveller | "My 6am flight is cancelled — fix day 1." |

---

## 3. End-to-end user journey (v1)

1. **Intake** — chat + structured form hybrid. Agent asks only for missing slots.
2. **Constraint reconciliation** — group members' prefs merged, conflicts surfaced to owner.
3. **Candidate generation** — destinations / themes / seasonality ranked.
4. **Itinerary synthesis** — day-by-day plan with transport, stay, activities, meals, buffers.
5. **Feasibility + guardrail pass** — time, budget, layover, mobility, closure checks.
6. **Present + iterate** — user edits ("swap day 3", "cheaper hotel"), agent re-plans locally.
7. **Handoff** — booking deep-links + printable/PDF itinerary + checklist.
8. **In-trip mode** — "what's next", disruption detection, re-plan of remaining days.
9. **Post-trip** — actual vs planned spend, preference learning written to long-term memory.

---

## 4. Functional requirements (the 12 locked use cases)

### 4.1 UC-1 — Trip planning for individual / family / group
- Inputs: origin, dates (or flexible window), party composition (adults/children/seniors),
  mobility constraints, dietary constraints, budget, pace (relaxed/packed), interests.
- Output: ranked destination shortlist with rationale, then a full day-wise itinerary.
- **Acceptance**: given a complete intake, produce a 100%-slot-filled itinerary with no
  unexplained gaps > 3h and no impossible transitions.

### 4.2 UC-2 — Season- & theme-aware package suggestion
- Themes (v1 enum): `romantic`, `spiritual`, `adventure`, `beach`, `hill-station`,
  `wildlife`, `heritage`, `food`, `family-kids`, `wellness`, `offbeat`.
- Seasonality signals: month-wise climate norms, monsoon windows, peak/off-peak pricing,
  festival calendar (Diwali, Pushkar, Hornbill, Rann Utsav…), school-holiday windows.
- **Acceptance**: every suggestion carries a *why-now* justification citing a season/festival/
  price signal; agent must **warn** when the user's dates are a known bad window
  (e.g. Ladakh in January, Kerala backwaters at monsoon peak).

### 4.3 UC-3 — Things to do in and around
- Radius-based discovery: in-city + day-trip ring (≤150 km).
- Each activity carries: duration, opening hours, weekly closures, ticket price, booking URL,
  age/mobility suitability, best time of day.
- **Acceptance**: no activity scheduled outside its opening hours or on its closed day.

### 4.4 UC-4 — Multi-state road trip + route optimisation
- Inputs: start, end (may equal start), must-visit nodes, max daily drive hours, dates.
- **Route optimisation is a deterministic tool, not LLM reasoning**: OR-Tools TSP/VRP solver
  over a real distance/duration matrix, with constraints:
  - max driving hours/day (default 6, hard cap 8)
  - no driving between 22:00–06:00 by default (night-drive safety guardrail)
  - overnight node must have a viable stay
  - mandatory rest stop every ~2.5h
- Output: ordered waypoints, per-leg distance/duration, fuel/toll estimate, stay + food stops.
- **Acceptance**: solver output is schedule-feasible and the LLM never reorders it silently.

### 4.5 UC-5 — Self-drive car search
- Surface self-drive availability (Zoomcar/Revv/MyChoize-class) for the pickup city/date.
- Must return: vehicle class, transmission, fuel policy, km cap + excess rate,
  security deposit, one-way availability, estimated all-in cost.
- **Guardrail**: flag km-cap overruns against the solver's total route distance.
- Fallback when no self-drive inventory: chauffeur-driven / outstation cab option.

### 4.6 UC-6 — Multi-modal transport + layover logic
- Supported modes v1: flight, train, intercity bus, self-drive/cab, ferry (Goa/A&N `[v2]`).
- **Hard layover guardrails** (non-negotiable, enforced in code not prompt):

| Transition | Min buffer |
|---|---|
| Flight → Flight (domestic, separate PNR) | 150 min |
| Flight → Flight (same PNR) | 60 min |
| Train → Flight | 240 min |
| Flight → Train | 180 min |
| Train → Train (same station) | 45 min |
| Any → Any, different station/terminal in same city | + real transfer time + 45 min |
| Last leg → international departure `[v2]` | 300 min |

- Add a **disruption-risk penalty** for legs with historically poor OTP or monsoon-season
  evening departures.
- **Acceptance**: no itinerary is ever emitted violating the table above; violations are a
  blocking validation error, not a warning.

### 4.7 UC-7 — Path + schedule optimisation (primary objective)
Objective function (weighted, user-tunable):

```
maximise  Σ(experience_value) 
subject to  total_cost ≤ budget
            total_travel_time ≤ travel_time_budget
            daily_active_hours ≤ pace_cap
            all guardrails satisfied
penalise    backtracking distance, mode churn, late-night arrivals,
            >2 hotel changes per 5 days
```

### 4.8 UC-8 — Budget & spend orchestration
- Real-time running ledger across: transport, stay, activities, car+fuel+tolls, food (est),
  buffer (default 10%).
- Budget bands: `hard_cap` (never exceed) vs `target` (soft).
- Traffic-light UI per category; agent must propose **specific** trade-offs when over
  (e.g. "drop the ₹8k seaplane, add Fort Aguada sunset — saves ₹7.4k").
- Group mode: per-head share, uneven splits, "who owes what" summary.
- **Guardrail**: itinerary exceeding `hard_cap` cannot be presented as final; it must be
  presented as *over-budget* with at least 2 concrete reduction options.

### 4.9 UC-9 — Document & ID readiness
- **v1 (India domestic)**: government-photo-ID checklist, child ID for trains, passport not
  required, Inner Line Permit / Protected Area Permit checks for **Arunachal, Nagaland,
  Mizoram, Lakshadweep, parts of Sikkim, Andaman tribal areas**, national-park entry permits,
  Vaishno Devi/Sabarimala-style pilgrimage registrations.
- **`[v2]`**: passport validity (6-month rule), visa matrix per nationality/destination,
  vaccination requirements, forex/TCS rules. Schema designed in v1, engine deferred.
- **Acceptance**: any destination requiring a permit surfaces it **before** the plan is
  finalised, with lead time.

### 4.10 UC-10 — Disruption handling & re-planning
Triggers: flight delay/cancel, train cancellation/late-running, hotel overbooking, road
closure/landslide, weather alert, attraction closed, user-initiated change.

Re-plan loop (ReAct: Observe → Diagnose → Generate options → Validate → Propose → HITL):
- **Locked segments** (already booked/confirmed by the user) are immutable unless the user
  unlocks them; re-plan only mutates the downstream unlocked subgraph.
- Must produce ≥2 options with cost delta, time delta and "what you lose".
- **Latency target**: first viable re-plan option in < 20s.

### 4.11 UC-11 — Group coordination & reconciliation
- Each member submits: budget share, dietary, mobility, must-do, must-not-do, date flexibility.
- Conflict detection + resolution strategies: `intersect` (safe), `rotate` (fair over days),
  `split-day` (group splits then rejoins), `owner-decides`.
- Constraints classified `hard` (mobility, allergy, budget cap) vs `soft` (preference).
  **Hard constraints are never traded away by the optimiser.**
- Owner sees a reconciliation report: what was honoured, what was traded, for whom.

### 4.12 UC-12 — In-trip living itinerary + post-trip
- **In-trip**: "what's next" card, today view, offline-safe export, proactive nudges
  (leave-by time, closing soon, weather), one-tap "something went wrong" → re-plan.
- **Post-trip**: planned-vs-actual spend, ratings per item, preferences written to long-term
  memory ("prefers boutique over chain", "hates 5am starts"), reusable trip template.

---

## 5. Recommended additional use cases (my recommendations — decide in/out per milestone)

| # | Use case | Why it matters for "production-ready" | Suggested milestone |
|---|---|---|---|
| R1 | **Safety & advisory layer** — weather alerts (IMD), landslide/flood advisories, high-altitude AMS warnings for Ladakh/Spiti, women-solo-travel safety notes, night-drive avoidance | Highest real-world liability surface; cheap to add, huge trust win | M3 |
| R2 | **Accessibility-first planning** — wheelchair access, step counts, senior-friendly pacing, medical-facility proximity for elderly/infant travel | Directly serves the "family with elderly parent" persona already in scope | M5 |
| R3 | **Explainability / "why this plan"** — every recommendation traceable to a source + a reason; a visible decision trail | Makes evals possible and is the difference between a demo and a trusted product | M6 |
| R4 | **Cost-of-change / cancellation awareness** — refundability, free-cancel deadlines, change fees per option | The single biggest factor in real booking decisions; also feeds disruption re-plan | M5 |
| R5 | **Multi-modal *input*** — user uploads a screenshot of a ticket/PNR/hotel voucher, or a photo ("plan a trip like this place"); OCR + vision extraction into the trip graph | This is your **Multi-Modal** concept made real and genuinely useful, not bolted on | M4 |
| R6 | **Voice / WhatsApp channel for in-trip** | In-trip users are on mobile with poor connectivity; text-first chat fails here | `[v2]` |
| R7 | **Local intelligence RAG** — curated corpus of travel blogs, govt tourism pages, forum threads, seasonal reports, embedded and cited | This is where **RAG** earns its keep vs. raw LLM knowledge (which is stale on closures/prices) | M2 |
| R8 | **Sustainability / crowd signal** — overtourism warnings, carbon estimate per mode, best-time-of-day to avoid crowds | Differentiator, low cost | `[v2]` |
| R9 | **Itinerary sharing & collaborative comments** — shareable link, per-member reactions/votes | Makes group mode actually usable | M5 |
| R10 | **Deterministic price-drift monitor** — re-check quoted prices before handoff; flag staleness | Prices are the #1 source of agent "lying"; grounding guardrail | M2 |
| R11 | **Fallback/degraded mode** — if a provider API is down, degrade to cached/RAG data and **label it clearly** as estimated | Production reality: APIs fail. Silent hallucination on failure is unacceptable | M6 |
| R12 | **Trip templates / "clone & tweak"** — reuse a past or public itinerary as a starting point | Big UX accelerant, near-zero agent cost | M6 |

---

## 6. Agentic architecture

### 6.1 Agent topology (LangGraph)

```
                          ┌──────────────────┐
   user ── chat ───────▶  │  Intake Agent    │  slot filling, clarification, group merge
                          └────────┬─────────┘
                                   ▼
                          ┌──────────────────┐
                          │   Orchestrator   │  supervisor graph, state owner
                          └────────┬─────────┘
        ┌───────────────┬──────────┼───────────┬────────────────┐
        ▼               ▼          ▼           ▼                ▼
 ┌────────────┐ ┌─────────────┐ ┌────────┐ ┌──────────┐ ┌──────────────┐
 │ Destination│ │  Transport  │ │  Stay  │ │Activities│ │   Budget     │
 │  & Season  │ │  Subagent   │ │Subagent│ │ Subagent │ │   Subagent   │
 │  Subagent  │ │ (multimodal │ │        │ │          │ │  (ledger)    │
 └────────────┘ │  + layover) │ └────────┘ └──────────┘ └──────────────┘
                └─────────────┘
        ┌──────────────────────────────────────────────┐
        ▼                                              ▼
 ┌────────────────┐                          ┌────────────────────┐
 │ Route Optimizer│  (OR-Tools, deterministic)│ Validator/Guardrail│ (pure code)
 └────────────────┘                          └────────────────────┘
                                   │
                                   ▼
                          ┌──────────────────┐
                          │ Itinerary Writer │  → structured JSON + narrative
                          └────────┬─────────┘
                                   ▼
                          ┌──────────────────┐
                          │  Critic/Reflect  │  self-check, then HITL
                          └──────────────────┘
```

**Design rule:** anything that is *arithmetic, ordering, or a hard constraint* is **code**
(solver/validator). The LLM does intent understanding, candidate generation, trade-off
narration and user dialogue. Never let the LLM be the source of truth for time, distance,
price or feasibility.

### 6.2 State object (single source of truth, persisted per turn)

```python
TripState = {
  "trip_id", "owner_id", "status",                  # draft|planned|in_trip|completed
  "intake": {...},                                  # raw + normalised slots
  "members": [ {id, prefs, hard_constraints, budget_share} ],
  "reconciliation": {conflicts, resolutions, strategy},
  "candidates": {destinations, themes, rationale},
  "segments": [ {id, type, mode, from, to, start, end, price, source_ref, locked:bool} ],
  "stays": [...], "activities": [...], "car_rental": {...},
  "route_solution": {order, legs, distance_km, drive_minutes, solver_version},
  "budget": {target, hard_cap, ledger[], by_category{}, per_head{}},
  "documents": {required[], status[]},
  "guardrail_report": {checks[], violations[], severity},
  "risk": {weather[], advisories[], otp_flags[]},
  "provenance": [ {claim, source, fetched_at, ttl} ],
  "memory_refs": {short_term, long_term_user, long_term_trip},
  "replan_history": [ {trigger, diagnosis, options, chosen, at} ],
  "trace_id"
}
```

### 6.3 Memory

| Layer | Store | Contents | TTL |
|---|---|---|---|
| Working / scratchpad | in-graph state | current reasoning, tool results | turn |
| Short-term conversational | Redis | last N turns, summarised | session |
| Trip memory | Postgres | full TripState versions, replan history | forever |
| Long-term user memory | Postgres + pgvector | preferences, past trips, learned dislikes, pace, budget habits | forever, user-editable |
| Semantic/RAG memory | pgvector | curated travel corpus, embeddings + citations | refresh cadence per source |

**Rules:** long-term writes are *explicit and reviewable* (no silent profile drift);
every memory item is user-visible and deletable (DPDP Act compliance).

### 6.4 Tools / MCP surface

Expose all data access as **MCP servers** so tools are reusable and independently testable:

| MCP server | Tools | Real-data candidates (India) |
|---|---|---|
| `mcp-flights` | search, fare_calendar, otp_stats | Amadeus Self-Service, Skyscanner/Kiwi partner, Duffel |
| `mcp-rail` | train_search, availability, station_info, live_status | IRCTC partner APIs / RailwayAPI-class aggregators |
| `mcp-bus` | search, operators | RedBus partner / aggregator |
| `mcp-stay` | hotel_search, rates, policies, cancel_terms | Booking.com Demand API, Amadeus Hotels, Expedia RapidAPI |
| `mcp-places` | poi_search, opening_hours, reviews, photos | Google Places, TripAdvisor content, Foursquare |
| `mcp-geo` | distance_matrix, directions, tolls, fuel_estimate | Google Routes / Mapbox / OSRM (self-host fallback) |
| `mcp-car` | selfdrive_search, terms | Zoomcar/Revv/MyChoize (or aggregator; scrape-free) |
| `mcp-weather` | forecast, climate_norms, alerts | IMD, Open-Meteo, OpenWeather |
| `mcp-events` | festivals, local_events | Govt tourism calendars, curated dataset |
| `mcp-permits` | permit_requirements | Curated + govt sources (ILP/PAP) |
| `mcp-optimizer` | solve_route, schedule_fit | OR-Tools (internal, deterministic) |
| `mcp-rag` | retrieve, cite | Internal pgvector corpus |

Every tool returns `{data, source, fetched_at, ttl, confidence}` — provenance is mandatory.

### 6.5 RAG design
- Corpus: govt tourism portals, IMD climate pages, curated blogs/forums, permit rules,
  festival calendars, prior itineraries (anonymised).
- Chunking by section + destination metadata; hybrid BM25 + dense retrieval; reranker.
- **Citation enforced**: any narrative claim not backed by a tool result or a retrieved chunk
  is stripped by the validator before display.
- Freshness: closures/prices are **never** served from RAG — API-only, else labelled estimate.

### 6.6 Guardrails (defence in depth)

**Input**: prompt-injection screening (esp. on retrieved web/RAG content and uploaded
documents), PII minimisation, jailbreak detection, out-of-scope rejection.

**Domain (hard, code-enforced)**:
- layover minimums (§4.6 table)
- driving hours/day + no night driving
- budget `hard_cap`
- hard group constraints (allergy, mobility)
- opening-hours/closure feasibility
- permit lead time
- child/senior suitability
- km-cap vs planned distance

**Output**: no fabricated prices/times/availability; every number traceable to
`provenance[]`; no booking language ("I've booked") — only "here's where to book";
no medical/legal/visa advice beyond citing official sources; currency/timezone correctness.

**Action**: agent has **no write/transaction scope** in v1 (architecturally, not by prompt).

**Escalation/HITL**: over-budget plans, hard-constraint conflicts, permit gaps, and all
disruption re-plans require explicit human confirmation.

### 6.7 Multi-modal (model-input sense)
- **In (v1 target, M4)**: ticket/PNR/voucher image → OCR → structured segment;
  passport/ID page `[v2]`; "plan like this photo" → vision → destination candidates.
- **Out**: itinerary map render, day-timeline visual, shareable PDF.

---

## 7. Milestones (locked build order)

### M1 — Core single-destination planner
**Scope**: UC-1, UC-2, UC-3 with curated/static data.
**Concepts**: agent loop, ReAct, tool use, grounding, short-term memory, structured output.
**Exit criteria**
- End-to-end: intake → shortlist → day-wise itinerary in the web UI.
- Itinerary schema validated; no opening-hours violations.
- 10 golden intake scenarios pass manual review.

### M2 — Real data + first real API integrations + RAG
**Scope**: flights + stays + places live; RAG corpus (R7); price-drift check (R10).
**Concepts**: MCP servers, grounding under real-world messiness, caching, provenance.
**Exit criteria**
- Zero unsourced prices/times in output (validator-enforced).
- p95 plan latency < 45s; provider failure → labelled degraded mode.
- Cache hit rate and cost/plan tracked.

### M3 — Multi-city road trip + route optimisation
**Scope**: UC-4, UC-5, plus safety/advisory layer (R1).
**Concepts**: deterministic planning tool, LLM↔solver handoff, replanning primitives.
**Exit criteria**
- OR-Tools solver integrated as an MCP tool; LLM cannot reorder waypoints.
- Drive-hour/night-drive guardrails enforced with tests.
- Self-drive results with km-cap cross-check.

### M4 — Multi-modal transport + layover logic + multi-modal input
**Scope**: UC-6, UC-7, plus ticket-image ingestion (R5).
**Concepts**: multi-agent supervisor topology, hard guardrails, vision/OCR.
**Exit criteria**
- Transport & Activities subagents coordinated by orchestrator.
- 100% of generated itineraries pass the layover table; adversarial tests included.
- Uploaded ticket correctly becomes a locked segment ≥90% on the test set.

### M5 — Group reconciliation, budget, disruption handling
**Scope**: UC-8, UC-9 (India permits), UC-10, UC-11, plus R2, R4, R9.
**Concepts**: guardrails, HITL, reflection/critic, re-plan subgraph.
**Exit criteria**
- Hard constraints never violated across a group-conflict test suite.
- Budget ledger accurate to ±2% vs. sum of sourced line items.
- Disruption injection → ≥2 valid options in <20s, locked segments preserved.

### M6 — Production hardening
**Scope**: evals, observability, safety toolkit, explainability (R3), degraded mode (R11),
templates (R12), in-trip + post-trip (UC-12).
**Exit criteria**: see §8 and §9 gates.

---

## 8. Evaluation strategy

### 8.1 Eval datasets
- **Golden set**: 50 hand-built intake→expected-property cases across personas/themes/seasons.
- **Adversarial set**: impossible budgets, 1-day Ladakh, monsoon Kerala trek, conflicting
  group constraints, injection payloads in RAG/uploads, provider-down scenarios.
- **Disruption set**: 20 scripted failures with expected re-plan properties.
- **Regression set**: every production bug becomes a permanent eval case.

### 8.2 Metrics

| Category | Metric | Target (v1 GA) |
|---|---|---|
| Correctness | Schedule-feasibility violations | **0** |
| Correctness | Layover-rule violations | **0** |
| Grounding | Unsourced factual claims | **0** |
| Grounding | Price accuracy vs. live re-check | ≥95% within 5% |
| Budget | Plans exceeding `hard_cap` presented as final | **0** |
| Safety | Injection attempts causing tool misuse | **0** |
| Quality | Human rubric (relevance/coherence/feasibility, 1–5) | ≥4.2 avg |
| Group | Hard-constraint violations | **0** |
| Disruption | Valid re-plan produced | ≥95% |
| Latency | p95 full plan / p95 re-plan | <45s / <20s |
| Cost | LLM+API cost per completed plan | tracked, budgeted |

### 8.3 Method
- Deterministic assertions first (validator = eval oracle), LLM-as-judge only for subjective
  quality, with a human-labelled calibration set.
- Evals run in CI on every prompt/graph change; block merge on regression of any zero-target.
- Online: sampled production traces auto-scored nightly.

---

## 9. Observability, safety & ops

- **Tracing**: LangSmith (or OTel + Langfuse) — every node, tool call, token, cost, latency,
  keyed by `trace_id` and surfaced in the UI as "why this plan" (R3).
- **Logging**: structured, PII-redacted; provenance table queryable.
- **Cost controls**: per-request token budget, model routing (cheap model for slot filling,
  strong model for synthesis/critic), aggressive caching of provider calls.
- **Rate limits & circuit breakers** per provider; degraded mode (R11) with explicit labelling.
- **Data protection**: India DPDP Act — consent, purpose limitation, user-visible memory,
  delete-my-data, no storage of payment instruments (none handled anyway).
- **Secrets**: vault-managed; no provider key ever reaches the LLM context.
- **Rollout**: feature flags per milestone, shadow-mode for new subagents, canary.

---

## 10. Tech stack (v1)

| Layer | Choice |
|---|---|
| Agent framework | LangGraph (Python) |
| API | FastAPI, async, SSE/WebSocket streaming |
| Persistence | Postgres (+ LangGraph checkpointer), pgvector |
| Cache/queue | Redis |
| Optimiser | Google OR-Tools |
| MCP | Python MCP servers per data domain |
| Frontend | Next.js + React, chat pane + itinerary canvas + map |
| Maps | Google Maps / Mapbox |
| Observability | LangSmith / Langfuse + OpenTelemetry |
| Tests | pytest, deterministic validator suite, eval harness in CI |
| Deploy | Docker, Fly.io/Render/GCP Cloud Run |

---

## 11. Open decisions (to resolve before M2)

1. Flight data provider — Amadeus Self-Service (easy, sandbox) vs Duffel vs aggregator.
2. Rail data — which IRCTC-authorised partner (this is the hardest India-specific gap).
3. Hotel data — Booking.com Demand API (approval needed) vs Amadeus Hotels vs RapidAPI tier.
4. Self-drive — is there a legitimate partner API, or does v1 degrade to deep-links only?
5. Primary LLM + fallback model, and the cheap/strong routing split.
6. Hosted vs self-hosted routing (Google Routes cost vs OSRM ops burden).
7. Auth/identity for group members (magic link vs Google sign-in).

---

## 12. Risks

| Risk | Mitigation |
|---|---|
| India rail API access is gated/unreliable | Start with flights+road; rail behind an adapter with a manual-entry fallback |
| Provider costs blow up during dev | Aggressive caching, fixtures for tests, `FAKE_PROVIDERS=1` |
| LLM silently overrides solver output | Solver output is immutable in state; validator diffs it before render |
| Hallucinated prices/availability | Provenance-required validator; price re-check at handoff |
| Scope creep across 12 UCs | Milestone exit criteria are binding; `[v2]` list is frozen |
| Group mode complexity explodes | Hard/soft constraint classification + owner-decides fallback |

---

*End of v1 locked spec.*
