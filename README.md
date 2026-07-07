# grill-me-lavish

An agent skill **and its own render/feedback engine** that interviews you
relentlessly about a plan or design — but instead of asking questions as terminal
text, it renders each question as a purpose-built HTML input widget in your
browser: drag-to-rank lists, 100-point allocations, 2-axis tradeoff pads,
forced-choice duels, scope-triage buckets, and more.

```
/grill-me-lavish here's my plan for the migration — grill me
```

## Why

Plain Q&A flattens what you actually mean. Asked "which of these matter?", everyone
answers "all of them." The right instrument extracts more:

- **Ordering** (drag-to-rank) reveals relative weight.
- **Spending 100 points** forces real tradeoffs and reveals intensity — you can't
  give everything 100.
- **A dot on a 2-axis pad** captures a stance ("fast-ish but not reckless") that no
  multiple-choice option offers.
- **Forced this-vs-that duels** expose true preference better than rating scales.
- **A confidence slider** attached to an answer tells the agent exactly where to
  keep digging and which branches to close.

The goal is elicitation *fidelity*: surfacing real intent, unstated constraints, and
relative priorities — not just collecting answers faster.

## How it works

1. The agent identifies the next unresolved decision in your plan and picks the
   widget that fits the decision type.
2. It writes a small self-contained HTML artifact (one decision per page) under
   `.grill-me-lavish/` and opens it in your browser via the bundled engine:
   `npx -y github:hotredsam/grill-me-lavish <file>`.
3. You interact — drag, allocate, place the dot, pick a verdict. Every widget shows
   the agent's recommended answer and usually starts pre-set to it, so you react to
   a concrete suggestion instead of a blank form.
4. The widget sends your answer back as **structured JSON** (never prose the agent
   has to parse) through the engine's long-poll:
   `npx -y github:hotredsam/grill-me-lavish poll <file>`.
5. The agent incorporates it and moves to the next branch of the decision tree.
   The round **＋ More questions** button on every widget tells the agent to go
   deeper on the current topic, and you can chat freely from the browser at any
   time — even while the agent is working.

## Lineage: grill-me and lavish-axi

- The interview methodology is the tiny `grill-me` skill's, unchanged: interview
  relentlessly until shared understanding, walk each branch of the decision tree
  resolving dependencies one at a time, ONE question at a time, always give your
  recommended answer, and explore the codebase instead of asking when possible.
  grill-me-lavish only upgrades the *instrument* each question is asked with.
- The engine in `src/` is derived from
  [lavish-axi](https://github.com/kunchenguid/lavish-axi) by
  [Kun Chen (@kunchenguid)](https://x.com/kunchenguid) (MIT — see
  [LICENSE-lavish-axi](LICENSE-lavish-axi)). This repo vendors and modifies that
  code rather than tracking upstream: the chrome is rebranded, the conversation
  composer never blocks while the agent is working, it runs on its own port (4429)
  and state directory (`~/.grill-me-lavish-state/`), so it coexists with a real
  lavish-axi install. The in-artifact SDK (`window.lavish`, `data-lavish-*`
  attributes) is intentionally unchanged, so artifacts stay compatible with
  upstream Lavish.

## Install

The skill, with the [`skills`](https://github.com/vercel-labs/skills) CLI:

```sh
npx skills add hotredsam/grill-me-lavish --skill grill-me-lavish
```

Add `-g` to install for all projects instead of just the current one.

Or manually: copy `skills/grill-me-lavish/` (the folder containing `SKILL.md` and
`widgets/`) into your project's `.claude/skills/` (or `~/.claude/skills/` for all
projects).

The engine needs no separate install — the skill invokes it as
`npx -y github:hotredsam/grill-me-lavish`, which fetches this repo on first use and
caches it.

## Usage

In an agent that exposes skills as slash commands (Claude Code):

```
/grill-me-lavish <the plan or design to stress-test>
```

Or just say "grill me with lavish about X" — the skill's description also triggers
when rich structured elicitation would beat plain Q&A.

You can also drive the engine directly, exactly like lavish-axi:

```sh
npx -y github:hotredsam/grill-me-lavish <html-file>        # open a session
npx -y github:hotredsam/grill-me-lavish poll <html-file>   # long-poll for feedback
npx -y github:hotredsam/grill-me-lavish end <html-file>    # end a session
```

## Widget catalog

| Widget | Fires when… |
| --- | --- |
| `drag-rank` | relative priority among N items is the question — ordering reveals weight |
| `point-allocation` | intensity matters — spending 100 points forces real tradeoffs |
| `tradeoff-pad` | the decision is a position on two competing axes (speed vs robustness) |
| `decision-matrix` | several options need rating across several criteria |
| `pairwise` | true preference is best exposed by forced this-vs-that duels |
| `confidence-slider` | an ordinary single question — the confidence tells the agent where to dig |
| `moscow-buckets` | scope triage: must-have / nice-to-have / won't-do |
| `suggested-answer` | the agent has a concrete default and needs accept / tweak / reject + reason |
| `annotation-canvas` | words fail — sketch it freehand with a note |

Every widget emits structured JSON through the injected SDK
(`window.lavish.queuePrompt(...)` with a `data` payload). Each widget file is
self-contained (no CDN dependencies) with a `CONFIG` block at the top the agent
adapts per question, includes the round **＋ More questions** button, and degrades
gracefully (logs to console) when opened outside a session.

## Roadmap

Decided by grilling its own author with it (2026-07-06) — priorities by 100-point
allocation: new widget types 30 · reliability 25 · interview flow 25 · theming 20.

- New widget types (candidates under triage: risk matrix, timeline picker, budget
  range, A/B compare, heat-map grid)
- Port the CPA v3 Night / Sepia / Sakura themes with a theme picker
- Interview flow: instant response to ＋ More questions, visible interview progress
- Every interview asks a minimum of three widget questions (now a SKILL.md rule)

## Requirements

- **Node.js ≥ 22** (for `npx`)
- An agent that supports [Agent Skills](https://agentskills.io) and can run shell
  commands — e.g. **Claude Code**
- A local browser (the engine serves sessions at `127.0.0.1:4429`; state lives
  under `~/.grill-me-lavish-state/`)

## Credit

The engine is a fork-in-place of
[lavish-axi](https://github.com/kunchenguid/lavish-axi) by
[Kun Chen (@kunchenguid)](https://x.com/kunchenguid) — most of the code underneath
is his, and the annotate/queue/poll interaction model is entirely his design. The
interview methodology is the `grill-me` skill's, unchanged.

## License

MIT — see [LICENSE](LICENSE) for this repo's additions and
[LICENSE-lavish-axi](LICENSE-lavish-axi) for the vendored lavish-axi code.
