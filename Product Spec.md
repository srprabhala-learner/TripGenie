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

### Decision log (all §11 open decisions now closed)

| # | Decision | Choice | Accepted trade-off |
|---|---|---|---|
| D1 | Flight data | Amadeus Self-Service | Weak Indian LCC coverage → labelled estimates + deep-link |
| D2 | Rail data | Static GTFS / timetable only | No availability, no live status; delays are user-reported |
| D3 | Stay data | Amadeus Hotel Search | Misses India budget tier → Places/curated backfill as estimates |
| D4 | Self-drive | Deep-link only + curated rate card | No live availability; agent owns sizing + cost math |
| D5 | LLM | Anthropic Claude (Sonnet / Haiku) | Single vendor in v1; cross-provider fallback deferred |
| D6 | Routing & maps | Mapbox (Matrix / Directions / GL JS) | Optimistic rural-road times → road-class derate model |
| D7 | Auth | Google OAuth (owner) + magic link (members) | Members get no persistent profile (deliberate, DPDP) |

Consequence running through all of them: **v1 is a "sourced-or-labelled" system.** Large parts of
Indian travel inventory have no clean API, so the design commitment is not full coverage — it is
that every number is either provider-sourced with provenance or explicitly marked `estimated`,
and the budget ledger keeps `sourced_total` and `estimated_total` separate.

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

### 4.5 UC-5 — Self-drive car sizing & cost estimation (**deep-link only, LOCKED**)

**v1 does not query live rental inventory.** No self-drive operator in India
(Zoomcar / Revv / MyChoize / Avis) exposes a usable public availability API, so TripGenie
solves the part it *can* do correctly and hands availability to the user.

What the agent **does** do (deterministic, code — not LLM guessing):
1. **Vehicle sizing** from party composition + luggage + terrain:
   - seats needed (incl. child seats), boot capacity, transmission preference
   - terrain class from the solver's route: hill/ghat sections or unpaved segments force a
     higher class (hatchback → compact SUV), high-altitude routes flag 4x4/AWD suitability
2. **Rate-card cost estimate** from a curated, versioned dataset:
   `{city, vehicle_class, weekday/weekend rate, km cap/day, excess-km rate, fuel policy,
     security deposit, one-way/drop charge, typical age/licence rules}`
3. **Total drive cost model**, cross-checked against the OR-Tools route:
   `base_rental + excess_km_charge + fuel_estimate + tolls + parking + one-way fee`
   where `fuel_estimate = route_km / vehicle_kmpl × current_fuel_price(state)`
   (fuel is state-taxed in India, so price is looked up per state crossed)
4. **Hard guardrail — km-cap overrun**: if `route_km > km_cap × days`, the plan must surface
   the excess-km cost explicitly, and compare against an unlimited-km or chauffeur option.
   Silently absorbing an excess-km charge into a total is a validator violation.
5. **Eligibility flags**: minimum driver age, licence vintage, out-of-state travel permission,
   interstate road-tax/permit implications for the states the route crosses.
6. **Deep-links** to the operators serving that pickup city with the dates prefilled where the
   URL scheme allows.

Labelling & guardrails:
- Every self-drive figure is `estimated` with a `rate_card_version` + `as_of` date; it enters
  the budget ledger's `estimated_total`, never `sourced_total`.
- The agent must **never** state a specific car is available, held, or bookable.
- Rate-card staleness > 90 days ⇒ prominent staleness warning.
- Fallback offered alongside, not only on failure: chauffeur-driven / outstation cab, with the
  same cost model (usually cheaper for long one-way routes once driver bata + return fare are
  counted — the agent should say so when true).

**Acceptance**: for any road-trip plan, vehicle class is justified against party + terrain,
the km-cap check has run, and the total drive cost is itemised and labelled estimated.

**`[v2]`**: real operator/aggregator integration for live availability and firm pricing.

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
- **Rail caveat (v1)**: with timetable-only data there is no live running status, so train delays
  are **user-reported**, not detected. "My train is late" is a first-class disruption trigger.

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
| `mcp-flights` | search, fare_calendar, otp_stats | **Amadeus Self-Service (LOCKED)**; Duffel/Kiwi as `[v2]` fallback adapters |
| `mcp-rail` | train_search (schedule), station_info | **Static GTFS / published timetable dataset (LOCKED)**. No seat availability, no live status in v1 — `availability` and `live_status` exist in the adapter interface but return `UNAVAILABLE` |
| `mcp-bus` | search, operators | RedBus partner / aggregator |
| `mcp-stay` | hotel_search, rates, policies, cancel_terms | **Amadeus Hotel Search (LOCKED)** — same credentials as `mcp-flights`. Budget/independent India inventory backfilled from Google Places + curated dataset as **labelled estimates** |
| `mcp-places` | poi_search, opening_hours, reviews, photos | Google Places, TripAdvisor content, Foursquare |
| `mcp-geo` | distance_matrix, directions, tolls, fuel_estimate, road_class | **Mapbox (LOCKED)** — Matrix + Directions API. Tolls and state fuel prices from a curated dataset (Mapbox does not cover Indian tolls reliably) |
| `mcp-car` | size_vehicle, estimate_drive_cost, rate_card_lookup, operator_deeplinks | **Curated rate-card dataset + deep-links (LOCKED)** — no live availability in v1; `search_availability` returns `UNAVAILABLE`. Fuel price by state, tolls via `mcp-geo` |
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

### 6.7 Routing & geo (LOCKED: Mapbox)

| Need | Mapbox surface | Notes |
|---|---|---|
| Distance/duration matrix for the solver | Matrix API | Batched; cached per `(node-set, date-bucket)` |
| Final leg geometry, turn-by-turn, ETA | Directions API | Only for the chosen route, after the solver |
| Map rendering in the UI | GL JS | Day-wise coloured route + POI pins |
| Geocoding place → coordinates | Geocoding API | Cached aggressively; India place-name ambiguity handled by disambiguation prompt |

**Accepted weakness: Indian rural / ghat / unclassified road quality.** Mapbox durations on
non-highway Indian roads are optimistic, and some rural links are missing or mis-tagged.
Mitigations (all code-side, none rely on the LLM):

- **Speed derate by road class**: apply a conservative multiplier to Mapbox durations for
  non-highway segments (state highway / district road / unclassified / ghat), tuned against a
  ground-truth set of known Indian corridors. The derated duration — never the raw one — is
  what the drive-hour guardrail (§4.4) and the layover engine (§4.6) consume.
- **Overnight-node road-quality rule**: an overnight stop must be reachable via a segment of at
  least state-highway class; otherwise the solver rejects it as an overnight node.
- **Monsoon / ghat penalty**: routes crossing known landslide-prone ghat sections in monsoon
  months get an additional time penalty and a surfaced advisory (ties into R1).
- **Sanity bounds**: any leg implying > 80 km/h average on a non-expressway, or > 60 km/h on a
  derated rural segment, is flagged as implausible and the leg is re-derated — a plan is never
  emitted on an implausible duration.
- **No live traffic in v1** (Mapbox traffic-aware durations are weaker in India): the solver uses
  typical-time durations plus a fixed urban-congestion buffer for metro ingress/egress windows.
- **Provider-agnostic adapter**: `mcp-geo` is written so Google Routes can be swapped in for a
  specific corridor or globally (`[v2]`) if the derate model proves insufficient.
- **Cost control**: matrices are the expensive call — cache by node-set hash, cap solver node
  count (v1 limit: 25 waypoints), and never re-query the matrix during re-planning unless the
  node set actually changed.

### 6.8 Model routing (LOCKED: Anthropic Claude)

| Node / job | Model | Why |
|---|---|---|
| Intake slot-filling, clarifying questions | **Haiku** | High turn count, low reasoning depth; cheapest path |
| Intent / theme / constraint classification | **Haiku** | Deterministic-ish, schema-constrained output |
| Group constraint extraction & normalisation | **Haiku** | Structured extraction |
| Destination & theme candidate generation | **Sonnet** | Needs breadth + seasonality judgement |
| Itinerary synthesis & trade-off narration | **Sonnet** | Core quality surface |
| Critic / reflection pass | **Sonnet** | Must be at least as strong as the writer |
| Disruption diagnosis + re-plan options | **Sonnet** | Highest-stakes reasoning, <20s budget |
| Ticket/voucher vision extraction (M4) | **Sonnet (vision)** | Native multimodal; OCR pre-pass evaluated in M4 |
| LLM-as-judge in evals | **Sonnet**, separate prompt + separate API key | Judge must not share the generator's prompt |

Implementation rules:
- Models are referenced via a **`model_router` abstraction**, never hardcoded per node — swapping
  a tier or adding a fallback provider must be a config change.
- **Tool calling** uses Claude's native tool-use, bridged to the MCP servers in §6.4.
- **Structured output** enforced via tool-schema-constrained responses + Pydantic validation;
  a parse failure retries once on the same tier, then escalates a tier, then hard-fails
  (never silently degrades to free-text).
- **Prompt caching** on the static system block (guardrail rules, schemas, destination taxonomy)
  to keep Sonnet cost viable at plan scale.
- **Fallback**: on provider outage or sustained 429s, the router degrades Sonnet→Haiku for
  non-critical nodes and **blocks** synthesis/critic rather than producing an unreviewed plan;
  a cross-provider fallback is `[v2]` and the abstraction exists to allow it.
- **Determinism for evals**: temperature pinned low for synthesis, model version pinned
  explicitly (no floating aliases) so eval results are comparable across runs.

### 6.9 Multi-modal (model-input sense)
- **In (v1 target, M4)**: ticket/PNR/voucher image → OCR → structured segment;
  passport/ID page `[v2]`; "plan like this photo" → vision → destination candidates.
- **Out**: itinerary map render, day-timeline visual, shareable PDF.

### 6.10 Identity & permissions (LOCKED)

| Role | Auth | Capabilities |
|---|---|---|
| **Trip owner** | Google OAuth (real account) | Create/edit/delete trips, invite members, resolve conflicts, lock segments, accept re-plans, owns long-term memory profile |
| **Group member** | Magic link, **no account** | Submit/edit **their own** constraints and budget share, view the itinerary, comment/vote (R9) |
| **Viewer** | Share link | Read-only itinerary view |
| **Anonymous visitor** | None | Can plan a throwaway trip; must sign in to save it |

Rules:
- **One owner per trip.** Ownership is transferable but never shared — this keeps UC-11's
  `owner-decides` conflict resolution unambiguous.
- **Member magic links are per-member, scoped, and expiring** (default 14 days, re-issuable).
  A member token grants access to exactly one trip and one member record — it cannot read other
  members' budget shares or edit the itinerary.
- **Member privacy**: dietary, medical/mobility and budget-share data is visible to the owner
  and that member only. The reconciliation report (§4.11) shows the owner *what* was traded and
  for whom; it does not expose raw sensitive fields to other members.
- **Anonymous → owner upgrade**: an anonymous session's draft trip is claimed on first sign-in.
- **Long-term memory attaches to owner accounts only.** Magic-link members get no persistent
  profile — avoids building shadow profiles of people who never signed up (DPDP consent).
- **In-trip mode** (UC-12) is accessible to all members via their existing magic link, so a
  traveller on the road never hits a login wall.
- Sessions are short-lived JWTs + refresh; no passwords stored anywhere in v1.
- Auth is a thin adapter — adding email/password or phone OTP later must not touch the graph.

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
- Zero **unlabelled** prices/times in output (validator-enforced): every figure is either
  provider-sourced with provenance, or explicitly marked `estimated`.
- Stay results show sourced vs estimated split; ≥X% sourced coverage measured per metro/
  non-metro destination (baseline to be recorded in M2, not a gate).
- p95 plan latency < 45s; provider failure → labelled degraded mode.
- Cache hit rate and cost/plan tracked.

### M3 — Multi-city road trip + route optimisation
**Scope**: UC-4, UC-5, plus safety/advisory layer (R1).
**Concepts**: deterministic planning tool, LLM↔solver handoff, replanning primitives.
**Exit criteria**
- OR-Tools solver integrated as an MCP tool over the Mapbox matrix; LLM cannot reorder waypoints.
- Road-class speed derate calibrated against a ground-truth corridor set; drive-hour and
  night-drive guardrails enforced on **derated** durations, with tests.
- Implausible-duration sanity check blocks plan emission.
- Self-drive: vehicle sizing + itemised cost estimate + km-cap overrun check, all labelled
  `estimated`; rate card versioned and seeded for the top pickup cities.
- Chauffeur/outstation-cab comparison presented whenever it beats self-drive on total cost.

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
| Correctness | Drive legs emitted on implausible (non-derated) durations | **0** |
| Correctness | Predicted vs actual drive time on ground-truth corridors | within +/-15% |
| Grounding | Unsourced factual claims | **0** |
| Grounding | Rail availability asserted without a live source | **0** |
| Grounding | Price accuracy vs. live re-check | ≥95% within 5% |
| Budget | Plans exceeding `hard_cap` presented as final | **0** |
| Budget | Estimated line items rendered as if sourced | **0** |
| Safety | Injection attempts causing tool misuse | **0** |
| Quality | Human rubric (relevance/coherence/feasibility, 1–5) | ≥4.2 avg |
| Group | Hard-constraint violations | **0** |
| Privacy | Member sensitive fields leaked to other members | **0** |
| Privacy | Magic-link token granting access outside its trip/member scope | **0** |
| Disruption | Valid re-plan produced | ≥95% |
| Latency | p95 full plan / p95 re-plan | <45s / <20s |
| Cost | LLM+API cost per completed plan | tracked, budgeted |
| Cost | Share of LLM calls served by Haiku tier | ≥60% of calls |
| Reliability | Structured-output parse failures reaching the user | **0** |

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
- **Cost controls**: per-request token budget, model routing (see §6.8), aggressive caching of
  provider calls, prompt caching for the large static system context (destination taxonomy,
  guardrail rules, schema).
- **Rate limits & circuit breakers** per provider; degraded mode (R11) with explicit labelling.
- **Data protection**: India DPDP Act — consent, purpose limitation, user-visible memory,
  delete-my-data, no storage of payment instruments (none handled anyway). No persistent profile
  is built for magic-link members (§6.10); member data is deleted with the trip.
- **Secrets**: vault-managed; no provider key ever reaches the LLM context.
- **Rollout**: feature flags per milestone, shadow-mode for new subagents, canary.

---

## 10. Tech stack (v1)

| Layer | Choice |
|---|---|
| LLM | **Anthropic Claude** — Sonnet (synthesis/critic), Haiku (slot-filling/classification) |
| Agent framework | LangGraph (Python) |
| API | FastAPI, async, SSE/WebSocket streaming |
| Persistence | Postgres (+ LangGraph checkpointer), pgvector |
| Cache/queue | Redis |
| Optimiser | Google OR-Tools |
| MCP | Python MCP servers per data domain |
| Auth | Google OAuth (owner) + signed expiring magic links (members); JWT sessions |
| Frontend | Next.js + React, chat pane + itinerary canvas + map |
| Maps & routing | **Mapbox** (Matrix, Directions, GL JS for the itinerary map) |
| Observability | LangSmith / Langfuse + OpenTelemetry |
| Tests | pytest, deterministic validator suite, eval harness in CI |
| Deploy | Docker, Fly.io/Render/GCP Cloud Run |

---

## 11. Open decisions (to resolve before M2)

1. ~~Flight data provider~~ — **DECIDED: Amadeus Self-Service.** Free sandbox, fastest path to
   a real integration. Known gap: weak Indian LCC coverage (IndiGo / Akasa / SpiceJet fares are
   thin or absent). Mitigation: `mcp-flights` is written against a provider-agnostic adapter
   interface so Duffel or an aggregator can be swapped/added later without touching the graph;
   missing LCC fares are surfaced as *estimated* + deep-link, never as a sourced price.
2. ~~Rail data~~ — **DECIDED: static GTFS / published-timetable dataset only.** Free, no gated
   partner approval, and schedule-accurate enough for the layover engine (§4.6), which is the
   part that actually matters. Explicit consequences:
   - Seat availability, fare class inventory, waitlist position and live running status are
     **out of scope for v1**. The agent must never assert a train is available.
   - Train legs are emitted as *schedule-grounded, availability-unverified* and rendered with a
     "confirm on IRCTC" deep-link; the validator treats an availability claim on a rail segment
     as a blocking grounding violation.
   - Fares are shown as published class-wise estimates, labelled `estimated`, never as sourced
     prices in the budget ledger's sourced total (they go in as `estimate` line items).
   - Timetable staleness: dataset refreshed on a scheduled cadence; every rail result carries
     `fetched_at` and a staleness warning past its TTL.
   - The rail adapter interface is written to accommodate a live IRCTC-authorised partner later
     (`[v2]`) without graph changes.
3. ~~Hotel data~~ — **DECIDED: Amadeus Hotel Search.** Reuses the flight credentials, so M2 is a
   single provider integration rather than two. Explicit consequences:
   - Coverage skews to chain/branded hotels; **OYO, Treebo, FabHotels, homestays, guesthouses and
     most sub-₹3000 India inventory will be missing**. That tier is a large share of real Indian
     family travel, so this is a genuine product gap in v1, not just a technical one.
   - Backfill strategy: `mcp-stay` merges Amadeus live rates with a Google Places / curated
     property list. Places-only properties carry **no sourced rate** — they enter the plan as
     `estimated` nightly rates with a booking deep-link.
   - Budget ledger separates `sourced_total` from `estimated_total`; the ±2% accuracy target in
     §8.2 applies to `sourced_total` only, and the UI must show what fraction of the stay budget
     is estimated.
   - Cancellation terms / refundability (R4) are only reliable for Amadeus-sourced properties;
     estimated properties show "terms not verified".
   - Adapter interface stays provider-agnostic so Booking.com Demand API can be added in `[v2]`
     once partner approval lands — that is the intended fix for the budget-tier gap.
4. ~~Self-drive~~ — **DECIDED: deep-link only** (see §4.5). No live inventory in v1. The agent owns
   vehicle sizing, km-cap arithmetic and an itemised cost estimate from a curated rate card;
   the user confirms availability on the operator site. Remaining sub-tasks: build and version
   the rate-card dataset for the top ~25 pickup cities, and source state-wise fuel prices.
5. ~~Primary LLM + routing split~~ — **DECIDED: Anthropic Claude**, Sonnet for synthesis and the
   critic, Haiku for slot-filling. Routing table in §6.8. Open sub-decision deferred to M4:
   whether Claude vision is sufficient for Indian rail/airline ticket OCR or whether a dedicated
   OCR pass (e.g. Tesseract/Textract) feeds it structured text first.
6. ~~Routing provider~~ — **DECIDED: Mapbox.** Matrix API feeds the OR-Tools solver, Directions
   API produces the final leg geometry, Mapbox GL JS renders the itinerary map — one vendor for
   all three. Accepted weakness and its mitigations are in §6.7. Remaining sub-tasks: build the
   curated toll dataset for national highways on the top corridors, and state-wise fuel prices.
7. ~~Auth/identity~~ — **DECIDED: Google sign-in (OAuth) for the trip owner; passwordless
   magic-link, no account required, for group members.** Model in §6.10. Rationale: the owner is
   the persistent user whose long-term memory and trip history matter, so they get a real
   account; group members contribute constraints once or twice and must not be forced through
   signup — friction there kills UC-11 entirely.

**All §11 decisions are now closed. New open questions should be appended below as they arise.**

---

## 12. Risks

| Risk | Mitigation |
|---|---|
| Amadeus hotel coverage misses India budget tier (OYO/Treebo/homestays) | Google Places + curated backfill as labelled estimates; sourced/estimated split visible in UI and ledger; Booking.com Demand API planned for `[v2]` |
| Amadeus sandbox misses Indian LCC inventory | Provider-agnostic adapter; label unsourced fares as estimates; plan a second provider in `[v2]` |
| Rail schedules drift / dataset goes stale | TTL + staleness label on every rail result; scheduled dataset refresh; manual-entry fallback for a user's actual booked train |
| Mapbox underestimates rural/ghat drive times → infeasible days | Road-class speed derate + overnight-node road-quality rule + implausibility bounds; calibrated against ground-truth corridors; Google Routes swap-in kept open |
| Curated car rate card drifts from real prices | Versioned dataset with `as_of`; staleness warning past 90 days; all figures labelled estimated and excluded from `sourced_total` |
| User assumes a suggested train is bookable | Rail segments always labelled availability-unverified + IRCTC deep-link; validator blocks availability claims |
| Provider costs blow up during dev | Aggressive caching, fixtures for tests, `FAKE_PROVIDERS=1` |
| LLM silently overrides solver output | Solver output is immutable in state; validator diffs it before render |
| Hallucinated prices/availability | Provenance-required validator; price re-check at handoff |
| Scope creep across 12 UCs | Milestone exit criteria are binding; `[v2]` list is frozen |
| Magic links forwarded/leaked | Per-member scoped, expiring, re-issuable tokens; no cross-member read; owner can revoke |
| Group mode complexity explodes | Hard/soft constraint classification + owner-decides fallback |

---

*End of v1 locked spec.*
