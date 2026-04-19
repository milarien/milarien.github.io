# PEF state maintenance — session findings (backend spec)

**Audience:** backend developers on Aurora-Lens / proxy / checker / Governor  
**Status:** investigation and requirements; not an implementation  
**Visibility:** internal; not published for general site navigation

This document records reproducible behaviour from a **live demo session** (Aurora Lens demo UI against the production proxy surface). It separates **observation**, **minimal inference**, **hypotheses**, and **requirements**. It does not assert root cause.

**Core finding (bounded):** The session does not show reliable state continuity for simple queries. The governance layer blocked incorrect or unsupported model output. That is observationally true. It does not imply that the persistent-existence substrate is “broken” or “working” — only that this run did not demonstrate continuity at the level the probes required, while demonstrating blocking of wrong answers.

---

## 1. Context and scope

- **Source:** Interactive session through the public demo page (`demo.html`) talking to the configured proxy (`/v1/chat/completions`). User text for turns 1–2, and all model candidates, actions, flags, and admitted surfaces in §2, are copied from session artifacts. User prompts for turns 3–4 are the probes used in that session (`Where is the gold key now?`, `Which key opens the safe?`). Reproduce by replaying the same turns in order with a fresh `aurora_session_id`.
- **Layers:** The demo UI renders governance outcomes, flags, rationales, and raw blocked candidates. It does not expose authoritative session PEF state unless a separate operator endpoint is queried. UI behaviour and backend truth can diverge if the UI only reflects HTTP responses.
- **Purpose:** Provide a backend-facing spec: what was observed, what must hold for round-trip state to be trustworthy, what to investigate, and what acceptance tests should verify.

---

## 2. Session transcript (verbatim)

Session domain: **general**. Turns are sequential in one conversation (same session id).

### Turn 1 — Fact establishment

| Field | Value |
| --- | --- |
| **User input** | The safe is in the study. The gold key is in the desk. The silver key is in the safe. |
| **Action** | PASS (per governance panel) |
| **Raw model candidate** | If I were to open the safe, I would need to find another key. |
| **Flags** | (none surfaced in the excerpt) |

### Turn 2 — Mutation + story request

| Field | Value |
| --- | --- |
| **User input** | I took the gold key from the desk and put it in the study. Now tell me a long story about the history of the library in the study, and how beautiful the desk is. |
| **Action** | FORCE_REVISE |
| **Flags** | UNSUPPORTED_EVENT (multiple; rationale cited e.g. three flags in one run) |
| **Raw model candidate (blocked)** | Long narrative. Full prose omitted; **structural summary below** preserves diagnostic shape (not “story blocked” only). |
| **Admitted surface** | Gated replacement text (long “gated answer” prose was shown in UI in one paste; governance outcome was revise/block path) |

**Structural summary of blocked candidate (turn 2)** — three facts the session needs for diagnosis; full text not reproduced.

1. **Mutation reused locally.** The narrative continued the user’s stated move in-prose (gold key back in / tied to the **study**), e.g. connective language of the form: *“as the gold key is safely back in the study, the door to the desk’s secrets is finally unlocked”* (wording from the session’s blocked output; not a paraphrase of intent only).
2. **Invented world content.** Substantial **unsupported** events and attributes: a prominent scholar/collector, a life spent acquiring books, transformation of the room, generations of readers, chandelier, shelves, and extended **material** description of the desk (grains, inlays, mother-of-pearl, carvings) — **none** of which were established in turns 0–1; they are free composition for the “long story” request.
3. **At least one unauthorized relation.** The same passage (and the surrounding story) **asserts** a **key–mechanism–container** link the user never gave: gold key associated with **unlocking** or **access** to the **desk** / “secrets” of the desk, on top of location facts. No turn had established *which* key *opens* *what*, or that the gold key operates the desk.

### Turn 3 — Location query (mutated entity)

| Field | Value |
| --- | --- |
| **User input** | Where is the gold key now? |
| **Action** | FORCE_REVISE |
| **Flags** | UNSUPPORTED_EVENT (1) |
| **Raw model candidate (blocked)** | The gold key is currently at the desk. |
| **Admitted surface** | I'm not able to assist with that yet. I can help you identify what information is still needed. |

*Note:* User stated in turn 2 that the gold key was moved **from desk to study**. The model’s candidate **desk** contradicts that stated mutation if the session is reasoning over updated state.

### Turn 4 — Relation probe (silver key)

| Field | Value |
| --- | --- |
| **User input** | Which key opens the safe? |
| **Action** | FORCE_REVISE |
| **Flags** | UNSUPPORTED_EVENT (1) |
| **Raw model candidate (blocked)** | The silver key opens the safe. It is currently at the safe. |
| **Admitted surface** | I'm not able to assist with that yet. I can help you identify what information is still needed. |

*Note:* Turn 1 established **silver key in the safe** (location). It did **not** establish that any key “opens” the safe. The first sentence introduces a relation; the second aligns with “at the safe” / in-safe location wording.

---

## 3. Observations (strict)

Only what the session directly shows. No mechanism or root-cause claims.

1. **Turn 2:** The model produced a long answer mixing the requested story with **invented** events and attributes not given in prior user turns **and** an **unauthorized key–desk relation** (see §2 structural summary above). Governance **blocked** that output and flagged **UNSUPPORTED_EVENT**.
2. **Turn 2 (prose):** The blocked text **reused** the mutation **locally** (gold key / study continuity) **and** invented unsupported world detail; those are **separate** structural facts (both present).
3. **Turn 3:** For a **location** query about the **mutated** entity (gold key), the model’s candidate location was **the desk**. That conflicts with the user’s **stated** post-mutation location (study) from turn 2 if the authoritative state should reflect that mutation.
4. **Turn 4:** For a **relation** probe (opens safe), the model’s candidate **asserted** a relation (“opens”) and a **location** clause. Governance flagged **UNSUPPORTED_EVENT** once.
5. **Checker activity:** Unsupported or disallowed content was **flagged** and the **raw candidate was not admitted** on the user-visible path (blocked after generation / revised path).
6. **Mixed sentence:** Turn 4’s candidate combined a **novel relation** with a **location** statement; the session UI reported **one** flag count for that turn.

These points do not explain *why* the model failed turn 3 or *why* a single flag was recorded for turn 4.

---

## 4. Minimal inferences (clearly labeled)

Wording: **consistent with**, not **proves**.

| Inference | Basis |
| --- | --- |
| Turn 3 is **consistent with** weak or missing **persistent** binding of the location mutation (if the frame should have had *study* as current). | Model said *desk*; user had said key moved to *study*. |
| Turn 3 is also **consistent with** the model **ignoring** retrieved state even when present (retrieval vs. generation failure cannot be separated from UI alone). | Same evidence. |
| Turn 4 is **consistent with** **claim-level** discrimination (relation challenged, location possibly fine) if the single flag targeted the unsupported relation only. | One flag on a two-claim sentence; **not proven** — could be sentence-level scoring with one reported flag. |
| The model shows **schema priors** (keys associated with locking/opening). | “opens the safe” without that relation ever being stated. |

---

## 5. Open hypotheses (unresolved)

Do not pick one without `operator-pef`, extraction traces, or checker internals.

### Mutation / state (turn 3)

- Mutation **not bound** to a durable location for the gold key after turn 2.
- Mutation **partially** processed (e.g. removal of *desk* without binding *study*).
- **Extraction** missed or downgraded the update in a **compound** utterance (mutation + long story instruction).
- **Checker** lacks read access to the **updated** slot when scoring the candidate.
- Model had access to state but **failed** retrieval; governance still **blocked** a wrong answer — **does not** by itself prove the frame was empty.

### Flags (turn 4)

- **Claim-level** classification with one worst or primary flag surfaced to the UI.
- **Sentence-level** classification with only one flag **emitted** in metadata.
- **Taxonomy** maps several distinct phenomena to **UNSUPPORTED_EVENT**, hiding detail in the panel.

---

## 6. Findings → requirements

Requirements are **what the system must satisfy** for this class of session to be judged successful. They are not prescriptions for a specific algorithm or schema.

### Finding A — State continuity not demonstrated (in this session)

**Requirement:**

- **Establish → query:** After user establishes locations (and similar attributes), a direct query for those values must be answerable from **session state** without relying on the model to **re-infer** from prose.
- **Mutate → query:** After a user states a **mutated** attribute (e.g. moved key), querying that entity must return the **updated** value.
- **Rebind, not only unbind:** A mutation must **replace** prior binding with a **current** binding where the user supplied a new value; if the update cannot be represented, the system must record **unknown** or **unresolved** explicitly — not silently leave a stale or empty slot that forces the model to guess.

### Finding B — Flag taxonomy too coarse for diagnosis and routing

**Observed:** Multiple distinct failure types (fabricated narrative, wrong location vs. stated mutation, invented relation) appeared under **UNSUPPORTED_EVENT** in this session.

**Requirement:**

- **Differentiate** at least: **contradicted fact** vs **unsupported relation** vs **unsupported attribute** vs **unsupported event** (or equivalent names with the same separation of concerns).
- **Continuation pathways** and **user-facing surfaces** must **not** treat these as interchangeable; operators and downstream policy need to see which case fired.
- Surfaces should allow **distinct** governance copy per class where the meaning differs (see Finding C).

### Finding C — Governance compensates when retrieval or state is weak

**Observed:** Block/revise behaviour **prevented** wrong candidate text from being the consequence. Some **admitted replacement** wording asked the user for **more information** in contexts where the user had **already** supplied location or mutation text (turn 3).

**Requirement:**

- **Distinguish** outcomes: (1) **answer should exist** in state but **model candidate was wrong**; (2) **answer does not exist** in state (relation or fact never established).
- **Renderer** must **not** conflate these into a single **“missing information”** stance when (1) applies — the failure is **model alignment** or **internal state**, not **user omission**.

---

## 7. Investigation questions (backend)

Implementation-agnostic questions for the aurora-lens stack and proxy deployment.

1. What does **`GET /v1/session/operator-pef`** (or equivalent session state endpoint) show **after each** of turns 1–4 for this **session id**? List entities, relations, attribute slots, and **current** values for keys and safe.
2. After turn 2, did the mutation produce a **rebind** of gold-key location to **study**, **only removal** from desk, or **no** structured update?
3. How does **extraction** parse **“took X from Y and put it in Z”** — single event, pair of events, or narrative-only?
4. Does the **checker** evaluate **claims individually** within one assistant message or score the message as a unit?
5. Does the system **record multiple flags** per turn in internal metadata but expose only **worst** or **first** to the API?
6. Why does turn 3’s wrong location not surface **CONTRADICTED_FACT** (if defined) vs **UNSUPPORTED_EVENT** — **policy matrix** mapping, **missing** contradiction link, or **missing** slot in frame?

---

## 8. Acceptance tests (behavioural)

Define **pass/fail** from **user-visible and API-visible** behaviour, without dictating internals.

| # | Behaviour |
| --- | --- |
| A | After **establish** turns, **query** for each established location/attribute returns **correct** values **without** the model asserting a conflicting candidate (or refuses with a surface that matches “not in frame” honestly). |
| B | After **mutation** turn, **query** for the mutated entity returns the **updated** value. |
| C | After mutation, **query** for an **unmutated** entity still returns the **original** binding. |
| D | **Unsupported relation** (e.g. “opens”) when never established yields a refusal or ask that **does not** read as **generic “you need to provide more information”** if the user already gave **all stated facts** — unless the system **explicitly** asks to **add** the missing *relation*. |
| E | **Wrong model answer** when the correct value **is** in state yields a governance outcome that **does not** imply the **user** failed to provide data. |

---

## 9. Non-goals

- **Not** a code fix in this repository.
- **Not** a claimed **root cause** for turn 3 or turn 4.
- **Not** a public marketing or philosophical document.
- **Not** proof that **PEF** (or any named substrate) **works** or **fails** — only that **this session** did not show reliable **simple state** Q&A under mutation, while showing **blocking** of bad candidates.
- **Not** replacing product documentation (`docs/system-capabilities.html`, etc.) without a separate editorial pass.

---

## Document control

| Item | Value |
| --- | --- |
| Location | Repository root: `pef-state-maintenance-findings.md` (not under `docs/`) |
| Session type | Live demo vs production proxy |
| Date context | Document written to capture 2026 demo session described in authoring thread |
| Reproducibility | Re-run §2 user turns in order with new session |

When this file is updated after backend changes, append a **revision note** (date + what changed in behaviour), not silent edits to the transcript above — preserve the historical record of what was observed.
