---
name: grill-me-lavish
description: Interview the user relentlessly about a plan or design, rendering each question as a rich HTML input widget in the browser — rankings, point allocations, tradeoff pads, forced choices — to extract priorities, weights, and gut reactions that plain Q&A flattens. Use when the user says "grill me with lavish", wants to stress-test a plan visually, or when rich structured elicitation would beat plain text questions.
author: Sam McHargue (hotredsam)
---

# grill-me-lavish

Interview the user relentlessly about their plan or design until you reach shared
understanding — but instead of asking questions as terminal text, render each question
as a purpose-built HTML input widget in the browser using the `grill-me-lavish` CLI
(this repo's own engine, derived from Kun Chen's lavish-axi). The goal is elicitation
FIDELITY: surface what the user really means, including unstated constraints and
relative priorities, which multiple-choice and freeform text flatten.

You do not need anything installed — always invoke the engine as
`npx -y github:hotredsam/grill-me-lavish` (first run installs it; later runs use the
npx cache). If engine output shows a follow-up command starting with
`grill-me-lavish`, run it as `npx -y github:hotredsam/grill-me-lavish ...`.

## The grilling discipline (non-negotiable)

1. **One decision per section, dependency order overall.** The interview is ONE
   growing page; each `<section>` on it is exactly one decision node. Serve a
   batch of sections at a time (start with 3+), walking the decision tree in
   dependency order — later batches build on what earlier answers unlock. The
   user answers in any order within a batch and sends when they choose.
2. **Always attach your recommendation.** Every widget displays YOUR recommended
   answer (every widget template has a `recommendation` config field — fill it in,
   never leave it empty). Pre-select or pre-position the widget to your
   recommendation when the widget supports it, so the user reacts to a concrete suggestion
   instead of facing a blank control.
3. **Explore before you ask.** If a question can be answered by exploring the
   codebase (or the files/documents under discussion), explore instead of asking.
   Only spend the user's attention on questions only they can answer: intent,
   priorities, tradeoffs, constraints, taste.
4. **Structured output only.** Every widget emits its answer through
   `window.lavish.queuePrompt()` with an `options.data` JSON payload. You never
   parse freeform prose to figure out what the user chose.
5. **Keep grilling until shared understanding.** After each answer, update your
   model of the plan, decide what the answer unlocks or invalidates, and move to
   the next unresolved branch. Summarize the full decision record at the end.
6. **Never re-ask what is already answered.** Keep a running record of every
   answer received. When feedback arrives, acknowledge it once and advance to new
   ground — do not repeat a request, re-send the same browser reply, or ask the
   user to redo an interaction that already round-tripped.
7. **Minimum three widget questions per interview.** One widget is not a
   grilling. Before wrapping up, you must have collected at least three widget
   answers (not counting meta/more-questions signals); if you believe you are
   done sooner, you have not walked the tree deep enough — close with a
   suggested-answer gut-check of your consolidated understanding.

## Workflow: one growing interview page

1. Create `.grill-me-lavish/<slug>-interview.html` from
   `widgets/interview-shell.html` (create the directory if needed; suggest
   gitignoring it). Set the title, delete the example sections, and add your
   opening batch: strongly consider a `bubble-burst` section first for mass
   context intake, then 2+ deeper sections (the 3-minimum is satisfied on first
   render). Each section = one decision, built by adapting the matching widget
   from `widgets/` into the shell's SECTION BLOCK pattern: unique `qNN` id,
   controls, and a `gml.bindSection("qNN", collect)` script where `collect`
   returns `{summary, data}` (or null while incomplete). Answers AUTO-QUEUE as
   controls change — no per-section buttons; the progress strip and per-section
   "saved ✓" line show state, and the chrome replays the queue after every live
   reload so answered sections stay visibly answered.
2. Open it: `npx -y github:hotredsam/grill-me-lavish .grill-me-lavish/<slug>-interview.html`.
3. Long-poll: `npx -y github:hotredsam/grill-me-lavish poll <same file>`. Leave it
   running (background task if your harness limits foreground duration; re-run if
   killed — queued feedback is never lost).
4. Answers arrive as a BATCH when the user presses "Send N answers" — could be
   3 or 100. Parse each prompt's `Context data:` JSON block. Freeform annotations
   and chat messages ride along; treat them as interview input.
5. When a prompt's data is `{ widget: "meta", request: "more-questions" }` (the
   round ＋ button, delivered alone via `sendPrompt`), APPEND 1-4 new sections to
   the SAME file before the `APPEND NEW SECTIONS` marker — never remove or edit
   existing sections, continue the `qNN` numbering, keep in-section ids
   suffixed with the section id. The page live-reloads taller; scroll position
   and the user's queued answers survive (the in-page badges reset — the
   Conversation panel keeps the truth). Do NOT wait for their answers before
   appending; the ＋ deliberately sends nothing else.
6. After each answered batch, incorporate, then either append the next batch of
   sections (dependency order) or, when the tree is resolved, run
   `npx -y github:hotredsam/grill-me-lavish end <file>` and deliver the
   consolidated decision summary in the conversation.
7. If poll returns fresh error-severity `layout_warnings`, fix and recheck before
   involving the user; persistent or low-severity ones can be noted and skipped.
8. If the user ends the session from the browser, stop polling and do not reopen
   uninvited — wrap up in chat.
9. Single-question artifacts (a lone widget file per decision) remain fine for
   quick one-off gut-checks outside a full interview.

## Choosing the widget (decision-type → widget)

Read the widget file before adapting it — each has a header comment stating its
decision type and exact output shape.

| Decision type | Widget file |
| --- | --- |
| Relative priority among N items (order reveals weight) | `widgets/drag-rank.html` |
| Intensity of preference — spend 100 points, forces real tradeoffs | `widgets/point-allocation.html` |
| Position on two competing axes (e.g. speed vs robustness) | `widgets/tradeoff-pad.html` |
| Rate several options across several criteria | `widgets/decision-matrix.html` |
| True preference via this-vs-that forced choices | `widgets/pairwise.html` |
| A single answer plus how sure the user is (tells you where to dig) | `widgets/confidence-slider.html` |
| Scope triage: must-have / nice-to-have / won't-do | `widgets/moscow-buckets.html` |
| React to your recommended default: accept / tweak / reject with reason | `widgets/suggested-answer.html` |
| When words fail: freeform sketch plus note | `widgets/annotation-canvas.html` |
| Triage several risks on likelihood × impact (quadrant = action) | `widgets/risk-matrix.html` |
| The user wants to WRITE, with structure — 2-4 labeled prompts, keyed answers | `widgets/free-text-form.html` |
| Mass context intake at interview START — 30-50 tap-to-select bubbles | `widgets/bubble-burst.html` |

Consider OPENING the interview with `bubble-burst.html` — 30-50 pre-generated
context bubbles knock out a pile of basics in seconds before the one-at-a-time
deep questions begin.

Default to `suggested-answer.html` when you have a strong recommendation and just need a
gut-check; default to `confidence-slider.html` for ordinary single questions. Reach
for the heavier widgets (matrix, pairwise, allocation) when the decision genuinely
involves competing options — do not use them to decorate simple questions.

## SDK contract the widgets rely on

- The engine injects `window.lavish` into the artifact iframe:
  `queuePrompt(prompt, options)`, `sendQueuedPrompts()`, `sendPrompt(prompt,
  options)` (grill-me-lavish addition: submits ONE prompt immediately, leaving
  the queued batch untouched — what the ＋ button uses), `endSession()`,
  `setStatus(message)`, `snapshot()`.
- `queuePrompt(prompt, {tag, text, element, data})` — `data` is appended to the
  prompt as `"\n\nContext data:\n" + JSON.stringify(data, null, 2)`. That is the
  structured channel you parse from the poll result.
- Native form controls are interactive automatically. Custom clickable/draggable
  elements must carry `data-lavish-action` so Lavish shows a pointer and does not
  annotate them. A `data-lavish-question` wrapper makes repeated queues replace the
  prior unsent answer for the same question.
- Option clicks only update local state; each section queues exactly ONE prompt
  from its explicit Queue-answer button (re-queues replace via the
  `data-lavish-question` wrapper). Only the user's "Send all answers" flushes the
  queue to your poll. Queued prompts persist in the chrome across live reloads.
- Poll output is TOON with `session`, `prompts[]` (each: `uid`, `prompt`,
  `selector`, `tag`, `text`), optional `layout_warnings[]`, and `next_step`.
- The SDK `<script>` is injected at the END of `<body>`, after inline artifact
  scripts have run — widgets must look up `window.lavish` lazily inside event
  handlers, never capture it at load time.
- Widgets degrade gracefully when opened without Lavish (answers log to console),
  so you can preview them directly in a browser while developing.

## Request

$ARGUMENTS

If the request above is non-empty, that is the plan/design to grill the user about —
begin the interview now. If empty, infer the plan under discussion from the
conversation; if there is none, ask the user what plan to grill them on (that first
bootstrap question may be plain text).
