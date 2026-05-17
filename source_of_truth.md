# FSD Readiness & Trust Coach — Source of Truth Specification
**Course:** MGMT 276 — Product Strategy | Anderson School of Management  
**Team:** Alberto Ruiz & Yirong (Jen) Peng  
**Version:** Final (v3.0) | May 2026  
**Prototype:** [fsd-trust-coach demo](index.html)

---

## 1. Overview

### Problem
FSD doesn't fail badly — it fails opaquely. The target user — a Tesla Model Y new owner on the default 30-day free FSD trial — is blindsided and penalized when FSD disengages without an intelligible reason, and churns before forming a habit. This is a "punishes first, educates never" UX failure.

**The broken loop:**
```
Driver activates FSD → System refuses or disengages → No explanation given →
Driver assumes product failure → Books service appointment → Churns from subscription
```

The problem is not that FSD failed. The problem is that the user felt penalized for something they did not know to prevent.

### Success Metrics

All metrics normalized to trial end (T=0 = Day 30):

| Metric | Target | Baseline |
|--------|--------|----------|
| Trial-to-paid conversion at T=0 | 10–12% | ~2% (YipitData, 2024) |
| Month-3 retention at T+90 | 40% | ~25% (internal estimate) |
| Camera-related service appointments | ↓10% | Baseline TBD |
| FSD engagement miles per subscriber | ↑20% | Baseline TBD |

**North star:** Month-3 retention among converted subscribers.  
**Must beat:** GM Super Cruise 2025 month-3 retention of ~30%.

### Guardrails

- **Multi-driver suspension correlation:** If suspensions from secondary drivers correlate with trialist churn, Sub-Agent C's multi-driver logic is retention-critical and cannot be deferred.
- **False-positive notification rate:** Track dismissals and snoozes. Target: <5% of DEGRADED alerts without a visible cause.
- **Self-resolve rate:** Fraction of disengagement explanations resolved without a service visit. Regressions signal Explanation Engine drift.
- **In-drive safety:** Zero notifications during active driving. Zero outputs may imply FSD is fully autonomous.

### Scope

**In scope:**
- Software-only, OTA-deliverable on existing Model Y hardware stack
- Four sub-agents: A (Readiness Dashboard), B (Explanation Engine), C (Trust Builder), D (Evaluation)
- Target: Tesla Model Y owners on 30-day free trial + secondary drivers on the vehicle

**Out of scope:**
- Extended-trial (90-day referral) users
- EV-only buyers (never activated FSD)
- Committed long-term subscribers
- Robotaxi / fleet use cases
- Changes to the FSD driving model or inference pipeline
- New hardware or sensors
- Service scheduling automation
- Driver monitoring integration

---

## 2. Sub-Agent A — FSD Readiness Dashboard

### Purpose
Surface camera health and FSD readiness **before** activation — not after failure. The driver sees system state before pressing the stalk, not after a disengagement strands them without explanation.

### Behavior

**Continuous monitoring (every 60 seconds):**
- Poll per-camera confidence scores for all 8 exterior cameras
- Check map data freshness timestamp
- Evaluate weather and solar angle against camera-specific risk thresholds
- Emit a structured status object (see output schema below)

**Overnight health check:**
- If any camera score drops below CAUTION threshold between park and next morning:
  - Push Tesla app notification naming the specific degraded camera
  - Highlight camera location on in-car car diagram
  - Driver cleans one lens, not all eight

**FSD status is vehicle-level:** All driver profiles see the same state, warnings, and countdown.

### Output Schema (Mode 1 — Monitor)

```json
{
  "overall_status": "READY" | "CAUTION" | "DEGRADED",
  "cameras": {
    "front_main":       { "score": 0.0–1.0, "flag": "CLEAR" | "WARN" | "BLOCKED" },
    "front_narrow":     { "score": 0.0–1.0, "flag": "CLEAR" | "WARN" | "BLOCKED" },
    "left_b_pillar":    { "score": 0.0–1.0, "flag": "CLEAR" | "WARN" | "BLOCKED" },
    "right_b_pillar":   { "score": 0.0–1.0, "flag": "CLEAR" | "WARN" | "BLOCKED" },
    "left_fender":      { "score": 0.0–1.0, "flag": "CLEAR" | "WARN" | "BLOCKED" },
    "right_fender":     { "score": 0.0–1.0, "flag": "CLEAR" | "WARN" | "BLOCKED" },
    "rear_main":        { "score": 0.0–1.0, "flag": "CLEAR" | "WARN" | "BLOCKED" },
    "rear_plate":       { "score": 0.0–1.0, "flag": "CLEAR" | "WARN" | "BLOCKED" }
  },
  "map_freshness_minutes": 0–∞,
  "map_status": "FRESH" | "STALE" | "CRITICAL",
  "degradation_cause": "OBSTRUCTION" | "GLARE" | "CONDENSATION" | "WEATHER" | "UNKNOWN",
  "driver_action_required": true | false,
  "notification_triggered": true | false,
  "notification_camera": "camera_id" | null
}
```

### Thresholds

| Status | Score Range | Action |
|--------|-------------|--------|
| READY | ≥ 0.80 | No action |
| CAUTION | 0.60–0.79 | Monitor; no notification unless trend declining |
| DEGRADED | < 0.60 | Push notification + in-car indicator |

### Data Requirements

| Source | Signal | Format | Frequency | New instrumentation? |
|--------|--------|--------|-----------|----------------------|
| FSD Vision Stack | Per-camera confidence score | Float 0.0–1.0 per camera | Every 60s | No — byproduct of existing inference |
| Weather API | Precipitation type, fog, visibility index | Structured JSON | Every 5 min | No — third-party API |
| Solar Angle API | Sun azimuth/elevation vs. camera orientation | Float (degrees) | Real-time | No — computed from GPS + time |
| Vehicle State Bus | Drive mode, FSD active, park state | Enum, event-driven | On state change | No — existing CAN bus signal |
| Session History | Last clean timestamp, prior DEGRADED events | Local key-value store | Per session | No — written by Sub-Agent A |

### Behavioral Rules

**Must:**
- Show DEGRADED only when score < 0.60 sustained for ≥ 30 seconds (prevents flicker)
- Name the specific camera in every notification (never "a camera is dirty")
- Show CAUTION rather than DEGRADED if fleet RAG data indicates likely self-resolution within 20 minutes
- Suppress overnight notifications if vehicle is in active drive mode

**Must not:**
- Show READY when any camera is at DEGRADED threshold (false positives destroy trust faster than silence)
- Generate more than one push notification per degradation event
- Fire any notification while vehicle is in Drive or Reverse

### Fallback
If camera score API is unavailable: emit `overall_status: UNKNOWN`, suppress all notifications, log the gap. Do not fabricate a status.

---

## 3. Sub-Agent B — Plain-Language Explanation Engine

### Purpose
When FSD disengages or becomes unavailable, surface a one-sentence, actionable plain-language reason — on the in-car display and in the Tesla app — within 700ms. The driver is never left to assume FSD itself failed.

### Architecture

Sub-Agent B is triggered by a disengagement event from the FSD stack. It is **not** triggered by Sub-Agent A — the two operate independently on different surfaces.

**Input:** Structured disengagement event payload (see schema below)  
**Output:** Plain-language explanation + routing decision  
**Delivery:** In-car touchscreen overlay + Tesla app notification (park mode only)

### Input Schema (Disengagement Event Payload)

```json
{
  "reason_code": "CAM_OBSTRUCTION" | "WEATHER_DEGRADED" | "GLARE" | "LANE_DETECTION_FAIL" |
                 "CONSTRUCTION_ZONE" | "ATTENTION_WARNING" | "MULTI_DRIVER_SUSPENSION" |
                 "EMERGENCY_MANEUVER" | "UNKNOWN",
  "camera_affected": "camera_id" | null,
  "camera_score": 0.0–1.0 | null,
  "weather": "string (structured: type · intensity)",
  "route_context": "string (road type, direction, time of day)",
  "severity": "LOW" | "MEDIUM" | "HIGH",
  "prior_disengagements_today": 0–∞,
  "caused_by_non_primary": true | false
}
```

### Output Schema

```json
{
  "explanation_text": "string (≤25 words, plain language, actionable)",
  "action_required": "CLEAN_CAMERA" | "MANUAL_CONTROL" | "WAIT" | "NONE",
  "service_visit_needed": true | false,
  "restore_condition": "string (what triggers FSD re-availability)",
  "safety_check_passed": true | false,
  "word_count": 0–25
}
```

### Behavioral Rules

**Must:**
- Reference only reason codes and sensor values present in the input payload (hallucination guard)
- Cap output at 25 words, enforced server-side
- Always include either an action or a timeframe ("it will restore when…")
- When `caused_by_non_primary: true`, name the cause explicitly ("another driver on this vehicle")
- Set `service_visit_needed: false` for all self-resolvable causes (CAM_OBSTRUCTION, WEATHER_DEGRADED, GLARE, CONSTRUCTION_ZONE, ATTENTION_WARNING)

**Must not:**
- Use the word "failure" in any user-facing output
- Imply FSD is fully autonomous (post-processing classifier screens every output)
- Reference GPS coordinates or route history (PII — stripped from input before inference)
- Generate output during active driving (in-drive display only; Tesla app notification deferred to park mode for non-urgent explanations)

**Fallback:**
- If input is outside known disengagement schema: reject upstream, route to safe fallback: *"FSD has disengaged — please take control of the vehicle."*
- If retrieval returns no matches: revert to static code-to-copy catalog. Never guess.

### RAG Strategy

Retrieval priority for disengagement/suspension copy:
1. **Tesla official support docs** — canonical for codes and suspension policy; first lookup for any safety-critical or policy claim
2. **Curated disengagement-code → copy catalog** — team-owned mapping to one-sentence, action-oriented in-drive messages; covers self-resolvable cases end-to-end
3. **Owner forums** (Tesla Motors Club, r/teslamotors) — valid for usage wording from real driver experience; never sole source for safety-critical claims

**Retrieval corpus index:**
```
Index: reason_code | camera_position | degradation_cause | weather_context | region | season
Runtime query: "code: CAM_OBSTRUCTION | camera: right_b_pillar | score: 0.23 |
               weather: clear | region: Los Angeles | season: spring"
```

### Safety Mechanisms

| Mechanism | Implementation |
|-----------|----------------|
| Input grounding | Prompt schema limits LLM to fields present in event payload only |
| Output length cap | 25 words maximum, enforced server-side before display |
| PII filter | GPS coordinates and route history stripped before inference |
| Autonomy classifier | Post-processing check on every output; flagged outputs → safe fallback |
| Adversarial input filter | Out-of-schema inputs rejected upstream, routed to fallback |

### Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Hallucination rate | 0% | Hard requirement — incorrect explanations create service visits |
| Correctness (human review) | ≥90% rated 3/3 | Fully correct and actionable |
| P50 latency | <300ms | Explanation renders before driver reacts |
| P95 latency | <700ms | Hard gate — do not ship if breached |
| Cost per inference | <$0.002 | Caps fleet cost at ~$6,000/month at 3M events |

---

## 4. Sub-Agent C — Progressive Trust Builder

### Purpose
Solve the cold-start problem. Most trialists don't churn because of a single bad experience — they churn because they never built the habit of using FSD. The Trust Builder personalizes the first 30 days to each driver's actual commute patterns and provides reinforcement before the renewal decision arrives.

### Behavior

**Route suggestions (in-car, during trial period only):**
- Analyze prior trip logs to identify commute patterns with high FSD compatibility
- Surface contextual nudge on map screen when a recommended route is detected
- Trigger conditions: compatible route + favorable weather + no active degradation flag
- Suppressed if: weather degraded, active DEGRADED status from Sub-Agent A, or prior prompt dismissed in last 24 hours

**Example prompt:** *"Your commute on the 405 is a great route to try FSD — tap to activate"*

**Weekly progress card (Tesla app):**
- FSD miles logged this week
- Progress bar toward 30-day milestone
- Updated Sunday midnight; visible in Tesla app home screen widget

**Usage-pattern coaching (park mode only, shared vehicle journal):**
- One journal per vehicle, shared across all profiles
- Entries written only in park mode — never during drive or reverse
- Coaching triggers:
  - Acceleration-while-engaged detected
  - Hands-off duration exceeding threshold
  - Off-route override taken
  - Clean session ≥ 10 consecutive miles (positive reinforcement)

**Multi-driver awareness:**
- Detect 2+ active profiles on vehicle
- When non-primary driver attention warnings accumulate, surface single aggregated notification to current active driver: *"Another driver: 3/5 attention warnings this week — 2 more will pause FSD for 7 days."*
- Updated in place (no duplicate notifications per week)
- Suspension reason copy is owned by Sub-Agent B, not Sub-Agent C

### Output Schema

```json
{
  "route_suggestion": {
    "triggered": true | false,
    "route_description": "string",
    "compatibility_score": 0.0–1.0,
    "prompt_text": "string (≤15 words)"
  },
  "weekly_progress": {
    "fsd_miles_this_week": 0–∞,
    "goal_miles": 0–∞,
    "day_of_trial": 1–30,
    "milestone_reached": true | false
  },
  "coaching_entry": {
    "triggered": true | false,
    "trigger_type": "ACCELERATION_OVERRIDE" | "HANDS_OFF" | "OFF_ROUTE" | "CLEAN_SESSION",
    "entry_text": "string (plain language, no word limit in park mode)",
    "write_to_journal": true | false
  },
  "multi_driver_alert": {
    "triggered": true | false,
    "non_primary_strike_count": 0–5,
    "alert_text": "string"
  }
}
```

### Data Requirements

| Source | Signal | Format | Frequency |
|--------|--------|--------|-----------|
| Trip logs | Route, engagement miles, disengagement locations | Structured JSON | Per trip |
| In-drive event stream | Acceleration-while-engaged, hands-off duration, off-route delta | Event-driven | Real-time |
| Per-profile attention-warning counter | Strike count with weekly decay | Integer | On event |
| Multi-driver profile registry | Active profiles + non-primary strike aggregation | Array | On profile switch |

### Behavioral Rules

**Must:**
- Suppress all route suggestions during active DEGRADED status (Sub-Agent A signal)
- Suppress prompts in weather conditions that would increase disengagement likelihood
- Gate coaching entries to park mode only — no in-drive coaching copy
- Aggregate multi-driver strikes into a single notification (never send one per driver)

**Must not:**
- Suggest FSD on routes the agent cannot confirm as compatible
- Send a route prompt if one was dismissed in the last 24 hours
- Write to the vehicle journal during drive or reverse mode
- Display multi-driver suspension copy (this is owned by Sub-Agent B)

---

## 5. Sub-Agent D — Evaluation Layer (No UI)

### Purpose
Backend measurement substrate that tracks whether the prototype meets the success metrics defined in §1. Sub-Agent D renders nothing to the driver — all UI belongs to A, B, and C.

### Per-Account Record

One row per 30-day trialist:
- Trial enrollment date (T=0 anchor = Day 30)
- Household type: single / multi-driver
- Subscription outcome with date and method: not converted / converted / converted-then-churned
- Stated cancellation reason (if provided)

### Per-Event Tags

All events emitted by A, B, and C carry:
- `days_from_T0`: signed integer (negative = days before trial end)
- `caused_by_non_primary`: boolean (on suspension and strike events)

### Measures

**Primary:**
- Trial-to-paid conversion at T=0
- Month-3 retention at T+90

**Guardrails:**
- Multi-driver suspension correlation with churn
- False-positive notification rate (dismissals + snoozes)
- Self-resolve rate (camera/map issues resolved without service visit)

**Secondary:**
- Camera-related service appointments
- FSD engagement miles per subscriber per week

**Hard constraint:** No Sub-Agent D code path writes to any driver-facing surface.

---

## 6. Build Order

```
Phase 1 — Sub-Agent A
  → Define camera-health thresholds for all 8 cameras
  → Ship glanceable Ready/Degraded panel with per-camera callout and car diagram
  → Wire overnight Tesla-app notification naming the specific camera
  → Make FSD state vehicle-level across profiles

Phase 2 — Sub-Agent B
  → Enumerate disengagement and unavailability codes
  → Build code-to-copy catalog against RAG priority order
  → Enforce ≤25-word guardrail server-side
  → Gate self-resolvable issues away from service routing
  → Implement suspension-caused-by-another-driver copy path

Phase 3 — Sub-Agent C
  → Derive commute patterns and ship Progressive Trust Builder route suggestions
  → Add usage-pattern coaching (acceleration-while-engaged, hands-off, off-route)
  → Aggregate non-primary strikes into single in-place notification
  → Build shared park-mode-only journal with weekly pattern grouping

Phase 4 — Sub-Agent D
  → Stand up event store and per-account record (30-day trialists only)
  → Tag events with two per-event tags
  → Instrument primary conversion at T=0 and retention at T+90
  → Verify no code path writes to a driver-facing surface
```

---

## 7. Known Limitations (v1)

| Limitation | Reason | Future Consideration |
|-----------|--------|---------------------|
| Extended-trial (90-day referral) users excluded | Avoids conflating self-selection bias with Trust Coach effect | Add in v2 after A/B results |
| Multi-driver personalization limited to aggregated strike counts | Per-driver FSD profiles would require hardware/software changes outside OTA scope | v3 roadmap |
| Route suggestions use historical commute data only | Real-time traffic-aware routing requires integration with Nav stack (separate team) | v2 dependency |
| Parking intelligence not addressed | Different trust loop; requires separate sensor + decision improvements | Out of scope |
| Defensive driving intelligence (nearby unsafe drivers) not addressed | Requires behavioral prediction model changes — changes inference stack | Out of scope |
| Human-AI handoff countdown indicator not included | Requires firmware change to drive/steer actuation layer — not OTA | Roadmap item |
| Sub-Agent B eval set uses simulated ground truth | Tesla safety engineer review requires internal access; proxy reviewers used for prototype eval | Production requirement |

---

## 8. Verification Checklist (Pre-Ship)

- [ ] In-drive copy ≤25 words, ≤2 seconds of driver attention; every message carries action or timeline
- [ ] Journal copy exempt from word count but not from plain-language standard
- [ ] Never show optimistic health status when camera score is at DEGRADED threshold
- [ ] Never imply FSD is more autonomous than it is (classifier passes on 100% of outputs)
- [ ] Never require a service visit for a self-resolvable issue
- [ ] No hardware or driving-model changes in this scope
- [ ] Sub-Agents A/B/C emit to Sub-Agent D with both per-event tags
- [ ] No Sub-Agent D path writes to a driver-facing surface
- [ ] P95 latency ≤700ms verified in load test before ship
