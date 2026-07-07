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

1. **One decision node at a time.** Walk down each branch of the decision tree,
   resolving dependencies between decisions one-by-one. Never batch unrelated
   decisions into one artifact. A small cluster of tightly-related sub-questions
   (e.g. an answer plus your confidence in it) is one node; a 10-question mega-form
   is not.
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

## Workflow per decision node

1. Identify the next unresolved decision and classify its TYPE (see widget catalog).
2. Copy the matching widget template from this skill's `widgets/` directory and
   adapt it: edit the `CONFIG` block at the top of the script (question, options,
   axes, your recommendation). Keep the widget's SDK wiring intact.
3. Write the artifact to `.grill-me-lavish/<nn>-<slug>.html` in the working
   directory (create the directory if needed; suggest gitignoring it). Number the
   files so the interview leaves an ordered trail.
4. Run `npx -y github:hotredsam/grill-me-lavish .grill-me-lavish/<nn>-<slug>.html` to open it in the
   browser. On the first artifact of a session tell the user the browser will open;
   subsequent artifacts reuse the same tab flow.
5. Run `npx -y github:hotredsam/grill-me-lavish poll .grill-me-lavish/<nn>-<slug>.html` to long-poll for
   the answer. The poll stays silent until the user acts — leave it running; if your
   harness limits foreground command duration, run it as a background task; if it is
   killed or times out, re-run it (queued feedback is never lost).
6. Parse the structured answer: each returned prompt's `prompt` field ends with
   `Context data:` followed by pretty-printed JSON — that JSON is the answer.
   Users can also add freeform annotations on top; treat those as additional context.
7. If poll returns `layout_warnings` with fresh error-severity findings, fix the
   HTML and recheck before involving the user; persistent or low-severity warnings
   can be noted and skipped.
8. Incorporate the answer, then either move to the next decision node (new artifact,
   repeat from step 1) or, when the tree is resolved, run
   `npx -y github:hotredsam/grill-me-lavish end <file>` and deliver the consolidated decision summary in
   the conversation.
9. If the user ends the session from the browser, stop polling and do not reopen
   uninvited — continue the remaining questions in plain chat or wrap up.
10. Every widget has a round "＋ More questions" button. If a poll returns a prompt
    whose data is `{ widget: "meta", request: "more-questions" }`, the user wants
    you to go DEEPER on that topic before moving on — generate additional,
    finer-grained questions about the current branch.
11. The user can send freeform chat from the browser at any time, even while you
    are working — treat those messages as interview input, not interruptions.

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

Default to `suggested-answer.html` when you have a strong recommendation and just need a
gut-check; default to `confidence-slider.html` for ordinary single questions. Reach
for the heavier widgets (matrix, pairwise, allocation) when the decision genuinely
involves competing options — do not use them to decorate simple questions.

## SDK contract the widgets rely on

- The engine injects `window.lavish` into the artifact iframe:
  `queuePrompt(prompt, options)`, `sendQueuedPrompts()`, `endSession()`,
  `setStatus(message)`, `snapshot()`.
- `queuePrompt(prompt, {tag, text, element, data})` — `data` is appended to the
  prompt as `"\n\nContext data:\n" + JSON.stringify(data, null, 2)`. That is the
  structured channel you parse from the poll result.
- Native form controls are interactive automatically. Custom clickable/draggable
  elements must carry `data-lavish-action` so Lavish shows a pointer and does not
  annotate them. A `data-lavish-question` wrapper makes repeated queues replace the
  prior unsent answer for the same question.
- Option clicks only update local widget state; the widget queues exactly ONE
  prompt from its explicit submit button, then calls `sendQueuedPrompts()` so the
  answer reaches your poll immediately.
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
