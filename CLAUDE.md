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

**Master is at v4.1.** BP1 is merged. The repo records 7 commits: initial v4 + BP1 spec archive + 4 logical BP1 implementation chunks + the merge commit.

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

**What's pending (Build Prompts to come):**

- **BP2** — Three interviewer icons (Info / Rephrase / Probe Deeper)
- **BP3** — Student scaffold cards for D-PHYS / D-EMO / D-ENV / D-UTD
- **BP4** — D-PEER question content (HS / MS / UE prompts, NICHD type, routing)
- **BP5** — Export schema (JSON + PDF)

**Open ambiguity flagged for future BPs** (none blocking, but worth resolving when relevant):

- D-PEER question types in BP4 — registry has `Inv./Inv./Dir.`; BP4 should sanity-check
- D-CTX-04 hide-vs-disable choice — currently hidden when `iepStatus !== 'yes'`; alternative is "shown but inert"
- Threat-pause UX richness — currently a passive pointer to the existing bar; could be richer
- Repo rename when Manifesto rebrand finalizes (currently `guard-protocol-student-voice`)
- Disability multi-select unknown-code preservation on load
- Settings dropdown styling per `<option>` (HTML doesn't support per-option indicators natively)

## Workflow

1. **Read everything before touching code.** Specifically:
   - This `CLAUDE.md` in full
   - `student_voice_protocol_v4.html` in full (it's ~2,300 lines but you need the architecture in your head)
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

Pattern: extract the inline `<script>` block containing `const DOMAINS`, evaluate it inside a function constructor with stubbed `window`, `document`, `localStorage`, `crypto.randomUUID`. Then assert against `FIELD_REGISTRY[id]` and `isFlagOn(ruleKey)` after mutating `state.profile` / `state.incident`. The Node test for BP1 (in PR #1 description and chat history) is the working template — it verified 20/20 assertions for ACs #4, #11, #12, #15, #16.

## Files in this repo

- `CLAUDE.md` — this file
- `student_voice_protocol_v4.html` — the entire application
- `build-prompts/BUILD_PROMPT_1.md` — BP1 spec, archived alongside the implementation
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
