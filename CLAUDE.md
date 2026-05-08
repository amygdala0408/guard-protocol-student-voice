# Manifesto Student Voice Protocol (SVP)

> **Read this file in full before touching the codebase.** It captures load-bearing constraints, current state, the workflow we use, and the patterns proven to work in this repo. Do not rationalize skipping it.

## What this project is

A single-file HTML application (vanilla JS + Tailwind via CDN + Web Speech API + localStorage) that supports trained school staff in conducting trauma-informed, NICHD-aligned post-incident student interviews.

**Dual purpose by design:**

- **Standalone** — a school-wide post-incident interview tool, usable by counselors, mentors, case managers, and other trusted adults regardless of the student's IEP/504 status.
- **Appendix D within Manifesto** — SVP plugs into the Manifesto framework (formerly named GUARD) when an interview supports a Manifestation Determination Review (MDR).

**Stakeholder demo:** approximately May 9, 2026.

## Where everything lives

| | |
|---|---|
| Working folder | `~/Projects/guard-protocol/` |
| Main file | `student_voice_protocol_v4.html` |
| Build prompt archive | `build-prompts/` — every Build Prompt (BP1 onward) is committed alongside the code it produces |
| GitHub | https://github.com/amygdala0408/guard-protocol-student-voice (private) |
| Local preview | `cd ~/Projects/guard-protocol && python3 -m http.server 8765` then open http://localhost:8765/student_voice_protocol_v4.html |

## Load-bearing architectural constraints — do not break

1. **Single HTML file. Vanilla JS. Tailwind via CDN. Web Speech API for TTS/STT. localStorage for persistence.** No new dependencies.
2. **Pure DOM construction via `h(tag, props, ...children)` helper.** `innerHTML` is never used by deliberate design. Every new component must use `h()`.
3. **`clear(node)`** to empty containers before re-rendering. Use it; do not introduce alternatives.
4. **Data-driven rendering pattern.** Top-level `const` arrays drive render functions: `DOMAINS`, `BIAS_QUESTIONS`, `CULTURAL_QUESTIONS`, `VOCAB_TIERS`, `SAFETY_RUBRIC`, `FIVE_PS`, `INTERVIEW_PURPOSES`, `SETTINGS`, `IDEA_CATEGORIES`, `CONDITION_RULES`, `FIELD_REGISTRY`. New content is a data change, not a UI rebuild.
5. **Equity gates** use the `.equity-gate` CSS class. Reuse it for any new gate.
6. **Existing `answers` and `notes` objects stay intact.** BP1+ code writes to the `state` object alongside. A future Build Prompt may consolidate them — until then, do not rip out either.
7. **Field IDs are immutable.** The registry derives from `DOMAINS` (single source of truth) plus three reserved D-PEER IDs. If a new ID is needed, add it to `DOMAINS` (or the D-PEER reservation block) first.
8. **NICHD-aligned, age-banded prompts in `DOMAINS` are not rewritten.** Those are research-validated content.

## Equity is a forcing function (also load-bearing)

- "Are you the disciplining admin?" check at preflight — **hard block**, cannot proceed if yes.
- Bias-mitigation reflection — mandatory before the substantive phase.
- Cultural self-preparation — mandatory before the substantive phase.
- Stubbed purposes route to Coming Soon — **no "interview anyway" escape hatch**.

## The tool guides; it does not decide

Nothing in this codebase produces a determination, verdict, or recommendation about manifestation, discipline, or behavior. Data is captured for the team. The 5 Ps routing in the report is *organization* of evidence for team deliberation, not *interpretation* of it.

## Language standards

- **Person-first throughout.** "Student with a disability," not "disabled student." "Student who uses AAC," not "AAC student."
- **Asset-based.** Home language is a resource, not a deficit. Disability is a category of identity, not a problem to overcome.
- **Avoid:** "stakeholders," "best practices," "leverage," "robust," "holistic," "utilize."

## Current state

**Master is at v4.1 with BP1 + BP2 + BP5 merged.** PRs [#1](https://github.com/amygdala0408/guard-protocol-student-voice/pull/1), [#2](https://github.com/amygdala0408/guard-protocol-student-voice/pull/2), and [#3](https://github.com/amygdala0408/guard-protocol-student-voice/pull/3) are merged. The file is now ~3,928 lines.

**What BP1 added:**

- Framework rename GUARD → Manifesto (5 branding references)
- **Purpose Selector** with 10 options (2 active: `mdr`, `universal`; 7 stubbed; 1 passthrough → threat-pause bar)
- **Coming Soon** page for stubbed purposes (with Back button)
- **Gate-visibility pattern** — downstream content (`#gated-interview-content` wrapper) hidden until an active purpose is selected
- **Student Profile expansion:** DOB, Grade dropdown (K–12), age band display (auto-derived), IEP/504/ELL radios, primary language with `<datalist>` autocomplete (conditional on ELL=Yes), AAC/ASL flags, IDEA disability categories multi-select (conditional on IEP=Yes)
- **Incident Intake section:** incident date, time of day, setting (12 options — only `classroom` active; stubbed settings fall back to classroom with a dismissable banner), who present, behavior nature
- **Conditional-logic engine:** `evaluateConditions(state, ruleConfig)`, `isFlagOn(ruleKey)`, predicates `equals / notEquals / in / includesAny / truthy / falsy`, special placeholder `_alwaysFalseForNow`
- **D-PEER** 10th conditional mini-domain shell (3 IDs reserved: `D-PEER-01/02/03`; surfacing wired; question content lands in BP4)
- **FIELD_REGISTRY** derived from `DOMAINS` at module load via IIFE
- **`state` object** alongside legacy `answers` / `notes`, with full BP1 shape (`schema_version`, `framework: "Manifesto"`, `interview_id` UUIDv4, `interviewer`, `interview`, `profile`, `incident`, `responses`, `observations`)
- **Provenance metadata wrapper** for `state.responses` — `captureResponse(fieldId, value)` writes the v8-shaped entry with `age_band / purpose / setting / prompt_version / language / timestamp_utc` populated; other fields stay at defaults until BP2/3 wire them
- **Per-interview localStorage persistence** under `svp_state_<interview_id>`, with `svp_current_interview_id` as the load pointer; legacy `svp-draft` key preserved

**What BP2 added:**

- **Three interviewer-facing icon buttons on every question card:** Info ("Why am I asking?"), Rephrase ("Ask it another way"), Probe ("Probe deeper") — alongside existing Listen / Record / Clear. All `no-print` — excluded from PDF view.
- **Three inline panels per question** appended after `.q-routing` inside the card. Single-panel-open semantics globally — opening a new panel anywhere closes whichever was open. Tracked via module-scoped `currentlyOpenPanelId`. Panels use `role="region"` + `aria-label`; triggers carry `aria-expanded` + `aria-controls` + `data-icon-trigger="true"` (for blanket-reset on close).
- **`ICON_CONTENT` data structure** — 40 keyed entries (28 authored mandatory-domain + 12 placeholder for D-BLG/D-RNT/D-PEER). Looked up via `getIconContent(fieldId)` with placeholder fallback for unknown IDs. Authored Info cards respect ≤100-word budget.
- **NICHD probe hierarchy.** Each probe entry is `{ text, kind }` where kind ∈ `{ open, cued, directive, option_posing }`. The Probe panel renders entries grouped by kind in hierarchy order (open → cued → directive → option_posing). Empty groups suppress their subhead. The option_posing group renders inside `.probe-group-option-posing` with an amber-rule subhead reading *"If the answer is still thin — option-posing carries suggestibility risk."* This is the pedagogy: the panel itself trains the interviewer in the hierarchy.
- **`tagProbe()` heuristic IIFE** normalizes bare-string probes to `{ text, kind }` at module load. Priority order: cued patterns ("Tell me more" / "Help me understand") → option_posing yes/no openers (Was/Did/Could/Were/Are/Is/Have/etc.) → "or"-disjunction → bracketed `[echo]` cued → What/How/Why/Anything open → Who/Where/When/Which directive → fallthrough directive. Conservative tag wins on ambiguity. Pre-tagged objects (D-EMO-02, D-UTD-03) ship verbatim.
- **`captureResponse` refactored to merge-aware.** Previously replaced the entry on every keystroke, which would have zeroed out BP2's `rephrase_count` / `probe_count`. Now: if entry exists → preserve provenance counters and `reliability_flags`, only update `value` + always-fresh provenance fields. Added `defaultProvenance()` (single source of truth for the shape) and `ensureResponse(id)` (init-if-absent helper used by Rephrase/Probe handlers before the textarea is touched).
- **Rephrase panel** derives variants from `q.prompts[hs|ms|ue]` at open-time. Shows the two non-current bands. "Use this version" updates only the visible prompt text node (does NOT mutate global `currentBand`) and increments `provenance.rephrase_count` via `ensureResponse`. Per-question rephrase is a soft override — a subsequent global `setBand()` overwrites it (agreed behavior).
- **Probe panel** — clicking "Used this probe" increments `provenance.probe_count` (via `ensureResponse`) and fades the item briefly. At `probe_count >= 3`, idempotently pushes `"suggestive_risk"` into `state.responses[<id>].reliability_flags` and appends a `.q-chip.suggestive-warn` chip ("Multi-probe") to the card's `q-header` with title/aria-label *"Multiple probes — review for suggestibility before using as evidence."*
- **D-PEER reserved cards** (in `renderDPeerShell`) also get the three affordances. Info + Probe fall through to placeholder content; Rephrase renders its own placeholder ("Rephrase variants pending — D-PEER question content lands in Build Prompt 4") since `q.prompts` is absent until BP4. D-PEER cards now include empty `.q-tools` + a minimal `.q-routing` line as attachment scaffolding.
- **CSS additions** scoped under `.icon-panel`: panel base style, `.placeholder` cream variant, `.probe-group-option-posing` (amber rule + heading color), `.rephrase-card`, `.probe-item` (with `.used` fade), `.q-chip.suggestive-warn` (background `#FFF8E5`, color `#7A6A2A`).

**What BP5 added:**

- **`EXPORT_SCHEMA_VERSION = '0.9.0-beta'`** — top-level const, single source of truth for both `initState()` (sets `state.schema_version`) and `buildExportPayload()`. Locks at `1.0.0` after the Manifesto v2 team answers the open evidence-mapping questions.
- **`buildExportPayload(state)`** — canonical export shape consumed by JSON download AND PDF print path so the two surfaces never drift. Returns `{ schema_version, framework, tool, tool_version, interview_id, generated_at_utc, purpose, purpose_label, interviewer, student_profile, incident_context, domains, five_ps_routing, quality_summary, interviewer_observations, aggregation_eligible, disclaimers, integrity }`.
- **Helpers** — `lookupPurposeLabel(id)`, `lookupSettingLabel(id)`, `redactProfileForPurpose(profile, purpose)` (Universal strips `name`/`dob`/`primaryLanguage` and converts `iepStatus`/`section504Status`/`ellStatus` to `*_present` booleans; MDR keeps full profile), `fivePsForField(fid)`, `isCaptured(r)`, `buildResponseEntry(fid, reg)`, `buildDomainsBlock()`, `buildFivePsBlock()`, `buildQualitySummary()`, `buildDisclaimers(purpose)` (third disclaimer references §300.530(e) for MDR; Universal/stubbed drop that statutory framing).
- **Domains contract** — every domain (incl. D-PEER) keyed with `domain_id`, `domain_title`, `was_surfaced` (reflects the conditional engine — `isFlagOn('domain.<id>.visible')`), `conditional_basis` (rule key when gated, null when surfaced), and `responses: [{ field_id, field_label, qtype, value, provenance, reliability_flags, five_ps_routing }]`. **Unanswered fields ship with `value: null` and `provenance: null`** so consumers distinguish "not asked" from "asked but no response."
- **5 Ps contract** — `predisposing / perpetuating / precipitating / protective / prognostic`, each an array of `{ field_id, field_label, value, reliability_flags }` for *captured* responses only. Routing reflects `FIVE_PS` membership exactly — never invents.
- **Quality summary** — `total_fields_captured`, `total_fields_unanswered`, `rephrase_invocations` (sum of `provenance.rephrase_count`), `probe_invocations` (sum of `provenance.probe_count`), `fields_with_suggestive_risk`, `fields_with_dysregulated_response` (where `regulation_observation === 'red'`), `fields_interpreter_mediated`, `fields_aac_mediated`, `scaffold_tier_distribution: { tier1, tier2, tier3, none }`, `regulation_check_observations`, `pauses_taken`, `total_duration_seconds` (null until `completed_at_utc` gets set — currently never), `nichd_utterance_distribution` (null pre-BP4).
- **JSON download path** — `sha256Hex(str)` (Web Crypto API, returns 64-char lowercase hex), `triggerDownload(content, filename, mime)` (Blob + object URL + `h()`-built `<a download>`, no `innerHTML`), `downloadJsonExport()` (filename pattern `svp-<purpose>-<interview_id>.json`). **Hash contract:** SHA-256 is computed over the payload with `integrity.sha256_of_payload === null`. Downstream consumers verify by zeroing that field and re-hashing. Verified by round-trip in the BP5 sandbox test.
- **PDF print path** — `generateReport()` is now `async` (awaits sha256). Renders the verbatim disclaimer banner (`PDF_DISCLAIMER_BANNER_TEXT`, `.legal-cite` class, four paragraphs), research-basis citations (MDR variant `PDF_CITATIONS_MDR` ships verbatim; Universal `PDF_CITATIONS_UNIVERSAL` drops adultification + MDR-specific refs), identification block, profile + incident block (Universal-redacted), quality summary (with Meguerdichian (2024) suggestive_risk reminder), domains block (provenance footer per captured field — `band / tier / source / lang / rephrases / probes / timestamp`; `suggestive_risk` flagged with amber top rule + "Multi-probe — review for suggestibility" inline note), 5 Ps routing, v4-continuity findings, then purpose-specific sections.
- **MDR Section A** — disability-causation prong (§300.530(e)(1)(i)), populated from `MDR_SECTION_A_FIELDS` (`D-PHYS-01..03`, `D-EMO-01..04`, `D-UTD-01..04`, `D-CTX-01,02,05`, `D-ENV-01,02`). D-EMO-04 carries the inline `REMORSE_TRAP_NOTE` (Critical Remorse Trap safeguard with Goff/Epstein adultification citation). The note is *unconditional* because SVP doesn't capture ethnicity — judgment call, flag for counsel pre-pilot.
- **MDR Section B** — IEP-implementation prong (§300.530(e)(1)(ii)), populated from `MDR_SECTION_B_FIELDS` (`D-CTX-04, D-CTX-03, D-ENV-03, D-SAF-03`). If `state.profile.iepStatus !== 'yes'` the section heading remains and renders the explicit not-applicability line — the explicit non-applicability is itself audit evidence.
- **Universal sections** — `UNIVERSAL_CLIMATE_FIELDS` (D-BLG.* + D-ENV.*) and `UNIVERSAL_RESTORATIVE_FIELDS` (D-RPR.*). Universal exports also flag `aggregation_eligible: true`.
- **Stubbed-purpose path** — JSON export ships full payload (architectural story is intact). PDF lands on a Coming Soon panel + identification block + machine-readable footer so the JSON twin can still be verified. Coming Soon UI panel surfaces the export buttons too, so demo can show end-to-end routing without authored PDF rendering for those purposes.
- **Machine-readable footer** — fixed-width block at bottom of every printed PDF: `interview_id`, `schema_version`, `tool_version`, `sha256`. Links the printed PDF to its JSON twin.
- **New "Export evidence record" UI panel** — `.no-print`, lives below Final Observations and above Footer Action. Intro paragraph (verbatim per spec) + Download JSON (primary) + Generate PDF (secondary) + status line (`#export-status`).
- **`window.downloadJsonExport`** + **`window.buildExportPayload`** exposed for parity with `saveDraft` / `generateReport`.

**Verification artifacts:** runtime engine assertions for BP5 live at `/tmp/svp_test_bp5.js` (same Node `vm` module + stubbed browser globals pattern as BP2; Web Crypto polyfilled via Node's `crypto.createHash('sha256')` so the round-trip is exercised). **57/57 PASS** covering ACs #2, #3, #4, #5, #6, #7, #8, #9, #12. The PR #3 description preserves the test plan for the visual ACs (#1, #10, #11, #13, #14, #15) the user runs in-browser.

**Important contracts BP5 introduced (preserve in future BPs):**

1. `buildExportPayload` is the only path that produces export data. JSON serializes it; PDF reads from it. Never let them drift.
2. SHA-256 is computed with `integrity.sha256_of_payload === null` so consumers can re-verify by zeroing and re-hashing. Don't change this contract.
3. Reliability flags travel with every captured field. Never strip them at export.
4. Provenance metadata is never silently dropped at export.
5. The disclaimer banner ships verbatim. The text is calibrated for legal protection; counsel will review pre-pilot. Do not edit without explicit approval.
6. Field IDs are immutable (already enforced in BP1, reaffirmed here — exports use only `FIELD_REGISTRY` IDs).

**What's pending (Build Prompts to come):**

- **BP3** — Student scaffold cards for D-PHYS / D-EMO / D-ENV / D-UTD (will populate `provenance.tier_used` and `provenance.scaffold_viewed`). BP5's `quality_summary.scaffold_tier_distribution` already reads these once BP3 starts writing them.
- **BP4** — D-PEER question content (HS / MS / UE prompts, NICHD type, routing). When this lands, refactor `renderDPeerShell`'s reserved cards to flow through `renderQuestion()` (so they pick up Listen/Record/Clear/textarea), and remove the Rephrase placeholder branch in `populateRephrasePanel`. BP5's `quality_summary.nichd_utterance_distribution` is wired to null and should populate once BP4 has a captured-response → qtype tagger.

**Open ambiguity flagged for future BPs** (none blocking, but worth resolving when relevant):

- BP3 may want to revisit whether `applyDomainConditionals()` should run on every state mutation (currently called on `whoPresent` change + load) — scaffold-card surfacing logic could need a similar hook
- BP4 needs to author probe content for D-PEER-01/02/03 (placeholder entries already in `ICON_CONTENT`)
- BP4 should sanity-check D-PEER question types — registry has `Inv./Inv./Dir.`
- D-BLG and D-RNT need authored Info/probe content (still placeholder; defer to research-bank synthesis pass §A.6 / §A.7 / §E.1 / §E.2)
- D-CTX-04 hide-vs-disable choice — currently hidden when `iepStatus !== 'yes'`; alternative is "shown but inert"
- Threat-pause UX richness — currently a passive pointer to the existing bar; could be richer
- Repo rename when Manifesto rebrand finalizes (currently `guard-protocol-student-voice`)
- Disability multi-select unknown-code preservation on load
- Settings dropdown styling per `<option>` (HTML doesn't support per-option indicators natively)
- **BP5 carryover:** `state.interview.completed_at_utc` is never set — `quality_summary.total_duration_seconds` ships null until a future BP wires closure. Decide for v1.0.0 whether Generate PDF / Download JSON should set it, or whether closure is a separate explicit action.
- **BP5 carryover:** Universal citations footer wording is mine, not in BP5 spec verbatim. Counsel review pre-pilot.
- **BP5 carryover:** PDF disclaimer banner first paragraph rendered bold + non-italic (rest italic via `.legal-cite`). Counsel review pre-pilot — easy to revert.
- **BP5 carryover:** Critical Remorse Trap inline note is unconditional alongside D-EMO-04 (SVP doesn't capture ethnicity, so the safeguard surfaces the bias risk regardless). Adultification citation preserved.
- **BP5 carryover:** `aggregation_eligible: true` for any Universal export, regardless of `state.profile.school_id` (which doesn't exist in the demo state). Worth confirming when the PBIS climate aggregator lands.

## Workflow

1. **Read everything before touching code.** Specifically:
   - This `CLAUDE.md` in full
   - `student_voice_protocol_v4.html` in full (it's ~3,928 lines after BP5 — the architecture has to be in your head, not the line count)
   - `build-prompts/BUILD_PROMPT_1.md` — the template for how prompts get processed
   - The Build Prompt the user is currently pasting
2. **Branch per Build Prompt.** Create a feature branch before any edits (e.g., `interviewer-icons` for BP2). Master stays clean.
3. **Commit per logical chunk.** Each commit message has a one-line summary and a structured body explaining what changed and why. The BP1 commits in `git log` are the reference style.
4. **Surface judgment calls upfront** before executing. The user wants no surprises. Anywhere the prompt is ambiguous about exact placement, naming, or scope, flag it before you act.
5. **Verify before claiming done.**
   - **Static checks** — grep for required/forbidden strings, count entries.
   - **JS syntax check** — extract inline scripts and run `node --check`.
   - **Runtime engine assertions** — Node with stubbed browser globals to test `FIELD_REGISTRY` and `isFlagOn()` rule evaluations (see verification patterns below).
   - **Visual smoke test** — open in browser; the user does this part. Pre-write a checklist of what should be visible.
6. **Open a PR** with structured body: Summary, Verification, Judgment calls flagged, Files changed.
7. **Merge after user approval.** Use `gh pr merge --merge --delete-branch`.

## Working with the user

- **Do not improvise scope.** If something feels missing from a Build Prompt, flag it in the finish summary rather than building it.
- **Strip paste artifacts.** The user's Cowork chat sometimes pastes nested markdown autolinks like `[state.profile.name](http://state.profile.name)` — strip these to plain identifiers before executing. Always grep `'](http://'` after editing — should return 0.
- **Concise, structured output.** The user reads carefully and acts on what's there. Lead with the answer, support with detail, mark options clearly when she needs to choose.
- **Surface trade-offs, don't mask them.** When a build prompt has internal contradictions or could be interpreted multiple ways, name the contradiction and propose a resolution rather than picking silently.

## Verification patterns (proven)

**Static checks:**
```bash
cd ~/Projects/guard-protocol
grep -c "GUARD" student_voice_protocol_v4.html         # should be 0 post-rename
grep -c "Manifesto" student_voice_protocol_v4.html     # should match expected branding count
grep -c '](http://' student_voice_protocol_v4.html     # should be 0 (no autolink artifacts)
```

**JS syntax check:**
```bash
python3 -c "
import re
with open('student_voice_protocol_v4.html') as f: html = f.read()
scripts = re.findall(r'<script(?![^>]*src=)[^>]*>(.*?)</script>', html, re.DOTALL)
with open('/tmp/svp_extracted.js', 'w') as f:
    for s in scripts: f.write(s + '\n')
"
node --check /tmp/svp_extracted.js
```

**Runtime engine assertions** (Node with stubbed browser globals):

Pattern: extract the inline `<script>` blocks (skip the Tailwind config block — it references `tailwind.config` which isn't stubbed), append `Object.assign(globalThis, { state, buildExportPayload, ... })` so the test can reach `const`-bound names, then `vm.createContext` + `vm.runInContext` with stubbed `document`, `window` (incl. `crypto.subtle.digest` polyfilled via Node's `crypto.createHash('sha256')`), `localStorage`, `Blob`, `URL.createObjectURL`, `TextEncoder`. Then assert against any exported helper. Working templates:

- BP1 — 20/20 (ACs #4, #11, #12, #15, #16). Test in PR #1 description.
- BP2 — 62/62 (`ICON_CONTENT`, `tagProbe`, `getIconContent`, `captureResponse`, `ensureResponse`, `FIELD_REGISTRY`, `isFlagOn`). Test at `/tmp/svp_test_probes.js`. Pass-list in PR #2 description.
- BP5 — 57/57 (`buildExportPayload`, `buildDomainsBlock`, `buildFivePsBlock`, `buildQualitySummary`, `redactProfileForPurpose`, `sha256Hex` round-trip, stubbed-purpose JSON, filename pattern). Test at `/tmp/svp_test_bp5.js`. Pass-list in PR #3 description.

## Files in this repo

- `CLAUDE.md` — this file
- `student_voice_protocol_v4.html` — the entire application
- `build-prompts/BUILD_PROMPT_1.md` — BP1 spec, archived alongside the implementation
- `build-prompts/BUILD_PROMPT_2.md` — BP2 spec, archived alongside the implementation
- `build-prompts/BUILD_PROMPT_5.md` — BP5 spec, archived alongside the implementation
- `.gitignore` — keeps OS junk out of the repo

## Ground truth for the v4-aligned 9 domains + D-PEER

| ID | Domain | NICHD types | Conditional? |
|---|---|---|---|
| D-CTX-01..05 | Context & Antecedents (5 questions) | Inv. / Dir. / Dir. / Opt. / Dir. | mandatory; D-CTX-04 conditional on IEP=yes |
| D-PHYS-01..04 | Physiological State | Inv. ×3, Dir. | mandatory |
| D-EMO-01..04 | Emotional State & Affect Regulation | Inv. / Dir. / Inv. / Inv. | mandatory; D-EMO-04 carries Remorse Trap Safeguard |
| D-ENV-01..04 | Environmental Factors & Perceived Triggers | Inv. ×3, Dir. | mandatory |
| D-UTD-01..04 | Understanding & Decision-Making | Inv. / Dir. / Inv. / Dir. | mandatory |
| D-SAF-01..04 | Safety Context | Inv. / Opt. / Dir. / Dir. | mandatory; carries Safety Construct definition + Leading Question Self-Check banners |
| D-RPR-01..03 | Repair & Restoration | Inv. ×3 | mandatory; carries Do-Not-Require-Remorse safeguard |
| D-BLG-01..05 | Belonging & Community Connectedness | Inv. ×3, Dir. ×2 | conditional (placeholder rule, wired in later BP); D-BLG-05 is Equity Gate (Appendix R) |
| D-RNT-01..04 | Repair & Re-Entry | Inv. ×4 | conditional (placeholder rule, wired in later BP) |
| D-PEER-01..03 | Peer Dynamics (NEW mini-domain) | Inv. / Inv. / Dir. | conditional on `incident.whoPresent` includes peers; **content authored in BP4** |
