---
name: manuscript-tour-authoring
description: Use this skill whenever a user wants to author, update, translate, or extend a guided web-page tour they can replay — keywords include "tour", "walkthrough", "demo", "onboarding", "product highlight", "visual guide", "step-by-step", "guided tour", "Manuscript scenario", "multi-page tour", "cross-site tour", "spotlight tour" — even if they don't say "Manuscript" by name. It generates one Manuscript scenario JSON file (schemaVersion 0.1.2) that the Manuscript browser extension (or its runtime bridge) replays with spotlight, narration, sub-element traversal, annotations (text / shape / arrow / freedraw), and cross-page resume. Prefer this skill over hand-writing JSON, drafting screencast scripts, or generating static HTML walkthroughs whenever a *replayable* tour is the actual goal.
---

# Manuscript Scenario Authoring — Agent Skill

> **Goal:** Inspect a web page (URL or HTML), pick the right elements,
> write narration, and emit a single JSON file. When this skill finishes,
> the user can hand the file to Manuscript and it plays a polished
> spotlight-and-narration tour with zero extra work.

## For the human reader (TL;DR)

Drop this file (and the URL you want toured) into any AI agent —
Claude, Cursor, Windsurf, Cline, GPT — and ask for a tour. The agent
follows the procedure below and returns one Manuscript scenario JSON
file (default: `tour.json`; name it whatever you like). Save it next
to the Manuscript runtime bridge, hit play. No DOM picking, no manual
JSON editing.

```
"Use SKILL.md. Generate a Manuscript tour for
 https://example.com/dashboard. English, 6 steps."
```

Expected response: **JSON only**, no commentary.

Everything below is written for the agent.

## For the agent — quick map

If you're in a hurry, read in this order:

- [§1 Output contract](#1-output-contract) — what to produce
- [§2 Procedure](#2-procedure-the-algorithm) — the algorithm
- [§10 Validation checklist](#10-validation-checklist) — verify before emitting
- [§12 Pseudocode](#12-generation-pseudocode) — the whole flow at a glance

The remaining sections are reference — read when you hit them:

- [§3 Inputs to ask for](#3-inputs-you-need-to-ask-for) · when context is thin
- [§4 Finding good targets](#4-page-analysis--finding-good-tour-targets) · while planning steps
- [§5 JSON schema](#5-json-schema-reference) · while emitting
- [§6 Selector chain (3-layer rule)](#6-selector-chain-authoring-the-3-layer-rule) · per target
- [§7 Annotation patterns](#7-annotation-patterns) · per step that needs labels/shapes
- [§8 Narration & pacing](#8-narration--pacing) · per step description
- [§9 Localization](#9-localization--same-selectors-different-language) · when generating multiple languages
- [§11 Worked example](#11-worked-example) · concrete model

---

## 1. Output contract

The skill produces **one JSON file per scenario** (one file per language
when multilingual).

```
<your-tour>.json   // schemaVersion === "0.1.2"
```

File name is up to the user. `tour.json`, `onboarding-en.json`,
`feature-launch.json`, `acme-pricing-ko.json` are all fine. When
emitting multiple languages, parallel names help: e.g.
`<slug>-en.json` / `<slug>-ko.json`. If the user doesn't specify,
default to `tour.json` (or `tour-<lang>.json` for multi-language).

Required:

* `schemaVersion: "0.1.2"` (literal — see "Schema version" note at end)
* `id: string` — stable kebab-case, unique within the user's library
  (e.g. `tour-acme-onboarding-en`)
* `name: string` — human title in the tour's language
* `url: string` — the canonical URL the scenario targets (the page
  the *first* step lives on). Steps may live on other URLs via
  per-step `pickedAtUrl` (§5.2, §8).
* `createdAt`, `updatedAt`: ISO-8601 / RFC 3339 string with `Z` suffix
  (e.g. `"2026-05-21T10:30:00Z"`). The extension parses with
  `new Date()`, so any valid ISO-8601 works; the trailing `Z` form
  keeps things unambiguous across time zones.
* `steps: Step[]` — 4–8 steps is the sweet spot

Each step requires `id`, `name`, `description`, `selectors`, an
`annotations` array (may be empty), `autoAdvanceMs`, and
`waitForNavigation`. Schema details are in §5.

The file must:

* Parse as valid JSON (no comments, no trailing commas).
* Use the literal string `"0.1.2"` for `schemaVersion`. Older `0.1.0`
  and `0.1.1` scenarios still load (migration runs automatically), but
  new files should be emitted at the current version.
* Be small enough to ship in a repo (under ~50 KB per file is normal;
  freedraw with many points or many sub-elements can push higher).

Anything outside the schema is **ignored** by the extension — don't
invent fields. Optional fields (e.g. `anchorOffset`, `siteFonts`,
`subElements`, `subDwellsMs`, `pickedAtUrl`) may be added when
meaningful.

---

## 2. Procedure (the algorithm)

Follow these phases in order.

**Phase A — Gather**
1. Read the target page (fetch HTML, render JS if needed, screenshot if
   you can).
2. Confirm the user's intent: tour purpose (onboarding? feature
   highlight? marketing tour?), the audience, and the language(s).
3. Note the page's heading hierarchy, primary CTA, distinct sections,
   and any obvious data-* attributes / ids / aria-labels.

**Phase B — Plan**
4. Pick **4–8 spotlight targets**. Order them so the camera moves
   roughly top-to-bottom or in narrative order (problem → solution →
   proof → CTA).
5. For each target, decide:
   - The element it spotlights (selector).
   - One narration sentence (read aloud during replay).
   - Whether to add an inline label (text annotation) or arrow.
   - The auto-advance time (5–8 seconds per step is normal; longer if
     narration is long).
   - Whether the step waits for a user click (`waitForNavigation`).

**Phase C — Selector pass**
6. For each spotlight target, build a `SelectorChain` with all three
   layers populated (see §6). Always fill all three — the chain only
   gracefully degrades when every layer is present.

**Phase D — Annotation pass**
7. Add text labels, shapes, or arrows only when they pull weight.
   Empty `annotations: []` is fine — spotlight + narration alone can
   carry a step.
8. Use `anchorOffset` for any annotation that should follow the
   spotlight element on window resize (§7).

**Phase E — Emit**
9. Assemble the JSON. Run the validation checklist (§10).
10. Output the JSON only. No surrounding prose unless the user asked
    for an explanation.

---

## 3. Inputs you need to ask for

If the user hasn't already provided these, ask before generating:

* **Target URL.** Required. If only HTML is provided, infer the URL or
  use `https://example.com/` and tell the user to edit it.
* **Tour purpose.** "Onboarding for X", "marketing walkthrough", "demo
  of the new dashboard". This shapes the narration tone.
* **Language(s).** Default to the language the user wrote their request
  in. Only generate a second-language file if the user explicitly asks
  for it (or asks for "both Korean and English", "ko + en", etc.).
  Selectors are language-neutral, so a second-language file is a
  selector-preserving translation of `name`, `description`, and
  annotation `text` — same `id`s, same selectors. Use parallel file
  names (`<slug>-en.json`, `<slug>-ko.json`).
* **Length preference.** Short (≤4 steps) vs. standard (5–7) vs.
  thorough (8+). Default to standard.

If the user gives only a URL with no other context, **default to: 6
steps, the user's request-language, marketing-walkthrough tone, ~6
seconds per step.**

---

## 4. Page analysis — finding good tour targets

A "good" spotlight target is something the user benefits from
*noticing*. Heuristics in priority order:

1. **The page's primary CTA** (sign-up button, "Get started", "Buy
   now" — or whatever action concludes the user's task on this page:
   a "Save", a "Submit", a "Run query"). Always tour-worthy.
2. **The hero / lede.** First impression. Usually `section.hero`,
   `header`, or the page's first `<h1>` container.
3. **Distinct product surfaces.** Features grid, pricing table,
   testimonials, FAQ. Each grid container is one step — don't tour
   every grid item individually.
4. **Navigation / language switcher / theme toggle.** Only if relevant
   to the user's goal.
5. **Settings, search, profile area.** Often, but skip if irrelevant.

Avoid:

* Decorative elements (background gradients, page-edge ornaments).
* Pieces of text that aren't visually distinct from neighbors.
* Anything below the fold that scroll-into-view can't easily show.

Order targets so the scroll path looks smooth — not jumping from
bottom back to top.

### Group homogeneous peers into one step (prefer sub-elements)

Whenever you spot **2+ visually similar peer items** on the same page —
feature cards in a grid, numbered process steps, pricing tiers, tab
buttons, navigation links, table column headers, settings tabs, FAQ
rows — strongly prefer a **single step with sub-elements** over
separate steps for each item. This is the default move, not an
advanced trick.

Why subs beat split steps for peer groups:

* **One coherent narration** — "여섯 가지 특징을 차례로 살펴봅니다…"
  reads more naturally than six separate sentences.
* **The walk shows structure** — the camera glides peer-to-peer with
  350 ms transitions, so the user *sees* this is a group, not a
  collection of unrelated highlights.
* **Step budget stays small** — a six-card grid collapses from 6 steps
  to 1, leaving room for the rest of the page inside the 4–8 sweet
  spot.

Practical pattern:

* Primary = the first item in visit order (top-left or step 1).
* `subElements[]` = the remaining peers, in DOM order.
* Pick `autoAdvanceMs` for the *whole* narration, then split into
  `subDwellsMs` of `(subs + 1)` entries. Even split is fine; bias the
  primary slightly longer if it's the lede.

Fall back to separate steps only when:

* Items are too dissimilar to share one narration (a hero + a CTA
  aren't peers — they're different beats).
* Items live on different pages — constraint G (§5.6) forbids
  cross-page subs; use separate steps with `pickedAtUrl` instead.
* Each item warrants its own user action (`waitForNavigation: true`)
  — only one action per step, so subs can't represent N click-targets.

Schema and timing details in §5.6 and §8. Worked example in §11.4.

---

## 5. JSON schema reference

### 5.1 `Scenario`

```ts
{
  schemaVersion: "0.1.2",          // literal
  id: string,                       // kebab-case, unique
  name: string,                     // displayed in extension's list
  url: string,                      // canonical page URL (first step's page)
  createdAt: string,                // ISO-8601
  updatedAt: string,                // ISO-8601
  steps: Step[],                    // 1 or more
  siteColors?: { text?: string[]; background?: string[] },
  siteFonts?: string[],             // CSS font-family stacks
  customColors?: { text?: string[]; background?: string[] }
}
```

`siteFonts` is helpful — fill it with a stack matching the target page
so annotation text doesn't visually clash. Example:

```json
"siteFonts": [
  "'Pretendard Variable', Pretendard, -apple-system, 'Asta Sans', sans-serif"
]
```

### 5.2 `Step`

```ts
{
  id: string,                                  // kebab-case
  name: string,                                // short, shown in step list
  description: string,                         // narration (read aloud)
  thumbnailDataUrl: null,                      // leave null when authoring by hand
  selectors: SelectorChain,                    // primary spotlight target — see §5.3
  annotations: Annotation[],                   // may be []
  autoAdvanceMs: number | null,                // null = manual advance only
  waitForNavigation: boolean,                  // true = action step (§8)
  pickedAtUrl?: string,                        // override the scenario url for this step
  subElements?: SubElement[],                  // auxiliary spotlight targets — see §5.6
  subDwellsMs?: number[]                       // per-node dwell ms, length = (subElements?.length ?? 0) + 1
}
```

`subElements` + `subDwellsMs` together describe a multi-element step
(v0.1.2+). During replay the spotlight walks **primary →
subElements[0] → subElements[1] → …** without ending the step or
restarting the narration. See §8 (sub-elements timing) and §11.4
(worked example).

### 5.3 `SelectorChain`

```ts
{
  layer1: { kind: "stable-attr",      cssSelector: string },
  layer2: { kind: "text-parent",      text: string, parentSelector: string, tagName: string },
  layer3: { kind: "visual-heuristic", x: number, y: number, width: number, height: number, nearbyText: string[] },
  framePath?: { index: number, url: string }[]   // only for iframe targets
}
```

Always populate all three layers. See §6 for how.

### 5.4 `Annotation` — discriminated union

#### `TextAnnotation`

```ts
{
  kind: "text",
  id: string,
  text: string,                                // user-visible label
  position: { x: number, y: number },          // absolute viewport coords (fallback)
  anchorOffset?: { x: number, y: number },     // preferred: offset from element top-left
  rotate?: number,                             // degrees
  entryAnimation?: EntryAnimation,
  style: {
    fontFamily: string,
    fontSize: number,                          // px, usually 14–20
    color: string,                             // CSS color
    bold: boolean,
    italic?: boolean,
    backgroundColor?: string,                  // 'transparent' or hex/rgb
    backgroundOpacity?: number,                // 0–1
    borderColor?: string                       // 'transparent' for no border
  }
}
```

#### `ArrowAnnotation`

```ts
{
  kind: "arrow",
  id: string,
  style: "excalidraw",                         // only kind in v0.1
  from: { x: number, y: number },              // absolute (fallback)
  to:   { x: number, y: number },
  fromAnchorOffset?: { x: number, y: number }, // preferred
  toAnchorOffset?:   { x: number, y: number },
  color: string,
  strokeWidth: number,                         // 2–6 typical
  entryAnimation?: EntryAnimation
}
```

#### `ShapeAnnotation`

```ts
{
  kind: "shape",
  id: string,
  shapeKind: "rectangle" | "ellipse" | "triangle" | "diamond"
           | "star"      | "callout" | "line"     | "block-arrow",
  bounds: { x: number, y: number, width: number, height: number },
  boundsAnchorOffset?: { x: number, y: number },  // preferred — w/h stay the same
  fill: string,                                   // 'transparent' = outline only
  stroke: string,
  strokeWidth: number,
  fillOpacity?: number,                           // 0–1
  rotate?: number,
  entryAnimation?: EntryAnimation
}
```

#### `FreeDrawAnnotation`

```ts
{
  kind: "freedraw",
  id: string,
  points: { x: number, y: number }[],
  pointsAnchorOffset?: { x: number, y: number }[],  // must be same length as points
  stroke: string,
  strokeWidth: number,
  strokeOpacity?: number,
  rotate?: number,
  entryAnimation?: EntryAnimation
}
```

### 5.5 `EntryAnimation`

```ts
{
  kind: "none" | "fade" | "slide-left" | "slide-right" | "slide-up"
      | "slide-down" | "bounce" | "zoom" | "rotate",
  durationMs: number,    // typically 400–800
  delayMs: number        // typically 0–400 (stagger across annotations)
}
```

### 5.6 `SubElement` (v0.1.2+)

```ts
{
  id: string,                       // kebab-case, unique within the step
  selectors: SelectorChain,         // same 3-layer rule as Step.selectors
  thumbnailDataUrl: string | null,  // null when authoring by hand
  pickedAtUrl: string               // MUST equal the parent step's pickedAtUrl
}
```

Constraint **G — same-page only**: every sub's `pickedAtUrl` must match
the parent step's `pickedAtUrl` (compared via `origin + pathname`,
ignoring `?query` and `#hash`). The Manuscript picker enforces this at
authoring time; when emitting JSON by hand, mirror the parent's
`pickedAtUrl` verbatim into every sub. To traverse elements across
different URLs, use separate **steps** with their own `pickedAtUrl`
(see §8, §11.5) — sub-elements are intra-page only.

Timing pairs with `Step.subDwellsMs`:

* `subDwellsMs[0]` = dwell on the primary element.
* `subDwellsMs[i]` = dwell on `subElements[i-1]`.
* Whenever subs exist: `Σ subDwellsMs === autoAdvanceMs`, regardless of
  `waitForNavigation`. When the total is edited, redistribute the gaps
  proportionally; the per-gap minimum is **100 ms**.
* For **action** steps with subs (`waitForNavigation: true`): same Σ
  invariant holds — the timed sub-walk plays first, then the action
  wait kicks in **on the final sub element** after its dwell finishes.
* For **pure action** steps (`waitForNavigation: true` AND no subs):
  `autoAdvanceMs` is `null` (the click is the only advance signal).
  Adding the first sub initializes `autoAdvanceMs` to
  `(subs + 1) × 2000 ms` so the invariant can hold; removing the last
  sub releases it back to `null`.

---

## 6. Selector chain authoring (the 3-layer rule)

The chain has three layers because pages change. Layer 1 is precise but
brittle; layer 3 is broad but always available. **Fill all three** so
replay degrades gracefully.

### Layer 1 — stable attribute

Use, in priority order:

1. `[data-testid="…"]`
2. `[data-test="…"]`
3. `#some-id` (if not generated by a build hash — e.g. CSS Modules
   produce ids like `Section_root__a1b2`, those are not stable)
4. `[aria-label="…"]`
5. A structural class chain (`section.hero`, `#features`, etc.) when
   nothing better exists. Pick a class the design system clearly
   "owns".

If nothing stable exists, fall back to a structural `tag:nth-of-type`
path — but try hard to find a real attribute first.

```json
"layer1": { "kind": "stable-attr", "cssSelector": "[data-testid=\"invite-btn\"]" }
```

### Layer 2 — text + parent

Useful when the element has visible text:

```json
"layer2": {
  "kind": "text-parent",
  "text": "Invite team",
  "parentSelector": "header.bar .user",
  "tagName": "BUTTON"
}
```

`text` matches the trimmed text content (exact). `parentSelector`
should be a 1–2 level ancestor selector that is itself stable.

### Layer 3 — visual heuristic

Last-resort recovery. Capture viewport rect + a few nearby text
snippets so the resolver can `elementFromPoint` and confirm.

```json
"layer3": {
  "kind": "visual-heuristic",
  "x": 1056, "y": 14, "width": 92, "height": 32,
  "nearbyText": ["Invite team", "+ New report", "JK"]
}
```

If you can't actually load the page (HTML-only input), estimate rect
from element ordering and a 1280×800 viewport. Manuscript will scroll
to the element if needed.

### Iframe targets

Only fill `framePath` when the target lives inside an iframe:

```json
"framePath": [{ "index": 0, "url": "https://embed.example.com/widget" }]
```

**Same-origin, depth-1 only.** Cross-origin iframes don't resolve at
replay time — Chrome blocks the resolver from reaching across origins,
so spotlight + replay show "not found". Picking in cross-origin frames
works via postMessage relay, but the resulting selectors only run in
the picker, not in playback. Nested iframes (depth ≥ 2) aren't
supported in v0.1 either. If the only good target is in a cross-origin
frame, prefer a wider container in the parent page that visually
contains the embed, and mention "the embedded panel" in narration.

---

## 7. Annotation patterns

Annotations are optional. A clean spotlight + good narration often
beats clutter.

### When to add text

* **Inline label** — name the highlighted element ("1. Invite", "Quick
  start"). Use `fontSize: 14–18`, `bold: true`, dark background, light
  text. Position with `anchorOffset` so it sticks near the element on
  resize.
* **Tip / caption** — a short tip ≤ 8 words. Bigger font is fine
  (18–22).

Don't add a text annotation that just repeats the narration the user
will hear.

### When to add shape

* **Highlight area** — `rectangle` or `ellipse` outline around a
  region that the spotlight can't precisely outline (e.g., a group of
  cards). `fill: 'transparent'`, `stroke: '#c9445b'`, `strokeWidth: 3`.
* **Numbered callout** — `callout` shape with a number inside.

### When to use freedraw / arrow

* **Arrow** — point from a label to its target. Always anchored to
  both endpoints (`fromAnchorOffset`, `toAnchorOffset`). Use
  `style: "excalidraw"`, `strokeWidth: 3`, color matching the page
  accent. This is the default pointer; reach for it first.
* **Freedraw** — circle, underline, or freeform mark. **Rarely
  needed.** Default to arrow or a `shape` outline unless the brand
  explicitly wants a hand-sketched look. Freedraw also requires
  hand-authoring a `points` array with matching `pointsAnchorOffset`
  (§5.4, §10), which is error-prone. The §11.3 example exists for
  reference, not as encouragement.

### `anchorOffset` rules

* Compute offset from the spotlight element's top-left.
* Above-left labels: negative y offset (e.g. `{ x: 12, y: -44 }`).
* Right-of labels: positive x offset wider than the element.
* Never set both `position` and `anchorOffset` to the same coords;
  `anchorOffset` wins when an anchor element resolves. Use `position`
  as the absolute fallback.

### Style defaults that look polished

```json
"style": {
  "fontFamily": "'EB Garamond', Garamond, serif",     // for labels — feels editorial
  "fontSize": 18,
  "color": "#ffffff",
  "bold": true,
  "backgroundColor": "#1a2438",
  "backgroundOpacity": 1,
  "borderColor": "transparent"
}
```

For body text or longer captions, switch to `'Pretendard Variable',
Pretendard, sans-serif` and a lighter background.

---

## 8. Narration & pacing

### Narration (`Step.description`)

Read aloud via Web Speech API during replay. Keep it:

* **One or two sentences.** ≤ 28 words is comfortable; ≤ 18 words is
  ideal.
* **Specific.** Name the element ("the Download button") rather than
  vague ("this thing").
* **In the user's language.** No mixed-language sentences unless
  explicitly requested.
* **Free of marketing fluff.** "Click to start" beats "Embark on
  your incredible journey".

### Step name (`Step.name`)

Shown in the prompter sidebar and as the floating-panel card title.
2–4 words. Title case for English, natural casing for Korean.

### `autoAdvanceMs`

Roughly **350 ms per spoken word** + 1 s buffer. Examples:

| Word count | autoAdvanceMs |
|---|---|
| 8 words | 4000 |
| 14 words | 6000 |
| 22 words | 8500 |

**Korean (and other non-space-separated languages):** word-count is a
poor proxy. Use **~250 ms per Korean syllable (글자)** + 1 s buffer,
since Web Speech reads Korean character-by-character at a slightly
faster cadence than English words. A 20-syllable sentence lands around
6 s. When in doubt, slightly over-estimate — finishing narration before
auto-advance is preferable to chopping it off.

Set to `null` only if the user explicitly wants manual advance.

### Action steps (`waitForNavigation: true`)

Use when the step is "now click this and the page changes":

* The replay pauses at the spotlight, shows "Click or type in this
  area," and waits.
* If the click navigates to a new URL, Manuscript resumes on the next
  page automatically — *as long as the bridge is not running in
  standalone mode*. In standalone bridge playback, action steps still
  pause but won't auto-navigate.

### Cross-page steps (`pickedAtUrl`)

Set `pickedAtUrl` per step when the step's target lives on a different
URL than the scenario's top-level `url`. The replay engine handles
navigation: when the previous step is an action step (`waitForNavigation:
true`), Manuscript waits for the user's click to land on the next
`pickedAtUrl`. If the next step's `pickedAtUrl` doesn't match the current
page and the previous step wasn't an action step, the extension prompts
the user to navigate before continuing.

A typical multi-page pattern: one **action step** with
`waitForNavigation: true` that spotlights a link/button, followed by
the next step's `pickedAtUrl` set to the destination page. See §11.5
for the worked example.

Skip `pickedAtUrl` for standalone bridge tours (they stay on one page
by design).

### Sub-elements timing (`subElements` + `subDwellsMs`)

When a step has `subElements`, the spotlight visits **primary →
sub[0] → sub[1] → …** with smooth transitions (350 ms cubic ease)
between nodes. Narration plays end-to-end *over* the whole sequence —
it does **not** restart per sub.

How to pick the per-node times:

* For a **non-action** step, `Σ subDwellsMs` must equal `autoAdvanceMs`.
  An even split is the safe default: e.g. `autoAdvanceMs: 6000` with
  primary + 2 subs → `subDwellsMs: [2000, 2000, 2000]`.
* For an **action** step, `subDwellsMs` only describes the playback
  length of the sub sequence; the action wait kicks in on the **last
  sub** after its dwell finishes.
* Minimum per-node dwell is **100 ms**. Sub-elements that flash by
  faster than that are pointless.
* Sub-elements should all live on the **same page** as the primary
  (constraint G, §5.6). For cross-page traversal, use separate steps.

---

## 9. Localization — same selectors, different language

Only generate a second-language file when the user explicitly asks for
it. Single-language is the default. When multilingual output **is**
requested:

* Selectors, step `id`s, and annotation `id`s are **identical** across
  languages. The DOM doesn't change with language.
* `Scenario.name`, `Step.name`, `Step.description`, and any annotation
  `text` are translated.
* `Scenario.id` differs per language so the extension lists them
  separately (e.g. `acme-onboarding-en`, `acme-onboarding-ko`).
* File names should be parallel and obvious. Pick whatever convention
  fits the user's repo — common patterns:
  `<slug>-en.json` / `<slug>-ko.json`, or `tour.en.json` /
  `tour.ko.json`, or just `en.json` / `ko.json`.

If the target page itself has bilingual `[data-en]` / `[data-ko]`
markers, you can either:

a. Target the language-neutral container (e.g. `section.hero`) — both
   language files share the same selectors for free.
b. Target language-specific elements (`h1[data-en]`, `h1[data-ko]`) —
   selectors then differ between files. Use this only when (a) isn't
   visually distinct enough.

Prefer (a) by default.

---

## 10. Validation checklist

Before emitting, verify:

- [ ] Parses as JSON (no comments, no trailing commas).
- [ ] `schemaVersion === "0.1.2"`.
- [ ] `id`, `name`, `url`, `createdAt`, `updatedAt`, `steps` all present.
- [ ] Each step has `id`, `name`, `description`, `selectors`,
      `annotations` (array, may be empty), `autoAdvanceMs`,
      `waitForNavigation`.
- [ ] Every selector chain has `layer1`, `layer2`, `layer3` (all three).
- [ ] `text-parent` layer's `tagName` is uppercase
      (`BUTTON`, `SECTION`, `DIV`).
- [ ] Annotation `id`s are unique within the file.
- [ ] Sub-element `id`s are unique within their parent step.
- [ ] No annotation references a font that the page or browser cannot
      provide. Including `'Asta Sans'` or `sans-serif` as a fallback in
      the family chain is good defense.
- [ ] If you wrote `anchorOffset`, you also wrote `position` (or
      `from/to/bounds/points`) as the absolute fallback.
- [ ] Step ordering tells a coherent story — scroll direction is mostly
      monotonic on each page; cross-page jumps are paired with an
      action step or an explicit `pickedAtUrl` change.
- [ ] No step is dead silent — even empty annotations must have
      narration in `description`.
- [ ] `createdAt` and `updatedAt` are valid ISO-8601 timestamps
      (`new Date(value)` returns a valid Date, not `Invalid Date`).
- [ ] If any `framePath` is present, it points at a same-origin,
      depth-1 iframe — the resolver can't reach into cross-origin or
      nested frames at replay time.
- [ ] For freedraw, `pointsAnchorOffset.length === points.length` if
      both arrays are present.
- [ ] If `subElements` is set: every sub's `pickedAtUrl` matches the
      parent step's `pickedAtUrl` (origin + pathname only — query and
      hash are ignored).
- [ ] If `subDwellsMs` is set: length === `(subElements?.length ?? 0) + 1`,
      every entry ≥ 100, and `Σ subDwellsMs === autoAdvanceMs`
      (the invariant now applies to action *and* non-action steps).
- [ ] If `waitForNavigation: true` AND `subElements` is set:
      `autoAdvanceMs` is the total playback length of the sub-walk
      (non-null, same as non-action). The action wait kicks in
      **on the final sub element** after its dwell finishes.
- [ ] If `waitForNavigation: true` AND no `subElements`:
      `autoAdvanceMs` must be `null` (pure action step — click is the
      only advance signal).

---

## 11. Worked example

The fragments below are reference snippets — the structure shown
applies to any page. The fully-rendered companion files
(`tour-en.json` / `tour-ko.json` next to this SKILL.md) happen to tour
the Manuscript landing page, but treat them as a model, not as the
"correct" subject of a tour.

Annotated highlights:

```json
{
  "id": "step-1",
  "name": "Welcome",
  "description": "Welcome to Manuscript — a Chrome extension for authoring DOM-aware product tours and manuals. Let me show you around.",
  "selectors": {
    "layer1": { "kind": "stable-attr", "cssSelector": "section.hero" },
    "layer2": { "kind": "text-parent", "text": "DOM-aware",
                "parentSelector": "main", "tagName": "SECTION" },
    "layer3": { "kind": "visual-heuristic",
                "x": 0, "y": 0, "width": 1200, "height": 480,
                "nearbyText": ["Manuscript", "Tours that don't break"] }
  },
  "annotations": [{
    "kind": "text", "id": "ann-1-text", "text": "1. Welcome",
    "position":     { "x": 80, "y": 80 },
    "anchorOffset": { "x": 24, "y": -52 },
    "style": {
      "fontFamily": "'EB Garamond', Garamond, serif",
      "fontSize": 18, "color": "#ffffff", "bold": true,
      "backgroundColor": "#1a2438", "backgroundOpacity": 1,
      "borderColor": "transparent"
    }
  }],
  "autoAdvanceMs": 6000,
  "waitForNavigation": false
}
```

Reasoning:

* **Selector layer 1** uses `section.hero` because the landing page's
  hero section is a stable, semantic CSS hook. No `data-testid` exists
  for it.
* **Layer 2** picks "DOM-aware" because it's distinctive English copy
  that appears once in the page — a good text anchor.
* **Layer 3** estimates a 1200×480 rect at the page top with text
  hints. If both layers above fail, the resolver tries this region.
* **Annotation** uses `anchorOffset` so a label stays glued ~24 px right
  and 52 px above the section's top-left as the user resizes.
* **autoAdvanceMs: 6000** — 17 words × ~350 ms ≈ 6 s.
* **No action step** because the user shouldn't have to click during a
  marketing tour.

### 11.1 Shape annotation — outlining a region

When the spotlight rect can't cleanly enclose a group of cards (e.g.,
a features grid with uneven gaps), drop a rectangle outline:

```json
{
  "kind": "shape",
  "id": "ann-2-rect",
  "shapeKind": "rectangle",
  "bounds":             { "x": 80,  "y": 360, "width": 1040, "height": 320 },
  "boundsAnchorOffset": { "x": 0,   "y": -8 },
  "fill": "transparent",
  "stroke": "#c9445b",
  "strokeWidth": 3,
  "fillOpacity": 0,
  "entryAnimation": { "kind": "fade", "durationMs": 500, "delayMs": 200 }
}
```

Notes: `fill: "transparent"` keeps it an outline only; the spotlight
darkens the surroundings, the outline highlights *which* surroundings.
`fillOpacity: 0` is redundant with `fill: "transparent"` but is
explicit for clarity. Use page-accent color for `stroke`.

### 11.2 Arrow annotation — pointing from label to target

When a text label sits some distance from the element it names, an
arrow ties them together. Anchor both endpoints so window resizing
doesn't break the pointer:

```json
{
  "kind": "arrow",
  "id": "ann-3-arrow",
  "style": "excalidraw",
  "from":             { "x": 240, "y": 120 },
  "to":               { "x": 480, "y": 220 },
  "fromAnchorOffset": { "x": 20,  "y": -28 },
  "toAnchorOffset":   { "x": 18,  "y": 12 },
  "color": "#1a2438",
  "strokeWidth": 3,
  "entryAnimation": { "kind": "slide-right", "durationMs": 600, "delayMs": 300 }
}
```

Notes: both `from`/`to` (absolute fallback) AND the two
`*AnchorOffset` (preferred) are supplied — Manuscript prefers the
anchored coords when the spotlight element resolves, and falls back
to absolute if it doesn't. `style: "excalidraw"` is the only kind in
v0.1 and gives a hand-drawn rough.js look.

### 11.3 Freedraw annotation — hand-drawn circle/underline

Freedraw is for the cases where an arrow feels too formal. A loose
circle around the target, or an underline beneath a phrase:

```json
{
  "kind": "freedraw",
  "id": "ann-4-circle",
  "points": [
    { "x": 720, "y": 60 }, { "x": 760, "y": 50 }, { "x": 790, "y": 58 },
    { "x": 805, "y": 80 }, { "x": 800, "y": 105 }, { "x": 775, "y": 118 },
    { "x": 740, "y": 116 }, { "x": 715, "y": 100 }, { "x": 710, "y": 78 },
    { "x": 720, "y": 60 }
  ],
  "pointsAnchorOffset": [
    { "x": -40, "y": -20 }, { "x": 0,   "y": -30 }, { "x": 30,  "y": -22 },
    { "x": 45,  "y": 0   }, { "x": 40,  "y": 25  }, { "x": 15,  "y": 38  },
    { "x": -20, "y": 36  }, { "x": -45, "y": 20  }, { "x": -50, "y": -2  },
    { "x": -40, "y": -20 }
  ],
  "stroke": "#c9445b",
  "strokeWidth": 4,
  "strokeOpacity": 0.9,
  "entryAnimation": { "kind": "fade", "durationMs": 800, "delayMs": 0 }
}
```

Notes: `pointsAnchorOffset` must be **the same length as `points`** —
one offset per point. The circle here is intentionally loose (10
points, slightly irregular spacing) so rough.js draws it
hand-sketched, not geometrically clean. Closing the loop by ending at
roughly the start point feels intentional.

### 11.4 Sub-elements — one step, multiple highlights

When the user wants to point at a row of related items (toolbar
buttons, feature cards, form fields) under a single narration, use
`subElements`. The primary is the first item; subs are the rest, in
visit order:

```json
{
  "id": "step-features",
  "name": "Features at a glance",
  "description": "Five things to know: spotlight, narration, annotations, sub-element traversal, and standalone playback.",
  "pickedAtUrl": "https://example.com/landing",
  "selectors": {
    "layer1": { "kind": "stable-attr", "cssSelector": "#features .card:nth-of-type(1)" },
    "layer2": { "kind": "text-parent", "text": "Spotlight",
                "parentSelector": "#features", "tagName": "DIV" },
    "layer3": { "kind": "visual-heuristic",
                "x": 120, "y": 600, "width": 220, "height": 180,
                "nearbyText": ["Spotlight", "Features"] }
  },
  "annotations": [],
  "autoAdvanceMs": 10000,
  "waitForNavigation": false,
  "subElements": [
    {
      "id": "sub-card-2", "thumbnailDataUrl": null,
      "pickedAtUrl": "https://example.com/landing",
      "selectors": {
        "layer1": { "kind": "stable-attr", "cssSelector": "#features .card:nth-of-type(2)" },
        "layer2": { "kind": "text-parent", "text": "Narration",
                    "parentSelector": "#features", "tagName": "DIV" },
        "layer3": { "kind": "visual-heuristic",
                    "x": 360, "y": 600, "width": 220, "height": 180,
                    "nearbyText": ["Narration"] }
      }
    },
    {
      "id": "sub-card-3", "thumbnailDataUrl": null,
      "pickedAtUrl": "https://example.com/landing",
      "selectors": {
        "layer1": { "kind": "stable-attr", "cssSelector": "#features .card:nth-of-type(3)" },
        "layer2": { "kind": "text-parent", "text": "Annotations",
                    "parentSelector": "#features", "tagName": "DIV" },
        "layer3": { "kind": "visual-heuristic",
                    "x": 600, "y": 600, "width": 220, "height": 180,
                    "nearbyText": ["Annotations"] }
      }
    }
  ],
  "subDwellsMs": [3500, 3300, 3200]
}
```

Reasoning:

* Primary + 2 subs = 3 nodes, so `subDwellsMs.length === 3`. The split
  isn't perfectly even because the primary card is the lede and gets
  slightly longer dwell — pick whatever feels natural, just keep the
  sum equal to `autoAdvanceMs` (here, 3500+3300+3200 = 10000).
* All three nodes share the same `pickedAtUrl` (constraint G) — no
  cross-page subs.
* Narration is one continuous sentence that names all five features;
  the spotlight just walks the eye through the first three cards
  while it plays.
* `waitForNavigation: false` — this is a *look, don't click* step.

The fixture in this example is a feature grid. The pattern adapts to
any homogeneous group: toolbar buttons, pricing cards, form steps,
table column headers, settings tabs, navigation items.

### 11.5 Multi-page tour — action step + `pickedAtUrl`

To carry the user from one URL to another, use **two consecutive steps**:
an action step on page A that spotlights the navigating link/button,
then a regular step whose `pickedAtUrl` is the destination URL.

```json
[
  {
    "id": "step-cta",
    "name": "Open pricing",
    "description": "Click the pricing link to see the plans.",
    "pickedAtUrl": "https://example.com/landing",
    "selectors": {
      "layer1": { "kind": "stable-attr", "cssSelector": "nav a[href='/pricing']" },
      "layer2": { "kind": "text-parent", "text": "Pricing",
                  "parentSelector": "nav", "tagName": "A" },
      "layer3": { "kind": "visual-heuristic",
                  "x": 940, "y": 16, "width": 70, "height": 32,
                  "nearbyText": ["Pricing", "Docs", "Sign in"] }
    },
    "annotations": [],
    "autoAdvanceMs": null,
    "waitForNavigation": true
  },
  {
    "id": "step-plans",
    "name": "Plans",
    "description": "Three plans — Free, Pro, and Team. Pick the one that fits.",
    "pickedAtUrl": "https://example.com/pricing",
    "selectors": {
      "layer1": { "kind": "stable-attr", "cssSelector": "section.plans" },
      "layer2": { "kind": "text-parent", "text": "Free",
                  "parentSelector": "main", "tagName": "SECTION" },
      "layer3": { "kind": "visual-heuristic",
                  "x": 0, "y": 200, "width": 1200, "height": 480,
                  "nearbyText": ["Free", "Pro", "Team"] }
    },
    "annotations": [],
    "autoAdvanceMs": 7000,
    "waitForNavigation": false
  }
]
```

Reasoning:

* The first step is an **action step** — `waitForNavigation: true`,
  `autoAdvanceMs: null`. Replay pauses until the user actually clicks
  the link.
* The second step's `pickedAtUrl` is the destination URL, so when the
  user's click navigates the browser to `/pricing`, Manuscript's
  cross-page resume picks the scenario back up on the new page.
* This pattern also works across **different origins** (e.g. spotlight
  a navigation link on `example.com`, then resume on `docs.example.org`)
  as long as the user actually clicks through. The selectors on each
  page run independently — no cross-origin selector resolution needed.
* Use this — not `subElements` — whenever the targets live on
  different URLs. Subs are intra-page only (§5.6 constraint G).

---

## 12. Generation pseudocode

```
function authorTour(url, opts):
    page    = fetchAndRender(url)
    intent  = opts.intent  or "marketing-walkthrough"
    lang    = opts.lang    or "en"
    nSteps  = opts.length  or 6

    targets = pickTargets(page, intent, nSteps)   // §4
    steps = []
    for i, target in enumerate(targets):
        chain   = buildSelectorChain(target)       // §6 — all 3 layers
        narrate = writeNarration(target, intent, lang)        // §8
        label   = composeInlineLabel(i+1, target, lang)       // §7 (optional)
        steps.append({
            id:                "step-" + (i+1),
            name:              shortName(target, lang),
            description:       narrate,
            thumbnailDataUrl:  null,
            selectors:         chain,
            annotations:       label ? [textAnnotation(label, target)] : [],
            autoAdvanceMs:     350 * wordCount(narrate) + 1000,
            waitForNavigation: target.requiresUserClick === true
        })

    scenario = {
        schemaVersion: "0.1.2",
        id:            slug("tour-" + page.host + "-" + lang),
        name:          tourTitle(page, intent, lang),
        url:           canonicalUrl(url),
        createdAt:     nowIso(),
        updatedAt:     nowIso(),
        steps:         steps,
        siteFonts:     guessFontStack(page)
    }

    runValidation(scenario)                                    // §10
    return JSON.stringify(scenario, null, 2)
```

When you produce the JSON, output **only the JSON** (no surrounding
explanation) unless the user explicitly asked for commentary.

---

*Schema version pinned to `0.1.2` at the time of writing — the
source-of-truth lives in `src/types/scenario.ts` (`SCHEMA_VERSION`).
If you see the source repo at a different value, follow that; the
migration code in `src/lib/json-schema.ts` keeps older `0.1.0` /
`0.1.1` scenarios readable. The `0.1.2` bump is additive — it
introduced `subElements` / `subDwellsMs` on `Step` without breaking
older files. The bridge library and the extension are independently
versioned (current bridge `manuscript-bridge.0.1.3.js`, legacy
compatibility `0.1.2.js`; extension reads
`chrome.runtime.getManifest().version`) — they may move forward
without bumping the scenario schema, since v0.1.x changes are
additive.*
