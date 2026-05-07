# Build Prompt 2 — Three Interviewer Icons + Content System

**Paste this entire file into your coding agent as a single message. Working directory: the project folder where `student_voice_protocol_v4.html` lives. Build Prompt 1 should be merged before running this prompt.**

**Paste-artifact note.** If your editor converts dot-separated identifiers like `state.responses`, `q.prompts`, `purpose.id`, or `ICON_CONTENT.info.research` into nested markdown autolinks, strip them before running. The intent is plain identifiers, not URLs.

---

## Context

You are continuing work on the **Manifesto Student Voice Protocol (SVP)** — Appendix D within Manifesto, also a standalone school-wide post-incident interview tool. Build Prompt 1 added the purpose selector, incident intake, conditional-logic engine, and field registry. **Build Prompt 1 must be merged into the file before this prompt runs.**

This prompt adds three per-question interviewer affordances on every interview question card: **Info ("Why am I asking this?"), Rephrase ("Ask it another way"), and Probe Deeper ("If the answer is thin")**. These join the existing Listen / Record / Clear buttons. The new icons are interviewer-facing (the student does not see them); the content behind each icon is research-grounded and developmentally calibrated.

Per the wiring principle, the icon component is built once. Content lives in a data object so authoring or revising icon copy later is content work, not engineering work.

---

## Files

- The single HTML file in your working directory is `student_voice_protocol_v4.html` (now at v4.1 after Build Prompt 1, ~1,543 lines + the Build Prompt 1 additions).
- Read the file fully before changing anything. Pay particular attention to `renderQuestion(q)` near line 1124 — it's the function you're extending.

---

## Architectural facts (carryover from Build Prompt 1)

- Single-page scroll architecture; sections render top-to-bottom.
- Pure DOM construction via `h(tag, props, ...children)` at line 913 — `innerHTML` is never used. Every new component must use `h()`.
- Use `clear(node)` at line 936 to empty containers before re-rendering.
- Data lives in top-level `const` arrays/objects; render functions iterate over those.
- Material Symbols icons via the `icon(name, size)` helper at line 931.
- Existing v4 question card structure (line 1124–1167):
  ```
  .question-card
    .q-header (id label + qtype chip)
    .q-prompt (the band-specific question text)
    [optional .warning card if q.warning]
    .q-tools (Listen / Record / Clear buttons)
    textarea (ans-<field-id>)
    .q-routing ("Routes to → ...")
  ```
- Existing state in `state.responses[<id>]` (added in Build Prompt 1) carries provenance metadata fields including `rephrase_count` and `probe_count`. This prompt populates them.

---

## Goal

Add three interviewer-facing icon buttons to every question card. Each button opens a small inline panel directly below the question card with the appropriate content. Content is data-driven, age-band-aware where applicable, and respects every research and equity constraint baked into the rest of the tool.

---

## What to build

### 1. Three new buttons in the `.q-tools` row

Extend `renderQuestion(q)` so the `.q-tools` row contains, in order:

1. Listen (existing — keep as-is)
2. Record (existing — keep as-is)
3. Clear (existing — keep as-is)
4. **Info** — Material Symbols icon `info`, label "Why am I asking?"
5. **Rephrase** — Material Symbols icon `autorenew`, label "Ask it another way"
6. **Probe** — Material Symbols icon `psychology_alt`, label "Probe deeper"

Use the existing `.btn-ghost` class for visual consistency. Each new button gets `aria-expanded` (initially "false") and `aria-controls` pointing to the panel ID it toggles.

The three new buttons each toggle one of three inline panels described below. **Only one panel can be open at a time per question.** Opening a different panel on the same question closes the others. Opening any panel on a different question closes whatever was open on the previous question.

### 2. Three inline panels per question

After the existing `.q-routing` line (so the panels live below the routing breadcrumb), append three sibling panels, each with `hidden` attribute set initially:

```
<div class="icon-panel info-panel"     id="panel-info-<id>"     hidden>...</div>
<div class="icon-panel rephrase-panel" id="panel-rephrase-<id>" hidden>...</div>
<div class="icon-panel probe-panel"    id="panel-probe-<id>"    hidden>...</div>
```

Add a single CSS rule (inline style block in `<head>` is fine — match how existing v4 styles are organized):

```css
.icon-panel { border:2px solid black; background:#FAFAF7; padding:14px 18px; margin-top:10px; }
.icon-panel h4 { font-family:'Newsreader',serif; font-weight:700; font-size:.95rem; text-transform:uppercase; letter-spacing:.06em; margin-bottom:8px; }
.icon-panel p { font-size:.9rem; line-height:1.55; }
.icon-panel ul { font-size:.9rem; line-height:1.5; margin-top:6px; padding-left:18px; }
.icon-panel li + li { margin-top:6px; }
.icon-panel .panel-meta { font-size:.7rem; text-transform:uppercase; letter-spacing:.06em; color:#5A5A55; margin-top:10px; }
.icon-panel.placeholder { background:#FFF8E5; border-color:#7A6A2A; }
```

Each panel includes a small "Close" button (icon `close`, text "Close") that toggles the panel hidden and sets the matching tool-row button's `aria-expanded` back to "false".

### 3. ICON_CONTENT data structure

Declare near the existing top-level `const` blocks. Keyed by field ID (matches `FIELD_REGISTRY`). Shape:

```js
const ICON_CONTENT = {
  "D-CTX-01": {
    info: {
      construct: "Temporal context — open narrative invitation",
      why: "Opening with an open invitation gives the student narrative authority and produces the highest-quality recall under NICHD evidence. The temporal frame anchors later questions in the student's own sequence rather than the adult's reconstruction.",
      watchFor: "Resist the urge to interrupt or compress the timeline. If the student loops back or jumps ahead, follow them — do not redirect to chronological order.",
      research: ["NICHD Investigative Interview Protocol (Lamb et al., 2008) — narrative-recall reliability"]
    },
    probe: [
      "Tell me more about that morning.",
      "What happened next?",
      "Help me understand what you mean by [echo a specific phrase the student used]."
    ]
  },
  // ... one entry per field ID — full content below
};
```

Notes:
- **Rephrase content is not stored in `ICON_CONTENT`.** It is derived at panel-open time from `DOMAINS[<domain>].questions[<i>].prompts` — the existing HS / MS / UE band prompts ARE the rephrase variants. The Rephrase panel shows the two bands the interviewer is NOT currently on, with band labels.
- `info.research` is an array of citation strings displayed in the panel as a small footnote-style list under the heading "Research basis."
- `probe` is an array of 2–4 NICHD-compliant follow-up entries specific to this question. **Each probe entry is an object: `{ text: "...", kind: "open" | "cued" | "directive" | "option_posing" }`.** The Probe panel renders entries grouped by kind in NICHD hierarchy order (see Section 6).
- The placeholder version (for conditional domains and any unauthored field) sets `info.draft: true` and `info.placeholderNote: "Content from research synthesis pending."` The panel renders with the `.placeholder` CSS variant and shows the placeholder note instead of authored copy. This makes the demo's stubbed state explicit to stakeholders.

### 4. Rephrase panel — derived from `q.prompts`

When the interviewer clicks Rephrase on a question, the panel shows:

- Heading: "Ask it another way"
- A short directive: "If the student didn't engage, try a different developmental band. Do not compound — pick one and use it whole."
- Up to two cards, one per non-current band, each labeled with the band name (HS / MS / UE) and showing the band's prompt verbatim
- Each card has a "Use this version" button. Clicking it: (a) updates the visible prompt in the question card to the chosen band's wording, (b) does NOT change the global `currentBand` (this is a per-question rephrase, not a global age-band switch), (c) increments `state.responses[<id>].provenance.rephrase_count` by 1, (d) closes the panel.

If the global `currentBand` is HS, the panel shows MS and UE. If MS, shows HS and UE. If UE, shows HS and MS.

### 5. Info panel — authored content

When the interviewer clicks Info, the panel shows:

- Heading: "Why am I asking this?"
- Three short paragraphs corresponding to `info.construct`, `info.why`, `info.watchFor`
- Footer: "Research basis" followed by a bulleted list of `info.research` strings

Word budget: total visible text under 100 words. Designed to be read in 10–15 seconds.

If `info.draft === true`, render the panel with the `.placeholder` CSS variant and the placeholderNote in place of authored content.

### 6. Probe panel — NICHD hierarchy

When the interviewer clicks Probe, the panel renders entries grouped by `kind` in NICHD invitational hierarchy order. The grouping is the pedagogy — interviewers see the highest-quality probe options first and option-posing only at the bottom under explicit risk framing.

- Heading: "Probe deeper — NICHD invitational hierarchy"
- Directive (small intro line): "Try open invitations first. Pick one. Never compound. Echo the student's own words when possible."
- **Group A — Open invitations:** all entries where `kind === "open"`. Subhead: "Open invitations (highest quality)."
- **Group B — Cued invitations:** all entries where `kind === "cued"`. Subhead: "Cued (echo the student's own language)."
- **Group C — Directive:** all entries where `kind === "directive"`. Subhead: "Directive (specific factual question)."
- **Group D — Option-posing (last resort):** all entries where `kind === "option_posing"`. Subhead: **"If the answer is still thin — option-posing carries suggestibility risk."** Render this group with a thin amber rule above it for visual de-emphasis.
- Empty groups render no subhead and no separator (don't show empty headings).
- Footer: "When the student offers nothing further, accept the silence. 'I don't know' is data."
- Each list item has a small "Used this probe" link/button. Clicking increments `state.responses[<id>].provenance.probe_count` by 1, fades the item briefly (visual feedback), and does NOT close the panel — interviewers may use multiple probes in sequence.

If the field has no authored probes (placeholder), render the placeholder variant.

**Probe-tagging rule (applies when authoring or revising probe content):**
- `open` — universal invitation that accepts any continuation. Patterns: "Tell me more about X," "What was that like," "What else," "Help me understand."
- `cued` — invitation that echoes specific language the student already used. Patterns: "Tell me more about [echoed phrase]."
- `directive` — neutral specific question (who / what / where / when / how) without implying an answer or introducing concepts the student has not raised.
- `option_posing` — yes/no, two-option binary, or "or"-disjunction question. Permitted only as last-resort follow-up. Even option-posing cannot ship if it is **suggestive** (introduces content the student has not raised, or implies a specific answer).

### 7. Provenance wiring

When a Rephrase variant is selected, increment `state.responses[<id>].provenance.rephrase_count`. Initialize the response shell if not yet present (mirror the shape from Build Prompt 1 Section 8).

When a Probe variant is used (clicked), increment `state.responses[<id>].provenance.probe_count`.

If `provenance.probe_count >= 3` on a single field, set `state.responses[<id>].reliability_flags` to include `"suggestive_risk"` (idempotent — don't add it twice). Show a small warning chip on the question card: "Multiple probes — review for suggestibility before using as evidence." This implements the export-schema reliability flag (per BP2 contract) — the `"suggestive_risk"` flag string is the contract.

### 8. Conditional domains — placeholder content

For all fields in D-BLG, D-RNT, and D-PEER, ship the placeholder variant with these strings:

```js
const PLACEHOLDER_INFO = {
  draft: true,
  construct: "Pending research synthesis",
  placeholderNote: "Detailed Info content for this conditional domain field will be authored after research synthesis (research-bank prompts §A.6, §A.7, §E.1, §E.2)."
};
const PLACEHOLDER_PROBES = [];  // probe panel renders the placeholder variant when probes is empty
```

This makes the demo's stubbed state visible to stakeholders without leaving blank UI.

### 9. Authored content for all 28 mandatory question fields

Populate `ICON_CONTENT` with the following authored entries. **Word budget per Info card: 100 words.** **Probe arrays: 2–4 entries each, every entry tagged with its `kind` per the NICHD hierarchy.** All copy uses person-first, asset-based language; trauma-informed framing; equity safeguards where relevant.

**Probe shape and tagging — apply these rules when reading the authored probes below.** Probes appear below as bare strings for readability. **You must convert every probe entry to the `{ text, kind }` shape using these heuristics, in this order:**

1. Starts with "Tell me more," "Help me understand," echoes a bracketed `[echoed phrase]` token, or directly references something the student said → `kind: "cued"`.
2. Starts with "What," "How," or otherwise asks for open-ended continuation without yes/no framing → `kind: "open"`.
3. Asks Who / Where / When / Which neutrally without implying an answer or introducing a specific concept → `kind: "directive"`.
4. Yes/no, two-option binary, or "or"-disjunction (including "Was…?", "Did…?", "Could…?", "Were…?") → `kind: "option_posing"`.
5. Two specific entries below are explicitly tagged objects — preserve their tags as written (D-EMO-02 and D-UTD-03 — both rewritten from earlier drafts to comply with the suggestibility constraint). Do not re-tag those.

If a probe could fit two kinds, prefer the more conservative tag (e.g., a "What" question that also functions as option-posing → `option_posing`). The Probe panel's grouping in Section 6 then renders them in NICHD-hierarchy order.

```js
// =========== D-CTX (Context & Antecedents) ===========

"D-CTX-01": {
  info: {
    construct: "Temporal context — open narrative invitation",
    why: "Opening with an open invitation gives the student narrative authority and produces the highest-quality recall under NICHD evidence. The temporal frame anchors later questions in the student's own sequence rather than the adult's reconstruction.",
    watchFor: "Resist the urge to interrupt or compress the timeline. If the student loops back or jumps ahead, follow them — do not redirect to chronological order.",
    research: ["NICHD Investigative Interview Protocol (Lamb et al., 2008)", "Orbach et al. (2000) — narrative-rapport effect"]
  },
  probe: [
    "Tell me more about that morning.",
    "What happened next?",
    "Help me understand what you mean by [echo a specific phrase].",
    "What else?"
  ]
},

"D-CTX-02": {
  info: {
    construct: "Immediate antecedent — directive, narrow",
    why: "After the open narrative, narrow to the moments immediately preceding the incident. This isolates the proximal trigger without leading the student to a hypothesis.",
    watchFor: "Do not propose what the antecedent might have been. Let the student name it. Note who was present in their words, not yours.",
    research: ["NICHD Protocol — directed sub-prompts after open invitation", "FBA antecedent-behavior-consequence chain"]
  },
  probe: [
    "Right before that, what was happening?",
    "Who else was there?",
    "What were they doing?"
  ]
},

"D-CTX-03": {
  info: {
    construct: "Instructions and expectations — clarity check",
    why: "If the student couldn't access the expectation, the behavior may be a manifestation of unclear instruction rather than refusal. This is a high-signal modifiable setting event for FBA.",
    watchFor: "Listen for whether the student understood the task at all, not whether they 'should have.' Adultified students are over-attributed comprehension.",
    research: ["Universal Design for Learning (CAST, 2024) — representation principles", "IDEA: Adverse effect on educational performance"]
  },
  probe: [
    "Were the instructions written, said out loud, or both?",
    "Were you sure what was being asked?",
    "Was anything confusing about the task?"
  ]
},

"D-CTX-04": {
  info: {
    construct: "IEP service delivery context — fidelity check",
    why: "Failure to deliver an IEP-mandated service at the time of the incident is a separate IDEA violation independent of any behavioral analysis. This question surfaces constructive removal patterns.",
    watchFor: "If the student says the service was missing, document verbatim. Do not soften. This may invoke 34 CFR §300.530(e)(1)(ii).",
    research: ["IDEA §300.323(d) — IEP implementation", "Endrew F. v. Douglas County (2017) — meaningful benefit standard"]
  },
  probe: [
    "Was your [paraprofessional / co-teacher / accommodation] there?",
    "Did you get [extended time / quiet space / movement break] when you needed it?",
    "If not, what happened instead?"
  ]
},

"D-CTX-05": {
  info: {
    construct: "Perceived control — autonomy and agency",
    why: "Behavior under perceived coercion is qualitatively different from behavior under perceived choice. The student's experience of agency is the relevant variable, not whether choice 'objectively' existed.",
    watchFor: "Do not argue with the student's experience. If they felt they had no choice, that is the data. Whether you agree with that perception is a separate question.",
    research: ["Self-Determination Theory (Deci & Ryan, 2000)", "Coercion-resistance literature in disability rights"]
  },
  probe: [
    "Did it feel like you had to do it?",
    "Was there anything else you felt you could have done?",
    "What would have helped it feel like a choice?"
  ]
},

// =========== D-PHYS (Physiological State) ===========

"D-PHYS-01": {
  info: {
    construct: "Pre-incident physiological baseline — interoception",
    why: "Polyvagal theory frames the body's autonomic state as the substrate for behavior. Pre-incident physiology often reveals the student was already past their window of tolerance — meaning behavior was a survival response, not a choice.",
    watchFor: "Use Tier 1 vocabulary first (Mahler). Wait 10 seconds. If the student is silent, offer Tier 2 from the reference table — present as a menu, not a suggestion.",
    research: ["Polyvagal Theory (Porges, 2011)", "Mahler Interoception Curriculum (2017)"]
  },
  probe: [
    "Tell me more about your body right then.",
    "Was anything happening in your chest? Your stomach? Your hands?",
    "What did your breathing feel like?"
  ]
},

"D-PHYS-02": {
  info: {
    construct: "Physiological changes during the incident — autonomic shift",
    why: "Documenting an autonomic shift (sympathetic activation, dorsal shutdown) provides direct evidence of dysregulation that bears on whether the behavior was within the student's voluntary control at that moment.",
    watchFor: "If the student describes feeling 'frozen' or 'far away,' that is dorsal shutdown — not defiance. Cool Pose and polyvagal freeze can look like 'attitude' to an untrained adult.",
    research: ["Porges (2011) — autonomic state taxonomy", "Majors & Billson (1992) — Cool Pose"]
  },
  probe: [
    "What changed in your body?",
    "Did your body feel faster or slower?",
    "Did you feel tight or loose? Hot or cold?"
  ]
},

"D-PHYS-03": {
  info: {
    construct: "Action impulses and regulation capacity — fight/flight/freeze",
    why: "Naming the urge separates the impulse from the action. A student who felt a flight urge but stayed is regulating; a student who acted on a fight urge was overwhelmed by it. Both data are relevant.",
    watchFor: "Avoid 'why did you...' — that question framing assumes choice the student may not have had. Stay with the impulse, not the verdict.",
    research: ["Polyvagal action systems", "Siegel (1999) — window of tolerance"]
  },
  probe: [
    "What did your body want to do?",
    "Did part of you want to run or hide?",
    "Did part of you want to push back or fight?"
  ]
},

"D-PHYS-04": {
  info: {
    construct: "Recovery timeline — autonomic regulation",
    why: "How long the dysregulation lasted predicts re-entry timing. Asking a student to engage in restoration before they have returned to ventral vagal capacity sets them up to fail.",
    watchFor: "If the student says they are still activated, re-entry conversations should wait. Document the timeline; flag for the team.",
    research: ["Porges — co-regulation and ventral return", "Trauma-informed re-entry literature"]
  },
  probe: [
    "When did your body start to feel normal again?",
    "Are you all the way back, or still a little wound up?",
    "What helped your body settle?"
  ]
},

// =========== D-EMO (Emotional State & Affect Regulation) ===========

"D-EMO-01": {
  info: {
    construct: "Emotional state pre-incident — granularity",
    why: "Affect granularity (the ability to name specific feelings) is a protective factor. Students with limited granularity may say 'mad' for an emotion that was actually 'overwhelmed' or 'humiliated' — and the difference matters for FBA.",
    watchFor: "Don't rush past 'mad' or 'fine.' Offer Tier 2 vocabulary cards if the response is sparse. Do NOT propose specific feelings before the student names one.",
    research: ["Barrett (2017) — emotion granularity", "Mahler vocabulary tiers"]
  },
  probe: [
    "What kind of [feeling word] was it?",
    "Was it mostly inside you, or about something around you?",
    "Did it feel like one thing or a few things mixed?"
  ]
},

"D-EMO-02": {
  info: {
    construct: "Emotional intensity — overwhelm threshold",
    why: "Intensity calibrates the team's understanding of the student's window of tolerance. A 9/10 emotion in a previously-calm student suggests a triggering event; a 9/10 in a chronically-activated student suggests an overflow.",
    watchFor: "Use the student's own intensity scale if they offer one. Avoid forcing a 1–10 number on a student who prefers 'small / medium / really big.'",
    research: ["Kuypers Zones of Regulation (2011)", "Affect intensity research (Larsen & Diener, 1987)"]
  },
  probe: [
    { text: "How big did the feeling get?", kind: "open" },
    { text: "How does it compare to other big feelings you've had?", kind: "open" },
    { text: "Was it a small feeling or a big one?", kind: "option_posing" }
  ]
},

"D-EMO-03": {
  info: {
    construct: "Emotional cognition during incident — amygdala dominance",
    why: "If the student couldn't think clearly during the incident, prefrontal access was compromised. That is direct evidence relevant to manifestation analysis under IDEA — was the behavior a 'choice' if the deciding part of the brain was offline?",
    watchFor: "'I don't know what I was thinking' is meaningful data, not evasion. Document it verbatim.",
    research: ["LeDoux (1996) — amygdala hijack", "Goleman (2005) — emotional intelligence under arousal"]
  },
  probe: [
    "Could you think clearly?",
    "Did your mind go fast, slow, or blank?",
    "Did the feeling change while it was happening?"
  ]
},

"D-EMO-04": {
  info: {
    construct: "Shame and self-judgment — Critical Remorse Trap safeguard",
    why: "Absence of visible remorse is NOT evidence of willfulness. Flat affect, anger, and defensiveness are protective responses. Students with autism, ADHD, or trauma may have genuine difficulty accessing or expressing remorse even when they fully understand impact. Adultification bias amplifies this misread for Black students.",
    watchFor: "Do not score absence of remorse as non-manifestation. If the student is calm or angry rather than tearful, that is a regulation strategy, not evidence of cold intent.",
    research: ["Goff et al. (2014) — adultification of Black children", "Epstein et al. (2017) — Black girls' adultification", "v4 §D-EMO-04 Critical Remorse Trap Safeguard"]
  },
  probe: [
    "How are you feeling about it now, talking about it?",
    "Is it easier or harder to talk about now than it was right after?",
    "Is there anything you wish people understood about how you feel?"
  ]
},

// =========== D-ENV (Environmental Factors & Perceived Triggers) ===========

"D-ENV-01": {
  info: {
    construct: "Sensory and environmental contributors — accommodation needs",
    why: "Sensory overload is a setting event that often gets coded as defiance. Documenting the sensory environment surfaces modifiable accommodations the IEP or 504 team may not know are needed.",
    watchFor: "Loud, bright, crowded, unpredictable — name them as conditions, not as student weaknesses. Sensory profiles vary; the student's account is the data.",
    research: ["UDL Guidelines 3.0 (CAST, 2024) — variability principle", "Sensory Processing Disorder literature (Miller, 2014)"]
  },
  probe: [
    "Was the room loud?",
    "Were the lights or movement bothering you?",
    "Was anything happening that made it harder to focus?"
  ]
},

"D-ENV-02": {
  info: {
    construct: "Social-environmental factors — peer dynamics, belonging",
    why: "Peer-driven setting events (exclusion, ridicule, coercion) are systematically underweighted in adult-driven incident reports. The student's account is often the only direct evidence of these dynamics.",
    watchFor: "If the student names peer dynamics, this should trigger the D-PEER mini-domain (Build Prompt 4). For now, capture verbatim.",
    research: ["Walton & Cohen (2007, 2011) — belonging interventions", "Yeager et al. (2016) — adolescent belonging effects"]
  },
  probe: [
    "Were other kids saying or doing anything?",
    "Did anyone leave you out?",
    "Was anyone laughing at you or pushing you to do something?"
  ]
},

"D-ENV-03": {
  info: {
    construct: "Staff interaction & perceived respect — relationship quality",
    why: "Adult-student relationship quality at the moment of incident is a protective or perpetuating factor. A student who perceived disrespect from the adult in the moment will read identical adult interventions as escalation rather than co-regulation.",
    watchFor: "The student's perception is the data. This is not a search for whether the staff member 'actually' behaved disrespectfully.",
    research: ["Pianta (1999) — student-teacher relationship quality", "Hammond (2015) — Ready for Rigor frame: trust and rapport"]
  },
  probe: [
    "How were the adults acting?",
    "Did you feel listened to?",
    "Did anything they did make it worse?"
  ]
},

"D-ENV-04": {
  info: {
    construct: "Changes to routine — unexpected transitions",
    why: "Unpredictability is a setting event for many disabilities (autism, anxiety, ADHD). A 'small' schedule change for a neurotypical student can be a major destabilizer for a student with predictability needs.",
    watchFor: "Substitute teachers, fire drills, schedule shifts, visitors — name them. They rarely appear in adult incident reports because they 'shouldn't' have mattered.",
    research: ["Volkmar et al. (2014) — autism and predictability", "ADHD executive function literature"]
  },
  probe: [
    "Was anything different about that day?",
    "Did anything change unexpectedly?",
    "Was there a substitute or someone new?"
  ]
},

// =========== D-UTD (Understanding & Decision-Making) ===========

"D-UTD-01": {
  info: {
    construct: "Cognitive clarity during incident — prefrontal access",
    why: "If the student couldn't think clearly, the behavior was less voluntary than it appeared. This is the IDEA manifestation question in concrete neurological terms.",
    watchFor: "Do not lead with 'you knew what you were doing, right?' That is an interrogation question and produces compliance, not truth.",
    research: ["Diamond (2013) — executive function development", "Casey et al. (2008) — adolescent prefrontal-limbic balance"]
  },
  probe: [
    "Could you think about what to do?",
    "Did your brain feel fast or slow?",
    "Did anything feel automatic?"
  ]
},

"D-UTD-02": {
  info: {
    construct: "Awareness of consequences — capacity vs. choice",
    why: "Knowing the rule is different from being able to apply it under arousal. Students with executive function impairments may KNOW the consequence and STILL be unable to inhibit. This is the knowledge-capacity dissociation.",
    watchFor: "If the student says 'I knew I'd get in trouble,' that is not a confession of voluntariness — it is awareness without inhibitory access.",
    research: ["Barkley (2012) — ADHD and inhibition", "Casey et al. — limbic override under arousal"]
  },
  probe: [
    "Did you think about what would happen?",
    "Could you stop yourself?",
    "Did the part of you that wanted to stop feel loud or quiet?"
  ]
},

"D-UTD-03": {
  info: {
    construct: "Intentionality — volitional vs. impulsive",
    why: "Intent is the lay synonym for the technical question of voluntariness. Students often describe impulsive behavior as 'just happening' — that linguistic frame is itself evidence.",
    watchFor: "'I didn't mean to' is data, not evasion. Capture verbatim. Do not interrogate.",
    research: ["Anderson (2002) — prefrontal development and intentional control", "ADHD impulse control literature"]
  },
  probe: [
    { text: "What did you want in that moment?", kind: "open" },
    { text: "What were you hoping would happen?", kind: "open" },
    { text: "Did you mean to, or did it just happen?", kind: "option_posing" }
  ]
},

"D-UTD-04": {
  info: {
    construct: "Alternative behavior awareness — flexibility evidence",
    why: "Could the student see another option in the moment? If not, that's behavioral inflexibility under arousal — a hallmark of dysregulation. If yes, why didn't they pick it? That answer is the FBA gold mine.",
    watchFor: "If the student says 'I didn't think of anything else,' do not coach replacement behaviors right now. Note it for BIP development.",
    research: ["Cognitive flexibility literature (Diamond, 2013)", "FBA replacement behavior protocol"]
  },
  probe: [
    "Did you know other things you could have done?",
    "Did anything else feel possible right then?",
    "What would you try next time?"
  ]
},

// =========== D-SAF (Safety Context) ===========

"D-SAF-01": {
  info: {
    construct: "Perceived threat — survival response evidence",
    why: "Perceived threat (subjective, not 'objective') is what triggers autonomic survival responses. A student who felt unsafe was operating under threat physiology regardless of whether the adults thought there was 'real' danger.",
    watchFor: "Use the existing leading-question safeguard from v4 (the Normalization-then-Ask pattern). Do not start with 'were you scared?' — that assumes fear.",
    research: ["Porges (2011) — neuroception", "v4 §D-SAF Leading Question Self-Check"]
  },
  probe: [
    "What did your body tell you about danger?",
    "Did anything feel like you needed to protect yourself?",
    "What were you watching for?"
  ]
},

"D-SAF-02": {
  info: {
    construct: "Normalization-then-ask — leading-question safeguard",
    why: "This question is itself a fidelity safeguard. Asking it is part of the protocol's bias-mitigation architecture, not a separate construct.",
    watchFor: "Read the question verbatim. The normalization frame ('sometimes when people are overwhelmed...') is the safeguard. Don't paraphrase.",
    research: ["NICHD — leading question control", "v4 §D-SAF protocol design"]
  },
  probe: [
    "Did any of that fit?",
    "Was yours different — and if so, how?",
    "Tell me about what was true for you."
  ]
},

"D-SAF-03": {
  info: {
    construct: "Access to trusted adults — support system",
    why: "Whether the student had a trusted adult to access in the moment is a protective factor and a system-design indicator. Settings without trusted-adult access are higher-risk settings, regardless of incident history.",
    watchFor: "If the student names no trusted adult, this is a flag for the team — not a deficit in the student.",
    research: ["Rita Pierson — every kid needs a champion", "Search Institute — Developmental Assets"]
  },
  probe: [
    "Was there an adult you could have asked for help?",
    "Did anyone feel safe enough to go to?",
    "Why was it hard to ask?"
  ]
},

"D-SAF-04": {
  info: {
    construct: "Internal vs. external locus of safety — trust and autonomy",
    why: "When the student perceives that adults are not keeping them safe, they shift to self-protection. Behavior under self-protection looks 'oppositional' but is actually adaptive. This distinction is central to manifestation analysis.",
    watchFor: "If the student says they had to protect themselves, treat that as a serious signal about systemic safety, not as defiance.",
    research: ["Trauma-informed schools literature (Cole et al., 2005)", "Adverse Childhood Experiences (Felitti et al., 1998)"]
  },
  probe: [
    "Who was supposed to be keeping you safe?",
    "Did you trust the adults to handle it?",
    "Did you feel like you had to handle it yourself?"
  ]
},

// =========== D-RPR (Repair & Restoration) ===========

"D-RPR-01": {
  info: {
    construct: "Repair capacity and willingness — voluntary, not required",
    why: "Repair offered is data. Repair refused is also data — it does NOT indicate non-manifestation. Students with autism may struggle with the social script of apology; students with trauma may protect themselves through anger.",
    watchFor: "Do not coach an apology. Do not require a repair statement to 'count' as cooperation. Capture whatever the student offers, including 'no.'",
    research: ["Restorative Justice in Schools (Zehr, 2002)", "v4 §D-RPR Do Not Require Remorse safeguard"]
  },
  probe: [
    "Is there anything you'd want to say to [person]?",
    "Is there something you'd want them to know?",
    "Is there something you'd want to be different next time?"
  ]
},

"D-RPR-02": {
  info: {
    construct: "Future safety and success conditions — student voice in BIP",
    why: "The student is the foremost expert on what will help them succeed. This input feeds BIP and IEP modifications directly. Adult-designed plans without student voice have low durability.",
    watchFor: "Specifics matter more than aspirations. 'I want a different teacher' is actionable; 'I want to do better' is platitude. Probe for concrete supports.",
    research: ["Walton et al. (2021) — student voice in re-entry", "PBIS individual support planning"]
  },
  probe: [
    "What would help you feel okay at school?",
    "Who would you want on your team?",
    "What needs to change about the place or the people?"
  ]
},

"D-RPR-03": {
  info: {
    construct: "Closing — student voice catch-all",
    why: "Closing with a fully open invitation surfaces what no domain question reached. Students often place the most important data at the end, after they've decided you can be trusted with it.",
    watchFor: "Wait. Silence after this question is normal. Do not fill it with another question.",
    research: ["NICHD Protocol — closing invitation", "Trauma-informed interviewing literature"]
  },
  probe: [
    "Anything I missed?",
    "Anything I should have asked but didn't?",
    "What else matters that we haven't talked about?"
  ]
}
```

---

## Constraints (non-negotiable)

1. **Single HTML file. Vanilla JS. Tailwind via CDN. Material Symbols icon font (already loaded).** No new dependencies.
2. **Pure DOM construction via `h()`.** No `innerHTML`, no template strings concatenated into HTML.
3. **The icons are interviewer-facing only.** They do not appear when the page is rendered into the print/PDF view. Add `no-print` class to the new buttons and panels.
4. **NICHD probe hierarchy.** Each probe entry is tagged with its `kind` (`open` | `cued` | `directive` | `option_posing`). Open and cued invitations are highest quality. Directive prompts are acceptable. **Option-posing entries (yes/no, two-option binary, "or"-disjunction) are permitted only as last-resort follow-ups** and render under explicit suggestibility-risk framing in the Probe panel. **Suggestive probes — those that introduce content the student has not raised, or that imply a specific answer — do not ship under any kind tag.**
5. **Person-first, asset-based, trauma-informed language throughout.** Re-read every Info card with the question "Would I want this read aloud at a hearing about a 13-year-old I love?" If the answer is no, rewrite.
6. **The tool guides; it does not decide.** No Info card tells the interviewer what to conclude about manifestation, intent, voluntariness, or discipline. Info cards explain what to LOOK FOR; conclusions are the team's.
7. **Provenance counters auto-increment.** Rephrase use → `rephrase_count++`. Probe use → `probe_count++`. Three probes on a single field → `reliability_flags` includes `"suggestive_risk"` plus a visible warning chip on that question card.
8. **Conditional domain content ships as the placeholder variant** with explicit "Pending research synthesis" labeling, rendered with the `.placeholder` CSS variant so the stubbed state is visible to stakeholders.
9. **Word budgets respected.** Info card total ≤100 words. Probe arrays 2–4 entries.
10. **Each panel has accessible markup.** `aria-expanded` on the trigger button, `aria-controls` pointing to the panel, panel has `role="region"` and `aria-label` matching the heading. Close button has `aria-label="Close panel"`.

---

## Do NOT touch in this prompt

- **Existing v4 question prompts in `DOMAINS`** — do not edit any HS / MS / UE prompt strings. The Rephrase panel reads them; it does not rewrite them.
- **Existing Listen / Record / Clear behavior.**
- **Domain-level banners and warnings (`d.banners`, `q.warning`).** Those continue to render via the existing path and are NOT moved into the Info panel.
- **Student scaffold cards** — coming in Build Prompt 3.
- **D-PEER question content** — Build Prompt 4.
- **Export schema** — Build Prompt 5.
- **The `currentBand` global.** Per-question rephrase does NOT change the global age band; it only changes the visible prompt for the one question the interviewer is currently on.

---

## Acceptance criteria

A reviewer can verify each of these without running the app:

1. Every interview question card now shows three new buttons (Info, Rephrase, Probe) in the `.q-tools` row, alongside the existing Listen / Record / Clear.
2. Clicking any of the three new buttons opens an inline panel below the question card. Clicking again closes it. Opening a different one on the same question closes the others.
3. The Rephrase panel shows the two non-current age band variants (e.g., MS and UE if `currentBand === 'hs'`) reading directly from `q.prompts`.
4. Selecting a Rephrase variant updates the visible question prompt for that ONE question only — the global `currentBand` is unchanged.
5. Selecting a Rephrase variant increments `state.responses[<id>].provenance.rephrase_count`.
6. The Info panel shows the three-paragraph authored content from `ICON_CONTENT[<id>].info` plus a "Research basis" footer.
7. The Probe panel renders entries grouped by `kind` in NICHD hierarchy order: open invitations first, then cued, then directive, then option-posing under the explicit suggestibility-risk subhead. Empty groups render no subhead. Footer reads "When the student offers nothing further, accept the silence. 'I don't know' is data."
8. Clicking "Used this probe" increments `state.responses[<id>].provenance.probe_count`.
9. After three probe uses on a single field, `state.responses[<id>].reliability_flags` includes `"suggestive_risk"` and a visible warning chip appears on that question card.
10. All D-BLG, D-RNT, D-PEER fields render the placeholder variant with the "Pending research synthesis" note and the `.placeholder` CSS variant.
11. Print / PDF view does not show the new icons or panels (`no-print` class applied).
12. Accessibility: trigger buttons have `aria-expanded` correctly toggled; panels have `role="region"` and `aria-label`.
13. No `innerHTML` introduced anywhere.
14. No regression in any v4 capability or in any feature delivered by Build Prompt 1.

---

## When you finish

Reply with:
- A summary of files changed and lines added/modified.
- A count of `ICON_CONTENT` entries shipped (28 mandatory + 12 placeholder = 40 expected).
- Any places where you had to make a judgment call not covered by this prompt.
- Any acceptance criterion you could not satisfy and why.
- Any new ambiguity you surfaced that affects future prompts (Build Prompts 3–5).

Do not improvise scope. If something feels missing — particularly in the authored Info copy — flag it for content revision rather than rewriting on the fly. The next research synthesis pass (research-bank §E.1, §E.2, §E.3) may revise these strings; the architecture must make that revision content work, not engineering work.
