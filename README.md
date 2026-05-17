# FSD Readiness & Trust Coach
**MGMT 276 — Product Strategy | Anderson School of Management**  
Alberto Ruiz & Yirong (Jen) Peng | May 2026

---

## What This Is

An AI-native prototype that re-architects Tesla's FSD onboarding experience for new Model Y subscribers. The core problem: FSD trialists churn not because FSD underperforms, but because when something goes wrong, the car says nothing. This product fixes that.

> *"I wasn't scared of the driving — I was scared of the silence."*  
> — Interview participant, FSD trialist who did not convert

---

## The Problem

Every month, tens of thousands of Tesla Model Y owners activate Full Self-Driving for the first time. Most cancel before day 30 — not because FSD failed, but because it failed **silently**. No explanation for disengagements. No camera health visibility before activation. No guidance on when or where to use FSD. The system punishes first and educates never.

**The broken loop:**
```
Activate FSD → Disengagement, no explanation → Assume product failure
→ Book service appointment → Cancel subscription → Tesla loses a flywheel turn
```

---

## The Solution — Three AI Agents

### Sub-Agent A · FSD Readiness Dashboard
Polls all 8 exterior cameras every 60 seconds. Surfaces camera health status **before** FSD activation — not after failure. If a camera degrades overnight, the driver gets a push notification naming the specific camera, so they clean one lens instead of all eight.

### Sub-Agent B · Plain-Language Explanation Engine  
When FSD disengages, an LLM converts structured disengagement event data into a one-sentence plain-language explanation — delivered on-screen within 700ms. *"Right B-pillar camera is dirty — clean it and FSD will restore immediately."* No error codes. No silence. No unnecessary service appointment.

### Sub-Agent C · Progressive Trust Builder
Analyzes commute patterns and surfaces personalized route suggestions during the first 30 days. Tracks weekly FSD miles toward a progress milestone. Coaches drivers on usage patterns (acceleration overrides, hands-off events) through a park-mode-only vehicle journal — habit formation before the renewal decision arrives.

---

## Live Prototype

**[→ Open Prototype](index.html)**  
*(Or view live at: `https://[albertorusot].github.io/fsd-trust-coach/`)*

The prototype demonstrates all three sub-agents in a single interactive interface:
- **Tab A:** Click any camera to simulate degradation and watch the agent respond in real time
- **Tab B:** Select a disengagement scenario (6 types) and run the Explanation Engine to see the full agent trace — input schema → RAG → output schema → in-car display
- **Tab C:** View the route recommendation, weekly progress tracking, and coaching journal

No backend required. All agent logic runs client-side in the browser.

---

## Repository Structure

```
fsd-trust-coach/
├── index.html          ← Interactive prototype (all three sub-agents)
├── source_of_truth.md  ← Full technical specification (agent prompts, schemas, RAG strategy)
└── README.md           ← This file
```

---

## Technical Specification

See [`source_of_truth.md`](source_of_truth.md) for the complete agent specification, including:
- System prompt architecture for all four sub-agents (A, B, C, D)
- Full input/output JSON schemas
- RAG strategy and retrieval corpus design
- Data requirements table (source, signal, format, frequency, new instrumentation)
- Performance targets (latency, cost, hallucination rate, correctness)
- Safety mechanisms and behavioral rules
- Build order and verification checklist

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| OTA-only, no new hardware | Keeps prototype shippable without physical intervention |
| Sub-Agent B capped at 25 words | In-vehicle safety UX standard — explanation must render before driver reacts |
| Sub-Agents A and B operate independently | Different triggers, different surfaces — conflating them creates untestable logic |
| Sub-Agent D has no UI | All measurement is backend-only; no driver-facing surface |
| Trialists only (30-day cohort) | Isolates intervention effect; avoids referral-user self-selection bias |

---

## Eval Set Summary

**Sub-Agent B (Explanation Engine):**
- 500 prompt-response pairs across 4 strata: routine (50%), edge cases (25%), adversarial (15%), safety-critical (10%)
- Ground truth written by domain reviewers; two-reviewer agreement required
- Targets: ≥90% correctness (3/3), 0% hallucination, P95 latency ≤700ms, cost ≤$0.002/inference
- Evaluation strategy: Agent-as-a-Judge (Claude Opus) for routine/edge + 100% human review for safety-critical tier

**Sub-Agent C (Progressive Trust Builder — A/B Experiment):**
- 90-day experiment design: 2,400 users/arm, 80% power, α=0.05, MDE=5pp
- Treatment: personalized route prompts + weekly progress card vs. control (no guidance)
- Primary: Month-3 resubscription rate (+5–10pp expected)
- Guardrails: FSD disengagement rate (≤+2pp), CSAT (≤−3 points), service appointments (flat)

---

## Built With

- HTML/CSS/JavaScript (no framework dependencies)
- All agent logic is client-side for demo reliability
- Architecture designed for production implementation with LLM API (Anthropic Claude or fine-tuned model) + Tesla vehicle data APIs

