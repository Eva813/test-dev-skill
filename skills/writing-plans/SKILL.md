---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, verification steps, docs they might need to check. Give them the whole plan as bite-sized tasks. DRY. YAGNI. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Spec input:** The "spec" referenced throughout this skill does not have to be a formal requirements doc — it may be a frontend_sdd document (UI component breakdown, state structure, API-to-field mapping, Headless Pattern logic split) produced by the frontend-spec-synthesizer skills. Treat it as the authoritative source the same way you would treat a written spec.

**Test-planning sections are NOT tasks:** frontend_sdd §7 (測試規劃 / test candidates) is a planning
artifact for engineers to decide later — do **not** generate implementation tasks from it unless the
user explicitly asks for tests. The project currently has no frontend test framework.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** If working in an isolated worktree, it should have been created via the `superpowers:using-git-worktrees` skill at execution time.

**Save plans to:** `docs/superpowers/YYYY-MM-DD-<feature-name>/plan.md`
- (User preferences for plan location override this default)

**Spec snapshot:** When saving the plan, simultaneously copy root-level spec files (`pm_spec.md`, `design_spec.md`, `frontend_sdd.md` if they exist) to the feature folder (`docs/superpowers/YYYY-MM-DD-<feature-name>/`). This creates a self-contained project snapshot for future reference.

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during brainstorming. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## File Structure

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- You reason best about code you can hold in context at once, and your edits are more reliable when files are focused. Prefer smaller, focused files over large ones that do too much.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure - but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.

This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.

## Task Right-Sizing

A task is the smallest unit that carries its own verification cycle and is worth a
fresh reviewer's gate. Verification cycle can be: manually confirming behaviour in
the browser matches the spec / Figma / AC, type-check passing (`vue-tsc --noEmit`),
lint passing, or checking off each AC item line by line.
When drawing task boundaries: fold setup, configuration, scaffolding, and documentation
steps into the task whose deliverable needs them; split only where a reviewer could
meaningfully reject one task while approving its neighbor. Each task ends with an
independently verifiable deliverable.

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Implement the minimal working version of the component / logic" — one step
- "Verify the specific behaviour in the browser" — one step (the developer supplies a
  running URL with auth tokens; don't plan on starting the server yourself)
- "Run type-check (`vue-tsc --noEmit`) and confirm no errors" — one step
- "Run lint and confirm no errors" — one step
- "Commit" — one step

> If a task has unit-test coverage (e.g. a composable with a test framework in place),
> replace the manual-verify step with TDD steps (write failing test → run → implement → run passing).
> Both patterns are valid — choose based on what verification infrastructure exists for that task.

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

## Global Constraints

[The spec's project-wide requirements — version floors, dependency limits,
naming and copy rules, platform requirements — one line each, with exact
values copied verbatim from the spec. Every task's requirements implicitly
include this section.]

- 專案慣例：遵循 `AGENTS.md` 與 frontend_sdd §0 列出的既有 pattern；
  不引入專案不存在的框架 / 語言 / 測試工具（composables、.ts、Pinia、測試框架等），
  除非 spec 明確要求。[This line is mandatory in every plan.]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.vue`
- Modify: `exact/path/to/existing.ts:123-145`

**Interfaces:**
- Consumes: [what this task uses from earlier tasks — exact signatures]
- Produces: [what later tasks rely on — exact component names, props, emits, composable return values.
  A task's implementer sees only their own task; this block is how they learn the names and
  types neighboring tasks use.]

**Acceptance Criteria:** [List the AC / Figma states this task must satisfy, e.g.
"loading state shows skeleton", "API failure shows error message with retry button"]

- [ ] **Step 1: Implement component / logic**

```vue
<!-- actual complete code -->
```

- [ ] **Step 2: Verify**

Environment: ask the developer for a running local URL with auth tokens
             (pages are gated by `secretEmpl` / `info` query params — do NOT
             start the dev server or invent a route; see executing-plans Step A)
Navigate: [route to reach from that URL, and how — e.g. "from analysis/car_f,
          click 「看詳細分析」 on the 自小客 row; in-app nav carries the tokens"]
Steps: [specific human-readable actions, e.g. "navigate to tab", "hover bar row"]
Expected (human): [specific observable UI behaviour]

<!-- Add this block ONLY for layout-critical tasks:
     triggers = flex/grid CSS (gap/d-flex), getBoundingClientRect() calls,
     hover-driven position changes, CSS transition with computed positions -->
Playwright checks:
  Invariant suite: standard
  Interaction states: [the two+ distinct states invariant 3 must compare,
                       e.g. "hover .bar-row i=0" vs "hover .bar-row i=3"]
  Figma baseline node: [node ID from design_spec]
  Additional numeric assertions:
  - `<selector>.<prop>` [op] `<value>`   ← design_spec §N Lxx

- [ ] **Step 3: Type-check and lint**

Run: `vue-tsc --noEmit && eslint src/path/to/file.vue`
Expected: no errors

- [ ] **Step 4: Commit**

```bash
git add src/path/to/file.vue
git commit -m "feat: add specific feature"
```
````

## Rules for the `Playwright checks:` Block

You do **not** author the geometry checks. `Invariant suite: standard` refers to a
fixed, spec-free suite defined in `executing-plans` Step 2.5 (no self-overflow, no
escaping the content column, interaction must change geometry, visible elements have
size, screenshot). You may not restate, narrow, or replace it.

Your only three jobs in this block:

1. **Name the interaction states** the suite should compare. The suite asserts "this
   changed"; only you know which two states are meaningful for this component.
2. **Give the Figma baseline node ID** so the executor can fetch the mockup image.
3. **Optionally add numeric assertions — each with a `← design_spec §N Lxx` citation.**

**A geometry number you cannot cite must not go into the plan at all** — not into the
assertions, and not into the code block either. If you're about to write `width: 260px`
and can't point at the line of design_spec it came from, that's the signal you invented
it; go find the real number instead of enshrining the guess.

**A citation is not the same as authorization to trust the number.** Before using a
cited value as a hard constant, check whether the source sentence itself hedges it
(`僅供參考`, `示意`, `mock`, `反推`, `估計`). A hedged number describes what one
illustrative example happened to render as — it is not a scaling constant safe to
hardcode into every render. If the only citation you can find is hedged, say so
explicitly and compute the value at runtime instead (from the real data's max, or from
`getBoundingClientRect()`) — treat it the same as if no citation existed at all.

```
✗ const BAR_MAX_WIDTH = 471  // design_spec §10: 車體甲式 10筆/471px
   (design_spec §10 itself says this ratio is "反推僅供參考，不是確定數字" —
   citing the line doesn't make the number a fact)
✓ const barWidth = (count / maxCount) * 100 + '%'  // scales to whatever the real data is
```

**Absence of a citable number is not permission to leave the property unset.** When a
layout value genuinely isn't in design_spec, but the component still needs it to render
consistently — most commonly, two or more sibling instances must line up on a shared
column/track width — you must still assign one. Pick a value, mark it
`[assumed, not in design_spec]` right next to it in the plan, and state why the
assumption is safe (usually: "these N instances render side-by-side and must share one
width, or their content will diverge and misalign"). Do not resolve "I have no citation"
by omitting the declaration — an unset shared dimension is a worse defect than an
honestly-labeled guess, because it lets each instance's own content silently decide its
layout instead.

This applies to any repeated structural element that must render at a consistent size,
not just legend columns — a table's row-label column, cards in a grid, tab widths, an
icon gutter that must line up across rows. The common thread: **more than one
instance/row of the same thing renders on screen, and their shared dimension has no
citable source** — whatever the component happens to be.

A real failure this would have prevented: a `PieChart` component's legend column had no
`width` set, because no citable number existed for it. Two side-by-side instances
rendered with legends of different natural widths (their label text differed in length),
so the two pie circles ended up at different horizontal offsets — visibly misaligned,
even though neither instance's CSS was individually "wrong."

**Never write an assertion that restates your own implementation formula.** These are
worthless — they pass by construction, and they convert a bug into a specification:

```
✗ connector.left === panelRect.right - containerRect.left   (this IS the code)
✗ connector.width > 0                                        (a dead 20px stub passes)
✓ connector.width when hovering row 0 ≠ connector.width when hovering row 3
```

The third form is the one that has caught real defects, because it can be right without
you knowing the correct answer.

## Sibling Scan Is Mandatory Before Writing Any Component

Before writing a task's code block, grep the repo for **every class name and component
name** the task will use. If an implementation already exists, paste its declarations
into the plan as the baseline, and write an explicit reason for each deviation.

This is not optional politeness toward existing code — it is the cheapest defect filter
available, because the existing sibling has already absorbed the project's global CSS.
A real failure it would have prevented:

> A new `.btn-back` was written as `class="btn btn-back"` with hand-rolled border, color
> and padding. Three siblings already existed (`views/CarAnalysis`, `views/AccidenceAnalysis`,
> `views/StaticViews/CarDetail`), all of them `class="btn btn-outline-primary btn-back"`
> with an explicit `width`. The new one dropped `btn-outline-primary`, dropped
> `data-gtm-cuv`, and — critically — never overrode `width`, so it inherited
> `width: 110px` from the project's global `.btn` rule while its label needed ~210px.
> The text overflowed the button. Nothing in the plan looked wrong; the plan was
> internally consistent and simply hadn't looked at the neighbors.

Record the scan in the task's `**Files:**` block, e.g.:

```
**Sibling baseline:**
- `.btn-back` — exists in views/CarAnalysis/CarAnalysis.scss:16 (`width: 170px`,
  `padding: 7.5px 20px`, relies on `.btn-outline-primary` for border/color).
  Deviation: this page's label is longer, so `width: auto` instead of `170px`.
```

Note the interaction with the global-CSS point: when a sibling sets a property you
*weren't planning to set at all*, that is usually because a global rule needs
overriding. An absent declaration is the easiest kind of difference to miss, so compare
what the sibling declares against what you declare, not just the values you both share.

## Acceptance Criteria Must Preserve Span-Level Spec Detail

When a design_spec (or pm_spec) describes a single line of text as having multiple differently-styled
segments — partial bold, a colored substring, a fixed label followed by a computed value — summarizing
it into one sentence drops exactly the detail an implementer and a verifier both need. Copy the spec's
own markup verbatim into the Acceptance Criteria instead of paraphrasing it.

This pattern shows up in more than one notation, and the thing to recognize is the *concept* — "this
line is really N differently-styled pieces" — not any single syntax. Don't only pattern-match on
`**bold**` markers; the same compound-style line just as often shows up as a bracketed decomposition,
e.g. `Des「保單數： 101 張」[Body 16 "保單數：" + Title 16 right "101" + Body 16 "張"]`. That `[A + B + C]`
form is exactly as much a per-segment style spec as inline bold markers are — it's just easier to read
past, because it doesn't visually scream "special formatting to preserve" the way `**bold**` does.

Two real instances from the same page make the point, caught at different times specifically because
they used different notations:
- `賠付險種：**車體險、竊盜險**，總理賠金額：500,238元` with `boldWeight=500 on bold parts` — the `**...**`
  marks only the claim-type names as bold, not the amount. A plan that reduces this to "顯示賠付險種與
  總理賠金額" loses that boundary, and it's easy for the resulting SCSS to bold both spans instead of
  one — a real defect that static verification later caught precisely because this AC form was followed.
- `Des「保單數： 101 張」[Body 16 "保單數：" + Title 16 right "101" + Body 16 "張"]` — the bracket notation
  says the number needs its own `Title 16` (500-weight) span, separate from the label and unit. A plan
  that flattens this into `<div class="status">保單數： {{ply_cnt}} 張</div>` as one plain-text
  interpolation loses that boundary just as thoroughly as the bold-marker case would — and because the
  resulting AC only said "顯示保單數與客戶數", nothing about it implied a second span should exist, so
  static verification passed even though the number never got its 500-weight span. Same underlying
  mistake, missed only because the notation didn't look like the one everyone was already watching for.

When you hit this pattern, regardless of which notation the spec used:
- Quote the spec's literal segmentation (bold markers, bracket decomposition, or whatever form it takes) directly in the Acceptance Criteria, not a paraphrase.
- If you know which CSS selector will render which span, name it next to the span it belongs to.
- Treat "which part gets which style" as its own AC line, not a detail folded into a broader "display X and Y" bullet.

## Code Blocks Must Match Project Style

Before writing any task's code block, Read the current content of the file being modified
(or the closest existing sibling file for new files, e.g. another `views/{Page}/{Page}.js`).
The code you put in the plan must match that file's real conventions — imports style,
state management pattern, naming, SCSS structure — not generic Vue best practices.

## Repeating / List-Type Visual Elements Deserve a Fresh Figma Look

Most sections translate cleanly from design_spec's prose straight into a plan's code block — trust the
spec and move on, there's no need to re-derive it. But an element that repeats (a `v-for` list, a set of
cards, a bullet-point block) where each repetition has more than one sub-element is exactly the shape
that tends to lose its layout direction in translation: whether the sub-elements sit in the same row or
stack top-to-bottom rarely survives being written as prose, even when the source Figma frame stores it
unambiguously (Figma's own data literally has a `layout.mode: row` field on the container — the
information isn't missing upstream, it just didn't make it through the prose hand-off).

If design_spec's description of the repeated item doesn't explicitly say "same row" or "stacked" for its
sub-elements, don't guess from the prose — call the Figma MCP tool directly for that specific node
before writing the code block, the same way figma-spec-extractor's own Step 3.3 drills into a specific
node when its first pass wasn't precise enough. Get the node ID from the block's **Figma Node ID anchor**
in design_spec (the element list and each component's detail-block header carry it precisely so you don't
have to guess which node) — then re-query that one node and read its `layout.mode`/`gap` and child order
straight from the source. This is a narrow, occasional check for one specific, recognizable risk shape
(repeating containers with multiple sub-elements each), not a general license to re-derive every section
from Figma — doing that everywhere would throw away the whole point of having a written spec to work from.

If the block has no node ID anchor (an older design_spec produced before extraction recorded them), treat
that absence as a signal in itself: the prose you're working from was never anchored to a verifiable
source, so rather than fabricating a code block from an ambiguous description, ask the developer for the
node ID (or to re-run figma-spec-extractor on that block) before committing the layout to code.

## No Placeholders

Every step must contain the actual content an engineer needs. These are **plan failures** — never write them:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" (without actual test code)
- Manual verification steps that omit concrete actions and expected observable results
- "Similar to Task N" (repeat the code — the engineer may be reading tasks out of order)
- Steps that describe what to do without showing how (code blocks required for code steps)
- References to types, functions, or methods not defined in any task

## Remember
- Exact file paths always
- Complete code in every step — if a step changes code, show the code
- Exact commands with expected output
- DRY, YAGNI, frequent commits
- Spec requirements are the primary goal; tests are welcome where infrastructure supports them, but never a prerequisite

## Self-Review

After writing the complete plan, look at the spec with fresh eyes and check the plan against it. This is a checklist you run yourself — not a subagent dispatch.

**1. Spec coverage:** Skim each section/requirement in the spec. Can you point to a task that implements it? List any gaps.

**2. Placeholder scan:** Search your plan for red flags — any of the patterns from the "No Placeholders" section above. Fix them.

**3. Type consistency:** Do the types, method signatures, and property names you used in later tasks match what you defined in earlier tasks? A function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a bug.

**4. Design style variable numbers match the spec:** Any literal px / color / weight value you wrote into a task's
code block (SCSS, inline style, whatever) should trace back to a number that's actually in the
design_spec token table — not one you half-remembered while writing the snippet. Put your plan's value
next to the spec's value for each one and look for transcription slips. This kind of error (a `148px`
label width copied down as `110px`) is invisible to every other check in this list, because the plan is
perfectly internally consistent — the code just quietly stopped matching the spec it came from.

**5. Geometry reconciliation:** Check 4 covers colors, weights and font sizes. Do the same
for **positions, widths, lengths and coordinates** — a separate pass, because geometry fails
differently: a color is either right or wrong, but a coordinate can be individually plausible
and still describe the wrong layout.

Build a table of every absolute coordinate / width / length in the design_spec, and next to
each one write **what your plan's CSS will actually produce**. Compute it; don't eyeball it.
Sum the widths and gaps of a flex row and check the total against the container.

One rule carries most of the value here:

> **When the same coordinate appears twice or more in the spec, write down which end is
> fixed.** Two data points that share a value are telling you about an anchor.

The failure this rule is for: design_spec recorded two hover examples of a connector line as
`x=408, width=221` and `x=100, width=529`. Both end at 629. That shared endpoint means the
*right* end is anchored (at the neighboring panel's left edge) and the *left* end tracks the
hovered bar. The plan read it backwards — anchored the left end and let the right end float —
which made the line a constant 20px stub for every row. Each number in the plan was
individually defensible; only putting the two examples side by side exposed it.

Also flag anything that doesn't reconcile even if you can't tell which side is wrong. In the
same plan, a panel was specified `260px` wide while the spec's longest bar inside it was
`222px` — after padding and labels the bar had ~52px to live in. That arithmetic
contradiction was visible without knowing the correct width, and was enough to stop and
re-derive it (1140 − 629 = 511px).

If you find issues, fix them inline. No need to re-review — just fix and move on. If you find a spec requirement with no task, add the task.

## Spec Snapshot Step

Before handoff, execute:

```bash
# Extract feature slug from plan filename (e.g., 2026-07-01-sales-analysis-v3-iins-expansion)
FEATURE_SLUG=$(basename $(dirname $PLAN_PATH))
FEATURE_DIR="docs/superpowers/$FEATURE_SLUG"

# Copy current spec files as snapshot (only if they exist in root)
cp -v pm_spec.md $FEATURE_DIR/ 2>/dev/null || true
cp -v design_spec.md $FEATURE_DIR/ 2>/dev/null || true
cp -v frontend_sdd.md $FEATURE_DIR/ 2>/dev/null || true

# Cleanup: remove root working copies once snapshot is verified identical
# (feature folder is the single surviving location — prevents stale root copies)
for f in pm_spec.md design_spec.md frontend_sdd.md; do
  if [ -f "$f" ] && diff -q "$f" "$FEATURE_DIR/$f" >/dev/null 2>&1; then
    rm -v "$f"
  fi
done
```

If a root copy exists but differs from the snapshot, **stop and ask** the user which version
is authoritative — do not delete or overwrite silently.

**Result:** Feature folder now contains the complete project snapshot (plan + specs) and is the
single location for these docs. Root working copies are removed after a verified-identical copy.

## Execution Handoff

After saving the plan and snapshot, offer execution choice:

**"Plan complete and saved to `docs/superpowers/<feature-slug>/plan.md` with spec snapshot. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?"**

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Fresh subagent per task + two-stage review

**If Inline Execution chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:executing-plans
- Batch execution with checkpoints for review