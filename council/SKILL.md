---
name: council
description: Convene a council of model delegates (Claude Opus 5.5 and Claude Fable 5.1) to independently answer, review, or critique the same request, then synthesize their findings, disagreements, and next steps. For code reviews, score each round out of 10 — per rubric area, with a recorded reason for every score — and iterate fixes until every delegate is satisfied, without scope creep. Hill-climb rounds record scores and reasons as git notes the Hill Climb Diff app reads. Use when the user says "council", "ask the council", "get a second opinion", "council review", "hill climb", or wants multiple models to weigh in.
---

# council

Send the same request to several model delegates in parallel, collect their independent answers, and act as chair. The chair gathers context, synthesizes what the delegates said, decides which recommendations to adopt, and keeps scope tight.

## Delegates

| Delegate | Model ID |
| --- | --- |
| Claude Opus 5.5 | `claude-opus-5-5` |
| Claude Fable 5.1 | `claude-fable-5-1` |

To add or remove a delegate, edit this table. Everything below applies to every delegate listed here.

## When to use

- The user says "council", "ask the council", "convene the council", or "council review".
- The user wants a second opinion or multiple models to weigh in on a plan, design, decision, or piece of code.
- The user asks for a multi-model code review before or after opening a pull request.

## Instructions

### 1. Gather context (chair)

- Restate the request in one or two sentences, and decide whether it is a **question/plan** (answer and recommend) or a **review** (critique existing work).
- Collect the relevant context so delegates don't start from zero: the user's request, relevant file paths, the diff (`but diff`, or `gh pr diff <n>` for an open PR), constraints, and anything already decided.
- **Write down the scope** explicitly, e.g. "Scope: the retry logic added in `src/client.ts`. Out of scope: the logging refactor." This line goes into every delegate prompt and every later round.
- **Decide the rubric.** For a hill-climb run (rounds the Hill Climb Diff app will read), the default rubric areas are **accuracy, clarity, structure, completeness, style**; the user can set different areas. For other code reviews a rubric is optional. Whatever rubric is set goes into every delegate prompt, every round.

### 2. Dispatch delegates in parallel

Give every delegate the **same prompt** and dispatch them all at once so they can't see each other's answers.

- **Preferred:** use your harness's subagent/Task tool with each delegate's model set to the model ID from the table.
- **Fallback:** if the harness can't pick a model per subagent, shell out to the Claude CLI in read-only mode, one background process per delegate:

  ```bash
  claude -p --model claude-opus-5-5  --permission-mode plan \
    --disallowedTools "Edit" "Write" "NotebookEdit" < /tmp/council-prompt.md > /tmp/council-opus.md &
  claude -p --model claude-fable-5-1 --permission-mode plan \
    --disallowedTools "Edit" "Write" "NotebookEdit" < /tmp/council-prompt.md > /tmp/council-fable.md &
  wait
  ```

- **Read-only for reviews and feedback:** delegates must not edit files, commit, push, or comment on PRs. Say this in the prompt and enforce it with tool restrictions where possible. Only the chair makes changes.

Delegate prompt template:

```markdown
You are a delegate on a review council. Work independently.

## Request
<the user's request, restated>

## Scope
In scope: <...>
Out of scope: <...>  Do not recommend changes outside the scope.

## Context
<file paths, diff, constraints, prior decisions>

## Rules
- READ-ONLY. Do not edit files, run git write commands, or post anything.
- Be concrete: cite file:line, and separate blocking issues from nice-to-haves.

## Output format
1. Summary (2-3 sentences)
2. Blocking issues (must fix before you'd approve)
3. Non-blocking suggestions
4. Scores   <- code reviews only
   - When a rubric is set, one line per area: `<area>: N/10 — <reason>`. The reason
     is required: say concretely what kept the score below 10 and what change would
     raise it. A score with no reason doesn't count.
   - Final line: `Score: N/10 — <overall reason>` (keep the `Score: N/10` prefix exact)
5. Verdict: SATISFIED or NOT SATISFIED   <- code reviews only
```

If a delegate fails or times out, retry once. If it fails again, record it as unavailable for that round and continue with the rest. Never fabricate its response.

### 3. Synthesize (chair)

Report back to the user in this shape:

- **Consensus findings:** points raised by more than one delegate.
- **Unique findings:** useful points only one delegate raised, credited to that delegate.
- **Disagreements:** where delegates conflict, what each said, and your call with a one-line reason.
- **Recommended next steps:** what you plan to do, in order.
- **Scores** (code reviews only): one line per delegate.

Judge findings on their merits, not by vote count. A single correct blocking finding outweighs two delegates who missed it.

### 4. Code review rounds

- **Round:** one full review pass in which every delegate reviews the current state of the work. A fresh review after fixes starts a new round. Number rounds from 1.
- **Score:** ask each delegate every round for an overall score out of 10 and, when a rubric is set, one score per area — each with its reason. Record exactly what it gave, scores and reasons both. If it gave no score, record `no score`; if it gave a score without a reason, keep the score and record `no reason given`. Never infer, estimate, or invent either.
- **Loop:**
  1. Run a round (steps 2–3).
  2. For each recommendation, decide whether to **adopt** it (you agree and it's in scope), **reject** it (you disagree, with a reason), or **defer** it (valid but out of scope, so list it as a follow-up and don't do it).
  3. Make the adopted changes yourself, aiming fixes at the recorded score reasons: a reason that's addressed should move that area's score next round. Run relevant checks, and commit per the repo's VCS rules.
  4. Start a fresh round. Tell each delegate what changed since the last round and why you rejected or deferred anything it raised, and include that delegate's own previous scores and reasons, per area, so it can judge whether each reason has been addressed. It can push back if it thinks a rejection is wrong.
  5. Stop when every delegate returns `SATISFIED` with no remaining blocking issues.
- **Deadlock:** if a delegate is still not satisfied after 3 rounds because of a disagreement you've already reasoned through (not a real unfixed bug), stop and ask the user to decide. Don't loop forever.

### 5. Fight scope creep

You are responsible for keeping scope small. Delegates will suggest more each round, so hold the line:

- Every round uses the **same scope statement** from step 1. Don't widen it unless the user approves.
- Adopt only changes that fix real problems in the in-scope work: bugs, regressions, missing tests for the new behavior, or clear violations of project conventions.
- Defer refactors of untouched code, new features, speculative abstractions, stylistic preferences, and "while you're here" ideas. List them under **Follow-ups**. Don't implement them.
- A later round may only raise *new* blocking issues if they come from the fixes themselves or were genuinely missed. Treat nitpicks that keep coming back as non-blocking.
- If the scope really does need to grow, ask the user first.

### 6. Report results

For code reviews, build the results table. Include one column per round and one row per delegate:

```markdown
## Council review
Rounds: 2

| Delegate | Round 1 | Round 2 |
| --- | --- | --- |
| Claude Opus 5.5 | 7/10 | 9/10 |
| Claude Fable 5.1 | 8/10 | 10/10 |
```

Use `no score` in a cell when a delegate didn't give one, and `unavailable` when it failed to respond.

### Hill-climb runs: record scores and reasons per round

When rounds are committed one by one for the Hill Climb Diff app (the user says "hill climb", or round commits follow the `round N` convention), attach a git note to each round's commit after committing it:

```bash
git notes --ref hillclimb add -F <notefile> <commit>
```

The app parses the bare score lines above the first heading and displays everything after them, so lay the note out as:

1. Score lines first, one per line, exactly this shape — **no reasons on these lines**:

   ```
   Claude Opus 5.5: 8/10
   Claude Opus 5.5 / accuracy: 9/10
   Claude Opus 5.5 / clarity: 8/10
   Claude Fable 5.1: 8/10
   ...
   ```

   One overall line plus one line per rubric area, per delegate. Omit any line the delegate didn't give.
2. A blank line, then each delegate's full review under a `## <Delegate> — review` heading. Keep the review's `<area>: N/10 — <reason>` lines intact: they are the record of *why* each score was given, and the app shows them alongside the scores.
3. End with `## Chair notes going into this round`: what was adopted, rejected, or deferred, and which score reasons the next round's fixes target.

Also write a `base` note the same way on the parent commit of round 1, from the baseline review. These notes are the per-run record: the app displays the reasons, and later rounds read them to check whether each reason was addressed.

Where the table goes:

- **Open PR exists:** edit the PR description. Don't post a comment. Add the `## Council review` section **just before the model and harness line at the end** of the description (for example, `🤖 Generated with Claude Code`), and replace any existing `## Council review` section rather than adding a second one:

  ```bash
  gh pr view <n> --json body -q .body > /tmp/pr-body.md
  # edit /tmp/pr-body.md: insert/replace the section above the trailing model/harness line
  gh pr edit <n> --body-file /tmp/pr-body.md
  ```

  If the description has no model/harness line, put the section at the end.
- **Work will end in a PR you're creating:** include the section in the description when you create the PR, using the same placement rule.
- **No PR:** put the same section in your final reply only.

Always finish with a short final reply covering the outcome, the scores table, deferred follow-ups, and anything left for the user to decide.
