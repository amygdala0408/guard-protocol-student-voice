# Build Prompt 1 — Intake Refactor + Conditional-Logic Engine

**Paste this entire file into your coding agent as a single message. Your working directory should be the project folder where the v4 HTML file lives. Do not edit this prompt — the constraints are load-bearing.**

**Paste-artifact note.** If your editor converts dot-separated identifiers like `state.profile.name`, `purpose.id`, or `INTERVIEW_PURPOSES.id` into nested markdown autolinks, strip them before running. The intent is plain identifiers, not URLs.

---

## Context and naming

You are working on the **Manifesto Student Voice Protocol (SVP)** — an interviewer-facing instrument for conducting post-incident student interviews in K–12 schools. SVP is **Appendix D within the Manifesto framework** for MDR use, **and** a standalone school-wide post-incident interview tool. The current build is a single-file HTML application (vanilla JS + Tailwind via CDN + Web Speech API + localStorage) that you previously built as v4.0. Stakeholder demo is approximately May 9, 2026.

**Framework rename in this build.** The framework is being renamed from "GUARD" to "Manifesto." The tool itself stays "Student Voice Protocol (SVP)." The existing v4 HTML file contains five GUARD-branded references that must be renamed in lock-step with this refactor:

- Line 6: `<title>GUARD Protocol Appendix D — Student Voice Protocol v4.0</title>` → `<title>Manifesto Student Voice Protocol — Appendix D · v4.1</title>`
- Line 143: `<span class="font-headline text-xl font-bold italic tracking-tight">GUARD Protocol Engine</span>` → `<span class="font-headline text-xl font-bold italic tracking-tight">Manifesto Engine</span>`
- Line 448: same fix as line 143 (the lg-size variant)
- Line 1386: `'GUARD Appendix D v4.0 · Evidence Collection for MDR Review'` → `'Manifesto Student Voice Protocol — Appendix D v4.1 · Evidence Collection for MDR Review'`
- Line 1503: `'Report generated from GUARD Appendix D v4.0...'` → `'Report generated from Manifesto Student Voice Protocol — Appendix D v4.1...'`

Do not rename the case-insensitive matches on "Safeguard" (lines 685 and 760) — those are the word "Safeguard," not the framework name.

This refactor pivots the tool from MDR-only to multi-purpose. The underlying nine-domain interview architecture (D-CTX, D-PHYS, D-EMO, D-ENV, D-UTD, D-SAF, D-RPR, D-BLG, D-RNT) stays constant, with one **new conditional mini-domain D-PEER** added (intentional — see section 6). What changes is the intake layer that drives downstream branching.

This is the foundational structural change for the demo. Get the data-driven architecture right once; everything else becomes content work.

---

## Files

- The single HTML file in your working directory is `student_voice_protocol_v4.html` (~1,543 lines, ~103 KB). Read it fully before making any changes.
- Do not split into multiple files. Keep the single-file architecture.

## Architectural facts about the existing file (do not break these)

- The UI is a **single-page scroll architecture**, not a multi-screen wizard. Sections render top-to-bottom on one long page.
- v4 has **equity-gate-styled sections** (the `.equity-gate` CSS class is present and used for visual distinction) but **does not yet implement actual visibility-gating behavior** beyond a consent-warning banner toggle. **This Build Prompt creates the gate-visibility pattern**, it does not extend an existing one. New gates (Purpose Selector, Coming Soon, Incident Intake) hide downstream sections by toggling a `.gate-blocked` class or `hidden` attribute on the affected section nodes; downstream sections remain in the DOM but are not displayed until their predecessor gate is satisfied.
- The codebase uses a **pure DOM construction helper `h(tag, props, ...children)` at line 913 — `innerHTML` is never used anywhere by deliberate design.** Every new component you add must use the `h()` helper. Do not introduce template strings, HTML-string concatenation, or `innerHTML` assignment.
- The codebase uses `clear(node)` at line 936 to empty containers before re-rendering. Use it; do not introduce alternatives.
- Data-driven rendering pattern: data lives in top-level `const` arrays (e.g., `DOMAINS` at line 592, `BIAS_QUESTIONS`, `CULTURAL_QUESTIONS`, `VOCAB_TIERS`, `SAFETY_RUBRIC`, `FIVE_PS`); render functions iterate over those arrays. Every new piece of content must follow this pattern — declare data at the top, render from it below.
- The existing `.equity-gate` CSS class is the visual pattern for new gates. Reuse it on the Purpose Selector outer container and the Coming Soon outer container so they visually fit.
- Existing state lives in module-scoped `answers` and `notes` objects (lines 908–909). **Do not rip these out.** Add the new top-level `state` object alongside them and have the new code write to `state` while the existing v4 code continues to use `answers` and `notes`. A later prompt will consolidate.
- v4 has a **threat-pause and routing-handoff bar** (sticky bar with timestamp + "route to threat assessment team" handoff). It is *not* a full threat-assessment flow with branching logic. Treat it as a handoff bar; do not assume there is logic to integrate with that does not exist.

---

## Goal

Refactor the intake from a single MDR-prep flow into a three-layer intake (Purpose → Student Profile → Incident Context) that drives a data-driven conditional-logic engine. Two purposes are fully active in the demo (MDR-prep, Universal post-incident); the other eight are visible in the menu but route to a "Coming soon — in development" landing page.

---

## What to build

### 1. Purpose Selector (data-driven, 10 items)

Add an "Interview Purpose" gate **section** (single-page scroll, not a separate screen). It sits immediately after the existing trusted-adult / consent gate and before the metadata (student profile) section. Until a purpose is selected, downstream sections (metadata, gate-bias, gate-cultural, environment-section, ground-rules, domains, closure, observation log, etc.) are hidden by toggling a `hidden` attribute or `.gate-blocked` CSS class on each downstream section's container — implementing the gate-visibility pattern that v4 has visually staged but not yet behaviorally enforced.

The gate's content reads from a JavaScript data array declared near the existing top-level `const` blocks. Adding or removing purposes later is a data change, never a UI rebuild.

```js
const INTERVIEW_PURPOSES = [
  { id: "mdr",                label: "Manifestation Determination preparation", description: "For students with IEPs facing disciplinary removal that triggers MDR procedures.",          status: "active",      route: "fullInterview" },
  { id: "universal",          label: "Universal post-incident interview",       description: "For any student after a behavioral incident, regardless of IEP/504 status.",                  status: "active",      route: "fullInterview" },
  { id: "504",                label: "Section 504 incident review",             description: "For students with 504 plans after a behavioral incident.",                                    status: "stubbed",     route: "comingSoon" },
  { id: "fba",                label: "FBA / BIP development input",             description: "Behavior specialist or school psych collecting student voice for FBA/BIP development.",        status: "stubbed",     route: "comingSoon" },
  { id: "restorative",        label: "Restorative practice preparation",        description: "Restorative coordinator prepping for a circle or conference.",                                 status: "stubbed",     route: "comingSoon" },
  { id: "re_entry",           label: "Re-entry interview",                      description: "Counselor or admin preparing a student's return after suspension or removal.",                 status: "stubbed",     route: "comingSoon" },
  { id: "bullying",           label: "Bullying / peer conflict investigation",  description: "Title IX or investigator interviewing a student involved in a bullying or conflict report.",   status: "stubbed",     route: "comingSoon" },
  { id: "tier2",              label: "Tier 2 check-in",                         description: "Mentor or case manager conducting a structured Tier 2 student check-in.",                      status: "stubbed",     route: "comingSoon" },
  { id: "student_requested",  label: "Student-requested debrief",               description: "Student asked to talk; counselor conducting a debrief with no downstream routing.",            status: "stubbed",     route: "comingSoon" },
  { id: "threat_pause",       label: "Threat assessment routing",               description: "Pass-through to the existing threat-pause and routing-handoff bar.",                            status: "passthrough", route: "existingThreatPauseBar" }
];
```

UI requirements:
- Render every purpose as a selectable card with the label and description visible.
- `active` purposes have a green dot indicator and route into the full interview flow on selection.
- `stubbed` purposes have an amber dot indicator labeled "In development." On selection, route to the Coming Soon landing page (defined below).
- `passthrough` purposes surface the **existing threat-pause and routing-handoff bar** — do not invent additional logic for it.
- The selected `purpose.id` is stored in `state.interview.purpose` and is included in every export payload.

### 2. Coming Soon landing page

Implement as a render function — `renderComingSoon(purpose)` — that uses the `h()` helper exclusively. No HTML strings, no `innerHTML`. Match the existing pattern in v4 (e.g., the way `renderClosure`, `renderDomain`, and `makeWarningCard` build DOM trees).

When a stubbed purpose is selected, the rest of the page (interview body, closure, observation log, etc.) is hidden and `renderComingSoon` populates the visible content area with:

- Purpose label as the headline.
- A clean message: "In development — currently in research phase."
- An unordered list of research areas being addressed for this purpose, read from `purpose.researchAreas`. Leave the array empty for stubbed purposes for now; content lands in a later prompt.
- A "Back to purpose selection" button that re-shows the Purpose Selector and clears the Coming Soon container.
- No "force into interview anyway" escape hatch. If they want to interview the student, they pick MDR or Universal.

Use the existing `.equity-gate` class on the outer container so it visually matches v4's gate styling.

### 3. Refactored Student Profile intake

The v4 metadata section already captures most of these. Confirm all of these fields exist and add any missing ones. **Mirror values into `state.profile` while preserving the existing v4 metadata behavior** (do not duplicate UI; the same form input writes to both the existing v4 metadata storage and the new `state.profile`).

| Field | Type | Required | Stored as |
|---|---|---|---|
| Student name | text | yes | `state.profile.name` |
| Grade | dropdown (K-12) | yes | `state.profile.grade` |
| Date of birth | date | yes | `state.profile.dob` |
| Age band (auto-derived from grade, with override) | derived: HS / MS / UE | yes | `state.profile.ageBand` |
| IEP status | radio: Yes / No / Unsure | yes | `state.profile.iepStatus` |
| 504 plan status | radio: Yes / No / Unsure | yes | `state.profile.section504Status` |
| ELL status | radio: Yes / No / Exited | yes | `state.profile.ellStatus` |
| Primary language at home | text (autocomplete from common list, free-form fallback) | conditional on ELL=Yes | `state.profile.primaryLanguage` |
| AAC user | checkbox | no | `state.profile.aacUser` |
| ASL user | checkbox | no | `state.profile.aslUser` |
| Known disability categories | multi-select (SLD, OHI, ASD, ED, ID, SLI, TBI, MD, OI, VI, HI, DB, DD, "Other or unspecified") | conditional on IEP=Yes | `state.profile.disabilityCategories` |

Notes:
- Person-first language throughout. Never "disabled student" — always "student with a disability" or framing that does not lead with the disability.
- The disability categories list above corresponds to the 13 IDEA categories. Use the official IDEA category labels in tooltips.
- Do not require disclosure of specific disability beyond categories. Specific diagnoses are not collected by this tool.

### 4. New Incident Intake section

After the existing student profile (metadata) section, before the bias/cultural/environment gates, add an **Incident Intake section** (single-page scroll architecture, not a separate screen). Use the same visual pattern as the metadata section. Fields:

| Field | Type | Required | Stored as |
|---|---|---|---|
| Incident date | date | yes | `state.incident.date` |
| Time of day | dropdown (Before school / Morning / Mid-day / Afternoon / After school) | yes | `state.incident.timeOfDay` |
| Setting | dropdown (see list below) | yes | `state.incident.setting` |
| Who was present | multi-select (Adults only / Peers only / Both adults and peers / Neither — student alone) | yes | `state.incident.whoPresent` |
| Nature of behavior | radio (Self-directed / Peer-directed / Property-directed / Authority-directed / Multiple) | yes | `state.incident.behaviorNature` |

Setting dropdown values:

```js
const SETTINGS = [
  { id: "classroom",      label: "Classroom (instructional time)",      status: "active",    fallbackTo: null },
  { id: "hallway",        label: "Hallway / passing period",            status: "stubbed",   fallbackTo: "classroom" },
  { id: "cafeteria",      label: "Cafeteria / lunch",                   status: "stubbed",   fallbackTo: "classroom" },
  { id: "athletic",       label: "Athletic field / gym",                status: "stubbed",   fallbackTo: "classroom" },
  { id: "playground",     label: "Playground / recess",                 status: "stubbed",   fallbackTo: "classroom" },
  { id: "locker_room",    label: "Locker room",                         status: "stubbed",   fallbackTo: "classroom" },
  { id: "bathroom",       label: "Bathroom",                            status: "stubbed",   fallbackTo: "classroom" },
  { id: "bus",            label: "Bus / transportation",                status: "stubbed",   fallbackTo: "classroom" },
  { id: "parking_lot",    label: "Parking lot",                         status: "stubbed",   fallbackTo: "classroom" },
  { id: "assembly",       label: "Assembly / field trip",               status: "stubbed",   fallbackTo: "classroom" },
  { id: "off_campus",     label: "Off-campus school event",             status: "stubbed",   fallbackTo: "classroom" },
  { id: "unstructured",   label: "Unstructured transition (catch-all)", status: "stubbed",   fallbackTo: "classroom" }
];
```

Setting behavior:
- Only `classroom` is fully active in this build.
- For all `stubbed` settings, the conditional-logic engine treats them as if `classroom` were selected (uses `fallbackTo`).
- When a stubbed setting is selected, show a clean, non-judgmental banner above the interview: "Setting-specific questions for [setting label] are in research. This interview uses the general question set." Banner is dismissable but reappears on next session in this interview.
- Do not make the setting selector a hard block — interviewers in the demo can pick any setting and the interview proceeds with the classroom question set.

### 5. Conditional-Logic Engine (the load-bearing piece)

Build a single `evaluateConditions(state, conditions)` function that takes the current interview state and a conditions config object and returns a boolean. Every place in the rest of the app that decides "should this question/domain/section surface?" calls this function. New rules become config entries — never new code paths.

Config object shape (this is the schema; populate it for the demo cases below):

```js
const CONDITION_RULES = {
  // Domain-level visibility
  "domain.D-PEER.visible": {
    description: "Peer dynamics mini-domain surfaces when peers were present at incident.",
    require: { "incident.whoPresent": { includesAny: ["Peers only", "Both adults and peers"] } }
  },
  "domain.D-BLG.visible": {
    description: "Belonging domain surfaces when behavior involved social dynamics, evidence of not belonging, or identity-based discrimination/microaggressions (per v4 conditional). Wired in a later prompt — placeholder for now.",
    require: { "_alwaysFalseForNow": true }
  },
  "domain.D-RNT.visible": {
    description: "Repair & Re-Entry domain surfaces when student has been or will be removed, or when the behavior caused documented harm (per v4 conditional). Wired in a later prompt — placeholder for now.",
    require: { "_alwaysFalseForNow": true }
  },

  // Field-level visibility
  "field.D-CTX-04.visible": {
    description: "IEP Service Delivery Context only asked when student has an IEP.",
    require: { "profile.iepStatus": { equals: "yes" } }
  },

  // Purpose-level routing
  "purpose.fullInterview.allowed": {
    description: "Full interview only allowed for active purposes.",
    require: { "interview.purpose": { in: ["mdr", "universal"] } }
  }
};
```

Engine rules:
- `equals`, `in`, `includesAny`, `notEquals`, `truthy`, `falsy` are the supported predicate operators. Add others if you need them, but document each.
- A condition with no `require` returns true.
- A condition with `require: { "_alwaysFalseForNow": true }` returns false. Use this as a placeholder for rules wired in later prompts so the rule key exists from day one.
- Resolution of a state path uses dot notation (`profile.iepStatus` reads `state.profile.iepStatus`).
- The function is pure — no side effects.

### 6. Field ID registry — derived from DOMAINS, with D-PEER additions

**Important correction from a prior version of this prompt:** the field IDs and constructs already exist in the `DOMAINS` array (lines 592–864 of v4) with NICHD-aligned, age-banded prompts. Those questions are research-validated and stay as-is. We do **not** redefine them.

What this build adds:

- A lightweight **field ID registry derived from `DOMAINS`** at module load time, keyed by field ID, returning `{ domain, label, qtype }`. This gives the conditional-logic engine and (later) the export schema a single lookup interface without duplicating content.
- Three **new D-PEER mini-domain field IDs** that don't yet exist in `DOMAINS` because the question content is authored in Build Prompt 4. The IDs are reserved here so the engine and registry recognize them.

```js
// Build the registry by reading DOMAINS so we don't duplicate content
const FIELD_REGISTRY = (() => {
  const reg = {};
  for (const d of DOMAINS) {
    for (const q of d.questions) {
      reg[q.id] = { domain: d.id, label: q.label, qtype: q.qtype };
    }
  }
  // Reserve D-PEER IDs (question content authored in Build Prompt 4)
  reg["D-PEER-01"] = { domain: "D-PEER", label: "Peer Coercion / Pressure",   qtype: "Inv.", reserved: true };
  reg["D-PEER-02"] = { domain: "D-PEER", label: "Peer Ridicule / Humiliation", qtype: "Inv.", reserved: true };
  reg["D-PEER-03"] = { domain: "D-PEER", label: "Bystander Dynamics",          qtype: "Dir.", reserved: true };
  return reg;
})();
```

For the human reader, here is the **v4-aligned field map** you are working with (this is documentation, not code — the registry above is the runtime source of truth):

| ID | Domain | v4 Label | NICHD type |
|---|---|---|---|
| D-CTX-01 | Context & Antecedents | Temporal Context | Inv. |
| D-CTX-02 | Context & Antecedents | Immediate Antecedent | Dir. |
| D-CTX-03 | Context & Antecedents | Instructions and Expectations | Dir. |
| D-CTX-04 | Context & Antecedents | IEP Service Delivery Context | Opt. |
| D-CTX-05 | Context & Antecedents | Perceived Control | Dir. |
| D-PHYS-01 | Physiological State | Pre-Incident Physiological Baseline | Inv. |
| D-PHYS-02 | Physiological State | Physiological Changes During Incident | Inv. |
| D-PHYS-03 | Physiological State | Action Impulses & Regulation Capacity | Inv. |
| D-PHYS-04 | Physiological State | Recovery Timeline | Dir. |
| D-EMO-01 | Emotional State & Affect Regulation | Emotional State Pre-Incident | Inv. |
| D-EMO-02 | Emotional State & Affect Regulation | Emotional Intensity and Overwhelm | Dir. |
| D-EMO-03 | Emotional State & Affect Regulation | Emotional Cognition During Incident | Inv. |
| D-EMO-04 | Emotional State & Affect Regulation | Shame and Self-Judgment | Inv. |
| D-ENV-01 | Environmental Factors & Perceived Triggers | Sensory and Environmental Contributors | Inv. |
| D-ENV-02 | Environmental Factors & Perceived Triggers | Social-Environmental Factors | Inv. |
| D-ENV-03 | Environmental Factors & Perceived Triggers | Staff Interaction & Perceived Respect | Inv. |
| D-ENV-04 | Environmental Factors & Perceived Triggers | Changes to Routine / Unexpected Transitions | Dir. |
| D-UTD-01 | Understanding & Decision-Making | Cognitive Clarity During Incident | Inv. |
| D-UTD-02 | Understanding & Decision-Making | Awareness of Consequences | Dir. |
| D-UTD-03 | Understanding & Decision-Making | Intentionality of Behavior | Inv. |
| D-UTD-04 | Understanding & Decision-Making | Alternative Behavior Awareness | Dir. |
| D-SAF-01 | Safety Context | Perceived Threat or Danger | Inv. |
| D-SAF-02 | Safety Context | Normalization-then-Ask (Leading Question Safeguard) | Opt. |
| D-SAF-03 | Safety Context | Access to Trusted Adults | Dir. |
| D-SAF-04 | Safety Context | Internal vs. External Locus of Safety | Dir. |
| D-RPR-01 | Repair & Restoration | Repair Capacity and Willingness | Inv. |
| D-RPR-02 | Repair & Restoration | Future Safety and Success Conditions | Inv. |
| D-RPR-03 | Repair & Restoration | Closing Question — Student Voice | Inv. |
| D-BLG-01 | Belonging & Community Connectedness (conditional) | Sense of Belonging to School Community | Inv. |
| D-BLG-02 | Belonging & Community Connectedness (conditional) | Peer Relationships and Social Status | Inv. |
| D-BLG-03 | Belonging & Community Connectedness (conditional) | Role of Belonging Struggles in the Incident | Dir. |
| D-BLG-04 | Belonging & Community Connectedness (conditional) | Future Belonging and Reintegration | Inv. |
| D-BLG-05 | Belonging & Community Connectedness (conditional) | Identity and Discrimination | Dir. |
| D-RNT-01 | Repair & Re-Entry (conditional) | Student Understanding of Impact | Inv. |
| D-RNT-02 | Repair & Re-Entry (conditional) | Accountability Orientation | Inv. |
| D-RNT-03 | Repair & Re-Entry (conditional) | Supports Needed for Successful Re-Entry | Inv. |
| D-RNT-04 | Repair & Re-Entry (conditional) | Student as Co-Designer of Re-Entry Plan | Inv. |
| **D-PEER-01** | **Peer Dynamics (NEW conditional mini-domain)** | **Peer Coercion / Pressure** | **Inv. (content in BP4)** |
| **D-PEER-02** | **Peer Dynamics (NEW conditional mini-domain)** | **Peer Ridicule / Humiliation** | **Inv. (content in BP4)** |
| **D-PEER-03** | **Peer Dynamics (NEW conditional mini-domain)** | **Bystander Dynamics** | **Dir. (content in BP4)** |

D-PEER is intentionally a 10th mini-domain. v4 has 9 domains; the addition is deliberate, scoped to three fields, conditional-only, and addresses the peer ridicule and coercion gap identified in research. **Question content for D-PEER-01/02/03 is authored in Build Prompt 4** — for now reserve the IDs and have the registry recognize them.

### 7. State shape

Add a new top-level `state` object alongside (not replacing) the existing `answers` and `notes` objects. The existing v4 code continues to write to `answers` and `notes`; new code (purpose, incident intake, conditional-logic engine, response provenance) writes to `state`. A later prompt consolidates them. Shape:

```js
state = {
  schema_version: "0.9.0-beta",
  framework: "Manifesto",
  tool: "Student Voice Protocol — Appendix D",
  interview_id: <uuid v4 generated at session start>,
  interviewer: {
    role: null,
    is_disciplining_admin: null,
    preflight_completed: false,
    bias_reflection_completed: false,
    cultural_preparation_completed: false
  },
  interview: {
    purpose: null,                  // one of INTERVIEW_PURPOSES.id
    started_at_utc: null,
    completed_at_utc: null
  },
  profile: { /* see field list above */ },
  incident: { /* see field list above */ },
  responses: { /* keyed by field ID — populated by interview screens */ },
  observations: {
    regulation_checks: [],
    pauses_taken: 0
  }
};
```

Persist to localStorage on every state change using the existing `saveDraft` / `loadDraft` pattern (see lines 1338+). Extend those functions to also serialize/deserialize the new `state` object under a distinct key (`svp_state_<interview_id>`); do not break the existing `answers` / `notes` persistence.

### 8. Provenance metadata structure for state.responses

Even though full provenance population happens in later prompts, every response entry written to `state.responses` should be shaped this way from day one:

```js
state.responses["D-PHYS-01"] = {
  value: "...",
  provenance: {
    age_band: state.profile.ageBand,
    tier_used: null,                 // populated in Build Prompt 3 (scaffold cards)
    scaffold_viewed: false,           // populated in Build Prompt 3
    rephrase_count: 0,                // populated in Build Prompt 2 (interviewer icons)
    probe_count: 0,                   // populated in Build Prompt 2
    response_source: "student_voice", // default; updated based on interpreter/AAC paths in later prompts
    language: "en-US",
    duration_seconds: null,
    regulation_observation: null,
    prompt_version: "v1.0.0",
    purpose: state.interview.purpose,
    setting: state.incident.setting,
    timestamp_utc: <captured at field completion>
  },
  reliability_flags: []
};
```

This shape is the contract for the export schema landing in Build Prompt 5. The fields populated in this prompt are: `age_band`, `purpose`, `setting`, `prompt_version`, `language`, `timestamp_utc`, plus the `value` itself when v4's existing capture path saves a response. Everything else stays at its default.

---

## Constraints (non-negotiable)

1. **Single HTML file. Vanilla JS. Tailwind via CDN. Web Speech API for the existing TTS/STT. localStorage for persistence.** No new dependencies.
2. **Preserve every existing v4 capability.** Bias-mitigation pre-flight, ground rules script, rapport-building script, practice narrative, regulation-check nudges, age band selector, listen/record icons, info icons where they exist, threat-pause and routing-handoff bar. Do not regress any of this.
3. **Equity is a forcing function.**
   - The "Are you the disciplining admin?" check stays as a hard block at preflight. Cannot proceed if yes.
   - Bias-mitigation reflection stays mandatory before the substantive phase.
   - Cultural self-preparation stays mandatory before the substantive phase.
   - Stubbed purposes do not have an "interview anyway" escape hatch. They route to Coming Soon.
4. **The tool guides; it does not decide.** Nothing in this refactor produces a determination, a verdict, or a recommendation about manifestation, discipline, or behavior. The data is captured for the team.
5. **Person-first language throughout.** "Student with a disability," not "disabled student." "Student who uses AAC," not "AAC student."
6. **Asset-based language.** Home language is a resource, not a deficit. Disability is a category of identity, not a problem to overcome.
7. **Field IDs are immutable.** The registry derives from `DOMAINS` (single source of truth) plus the three reserved D-PEER IDs. If a new ID is needed mid-build, stop and add it to `DOMAINS` (or the D-PEER reservation block) first.
8. **NICHD-aligned, age-banded prompts in `DOMAINS` are not rewritten in this prompt.** The constructs and prompts (HS / MS / UE) at lines 592–864 of v4 stay intact. We are adding intake and engine, not editing question content.

---

## Do NOT touch in this prompt

- **Setting-conditional question variants** for hallway / cafeteria / athletic field / etc. — deferred post-demo per scope decision. The setting dropdown ships UI-only with the fallback-to-classroom behavior described above.
- **The three interviewer icons** (Info / Rephrase / Probe Deeper) — coming in Build Prompt 2.
- **Student scaffold cards** for D-PHYS / D-EMO / D-ENV / D-UTD content — coming in Build Prompt 3.
- **D-PEER question content** — IDs are reserved; surfacing logic is wired; the actual question fields land in Build Prompt 4.
- **Export schema (JSON + PDF)** — coming in Build Prompt 5.
- **Existing NICHD-aligned prompts in `DOMAINS`** — these are the validated content and are not rewritten here.

---

## Acceptance criteria

A reviewer can verify each of these without running the app:

1. The HTML file still loads and runs in a modern browser with no console errors.
2. The interviewer pre-flight (consent, threat-pause/handoff disclosure, ground rules, rapport, practice narrative, bias reflection, cultural prep, disciplining-admin check) is unchanged in behavior.
3. The 5 GUARD-branded references (lines 6, 143, 448, 1386, 1503) are renamed to Manifesto branding per the Context section. The "Safeguard" word matches at lines 685 and 760 are NOT renamed.
4. After pre-flight, a Purpose Selector gate appears with all 10 purposes, correctly labeled with their status indicators (green dot / amber dot / passthrough).
5. Selecting MDR or Universal reveals the downstream sections (metadata → Incident Intake → bias gate → cultural gate → environment → ground rules → domains → closure → observation log) and routes into the full interview flow.
6. Selecting any of the 8 stubbed purposes routes to the Coming Soon landing page with a working Back button.
7. Selecting Threat assessment routing surfaces the existing v4 threat-pause and routing-handoff bar (unchanged).
8. The Student Profile section captures every field in the table above and writes to `state.profile`.
9. The Incident Intake section captures every field in the table above and writes to `state.incident`.
10. Selecting a stubbed setting shows the "Setting-specific questions in research" banner and proceeds with the classroom question set.
11. The conditional-logic engine evaluates `domain.D-PEER.visible` correctly: returns true only when `incident.whoPresent` includes peers.
12. The conditional-logic engine evaluates `field.D-CTX-04.visible` correctly: returns true only when `profile.iepStatus === "yes"`.
13. `state.responses["D-CTX-01"]` etc. populate with the new shape including provenance metadata when a response is captured (reuse v4's existing capture path; populate the new shape on top of it).
14. localStorage persistence works: refreshing the page mid-interview restores the session, including the new `state` object under the `svp_state_<interview_id>` key.
15. `FIELD_REGISTRY["D-CTX-01"]` returns `{ domain: "D-CTX", label: "Temporal Context", qtype: "Inv." }` — i.e., the registry derives correctly from `DOMAINS`.
16. `FIELD_REGISTRY["D-PEER-01"]` returns `{ domain: "D-PEER", label: "Peer Coercion / Pressure", qtype: "Inv.", reserved: true }`.
17. No regression in existing v4 features.
18. No nested markdown autolinks or paste artifacts ship in the code.

---

## When you finish

Reply with:
- A summary of files changed and lines added/modified.
- Any places where you had to make a judgment call not covered by this prompt.
- Any acceptance criterion you could not satisfy and why.
- Any new ambiguity you surfaced that affects future prompts (Build Prompts 2–5).

Do not improvise scope. If something feels missing, flag it in your summary rather than building it.
