# Build Prompt 5 — Export Schema (JSON + PDF)

**Paste this entire file into your coding agent as a single message. Working directory: `~/Projects/guard-protocol/` where `student_voice_protocol_v4.html` lives. Build Prompts 1 and 2 must be merged before this prompt runs.**

**Paste-artifact note.** If your editor converts dot-separated identifiers like `state.responses`, `provenance.purpose`, `interview.purpose`, `state.profile.iepStatus`, or `FIVE_PS.precipitating` into nested markdown autolinks, strip them before running.

---

## Context

You are continuing the **Manifesto Student Voice Protocol (SVP)** — Appendix D within Manifesto and a standalone post-incident interview tool. BP1 added the intake refactor and conditional-logic engine. BP2 added the three interviewer icons, NICHD probe hierarchy, and provenance counters. This prompt — **BP5** — builds the export. It is the load-bearing piece for the May 9 stakeholder demo: without it, the end-to-end value (capture → structured export → downstream consumers) isn't visible.

The export does not produce a determination. It produces a structured evidence record that organizes the student's account for team deliberation, with reliability flags traveling with every captured field, and routes the data to one of several downstream consumers based on the selected interview purpose.

## Why this build is research-grounded the way it is

This is not arbitrary schema design. Every architectural choice closes a documented gap in the published MTSS/PBIS literature:

- The **29% "Unknown motivation" rate** in SWIS Office Discipline Referral data (PBISApps, 2026) — a system-level evidentiary gap the SVP export directly addresses by providing the student's account of antecedents, function, and setting events.
- The **TATE FBA quality findings** (Iovannone et al., 2024) — setting events scored mean 0.34/2 and antecedents-for-appropriate-behavior scored 0.49/2 across 135 FBAs in 13 districts. SVP export structures evidence in the categories where current FBA practice is weakest.
- The **two statutory MDR questions** under 34 CFR §300.530(e) — (a) was the conduct caused by, or had a direct and substantial relationship to, the student's disability; (b) was the conduct the direct result of failure to implement the IEP. The export must structure evidence to inform both questions without answering either.
- **Hidden profile research** (Stasser & Titus, 1985; Stasser & Stewart, 1992) — teams systematically fail to share uniquely-held information unless the task is framed as fact-finding rather than consensus-seeking. The export's section structure surfaces uniquely-held student-perspective information that may otherwise stay buried in deliberation.
- **Cognitive bias literature** (Meguerdichian et al., 2024) — confirmation, anchoring, halo/horn, and fundamental attribution bias affect facilitated team conversations. The export deliberately separates the student's account from any framing that would invite premature conclusion.
- **Adultification bias** (Goff et al., 2014; Epstein et al., 2017) — Black children are perceived as older and less innocent, which distorts evaluation of remorse, intent, and capacity in MDR. The export's `reliability_flags` and Critical Remorse Trap surfacing operationalize this evidence.
- **Evidence summarization governance** (NIST AI RMF; UNESCO GenAI guidance; ED SPPO/FERPA) — outputs that influence high-stakes educational decisions require source-grounded, traceable, audit-logged generation with human review. The schema enforces these by structure.

These citations belong inline in the rendered PDF report header so the audit trail is honest and visible. Do not add citations to the JSON.

---

## Files

- `student_voice_protocol_v4.html` (now ~3,156+ lines after BP1 + BP2 merges). Read it fully before changing anything.
- Pay particular attention to:
  - `state` object shape (added in BP1).
  - `state.responses[<id>]` provenance and reliability_flags (BP1 + BP2).
  - The existing print-render path at line ~1386 area (`'Manifesto Student Voice Protocol — Appendix D v4.1 · Evidence Collection for MDR Review'`) — this is the report block. Extend it, don't replace it.
  - `FIVE_PS` object — the 5 Ps routing map (predisposing / perpetuating / precipitating / protective / prognostic) used to organize export sections.
  - `FIELD_REGISTRY` (BP1) and `ICON_CONTENT` (BP2) — both used to label and contextualize fields in the export.

## Architectural facts (carryover)

- Single-page HTML, vanilla JS, Tailwind via CDN, Material Symbols icons. No new dependencies.
- Pure DOM construction via `h(tag, props, ...children)`. `innerHTML` is never used.
- Use `clear(node)` to empty containers before re-rendering.
- Data lives in top-level `const` arrays/objects; render functions iterate.
- The existing `state` object alongside legacy `answers` and `notes` — both stay intact.

## Goal

Build two export paths and one schema:

1. **JSON download** — full schema-versioned export of the captured interview, suitable for ingestion by Manifesto, FBA templates, restorative prep sheets, re-entry plan generators, and the PBIS climate aggregator. Includes provenance metadata on every response, reliability flags, 5 Ps routing, and a SHA-256 integrity hash placeholder (computed at download time).
2. **PDF print-to-file** — human-readable evidence record. Built by extending v4's existing print path. Includes the disclaimer banner, the quality summary, the 5 Ps-routed evidence sections, the reliability flag surfacing, and a machine-readable footer block linking the PDF to its JSON twin via `interview_id`.
3. **Single source of truth schema** declared once as a top-level `const`, used to build both exports.

Purpose-based routing: when `interview.purpose === "mdr"` the PDF emphasizes the two §300.530(e) statutory questions and the IEP-implementation evidence. When `interview.purpose === "universal"` the PDF strips IEP-specific framing and emphasizes school-climate signals. Stubbed purposes (`504`, `fba`, `restorative`, etc.) export the same JSON but the PDF lands on a "Coming soon — exports for this purpose are routed but not yet rendered" page so the demo can still demonstrate the routing architecture.

---

## What to build

### 1. The export schema declaration

Declare a single top-level `const EXPORT_SCHEMA_VERSION = "0.9.0-beta";` near the BP1 state declaration. Both JSON and PDF reference this version.

Declare the canonical export shape as a function `buildExportPayload(state)` that returns the full structured object. This is used by the JSON download path; the PDF path reads from this same function so the two exports never drift.

```js
function buildExportPayload(state) {
  return {
    schema_version: EXPORT_SCHEMA_VERSION,
    framework: "Manifesto",
    tool: "Student Voice Protocol — Appendix D",
    tool_version: "v4.1",
    interview_id: state.interview_id,
    generated_at_utc: new Date().toISOString(),
    purpose: state.interview.purpose,
    purpose_label: lookupPurposeLabel(state.interview.purpose),

    interviewer: { ...state.interviewer },

    student_profile: redactProfileForPurpose(state.profile, state.interview.purpose),
    incident_context: { ...state.incident },

    domains: buildDomainsBlock(state),         // see Section 4
    five_ps_routing: buildFivePsBlock(state),  // see Section 5
    quality_summary: buildQualitySummary(state),
    interviewer_observations: { ...state.observations },

    disclaimers: [
      "This capture does not constitute a Manifestation Determination.",
      "Reliability flags must be reviewed before any captured field is used as evidence.",
      "Captured information is 'relevant information' under 34 CFR §300.530(e); the determination of whether the conduct was caused by or had a direct and substantial relationship to the student's disability, or was the direct result of failure to implement the IEP, must be made by the team of qualified professionals."
    ],

    integrity: {
      sha256_of_payload: null  // computed at download time after the rest is serialized
    }
  };
}
```

Implement supporting helpers:

- `lookupPurposeLabel(id)` — pulls the purpose's `label` from `INTERVIEW_PURPOSES`.
- `redactProfileForPurpose(profile, purpose)` — for `purpose === "universal"`, returns a de-identified profile (strips `name`, `dob`, `primaryLanguage` text; keeps `ageBand`, `grade`, `iepStatus`, `section504Status`, `ellStatus` boolean-flagged). For `purpose === "mdr"` and all other authenticated purposes, returns the full profile. The de-identification is conservative; demo only ships MDR + Universal active, but the helper exists so the universal path is honest at the architectural level.
- `buildDomainsBlock(state)` — see Section 4.
- `buildFivePsBlock(state)` — see Section 5.
- `buildQualitySummary(state)` — see Section 6.

### 2. The JSON download path

Add a new export panel near the bottom of the page (after the existing observation log, before the print-trigger button) titled **"Export evidence record."** It contains:

- A short paragraph: "Generates the structured evidence record for this interview. Reliability flags travel with each captured field. The export does not contain a determination — it organizes evidence for team review."
- A "Download JSON" button (primary) — wired to the JSON download flow below.
- A "Generate PDF" button (secondary) — wired to the existing print-render path, extended per Section 8.

JSON download flow:

```js
async function downloadJsonExport() {
  const payload = buildExportPayload(state);
  // Compute integrity hash AFTER all other fields are populated
  const serialized_without_hash = JSON.stringify({ ...payload, integrity: { sha256_of_payload: null } }, null, 2);
  const hash = await sha256Hex(serialized_without_hash);
  payload.integrity.sha256_of_payload = hash;
  const final = JSON.stringify(payload, null, 2);
  const filename = "svp-" + state.interview.purpose + "-" + state.interview_id + ".json";
  triggerDownload(final, filename, "application/json");
}
```

Implement helpers:

- `sha256Hex(str)` — using `window.crypto.subtle.digest("SHA-256", new TextEncoder().encode(str))` and converting the resulting `ArrayBuffer` to a hex string. Returns a Promise.
- `triggerDownload(content, filename, mime)` — creates a `Blob`, an object URL, an `<a download>` element with `h()`, clicks it, then revokes the URL. Standard pattern.

The `sha256_of_payload` is computed over the payload with `integrity.sha256_of_payload === null`, so downstream consumers can re-verify by zeroing that field and re-hashing. **This is the contract.**

### 3. The PDF generation path

Extend the existing print-render path that v4 already has at line ~1386 area. The existing code renders the report as a print-friendly DOM block; this prompt adds new sections to that block based on `buildExportPayload(state)`. **Do not write a separate PDF library or replace the print path.** The user invokes "Generate PDF" → the page enters print mode → user prints to file.

Rendering changes to the print block:

1. **Top-of-report disclaimer banner** — bordered, prominent, in the same visual register as the existing legal-cite block at line ~1503. Contains the three disclaimer strings from `payload.disclaimers`. Citations from the "Why this build is research-grounded" block at the top of this prompt appear directly below the disclaimer in a small-type "Research basis" footer.
2. **Identification block** — `interview_id`, `schema_version`, `tool_version`, `generated_at_utc`, purpose label, setting label.
3. **Profile + incident context block** — uses `redactProfileForPurpose` so a Universal export does not print the student's name.
4. **Quality summary block** — see Section 6. Includes counts of NICHD utterance types used (open/cued/directive/option-posing), total rephrase invocations, total probe invocations, fields with `suggestive_risk` flag, and the regulation-check observations.
5. **Domains block** — for each captured response: field ID, field label (from `FIELD_REGISTRY`), the response value, the provenance metadata in a small-type footer (age band, tier used, response source, language, timestamp), and the reliability flags rendered as visible chips. Fields with `suggestive_risk` get an amber rule above them and a "Multi-probe — review for suggestibility" inline note.
6. **5 Ps routing block** — see Section 5. Organizes the captured response IDs into the five formulation categories. This is the block that maps directly to Manifesto's downstream clinical formulation step.
7. **MDR-specific block** (when purpose === "mdr") — Two sections corresponding to the two §300.530(e) statutory questions, each populated with the evidence available to inform the question, never with a conclusion. See Section 7.
8. **Machine-readable footer** — `interview_id`, `schema_version`, and the SHA-256 hash from the JSON payload, in a small fixed-width font block at the bottom of the printed report. This allows downstream teams to verify a printed PDF matches the JSON they received.

### 4. `buildDomainsBlock(state)` — domain-by-domain response payload

Returns an object keyed by domain ID (`D-CTX`, `D-PHYS`, `D-EMO`, `D-ENV`, `D-UTD`, `D-SAF`, `D-RPR`, `D-BLG`, `D-RNT`, `D-PEER`). Each domain object contains:

```js
{
  domain_id: "D-CTX",
  domain_title: "Context & Antecedents",
  was_surfaced: true,                     // false if the conditional engine kept it hidden
  conditional_basis: null,                // when not surfaced, the rule key that gated it
  responses: [
    {
      field_id: "D-CTX-01",
      field_label: "Temporal Context",   // from FIELD_REGISTRY
      qtype: "Inv.",                     // from FIELD_REGISTRY
      value: "...",                      // from state.responses[id].value
      provenance: { ... },               // full provenance metadata
      reliability_flags: [...],          // including suggestive_risk if applicable
      five_ps_routing: ["precipitating"] // computed from FIVE_PS membership
    }
    // ...
  ]
}
```

**Important:** unanswered fields (no entry in `state.responses` or empty value) are included in the array with `value: null` and `provenance: null` so the downstream consumer can distinguish "not asked" from "asked but no response." This matters for the IEP-implementation prong of MDR — a missing D-CTX-04 means the question was never put to the student.

### 5. `buildFivePsBlock(state)` — clinical formulation routing

Reads `FIVE_PS` (already in v4 at line ~584). Returns an object with five keys (predisposing / perpetuating / precipitating / protective / prognostic), each containing an array of `{ field_id, field_label, value, reliability_flags }` for every captured response that maps to that P.

A field can route to multiple Ps; reflect the `FIVE_PS` membership exactly. Do not invent routings.

For MDR exports, this block is the single most-consumed structure by the downstream Manifesto build — treat it as the API contract.

### 6. `buildQualitySummary(state)` — interviewer fidelity metadata

Returns:

```js
{
  total_fields_captured: <number of state.responses entries with non-empty value>,
  total_fields_unanswered: <number with value === null or empty string>,
  rephrase_invocations: <sum of all provenance.rephrase_count>,
  probe_invocations: <sum of all provenance.probe_count>,
  fields_with_suggestive_risk: <count of state.responses entries whose reliability_flags includes "suggestive_risk">,
  fields_with_dysregulated_response: <count where regulation_observation === "red">,
  fields_interpreter_mediated: <count where response_source === "interpreter_mediated">,
  fields_aac_mediated: <count where response_source === "AAC_mediated">,
  scaffold_tier_distribution: { tier1: <count>, tier2: <count>, tier3: <count>, none: <count> },  // from provenance.tier_used
  regulation_check_observations: [...state.observations.regulation_checks],
  pauses_taken: state.observations.pauses_taken,
  total_duration_seconds: <computed from state.interview.started_at_utc to state.interview.completed_at_utc, null if either is missing>,
  nichd_utterance_distribution: null  // populated only if BP4 has tagged the captured responses by qtype; otherwise null
}
```

This block answers the audit questions: how thorough was the interview, where might evidence be weak, and what conditions affected capture. It is the equivalent of the TATE Item 1 evaluation — but applied to the interview itself, not retroactively scored.

### 7. MDR-specific evidence block (when `purpose === "mdr"`)

Two sub-sections in the printed PDF only (the JSON `domains` block already contains all the data; this is a presentation layer for MDR teams).

**Section A — Evidence relevant to the disability-causation prong (34 CFR §300.530(e)(1)(i))**

Lists, with field IDs and verbatim student responses, the captured data that bears on whether the conduct was caused by or had a direct and substantial relationship to the student's disability. Pulled from these fields when present and surfaced:

- D-PHYS-01, D-PHYS-02, D-PHYS-03 (autonomic state evidence — Polyvagal)
- D-EMO-01, D-EMO-02, D-EMO-03 (affect dysregulation evidence)
- D-EMO-04 (with the Critical Remorse Trap safeguard inline — adultification flag if applicable)
- D-UTD-01, D-UTD-02, D-UTD-03, D-UTD-04 (cognitive access, intentionality, flexibility)
- D-CTX-01, D-CTX-02, D-CTX-05 (incident reconstruction, perceived control)
- D-ENV-01, D-ENV-02 (sensory and social setting events)

Heading text:
> "Section A — Evidence to inform whether the conduct was caused by, or had a direct and substantial relationship to, the student's disability (34 CFR §300.530(e)(1)(i)). The team of qualified professionals — not this report — makes that determination."

**Section B — Evidence relevant to the IEP-implementation-failure prong (34 CFR §300.530(e)(1)(ii))**

Lists captured data bearing on whether the conduct was the direct result of failure to implement the IEP. Pulled from:

- D-CTX-04 (IEP service delivery context — verbatim)
- D-CTX-03 (instructions and expectations clarity — pedagogical fit)
- D-ENV-03 (staff interaction and perceived respect)
- D-SAF-03 (access to trusted adults — system safety)

If `state.profile.iepStatus !== "yes"`, this section renders a single line: "Student profile indicates no active IEP. Section B is not applicable." Do not omit the section heading entirely — the explicit non-applicability is itself audit evidence.

Heading text:
> "Section B — Evidence to inform whether the conduct was the direct result of failure to implement the IEP (34 CFR §300.530(e)(1)(ii)). The team — not this report — makes that determination."

### 8. Universal-purpose export adaptation

When `purpose === "universal"`, the PDF strips the MDR-specific Section A and Section B headers (those statutory frames don't apply outside an MDR context). The PDF instead renders:

- A "School-climate signals" block listing fields tagged with belonging or environment perceptions (D-BLG.*, D-ENV.*) — useful for PBIS aggregation.
- A "Restorative-readiness signals" block listing D-RPR.* responses.
- The disclaimer banner remains; the disclaimer text adapts per `purpose` (the third disclaimer in the array conditionally renders only the relevant statutory framing — for Universal it does not reference §300.530(e)).

### 9. Stubbed-purpose export adaptation

When `purpose` is one of the seven stubbed values (`504`, `fba`, `restorative`, `re_entry`, `bullying`, `tier2`, `student_requested`), the JSON export still produces the full payload (the architectural story is intact). The PDF lands on a clean panel: "Exports for this purpose are routed in the architecture but not yet rendered. The JSON record is downloadable. PDF rendering arrives in the post-demo build phase per the project roadmap." The "Coming Soon" gate already exists (BP1) — reuse its style.

### 10. Disclaimer banner — exact text

In the PDF top-of-report banner, render this block verbatim. Text copy is calibrated for legal protection against decision-replacement creep and aligns with the v4 existing legal-cite block at line ~1503. Use the existing `.legal-cite` CSS class so visual register matches.

```
This is not a Manifestation Determination.

The information captured in this interview is "relevant information" under
34 CFR §300.530(e). The determination of whether the conduct was caused by,
or had a direct and substantial relationship to, the student's disability,
or was the direct result of failure to implement the IEP, must be made by
a team of qualified professionals — including the parent or guardian.

Reliability flags travel with each captured field. Review them before any
captured response is treated as evidence.

Person-first language is preserved throughout. Data retention and FERPA
handling per district policy.
```

Citations footer (small type, directly below the disclaimer banner, when `purpose === "mdr"`):

```
Research basis: NICHD Investigative Interview Protocol (Lamb et al., 2008);
Polyvagal Theory (Porges, 2011); Mahler Interoception Curriculum (2017);
Hidden profile research (Stasser & Titus, 1985; Stasser & Stewart, 1992);
Cognitive bias in facilitated debriefing (Meguerdichian et al., 2024);
Adultification bias (Goff et al., 2014; Epstein et al., 2017); UDL Guidelines
3.0 (CAST, 2024); Hammond — Culturally Responsive Teaching and the Brain
(2015); MDR Facilitator Cross-Domain Synthesis (2026); MTSS/PBIS Student
Data Collection Gaps Inventory (2026).
```

For Universal purpose, the citations footer drops the adultification and MDR-specific references and emphasizes the climate / belonging research base instead.

### 11. Suggestive-risk surfacing in the PDF

Per BP2's contract, fields where `provenance.probe_count >= 3` carry `reliability_flags: ["suggestive_risk"]`. In the PDF:

- The field renders with an amber rule above its block.
- A small inline note: "Multiple probes used — review for suggestibility before treating as evidence."
- The verbatim student response is shown intact (never redacted) — the flag qualifies, never deletes.
- The PDF top-of-report quality summary block lists the count of fields with `suggestive_risk` so the team sees it at a glance.

This is a structural protection, not a conditional one. Per Meguerdichian et al. (2024), cognitive biases in MDR facilitation include confirmation, anchoring, halo/horn — surfacing the suggestive-risk count up-front interrupts the "this answer must be true because the student said it three times" failure mode.

### 12. Universal aggregate placeholder (architectural shell only)

When `purpose === "universal"` and the `state.profile.school_id` field is present (it isn't for the demo — placeholder for future), the JSON export should support being merged into a building-level aggregate. Add a top-level `"aggregation_eligible": true` flag in the payload when purpose is universal. Demo does not implement aggregation; this flag exists so the PBIS climate aggregator (post-demo) can identify eligible records.

---

## Constraints (non-negotiable)

1. **Single HTML file. Vanilla JS. Tailwind via CDN. Web Crypto API for SHA-256. localStorage for persistence.** No new dependencies. No PDF library — the PDF is generated via the browser's native print-to-file path.
2. **Pure DOM construction via `h()`.** No `innerHTML`, no template strings concatenated into HTML.
3. **The export does not produce a determination.** Nothing in the JSON or PDF tells the team what to conclude about manifestation, intent, voluntariness, IEP-implementation failure, or discipline. The exports organize evidence; the team decides.
4. **Reliability flags travel with every field.** They are never stripped at export. Downstream consumers' contract is to honor them or quarantine the field; either is acceptable, ignoring is not.
5. **Provenance metadata is never silently dropped at export.** Every captured response in the JSON carries its full provenance block.
6. **Person-first, asset-based language throughout the PDF.** "Student with a disability." "Student who uses AAC." Home language as resource. Disability as identity, not deficit.
7. **The disclaimer banner ships verbatim as written above.** Do not edit the legal language; it is calibrated for protection against decision-replacement creep.
8. **The citations footer ships verbatim** for the MDR purpose; the variant for Universal is described in Section 10.
9. **JSON schema version is `0.9.0-beta`** for this demo. Locks at `1.0.0` after the Manifesto v2 team answers the open questions in `04_EVIDENCE_MAPPING_STARTER.md` Part 6.
10. **SHA-256 integrity hash is computed over the payload with `integrity.sha256_of_payload === null`** so downstream consumers can re-verify by zeroing that field and re-hashing.
11. **Field IDs are immutable.** The export uses only IDs declared in `FIELD_REGISTRY`. New IDs require updating the registry first.
12. **No new dependencies, no external network calls at export time.** All processing is local.

## Do NOT touch in this prompt

- The existing v4 print-render path at line ~1386. **Extend it; do not replace it.** The existing legal-cite, evidence-collection header, and observation log integration all stay.
- The conditional-logic engine from BP1. The export reads from current state; the engine isn't called at export time.
- The three interviewer icons from BP2. Their behavior is unchanged. Their provenance contributions are read.
- The student scaffold cards — Build Prompt 3 territory.
- D-PEER question content — Build Prompt 4.
- Existing equity gates and pre-flight gates. Export is gated only by completion of pre-flight (cannot export before pre-flight is satisfied — same gate as the rest of the interview).

## Acceptance criteria

A reviewer can verify each without running a full interview, using existing localStorage state from a test interview:

1. The new "Export evidence record" panel appears below the observation log.
2. "Download JSON" produces a `.json` file named `svp-<purpose>-<interview_id>.json`.
3. The downloaded JSON has every key in the `buildExportPayload` shape: `schema_version`, `framework`, `tool`, `tool_version`, `interview_id`, `generated_at_utc`, `purpose`, `purpose_label`, `interviewer`, `student_profile`, `incident_context`, `domains`, `five_ps_routing`, `quality_summary`, `interviewer_observations`, `disclaimers` (array of three strings), `integrity.sha256_of_payload` (a 64-character hex string).
4. The integrity hash, when re-computed locally over the payload-with-hash-zeroed, matches the value in the file.
5. The `domains` block contains every domain ID with `was_surfaced` reflecting the conditional engine's decision; unanswered fields show `value: null`.
6. The `five_ps_routing` block routes every captured response according to `FIVE_PS` membership (a field can appear under multiple Ps).
7. The `quality_summary` block reports `total_fields_captured`, `rephrase_invocations`, `probe_invocations`, `fields_with_suggestive_risk` correctly.
8. For `purpose === "universal"`, the JSON's `student_profile` is de-identified (no name, no DOB, no primary language string).
9. For `purpose === "universal"`, the JSON has `aggregation_eligible: true`.
10. "Generate PDF" enters the browser print path and the printed page renders the disclaimer banner verbatim, the citations footer (MDR or Universal variant per `purpose`), the identification block, the profile + incident block, the quality summary, all surfaced domains with provenance footers, the 5 Ps block, and (for MDR) Sections A and B.
11. For `purpose === "mdr"`, the PDF renders Section A and Section B headings with the verbatim §300.530(e) framing. Section B renders the not-applicable line if `iepStatus !== "yes"`.
12. For stubbed purposes, "Generate PDF" lands on the Coming Soon-styled rendering panel; "Download JSON" still produces a complete payload.
13. Fields with `reliability_flags` including `"suggestive_risk"` render in the PDF with the amber rule and the inline note. The quality summary's `fields_with_suggestive_risk` count appears in the top-of-report block.
14. Verbatim student responses are never redacted, paraphrased, or auto-summarized. They appear exactly as captured in `state.responses[<id>].value`.
15. The machine-readable footer block at the bottom of the PDF contains `interview_id`, `schema_version`, `tool_version`, and the SHA-256 hash.
16. No `innerHTML` introduced anywhere in the new code.
17. No external network calls during export.
18. No regression in any v4 capability or in any feature delivered by BP1 or BP2.

## When you finish

Reply with:

- A summary of files changed and lines added/modified.
- A confirmation that `buildExportPayload(state)` is the single source of truth for both JSON and PDF.
- The exact disclaimer banner string as it ships, verbatim, so we can confirm calibration.
- Any places where you had to make a judgment call not covered by this prompt.
- Any acceptance criterion you could not satisfy and why.
- Any new ambiguity surfaced for future build work.

**Do not improvise scope.** If the §300.530(e) framing or the disclaimer banner feels like it needs editorial change, flag it — do not edit it. The legal calibration is load-bearing and will be reviewed by counsel pre-pilot.
